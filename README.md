# ESP32 Media Viewer

Public artifact-only distribution repo for the ESP32 Media Viewer project.

This repo intentionally contains no source code. It only publishes:

- compiled companion app installers
- compiled ESP32-C6 firmware
- compiled ESP32-S3 firmware

The source code lives in private repositories.

## Companion App Installers

```text
installers/
  ESP32MediaViewerCompanion-Web-Setup.exe
  ESP32MediaViewerCompanion-Full-Setup.exe
```

`ESP32MediaViewerCompanion-Web-Setup.exe` installs the companion app, then lets users choose All or Custom runtime downloads for tools and firmware.

`ESP32MediaViewerCompanion-Full-Setup.exe` installs the companion app, tools, and firmware from an embedded offline payload.

If a common folder such as Desktop is selected during setup, the installer creates an `ESP32MediaViewerCompanion` subfolder instead of placing app files directly into that folder. If the user selects a newly-created empty folder, the installer uses that folder directly.

Setup installs an `Uninstall.exe` in the app folder and registers the app under Windows Apps & features for the current user. Start Menu shortcuts are enabled by default, Desktop shortcut creation is optional, and taskbar pinning should be done manually from the Start Menu shortcut after install.

Setup shows install progress with a percentage and current phase while it extracts files, downloads selected runtime files, and writes shortcuts.

The companion app requires users to accept the included [User Agreement](USER_AGREEMENT.md) before the app becomes usable.

## Compiled Firmware

```text
firmware_release/
  esp32c6/
    firmware_package.json
    bootloader.bin
    partition-table.bin
    esp32_media_viewer.bin
  esp32s3/
    firmware_package.json
    bootloader.bin
    partition-table.bin
    esp32_media_viewer.bin
```

The companion app uses each `firmware_package.json` to flash the correct board firmware and media partition offsets.
