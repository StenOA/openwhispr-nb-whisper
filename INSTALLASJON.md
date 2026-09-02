# Kom i gang

En praktisk oppskrift for å sette opp OpenWhispr med Nasjonalbibliotekets NB-Whisper, fra installasjon til første diktering. Regn med et kvarter.

Når du er ferdig, holder du to taster nede, snakker og slipper. Teksten står der markøren er. Ingenting annet skal startes, åpnes eller vekkes.

For den tekniske bakgrunnen — målinger, årsaker og kildekode — se [README](README.md).

---

## Først: skaff programmet

Selve installasjonsfila distribueres ikke herfra. Du har to veier:

**Har du fått en `OpenWhispr Setup`-fil av noen?** Da hopper du rett til DEL 1.

**Skal du bygge selv?** Følg [byggeoppskrifta i README](README.md#installasjon). Det er en `git clone`, en `git apply` og en `npm`-kommando, og du ender opp med den samme installasjonsfila.

---

## DEL 1 — Installasjon

1. Kjør `OpenWhispr Setup 1.9.2.exe`

2. Windows melder **«Windows beskyttet PC-en din»**. Trykk **Mer info**, deretter **Kjør likevel**.

   Dette er normalt. Fila er ikke kodesignert, fordi signering krever OpenWhispr sin egen Azure-konto. Det sier ingenting om innholdet i fila.

3. La installasjonen gå ferdig. Appen starter av seg selv.

---

## DEL 2 — Seks innstillinger

Ta dem i rekkefølge. Alle seks betyr noe.

### 1. Ikke logg inn

Når appen spør, velg **On your device**.

Trykk aldri «Sign in to use Cloud». Et bygg fra kildekoden har ingen adresse til OpenWhisprs sky-API, og innlogging låser da hele appen med meldingen «No setup option is available — contact your administrator».

Skulle det skje: **Settings → Profile → Sign Out**.

### 2. Last ned den norske modellen

Trykk **Download models**. Provider skal stå på **OpenAI** — det er Whisper. NVIDIA er Parakeet, som ikke kan norsk.

Lista viser fire modeller om gangen, og rullefeltet er skjult. Rull med musehjulet; de norske ligger nederst.

Velg **NB-Whisper Large (norsk, kvantisert)**, 1031 MB.

Ikke ta «full presisjon» på 3,1 GB. Kvalitetsforskjellen er marginal, mens minnebruken på skjermkortet er mer enn dobbelt så høy. Den kvantiserte er rett valg på alle maskiner.

### 3. Velg språk

**Settings → Preferences → Transcription language**

Velg **Norwegian** for bokmål, eller **Norwegian Nynorsk**.

La den ikke stå på «auto». Da gjetter modellen språk for hvert opptak, og korte opptak blir lett tolket som engelsk.

Bokmål og nynorsk ligger i samme modellfil. Målform byttes her, ikke ved å laste ned noe nytt.

### 4. Slå av tekstoppryddingen

**Settings → Language Models → fanen «Dictation Cleanup»** → slå av **Enable text cleanup**

### 5. Slå av stemmeassistenten

Samme sted, fanen **Voice Assistant** → slå av **Enable voice assistant**

Punkt 4 og 5 er de viktigste i hele oppskrifta. Begrunnelsen står under [Hvorfor oppryddingen skal være av](#hvorfor-oppryddingen-skal-være-av).

### 6. Slå av automatiske oppdateringer

**Settings → Preferences → Notifications → App updates** av.

Ellers kan en offisiell oppdatering overskrive bygget, og de norske modellene forsvinner fra modellista.

---

## DEL 3 — Slik bruker du den

> Hold nede **Ctrl + Windows-tasten** · snakk · slipp

Teksten kommer der markøren står, med en gang du slipper. Normal ventetid er halvannet til to sekunder.

Det spiller ingen rolle hvilket program du står i. Appen ligger i systemkurven, starter ved innlogging og laster den norske modellen inn med det samme, så den første dikteringen etter oppstart er like rask som resten.

---

## DEL 4 — Hvis noe ikke virker

**Det tar 15 sekunder før teksten kommer.**
En språkmodell er slått på et sted. Settings → Language Models har fem faner — Dictation Cleanup, Voice Assistant, Translation, Note Formatting og Chat — og hver har sitt eget modellvalg. Står bare én av dem på «Local», startes den store modellen likevel, og du betaler 15 sekunders oppstart.

**Det tar mange sekunder, hver gang.**
Modellen fikk ikke plass på skjermkortet og kjører på prosessoren i stedet. Lukk nettleservinduer og andre program. På en maskin uten kraftig skjermkort er dette normalt; da kan du velge en mindre modell under Settings → Speech-to-Text, men den blir merkbart dårligere på norsk.

**Melding om «Whisper model turbo not downloaded».**
Notatopptak har eget modellvalg: Settings → Speech-to-Text → fanen «Note Recording». Sett den til NB-modellen. Dikteringen er ikke berørt av denne meldingen.

**Ingenting skjer når jeg holder tastene.**
Sjekk at ikke et annet program har tatt mikrofonen. Andre dikteringsprogram, møteprogram og Teams er vanlige syndere.

**Teksten blir engelsk.**
Språket står på «auto». Se punkt 3.

---

## Hvorfor oppryddingen skal være av

Tekstoppryddingen sender den ferdige teksten gjennom en ekstra språkmodell som skal fjerne fyllord og rette tegnsetting. Det høres nyttig ut. To grunner til å la det være.

### Den gjør det tregt

Språkmodellen bruker rundt 15 sekunder på å starte, og stenger seg selv etter fem minutter uten bruk for å frigjøre minne. Dikterer du, tar en pause og dikterer igjen, betaler du oppstarten på nytt.

Det er dette som føles som treg diktering, selv om selve talegjenkjenningen er ferdig på under to sekunder. Feilsøkingen som ledet fram til denne oppskrifta gikk først i den fella: stadig mindre Whisper-modeller ble prøvd, helt ned til 141 MB, mot en flaskehals som lå et helt annet sted.

### Den finner på ord

Dette er den alvorlige grunnen.

En liten språkmodell gjetter. Under testing ble «hele kjeden virker» til «hele kilden virker». Feilen er vanskelig å oppdage, nettopp fordi resultatet er et ekte norsk ord som passer inn i setningen.

Et manglende komma ser du med en gang. Et ombyttet ord gjør du ikke — særlig ikke i tekst du selv nettopp har formulert, der du leser det du mente å si.

Skal teksten publiseres eller sendes videre, er det feil pris å betale for litt penere tegnsetting. NB-Whisper skriver velformet norsk på egen hånd, med stor forbokstav og punktum.

---

## Valgfritt: opprydding som ikke er treg

Hopp over dette med mindre du bevisst vil ha tekstopprydding likevel.

De 15 sekundene lar seg løse: et eget program kan holde språkmodellen lastet hele tiden, i stedet for å starte den på nytt.

```powershell
winget install Ollama.Ollama
ollama pull gemma3:1b
setx OLLAMA_KEEP_ALIVE -1
```

Deretter, i hver fane under Settings → Language Models du vil bruke: velg **Self-Hosted**, skriv adressen `http://127.0.0.1:11434/v1`, trykk **Apply & Refresh**, og klikk på `gemma3:1b` i lista slik at den får merket **Active**.

Målt resultat: svartid ned fra 15 600 til 765 millisekunder. Alt kjører fortsatt lokalt — ingen tekst forlater maskinen.

Men les avsnittet over en gang til først. Farten var aldri hovedinnvendingen mot tekstoppryddingen. Ordbyttene var det, og de forsvinner ikke av at modellen svarer raskere.

---

Modellene er laget av [Nasjonalbiblioteket](https://huggingface.co/NbAiLab/nb-whisper-large). Prosjektet er ikke tilknyttet OpenWhispr eller Nasjonalbiblioteket.
