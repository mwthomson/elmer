# Elmer SDR — Software Defined Radio on Linux

Install drivers and configure SDR software: GQRX, SDR++, GNU Radio, and SDR dongles.

## STARTUP

Read `~/.config/elmer/config.yaml`. If missing, direct to `/elmer-setup`.

Ask what they want to work on:
1. Set up an RTL-SDR or other SDR dongle
2. Install and configure GQRX
3. Install and configure SDR++
4. GNU Radio basics
5. Use SDR to monitor amateur radio frequencies
6. Troubleshoot SDR not being detected

---

## SECTION A — SDR DONGLE SETUP

### Detect the dongle

```bash
lsusb | grep -iE "rtl|hackrf|airspy|sdrplay|funcube|nooelec"
```

Common USB IDs:
- RTL-SDR Blog V3: 0bda:2838 or 0bda:2832
- HackRF One: 1d50:6089
- Airspy: 1d50:60a1
- SDRplay RSP: 1df7:xxxx

### Install RTL-SDR drivers (most common)

```bash
# Debian/Ubuntu/MX Linux:
sudo apt install rtl-sdr librtlsdr-dev

# Arch:
sudo pacman -S rtl-sdr

# Fedora:
sudo dnf install rtl-sdr
```

### Blacklist the kernel DVB driver

The RTL-SDR chip is often grabbed by the kernel's DVB-T driver before librtlsdr can use it:

```bash
# Check if DVB driver has claimed the device:
lsmod | grep dvb_usb_rtl28xxu

# Blacklist it:
echo "blacklist dvb_usb_rtl28xxu" | sudo tee /etc/modprobe.d/rtlsdr.conf
echo "blacklist rtl2832" | sudo tee -a /etc/modprobe.d/rtlsdr.conf
echo "blacklist rtl2830" | sudo tee -a /etc/modprobe.d/rtlsdr.conf

# Reload:
sudo rmmod dvb_usb_rtl28xxu 2>/dev/null
sudo udevadm control --reload-rules
```

### udev rules for RTL-SDR (non-root access)

```bash
# Create udev rule for RTL-SDR
cat << 'EOF' | sudo tee /etc/udev/rules.d/20-rtlsdr.rules
SUBSYSTEM=="usb", ATTRS{idVendor}=="0bda", ATTRS{idProduct}=="2838", GROUP="plugdev", MODE="0664"
SUBSYSTEM=="usb", ATTRS{idVendor}=="0bda", ATTRS{idProduct}=="2832", GROUP="plugdev", MODE="0664"
EOF
sudo udevadm control --reload-rules
sudo udevadm trigger
```

Add user to plugdev group if not already:
```bash
groups $USER | grep plugdev || sudo usermod -aG plugdev $USER
```

### Test RTL-SDR

```bash
# Test basic detection
rtl_test -t

# Expected output: "Found 1 device(s)..." then tuner type
# If error: device not recognized or driver conflict
```

---

## SECTION B — GQRX

### Install

Check config `software.gqrx.installed`. If false:
```bash
# Debian/Ubuntu/MX Linux:
sudo apt install gqrx-sdr

# Arch:
sudo pacman -S gqrx

# Fedora:
sudo dnf install gqrx

# Or use flatpak for latest version:
flatpak install flathub org.gqrx.gqrx
```

### Configure GQRX

On first launch, GQRX asks for device configuration:

**Device:** Select "RTL-SDR" or the appropriate dongle
**Device string:** `rtl=0` for first RTL-SDR, `rtl=1` for second
**Sample rate:** 2000000 (2 MSPS) is a good default for RTL-SDR

Input rate and bandwidth will auto-populate.

### GQRX settings for ham radio use

**Receiver options:**
- Mode: USB for HF SSB, FM-N for VHF NFM, WFM for broadcast
- Filter width: adjust for signal width (2.4 kHz for SSB, 12-15 kHz for NFM)

**FFT settings:**
- FFT size: 8192 or larger for good resolution
- Rate: 25 fps is smooth without excessive CPU

**Audio:**
- Audio device: use system audio (not radio audio!)
- Sample rate: 48000

### GQRX for monitoring ham radio

Good starting frequencies:
- 14.200–14.350 MHz: 20m SSB
- 144.200 MHz: 2m SSB calling frequency
- 146.520 MHz: 2m FM simplex calling
- Local repeaters: check repeaterbook.com

---

## SECTION C — SDR++

SDR++ is a modern, high-performance SDR application with plugin architecture.

### Install

Check config `software.sdrpp.installed`. If false:
```bash
# Check for package:
apt-cache search sdrpp 2>/dev/null

# If not in repos, download AppImage from github.com/AlexandreRouma/SDRPlusPlus/releases
# AppImage install:
chmod +x SDRPlusPlus-*.AppImage
./SDRPlusPlus-*.AppImage  # run directly, or install to /usr/local/bin
```

### Configure SDR++

On first launch:
1. Source module panel → select RTL-SDR (or your dongle type)
2. Set frequency, sample rate
3. Press Play button

SDR++ auto-saves configuration.

**Notable SDR++ features:**
- Multiple VFOs simultaneously
- Recording (baseband and audio)
- Plugin system (rigctl, frequency manager, etc.)
- Better performance than GQRX on older hardware

### SDR++ rigctl plugin

SDR++ can sync frequency with your rig via rigctl:
- Modules → rigctl client → enable
- Set to localhost:4532 (if rigctld is running)
- Frequency changes in SDR++ will track rig, or vice versa

---

## SECTION D — GNU RADIO

GNU Radio is a powerful signal processing framework — more complex but extremely capable.

### Install

```bash
# Debian/Ubuntu/MX Linux:
sudo apt install gnuradio

# Arch:
sudo pacman -S gnuradio

# Fedora:
sudo dnf install gnuradio
```

### GNU Radio basics

GNU Radio Companion (GRC) is the graphical flow graph editor:
```bash
gnuradio-companion
```

For ham radio use, common flow graphs are available online:
- RTL-SDR FM receiver
- WBFM receiver
- ADS-B decoder
- APRS decoder (as alternative to Direwolf)

### OsmoSDR source block

For RTL-SDR in GNU Radio, use the OsmoSDR source:
```bash
sudo apt install gr-osmosdr
```

In GRC: add "Osmocom Source" block, device arguments: `rtl=0`

---

## SECTION E — COMMON SDR PROBLEMS

**Device not found:**
```bash
rtl_test  # Is the device seen at all?
lsusb     # USB enumerated?
dmesg | tail -20  # USB errors?
```

**DVB driver conflict:**
```bash
lsmod | grep dvb
# If listed: sudo rmmod dvb_usb_rtl28xxu
# Then add to blacklist (see Section A)
```

**Permission denied:**
```bash
ls -la /dev/bus/usb/[bus]/[device]  # Check permissions
# Fix: udev rule with GROUP="plugdev" and usermod -aG plugdev $USER
```

**Poor sensitivity / noise:**
- For RTL-SDR: enable bias tee only if you have an LNA that needs it
- Check antenna connection
- RTL-SDR V3 direct sampling mode for HF: add `direct_samp=2` to device string

**Frequency offset (RTL-SDR):**
- RTL-SDR has inaccurate oscillator — may need PPM correction
- In GQRX: Input controls → Freq. correction (ppm)
- Find PPM: tune to a known-frequency signal and measure offset
- Or use `rtl_test -p` for automatic measurement (takes ~10 minutes)
