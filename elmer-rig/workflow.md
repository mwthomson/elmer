# Elmer Rig — CAT Control, Hamlib, flrig, udev

Help the user establish reliable rig control: consistent device naming via udev, Hamlib configuration, flrig setup, and CAT control testing.

## STARTUP

Read `~/.config/elmer/config.yaml`. If missing, tell user to run `/elmer-setup` first.

Determine what to work on:
- If user describes a specific problem, go directly to the relevant section
- Otherwise ask: "What would you like to work on? (1) udev device naming, (2) Hamlib setup, (3) flrig configuration, (4) test CAT control, (5) find my rig's Hamlib ID"

---

## SECTION A — UDEV DEVICE NAMING

**Why this matters:** USB serial devices get assigned ports like `/dev/ttyUSB0` in the order they're plugged in. With multiple radios, the assignments can change. udev rules create stable aliases like `/dev/hf-cat` that always point to the right device.

### Step 1 — Identify USB device attributes

```bash
# List all connected USB serial devices with vendor/product IDs
lsusb
ls -la /dev/ttyUSB* /dev/ttyACM* 2>/dev/null
```

For each relevant device, get its udev attributes:
```bash
udevadm info -a -n /dev/ttyUSB0 | grep -E "(idVendor|idProduct|serial|manufacturer|product)" | head -20
# Repeat for each ttyUSB/ttyACM device
```

### Step 2 — Build udev rules

Show the user what was found. For each radio device, construct a udev rule using the serial number (preferred — unique per device) or vendor+product ID (less specific but still useful).

Example rule format:
```
# Yaesu FT-710 CAT port
SUBSYSTEM=="tty", ATTRS{idVendor}=="10c4", ATTRS{idProduct}=="ea60", ATTRS{serial}=="[SERIAL]", SYMLINK+="hf-cat", MODE="0666"

# Digirig Lite (for VHF rig)
SUBSYSTEM=="tty", ATTRS{idVendor}=="10c4", ATTRS{idProduct}=="ea60", ATTRS{serial}=="[SERIAL]", SYMLINK+="vhf-cat", MODE="0666"
```

Use the actual vendor/product IDs and serial numbers found on their system. Use the config's suggested symlink names.

### Step 3 — Apply rules

```bash
# Show what will be written
cat << 'EOF'
[rules content]
EOF
```

Ask for confirmation, then:
```bash
sudo tee /etc/udev/rules.d/99-hamradio.rules << 'EOF'
[rules content]
EOF

sudo udevadm control --reload-rules
sudo udevadm trigger
```

Verify:
```bash
ls -la /dev/hf-cat /dev/vhf-cat 2>/dev/null || echo "Aliases not yet visible — try unplugging and replugging the devices"
```

Update `udev.configured: true` in the config.

---

## SECTION B — HAMLIB

### Find a rig's Hamlib ID

```bash
# Search by manufacturer/model name
rigctl -l 2>/dev/null | grep -i "[rig name from config]"
# If hamlib not installed:
echo "Install hamlib first with: sudo apt install libhamlib-utils  (or equivalent for their distro)"
```

Show results and help them identify the correct entry. Common rigs for reference (Hamlib 4.6; IDs can change between releases, so always confirm with `rigctl -l`):
- Yaesu FT-710: 1049
- Yaesu FT-991A: 1035
- Yaesu FT-857D: 1022
- Icom IC-7300: 3073
- Icom IC-705: 3085
- Kenwood TS-590SG: 2037

Update config with confirmed Hamlib ID.

### Test Hamlib connection

Using values from config:
```bash
rigctl -m [hamlib_id] -r [cat_port] -s [baud_rate] get_freq
```

If it fails, work through common issues:
1. **Permission denied** → `sudo usermod -aG dialout $USER` (requires logout/login) or check udev MODE
2. **Device not found** → verify port with `ls /dev/ttyUSB*`, check udev rules
3. **Wrong baud rate** → try common rates: 4800, 9600, 19200, 38400
4. **CI-V address wrong** (Icom) → check rig menu for CI-V address, add `-c [address]` flag
5. **Flow control** → try adding `-C serial_handshake=None`

---

## SECTION C — FLRIG SETUP

### Install (if needed)

Check config `software.flrig.installed`. If false:
```bash
# Debian/Ubuntu/MX Linux:
sudo apt install flrig

# Arch:
sudo pacman -S flrig

# Fedora:
sudo dnf install flrig

# Or build from source / download AppImage from W1HKJ.com
```

### Configure flrig

flrig config file is at `~/.config/flrig/flrig.prefs` or `~/.flrig/`.

Guide through the flrig GUI settings or edit config directly:

Key settings to verify:
- **Rig**: select from dropdown matching their rig
- **Device**: set to CAT port from config (e.g., `/dev/hf-cat`)
- **Baud rate**: match rig setting
- **PTT via**: none for HF (rig handles TX) or RTS/DTR if using separate interface

Test: click Connect in flrig. It should show the current frequency.

### flrig as Hamlib proxy

Many apps (fldigi, WSJT-X) can connect to flrig via rigctld rather than controlling the rig directly. This prevents port conflicts.

To start flrigd (if using flrig as server):
flrig already exposes XML-RPC on port 12345 by default.

---

## SECTION D — USER GROUPS

Ensure the user is in the right groups for serial port access:

```bash
groups $USER
```

If `dialout` (Debian/Ubuntu) or `uucp` (Arch) is missing:
```bash
# Debian/Ubuntu/MX Linux:
sudo usermod -aG dialout $USER

# Arch:
sudo usermod -aG uucp $USER
```

Remind: "You'll need to log out and back in for the group change to take effect."

---

## STYLE NOTES

- udev rules are the foundation — recommend doing this first if multiple USB devices are present
- When a CAT test fails, be methodical: port → baud rate → permissions → rig-specific settings
- If the user is on MX Linux or another Debian derivative, `dialout` is the correct group
