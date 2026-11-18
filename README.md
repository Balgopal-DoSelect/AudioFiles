# AudioFiles

Test audio for the SpeechSense partner submission API. Each file triggers one audio case. Use its URL as the `audio_url` of an `ADO` (spoken) response.

> The links work only after the files are pushed to `main` and while this repository is public. The API downloads the file from the URL, so it must open in a browser without logging in.

## How partner audio is checked

Every `audio_url` is downloaded, checked and converted to mono 16-bit WAV before evaluation. The checks run in this order, and the first one that fails sets the question's `status_code`:

1. **Download.** A network or HTTP error gives `download_failed`. More than 10 MB gives `file_too_large`. 0 bytes gives `empty_file`.
2. **Read** (ffprobe). If the file can't be read, it gives `unreadable_audio`.
3. **Length.** Under 2 s gives `empty_file`. Over 60 s (Read & Repeat) or over 180 s (Free Speech) gives `invalid_duration`.
4. **Convert** (ffmpeg). If conversion fails, it gives `conversion_failed`.
5. **Volume.** If the loudest point is at or below −50 dB, it gives `silent_audio`.

If any check fails, the question is not evaluated. Each of its competencies is stored as `score: 0, grade: "A0"`, and the question keeps its weight. If every question in the attempt fails, the overall `cefr_score` is `A0`, and `cefr_analysis` lists the reasons.

## Cases

| # | Case | File | Expected `status_code` |
|---|---|---|---|
| 1 | Valid recording | [Record (online-voice-recorder.com).wav](#1-valid-recording) | `evaluated` |
| 2 | Silent recording | [empty-audio.wav](#2-silent-recording) | `silent_audio` |
| 3 | Corrupt file | [corrupt-audio.wav](#3-corrupt-file) | `unreadable_audio` |
| 4 | Empty file (0 bytes) | [empty-size-audio.wav](#4-empty-file-0-bytes) | `empty_file` |
| 5 | Audio conversion failed | [conversion_failed.wav](#5-audio-conversion-failed) | `conversion_failed` |
| 6 | Upload error (URL not found) | none, a broken URL | `download_failed` |
| 7 | File too large (over 10 MB) | [sample-speech-5m.wav](#7-file-too-large-over-10-mb) | `file_too_large` |
| 8 | Recording too long | [long-190s.wav](#8-recording-too-long) | `invalid_duration` |

---

### 1. Valid recording

**File:** [Record (online-voice-recorder.com).wav](https://raw.githubusercontent.com/Balgopal-DoSelect/AudioFiles/main/Record%20%28online-voice-recorder.com%29.wav)

```text
https://raw.githubusercontent.com/Balgopal-DoSelect/AudioFiles/main/Record%20%28online-voice-recorder.com%29.wav
```

**Why it is valid:** a real voice recording. It is 8.4 s long, uncompressed WAV at 44.1 kHz, and its loudest point is −4.9 dB. It passes every check, is converted to 16 kHz and is sent for evaluation.

**Expect:** `status_code: "evaluated"` and real scores and grades for the question. It is within the limits for both Read & Repeat and Free Speech.

---

### 2. Silent recording

**File:** [empty-audio.wav](https://raw.githubusercontent.com/Balgopal-DoSelect/AudioFiles/main/empty-audio.wav)

```text
https://raw.githubusercontent.com/Balgopal-DoSelect/AudioFiles/main/empty-audio.wav
```

**Why it fails:** a readable 10.2 s WAV (16 kHz) with no speech. Its loudest point is −84.3 dB, below the −50 dB silence limit. The name says "empty", but the file has content and is silent.

**Expect:** `status_code: "silent_audio"`; the question is scored 0 and graded A0; reason text "silent recording".

---

### 3. Corrupt file

**File:** [corrupt-audio.wav](https://raw.githubusercontent.com/Balgopal-DoSelect/AudioFiles/main/corrupt-audio.wav)

```text
https://raw.githubusercontent.com/Balgopal-DoSelect/AudioFiles/main/corrupt-audio.wav
```

**Why it fails:** the file has a `.wav` name, but its content is not valid audio. ffprobe stops with "Invalid data found when processing input".

**Expect:** `status_code: "unreadable_audio"`; the question is scored 0 and graded A0; reason text "corrupt audio file".

---

### 4. Empty file (0 bytes)

**File:** [empty-size-audio.wav](https://raw.githubusercontent.com/Balgopal-DoSelect/AudioFiles/main/empty-size-audio.wav)

```text
https://raw.githubusercontent.com/Balgopal-DoSelect/AudioFiles/main/empty-size-audio.wav
```

**Why it fails:** the download succeeds but returns 0 bytes, so the file is never read.

**Expect:** `status_code: "empty_file"`; the question is scored 0 and graded A0; reason text "empty file".

---

### 5. Audio conversion failed

**File:** [conversion_failed.wav](https://raw.githubusercontent.com/Balgopal-DoSelect/AudioFiles/main/conversion_failed.wav)

```text
https://raw.githubusercontent.com/Balgopal-DoSelect/AudioFiles/main/conversion_failed.wav
```

**Why it fails:** a 5 s, 16 kHz tone whose header names a codec that doesn't exist (format tag `0x1234`). ffprobe can still read the length and sample rate from the header, so it passes the read and length checks. ffmpeg has no decoder for that codec, so the conversion stops with "Decoder (codec none) not found".

**Expect:** `status_code: "conversion_failed"`; the question is scored 0 and graded A0; reason text "audio conversion failed".

How it was made:

```bash
ffmpeg -y -f lavfi -i "sine=frequency=440:duration=5" -ac 1 -ar 16000 -c:a pcm_s16le good.wav
python3 -c "
d = bytearray(open('good.wav', 'rb').read())
i = d.index(b'fmt ') + 8
d[i:i+2] = (0x1234).to_bytes(2, 'little')
open('conversion_failed.wav', 'wb').write(d)
"
```

---

### 6. Upload error (URL not found)

**File:** none. [empty-audiowyriuwq.wav](https://github.com/Balgopal-DoSelect/AudioFiles/raw/refs/heads/main/empty-audiowyriuwq.wav) is not in this repository, so the download returns HTTP 404.

```text
https://github.com/Balgopal-DoSelect/AudioFiles/raw/refs/heads/main/empty-audiowyriuwq.wav
```

**Expect:** `status_code: "download_failed"`; the question is scored 0 and graded A0; reason text "upload error".

---

### 7. File too large (over 10 MB)

**File:** [sample-speech-5m.wav](https://raw.githubusercontent.com/Balgopal-DoSelect/AudioFiles/main/sample-speech-5m.wav)

```text
https://raw.githubusercontent.com/Balgopal-DoSelect/AudioFiles/main/sample-speech-5m.wav
```

**Why it fails:** 12.62 MB, over the 10 MB limit. It's a clean 5 min speech recording (mono, 22.05 kHz). Size is checked during download, before anything else, so this file stops there. Its length is also over the limit, but that check is never reached.

**Expect:** `status_code: "file_too_large"`; the question is scored 0 and graded A0; reason text "audio file too large".

---

### 8. Recording too long

**File:** [long-190s.wav](https://raw.githubusercontent.com/Balgopal-DoSelect/AudioFiles/main/long-190s.wav)

```text
https://raw.githubusercontent.com/Balgopal-DoSelect/AudioFiles/main/long-190s.wav
```

**Why it fails:** the first 190 s of `sample-speech-5m.wav`. At 8.38 MB it is under the 10 MB limit, so it is downloaded and read. It is then rejected for its length: over 60 s for Read & Repeat and over 180 s for Free Speech. Works on either question type.

**Expect:** `status_code: "invalid_duration"`; the question is scored 0 and graded A0; reason text "recording too long".

How it was made:

```bash
ffmpeg -i sample-speech-5m.wav -t 190 -c copy long-190s.wav
```

---

## Where to check the result

- **Webhook / report API:** `solutions[].status_code` for each question.
- **Solution data:** `extra_data.evaluation_flags` holds the reason code. Every competency in `extra_data.cefr_competencies` is `{"score": 0, "grade": "A0", "transcription": "", "reason": "The file could not be read."}`.
- **Overall result:** `language_assessment.unscored` gives `{count, total, reasons}`.
  - **Every question fails** (for example, every response uses one of cases 2–8): `cefr_score` is `A0` and `cefr_analysis.analysis` reads "The candidate's English level could not be assessed … Reasons: …".
  - **Some questions are valid:** the overall grade is calculated as usual.

## Not covered by a file yet

| Case | Expected `status_code` | How to make a file |
|---|---|---|
| Shorter than 2 s | `empty_file` | `ffmpeg -i "Record (online-voice-recorder.com).wav" -t 1 short.wav` |
| Low sample rate (still scored) | `low_sample_rate` | `ffmpeg -i "Record (online-voice-recorder.com).wav" -ar 8000 low-8khz.wav` |
