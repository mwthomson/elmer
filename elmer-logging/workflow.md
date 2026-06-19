# Elmer Logging — QSO Logging, ADIF, CQRLOG, Contest Logging

Set up and use amateur radio logging software on Linux.

## STARTUP

Read `~/.config/elmer/config.yaml`. If missing, direct to `/elmer-setup`.

Ask what they want to work on:
1. Install and configure CQRLOG
2. Work with ADIF files (import/export/convert)
3. Contest logging
4. Connect logging to digital mode software (fldigi, WSJT-X, JS8Call)
5. Troubleshoot logging issues

---

## SECTION A — CQRLOG

CQRLOG is the most full-featured Linux logging application with rig control integration.

### Install

Check config `software.cqrlog.installed`. If false:
```bash
# Debian/Ubuntu/MX Linux:
sudo apt install cqrlog

# Arch:
# AUR: yay -S cqrlog
# or build from source: github.com/ok2cqr/cqrlog

# Fedora:
sudo dnf install cqrlog
```

CQRLOG requires MySQL/MariaDB:
```bash
# Check if MariaDB is installed:
which mysqld mariadbd 2>/dev/null || sudo apt install mariadb-server
```

### First-run setup

On first launch, CQRLOG creates the database:
- Database host: localhost
- Database user: cqrlog
- Database password: set during first run

CQRLOG will prompt to create a new log — give it a name (e.g., "Main Log 2025").

### Configure CQRLOG

**File → Preferences → Program:**
- Your callsign (from config)
- Your locator (grid from config)
- DXCC country file: update via Help → Update DXCC database

**File → Preferences → Hamlib:**
- Enable Hamlib if you want rig control from CQRLOG
- Rig: select your rig type
- Device: CAT port from config
- Or use "localhost" and port 4532 for rigctld

**File → Preferences → UDP:**
- Enable UDP logging (for receiving QSOs from fldigi/WSJT-X)
- Default port: 2237

### Log a QSO manually

In the QSO entry area:
- Frequency auto-fills from rig if connected
- Enter callsign → Tab fills in DXCC/prefix info automatically
- Enter RST sent/received
- Mode auto-fills from rig
- Press Enter to save

### CQRLOG tips
- DX cluster: File → Preferences → DX Cluster for live spotting
- Awards tracking: View → DXCC, IOTA, etc.
- QSL labels: File → Print QSL labels

---

## SECTION B — ADIF FILE OPERATIONS

ADIF (Amateur Data Interchange Format) is the universal log exchange format.

### View ADIF file contents

```bash
# ADIF files are plain text — can open in any text editor
less ~/Desktop/log.adif
grep "CALL" ~/Desktop/log.adif | head -20
```

### Convert ADIF to CSV (for spreadsheets)

```bash
python3 << 'EOF'
import re, csv, sys

with open(sys.argv[1]) as f:
    data = f.read()

# Find all records
records = re.findall(r'((?:<[^>]+>[^<]*)*)<EOR>', data, re.IGNORECASE)
fields = []
for rec in records:
    d = {}
    for m in re.finditer(r'<([^:>]+)(?::\d+)?(?::[^>]*)?>([^<]*)', rec, re.IGNORECASE):
        d[m.group(1).upper()] = m.group(2).strip()
    if d:
        fields.append(d)

if not fields:
    print("No records found")
    sys.exit(1)

all_keys = sorted(set(k for r in fields for k in r))
with open('log_export.csv', 'w', newline='') as f:
    w = csv.DictWriter(f, fieldnames=all_keys, extrasaction='ignore')
    w.writeheader()
    w.writerows(fields)
print(f"Exported {len(fields)} QSOs to log_export.csv")
EOF [adif filename]
```

### Upload to LoTW (ARRL Logbook of the World)

```bash
# tqsl is the LoTW signing tool
sudo apt install tqsl  # or download from arrl.org/tqsl
tqsl  # GUI tool for signing and uploading ADIF files
```

### Upload to QRZ.com logbook

QRZ.com accepts direct ADIF upload via their website. Export from CQRLOG:
- File → Export → ADIF → save file → upload at qrz.com/logbook

---

## SECTION C — DIGITAL MODE SOFTWARE INTEGRATION

### fldigi → CQRLOG

fldigi can send QSO data to CQRLOG via UDP:

In fldigi:
- Configure → Logging → enable "UDP logger"
- Host: localhost
- Port: 2237 (CQRLOG default)

In CQRLOG:
- File → Preferences → UDP → enable UDP listener on port 2237
- QSOs worked in fldigi automatically appear in CQRLOG entry form

### WSJT-X → CQRLOG

WSJT-X also supports UDP logging:

In WSJT-X:
- File → Settings → Reporting
- UDP server: localhost
- UDP port: 2237
- Enable "Accept UDP requests"

WSJT-X sends QSO data after each contact.

### JS8Call → CQRLOG

JS8Call supports UDP logging similarly:
- File → Settings → Reporting → UDP server → localhost:2237

---

## SECTION D — CONTEST LOGGING

### Tucnak (VHF/UHF contest logger)

```bash
sudo apt install tucnak
```

Good for VHF/UHF contests (ARRL September VHF, etc.)

### TLF (contest logger, terminal-based)

```bash
sudo apt install tlf
```

TLF is a powerful terminal-based contest logger supporting CW, SSB, and digital modes.

### fldigi contest mode

fldigi has built-in contest modes:
- Configure → Contest → select contest type
- Macros auto-fill exchange information

### General contest tips
- Set up macros before the contest
- Test rig control and logging integration the day before
- Sync to NTP for accurate timestamps
- Export to Cabrillo format for submission: most logging apps support this

---

## SECTION E — BACKUP AND SYNC

### Back up CQRLOG database

```bash
# Export to ADIF (portable backup):
# File → Export → ADIF → Full log

# Or dump the MySQL database directly:
mysqldump -u cqrlog -p[password] cqrlog > ~/cqrlog-backup-$(date +%Y%m%d).sql
```

### Restore CQRLOG database

```bash
mysql -u cqrlog -p[password] cqrlog < ~/cqrlog-backup-YYYYMMDD.sql
```

### Sync across machines

If you operate from multiple stations:
- Export to ADIF after each session
- Import ADIF into the other machine's log
- Or run CQRLOG's database on a shared server
