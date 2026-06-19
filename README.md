# Elmer — Amateur Radio Linux Assistant

Elmer is a set of Claude Code skills that act as a knowledgeable, patient ham radio mentor for Linux users. Named after the traditional ham radio term for an experienced operator who helps newcomers, Elmer guides you through configuring and operating amateur radio software step by step.

## Skills

### `/elmer` — Main entry point

Routes you to the right sub-skill based on what you describe. Reads your saved config (`~/.config/elmer/config.yaml`) and greets you by callsign. If no config exists, directs you to `/elmer-setup` first.

### `/elmer-setup` — Initial configuration

Walks through a friendly interview to learn your system and radio hardware. Auto-detects your Linux distro, audio stack, package manager, and installed ham radio software. Collects your callsign, grid square, rig details (HF and VHF/UHF), and audio device names. Writes everything to `~/.config/elmer/config.yaml`.

### `/elmer-rig` — CAT control, Hamlib, flrig, udev

Sets up reliable rig control:
- **udev rules** — creates stable device aliases like `/dev/hf-cat` so USB serial ports don't shift around between reboots
- **Hamlib** — finds your rig's model ID, tests CAT connectivity, diagnoses common failures (permissions, baud rate, CI-V address)
- **flrig** — installs and configures flrig, sets it up as a Hamlib proxy so multiple apps can share rig control without port conflicts
- **User groups** — ensures you're in `dialout` (Debian/Ubuntu) or `uucp` (Arch) for serial port access

### `/elmer-audio` — Sound card setup for digital modes

Identifies your audio devices, separates radio audio from system audio, and sets correct levels:
- Identifies HF rig built-in USB audio (IC-7300, FT-710, etc.), Digirig, SignaLink, and other interfaces
- Recommends transmit and receive level starting points, explains ALC
- Configures PipeWire/PulseAudio to keep radio audio devices from being grabbed by the desktop
- Interface-specific guidance for Digirig Lite, SignaLink USB, and rigs with built-in USB audio
- End-to-end audio path testing (record from radio, generate test tones)

### `/elmer-digital` — HF digital modes

Sets up and troubleshoots HF digital mode software:
- **fldigi** — configuration, audio routing, rig control connection, waterfall tuning
- **WSJT-X** — FT8, FT4, WSPR setup; audio device selection; CAT and PTT configuration
- **JS8Call** — configuration for JS8 messaging and the JS8 Spotter tool

### `/elmer-winlink` — Winlink email over radio

Configures Winlink for emergency/off-grid email:
- **Pat** — the primary Winlink client for Linux
- **ARDOP** — open-source HF soundcard modem
- **Vara HF** — Windows app running under Bottles (Wine) on Linux

### `/elmer-packet` — Packet radio

Sets up AX.25 packet radio on Linux:
- **Direwolf** — software TNC configuration for packet and APRS
- **AX.25 kernel stack** — `kissattach`, `axports`, and related tools
- PTT configuration for packet interfaces

### `/elmer-aprs` — APRS

Configures APRS receive, igate, and digipeating:
- **Direwolf** — as an APRS igate or digipeater
- **Xastir** — GUI APRS client with map display
- **aprx** — lightweight igate/digipeater daemon
- Beacon configuration

### `/elmer-sdr` — Software defined radio

Sets up SDR hardware and software:
- **RTL-SDR** — driver installation, udev rules for dongle access
- **GQRX** — configuration and use
- **SDR++** — setup and plugin configuration

### `/elmer-logging` — QSO logging

Sets up logging software and manages log data:
- **CQRLOG** — install, configure, and use the most popular Linux logging app
- **ADIF** — import/export log files for sharing between applications
- Contest logging setup

## Setup

Run `/elmer-setup` first. It will:

1. Detect your Linux distro, audio stack, and package manager
2. Ask about your HF and VHF/UHF rigs
3. Detect installed ham radio software
4. Check for existing udev rules
5. Write your config to `~/.config/elmer/config.yaml`

After setup, just run `/elmer` and describe what you want to work on — Elmer routes to the right skill automatically.

## Config file

The config lives at `~/.config/elmer/config.yaml`. It tracks your operator info, rig hardware, audio devices, udev status, and which software is installed. All skills read it on startup. You can edit it by hand or re-run `/elmer-setup` to update specific sections.

See `elmer/config-template.yaml` in this repo for the full schema with comments.

## Supported hardware (examples)

| Category | Examples |
|---|---|
| HF rigs | Yaesu FT-710, FT-991A, FT-857D; Icom IC-7300, IC-705; Kenwood TS-590SG |
| VHF/UHF rigs | Any rig with serial CAT or audio interface |
| Interfaces | Digirig Lite, SignaLink USB, RigBlaster |
| SDR dongles | RTL-SDR and compatible hardware |

## Supported software

fldigi · flrig · Hamlib · WSJT-X · JS8Call · JS8 Spotter · Direwolf · Pat · ARDOP · Vara HF (via Bottles) · GQRX · SDR++ · Xastir · aprx · CQRLOG

## Supported Linux distros

Debian · Ubuntu · MX Linux · Arch Linux · Fedora · and other distros using apt, dnf, pacman, or zypper
