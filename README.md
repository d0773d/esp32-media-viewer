# ESP32 Media Viewer

Public companion app and compiled firmware bundles for the Waveshare ESP32-C6-LCD-1.3 and ESP32-S3-LCD-1.3 media viewer.

The ESP-IDF firmware source for the C6 and S3 boards lives in a separate private repository. This public repo intentionally keeps only:

- the native Windows companion app source
- shared public media/upload protocol headers used by the companion app
- compiled C6 and S3 firmware bundles
- packaging scripts for full/offline and web/downloader companion app releases

## Firmware Bundles

Compiled firmware files are checked in under:

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

The companion app uses each `firmware_package.json` to select the chip, flash settings, firmware file names, and media partition location.

## Companion App

The companion app is a native C++20 Win32 application in `companion_app/`. It does not require Python, .NET, Git, ESP-IDF, or user-installed FFmpeg on an end user's machine.

It can:

- import images, GIFs, and video files
- convert media to device-playable JPEG/MJPEG/RGB565 packages
- flash C6 or S3 firmware
- flash `media.bin` into the firmware media partition
- upload converted media to an S3 SD card over the USB serial COM port
- format the S3 SD card through the running firmware

Runtime tools are provided by either the full/offline app payload or by the web/downloader app:

```text
tools/
  esptool.exe
  ffmpeg.exe
firmware/
  esp32c6/
  esp32s3/
```

## Release Flavors

Full/offline app:

- ship `ESP32MediaViewerCompanion-Full.exe`
- contains a compressed runtime payload
- extracts `tools\` and `firmware\` on first launch
- works without internet

Web/downloader app:

- ship `ESP32MediaViewerCompanion-Web.exe`
- opens a Downloads window
- lets users choose All or Custom downloads
- installs tools and firmware to AppData, the app folder, or a custom folder

## Build Companion App

Example with MSVC and Ninja:

```powershell
cmake -S companion_app -B build\companion-msvc -G Ninja -DCMAKE_BUILD_TYPE=Release
cmake --build build\companion-msvc --config Release
```

Stage the portable folder:

```powershell
powershell -ExecutionPolicy Bypass -File companion_app\package_windows.ps1 -OutDir ..\dist\ESP32MediaViewerCompanion
```

Create the embedded full/offline EXE:

```powershell
powershell -ExecutionPolicy Bypass -File companion_app\make_embedded_payload_zip.ps1 -PackageDir ..\dist\ESP32MediaViewerCompanion -OutZip ..\dist\embedded-runtime-payload.zip
cmake -S companion_app -B build\companion-msvc-full -G Ninja -DCMAKE_BUILD_TYPE=Release -DMEDIA_VIEWER_EMBEDDED_PAYLOAD=ON -DMEDIA_VIEWER_PAYLOAD_ZIP=%CD%\dist\embedded-runtime-payload.zip
cmake --build build\companion-msvc-full --config Release
powershell -ExecutionPolicy Bypass -File companion_app\package_embedded_windows.ps1 -BuildDir ..\build\companion-msvc-full -OutDir ..\dist\ESP32MediaViewerCompanion-Full
```

Create the small web/downloader package:

```powershell
powershell -ExecutionPolicy Bypass -File companion_app\package_web_windows.ps1 -OutDir ..\dist\ESP32MediaViewerCompanion-Web
```

Stage web-downloadable release assets:

```powershell
powershell -ExecutionPolicy Bypass -File tools\make_web_release_assets.ps1 -PackageDir dist\ESP32MediaViewerCompanion -OutDir dist\web-release-assets
```

Publish the staged files to the release download host expected by the app:

```text
https://github.com/d0773d/esp32-media-viewer/releases/latest/download/
```

## Project Layout

```text
companion_app/
  include/                   Shared public package/upload protocol headers.
  src/main.cpp               Native portable Windows companion app.
  package_windows.ps1        Stages the portable app folder.
  make_embedded_payload_zip.ps1
  package_embedded_windows.ps1
  package_web_windows.ps1
  make_release_zip.ps1
firmware_release/
  esp32c6/                   Compiled C6 firmware bundle.
  esp32s3/                   Compiled S3 firmware bundle.
tools/
  make_web_release_assets.ps1
```
