# NB-Whisper for OpenWhispr

En liten patch som legger Nasjonalbibliotekets **NB-Whisper**-modeller inn i
[OpenWhispr](https://github.com/OpenWhispr/openwhispr) 1.9.2, og legger til **nynorsk**
som eget språkvalg.

Den offisielle appen har bare `Norwegian`, som i praksis gir bokmål. NB-Whisper er trent på
norsk tale av Nasjonalbiblioteket og gir merkbart bedre resultat på norsk enn de generelle
Whisper-modellene — særlig på navn, stedsnavn og fagord. Den håndterer både bokmål og nynorsk.

28 linjer i tre filer. Ingenting annet er endret.

---

## Modellene

Begge filene ligger åpent ute hos Nasjonalbiblioteket på Hugging Face, i repoet
[**NbAiLab/nb-whisper-large**](https://huggingface.co/NbAiLab/nb-whisper-large):

| Modell i appen | Fil hos Nasjonalbiblioteket | Størrelse |
|---|---|---|
| NB-Whisper Large (norsk, kvantisert) | [`ggml-model-q5_0.bin`](https://huggingface.co/NbAiLab/nb-whisper-large/resolve/main/ggml-model-q5_0.bin) | 1,08 GB |
| NB-Whisper Large (norsk, full presisjon) | [`ggml-model.bin`](https://huggingface.co/NbAiLab/nb-whisper-large/resolve/main/ggml-model.bin) | 3,1 GB |

Appen laster dem ned automatisk første gang du velger dem, og lagrer dem lokalt som
`ggml-nb-large-q5_0.bin` og `ggml-nb-large-full.bin`. Se modellkortet for lisens og vilkår
for modellvektene.

**Bruk den kvantiserte til daglig.** Kvalitetsforskjellen er marginal, og full presisjon
trenger ~3,6 GB VRAM mot ~1,6 GB — se VRAM-avsnittet nedenfor.

## Ta patchen i bruk

```bash
git clone https://github.com/OpenWhispr/openwhispr.git
cd openwhispr
git apply /sti/til/nb-whisper.patch
npm install
npm run download:whisper-cpp
```

`npm run dev` henter **ikke** whisper.cpp av seg selv — den kommandoen må kjøres for seg.

Bygg på Windows:

```bash
npx electron-builder --config electron-builder.unsigned-win.json
```

Bruk den usignerte varianten. Standardoppskriften prøver å signere med OpenWhisprs eget
Azure-sertifikat, som du ikke har tilgang til.

---

## Hva som skal til for at det faktisk virker

Dette er fellene som kostet mest tid. Alle er reelle, alle er målt.

### 1. Ikke logg inn

Et selvbygget OpenWhispr har ingen `VITE_OPENWHISPR_API_URL`. Logger du inn, spør appen et
API den ikke når om «workspace policy». `policyRules.ts` er *fail closed* og nekter da alt —
**også lokal transkribering**, som ikke trenger nettet i det hele tatt.

Symptom: «No setup option is available — contact your administrator» i onboardingen, eller at
NB-modellen ikke lar seg velge.

Fiks: **Settings → Profile → Sign Out.**

### 2. Regn på VRAM-en

Whisper og språkmodellen for tekstopprydding deler samme skjermkort. Måling på et
**RTX 2070 med 8 GB**:

| Det som ligger på kortet | VRAM |
|---|---|
| NB-Whisper q5_0 (vekter + KV-cache + compute-buffere) | ~1,6 GB |
| NB-Whisper full presisjon | ~3,6 GB |
| Llama 3.2 3B (standard opprydningsmodell) | 3,1 GB |
| Nettleser, Electron-apper, skrivebord | 2–3 GB |

Får ikke whisper plass på GPU-en, faller den ned på CPU. En large-modell på CPU tar
**mange sekunder** per ytring i stedet for under to.

Med q5_0 på GPU: **1,5–2 sekunder** fra du slipper tasten til teksten står der.

### 3. Tekstoppryddingen er den store tidstyven

`llama-server` starter på **15,6 sekunder** og **stenger seg selv etter fem minutter** uten
bruk for å frigjøre VRAM (`IDLE_TIMEOUT_MS` i `src/helpers/llamaServer.js`). Dikterer du,
venter seks minutter og dikterer igjen, betaler du hele oppstarten på nytt.

To utveier:

- **Slå det av:** Settings → AI Models → «Enable text cleanup». Whisper alene gir god nok
  norsk tekst.
- **Bruk en liten modell:** Gemma 3 1B (0,81 GB) er trent på 140+ språk og håndterer norsk.
  Llama 3.2 støtter offisielt bare åtte språk, og norsk er ikke ett av dem. Qwen3-familien
  kan la resonneringen lekke inn i teksten.

### 4. Notatopptak har sitt eget modellvalg

**Settings → Speech-to-Text → fanen «Note Recording»** velger modell uavhengig av dikteringen.
Bytter du dikteringsmodell, følger ikke denne med — og da kommer
`Whisper model "turbo" not downloaded` hver gang funksjonen brukes.

Sett begge fanene til NB-modellen.

### 5. Slå av automatiske oppdateringer

**Settings → Preferences → Notifications → «App updates»** av. En offisiell oppdatering
overskriver bygget ditt, og NB-modellene forsvinner fra lista.

---

## Ting som *ikke* er problemet

Verdt å vite, så du ikke jakter på feil ting:

- **`--language auto` på kommandolinja til whisper-server.** Appen sender språket som
  skjemafelt i hver enkelt forespørsel (`src/helpers/whisperServer.js`), og det overstyrer
  standardverdien. Språket settes i Preferences → Transcription language.
- **En modellfil som er litt mindre enn oppgitt.** `validateFileSize` har 10 % slingringsmonn.

---

## Avinstallering

Avinstallering sletter `.cache/openwhispr/models` (språkmodellene), men **ikke**
`.cache/openwhispr/whisper-models`. Whisper-modellene blir liggende, og må slettes for hånd
om du vil ha plassen tilbake.

## macOS

Windows kan ikke bygge macOS-pakker. Trenger du nynorsk på Mac, kan
[VoiceInk](https://github.com/Beingpax/VoiceInk) importere den samme `.bin`-fila.

---

## In English

This patch adds the Norwegian National Library's **NB-Whisper** models to OpenWhispr 1.9.2,
plus Nynorsk as a separate language option. 28 lines across three files.

The two model files are published openly by the National Library at
[NbAiLab/nb-whisper-large](https://huggingface.co/NbAiLab/nb-whisper-large):
[`ggml-model-q5_0.bin`](https://huggingface.co/NbAiLab/nb-whisper-large/resolve/main/ggml-model-q5_0.bin)
(1.08 GB) and
[`ggml-model.bin`](https://huggingface.co/NbAiLab/nb-whisper-large/resolve/main/ggml-model.bin)
(3.1 GB). They are trained on Norwegian speech and clearly outperform the general Whisper
models on Norwegian, and they handle both written standards, Bokmål and Nynorsk.

The practical points:

1. **Do not sign in** to a self-built copy — `policyRules.ts` fails closed without an API URL
   and blocks local transcription entirely. Settings → Profile → Sign Out.
2. **Watch your VRAM.** Whisper and the cleanup LLM share one GPU. On an 8 GB card, the
   quantised model (~1.6 GB) plus Llama 3.2 3B (3.1 GB) plus a browser will not fit, and
   whisper silently falls back to CPU — many seconds instead of under two.
3. **The cleanup model is the latency.** `llama-server` takes 15.6 s to start and shuts itself
   down after five idle minutes. Turn cleanup off, or pick a ~1 GB model.
4. **Note Recording has its own model setting**, separate from dictation.
5. **Turn off app updates**, or an official release will overwrite your build.

---

## Lisens og kreditering

Patchen er MIT, som OpenWhispr selv.

OpenWhispr — MIT, © 2024 OpenWhispr Team.
NB-Whisper — laget av [Nasjonalbiblioteket (NbAiLab)](https://huggingface.co/NbAiLab).
Se modellkortet deres for vilkårene som gjelder modellvektene.

Dette repoet er ikke tilknyttet verken OpenWhispr eller Nasjonalbiblioteket.
