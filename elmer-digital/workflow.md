# Elmer Digital — HF Digital Modes

Configure and operate HF digital mode software: fldigi, WSJT-X, JS8Call, and JS8 Spotter.

## STARTUP

Read `~/.config/elmer/config.yaml`. If missing, direct to `/elmer-setup`.

Check prerequisites — rig control and audio should be working first. If config shows rig or audio not yet configured, suggest running `/elmer-rig` and `/elmer-audio` first.

Ask what they want to work on:
1. fldigi (PSK31, RTTY, Olivia, MT63, and many others)
2. WSJT-X (FT8, FT4, JT65, WSPR, etc.)
3. JS8Call
4. JS8 Spotter
5. Troubleshoot an existing setup

---

## SECTION A — FLDIGI

### Install

Check config `software.fldigi.installed`. If false:
```bash
# Debian/Ubuntu/MX Linux:
sudo apt install fldigi

# Arch:
sudo pacman -S fldigi

# Fedora:
sudo dnf install fldigi

# Or download from W1HKJ.com (often newer version)
```

### Configure fldigi

fldigi config is at `~/.config/fldigi/` or `~/.fldigi/`.

Walk through Configure menu → key sections:

**Operator:**
- Callsign, name, QTH, locator (grid square from config)

**Soundcard → Devices:**
- Select the radio's audio device for both Capture and Playback
- Use values from config `rigs.hf.audio_device`
- Sample rate: try 48000 first; if issues try 8000 or 44100

**Soundcard → Right channel:**
- For rigs without PTT via CAT: can use right channel for PTT
- Usually leave disabled for rigs with CAT PTT

**Rig Control:**
Choose one method:

*Option 1 — flrig (recommended if flrig is running):*
- Configure → Rig Control → flrig
- Enable "Use flrig CAT PTT"

*Option 2 — rigctld:*
- Configure → Rig Control → RigCAT or Hamlib
- Set to rigctld localhost port 4532

*Option 3 — Direct Hamlib:*
- Configure → Rig Control → Hamlib
- Rig: select from dropdown
- Device: CAT port from config
- Baud rate from config

**PTT:**
- Via CAT (preferred): Configure → Rig Control → use PTT via CAT/RTS
- Via RTS/DTR: Configure → PTT → select port and method

### Waterfall tips
- Adjust the RX level until signals are visible but noise floor isn't solid white
- Click on a signal to tune to it
- Double-click to start decoding immediately

---

## SECTION B — WSJT-X

### Install

Check config `software.wsjt_x.installed`. If false:
```bash
# Debian/Ubuntu/MX Linux:
sudo apt install wsjtx

# Or download latest from physics.princeton.edu/pulsar/k1jt/wsjtx.html
# (apt version is often old — recommend downloading directly)
```

### Configure WSJT-X

Config at `~/.config/WSJT-X Inc/WSJT-X.ini`

**File → Settings → General:**
- Callsign, grid (from config)
- Check "Decode after EME delay" only if doing EME

**File → Settings → Radio:**
- Rig: select Hamlib NET rigctl (recommended) or specific rig
- If using Hamlib NET: set host localhost, port 4532 (requires rigctld to be running)
- If using flrig: select flrig
- PTT Method: CAT / RTS / DTR depending on setup

**File → Settings → Audio:**
- Input: radio receive audio device (from config)
- Output: radio transmit audio device
- Sample rate: use default (48000)

**File → Settings → Reporting:**
- Enable PSK Reporter for spotting (optional but recommended)
- Log QSOs to ADIF file

### Start a FT8 session
1. Set rig to 14.074 MHz USB (or band of choice)
2. Set mode to FT8 in WSJT-X
3. Check time sync: `timedatectl status` — clock must be within ±1 second for FT8
   ```bash
   # If clock is off:
   sudo timedatectl set-ntp true
   # or: sudo ntpdate pool.ntp.org
   ```
4. Press Enable Tx, pick a TX slot, double-click a station to call

### Common WSJT-X issues
- **No decodes:** audio level issue (see `/elmer-audio`) or clock sync
- **TX not working:** PTT method wrong, check rig is in USB mode not LSB
- **Frequency off:** rig not on exact frequency — verify CAT connection

---

## SECTION C — JS8CALL

### Install

Check config `software.js8call.installed`. If false:
```bash
# JS8Call is often not in repos — check js8call.com for Linux download
# Download .deb or AppImage

# For .deb:
sudo dpkg -i js8call_*.deb
sudo apt --fix-broken install  # if dependency issues
```

### Configure JS8Call

Very similar to WSJT-X — shares the same audio/rig framework.

**File → Settings → General:**
- Callsign, grid

**File → Settings → Radio:**
- Same options as WSJT-X — Hamlib NET rigctl or flrig recommended

**File → Settings → Audio:**
- Same as WSJT-X

**JS8Call-specific settings:**
- Speed: Normal (default) for most HF work; Slow for weak signal; Fast for strong paths; Turbo for local
- Heartbeat: enable to automatically send @HB at configured interval
- Inbox: enable for message store-and-forward

### JS8Call tips
- Primary frequency: 14.078 MHz (HF day), 7.078 MHz (NVIS/night)
- JS8Call is a keyboard QSO mode — type CQ or replies in the message box
- Can be used offline as a messaging system — messages stored for later retrieval
- **Config lives in `~/.config/JS8Call.ini`** — one section per profile under
  `[MultiSettings]`, active profile is `CurrentName`. Audio keys `SoundInId` /
  `SoundInName` are in `[Configuration]`. Delete stale profiles (e.g. `Default - Copy`)
  so you can't accidentally load one with wrong device/CAT settings.
- **If JS8Call decodes nothing and seems to use the wrong microphone:** it fell back to
  the system default input because the radio's USB audio device wasn't present when
  JS8Call started (radio powered on after JS8Call, or launched too early). The audio
  dropdown still shows the right device — but check with
  `pactl list source-outputs | grep -E 'application.name|Source:'`. See
  `/elmer-audio` → Section E, "records from the built-in mic instead of the radio".

---

## SECTION D — JS8 SPOTTER

JS8 Spotter monitors JS8Call traffic and provides a propagation/activity dashboard.

### Install

```bash
# Check if installed
which js8spotter 2>/dev/null || find /opt /usr/local ~/Applications -name "js8spotter*" 2>/dev/null

# If not found, check js8spotter.live or GitHub for latest release
# Typically distributed as AppImage or .deb
```

### Configure JS8 Spotter

JS8 Spotter connects to a running JS8Call instance via its API:
- JS8Call must be running with API enabled: File → Settings → Reporting → Enable TCP Server API (default port 2442)

JS8 Spotter settings:
- Host: localhost
- Port: 2442 (JS8Call default)

JS8 Spotter will then display decoded stations, signal reports, and propagation data from your JS8Call session.

---

## SECTION E — RUNNING MULTIPLE APPS

If running fldigi and WSJT-X or JS8Call simultaneously (e.g., monitoring multiple bands):
- Use flrig or rigctld as a single rig control server — don't have multiple apps controlling the rig directly
- Audio: each app needs exclusive access to the radio audio device, so only run one at a time unless you have multiple radios
- Consider virtual audio cables (JACK or PipeWire loopback) for advanced routing

---

## SECTION F — COMMON SETUP FOR ALL DIGITAL APPS

### rigctld as shared rig server

If running multiple apps, start rigctld once and have all apps connect to it:

```bash
# Start rigctld (use values from config)
rigctld -m [hamlib_id] -r [cat_port] -s [baud_rate] &

# Test connection
rigctl -m 2 get_freq  # model 2 = NET rigctl
```

Consider a systemd user service for auto-start:
```bash
# Show user a .service file to create, ask for confirmation
mkdir -p ~/.config/systemd/user/
```

### Time sync (critical for FT8/JS8)

```bash
timedatectl status
# Should show: "System clock synchronized: yes"
# If not:
sudo timedatectl set-ntp true
sudo systemctl enable --now systemd-timesyncd
```
