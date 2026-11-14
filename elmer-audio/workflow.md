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

### Persistent, human-readable device names

USB sound codecs enumerate in unpredictable order — two identical "USB Audio Device"
entries that swap card number on reboot or replug, and PipeWire node names that carry a
USB-path suffix (`...-00.2...`) which changes with the port. Pin them by USB **product
ID** (`lsusb` → `idVendor:idProduct`) so the name is stable and says which rig it is.

Placeholders below: `[VID]`/`[PID]` come from `lsusb` for each audio device.
`[NAME]` is a short id with no spaces, taken from the config's `rigs.*.audio_device`
(e.g. `MyRig`). `[name]` is the same id in lowercase, and `[Label]` is a friendly
display name. Write one rule/match per device.

**ALSA card id** — `/etc/udev/rules.d/85-ham-audio-names.rules`:
```
ACTION=="add|change", SUBSYSTEM=="sound", KERNEL=="card*", ATTRS{idVendor}=="[VID]", ATTRS{idProduct}=="[PID]", ATTR{id}="[NAME]"
```
Reload: `sudo udevadm control --reload && sudo udevadm trigger`. Now `hw:[NAME],0`
is stable regardless of card number.
Note: two devices that share a VID:PID (common with C-Media codecs) also need
`ATTRS{serial}` or a USB path to tell them apart.

**PipeWire node name + friendly label** — `~/.config/wireplumber/wireplumber.conf.d/51-ham-radio-names.conf`:
```
monitor.alsa.rules = [
  {
    matches = [ { device.product.id = "0x[PID]" } ]
    actions = { update-props = {
      device.name        = "alsa_card.[name]"
      device.description = "[Label]"
    } }
  }
]
```
Pinning `device.name` drops the USB-path suffix, so the source becomes
`alsa_input.[name].<profile>` (e.g. `.mono-fallback`; port-independent) and the app
dropdown shows "[Label]".
Restart: `systemctl --user restart wireplumber`, or log out/in on sessions without
`systemd --user`.

After renaming, **re-select the device once in each app** — its saved config still
points at the old node name.

Handy: a small script that prints which node is which rig (map by USB PID):
```bash
pactl list short sources | grep -E '[name1]|[name2]'
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

**Digital-mode app records from the built-in mic instead of the radio:**

Qt apps (WSJT-X, JS8Call, JTDX) save the input by device *name*. If that device is not
present the instant the app starts, Qt silently falls back to the **system default
source** — usually the laptop mic — and stays there for the whole session, even after the
radio's device appears. The app's audio dropdown still shows the correct device selected;
it just isn't the one being used.

Why the device is missing at launch:
- A rig's built-in USB codec only enumerates while the rig is **powered on**. Starting
  the app before the radio is up = fallback.
- The app started (autostart, or resume from suspend) before WirePlumber finished
  renaming the card.

Confirm what it's *actually* using — don't trust the dropdown:
```bash
pactl list source-outputs | grep -E 'application.name|Source:'   # which source the app opened
pw-link -l | grep -i js8call                                     # what it's linked to
cat ~/.local/state/wireplumber/default-nodes                     # the configured default source
```

Fixes, simplest first:
1. Power the radio on, confirm the node exists (`wpctl status` shows it under Sources),
   *then* launch the app. If already stuck, re-pick the input in the app's audio
   settings — no restart needed.
2. Launch the app from a wrapper that waits for the radio:
   ```bash
   until pactl list short sources | grep -q 'alsa_input.[name]'; do sleep 1; done
   exec /path/to/js8call.appimage "$@"
   ```
3. Pin the app's capture stream to the radio node so launch order stops mattering —
   `~/.config/wireplumber/wireplumber.conf.d/50-js8call-capture.conf`:
   ```
   node.rules = [
     {
       matches = [ { application.name = "JS8Call", media.class = "Stream/Input/Audio" } ]
       actions = { update-props = { target.object = "[radio source name from pactl]" } }
     }
   ]
   ```
   Restart wireplumber and the app. Verify with the `pw-link` check above before trusting it.

**Audio works but PTT doesn't:**
→ Go to `/elmer-rig` for PTT configuration
