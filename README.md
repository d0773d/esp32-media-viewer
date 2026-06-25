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
  ESP32MediaViewerCompanion-Web.exe
  ESP32MediaViewerCompanion-Full.exe
```

`ESP32MediaViewerCompanion-Web.exe` is the small downloader app. It opens a Downloads window where users can choose All or Custom and install runtime files to AppData, the app folder, or a custom folder.

`ESP32MediaViewerCompanion-Full.exe` is the offline app. It has the runtime payload embedded and extracts tools and firmware on first launch.

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
