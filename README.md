# SimpleMtkWlan — `mtk-dev` branch (experimental, working on one MT7921 laptop)

This is a fork of [laobamac/SimpleMtkWlan](https://github.com/laobamac/SimpleMtkWlan),
a macOS kernel extension for MediaTek MT792x PCIe Wi-Fi cards. The `mtk-dev`
branch starts from upstream `main` plus the open
[PR #5 by mavizao](https://github.com/laobamac/SimpleMtkWlan/pull/5) and adds
the fixes listed below. With these changes the driver connects to a WPA2
network, gets an IP address via DHCP and passes real traffic on the machine it
was tested on.

> **Status: experimental.** It has been tested on exactly one machine and one
> access point. It may panic or hang your Mac. Keep a bootable backup EFI (for
> example a USB stick) before trying it.

> **You need two pieces:**
> 1. **`SimpleMtkWlan.kext`** (this repository) — the driver, loaded by OpenCore.
> 2. **`SimpleMTK.app`** — the menu bar client from
>    [laobamac/SimpleMTKClient](https://github.com/laobamac/SimpleMTKClient).
>
> The driver does **not** plug into the macOS Wi-Fi menu or System Settings ->
> Wi-Fi. Scanning, choosing a network and entering the password are done only
> in SimpleMTK. Without the client the kext loads but never connects. See
> [SimpleMTK client app](#simplemtk-client-app-required).

## Who did the work (please read)

All code changes, debugging and diagnosis on this branch after PR #5, and this
README, were done by **Claude Code** (Anthropic's AI coding agent, model Claude
Opus 5.5) at the request of the fork owner,
[@KyatNguyen](https://github.com/KyatNguyen). The fork owner did not write the
code. Their part was hardware testing: installing every build on their
hackintosh, rebooting and reproducing the connection after each change, and
deciding what to try next.

The commits on this branch show `KyatNguyen` as git author because Claude Code
committed with the local git identity on the owner's machine. Every one of
those commits carries a `Co-Authored-By: Claude` trailer. The 9 commits by
**mavizao** are PR #5, unchanged.

The driver itself is the work of the people listed under [Credits](#credits).

## Tested setup

| Item | Value |
|---|---|
| Laptop | ASUS TUF Gaming F15 FX506LHB (Comet Lake, HM470) |
| Wi-Fi card | MediaTek MT7921, PCI ID `14c3:7961`, subsystem `1a3b:4680` |
| macOS | 15.8.1 (24H32), OpenCore (DEBUG build) |
| Access point | WPA2-PSK, CCMP, 5 GHz channel 153, 20 MHz |
| Result | DHCP OK, ping gateway ~1 ms, ~13.5 Mbit/s down, ~2.7 Mbit/s up |

MT7921K (`14c3:0608`) and MT7922 (`14c3:7922`) are still matched by the kext as
upstream does, but **they were not tested with these changes**.

## What works / what does not

Works on the tested setup:

- firmware download, scan, authentication, association
- WPA2-PSK 4-way handshake, DHCP, normal IPv4 traffic
- the interface appears as an **Ethernet** port (e.g. `en2`), not as Wi-Fi

Known limitations:

- **Encryption is done in software** (net80211 CCMP). With hardware encryption
  the firmware raises a system error (SER, `STOP_DMA`) after a few data frames;
  that is not fixed yet.
- **TX uses a fixed 6 Mbit/s rate**, so upload is slow.
- **Scanning is disabled while connected**; the client's network list does not
  refresh until you disconnect.
- No recovery if the firmware raises SER: Wi-Fi stops until reboot.
- Only WPA2-PSK/CCMP on 5 GHz was tested. WPA3, TKIP, enterprise, 2.4 GHz,
  sleep/wake and long-term stability were not.
- The branch still contains a lot of diagnostic logging (see
  [Reading the driver log](#reading-the-driver-log)).

## Download

Prebuilt kexts come from this fork's GitHub Actions workflow
(`Build SimpleMtkWlan kext`, branch `mtk-dev`):

1. Open the [workflow runs](https://github.com/KyatNguyen/SimpleMtkWlan/actions/workflows/build-kext.yml)
   and pick the latest successful run on `mtk-dev`.
2. Download the artifact `SimpleMtkWlan-Release-<commit>` (you must be signed
   in to GitHub). It contains `SimpleMtkWlan.kext`, `build.log` and
   `SHA256.txt`.
3. Artifacts expire after 90 days; you can always rebuild (see below).

The build tested in the table above is commit `eaedc9e`.

The client app is downloaded separately, see the next section.

## SimpleMTK client app (required)

SimpleMTK is laobamac's menu bar app (a HeliPort fork) that talks to the kext
through its IOKit user client. It was used **unmodified** for all testing.

### Get it

laobamac does not publish a prebuilt release, so either:

- **Download a build:** open the
  [`Build SimpleMTK app` runs](https://github.com/KyatNguyen/SimpleMTKClient/actions/workflows/build-app.yml)
  of the [KyatNguyen/SimpleMTKClient](https://github.com/KyatNguyen/SimpleMTKClient)
  fork and download the artifact `SimpleMTK-Release-<commit>` (signed in to
  GitHub). It contains `SimpleMTK.zip` (x86_64, ad-hoc signed, not notarized)
  and `SHA256.txt`. That fork only adds this build workflow; the app source is
  laobamac's. The tested build is commit `4f184e6`,
  `SimpleMTK.zip` SHA256 `056c526765423f1a862342e29550ff42cfb263e0db0be41f13bd7f0bbecfba5a`.
- **Or build it yourself** from
  [laobamac/SimpleMTKClient](https://github.com/laobamac/SimpleMTKClient) with
  Xcode:

  ```bash
  xcodebuild -scheme SimpleMTK -configuration Release ARCHS=x86_64 \
    ONLY_ACTIVE_ARCH=NO CODE_SIGN_IDENTITY="" CODE_SIGNING_REQUIRED=NO \
    -derivedDataPath build build
  ```

### Install

1. Unzip `SimpleMTK.zip` and move `SimpleMTK.app` to `/Applications`.
2. Open it. The app is not notarized, so macOS 15 blocks the first launch of a
   copy downloaded with a browser: open System Settings -> Privacy & Security
   and click **Open Anyway** for SimpleMTK (or remove the quarantine flag with
   `xattr -dr com.apple.quarantine /Applications/SimpleMTK.app`).
3. A Wi-Fi icon appears in the menu bar. To start it automatically, use the
   **Launch At Login** item in the SimpleMTK menu.

### Use

1. Click the SimpleMTK menu bar icon and wait for the network list.
2. Pick your network, enter the password, and connect **once**. Do not press
   connect again while it is connecting.
3. When connected, the connection appears in System Settings -> Network as an
   **Ethernet** interface (for example `en2`) and gets its address by DHCP.
4. The menu bar icon reflects whether the Mac has Internet through **any**
   interface (for example USB tethering), so check `ifconfig en2` if you want
   to know whether Wi-Fi itself is up.
5. While connected the network list does not refresh (scanning is disabled in
   RUN on this branch). Disconnect in SimpleMTK to scan again.
6. When it is not connected, the client tries your saved networks
   automatically. If it picks another saved network first, the driver handles
   that, but the first connection can take longer.

## Install (OpenCore)

1. Copy `SimpleMtkWlan.kext` to `EFI/OC/Kexts/`.
2. Add it to `Kernel -> Add` in `config.plist` (no Lilu dependency; order does
   not matter):

   ```xml
   <dict>
       <key>Arch</key>
       <string>x86_64</string>
       <key>BundlePath</key>
       <string>SimpleMtkWlan.kext</string>
       <key>Comment</key>
       <string>MediaTek MT7921 Wi-Fi (mtk-dev)</string>
       <key>Enabled</key>
       <true/>
       <key>ExecutablePath</key>
       <string>Contents/MacOS/SimpleMtkWlan</string>
       <key>MaxKernel</key>
       <string></string>
       <key>MinKernel</key>
       <string></string>
       <key>PlistPath</key>
       <string>Contents/Info.plist</string>
   </dict>
   ```

3. If your EFI hides the Wi-Fi card (for example an SSDT such as
   `SSDT-Disable_Network_RP14.aml` that spoofs its IDs to `ffff:ffff`), disable
   it, otherwise the kext never matches.
4. Install the **SimpleMTK** client app (see
   [SimpleMTK client app](#simplemtk-client-app-required)), reboot, then pick
   your network in SimpleMTK and connect **once**. The connection shows up as
   an Ethernet interface in System Settings -> Network.

Check that the kext attached:

```bash
ioreg -rc SimpleMtkWlan -d1 | head -3
```

## Reading the driver log

On the test machine `printf`/`IOLog` from this kext reached neither the unified
log nor `dmesg`, so the driver keeps its own log buffer and publishes it as the
`SimpleMtkWlanLog` property of the PCI device (refreshed about every 2 s while
the interface is up). Plain `ioreg` only prints the first line of it; this
prints all of it:

```bash
ioreg -a -r -n PXSX -l | python3 -c 'import plistlib,sys
def w(o):
    if isinstance(o, dict):
        if "SimpleMtkWlanLog" in o: print(o["SimpleMtkWlanLog"])
        for v in o.values(): w(v)
    elif isinstance(o, list):
        for v in o: w(v)
w(plistlib.loads(sys.stdin.buffer.read()))'
```

## Build from source

Requires Xcode and [MacKernelSDK](https://github.com/acidanthera/MacKernelSDK)
inside the project directory:

```bash
git clone -b mtk-dev https://github.com/KyatNguyen/SimpleMtkWlan.git
cd SimpleMtkWlan
git clone https://github.com/acidanthera/MacKernelSDK.git
xcodebuild -project SimpleMtkWlan.xcodeproj -scheme SimpleMtkWlan \
  -configuration Release ARCHS=x86_64 ONLY_ACTIVE_ARCH=NO CODE_SIGNING_ALLOWED=NO \
  HEADER_SEARCH_PATHS="$PWD/MacKernelSDK/Headers" \
  KERNEL_EXTENSION_HEADER_SEARCH_PATHS="$PWD/MacKernelSDK/Headers" \
  -derivedDataPath build build
```

The CI workflow (`.github/workflows/build-kext.yml`) does the same on
`macos-latest` with MacKernelSDK pinned to a fixed commit.

## Changes on this branch (on top of upstream + PR #5)

Fixes:

| Commit | Change |
|---|---|
| `6a1253b` | Firmware scatter chunk + MCU header exceeded a 4 KB mbuf and needed a 16 KB cluster (ENOMEM), so firmware never loaded on this machine. Chunks are now capped at 4096 bytes including the header, like Linux mt76. |
| `692092a` | Leaving RUN enabled firmware deep sleep while the BSS was still active; the next MCU command timed out and the MCU stopped answering. The BSS/STA are now removed first and the chip stays awake. |
| `2019821` | A repeated associate request for the SSID already in RUN sent a deauth and rejoined; it is now ignored. |
| `7c80473` | A failed earlier attempt left a STA record on the same WTBL index; firmware rejected the new one (`0x10003`) and RX from the AP was never matched (WLAN index 1023). The stale record is removed before adding the AP. |
| `7c62f89` | Insert the CCMP header for protected 802.11-format TX frames (used only when no software key context exists). |
| `eaedc9e` | **Software CCMP for TX and RX** next to the hardware key. This made traffic work: with hardware encryption the firmware raised SER after ~6 data frames. |

Workaround:

| Commit | Change |
|---|---|
| `e63b73f` | Do not start a hardware scan while associated (a scan in RUN preceded an MCU hang). |

Diagnostics (to be cleaned up later): `e6b9a74`, `a51babc`, `2a82af9`,
`feefba0`, `f14062c`, `8445455`, `1db97f7`, `d0f4a4f`, `44d0824`, `4b8103e`,
`d09add6` — in-driver log buffer published to IORegistry, 802.11 state and MCU
event tracing, DMA ring / interrupt / firmware-state dumps.

## License

GPLv2, same as upstream. Individual files keep their original GPL/BSD/ISC
notices.

## Credits

- **laobamac (王孝慈)** — SimpleMtkWlan and the SimpleMTK client
- **mavizao** — PR #5 (association context, TX serialization, packet ownership)
- **zxystd (钟先耀)** — the itlwm driver framework this is based on
- **OpenBSD** `mwx(4)` and net80211 authors — MT7921 driver code and 802.11 stack
- **Linux mt76** authors — used as the reference for MT7921 behaviour
- **Claude Code** (Anthropic) — the changes on this branch, see above

---

## Original upstream README

<!--
  README.md
  SimpleMtkWlan

  Created by laobamac (王孝慈) on 2026/5/5.
  Copyright © 2026 laobamac (王孝慈). All rights reserved.
-->

# SimpleMtkWlan

SimpleMtkWlan is an experimental macOS kernel extension for MediaTek MT792x PCIe
wireless adapters.

Author: laobamac (王孝慈)

## Status

This project is under active hardware bring-up. Current development focuses on:

- MT7921/MT7922 PCIe probe and attach
- Firmware loading
- DMA queue setup
- Hardware scan support
- Association/authentication TX path debugging

The driver is not release-stable yet.

## Supported Hardware

Currently targeted device:

- MediaTek MT7921, PCI ID `14c3:7961`
- MediaTek MT7921K/RZ608, PCI ID `14c3:0608`
- MediaTek MT7922, PCI ID `14c3:7922`

Other MediaTek devices are not currently supported.

## Building

Requirements:

- macOS with Xcode
- Kernel SDK headers matching the local development setup
- The project-local firmware generation script and embedded firmware data

Build:

```sh
xcodebuild -project SimpleMtkWlan.xcodeproj -scheme SimpleMtkWlan -configuration Debug build
```

The debug kext will be produced by Xcode as `SimpleMtkWlan.kext`.

## Repository Layout

- `SimpleMtkWlan/` - kernel extension entry point and MediaTek HAL
- `SimpleMtkWlan/hal_mwx/` - MT7921/MT7922 driver implementation
- `mtk80211/` - compatibility, OpenBSD net80211, and support code
- `include/` - shared driver interfaces and generated firmware table
- `scripts/` - firmware generation helpers

## License

SimpleMtkWlan is distributed under GPLv2. See `LICENSE`.

Some source files include additional or more specific license notices inherited
from upstream GPL/BSD/OpenBSD/NetBSD/FreeBSD components. Those notices are
preserved in place and apply to their respective files.

## Credits

See `CREDITS.md`.
