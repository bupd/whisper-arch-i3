# Voice Input Plan

## Goal

Add offline voice typing to this Arch + i3 + X11 setup:

1. Press `Alt+Shift+m` to start recording.
2. Speak in English.
3. Press `Alt+Shift+m` again to stop.
4. Whisper transcribes the recording.
5. The transcript is typed into the currently focused input field.
6. `dunstify` shows clear, good-looking status notifications throughout.

## Local Context

- OS: Arch Linux.
- Session: X11.
- Desktop/window manager: i3.
- `$mod` in i3: `Mod1`, so `$mod+Shift+m` is `Alt+Shift+m`.
- Audio stack: PipeWire exposing PulseAudio compatibility.
- Already available locally:
  - `i3`
  - `xdotool`
  - `xbindkeys`
  - `ffmpeg`
  - `parec`
  - `pw-cat`
  - `python3`
  - `pip`
  - `notify-send`
  - `dunstify`
  - `rofi`
- `dunstify` server: `dunst 1.13.1`.
- `dunstify` capabilities include:
  - `actions`
  - `body`
  - `body-markup`
  - `icon-static`
  - `synchronous`
  - `x-dunst-stack-tag`

## Chosen Approach

Use OpenAI Whisper through the official Arch package, plus a small toggle script.

Do not use `nerd-dictation` for this setup. It uses Vosk, and the quality is not good enough for this goal.

Do not start with `whisper.cpp`. It is attractive long-term, but the AUR package has recent build instability in comments. Keep it as a later optimization path if needed.

## Dependencies

Install Whisper from Arch packages instead of `pip`:

```bash
sudo pacman -S python-openai-whisper
```

Reason:

- The system Python is `3.14`.
- Upstream OpenAI Whisper docs mention compatibility around Python `3.8-3.11`.
- The Arch package is maintained for Arch's current Python stack and avoids fragile `pip` installs.

Required tools:

```bash
sudo pacman -S python-openai-whisper ffmpeg xdotool libnotify
```

Most of these are already installed locally except `python-openai-whisper`.

## Model Choice

Default model: `base.en`.

Reason:

- English-only use case.
- Better quality than `tiny.en`.
- More practical latency than `small.en` on CPU.
- `.en` models are more appropriate than multilingual models for English-only dictation.

Allow override with an environment variable:

```bash
VOICE_TYPE_MODEL=small.en scripts/voice-type-toggle
VOICE_TYPE_MODEL=tiny.en scripts/voice-type-toggle
```

Tuning guidance:

- Use `base.en` first.
- If quality is poor, try `small.en`.
- If latency is too high, try `tiny.en`.

## Files To Add Or Change

### Add Script

Path:

```text
scripts/voice-type-toggle
```

Responsibilities:

- Toggle recording on/off.
- Store runtime state under `${XDG_RUNTIME_DIR}/voice-type`.
- Record from the default Pulse/PipeWire mic using `ffmpeg`.
- Stop `ffmpeg` cleanly so the WAV file finalizes.
- Run Whisper transcription.
- Clean transcript whitespace.
- Type transcript into the currently focused input using `xdotool`.
- Show polished `dunstify` notifications.
- Handle stale PID files and common errors.

### Update i3 Config

Path:

```text
.config/i3/config
```

Add binding:

```i3
bindsym $mod+Shift+m exec --no-startup-id ~/dotfiles/scripts/voice-type-toggle
```

No existing conflict was found for `$mod+Shift+m`.

## Runtime Directory Layout

Use:

```text
${XDG_RUNTIME_DIR}/voice-type
```

Files:

```text
recording.pid
recording.wav
transcript.txt
whisper-output/
```

Why `${XDG_RUNTIME_DIR}`:

- Per-user runtime storage.
- Cleared after logout/reboot.
- Avoids leaving voice recordings in the dotfiles repo or home directory.

Fallback if `XDG_RUNTIME_DIR` is unset:

```text
/tmp/voice-type-$USER
```

## Toggle Behavior

### Start Recording

When no active recording PID exists:

1. Create runtime directory.
2. Remove previous temp files.
3. Start `ffmpeg` in the background:

```bash
ffmpeg -y -hide_banner -loglevel error \
  -f pulse -i default \
  -ar 16000 -ac 1 \
  "$runtime_dir/recording.wav"
```

4. Save PID to `recording.pid`.
5. Show recording notification.

### Stop And Transcribe

When active recording PID exists:

1. Send `SIGINT` to `ffmpeg`.
2. Wait briefly for WAV finalization.
3. Remove PID file.
4. Verify recording file exists and is non-empty.
5. Run Whisper:

```bash
whisper "$runtime_dir/recording.wav" \
  --model "${VOICE_TYPE_MODEL:-base.en}" \
  --language en \
  --task transcribe \
  --fp16 False \
  --output_format txt \
  --output_dir "$runtime_dir/whisper-output"
```

6. Read the generated `.txt` file.
7. Normalize whitespace:
   - Trim leading/trailing whitespace.
   - Collapse newlines to spaces.
   - Preserve punctuation produced by Whisper.
8. If transcript is empty, notify and do not type.
9. Type into focused input:

```bash
xdotool type --clearmodifiers --delay 0 --file "$runtime_dir/transcript.txt"
```

10. Show success notification.

## Notification Design

Use `dunstify` directly instead of plain `notify-send`.

Shared options:

```bash
dunstify \
  -a "Voice Type" \
  -h string:x-dunst-stack-tag:voice-type
```

Use `x-dunst-stack-tag` so status messages replace each other instead of piling up.

### Recording Started

```bash
dunstify -a "Voice Type" -u normal -t 2500 -i audio-input-microphone \
  -h string:x-dunst-stack-tag:voice-type \
  -h int:value:20 \
  "Voice typing" "Recording... press <b>Alt+Shift+M</b> to stop"
```

### Stopping Recording

```bash
dunstify -a "Voice Type" -u low -t 1200 -i media-playback-stop \
  -h string:x-dunst-stack-tag:voice-type \
  -h int:value:45 \
  "Voice typing" "Recording saved"
```

### Transcribing

```bash
dunstify -a "Voice Type" -u normal -t 0 -i accessories-dictionary \
  -h string:x-dunst-stack-tag:voice-type \
  -h int:value:70 \
  "Voice typing" "Transcribing with Whisper <b>${VOICE_TYPE_MODEL:-base.en}</b>..."
```

### Inserted Successfully

```bash
dunstify -a "Voice Type" -u low -t 2500 -i emblem-default \
  -h string:x-dunst-stack-tag:voice-type \
  -h int:value:100 \
  "Voice typing" "Inserted transcript"
```

### Empty Transcript

```bash
dunstify -a "Voice Type" -u normal -t 3500 -i dialog-information \
  -h string:x-dunst-stack-tag:voice-type \
  "Voice typing" "No speech detected"
```

### Error

```bash
dunstify -a "Voice Type" -u critical -t 6000 -i dialog-error \
  -h string:x-dunst-stack-tag:voice-type \
  "Voice typing failed" "$reason"
```

## Optional Sound Feedback

Available locally:

- `paplay`
- `canberra-gtk-play`

Recommendation: keep sound off by default. Add it later if visual notifications are not enough.

Potential future env flag:

```bash
VOICE_TYPE_SOUND=1
```

## Robustness Requirements

The script should handle:

- Missing `whisper` command.
- Missing `ffmpeg`.
- Missing `xdotool`.
- Missing `dunstify`, with fallback to `notify-send` if available.
- Stale `recording.pid`.
- Empty recording file.
- Empty transcript.
- Whisper failure.
- User pressing the hotkey repeatedly.

Suggested stale PID logic:

```bash
if [ -f "$pid_file" ] && ! kill -0 "$(cat "$pid_file")" 2>/dev/null; then
  rm -f "$pid_file"
fi
```

## First-Run Warmup

The first Whisper run downloads the model to:

```text
~/.cache/whisper
```

After implementation, do one manual warmup with a short recording before relying on the hotkey.

Expected first-run behavior:

- Slower than normal because the model downloads.
- Later runs should be much faster.

## Verification Plan

### 1. Check Commands

```bash
command -v whisper ffmpeg xdotool dunstify
```

### 2. Check Audio Source

```bash
ffmpeg -hide_banner -sources pulse
```

Expected default source observed locally:

```text
alsa_input.pci-0000_00_1f.3.analog-stereo
```

### 3. Manual Recording Test

```bash
ffmpeg -y -f pulse -i default -t 3 -ar 16000 -ac 1 /tmp/voice-test.wav
```

### 4. Manual Whisper Test

```bash
whisper /tmp/voice-test.wav --model base.en --language en --task transcribe --fp16 False --output_format txt --output_dir /tmp
```

### 5. Manual Typing Test

Focus a text input and run:

```bash
printf 'voice typing test' >/tmp/voice-type-text.txt
xdotool type --clearmodifiers --delay 0 --file /tmp/voice-type-text.txt
```

### 6. Script Test

```bash
scripts/voice-type-toggle
```

Speak, then run again:

```bash
scripts/voice-type-toggle
```

Expected:

- Recording notification appears.
- Transcribing notification appears.
- Text is inserted into focused input.

### 7. i3 Hotkey Test

Reload i3:

```bash
i3-msg reload
```

Then test `Alt+Shift+m` in:

- Browser input.
- Terminal prompt.
- Obsidian.
- Slack or another Electron app.

## Future Improvements

- Add a `VOICE_TYPE_PROMPT_FIXUP=1` mode that passes the transcript through a local formatter or LLM before typing.
- Add clipboard fallback if typing into certain apps is unreliable.
- Add `small.en` as default if quality matters more than latency.
- Add `whisper.cpp` backend if the AUR package stabilizes or if building manually is acceptable.
- Add a rofi model switcher for `tiny.en`, `base.en`, and `small.en`.
- Add automatic sentence cleanup commands such as:
  - Say `new line` to insert newline.
  - Say `period` to insert `.`.
  - Say `comma` to insert `,`.

## Final Decision Summary

- Hotkey: `Alt+Shift+m`.
- Language: English only.
- Mode: toggle start/stop.
- Engine: OpenAI Whisper.
- Package source: official Arch `python-openai-whisper`.
- Default model: `base.en`.
- Audio capture: `ffmpeg` from Pulse/PipeWire default source.
- Text injection: `xdotool` for X11.
- Notifications: polished `dunstify` with stack tag, icons, markup, urgency, and progress hints.
