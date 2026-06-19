# Elmer APRS — igate, Xastir, aprx, APRS Beaconing

Set up APRS receive, igate, and digipeating using Direwolf, Xastir, or aprx.

## STARTUP

Read `~/.config/elmer/config.yaml`. If missing, direct to `/elmer-setup`.

Direwolf (from `/elmer-packet`) is the foundation for APRS on Linux. Ensure it's configured first.

Ask what they want to work on:
1. Set up an APRS igate (receive → internet)
2. Configure APRS beaconing (transmit position)
3. Install and use Xastir (map-based APRS client)
4. Set up aprx (lightweight igate daemon)
5. APRS via Direwolf only (no separate app)

---

## SECTION A — APRS WITH DIREWOLF

Direwolf can act as a standalone igate without additional software.

### Direwolf igate configuration

Add to `~/.direwolf/direwolf.conf`:

```
# APRS-IS igate settings
IGSERVER rotate.aprs2.net
IGLOGIN [CALLSIGN] [PASSCODE]   # passcode from aprs.fi or aprsisce.com/hampassword

# Receive-only igate (safe starting point):
IGTXLIMIT 6 10

# Your location (for distance filtering)
PBEACON delay=1 every=30 overlay=R symbol="igate" lat=[LAT] long=[LON] \
  power=5 height=10 gain=3 comment="Linux igate"

# Filter — only forward packets within 200km:
FILTER IG 200km/[LAT]/[LON]
```

Get APRS passcode:
```bash
# Many online calculators — or use this quick Python:
python3 -c "
call='[CALLSIGN]'.upper().split('-')[0]
h=0x73e2
for c in call[:10]:
    h ^= ord(c)<<8
    h ^= ord(c) if len(call)>call.index(c)+1 else 0
    for _ in range(8): h=(h<<1)^(0x8408 if h&0x8000 else 0)
print(h & 0x7fff)
"
```

(Fill in actual callsign from config)

### Test igate

```bash
direwolf -c ~/.direwolf/direwolf.conf
```

Watch for lines starting with `[ig]` — these are packets being forwarded to APRS-IS.

Verify on aprs.fi — search for your callsign to see if it appears as an igate.

---

## SECTION B — APRS BEACONING

### Beacon your position via Direwolf

Add to direwolf.conf:
```
# Position beacon every 10 minutes
PBEACON delay=1 every=10 overlay=S symbol="house" \
  lat=[LAT] long=[LON] \
  comment="[CALLSIGN] Linux station"
```

For mobile/moving station:
```
# SmartBeaconing (speed-adaptive)
SMARTBEACONING 60 2:0 30:70 255 0:30 5
```

Get your lat/lon:
```bash
# Quick lookup if you know your grid square:
python3 -c "
import sys
grid='[GRID]'.upper()
lon=(ord(grid[0])-ord('A'))*20-180+(ord(grid[2])-ord('0'))*2+(ord(grid[4])-ord('a'))*2/24+1/24 if len(grid)>=6 else None
lat=(ord(grid[1])-ord('A'))*10-90+(ord(grid[3])-ord('0'))+(ord(grid[5])-ord('a'))/24+1/48 if len(grid)>=6 else None
print(f'lat={lat:.4f}, lon={lon:.4f}')
"
```

---

## SECTION C — XASTIR

Xastir is a full-featured APRS client with map display.

### Install

```bash
# Debian/Ubuntu/MX Linux:
sudo apt install xastir

# Arch:
sudo pacman -S xastir

# Fedora:
sudo dnf install xastir
```

### Configure Xastir

**Interface → Interface Control → Add:**
- For Direwolf (AGWPE/TCP): select "AGWPE Tnc" → localhost:8000
- For APRS-IS internet feed: select "Internet Server" → rotate.aprs2.net:14580

**Configure → Station:**
- Callsign (from config)
- Position (lat/lon)
- Symbol: choose appropriate icon

**Maps:**
Xastir supports many map sources. For OpenStreetMap tiles:
- Configure → Map Chooser → OSM maps (may need additional packages)

### Xastir tips
- Colors indicate packet age: recent = bright, old = faded
- Ctrl+click a station to see packet history
- Right-click for messaging

---

## SECTION D — APRX (LIGHTWEIGHT IGATE DAEMON)

aprx is a minimal APRS igate/digipeater daemon — good for headless systems.

### Install

```bash
# Debian/Ubuntu/MX Linux:
sudo apt install aprx

# Arch: may need to build from source
# github.com/PhirePhly/aprx
```

### Configure aprx

Config at `/etc/aprx.conf`:
```
mycall [CALLSIGN]-10   # SSID -10 common for igates

<aprsis>
  login [CALLSIGN]-10
  passcode [PASSCODE]
  server rotate.aprs2.net
  heartbeat-timeout 0
  filter "r/[LAT]/[LON]/200"  # 200km radius
</aprsis>

<interface>
  # Connect to Direwolf KISS port via TCP
  tcp-device localhost 8001 KISS
  <tnc>
    kiss-subif 0
  </tnc>
</interface>

<beacon>
  beaconmode both      # forward and transmit
  cycle-size 20m
  beacon symbol "I&" lat "[LAT]N" lon "[LON]W" \
    comment "Linux igate running aprx"
</beacon>
```

```bash
# Start aprx
sudo systemctl enable --now aprx
sudo systemctl status aprx
```

---

## SECTION E — APRS FREQUENCIES

Standard APRS frequencies:

| Region | Frequency |
|--------|-----------|
| North America | 144.390 MHz FM |
| Europe | 144.800 MHz FM |
| Australia | 145.175 MHz FM |
| New Zealand | 144.575 MHz FM |

Set VHF/UHF rig to the appropriate frequency, FM mode, 12.5 kHz channel spacing if selectable.

---

## SECTION F — VERIFY YOUR IGATE IS WORKING

```bash
# Check APRS-IS connection in Direwolf output
# Should see: "Connected to APRS-IS server"
# Then packets starting with [ig]

# Check aprs.fi
# Search your callsign — should appear as igate station
# Nearby stations you're hearing should show "heard by [yourcall]"
```
