# Elmer Winlink — Pat, ARDOP, Vara HF

Set up and operate Winlink email over radio using Pat as the client, with ARDOP or Vara HF as the transport.

## STARTUP

Read `~/.config/elmer/config.yaml`. If missing, direct to `/elmer-setup`.

Prerequisites: HF rig should be configured and working. If not, suggest `/elmer-rig` and `/elmer-audio` first.

Ask what they want to work on:
1. Install and configure Pat
2. Set up ARDOP
3. Set up Vara HF (via Bottles)
4. Set up Mercury (open-source OFDM modem)
5. Make my first Winlink connection
6. Troubleshoot a connection problem

---

## SECTION A — PAT (WINLINK CLIENT)

Pat is a cross-platform Winlink client that works with multiple transport protocols.

### Install Pat

Check config `software.pat.installed`. If false:
```bash
# Download latest release from GitHub (github.com/la5nta/pat/releases)
# Look for linux_amd64 .deb or .tar.gz

# For .deb:
wget [latest .deb URL from GitHub releases]
sudo dpkg -i pat_*.deb

# Or via package manager if available:
# Check: apt-cache search pat winlink
```

### Configure Pat

```bash
# Initialize Pat config
pat configure
# This opens config in $EDITOR
```

Key config fields in `~/.config/pat/config.json`:
```json
{
  "mycall": "[callsign from elmer config]",
  "secure_login_password": "",  // Set via Winlink.org account
  "locator": "[grid from elmer config]",
  "hamlib_rigs": {
    "hf-rig": {
      "address": "localhost:4532",  // rigctld address
      "network": "tcp"
    }
  },
  "ardop": {
    "addr": "localhost:8515",
    "rig": "hf-rig",
    "ptt_ctrl": true
  },
  "vara": {
    "host": "localhost",
    "cmdPort": 8300,
    "dataPort": 8301,
    "rig": "hf-rig",
    "ptt_ctrl": true
  }
}
```

Guide user through filling in their callsign and grid from config, then choose transport (ARDOP or Vara HF).

### Pat web interface

Pat includes a web UI:
```bash
pat http
# Opens browser UI at http://localhost:8080
```

For headless use or terminal preference:
```bash
pat --help  # Show all commands
pat connect ardop:///[gateway-callsign]
pat connect varahf:///[gateway-callsign]
```

### Find Winlink gateways

```bash
# Pat can list nearby gateways
pat listen --list  # or check winlink.org/sysop for gateway list
```

---

## SECTION B — ARDOP

ARDOP (Amateur Radio Digital Open Protocol) is a free software modem for HF Winlink.

### Install ardop2call or piardop2

```bash
# Check if installed
which ardop2call piardop2 2>/dev/null

# ardop2call (recommended for Linux desktop):
# Download from github.com/pflarue/ardop or ardop.groups.io
# Often distributed as binary

# Common install location after download:
# sudo cp ardop2call /usr/local/bin/
# sudo chmod +x /usr/local/bin/ardop2call
```

### Configure ARDOP

ARDOP needs:
- Audio device (radio receive/transmit)
- PTT control method

```bash
# Run ardop2call (use audio device names from config)
ardop2call 8515 [capture-device] [playback-device] -p [ptt-port]

# Example with RTS PTT:
ardop2call 8515 hw:2,0 hw:2,0 -p /dev/hf-cat --keystring "RTSCTS"

# Example with CAT PTT via rigctld:
ardop2call 8515 hw:2,0 hw:2,0  # then configure Pat to handle PTT via hamlib
```

The `8515` is the ARDOP port — Pat connects here.

### Audio device format for ARDOP

ARDOP uses ALSA device names. Get the right format:
```bash
aplay -l  # note card number and device number
# Format: hw:[card],[device] e.g., hw:2,0
```

### ARDOP startup script

Create a launch script:
```bash
cat << 'EOF' > ~/bin/start-ardop.sh
#!/bin/bash
ardop2call 8515 hw:[card],0 hw:[card],0 -p /dev/hf-cat --keystring "RTSCTS"
EOF
chmod +x ~/bin/start-ardop.sh
```

Ask user to confirm values, write the script.

### Test ARDOP

```bash
# Start ARDOP
~/bin/start-ardop.sh &

# In Pat, connect to a test gateway or local echo server
# pat connect ardop:///W1AW  # replace with nearby gateway
```

---

## SECTION C — VARA HF (VIA BOTTLES)

Vara HF is a commercial (free tier available) high-performance HF modem. It's Windows-only software, so on Linux it runs under Wine via Bottles.

### Prerequisites

```bash
# Check Bottles is installed
flatpak list | grep -i bottles
# or
which bottles 2>/dev/null
snap list | grep -i bottles
```

If not installed:
```bash
# Bottles is best installed via Flatpak:
flatpak install flathub com.usebottles.bottles

# Or via your distro's package manager if available
```

### Create a Vara HF bottle

1. Open Bottles
2. Create new bottle: Name it "VaraHF", type "Application"
3. Install VARA HF:
   - Download VARA HF installer from sigmalinks.com (free tier works for Winlink)
   - In Bottles, open the "VaraHF" bottle
   - Click "Run Executable" → select VARA HF installer
   - Follow Windows installer prompts

4. Configure VARA HF audio:
   - In VARA HF settings (accessible from Bottles), set audio device to the radio's audio device
   - Wine may present audio devices differently — look for "USB Audio" or the rig's audio name

5. Configure VARA HF ports:
   - VARA HF listens on: command port 8300, data port 8301 (defaults)
   - These are the ports Pat uses

### Update config

After Vara HF is installed:
```yaml
vara_hf:
  installed: true
  via_bottles: true
  bottles_bottle: "VaraHF"
```

### Launch VARA HF from command line

```bash
# Find the Bottles flatpak run command for the bottle
flatpak run com.usebottles.bottles --bottle VaraHF --exe "C:\\Program Files\\VARA\\VARA.exe"
# Adjust path to match actual install location inside bottle
```

Create a launch script and add to config.

### Test Vara HF with Pat

1. Start VARA HF (via Bottles)
2. Start rigctld (if not running)
3. Start Pat: `pat http`
4. In Pat, select Vara HF transport and connect to a gateway

---

## SECTION C2 — MERCURY

Mercury (github.com/Rhizomatica/mercury) is an open-source HF OFDM modem (FreeDV-based). It's native on Linux, so no Bottles or Wine. It exposes a **VARA-compatible TNC**: control port 8300, data port 8301 (plus broadcast on 8100). See `docs/TNC.md` in the Mercury repo.

### How Pat talks to it

Stock Pat has no `mercury` transport. Use Pat's **VARA HF transport** pointed at Mercury's port:
```json
"varahf": { "addr": "localhost:8300", "rig": "...", "ptt_ctrl": true }
```
```bash
pat connect varahf:///[GATEWAY-CALL]   # the gateway must also run Mercury
```
Mercury and VARA HF both default to 8300, so **run only one at a time**. Pat's `varahf` entry talks to whichever one is listening. To run both, move Mercury with `arq_tcp_base_port` (or `-p`) and give Pat a second address.

### Config file: location matters

Mercury searches, in order:
1. `./mercury.ini` (current directory)
2. `$XDG_CONFIG_HOME/mercury/mercury.ini` → `~/.config/mercury/mercury.ini` (**lowercase** `mercury`)
3. `/etc/mercury/mercury.ini`

Or force one with `-C <file>`. **If none is found it silently runs on built-in defaults**, which means ALSA's default device, i.e. the laptop speakers. Always check the first line of output:
```bash
mercury -h 2>&1 | head -1
# Good: "Loaded configuration from /home/.../.config/mercury/mercury.ini"
# Bad:  "No configuration file found ... running on built-in defaults"
```
Linux paths are case-sensitive: `~/.config/Mercury/` is NOT found. Symptom: TX audio comes out of the computer speakers instead of the rig.

### Key mercury.ini settings

```ini
[main]
sound_system   = pulse                            ; works on PipeWire via pipewire-pulse
input_device   = [radio source name]   ; from: pactl list short sources
output_device  = [radio sink name]     ; from: pactl list short sinks
capture_channel = left
radio_model    = 2                ; Hamlib NET rigctl → talk to rigctld
radio_device   = 127.0.0.1:4532   ; rigctld default port (rigctld may run the rig directly or proxy flrig with -m 4)
arq_tcp_base_port = 8300

[audio]
tx_gain_db = 0.0   ; Mercury-only TX drive, so you don't touch the shared mixer
```
- `mercury -z` lists sound devices and `mercury -K` lists Hamlib models.
- Use the stable WirePlumber names (see `/elmer-audio`, "Persistent, human-readable device names"), not card numbers.
- PTT: `mercury -Q` keys the configured PTT for 1 second and exits. **This transmits.** Confirm the rig is on a dummy load or a clear frequency first.
- If both Mercury (`radio_model`) and Pat (`ptt_ctrl: true`) are set up to key PTT, pick one owner if you see double-keying or PTT errors.

### Troubleshooting Mercury

- **Audio out the laptop speakers:** the config wasn't loaded. Check the first output line and the folder case.
- **Pat "connection refused" on 8300:** Mercury isn't running, or VARA HF grabbed the port first.
- **No decodes:** check `capture_channel` (most rig USB codecs put RX audio on left/mono) and RX levels (`/elmer-audio`).

---

## SECTION D — MAKING A WINLINK CONNECTION

### Pre-flight checklist

- [ ] rigctld running (or flrig): rig is responding to CAT
- [ ] ARDOP, VARA HF, or Mercury running (Mercury: first line says "Loaded configuration from ...")
- [ ] Pat configured with callsign and transport
- [ ] Rig set to correct Winlink frequency (check winlink.org/frequencies)
- [ ] Rig in USB mode (not LSB)
- [ ] Audio levels set (see `/elmer-audio`)

### Common Winlink frequencies (HF)

| Band | Frequency (USB) |
|------|----------------|
| 80m  | 3.598.5 kHz    |
| 40m  | 7.102.0 kHz    |
| 30m  | 10.147.2 kHz   |
| 20m  | 14.102.0 kHz   |
| 17m  | 18.102.0 kHz   |

### Connect

```bash
# In Pat web UI or CLI:
pat connect ardop:///[GATEWAY-CALL]
# or
pat connect varahf:///[GATEWAY-CALL]
```

Watch for: "Connected", handshake, message exchange, "Disconnected."

---

## SECTION E — TROUBLESHOOTING

**ARDOP won't start:** audio device not found — double-check hw: device numbers with `aplay -l`

**Vara HF not connecting to Pat:** check ports (8300/8301) and that Vara is actually running

**PTT not working:** verify rigctld is running and Pat's rig config points to it

**Connection fails immediately:** gateway may be busy or out of range — try another gateway or time

**Audio clipping/distortion on TX:** reduce system audio output level to radio (see `/elmer-audio`)
