# Elmer Audio — Sound Card Configuration for Digital Modes

Help the user identify audio devices, set correct levels, configure audio routing for digital modes, and diagnose audio problems.

## STARTUP

Read `~/.config/elmer/config.yaml`. If missing, direct to `/elmer-setup`.

Ask what they're trying to solve:
1. Find and identify my audio devices
2. Set audio levels for digital modes
3. Audio not working with [specific app]
4. Configure audio for a specific interface (Digirig, SignaLink, etc.)
5. Set up audio for both HF and VHF/UHF rigs simultaneously

---

## SECTION A — IDENTIFY AUDIO DEVICES

```bash
# All playback devices
aplay -l

# All capture devices
arecord -l

# PulseAudio/PipeWire sink and source names (needed for some apps)
pactl list short sinks
pactl list short sources
```

Show output and help user identify:
- Their rig's built-in USB audio (if applicable)
- Their interface audio (Digirig, SignaLink, etc.)
- Their system audio (to be kept SEPARATE from radio audio)

Common device names to look for:
- Yaesu FT-710: appears as "USB Audio CODEC" or similar
- Digirig: appears as "USB Audio Device" or "C-Media USB"
- SignaLink: appears as "USB Audio CODEC"

If multiple devices have similar names, use `aplay -L` for the full list with card numbers.

Update config with confirmed audio device names.

---

## SECTION B — AUDIO LEVELS

### Check current levels

```bash
# PulseAudio/PipeWire:
pavucontrol &   # GUI mixer — recommend opening this

# ALSA:
alsamixer       # Terminal mixer — use F6 to select sound card
```

### Recommended starting levels for digital modes

**Transmit (radio input from computer):**
- Start at 30–50% output level
- Too high = audio distortion, wide signal, splatter
- Too low = weak signal, may not trigger AGC properly
- Watch the rig's ALC meter — ALC should barely move on voice peaks

**Receive (computer input from radio):**
- Set receive level so digital mode software shows signal peaks at roughly 0 dB (not clipping)
- fldigi: waterfall should show signals clearly without solid white bands
- WSJT-X: aim for noise level around 30 dB on the band activity panel

### For Digirig specifically

Digirig Lite has no hardware level controls — all level control is via software. This is normal. Use:
```bash
alsamixer -c [card number for digirig]
```

---

## SECTION C — PIPEWIRE/PULSEAUDIO CONFIGURATION

### Prevent audio device from being grabbed by desktop

Some distros auto-connect USB audio to the desktop audio mixer, causing issues when ham radio apps try to use it exclusively.

For PipeWire — create a device rule to exclude radio audio from default routing:
```bash
# Check current PipeWire config location
ls ~/.config/pipewire/

# Create a rule to mark radio audio as non-default
mkdir -p ~/.config/pipewire/pipewire.conf.d/
```

Show user the rule to write, ask for confirmation, then write it.

For PulseAudio — set radio audio device to non-default:
```bash
# List current default sink
pactl info | grep "Default Sink"

# List all sinks with index
pactl list sinks short

# Set a non-radio device as default
pactl set-default-sink [system-audio-sink-name]
```

### Verify audio path end-to-end

Test receive audio (radio → computer):
```bash
# Record a few seconds from the radio device, then play it back
arecord -D [audio-device] -f cd -d 5 /tmp/radio-test.wav && aplay /tmp/radio-test.wav
```

Test transmit audio (computer → radio):
```bash
# Generate a test tone to the radio device
# WARNING: this will key the rig if PTT is not controlled — advise user to check PTT before running
speaker-test -D [audio-device] -t sine -f 1000 -l 1
```

---

## SECTION D — INTERFACE-SPECIFIC NOTES

### Digirig Lite
- Uses CP210x USB-to-serial for PTT (via RTS)
- PTT and audio are on separate USB interfaces (separate device nodes)
- Audio device: usually shows as "USB Audio Device"
- No onboard level controls — all levels are software
- Check Digirig documentation for jumper settings (some rigs need specific jumpers for VOX vs RTS PTT)

### SignaLink USB
- All level control via hardware knobs on the device (TX, RX, DLY)
- Shows as generic USB Audio
- DLY knob controls VOX delay — set to minimum for digital modes
- TX knob: start at 9 o'clock, adjust for ALC

### Rig with built-in USB audio (IC-7300, FT-710, etc.)
- Audio device and CAT are on the same USB cable
- Set rig's input/output level in rig menu (not just system audio)
- Check rig menu for USB AF output level and USB IF gain

---

## SECTION E — COMMON PROBLEMS

**No audio device visible:**
```bash
lsusb  # Is the device recognized at all?
dmesg | tail -20  # Any USB errors?
```

**Audio device visible but no signal:**
- Check physical connections (audio cable seated, correct port)
- Check rig menu — USB audio may need to be enabled in rig settings
- Verify monitor/receiver is set to USB audio in rig menu

**Transmit audio distorted:**
- Reduce system output level to radio
- Check ALC on rig — should barely move
- Check for sample rate mismatch (rig may require 48000 Hz)

**Sample rate issues:**
```bash
# Check what sample rates the device supports
cat /proc/asound/card[N]/stream0 2>/dev/null
```

**Audio works but PTT doesn't:**
→ Go to `/elmer-rig` for PTT configuration
