# NB-Whisper for OpenWhispr

Norske tale-til-tekst-modeller fra Nasjonalbiblioteket i [OpenWhispr](https://github.com/OpenWhispr/openwhispr) 1.9.2, med nynorsk som eget språkvalg.

OpenWhispr leveres med Whisper-modellene fra OpenAI. De forstår norsk, men er trent på engelsk i hovedsak, og treffer ujevnt på navn, stedsnavn og fagord. NB-Whisper er trent på norsk tale av Nasjonalbiblioteket og gir vesentlig bedre resultat.

Bokmål og nynorsk ligger i samme modellfil. Det trengs ingen egen nynorskmodell; målformen styres av språkvalget i programmet, og patchen legger inn nynorsk som eget valg ved siden av «Norwegian». Den offisielle appen har bare det siste, som i praksis gir bokmål.

Endringen er 28 linjer fordelt på tre filer i modell- og språkregisteret. Ingen annen del av programmet er berørt.

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

`llama-server` bruker 15,6 sekunder på å starte, og avslutter seg selv etter fem minutter uten bruk for å frigjøre minne på kortet. Hver diktering som følger etter en lengre pause betaler oppstarten på nytt. Dette er uavhengig av hvilken Whisper-modell som er valgt, og forveksles lett med treg talegjenkjenning.

To alternativer:

* Slå funksjonen av under Settings → AI Models → «Enable text cleanup». NB-Whisper leverer velformet norsk tekst uten etterbehandling.
* Velg en mindre modell. Gemma 3 1B (0,81 GB) dekker over 140 språk, norsk inkludert. Llama 3.2 oppgir offisiell støtte for åtte språk, og norsk er ikke blant dem. Qwen3-familien kan la resonneringstekst følge med i utdataene.

### Notatopptak har eget modellvalg

Settings → Speech-to-Text har separate faner for «Dictation» og «Note Recording», med hvert sitt modellvalg. Notatopptak følger ikke med når dikteringsmodellen endres, og melder `Whisper model "turbo" not downloaded` dersom den peker på en modell som ikke er lastet ned. Begge faner bør settes til NB-modellen.

### Slå av automatiske oppdateringer

Settings → Preferences → Notifications → «App updates» bør slås av. En offisiell oppdatering overskriver det lokale bygget, og NB-modellene forsvinner fra modellisten.

## Feilsøking

To forhold ser ut som feil, men er det ikke:

`--language auto` i kommandolinjen til whisper-server er ikke til hinder for norsk. Språket sendes som skjemafelt i hver enkelt forespørsel, og overstyrer standardverdien. Språket settes under Preferences → Transcription language.

En modellfil som er noe mindre enn oppgitt størrelse blir ikke avvist. `validateFileSize` opererer med ti prosent slingringsmonn.

## Avinstallering

Avinstallasjon fjerner `.cache/openwhispr/models`, men lar `.cache/openwhispr/whisper-models` stå. Whisper-modellene må slettes manuelt dersom diskplassen skal frigjøres.

## macOS

macOS-pakker kan ikke bygges fra Windows. [VoiceInk](https://github.com/Beingpax/VoiceInk) kan importere den samme `.bin`-filen og er et alternativ for norskspråklige Mac-brukere.

## In English

This patch adds the Norwegian National Library's NB-Whisper models to OpenWhispr 1.9.2, along with Nynorsk as a separate language option. The change is 28 lines across three files in the model and language registries.

The National Library publishes both model files openly at [NbAiLab/nb-whisper-large](https://huggingface.co/NbAiLab/nb-whisper-large): [ggml-model-q5_0.bin](https://huggingface.co/NbAiLab/nb-whisper-large/resolve/main/ggml-model-q5_0.bin) (1.08 GB) and [ggml-model.bin](https://huggingface.co/NbAiLab/nb-whisper-large/resolve/main/ggml-model.bin) (3.1 GB). Both are trained on Norwegian speech and handle Bokmål and Nynorsk alike.

Five points determine whether the result is usable:

1. Do not sign in to a self-built copy. Without an API URL, `policyRules.ts` fails closed and blocks local transcription entirely. Settings → Profile → Sign Out.
2. Budget your VRAM. Transcription and the cleanup model share one GPU. On an 8 GB card, the quantised model (1.6 GB) alongside Llama 3.2 3B (3.1 GB) and a browser will not fit, and transcription falls back to the CPU silently.
3. The cleanup model governs latency. `llama-server` takes 15.6 seconds to start and shuts down after five idle minutes. Disable it, or choose a model around 1 GB.
4. Note Recording keeps a model setting of its own, separate from dictation.
5. Disable automatic updates, or an official release will overwrite the local build.

## Lisens

Patchen er utgitt under MIT-lisensen.

OpenWhispr er MIT-lisensiert, © 2024 OpenWhispr Team. NB-Whisper er utviklet av [Nasjonalbiblioteket](https://huggingface.co/NbAiLab); vilkårene for modellvektene framgår av modellkortet.

Prosjektet er ikke tilknyttet OpenWhispr eller Nasjonalbiblioteket.
