# NB-Whisper for OpenWhispr

Norske tale-til-tekst-modeller fra Nasjonalbiblioteket i [OpenWhispr](https://github.com/OpenWhispr/openwhispr) 1.9.2, med nynorsk som eget språkvalg.

OpenWhispr leveres med Whisper-modellene fra OpenAI. De forstår norsk, men er trent på engelsk i hovedsak, og treffer ujevnt på navn, stedsnavn og fagord. NB-Whisper er trent på norsk tale av Nasjonalbiblioteket og gir vesentlig bedre resultat.

Bokmål og nynorsk ligger i samme modellfil. Det trengs ingen egen nynorskmodell; målformen styres av språkvalget i programmet, og patchen legger inn nynorsk som eget valg ved siden av «Norwegian». Den offisielle appen har bare det siste, som i praksis gir bokmål.

Endringen er 28 linjer fordelt på tre filer i modell- og språkregisteret. Ingen annen del av programmet er berørt.

**Forutsetninger.** Selve patchen er plattformuavhengig — det er ren JSON og TypeScript. Byggeoppskrifta og alle målinger i dette dokumentet gjelder derimot **Windows med et NVIDIA-kort**. Modellene kjører også på prosessoren, men da med flere sekunders ventetid per ytring i stedet for under to. Bruker du Mac, se avsnittet [macOS](#macos) nederst.

Tallene er målt på én bestemt maskin — en Intel i9-9900K med RTX 2070 og 8 GB skjermminne. [Full spesifikasjon står under Målinger](#målinger). Skjermkortets 8 GB er den avgjørende begrensningen i det meste av det som følger.

**Skal du bare sette det opp?** Se [Kom i gang](INSTALLASJON.md) — installasjon og innstillinger, steg for steg. Resten av dette dokumentet er bakgrunnen: målinger, årsaker og feller.

*English readers: the notes below are in Norwegian, but there is a [summary in English](#in-english) at the end.*

## Konklusjonen først

Det korte svaret, for den som ikke vil lese hele veien:

> **Bruk NB-Whisper Large kvantisert (1,08 GB), og slå av tekstoppryddingen.**
> Da kommer teksten i det øyeblikket du slipper tasten, og ingen språkmodell kan finne på å bytte ord underveis.

Konklusjonen bygger på målinger, ikke antakelser. Her er hva som ble prøvd, og hva hvert forsøk viste:

| Forsøk | Resultat |
|---|---|
| Byttet til stadig mindre Whisper-modeller, ned til 141 MB | Ingen bedring. Flaskehalsen lå ikke i talegjenkjenningen. |
| Fant at opprydningsmodellen brukte 15,6 s på å starte, og stengte seg selv hvert femte minutt | Forklarte hele ventetiden. |
| Flyttet oppryddingen til Ollama, som kjører CUDA og kan holde modellen lastet | 15,6 s → under ett sekund. 3,9 GB VRAM frigjort. |
| Vurderte NB-Whisper full presisjon (3,1 GB) i stedet for kvantisert | Droppet. Marginal gevinst, og ikke plass ved siden av noe annet. |
| Testet kvaliteten på oppryddingen med Gemma 3 1B | Modellen byttet ut ord med andre ekte norske ord. Slått av. |
| Testet på nytt med Gemma 3 4B, fire ganger så stor | Endret 14,6 % av teksten og fant på et partinavn. Forkastet. |

Sluttresultatet er enklere enn veien dit: talegjenkjenning alene, ingen etterbehandling, **1,5–2 sekunder**.

Oppskriften for å kjøre opprydding mot Ollama ligger likevel lenger nede, for den som vil prøve. Selve oppsettet er målt og fungerer: 765 ms svartid, modellen liggende på skjermkortet. Det som ikke holdt, var kvaliteten på en modell med én milliard parametere.

Om en større modell unngår problemet, er ikke undersøkt her. Et kort på 8 GB har uansett ikke minne til den ved siden av talegjenkjenningen.

Detaljene, tallene og fellene står i avsnittene under.

## Modellene

Nasjonalbiblioteket publiserer modellene åpent på Hugging Face, i repoet [NbAiLab/nb-whisper-large](https://huggingface.co/NbAiLab/nb-whisper-large). Patchen legger inn disse to:

| Modell i appen | Kildefil | Størrelse | VRAM i drift |
|---|---|---|---|
| NB-Whisper Large (norsk, kvantisert) | [ggml-model-q5_0.bin](https://huggingface.co/NbAiLab/nb-whisper-large/resolve/main/ggml-model-q5_0.bin) | 1,08 GB | ca. 1,6 GB |
| NB-Whisper Large (norsk, full presisjon) | [ggml-model.bin](https://huggingface.co/NbAiLab/nb-whisper-large/resolve/main/ggml-model.bin) | 3,1 GB | ca. 3,6 GB |

Appen laster ned filen første gang modellen velges, og lagrer den lokalt som `ggml-nb-large-q5_0.bin` eller `ggml-nb-large-full.bin`.

Den kvantiserte modellen anbefales til vanlig bruk. Kvalitetsforskjellen er marginal, mens minnebehovet er under halvparten. Se avsnittet om minne nedenfor.

Vilkårene for modellvektene framgår av modellkortet hos Nasjonalbiblioteket.

## Installasjon

```bash
git clone https://github.com/OpenWhispr/openwhispr.git
cd openwhispr
git apply /sti/til/nb-whisper.patch
npm install
npm run download:whisper-cpp
```

Kommandoen `npm run download:whisper-cpp` må kjøres separat. `npm run dev` henter ikke whisper.cpp av seg selv.

Bygging på Windows:

```bash
npx electron-builder --config electron-builder.unsigned-win.json
```

Den usignerte konfigurasjonen er nødvendig. Standardoppskriften signerer med OpenWhisprs eget Azure-sertifikat, som ikke er tilgjengelig utenfor prosjektet. Windows vil melde «Windows beskyttet PC-en din» ved installasjon av et usignert bygg; velg «Mer info» og deretter «Kjør likevel».

## Oppsett

Fem punkter avgjør om resultatet blir brukbart. Alle er verifisert på et bygg av 1.9.2.

### Ikke logg inn

Et selvbygget OpenWhispr har ingen `VITE_OPENWHISPR_API_URL`. Ved innlogging forsøker programmet å hente en «workspace policy» fra et API det ikke når. `policyRules.ts` er skrevet *fail closed*, og avviser da samtlige funksjoner, også lokal transkribering som ikke er avhengig av nett.

Symptomene er meldingen «No setup option is available — contact your administrator» under førstegangsoppsettet, eller at NB-modellen ikke lar seg velge. Løsningen er Settings → Profile → Sign Out.

### Minnebudsjettet på skjermkortet

Talegjenkjenningen og språkmodellen som rydder teksten deler samme skjermkort. Målinger fra et kort med 8 GB (RTX 2070):

| Komponent | VRAM |
|---|---|
| NB-Whisper kvantisert, inkl. KV-cache og buffere | 1,6 GB |
| NB-Whisper full presisjon | 3,6 GB |
| Llama 3.2 3B, standard opprydningsmodell | 3,1 GB |
| Nettleser og øvrige Electron-programmer | 2–3 GB |

Når talegjenkjenningen ikke får plass på kortet, faller den tilbake til prosessoren uten å melde fra. En large-modell på CPU bruker flere sekunder per ytring. Med den kvantiserte modellen på kortet ligger svartiden på 1,5–2 sekunder fra tasten slippes til teksten står i vinduet.

### Opprydningsmodellen bestemmer ventetiden

Slipper du dikteringstasten og venter et kvarts minutt på teksten, er den nærliggende antakelsen at talegjenkjenningen sliter. Den er allerede ferdig.

```
[tast slippes]
   │
   ├─  0,5 s   whisper-server transkriberer ferdig   (CUDA, port 8178)
   │
   └─ 15,6 s   llama-server starter opp fra dvale    (Vulkan, port 8221)
                  └─ teksten dukker opp i vinduet
```

**Programmet er to prosesser, ikke én.** Talegjenkjenningen på port 8178 gjør lyd om til tekst. Opprydningsmodellen på port 8221 tar imot den teksten og retter tegnsetting og grammatikk. De er uavhengige, og bare den første berøres av hvilken Whisper-modell du velger.

Det er forklaringen på et resultat som ellers er uforståelig: å bytte til stadig mindre Whisper-modeller, helt ned til 141 MB, gir ingen målbar effekt på ventetiden. Flaskehalsen inntreffer etter at Whisper er ferdig.

Og programmet viser ingen egen indikator for det andre leddet. Forsinkelsen inntreffer nøyaktig der man forventer at transkriberingen skjer, så den tilskrives talegjenkjenningen.

**Nedstengingen etter fem minutter er tilsiktet.** En modell på 3 GB som blir liggende, opptar kortet døgnet rundt og fortrenger både nettleseren og talegjenkjenningen. Programmet gir derfor fra seg minnet og betaler oppstarten på nytt ved behov. Avveiningen passer lange, sammenhengende økter. Diktering er det motsatte: korte ytringer med pauser mellom, og da betales oppstarten om og om igjen.

**Et paradoks i målingene.** Talegjenkjenningen laster en modell på 3,1 GB på 9,7 sekunder. Opprydningsmodellen bruker 15,6 sekunder på 2,0 GB. Større fil, kortere tid.

Forskjellen ligger i hvordan de er bygget. Talegjenkjenningen kjører et CUDA-bygg med ferdigkompilerte rutiner; oppstarten er i hovedsak overføring av vekter fra disk til kort. Språkmodellen kjører Vulkan, som bygger sine beregningsrutiner ved oppstart.

Vulkan-kompileringen er den mest nærliggende forklaringen på differansen, men den er ikke isolert og målt her — det er en slutning fra tallene, ikke et eget måleresultat.

Uansett årsak lar det seg ikke konfigurere bort. Programmet leverer ett enkelt llama-bygg:

```
%APPDATA%\open-whispr\bin\
   llama-vulkan
   whisper-cuda
```

Ingen CUDA-variant for språkmodellen følger med, heller ikke på NVIDIA-maskiner.

Tre alternativer:

* Slå funksjonen av under Settings → Language Models → fanen «Dictation Cleanup» → «Enable text cleanup». NB-Whisper leverer velformet norsk tekst uten etterbehandling.
* Velg en mindre modell. Gemma 3 1B (0,81 GB) dekker over 140 språk, norsk inkludert. Llama 3.2 oppgir offisiell støtte for åtte språk, og norsk er ikke blant dem. Qwen3-familien kan la resonneringstekst følge med i utdataene. En mindre modell forkorter innlastingen, men ikke den delen av oppstarten som er uavhengig av størrelse.
* Sett opprydningen til en ekstern server, som beskrevet nedenfor. Det er den eneste løsningen som fjerner ventetiden helt.

### Ekstern språkmodell som fjerner oppstartstiden

Programmet kan bruke en hvilken som helst OpenAI-kompatibel server til opprydningen, i stedet for sin egen innebygde. `providerConnectionTest.js` navngir Ollama, LM Studio og vLLM, og innstillingen `remoteReasoningType` tar verdien `openai-compatible`.

Dette løser begge problemene på én gang. Ollama bruker CUDA på NVIDIA-kort, ikke Vulkan, slik at shaderkompileringen faller bort. Viktigere er `OLLAMA_KEEP_ALIVE`: settes den til `-1`, blir modellen liggende på kortet i stedet for å lastes ut. Da er det ingen oppstart å betale for i det hele tatt. Femminutters-nedstengingen i den innebygde serveren er en fast verdi i kildekoden og kan ikke settes opp.

Alt kjører fortsatt lokalt. Ingen tekst forlater maskinen.

Oppsett på Windows:

```powershell
winget install Ollama.Ollama
ollama pull gemma3:1b
setx OLLAMA_KEEP_ALIVE -1
```

I programmet velges deretter «Self-Hosted» under Settings → Language Models, med adressen `http://127.0.0.1:11434/v1`, i hver av fanene du vil bruke.

Kostnaden er en bakgrunnstjeneste som holder på minnet permanent. En modell på 1 GB ved siden av NB-Whispers 1,6 GB legger beslag på 2,6 GB av et kort på 8 GB, hvilket lar seg forsvare. En større modell gjør det ikke.

Å bytte ut `llama-server-vulkan.exe` med et CUDA-bygg fra llama.cpp er teknisk mulig, siden kommandolinjen er den samme, men frarådes. Filen blir overskrevet ved oppdatering eller reparasjon av programmet, og CUDA-bibliotekene må legges ved manuelt.

### Hver funksjon har sitt eget modellvalg

Settings → Language Models har fem faner — Dictation Cleanup, Voice Assistant, Translation, Note Formatting og Chat — og **hver av dem velger leverandør, adresse og modell for seg**. Endrer du én, følger ikke de andre med.

Dette er verdt å merke seg, for konsekvensen er ikke åpenbar: står bare én av fanene på «Local», startes den innebygde `llama-server` likevel, og den legger beslag på 3–4 GB av skjermkortet enten du bruker funksjonen eller ikke. Å sette Dictation Cleanup til noe annet er altså ikke nok. Fanene med avslått funksjon, som Translation, teller ikke med.

Adressefeltet er også per fane. Bytter du en fane til «Self-Hosted», står feltet tomt med en tilfeldig eksempeladresse som grå plassholder, og modellista er tom til du har fylt inn adressen og trykt «Apply & Refresh».

### Notatopptak har eget modellvalg

Settings → Speech-to-Text har separate faner for «Dictation» og «Note Recording», med hvert sitt modellvalg. Notatopptak følger ikke med når dikteringsmodellen endres, og melder `Whisper model "turbo" not downloaded` dersom den peker på en modell som ikke er lastet ned. Begge faner bør settes til NB-modellen.

### Slå av automatiske oppdateringer

Settings → Preferences → Notifications → «App updates» bør slås av. En offisiell oppdatering overskriver det lokale bygget, og NB-modellene forsvinner fra modellisten.

## Målinger

Alle tall nedenfor er målt på den samme maskinen, med NB-Whisper Large kvantisert som talegjenkjenning hele veien:

| | |
|---|---|
| Prosessor | Intel Core i9-9900K, 8 kjerner / 16 tråder, 3,6 GHz |
| Hovedkort | Gigabyte Z390 DESIGNARE |
| Minne | 48 GB DDR4, 2133 MHz |
| Skjermkort | NVIDIA GeForce RTX 2070, 8 GB, driver 595.79 |
| System | Windows 11 Pro (build 26200) |

Skjermkortets 8 GB er den avgjørende begrensningen i alt som følger. På et kort med mer minne faller flere av avveiningene annerledes ut.

### Hvor rask er talegjenkjenningen

Målt i to omganger, med den kvantiserte NB-modellen på skjermkortet.

**Kontrollerte lengder.** Syntetisk tale fra Windows-stemmen Microsoft Jon (nb-NO), sendt direkte til `whisper-server` på port 8178:

| Lyd inn | Prosessering | Andel av taletiden |
|---|---|---|
| 24 s | 1 988 ms | 8,2 % |
| 32 s | 2 108 ms | 6,7 % |
| 73 s | 5 333 ms | 7,3 % |
| 100 s | 6 286 ms | 6,3 % |
| 145 s | 6 959 ms | 4,8 % |
| 291 s | 9 513 ms | 3,3 % |

**Ekte tale.** Norwegian Parliamentary Speech Corpus fra Nasjonalbiblioteket — opptak fra stortingssalen med kontrollerte transkripsjoner, utgitt som CC0. Ekte stemmer, ekte akustikk fra et rom med mikrofonanlegg, og mange ulike talere.

| Materiale | Lyd inn | Prosessering | Andel |
|---|---|---|---|
| 25 enkeltinnlegg | 206 s | 17,3 s | 8,4 % |
| 99 innlegg satt sammen | 900 s (15 min) | 56,7 s | 6,3 % |

Prosesseringen ligger altså på **5–8 % av tiden det tok å si det**, og andelen synker med lengden fordi den faste oppstartskostnaden fordeles på mer lyd. Femten minutter sammenhengende tale ble transkribert komplett på under ett minutt.

**Målt inne i programmet.** Med feilsøkingsloggen påslått rapporterer programmet selv hvert ledd. For vanlig diktering:

| Lyd | Transkribering | Full rundtur |
|---|---|---|
| 3,0 s | 495 ms | 497 ms |
| 3,3 s | 607 ms | 609 ms |

Konverteringen med ffmpeg tok 67 ms for disse. Selve talegjenkjenningen er altså ferdig godt under ett sekund; den ventetiden man opplever i tillegg, går med til å avslutte opptaket og lime inn teksten.

For opplasting av femten minutter så tidslinjen slik ut:

```
ffmpeg-konvertering av 28,8 MB      239 ms
whisper-server, 901,8 sekunder lyd   54 370 ms
```

**Framdriftsindikatoren er misvisende ved opplasting.** Den fyller seg mens filen leses og konverteres — under et halvt sekund — og står deretter stille gjennom hele det virkelige arbeidet. whisper.cpp rapporterer ingen framdrift underveis, så programmet har ingenting å vise. De siste prosentene *er* jobben.

**Opplastingsveien sender ikke med språkvalget.** Loggen viser `lang = auto` for opplastede filer, mens vanlig diktering kjører `lang = no`. Modellen må altså gjette språk på opplastet lyd, uansett hva som står under Transcription language. På norsk går det som regel bra, men det er unødvendig arbeid, og på korte opptak kan den bomme.

**Treffsikkerhet er ikke målt her.** Det ble forsøkt mot fasitene i korpuset, men tallet blir misvisende: Stortingets transkripsjoner skriver tall som ord etter uttale — «innstilling hundre og énogsytti L tjueseksten tjuesytten» der modellen skriver «Innst. 171 L for 2016–2017». Samme innhold, men hvert siffer telles som feil. En reell måling krever normalisering av tall, forkortelser og egennavn, og det er ikke gjort. Tallene over gjelder derfor fart, ikke kvalitet.

### Utgangspunktet

| Ledd | Tid |
|---|---|
| Talegjenkjenning, modellen på skjermkortet | 1,5–2 s |
| Talegjenkjenning, modellen falt tilbake til prosessoren | flere sekunder per ytring |
| Innebygd `llama-server` (Vulkan), oppstart | 15,6 s |
| Innebygd `llama-server`, nedstenging etter inaktivitet | 5 min |

Oppstarten på 15,6 sekunder ble betalt på nytt for hver diktering som fulgte en pause på over fem minutter, hvilket i praksis vil si annenhver gang. Til sammenligning laster talegjenkjenningen en større modell — 3,1 GB mot 2,0 — på 9,7 sekunder, fordi den kjører CUDA og ikke Vulkan.

### Ekstern server i stedet

Opprydningen ble flyttet til Ollama med Gemma 3 1B, kjørende på samme maskin med `OLLAMA_KEEP_ALIVE=-1`.

| Ledd | Tid |
|---|---|
| Første kall, modellen lastes inn | 94 s |
| Påfølgende kall, kort tekst | 246–362 ms |
| Reell diktering, 730 tokens inn | 765 ms |

Førstegangsinnlastingen på 94 sekunder skjedde mens skjermkortet var nesten fullt, og inntreffer bare én gang. Deretter blir modellen liggende, og oppstartskostnaden er borte for godt. `ollama ps` bekrefter `100% GPU` og `UNTIL: Forever`.

Minnebildet før og etter, med talegjenkjenningen lastet i begge tilfeller:

| | Før | Etter |
|---|---|---|
| Ledig VRAM | 512 MiB | 2143 MiB |
| Innebygd `llama-server` | 3939 MB | avsluttet |
| Opprydningsmodell på kortet | — | 877 MB |

### Hvorfor opprydningen likevel ble slått av

Med alt på plass fungerte kjeden, og svartiden var god. Kvaliteten var det ikke.

Gemma 3 1B skrev om ord den ikke hadde grunnlag for å endre. I én diktering ble «hele kjeden virker» til «hele kilden virker». Feilen er vanskelig å oppdage, nettopp fordi resultatet er et ekte norsk ord som passer i setningen. Manglende tegnsetting ser man med én gang; et ombyttet ord gjør man ikke.

Opprydningen ble derfor slått av for diktering. NB-Whisper leverer velformet norsk på egen hånd, og feilene den gjør er synlige feil. For tekst som skal publiseres, er det en bedre avveining enn en liten språkmodell som retter tegnsetting og samtidig gjetter på ord.

**Testet på nytt med en fire ganger større modell.** Da opprydningen var slått av, ble det ledig minne på kortet, og Gemma 3 4B (2,49 GB) fikk plass ved siden av talegjenkjenningen. Antakelsen var at flere parametere ville gi mindre gjetting.

Testen brukte de 1801 ordene fra 15 minutter stortingstale, kjørt gjennom programmets egen opprydningsinstruks, sammenlignet ord for ord mot råteksten:

| | |
|---|---|
| Ord inn | 1801 |
| Ord ut | 1666 |
| Erstattet | 160 |
| Slettet | 84 |
| Lagt til | 19 |
| **Andel av teksten endret** | **14,6 %** |

135 ord forsvant. Og blant erstatningene:

```
FRA:  ... skal bli regulert i forskrift. Likevel har ...
TIL:  ... skal bli regulert i forskrift. Senterpartiet har ...
```

Modellen satte inn et partinavn som aldri ble sagt, i en tekst om stortingsbehandling. Setningen er grammatisk korrekt og fullt plausibel, og ville passert enhver rask korrekturlesing.

Andre eksempler fra samme kjøring: «og det gir oss muligheter for» ble til «samt»; «gjort framlegg om» ble til «foreslått». Innhold komprimeres bort, og talerens egne ord normaliseres.

Fire ganger så mange parametere gjorde det altså ikke tryggere. Det er heller ikke overraskende når man leser instruksen programmet sender med: den ber uttrykkelig modellen «fix obvious transcription errors from context». Ordbyttene er tilsiktet oppførsel, ikke en feil — og modellen har ingen måte å vite hva som faktisk var en feil.

For tekst som skal publiseres, med tall, kilder og egennavn, er det feil verktøy uansett størrelse.

## Feilsøking

Programmet oppgir ikke i grensesnittet hvilken modell som faktisk er lastet, eller om den ligger på skjermkortet. Tre kommandoer gir svaret.

Modellen talegjenkjenningen kjører med, sammen med hele kommandolinjen:

```powershell
Get-CimInstance Win32_Process -Filter "Name LIKE 'whisper-server%'" | ForEach-Object { $_.CommandLine }
```

Ledig minne på skjermkortet, som avgjør om modellen får plass:

```powershell
nvidia-smi --query-gpu=memory.total,memory.used,memory.free --format=csv
```

Er `llama-server` oppe, holder den 2–3 GB av dette:

```powershell
Get-Process llama-server-vulkan
```

Detaljert logg, med oppstartstider og utdata fra whisper-server, skrives til `%APPDATA%\open-whispr\logs\` når feilsøkingslogg er slått på under Settings. Loggen oppgir `startupTimeMs` for begge serverne og gjengir modellinnlastingen linje for linje, inkludert hvor mye som ble lagt på kortet.

Modellfilene ligger i `%USERPROFILE%\.cache\openwhispr\whisper-models\`. Mangler en fil der, melder programmet at modellen ikke er lastet ned, selv om den er valgt i innstillingene.

To forhold ser ut som feil, men er det ikke:

`--language auto` i kommandolinjen til whisper-server er ikke til hinder for norsk. Språket sendes som skjemafelt i hver enkelt forespørsel, og overstyrer standardverdien. Språket settes under Preferences → Transcription language.

En modellfil som er noe mindre enn oppgitt størrelse blir ikke avvist. `validateFileSize` opererer med ti prosent slingringsmonn.

## Avinstallering

Avinstallasjon fjerner `.cache/openwhispr/models`, men lar `.cache/openwhispr/whisper-models` stå. Whisper-modellene må slettes manuelt dersom diskplassen skal frigjøres.

## macOS

macOS-pakker kan ikke bygges fra Windows, så denne oppskrifta hjelper deg ikke direkte på en Mac.

Men modellen gjør det. [VoiceInk](https://github.com/Beingpax/VoiceInk) er et fritt dikteringsprogram for macOS som kan importere den samme `.bin`-filen fra Nasjonalbiblioteket. Last ned [ggml-model-q5_0.bin](https://huggingface.co/NbAiLab/nb-whisper-large/resolve/main/ggml-model-q5_0.bin) og legg den inn der. Da får du de norske modellene på Mac uten å bygge noe selv.

## In English

This patch adds the Norwegian National Library's NB-Whisper models to OpenWhispr 1.9.2, along with Nynorsk as a separate language option. The change is 28 lines across three files in the model and language registries.

The National Library publishes both model files openly at [NbAiLab/nb-whisper-large](https://huggingface.co/NbAiLab/nb-whisper-large): [ggml-model-q5_0.bin](https://huggingface.co/NbAiLab/nb-whisper-large/resolve/main/ggml-model-q5_0.bin) (1.08 GB) and [ggml-model.bin](https://huggingface.co/NbAiLab/nb-whisper-large/resolve/main/ggml-model.bin) (3.1 GB). Both are trained on Norwegian speech and handle Bokmål and Nynorsk alike.

Five points determine whether the result is usable:

1. Do not sign in to a self-built copy. Without an API URL, `policyRules.ts` fails closed and blocks local transcription entirely. Settings → Profile → Sign Out.
2. Budget your VRAM. Transcription and the cleanup model share one GPU. On an 8 GB card, the quantised model (1.6 GB) alongside Llama 3.2 3B (3.1 GB) and a browser will not fit, and transcription falls back to the CPU silently.
3. The cleanup model governs latency. `llama-server` takes 15.6 seconds to start and shuts down after five idle minutes to release VRAM. Most of that startup is Vulkan pipeline compilation: transcription loads a 3.1 GB model in 9.7 seconds on its CUDA build, while the cleanup model needs 15.6 seconds for 2.0 GB on Vulkan. Only a Vulkan build of `llama-server` ships with the application, so this cannot be configured away. The two run as separate processes on ports 8178 and 8221, which is why changing the Whisper model does not shorten the wait. Disable cleanup, choose a model around 1 GB, or point cleanup at an external OpenAI-compatible server — Ollama with `OLLAMA_KEEP_ALIVE=-1` uses CUDA and keeps the model resident, removing the startup cost entirely while staying local.
4. Note Recording keeps a model setting of its own, separate from dictation.
5. Disable automatic updates, or an official release will overwrite the local build.

Measured on an RTX 2070 (8 GB), transcribing with the quantised NB-Whisper throughout:

| | Time |
|---|---|
| Transcription, model resident on the GPU | 1.5–2 s |
| Bundled `llama-server` (Vulkan) cold start | 15.6 s |
| Ollama + Gemma 3 1B, first call | 94 s |
| Ollama + Gemma 3 1B, subsequent calls | 246–765 ms |

Moving cleanup to Ollama with `OLLAMA_KEEP_ALIVE=-1` removed the recurring 15.6-second startup and freed 3.9 GB of VRAM. Note that every tab under Language Models — Dictation Cleanup, Voice Assistant, Translation, Note Formatting, Chat — carries its own provider, endpoint and model. A single tab left on "Local" starts the bundled server and claims the memory regardless of whether you use that feature.

Cleanup was nevertheless switched off for dictation. Gemma 3 1B substituted plausible Norwegian words for ones that were never spoken — "kjeden" became "kilden" — and a substituted real word is far harder to catch when proofreading than a missing comma. NB-Whisper produces well-formed Norwegian on its own, and the errors it does make are visible ones.

## Lisens

Patchen er utgitt under MIT-lisensen.

OpenWhispr er MIT-lisensiert, © 2024 OpenWhispr Team. NB-Whisper er utviklet av [Nasjonalbiblioteket](https://huggingface.co/NbAiLab); vilkårene for modellvektene framgår av modellkortet.

Prosjektet er ikke tilknyttet OpenWhispr eller Nasjonalbiblioteket.
