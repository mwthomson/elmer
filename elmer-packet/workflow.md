# Elmer Packet — Direwolf, AX.25, Packet Radio

Configure Direwolf as a software TNC, set up the Linux AX.25 stack, and get on packet radio.

## STARTUP

Read `~/.config/elmer/config.yaml`. If missing, direct to `/elmer-setup`.

VHF/UHF rig and audio should be configured. If not, suggest `/elmer-rig` and `/elmer-audio` first.

Ask what they want to work on:
1. Install and configure Direwolf
2. Set up AX.25 and kissattach
3. Connect Direwolf to Pat (Winlink via packet)
4. Troubleshoot Direwolf or decoding issues

---

## SECTION A — DIREWOLF

### Install

Check config `software.direwolf.installed`. If false:
```bash
# Debian/Ubuntu/MX Linux:
sudo apt install direwolf

# Arch:
sudo pacman -S direwolf

# Fedora:
sudo dnf install direwolf

# Or build from source for latest version:
# github.com/wb2osz/direwolf
```

### Configure Direwolf

Config file: `~/.direwolf/direwolf.conf` or `/etc/direwolf.conf`

```bash
# Copy the sample config to start
cp /etc/direwolf.conf ~/.direwolf/direwolf.conf 2>/dev/null || \
  cp /usr/share/doc/direwolf/examples/direwolf.conf ~/.direwolf/direwolf.conf 2>/dev/null || \
  echo "Sample config not found — will create from scratch"
```

Key settings to configure (use values from elmer config):

```
# Callsign
MYCALL [callsign from config]-1   # Use SSID -1 for packet; adjust as needed

# Audio device
ADEVICE [audio_device from vhf_uhf config]
# Example: ADEVICE plughw:1,0
# or using ALSA device name: ADEVICE hw:Digirig,0

# Sample rate (optional — Direwolf auto-detects)
ARATE 48000

# Channel 0 — VHF packet (1200 baud AFSK for 2m packet)
CHANNEL 0
MODEM 1200

# PTT control
# For Digirig Lite (RTS on serial port):
PTT /dev/vhf-cat RTS
# For VOX:
# PTT VOX
# For CM108 GPIO (some interfaces):
# PTT CM108

# KISS interface (for AX.25 stack or Pat)
KISSPORT 8001
```

Walk user through each setting, using their config values. Help them find the correct ALSA device format:
```bash
aplay -l  # Find card and device numbers for Digirig
# Format: plughw:[card],[device] — use plughw (not hw) for sample rate conversion
```

### Test Direwolf

```bash
direwolf -c ~/.direwolf/direwolf.conf
```

On 2m (144.390 MHz for APRS, or listen on local packet frequency):
- Should show decoded packets if any are received
- If no packets: check audio level with `/elmer-audio`, verify correct audio device

### PTT test

To test PTT without transmitting audio:
- Tune to a clear frequency
- Watch TX LED on rig
- Direwolf will show "T" when transmitting

---

## SECTION B — AX.25 STACK

### Install kernel modules and tools

```bash
# Check if AX.25 is available
lsmod | grep ax25
modprobe ax25 2>/dev/null && echo "AX.25 available" || echo "AX.25 not available"

# Install userspace tools
sudo apt install ax25-tools ax25-apps  # Debian/Ubuntu
sudo pacman -S ax25-tools              # Arch
```

### kissattach — connect Direwolf to AX.25

First, Direwolf must be running with KISS interface enabled (KISSPORT set in config).

```bash
# Connect KISS port to AX.25 stack
sudo kissattach /dev/ptmx ax0 44.x.x.x
# or using socat for TCP KISS:
sudo kissattach pty ax0

# Better approach — connect via TCP KISS to Direwolf:
# 1. In one terminal: socat -d -d pty,raw,echo=0 tcp:localhost:8001
# 2. Note the pty device (e.g., /dev/pts/3)
# 3: sudo kissattach /dev/pts/3 ax0
```

### /etc/ax25/axports

Define the AX.25 port:
```bash
sudo nano /etc/ax25/axports
# Add line:
# ax0    [CALLSIGN]-1    9600    255     2       2m Packet
```

Format: `portname callsign speed paclen window description`

### Test AX.25

```bash
# Check interface is up
ifconfig ax0 2>/dev/null || ip addr show ax0 2>/dev/null

# Listen for packets
axlisten -a

# Connect to a node
axcall ax0 [NODE-CALLSIGN]
```

---

## SECTION C — DIREWOLF + PAT (WINLINK VIA PACKET)

Pat can use Direwolf as a KISS TNC for Winlink connections on VHF.

### Configure Pat for KISS/packet

In `~/.config/pat/config.json`, add a KISS transport config:
```json
"ax25": {
  "port": "ax0",
  "rig": ""
}
```

Or use the AGWPE/KISS TCP interface:
```json
"agwpe": {
  "addr": "localhost:8000"
}
```

Note: Direwolf supports both KISS (TCP port 8001) and AGW (port 8000) interfaces simultaneously.

### Winlink on VHF (RMS Packet)

Frequencies vary by region — check winlink.org/RMSChannels for local RMS Packet stations.

```bash
# Connect via Pat
pat connect ax25:///[GATEWAY-CALLSIGN]
```

---

## SECTION D — SYSTEMD SERVICE FOR DIREWOLF

For automatic startup:

```bash
mkdir -p ~/.config/systemd/user/
```

Create `~/.config/systemd/user/direwolf.service`:
```ini
[Unit]
Description=Direwolf APRS/Packet TNC
After=sound.target

[Service]
ExecStart=/usr/bin/direwolf -c %h/.direwolf/direwolf.conf
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
```

```bash
systemctl --user enable direwolf
systemctl --user start direwolf
systemctl --user status direwolf
```

---

## SECTION E — TROUBLESHOOTING

**Direwolf shows audio but no decodes:**
- Check audio level — too high causes clipping, too low misses signals
- Try `MODEM 1200 0 0` to adjust AFSK discriminator
- Run `direwolf -c ~/.direwolf/direwolf.conf -T "%H:%M:%S"` for timestamps

**PTT not keying:**
- Verify RTS device: `ls -la /dev/vhf-cat` (should exist if udev configured)
- Check permissions: `ls -la /dev/ttyUSB*` — needs dialout group
- Try DTR instead of RTS if RTS doesn't work for your interface

**kissattach fails:**
- AX.25 module not loaded: `sudo modprobe ax25`
- Port busy: check if Direwolf is running on the KISS port

**Audio device "busy" error:**
- Another app has the device open
- Use `plughw:` prefix instead of `hw:` to allow sharing
- Or use PulseAudio/PipeWire virtual device
