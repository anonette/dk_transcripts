# Lithopia — audio restoration & voice-clone dub

The story of what was done to `media/lithopia.flv`, in order.

## 1. The source

`lithopia.flv` — a ~15.5 minute screen-recorded demo by Denisa Kera walking through Lithopia, a Hyperledger Composer smart-contract prototype for a satellite-verified land registry ("Lithopians" own "Lithopia Place" plots; a red "flag color" pixel from Sentinel-2 satellite imagery triggers ownership-transfer transactions). The recording's mic chain has a noticeable metallic/harsh quality.

## 2. Cleaning the original audio

`ffmpeg` chain: `highpass=f=80` (remove rumble) → `afftdn` (spectral denoise) → `loudnorm` (loudness normalize). Conservative settings on the second pass specifically (see step 6) — light denoise (3–8dB), no heavy compression, notch filters only applied where hum was actually measured present (it wasn't, here).

## 3. Transcription

[`victor-upmeet/whisperx`](https://replicate.com/victor-upmeet/whisperx) on Replicate, word-aligned, no diarization (single speaker). Raw output committed as-is in `transcripts/lithopia.json` / `.srt` (commit `b67cead`) before any manual correction, specifically so AI-driven corrections would be reviewable as a diff rather than silently baked in.

## 4. Transcript correction

Two reviewed correction passes on top of the raw transcript (commits `e43c845`, `d41fb04`):
- Fixed a recurring pattern: "Lithopia" and its demonym "Lithopian" got mangled by ASR many different ways across the video — Utopia/Utopian, Dystopian, Lusopian, Letopian, Listopian, Sofian, Ethiopia, Loopy, "Tokyo Place". All confirmed against the real, consistent Hyperledger Composer terminology used throughout (participants, assets, transactions, Business Network Archive).
- Other confirmed fixes: `FlexColor` → `FlagColor`, "flat collar/column" → "flag color", "Tatra Ledger Composer" → "Hyperledger Composer" (a real product), "The costume was" → "The outcome was".
- Only segment `text` fields were ever edited — the word-level `words[]` alignment arrays are left exactly as whisperx produced them, since patching per-word timestamps without real re-alignment would be fabrication, not correction.
- Anything too speculative to fix confidently (names, sentences with content that looks like it dropped out of the ASR entirely) was deliberately left alone and flagged rather than guessed. See `transcripts/REVIEW_NOTES.md` for what's still open.

## 5. Voice cloning — first attempt, cloned from the video itself

Cloned Denisa's voice using an ~80s clean segment of `lithopia.flv`'s own (cleaned) audio as reference, via two engines:
- [`minimax/voice-cloning`](https://replicate.com/minimax/voice-cloning) → [`minimax/speech-2.8-turbo`](https://replicate.com/minimax/speech-2.8-turbo)
- [`qwen/qwen3-tts`](https://replicate.com/qwen/qwen3-tts) (`mode: voice_clone`)

All 137 transcript segments were resynthesized through MiniMax, each time-stretched (`ffmpeg atempo`) to fit its original `[start, end]` window, and assembled into a second audio track — muxed into `media/lithopia_dubbed.mkv` alongside the cleaned original and soft subtitles.

## 6. A better source recording

The video's own audio still carries its metallic quality even where "clean," so no amount of resynthesis from it fully escapes that coloration in the reference. Switched to a much better source: an earlier, cleanly-recorded talk — *"Alchemical Emblems & Electronic Circuits"*, Denisa Kera at FIBER Festival 2017 — as the voice-cloning reference instead. Transcribed the same way (whisperx), picked an ~80s clean window, and re-ran both cloning engines from that source for comparison.

## 7. Benchmarking

Built up `media/benchmark/` as a set of same-text, same-excerpt comparisons: original vs. MiniMax-from-video vs. Qwen3-from-video vs. a conservative traditional-ffmpeg-only restoration (no cloning) vs. Qwen3 with a "warmer" `style_instruction` vs. MiniMax-from-FIBER vs. Qwen3-from-FIBER. Along the way, found and worked around a Qwen3-TTS quirk: passing `reference_text` alongside `reference_audio` causes it to audibly echo the reference transcript before continuing into the new text — dropping `reference_text` (keeping only `reference_audio`) fixed it, at some presumed cost to alignment accuracy.

## 8. Final dub

Full-video Qwen3 clone, voice cloned from the **FIBER 2017 recording**, text from the **corrected transcript** (commit `d41fb04`). Same per-segment generate → time-stretch-to-original-timing → concatenate pipeline as step 5. Output: `media/lithopia_qwen_fiber.mkv` — Qwen3 clone as the default/first audio track, original cleaned audio as the second, soft subtitles, all audio tracks downmixed to mono.

## Tools used

- **ffmpeg** — audio cleaning, time-stretching, muxing
- **[victor-upmeet/whisperx](https://replicate.com/victor-upmeet/whisperx)** — transcription + word alignment
- **[minimax/voice-cloning](https://replicate.com/minimax/voice-cloning) + [minimax/speech-2.8-turbo](https://replicate.com/minimax/speech-2.8-turbo)** — voice clone (first attempt)
- **[qwen/qwen3-tts](https://replicate.com/qwen/qwen3-tts)** — voice clone (final dub)
- **git** — tracking transcript corrections as reviewable, reversible diffs rather than silent overwrites
- **Tailscale Serve** — private tailnet hosting for review/comparison, not committed to this repo

## Repo layout

```
transcripts/           tracked — the whole point of version-controlling this
  lithopia.json         whisperx output, corrected (see git log for the diff trail)
  lithopia.srt           derived subtitles
  REVIEW_NOTES.md         open items not yet confirmed
media/                  gitignored — original + all generated audio/video, large binaries
README.md              this file
```
