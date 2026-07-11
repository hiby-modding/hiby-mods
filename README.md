# HiBy Mods

A collection of custom firmware modifications for HiBy digital audio players, along with documentation of the proprietary OTA firmware format for the benefit of the modding community.

This project is part of the [hiby-modding](https://github.com/hiby-modding) organization. Also see:

- [hiby_os_crack](https://github.com/hiby-modding/hiby_os_crack) by Tartarus6 — complementary firmware modding project with clean unpack/repack scripts, QEMU emulation setup, Windows support, and SOC hardware documentation.

[![Ko-fi](https://img.shields.io/badge/Ko--fi-Support-FF5E5B?logo=ko-fi&logoColor=white)](https://ko-fi.com/noisetta) [![GitHub Sponsors](https://img.shields.io/badge/Sponsor-GitHub-EA4AAA?logo=github-sponsors)](https://github.com/sponsors/noisetta)

**Maintenance status:** This project is currently maintained solely by [@noisetta](https://github.com/noisetta). If you'd like to support the project, Ko-fi and GitHub Sponsors donations go toward ongoing maintenance and development time. See Acknowledgements below for everyone who's helped build this project over time.

## What This Project Does

Provides custom firmware builds and tools for modifying HiBy OS on supported devices. Modifications are built on top of the stock firmware and distributed as ready-to-flash `.upt` files.

**Current modifications in v1.5:**
- Database Manager — new Settings menu entry to save/load the music library database to/from SD card without rescanning. Supports all 13 languages. Shows a "Done!" popup after each operation.
- PC Database Updater — Python script and GUI app to generate the music library database on your PC in seconds. Embeds and resizes cover art to 360×360px across all supported formats.
- Unified Theme — modernized UI including updated system dialogs, battery icon, WiFi/BT scanning dialogs, ADB icon on About page, and more
- Alphabetical sorting — multi-language article support, symbols sort to top, updated scrollbar
- Playlist menu fix — three-dot context menu now works correctly on all playlists
- Most Played playlist — tracks ordered by play count, accessible from the Playlists menu
- Bluetooth optimization — BT init time reduced from ~11s to ~7s
- Extended Unicode font support — adds rendering for scripts not included in the stock firmware
- ADB support — root shell access via USB, enabled through a hidden easter egg
- Custom firmware identifier — About page shows `1.5` to distinguish from stock

**Known Limitations:**
- Added Unicode scripts display left-to-right regardless of the script's natural reading direction — the HiBy OS text renderer does not support bidirectional text
- Characters appear in isolated forms and do not connect to each other — the OS renderer does not support contextual shaping
- Song limit (currently 50,000 tracks) patch is under investigation — will be included in a future release

## Tested On

- Device: HiBy R3 Pro II (r3proii)
- Base firmware: Stock HiBy OS (January 2026)
- Host OS used for building: Ubuntu / Pop!_OS (Linux)

> ⚠️ **Compatibility:** The pre-built firmware in this repository has only been
> tested on the HiBy R3 Pro II. Do not flash on other devices unless you have
> built and verified it yourself.

## Quick Install (Pre-built Firmware)

If you just want to flash the latest modifications without building from source:

1. Download `r3proii-v1.5-hmod.upt` from the [Releases](https://github.com/hiby-modding/hiby-mods/releases) section
2. Verify the checksum matches the one listed in the release notes
3. Copy it to the root of your SD card and rename it to `r3proii.upt`
4. Insert the SD card into your HiBy R3 Pro II
5. Hold **Volume Up** and press **Power** to enter the updater
6. Let it flash — it will say "Upgrading..." then "Succeeded"
7. Remove the SD card before the device reboots
8. Done

> **Note:** After flashing go to **Settings → UI Themes** and reselect your preferred theme to ensure the correct boot logo displays.

> **Recovery:** If something goes wrong, you can always restore by flashing the original stock firmware from HiBy's website using the same procedure.

## Using the Database Manager

Instead of waiting for the device to rescan your music library, generate the database on your PC in seconds:

1. Insert your SD card into your PC
2. Run `tools/Update_Database.py` or use the GUI app (`tools/HiBy_Database_Updater_GUI.py`)
3. Insert the SD card back into the device
4. Go to **Settings → Database Manager → Copy Database from SD**
5. The library reloads instantly — no rescan needed

## Enabling ADB

ADB can be enabled on any firmware version (including stock) without any modification:

1. Go to **Settings → About**
2. Tap the **"About"** text 10 times (or tap the ADB icon in the top-right corner when using the Unified Theme)
3. A message saying "Test Mode Turned On" will appear
4. Connect the device to your PC via USB
5. Run `adb devices` to confirm the connection

To disable ADB, tap "About" 10 times again.

> **Note:** ADB and USB mass storage share the same USB interface and cannot run simultaneously. While ADB is enabled the device will not appear as a storage drive.

## Building From Source

### Requirements

```bash
sudo apt install squashfs-tools xorriso fontforge python3-fontforge p7zip-full
```

### Using the Mod Tool (Recommended)

```bash
# 1. Clone this repository
git clone https://github.com/hiby-modding/hiby-mods
cd hiby-mods

# 2. Place your stock .upt firmware in the firmware/ folder

# 3. Run the mod tool
./tools/universal_mod_tool.sh
```

### Manual Build

```bash
# 1. Clone this repository
git clone https://github.com/hiby-modding/hiby-mods
cd hiby-mods

# 2. Download the stock firmware from HiBy's website
# Place it at: firmware/r3proii_stock.upt

# 3. Run the rebuild script
./tools/rebuild.sh
```

## Repository Structure

```text
hiby-mods/
├── binaries/      Patched HiBy player binaries and patch documentation
├── docs/          Firmware research, device guides, and screenshots
├── firmware/      Release and firmware-handling information
├── themes/        Community theme assets
├── tools/         Firmware-building and database utilities
├── LICENSE
├── NOTICE.md
└── README.md
```

Ready-to-flash `.upt` files are distributed through
[GitHub Releases](https://github.com/hiby-modding/hiby-mods/releases)
rather than stored directly in the source tree.

## Documentation

See [docs/FIRMWARE_FORMAT.md](docs/FIRMWARE_FORMAT.md) for complete documentation of the OTA update format, partition layout, boot sequence and hardware details.

See [docs/SCREENSHOTS.md](docs/SCREENSHOTS.md) for a guide on capturing screenshots from the device via ADB.

See [docs/FACTORY_MODE.md](docs/FACTORY_MODE.md) for documentation of the built-in factory diagnostic mode.

## Changelog

### v1.5
- Playlist menu fix — three-dot context menu now appears on all playlists and targets the correct entry
- DB Manager "Done!" popup — visual confirmation after Save/Copy operations
- Most Played playlist — new playlist showing tracks ordered by play count
- Bluetooth optimization — BT init reduced from ~11s to ~7s
- PC Database Updater — album art embed/resize to 360×360px for all supported formats, folder cover resize, GUI app added
- Unified Theme — ADB icon on About page, updated prompts, balance icon, Wi-Fi icon fix, create playlist page fix, battery icon update
- Spanish buffer overflow crash fixed
- All 13 language files updated with DB Manager and Most Played strings
- Factory diagnostic mode documented

### v1.4+
- Database Manager — load/save music library database via Settings menu, all 13 languages supported
- PC Database Updater — Python script and GUI app for fast library generation on PC
- Custom firmware identifier — About page now shows `1.4+`

### v1.3
- Updated Unified Theme — expanded system dialog modernization, new icons throughout
- ADB support — fixed USB gadget conflict enabling reliable ADB connectivity

### v1.2
- Unified Theme — modernized launcher, screensaver, playing screen, stream media icons, boot logo
- Updated alphabetical sorting — multi-language article support, symbols sort to top

### v1.1
- Extended Unicode font support for scripts not included in stock firmware
- Alphabetical sorting fix — entries starting with "The" now sort correctly

### v1.0
- Initial release
- Documented OTA firmware format
- Proof of concept custom firmware build

## Disclaimer

This firmware is provided as-is with no warranty. Flashing custom firmware carries a small risk of making your device temporarily unbootable. In the event of a boot loop, you can always recover by flashing the original stock firmware from HiBy's website using the Volume Up + Power method described above.

This project is not affiliated with or endorsed by HiBy Music.

## Contributing

Contributions are welcome. Areas where help would be valuable:

- Testing on other HiBy devices that share the same firmware format
- Investigating kernel source availability (check github.com/hiby-music)
- Adding support for additional Unicode scripts
- GUI and theme modifications
- RockBox Bluetooth/WiFi integration
- Further ADB exploration and live system research
- Song limit investigation and binary patch

## Acknowledgements

- [@Jepl4r](https://github.com/Jepl4r) — Unified Theme, sorting patch, Database Manager, playlist fix, Done dialog, PC database updater, mod tool (contributed through v1.5)
- [@Tartarus6](https://github.com/Tartarus6) — hiby_os_crack tooling, binary decompilation and collaboration (contributed through v1.5)
- [@greenturtle537](https://github.com/greenturtle537) — QEMU board support research
- Noto Naskh Arabic font by Google (OFL licensed)
- HiBy Music for making a great device
- The [head-fi.org](https://head-fi.org) community
- The Reddit [r/hiby](https://reddit.com/r/hiby) community
