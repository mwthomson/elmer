# Elmer Setup — System Configuration Interview

Conduct a friendly setup interview to learn the user's system and radio hardware. Auto-detect what you can, confirm with the user, ask about the rest. Write a completed config to `~/.config/elmer/config.yaml`.

## BEFORE STARTING

Check if `~/.config/elmer/config.yaml` already exists.
- If yes: "I already have a config saved. Do you want to (1) update specific sections, or (2) start fresh?"
- If starting fresh or no config: proceed with full interview below.

---

## PHASE 1 — OPERATOR INFO

Ask:
1. What is your callsign?
2. What is your Maidenhead grid square locator? (e.g., FN31pr — if unsure, offer: "You can find it at qth.com/grid.htm")
3. How would you describe your Linux experience? (beginner / intermediate / experienced)

---

## PHASE 2 — SYSTEM DETECTION

Run these commands and show the user what was found, asking them to confirm or correct:

```bash
# Distro info
cat /etc/os-release | grep -E "^(NAME|VERSION_ID|ID_LIKE|ID)="
```

```bash
# Audio stack
if command -v pw-cli &>/dev/null && pw-cli info 2>/dev/null | grep -q "PipeWire"; then
  echo "audio_stack: pipewire"
elif command -v pactl &>/dev/null; then
  pactl info 2>/dev/null | head -1
else
  echo "audio_stack: alsa-only"
fi
```

```bash
# Package manager
for pm in apt dnf pacman zypper; do command -v $pm &>/dev/null && echo $pm && break; done
```

Present findings as: "Here's what I found about your system — does this look right?"

---

## PHASE 3 — HF RIG

Ask: "Do you have an HF transceiver connected or planned?"

If yes:
1. What is the make and model? (e.g., Yaesu FT-710, Icom IC-7300)
2. How is it connected to your computer? (USB / serial / network)
3. Does it have a built-in USB sound card? (many modern rigs do — Icom IC-7300, Yaesu FT-710, etc.)

After they answer, run:
```bash
# Show connected USB devices to help identify ports
lsusb
ls /dev/ttyUSB* /dev/ttyACM* 2>/dev/null || echo "No serial devices found"
```

4. Look up the Hamlib rig ID:
```bash
rigctl -l 2>/dev/null | grep -i "[manufacturer] [model]"
# or if hamlib not installed yet:
echo "Hamlib not installed — we'll get the ID during rig setup"
```

5. What serial port does the CAT control use? (show list from above, or note we'll confirm with udev later)
6. What is the audio device name? Run:
```bash
aplay -l 2>/dev/null | grep -i "card"
```
Show output and ask them to identify their rig's audio device.

7. What baud rate for CAT? (common: 9600, 19200, 38400 — check rig menu)

---

## PHASE 4 — VHF/UHF RIG

Ask: "Do you have a VHF/UHF radio connected or planned?"

If yes:
1. What is the make and model?
2. What interface are you using? (Digirig / SignaLink / RigBlaster / direct audio / other)
3. How is it connected? (USB)

Run again if not already done:
```bash
lsusb
ls /dev/ttyUSB* /dev/ttyACM* 2>/dev/null
aplay -l 2>/dev/null | grep -i "card"
```

4. Which serial port for CAT (if supported)?
5. Which audio device for the interface?
6. PTT method? (RTS / DTR / VOX / CM108 / none)
7. Is PTT on the same port as CAT, or a separate port?

---

## PHASE 5 — SOFTWARE INVENTORY

Run a detection sweep:

```bash
echo "=== Checking installed ham radio software ==="
for app in fldigi flrig rigctl js8 direwolf pat ardop gqrx sdr++ xastir cqrlog; do
  path=$(which $app 2>/dev/null)
  if [ -n "$path" ]; then
    version=$($app --version 2>/dev/null | head -1 || echo "version unknown")
    echo "FOUND: $app — $version"
  else
    echo "NOT FOUND: $app"
  fi
done

# Check for WSJT-X separately (often installed as wsjtx or wsjt-x)
for name in wsjtx wsjt-x WSJTX; do
  path=$(which $name 2>/dev/null || find /usr /opt /home -name "$name" -type f 2>/dev/null | head -1)
  [ -n "$path" ] && echo "FOUND: wsjt-x at $path" && break
done

# Check for JS8 Spotter
find /usr /opt /home -name "js8spotter*" -o -name "JS8Spotter*" 2>/dev/null | head -3

# Check for Vara HF via Bottles
if command -v bottles &>/dev/null; then
  echo "FOUND: Bottles"
  bottles list 2>/dev/null || true
fi

# Check for Pat
which pat 2>/dev/null && pat version 2>/dev/null || echo "NOT FOUND: pat"
```

Present findings: "Here's what I found installed. Anything missing or incorrect?"

Ask: "Are there any of these you'd like me to help install or configure today?"
- fldigi / flrig / Hamlib
- WSJT-X
- JS8Call
- JS8 Spotter
- Direwolf
- Pat (Winlink)
- ARDOP
- Vara HF (via Bottles)
- GQRX / SDR++
- Xastir / aprx
- CQRLOG

---

## PHASE 6 — UDEV STATUS

Check if udev rules exist:
```bash
cat /etc/udev/rules.d/99-hamradio.rules 2>/dev/null || echo "No hamradio udev rules found"
```

If no rules and they have multiple USB devices: "You have multiple USB radio devices. I recommend setting up udev rules to give each device a consistent, predictable name like `/dev/hf-cat` and `/dev/vhf-cat`. Want me to help with that? We can do it now or later with `/elmer-rig`."

---

## PHASE 7 — WRITE CONFIG

Summarize everything collected and show the user what will be written.

Ask: "Does this all look correct? I'll save it to `~/.config/elmer/config.yaml`."

On confirmation:
```bash
mkdir -p ~/.config/elmer
```

Write the completed config to `~/.config/elmer/config.yaml` using the template from the Elmer skill directory, filled in with all collected values. Set `installed: true/false` and `version` fields based on what was detected.

Confirm: "Config saved! You're all set. Run `/elmer` to get started, or tell me what you'd like to work on first."

---

## STYLE NOTES

- Don't ask all questions at once — work through phases conversationally
- If the user doesn't know an answer (e.g., Hamlib ID), note it as blank and explain they can fill it in later or use `/elmer-rig` to find it
- If a rig model is well-known, proactively offer the Hamlib ID and common settings rather than making them look it up
- Keep it friendly — setup is the boring part before the fun part
