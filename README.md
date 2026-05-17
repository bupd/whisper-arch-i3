# whisper-arch-i3

Offline voice typing for Arch Linux + i3 + X11 using OpenAI Whisper.

Press `Alt+Shift+m` to start recording. Press it again to stop, transcribe, and type the text into the focused input field.

## Requirements

Install the Arch packages:

```bash
sudo pacman -S python-openai-whisper ffmpeg xdotool libnotify
```

Recommended notification daemon:

```bash
sudo pacman -S dunst
```

## i3 Binding

Add this to your i3 config:

```i3
bindsym $mod+Shift+m exec --no-startup-id /home/bupdlap/dotfiles/whisper-arch-i3/voice-type-toggle
```

Then reload i3:

```bash
i3-msg reload
```

## Usage

Start recording:

```bash
/home/bupdlap/dotfiles/whisper-arch-i3/voice-type-toggle
```

Stop recording and transcribe:

```bash
/home/bupdlap/dotfiles/whisper-arch-i3/voice-type-toggle
```

The default model is `base.en`. Override it when needed:

```bash
VOICE_TYPE_MODEL=tiny.en /home/bupdlap/dotfiles/whisper-arch-i3/voice-type-toggle
VOICE_TYPE_MODEL=small.en /home/bupdlap/dotfiles/whisper-arch-i3/voice-type-toggle
```

## Runtime Files

Recordings and transcripts are temporary and live under:

```text
${XDG_RUNTIME_DIR}/voice-type
```

If `XDG_RUNTIME_DIR` is missing, the script uses:

```text
/tmp/voice-type-$USER
```

## Verification

Check dependencies:

```bash
command -v whisper ffmpeg xdotool dunstify
```

Check the microphone source:

```bash
ffmpeg -hide_banner -sources pulse
```

Warm up the default model:

```bash
ffmpeg -y -hide_banner -loglevel error -f pulse -i default -t 2 -ar 16000 -ac 1 /tmp/voice-type-smoke.wav
whisper /tmp/voice-type-smoke.wav --model base.en --language en --task transcribe --fp16 False --output_format txt --output_dir /tmp
```

## Files

`voice-type-toggle` is the main toggle script.

`i3-snippet.conf` contains the i3 binding.

`PLAN.md` is the original implementation plan and deeper design notes.
