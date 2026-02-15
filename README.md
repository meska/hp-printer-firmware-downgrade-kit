# HP Printer Firmware Downgrade Kit

Downgrade HP OfficeJet Pro 7740 firmware to version **1849A** to restore compatibility with third-party and refurbished ink cartridges.

HP pushes firmware updates that block non-HP cartridges via "dynamic security". This repo contains the tools and instructions to reverse that.

## What's Included

- `OJP7740_1849A.ful2` — firmware file (v1849A, 60 MB)
- `Downgrade_Guide.md` — detailed step-by-step guide (human and agent-readable)

## How It Works

1. **Switch the printer to recovery mode** via its HTTP API (bypasses downgrade protection)
2. **Send the older firmware** to the printer over the network
3. **Disable auto-updates** so HP doesn't re-upgrade it

That's it. Three steps, no special software needed — just `curl` and `nc` (netcat), available on Mac and Linux by default.

## Easiest Way: Use a Coding Agent

You don't need to be technical. Open any AI coding agent that has terminal access and tell it:

> Read the file Downgrade_Guide.md and help me downgrade my HP OfficeJet Pro 7740 printer. My printer IP is 192.168.1.XXX.

The guide is written so that an AI agent can follow it step by step, run the commands for you, and verify the result.

Examples of free coding agents that can do this:
- [Gemini CLI](https://github.com/google-gemini/gemini-cli) (free)
- [Claude Code](https://claude.ai/claude-code) (paid)

## Firmware Downloads

If you need to download the firmware `.exe` or `.dmg` for the OJP7740 (or other HP models):

- https://www.healthy-computers.com/HP/OJP7740.htm
- https://ybtoner.com/how-to-downgrade-hp-printer-firmware/
- https://printcopytech.net/download/hp-officejet-pro-7740-firmware-version-1849a/

## Adapting to Other HP Printers

The `Downgrade_Guide.md` includes a section on extracting `.ful2` firmware files from any HP firmware `.exe` installer using a Python script. The recovery mode API and flashing method work across HP models that support the LEDM interface.

## Disclaimer

**Use at your own risk.** Flashing firmware can potentially brick your printer if something goes wrong (power loss, network interruption, incompatible file). The authors of this repository are not responsible for any damage to your hardware. Make sure you understand what you are doing, or let a coding agent guide you through it carefully.

This repository is provided for educational and personal use. Downgrading firmware on a printer you own is legal, but verify your local regulations.

## License

MIT License

Copyright (c) 2025

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
