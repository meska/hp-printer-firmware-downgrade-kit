# HP OfficeJet Pro 7730 - Firmware Downgrade Guide

## Overview

This guide documents how to downgrade the HP OfficeJet Pro 7730 firmware to
version **1851A**, which restores compatibility with third-party and
refurbished ink cartridges. Newer HP firmware versions block non-HP cartridges
via "dynamic security".

- **Target version**: `ELLISXLP1N003.1851A.00` (2018-12-17)
- **Firmware file**: `OJP7730_1851A.ful2` (75 MB, included in this folder)
- **Original source**: `https://www.healthy-computers.com/HP/firmware/OJP7730_1851A.exe`

## Important Compatibility Note

The HP OfficeJet Pro 7730 does **not** use the same firmware family as the
7740.

- 7730 firmware family: `ELLISXLP1N003`
- 7740 firmware family: `EDWINXPP1N002`

Do **not** flash `OJP7740_1849A.ful2` to a 7730. Use only the 7730 firmware
file included here.

## Prerequisites

1. **Disable automatic firmware updates** on the printer:
   - Open the printer's web interface (EWS) at `http://<printer-ip>`
   - Navigate to **Settings > Firmware Update**
   - Set automatic checks and automatic updates to **disabled**
   - This prevents HP from re-upgrading the firmware after the downgrade

2. **Note your printer's IP address**:
   - Check the front panel under network settings, or
   - Use your router / DHCP lease table, or
   - Discover it with `dns-sd -B _http._tcp local.` on macOS

3. **Verify the current model and firmware**:

```bash
curl --compressed -fsSL "http://<printer-ip>/DevMgmt/ProductConfigDyn.xml" | grep -E 'Revision|MakeAndModel|SKUIdentifier'
```

Expected model markers for the 7730 include:

- `OfficeJet Pro 7730 Wide Format All-in-One`
- `SKUIdentifier>7730`
- A firmware revision in the `ELLISXLP1N003.*` family

## Step 1: Disable Auto-Updates via LEDM

Use the LEDM API so the settings are definitely written, even if the control
panel UI changes across firmware versions.

```bash
curl -k -X PUT "https://<printer-ip>/FirmwareUpdate/WebFWUpdate/Config" \
  -H "Content-Type: text/xml" \
  --data-binary @- <<'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<fwudyn:FirmwareUpdateConfig xmlns:fwudyn="http://www.hp.com/schemas/imaging/con/ledm/firmwareupdatedyn/2010/12/12"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://www.hp.com/schemas/imaging/con/ledm/firmwareupdatedyn/2010/12/12 ../../schemas/FirmwareUpdateDyn.xsd">
  <fwudyn:AutomaticCheck>disabled</fwudyn:AutomaticCheck>
  <fwudyn:AutomaticUpdate>disabled</fwudyn:AutomaticUpdate>
  <fwudyn:UpdateLockOption>enabled</fwudyn:UpdateLockOption>
  <fwudyn:UpdateLockState>on</fwudyn:UpdateLockState>
</fwudyn:FirmwareUpdateConfig>
EOF
```

Verify:

```bash
curl --compressed -fsSL "http://<printer-ip>/FirmwareUpdate/WebFWUpdate/Config"
curl --compressed -fsSL "http://<printer-ip>/FirmwareUpdate/FWUpdateJob/Config"
```

## Step 2: Switch Printer to Recovery Mode

Recent HP firmware such as `2602A` blocks downgrades in normal mode. The key is
to switch the printer into **recovery mode** first via the LEDM API. This
bypasses the version check.

```bash
curl -k -X PUT "https://<printer-ip>/FirmwareUpdate/FirmwareMode" \
  -H "Content-Type: text/xml" \
  --data-binary @- <<'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<fwudyn:FirmwareMode xmlns:fwudyn="http://www.hp.com/schemas/imaging/con/ledm/firmwareupdatedyn/2010/12/12"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://www.hp.com/schemas/imaging/con/ledm/firmwareupdatedyn/2010/12/12 ../../schemas/FirmwareUpdateDyn.xsd">recovery</fwudyn:FirmwareMode>
EOF
```

The printer will reboot. Wait about 30 seconds for it to come back online.

Verify recovery mode:

```bash
curl --compressed -fsSL "http://<printer-ip>/FirmwareUpdate/FirmwareMode"
```

It should return `recovery` instead of `normal`.

## Step 3: Send the Firmware

Once in recovery mode, send the `.ful2` file via netcat on port `9100`
(JetDirect):

```bash
nc -w 180 <printer-ip> 9100 < OJP7730_1851A.ful2
```

The printer will process the firmware and restart automatically. This may take
a few minutes.

## Step 4: Verify

After the printer restarts, confirm the firmware version:

- **Front panel**: Setup > Reports > Printer Status Report
- **EWS**: Open `http://<printer-ip>` in a browser
- **Command line**:

  ```bash
  curl --compressed -fsSL "http://<printer-ip>/DevMgmt/ProductConfigDyn.xml" | grep Revision
  ```

It should show:

```text
ELLISXLP1N003.1851A.00
```

Also verify that the firmware mode is back to `normal` and automatic updates
remain disabled:

```bash
curl --compressed -fsSL "http://<printer-ip>/FirmwareUpdate/FirmwareMode"
curl --compressed -fsSL "http://<printer-ip>/FirmwareUpdate/WebFWUpdate/Config"
curl --compressed -fsSL "http://<printer-ip>/FirmwareUpdate/FWUpdateJob/Config"
```

## Quick Reference

Replace `192.168.1.134` with your printer's IP:

```bash
# 1. Disable web-triggered auto-updates
curl -k -X PUT "https://192.168.1.134/FirmwareUpdate/WebFWUpdate/Config" \
  -H "Content-Type: text/xml" \
  --data-binary @- <<'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<fwudyn:FirmwareUpdateConfig xmlns:fwudyn="http://www.hp.com/schemas/imaging/con/ledm/firmwareupdatedyn/2010/12/12"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://www.hp.com/schemas/imaging/con/ledm/firmwareupdatedyn/2010/12/12 ../../schemas/FirmwareUpdateDyn.xsd">
  <fwudyn:AutomaticCheck>disabled</fwudyn:AutomaticCheck>
  <fwudyn:AutomaticUpdate>disabled</fwudyn:AutomaticUpdate>
  <fwudyn:UpdateLockOption>enabled</fwudyn:UpdateLockOption>
  <fwudyn:UpdateLockState>on</fwudyn:UpdateLockState>
</fwudyn:FirmwareUpdateConfig>
EOF

# 2. Switch to recovery mode
curl -k -X PUT "https://192.168.1.134/FirmwareUpdate/FirmwareMode" \
  -H "Content-Type: text/xml" \
  --data-binary @- <<'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<fwudyn:FirmwareMode xmlns:fwudyn="http://www.hp.com/schemas/imaging/con/ledm/firmwareupdatedyn/2010/12/12"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://www.hp.com/schemas/imaging/con/ledm/firmwareupdatedyn/2010/12/12 ../../schemas/FirmwareUpdateDyn.xsd">recovery</fwudyn:FirmwareMode>
EOF

# 3. Wait for reboot (~30 seconds)
sleep 30

# 4. Confirm recovery mode
curl --compressed -fsSL "http://192.168.1.134/FirmwareUpdate/FirmwareMode"

# 5. Send firmware
nc -w 180 192.168.1.134 9100 < OJP7730_1851A.ful2

# 6. Verify (after printer restarts)
curl --compressed -fsSL "http://192.168.1.134/DevMgmt/ProductConfigDyn.xml" | grep Revision
```

## Extracting `.ful2` Firmware from the HP `.exe`

HP distributes firmware as Windows `.exe` installers, but the actual firmware
payload is a `.ful2` file embedded inside. This method works for other HP
printer models too, as long as you use the firmware package that matches the
device family exactly.

### Step 1: Download the `.exe` installer

```bash
curl -L -o /tmp/OJP7730_1851A.exe \
  "https://www.healthy-computers.com/HP/firmware/OJP7730_1851A.exe"
```

### Step 2: Find the embedded `.ful2` name

```bash
strings /tmp/OJP7730_1851A.exe | grep '\.ful2'
```

Expected output:

```text
jellis_dist_lp1_003.1851A_nonassert_appsigned_lbi_rootfs_secure_signed.ful2
```

### Step 3: Extract the `.ful2` payload

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

with open(sys.argv[1], "rb") as f:
    data = f.read()

pjl_marker = b"%-12345X@PJL"
uel_marker = b"%-12345X"

pos = data.find(pjl_marker)
if pos == -1:
    print("ERROR: No PJL header found in file.")
    sys.exit(1)

fw_start = pos - 1 if pos > 0 and data[pos - 1:pos] == b"\x1b" else pos
fw_end = data.rfind(uel_marker) + len(uel_marker)
firmware = data[fw_start:fw_end]

with open(sys.argv[2], "wb") as out:
    out.write(firmware)

print(f"Extracted {len(firmware)} bytes ({len(firmware) / 1024 / 1024:.1f} MB)")
print(f"Saved to: {sys.argv[2]}")

header = firmware[:500].decode("ascii", errors="ignore")
for line in header.split("\\n"):
    if "VERSION" in line or "MODEL" in line:
        print(line.strip())
```

Example:

```bash
python3 extract_firmware.py /tmp/OJP7730_1851A.exe /tmp/OJP7730_1851A.ful2
```

Expected output:

```text
Extracted 78191818 bytes (74.6 MB)
Saved to: /tmp/OJP7730_1851A.ful2
@PJL COMMENT MODEL=HP Officejet Pro X577dw MFP
@PJL COMMENT VERSION=ELLISXLP1N003.1851A.00
```

## Alternative Methods

### USB Flash Drive

1. Format a USB drive as **FAT32**
2. Copy `OJP7730_1851A.ful2` to the root of the drive
3. Plug it into the printer's USB port

### Windows Installer

1. Download `OJP7730_1851A.exe` from the source URL above
2. Run it and select your printer
3. If you get "Not Applicable": extract with 7-Zip, edit `EnterpriseDU.ini`,
   change `VerifyDownloadID` from `1` to `0`, then run `EnterpriseDU.exe`

## Useful API Endpoints

| Endpoint | Method | Description |
|---|---|---|
| `/DevMgmt/ProductConfigDyn.xml` | GET | Printer info and firmware version |
| `/FirmwareUpdate/FirmwareMode` | GET/PUT | Check or change firmware mode |
| `/FirmwareUpdate/FWUpdateJob/Config` | GET/PUT | Update lock settings |
| `/FirmwareUpdate/WebFWUpdate/Config` | GET/PUT | Auto-update settings |
| `/FirmwareUpdate/FirmwareUpdateCap.xml` | GET | Firmware update capabilities |

All PUT endpoints require HTTPS (`https://<printer-ip>/...`) with `-k` for the
self-signed certificate.

## Sources

- https://www.healthy-computers.com/HP/OJP7730.htm
- https://www.healthy-computers.com/firmware_downgrades.htm
- https://github.com/lionpeloux/hp-printer-firmware-downgrade-kit
