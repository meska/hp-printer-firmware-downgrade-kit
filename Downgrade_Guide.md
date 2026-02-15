# HP OfficeJet Pro 7740 - Firmware Downgrade Guide

## Overview

This guide documents how to downgrade the HP OfficeJet Pro 7740 firmware to version **1849A**, which allows the use of third-party / refurbished ink cartridges. Newer HP firmware versions block non-HP cartridges via "dynamic security".

- **Target version**: EDWINXPP1N002.1849A.00 (2018-12-06)
- **Firmware file**: `OJP7740_1849A.ful2` (60 MB, included in this folder)
- **Original source**: https://www.healthy-computers.com/HP/firmware/OJP7740_1849A.exe

## Prerequisites

1. **Disable automatic firmware updates** on the printer:
   - Open the printer's web interface (EWS) at `http://<printer-ip>`
   - Navigate to **Settings > Firmware Update**
   - Set automatic updates to **disabled**
   - This prevents HP from re-upgrading the firmware after downgrade

2. **Note your printer's IP address** (check the printer's front panel under Network settings)

## Step 1: Switch Printer to Recovery Mode

Recent HP firmware (2530A, 2602A, etc.) blocks downgrades in normal mode. The key is to switch the printer into **recovery mode** first via the LEDM API. This bypasses the version check.

```bash
curl -k -X PUT "https://<printer-ip>/FirmwareUpdate/FirmwareMode" \
  -H "Content-Type: text/xml" \
  -d '<?xml version="1.0" encoding="UTF-8"?>
<fwudyn:FirmwareMode xmlns:fwudyn="http://www.hp.com/schemas/imaging/con/ledm/firmwareupdatedyn/2010/12/12"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://www.hp.com/schemas/imaging/con/ledm/firmwareupdatedyn/2010/12/12 ../../schemas/FirmwareUpdateDyn.xsd">recovery</fwudyn:FirmwareMode>'
```

The printer will reboot. Wait about 30 seconds for it to come back online.

You can verify it's in recovery mode:

```bash
curl -k "https://<printer-ip>/FirmwareUpdate/FirmwareMode"
```

Should return `recovery` instead of `normal`.

## Step 2: Send the Firmware

Once in recovery mode, send the .ful2 file via netcat on port 9100 (JetDirect):

```bash
nc -w 100 <printer-ip> 9100 < OJP7740_1849A.ful2
```

The printer will process the firmware and restart automatically. This may take a few minutes.

## Step 3: Verify

After the printer restarts, confirm the firmware version:

- **Front panel**: Setup > Reports > Printer Status Report
- **EWS**: Open `http://<printer-ip>` in a browser
- **Command line**:
  ```bash
  curl -s "http://<printer-ip>/DevMgmt/ProductConfigDyn.xml" | grep Revision
  ```
  Should show: `EDWINXPP1N002.1849A.00`

## Quick Reference (copy-paste commands)

Replace `192.168.1.134` with your printer's IP:

```bash
# 1. Switch to recovery mode
curl -k -X PUT "https://192.168.1.134/FirmwareUpdate/FirmwareMode" \
  -H "Content-Type: text/xml" \
  -d '<?xml version="1.0" encoding="UTF-8"?>
<fwudyn:FirmwareMode xmlns:fwudyn="http://www.hp.com/schemas/imaging/con/ledm/firmwareupdatedyn/2010/12/12"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://www.hp.com/schemas/imaging/con/ledm/firmwareupdatedyn/2010/12/12 ../../schemas/FirmwareUpdateDyn.xsd">recovery</fwudyn:FirmwareMode>'

# 2. Wait for reboot (~30 seconds)
sleep 30

# 3. Send firmware
nc -w 100 192.168.1.134 9100 < ~/Desktop/HP7740/OJP7740_1849A.ful2

# 4. Verify (after printer restarts)
curl -s "http://192.168.1.134/DevMgmt/ProductConfigDyn.xml" | grep Revision
```

## Extracting .ful2 Firmware from HP .exe Installers

HP distributes firmware as Windows `.exe` installers, but the actual firmware payload is a `.ful2` file embedded inside. You can extract it on any OS (Mac, Linux, Windows) using Python. This method works for other HP printer models too — just adapt the filename.

### Step 1: Download the .exe installer

Find the firmware `.exe` for your printer model. Common sources:
- https://www.healthy-computers.com/HP/
- https://ybtoner.com/how-to-downgrade-hp-printer-firmware/
- HP's official support site (for current versions only)

Example for the OJP7740:
```bash
curl -L -o /tmp/OJP7740_1849A.exe \
  "https://www.healthy-computers.com/HP/firmware/OJP7740_1849A.exe"
```

### Step 2: Find the firmware filename inside the .exe

```bash
strings /tmp/OJP7740_1849A.exe | grep '\.ful2'
```

This will output something like:
```
edwin_dist_pp1_002.1849A_nonassert_appsigned_lbi_rootfs_secure_signed.ful2
```

### Step 3: Extract the .ful2 file

The firmware payload starts with a PJL (Printer Job Language) header: `ESC%-12345X@PJL`. The following Python script locates this marker and extracts everything from there to the last UEL (Universal Exit Language) marker.

```python
#!/usr/bin/env python3
"""
Extract .ful2 firmware from HP firmware updater .exe files.
Works for any HP printer model (OfficeJet, LaserJet, etc.)

Usage: python3 extract_firmware.py <input.exe> <output.ful2>
"""
import sys

if len(sys.argv) != 3:
    print(f"Usage: {sys.argv[0]} <input.exe> <output.ful2>")
    sys.exit(1)

with open(sys.argv[1], 'rb') as f:
    data = f.read()

# PJL start marker (ESC + %-12345X@PJL)
pjl_marker = b'%-12345X@PJL'
uel_marker = b'%-12345X'

# Find first PJL header
pos = data.find(pjl_marker)
if pos == -1:
    print("ERROR: No PJL header found in file.")
    sys.exit(1)

# Include the ESC byte before %-12345X if present
fw_start = pos - 1 if pos > 0 and data[pos - 1:pos] == b'\x1b' else pos

# Find the last UEL marker (end of firmware)
fw_end = data.rfind(uel_marker) + len(uel_marker)

firmware = data[fw_start:fw_end]
with open(sys.argv[2], 'wb') as out:
    out.write(firmware)

print(f"Extracted {len(firmware)} bytes ({len(firmware) / 1024 / 1024:.1f} MB)")
print(f"Saved to: {sys.argv[2]}")

# Show firmware version from PJL header
header = firmware[:500].decode('ascii', errors='ignore')
for line in header.split('\n'):
    if 'VERSION' in line or 'MODEL' in line:
        print(line.strip())
```

Save as `extract_firmware.py` and run:

```bash
python3 extract_firmware.py /tmp/OJP7740_1849A.exe /tmp/OJP7740_1849A.ful2
```

Output:
```
Extracted 62694842 bytes (59.8 MB)
Saved to: /tmp/OJP7740_1849A.ful2
@PJL COMMENT MODEL=HP Officejet Pro X577dw MFP
@PJL COMMENT VERSION=EDWINXPP1N002.1849A.00
```

### Step 4: Verify the extracted file

Check that the file starts with a valid PJL header:

```bash
head -c 300 /tmp/OJP7740_1849A.ful2 | strings
```

You should see:
```
%-12345X@PJL
@PJL COMMENT MODEL=<your printer model>
@PJL COMMENT VERSION=<firmware version>
@PJL COMMENT DATECODE=<date>
@PJL UPGRADE SIZE=<size>
```

### Adapting for Other Printer Models

This extraction method works for any HP printer that uses `.ful2` firmware files. The steps are identical — only the source `.exe` and filenames change. For example:

| Model | Example .exe | Firmware prefix |
|---|---|---|
| OfficeJet Pro 7740 | OJP7740_1849A.exe | edwin_dist_pp1_002 |
| OfficeJet Pro 8710 | OJP8710_XXXX.exe | varies |
| LaserJet Pro M404 | LJM404_XXXX.exe | varies |

The recovery mode API endpoint and netcat flash method also work across HP models that support the LEDM interface.

## Alternative Methods

### USB Flash Drive
1. Format a USB drive as **FAT32**
2. Copy `OJP7740_1849A.ful2` to the root of the drive
3. Plug it into the printer's USB port

### Windows Installer
1. Download `OJP7740_1849A.exe` from the source URL above
2. Run it and select your printer
3. If you get "Not Applicable": extract with 7-Zip, edit `EnterpriseDU.ini`, change `VerifyDownloadID` from `1` to `0`, then run `EnterpriseDU.exe`

## Useful API Endpoints

| Endpoint | Method | Description |
|---|---|---|
| `/DevMgmt/ProductConfigDyn.xml` | GET | Printer info & firmware version |
| `/FirmwareUpdate/FirmwareMode` | GET/PUT | Check or change firmware mode (normal/recovery) |
| `/FirmwareUpdate/FWUpdateJob/Config` | GET/PUT | Update lock settings |
| `/FirmwareUpdate/WebFWUpdate/Config` | GET/PUT | Auto-update settings |
| `/FirmwareUpdate/FirmwareUpdateCap.xml` | GET | Firmware update capabilities |

All PUT endpoints require HTTPS (`https://<printer-ip>/...`) with `-k` flag for self-signed cert.

## Sources

- https://www.healthy-computers.com/HP/OJP7740.htm
- https://mypcjobs.wordpress.com/2020/06/07/downgrading-a-hp-7740-firmware-because-of-ink-cartridge-problem/
- https://ybtoner.com/how-to-downgrade-hp-printer-firmware/
- https://h30434.www3.hp.com/t5/Printer-Setup-Software-Drivers/Roll-back-Firmware-on-HP-OfficeJet-Pro-7740/td-p/9243185
