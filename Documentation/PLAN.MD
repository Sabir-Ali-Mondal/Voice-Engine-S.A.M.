# Bengali Teaching-Voice — Complete Plan (Careful, Practical, Resumable)

**Purpose:** End-to-end, practical plan to collect teaching-style Bengali+English-mixed voice data, build a lightweight TTS model, and *deploy a native, offline voice engine that browsers can use via the OS speech API (SAPI / OneCore)* — with strong resumability, safe backups, and a robust session/merge workflow.

**Scope:**

* Data collection web app (offline, resumable, mergeable sessions)
* Dataset schema, prompts, and phoneme guidance
* Lightweight model fine-tuning workflow on **Google Colab Free** (resume-safe)
* Packaging options: run model locally for testing; create a **SAPI-compatible** engine (Windows) so browsers use it via `speechSynthesis` (no API server)
* Backup, checkpoints, and disaster recovery policies

---

## Table of contents

1. Summary goals & constraints
2. Final dataset spec (sizes, split, emotion mix)
3. Master dataset generation prompts (for 1.5k and 4k datasets)
4. `dataset.json` schema and example entries
5. Recorder website v2 — features & implementation notes (resume, backups, merge)
6. Recording best practices & file naming rules
7. Colab fine-tuning workflow (resume-safe) — commands & checkpoints
8. Model choice & expected sizes (Glow-TTS-mini / VITS-small + HiFi-GAN small)
9. Export paths: testing locally, packaging as native engine (SAPI/eSpeak/MaryTTS)
10. Packaging a Windows SAPI voice engine (high-level steps)
11. Validation, test, and iteration cycles (MOS, unit tests)
12. Backup & versioning policy (autosave rules, naming conventions)
13. Quick checklist & timeline
14. Helpful snippets (shell, Python, JS)
15. Developed by S.A.M.

---

## 1. Summary goals & constraints

* Use **Google Colab Free** for model fine-tuning → limited GPU time / intermittent sessions. Design all training to be resumable and checkpoint-first (save to Drive frequently).
* No external API servers for runtime. Final voice must be usable **offline** and recognized by browsers as a system voice (SAPI/OneCore or equivalent).
* Start with a **1–2 hour** dataset (≈1,000–1,500 lines) to validate pipeline; then scale to 4–5 hours (3k–4k lines) for a "good" model.
* Provide a browser-based recorder app (single HTML file) that: records, autosaves to IndexedDB, exports ZIP (audio + `dataset.json`), merges multiple session ZIPs into one canonical dataset, prevents duplicate IDs, and resumes from last saved state.

---

## 2. Final dataset specification (recommended for initial Colab runs)

* **Pilot (fast test):** 500 lines ≈ 30–60 minutes → for pipeline QA
* **Step-1 (Colab target):** 1,500 lines ≈ 1–2 hours → good for recognizably your voice after fine-tuning
* **Step-2 (Good model):** 3,000–4,000 lines ≈ 4–5 hours → production quality

**Emotion / style mix (TEACHING FOCUS):**

* `teaching` / explanatory: 35%
* `neutral` / instructional: 25%
* `motivational`: 15%
* `scolding` / corrective: 10%
* `friendly`: 10%
* `surprised` / expressive: 5%

**Language mix:** 80–85% Bengali, 15–20% Bengali with natural English code-switching.

**Sentence types (overall):**

* statement: 45%
* question: 15%
* exclamation: 10%
* command: 10%
* long (10–20 words): 10%
* numbers/dates: 3%
* english_mixed: 5–7%

---

## 3. Master prompts to generate dataset JSON

Use these prompts in your preferred LLM (ChatGPT/Gemini/local LLM):

**A. 1,500-line prompt (teaching-focused starter)**

```
Generate a JSON array of 1500 objects for a Bengali teaching-style TTS dataset.
Each object must include:
- "id": unique integer starting from 1
- "text": a natural Bengali sentence (5–25 words). Include short English words naturally in ~15% of entries (e.g., "Chrome", "function", "assignment").
- "emotion": one of ["teaching","neutral","motivational","scolding","friendly","surprised"]
- "type": one of ["explanation","command","question","correction","story","demonstration","english_mixed","number"]
- "duration_est": integer 3–8 representing expected seconds.

Distribution (approx):
- emotions: teaching 35%, neutral 25%, motivational 15%, scolding 10%, friendly 10%, surprised 5%
- types: statement & explanation 45%, question 15%, exclamation 10%, command 10%, long 10%, numbers 3%, english_mixed 7%

Return valid JSON array only.
```

**B. 4,000-line full dataset prompt** — same as above but `4000` entries and slightly more english_mixed (8–12%) if desired.

**C. Phoneme-coverage checklist (200 lines)** — use to ensure all vowels, consonants, conjuncts, common diphones appear.

---

## 4. `dataset.json` schema & examples

Canonical record:

```json
{
  "id": 142,
  "text": "আজ আমরা binary search এর ধারণা শিখবো।",
  "emotion": "teaching",
  "type": "explanation",
  "duration_est": 5,
  "filename": "",        // populated by recorder when saved
  "recorded": false,
  "notes": ""
}
```

**Small example:**

```json
[
  {"id":1,"text":"আজ আমরা Computer memory নিয়ে আলোচনা করবো।","emotion":"teaching","type":"explanation","duration_est":5,"filename":"","recorded":false},
  {"id":2,"text":"একটি উদাহরণ দেখে বুঝে নাও।","emotion":"teaching","type":"demonstration","duration_est":5,"filename":"","recorded":false}
]
```

---

## 5. Recorder website v2 — features & implementation notes

**Core goals:** offline-first, session-safe, auto-backup, merge-friendly, duplicate-ID safe, easy ZIP export.

### Required features

* Load `dataset.json` or generate from prompt file.
* Display current entry with id/text/emotion/type + short reading instructions.
* Record & preview audio using `MediaRecorder`. Convert to WAV (PCM16) client-side.
* On Accept: save audio blob to IndexedDB and update `dataset.json` entry:

  * `filename = sam_<id>_<YYYYMMDDTHHMMSS>.wav`
  * `recorded = true`
* Auto-backup:

  * After each accepted clip, update IndexedDB and trigger a background JSON save to browser download folder (or prompt to save).
  * Offer "Auto-save to Google Drive" optional via picker API (user consent).
* Resume:

  * On page load, detect last `dataset.json` in IndexedDB → ask "Resume from id X?"
  * If imported ZIP, extract and validate `dataset.json`.
* Merge:

  * Upload multiple ZIPs → app extracts `dataset.json` from each → merge by `id`. If duplicate IDs found, present conflict dialog with options: Keep Old / Replace with New / Keep Both (reassign id).
* Export:

  * Exports ZIP with `/audio/` and `dataset.json` and `session_info.json` (session metadata). Use JSZip.
* Safety:

  * Confirm before deleting any entry or overwriting existing filename.
* UX:

  * Keyboard shortcuts: Space = record/stop, Enter = accept.

### Implementation notes

* Use `IndexedDB` for audio blobs (large storage); fallback to FileSystem API if available.
* Use `WebAudio` to compute RMS and show waveform.
* Use `lib/data-wav-encoder.js` or write simple WAV header + PCM16 writer in JS.

---

## 6. Recording best practices & file naming

**Audio specs:**

* Format: WAV PCM16, Mono
* Sample rate: 22050 Hz (or 24000 Hz) — keep consistent
* Bit depth: 16-bit
* File length: 3–8 sec per clip

**Mic & room:**

* Use a single microphone and consistent mic position
* Use a quiet, low-reverb room or basic foam / blanket shield
* Use pop filter for plosives

**Volume & trimming:**

* Keep mouth-to-mic distance stable (20–30 cm)
* Trim leading/trailing silence to 0.05–0.2s
* Apply fade-in/out 5–10ms to avoid clicks

**File naming rule (unique & resumable):**

```
<speaker>_<id>_<YYYYMMDDTHHMMSS>.wav
example: sam_0142_20251031T141002.wav
```

**Session info file (`session_info.json`):**

```json
{
  "session_id": "2025-10-31T14-10-02",
  "speaker": "SabirAli",
  "lines_recorded": 142,
  "duration_sec": 846,
  "notes": ""
}
```

---

## 7. Colab fine-tuning workflow (resume-safe)

**High-level:**

* Always mount Google Drive and save checkpoints & logs to Drive.
* Save a `training_state.json` with last step, paths, and current epoch.
* Use `torch.save` frequently (every N steps) to Drive to avoid loss.

**Notebook outline**

1. Setup & install

```bash
!git clone https://github.com/coqui-ai/TTS.git
%cd TTS
!pip install -q -r requirements.txt
!pip install -q -e .
```

2. Mount Drive

```python
from google.colab import drive
drive.mount('/content/drive')
CHECKPOINT_DIR = "/content/drive/MyDrive/tts_checkpoints/"
```

3. Unzip dataset

```bash
!unzip -q /content/drive/MyDrive/dataset.zip -d /content/dataset
```

4. Preprocess

```bash
# Use TTS toolkit preprocess or ffmpeg for resample
!ffmpeg -i input.wav -ar 22050 -ac 1 output.wav
```

5. Training (example pseudo)

```python
# training loop skeleton
for step, batch in enumerate(dataloader):
    loss = model.train_step(batch)
    if step % save_interval == 0:
        ckpt_name = f"ckpt_step_{step}_{int(time.time())}.pth"
        torch.save({'model': model.state_dict(), 'optim': optimizer.state_dict(), 'step': step}, CHECKPOINT_DIR + ckpt_name)
        with open(CHECKPOINT_DIR + "training_state.json", "w") as f:
            json.dump({'last_step': step, 'ckpt': ckpt_name}, f)
```

6. Resume

```python
# Load latest checkpoint
state = torch.load("/content/drive/MyDrive/tts_checkpoints/ckpt_step_8000_1700000000.pth")
model.load_state_dict(state['model'])
optimizer.load_state_dict(state['optim'])
start_step = state.get('step', 0)
```

**Configuration recommendations for Colab Free**

* Batch size: 8
* Use mixed precision if available (`torch.cuda.amp`)
* Save checkpoints every ~1500–3000 steps (balance between Drive writes and safety)
* Keep each checkpoint size < 500 MB (depends on model)

---

## 8. Model choice & expected sizes (practical)

**Recommended stack for Colab Free:**

* Acoustic: `Glow-TTS-mini` or `FastSpeech2-small` (compact)
* Vocoder: `HiFi-GAN (v1 small)` or `MelGAN` (fast, smaller)

**Estimated sizes**

* Pretrained Glow-TTS-mini: ~60–150 MB
* HiFi-GAN small: ~30–120 MB
* Fine-tuned checkpoints: 200–500 MB (depending on saving full optimizer state)
* Final compressed deployable model: 60–200 MB (after pruning/strip optimizer)

**Inference speed on Colab Free GPU (T4):** near real-time; on CPU (Ryzen 7) much slower but usable for testing.

---

## 9. Export paths: local testing → SAPI packaging

**A. Local testing (no system integration):**

* Use Coqui CLI to synthesize and produce `.wav` files:

  ```bash
  tts --text "আমি আজ পাঠ দেবো" --model_path ./models/my_acoustic.pth --vocoder_path ./models/my_vocoder.pth --out_path sample.wav
  ```
* Build a tiny GUI wrapper (Python + Tkinter / Electron) for local playback.

**B. Packaging as SAPI-compatible voice (Windows) — two approaches:**

1. **Phoneme-based SAPI engine** (lightweight, real-time but robotic)

   * Use **eSpeak NG** or **MaryTTS voice pack** model format.
   * Create a voice package where recorded phoneme WAVs are referenced; register using the SAPI registry entries or vendor tools.
   * Browser will detect it automatically (`speechSynthesis.getVoices()`).

2. **Native wrapper using local TTS executable (recommended practical approach)**

   * Package a small native app/service (Windows service or local HTTP server) that exposes a system-level SAPI interface or uses Windows Speech SDK to register a voice that forwards requests to local TTS engine.
   * This is more work but makes the acoustic model run locally and be available to all apps.
   * Option: create a lightweight COM object (SAPI) that wraps calls to your PyTorch/ONNX model (via an embedded runtime, e.g., ONNX Runtime C++), then register as a SAPI provider. This requires C++/C# dev.

**Notes:** Building a full SAPI driver requires Windows SDK dev & some native code. For a simpler path, eSpeak NG voice pack may be enough as v1 — later you can create a native COM wrapper for high-quality NN model outputs.

---

## 10. Packaging a Windows SAPI voice engine — high-level steps

> This section is intentionally high-level: building a SAPI driver is a native project but achievable.

1. **Decide engine type:** rule-based (eSpeak NG / MaryTTS) OR model-backed (ONNX runtime service + SAPI wrapper).
2. **Prepare voice assets:** phoneme recordings or NN model files. For SAPI phoneme engines, arrange voice data per vendor format.
3. **Develop SAPI wrapper:** using Visual Studio (C++ or C#) + Microsoft Speech API SDK. Implement `ISpVoice` or register an appropriate engine provider.
4. **Installation & registry:** build installer (MSI) that registers the engine and voice entries in the registry (`HKLM\SOFTWARE\Microsoft\Speech\Voices\Tokens\...`).
5. **Test:** open Chrome/Edge and run:

```js
speechSynthesis.getVoices().forEach(v => console.log(v.name, v.lang));
```

6. **Iterate:** if voice is detected — map wave generation calls to your local model (or to pre-recorded phoneme playback).

**Simpler alternative:** If native SAPI dev is too heavy, use eSpeak NG voice packaging (easier) as v1, and later replace with the model-backed engine.

---

## 11. Validation & testing plan

* **Unit tests:** check every metadata entry has a matching audio file; filenames follow pattern.
* **Automatic audio QA:** compute RMS and duration, ensure no file shorter than 2s or longer than expected.
* **Perceptual tests:** small panel (5–10 listeners) to give MOS for clarity & naturalness on 20 sample lines (neutral + each emotion).
* **Pre-deployment checks:** run `speechSynthesis` detection and ensure platform voice selection works (Chrome on Windows detects SAPI voice).
* **Edge tests:** mixed English/Bengali lines, numbers, dates, URLs.

---

## 12. Backup & versioning policy (must-follow)

* **After each recording:** update `dataset.json` and save to IndexedDB + trigger a local JSON download (incremental file `dataset_backup_<timestamp>.json`).
* **After each session:** export full ZIP `dataset_<session>_<timestamp>.zip` (audio + dataset.json + session_info.json) and upload to Google Drive.
* **Training checkpoints:** save every 1500–3000 steps to Google Drive and keep the last 10 checkpoints. Also keep lightweight `latest.pth` that always points to the latest good/save.
* **Naming convention:** use ISO timestamps `YYYYMMDDTHHMMSS` and zero-padded IDs.
* **Retention:** keep 30-day rolling backups for sessions; keep final dataset zip per major version.

---

## 13. Quick checklist & timeline (practical)

**Phase 0 — Setup (1 day)**

* Prepare prompts + generate 500-line starter JSON
* Create recorder HTML v2 and test with 50 lines

**Phase 1 — Pilot recording (1–3 days)**

* Record 500 lines (30–60 min) in sessions of 100 lines
* Merge & export ZIP → upload to Drive

**Phase 2 — Colab fine-tune (1–3 days)**

* Run Colab notebook, fine-tune Glow-TTS-mini, save checkpoints every ~1500 steps
* Test sample synthesis (generate 20 lines) → evaluate

**Phase 3 — Scale & polish (1–2 weeks)**

* Record remaining lines (to reach 3–4k lines)
* Re-run training / continue fine-tune from checkpoint(s)
* Final evaluation and iterate

**Phase 4 — Packaging (1–2 weeks, depends on dev skill)**

* Option A: create eSpeak NG voice pack as v1 (fast)
* Option B: implement SAPI wrapper for model-backed engine (advanced)

---

## 14. Helpful snippets

**A. JS: simple WAV PCM16 encoder (outline)**

```js
function encodeWAV(samples, sampleRate) {
  const buffer = new ArrayBuffer(44 + samples.length * 2);
  const view = new DataView(buffer);
  // write WAV header ...
  // PCM16 data
  let offset = 44;
  for (let i = 0; i < samples.length; i++, offset += 2) {
    let s = Math.max(-1, Math.min(1, samples[i]));
    view.setInt16(offset, s < 0 ? s * 0x8000 : s * 0x7FFF, true);
  }
  return new Blob([view], { type: 'audio/wav' });
}
```

**B. Shell: resample all WAVs to 22050**

```bash
mkdir resampled
for f in audio/*.wav; do
  ffmpeg -y -i "$f" -ar 22050 -ac 1 "resampled/$(basename "$f")"
done
```

**C. Python: save training checkpoint to Drive**

```python
ckpt = {'model': model.state_dict(), 'optim': optimizer.state_dict(), 'step': step}
ckpt_path = f"/content/drive/MyDrive/tts_checkpoints/ckpt_step_{step}_{int(time.time())}.pth"
torch.save(ckpt, ckpt_path)
with open("/content/drive/MyDrive/tts_checkpoints/training_state.json","w") as f:
    json.dump({"last_step": step, "ckpt": ckpt_path}, f)
```

---

## 15. Closing line

This plan is intentionally **practical**, **resumable**, and **modular** — it lets you:

* record in many short sessions and merge reliably,
* train safely on Google Colab Free by saving frequent checkpoints,
* choose a fast deployment path (eSpeak NG voice pack) as v1, and upgrade to a full SAPI model later, and
* ensure that browsers can use your voice **natively** without any API server.

---

**Developed by S.A.M.**
