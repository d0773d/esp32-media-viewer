# ESP32 LCD Media Viewer

ESP-IDF v6 project for the Waveshare ESP32-C6-LCD-1.3 and ESP32-S3-LCD-1.3. The firmware displays JPEG images, MJPEG video packages, and legacy RGB565 video clips on the 240x240 ST7789 LCD. The C6 build uses the BOOT button as "next media" after the app has booted; the S3 build auto-rotates through the media playlist.

The current design keeps user media out of the application binary. The board runs a prebuilt firmware image, and the companion app creates a separate `media.bin` package that is flashed into a dedicated data partition.

## Hardware Notes

- Waveshare ESP32-C6-LCD-1.3: ST7789V2 240x240 SPI LCD, SCLK GPIO7, MOSI GPIO6, CS GPIO14, DC GPIO15, RST GPIO21, backlight GPIO22, BOOT GPIO9.
- Waveshare ESP32-S3-LCD-1.3: ST7789VW 240x240 SPI LCD, SCLK GPIO40, MOSI GPIO41, CS GPIO39, DC GPIO38, RST GPIO42, backlight GPIO4. The S3 build uses the vendor demo's RGB order and 80-pixel Y gap.
- The S3 board has onboard 16MB flash and 8MB PSRAM. Its firmware uses a much larger media partition than the C6 build.
- The S3 board does not expose a usable next-media button in this project, so its firmware auto-advances instead of waiting for button input.
- USB is used for serial flashing. The companion app flashes firmware and `media.bin`; it does not require the board to expose USB Mass Storage.

## Media Workflow

End-user workflow:

1. Run the portable Windows companion app.
2. Add image, GIF, or video files.
3. Pick the fit mode: `contain`, `cover`, or `stretch`.
4. Select the board type and COM port.
5. Press Flash.

The app converts media to device-ready files, builds a `media.bin` package, downloads or uses bundled prebuilt firmware files, and flashes:

```text
0x0000   bootloader.bin
0x8000   partition-table.bin
0x10000  esp32_media_viewer.bin
0x170000 media.bin on C6
0x210000 media.bin on S3
```

The device firmware reads the `media` partition at runtime. On C6, press BOOT to advance to the next item. On S3, still images remain on screen for 5 seconds and animated/video items advance after their last frame.

Animated GIFs and videos are converted to MJPEG packages and auto-fit to the available flash budget. The app starts from the selected FPS and video-seconds values, then lowers per-file frame count when needed so the final `media.bin` fits the `media` partition without slowing the selected playback FPS.

The output selector near the Convert/Flash buttons controls where `media.bin` is saved: the default app data folder, the same folder as the EXE, or a custom folder. The app first looks for `settings.ini` next to the EXE for portable mode; if it is not present, it falls back to `%LOCALAPPDATA%\ESP32MediaViewerCompanion\settings.ini`. The Settings selector can switch between those two storage locations.

Optimized GIF frames are composited before conversion, and firmware video playback schedules frame timing from the start of each frame so LCD transfer time does not get added to the requested delay.

The log area includes Clear Log, Save Log, and an Auto clear on run option. The auto-clear option is stored in the app settings file.

The Flash workflow includes an Erase media first option. When enabled, the app runs `esptool erase-region` for the configured media partition before writing the new `media.bin`, which removes any leftover bytes from older, larger media packages.

## Partition Layouts

ESP32-C6, 4MB flash:

```text
# Name,   Type, SubType, Offset,  Size
nvs,      data, nvs,     0x9000,  0x6000
phy_init, data, phy,     0xf000,  0x1000
factory,  app,  factory, 0x10000, 0x160000
media,    data, 0x40,    0x170000,0x290000
```

ESP32-S3, 16MB flash:

```text
# Name,   Type, SubType, Offset,  Size
nvs,      data, nvs,     0x9000,  0x6000
phy_init, data, phy,     0xf000,  0x1000
factory,  app,  factory, 0x10000, 0x200000
media,    data, 0x40,    0x210000,0xdf0000
```

The `media` partition is a custom data partition. Its package format is defined in `components/media_viewer_assets/include/media_package_format.h` and is shared by the firmware and companion app.

## LCD Asset Pipeline

The LCD path is intentionally big-endian RGB565 at the panel boundary:

- JPEG assets are decoded by the firmware as `JPEG_IMAGE_FORMAT_RGB565` with `.swap_color_bytes = 1`.
- MJPEG video frames are stored as per-frame JPEG data, decoded to RGB565 on the device, and then sent to the LCD.
- Legacy RGB565 video frames are stored big-endian.
- The C6 ST7789V2 init sends `RAMCTRL` (`0xB0`) as `0x00, 0xF0` in `components/media_viewer_display/src/display.c`. ESP-IDF v6's ST7789 driver uses `0xF0` as the big-endian default; setting the little-endian bit caused wrong colors on this board.
- The C6 panel is configured as BGR. The S3 panel is configured as RGB with the Waveshare demo's display offset.
- `INVON` (`0x21`) is intentional and matches the Waveshare-style ST7789 init sequences.

If an image has the right shape but looks smeared, noisy, or partially corrupt, check transfer chunking and the final display wait first. If the geometry is right but colors are psychedelic, check `RAMCTRL`: `0x00, 0xE8` is wrong for the current asset pipeline, while `0x00, 0xF0` is correct.

## Native Companion App

The companion app is a native C++20 Win32 application in `companion_app/`. It does not require Python, .NET, Git, ESP-IDF, or user-installed FFmpeg on the end user's machine.

Current runtime dependencies:

- Windows Imaging Component for image and GIF decoding.
- A bundled `tools/ffmpeg.exe` for common video conversion.
- Windows Media Foundation as a backup video decoder if FFmpeg is not bundled or cannot decode a file.
- A bundled `tools/esptool.exe` or `esptool.exe` next to the app for flashing.
- Either bundled firmware files under `firmware/` next to the app, or downloadable GitHub release assets.

Expected portable folder layout:

```text
ESP32MediaViewerCompanion.exe
tools/
  esptool.exe
  ffmpeg.exe
firmware/
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

The Board dropdown selects which firmware subfolder to use. If the matching local firmware folder is absent, the app tries to download assets from the latest GitHub release:

```text
https://github.com/d0773d/esp32-media-viewer/releases/latest/download/
```

## Build Firmware

```powershell
idf.py -B build\esp32c6 -D SDKCONFIG=sdkconfig.esp32c6 -D IDF_TARGET=esp32c6 build
idf.py -B build\esp32s3 -D SDKCONFIG=sdkconfig.esp32s3 -D IDF_TARGET=esp32s3 build
```

Stage the prebuilt firmware release payload:

```powershell
powershell -ExecutionPolicy Bypass -File tools\stage_firmware_release.ps1 -BuildDir build\esp32c6 -OutDir dist\firmware-release-esp32c6 -Target esp32c6
powershell -ExecutionPolicy Bypass -File tools\stage_firmware_release.ps1 -BuildDir build\esp32s3 -OutDir dist\firmware-release-esp32s3 -Target esp32s3
```

Package the companion app with the firmware bundle for the board you want to ship:

```powershell
powershell -ExecutionPolicy Bypass -File companion_app\package_windows.ps1 -FirmwareReleaseDir ..\dist\firmware-release-esp32c6 -OutDir ..\dist\ESP32MediaViewerCompanion-C6
powershell -ExecutionPolicy Bypass -File companion_app\package_windows.ps1 -FirmwareReleaseDir ..\dist\firmware-release-esp32s3 -OutDir ..\dist\ESP32MediaViewerCompanion-S3
```

## Build Companion App

Example with MSVC and Ninja:

```powershell
cmake -S companion_app -B build\companion-msvc -G Ninja -DCMAKE_BUILD_TYPE=Release
cmake --build build\companion-msvc --config Release
```

On this development machine, the Espressif-installed CMake/Ninja and Visual Studio Build Tools can also be used explicitly.

## Developer Utilities

The old Python scripts under `tools/` are developer helpers and experiments. The shipping companion app path is native C++ and does not use them for the end-user workflow.

## Project Layout

```text
main/
  app_main.c                 App loop and high-level state flow.
assets/
  source/                    Developer media scratch folder.
  firmware/                  Legacy generated media scratch folder.
components/
  media_viewer_assets/       Runtime media partition reader and shared package format.
  media_viewer_board/        Waveshare C6/S3 pin maps and shared SPI bus setup.
  media_viewer_display/      ST7789 LCD and backlight driver.
  media_viewer_input/        BOOT button debounce and press handling.
  media_viewer_media/        JPEG, MJPEG, and RGB565 video playback.
firmware_release/
  firmware_package.json      C6 release manifest consumed by the companion app.
  firmware_package.esp32s3.json
tools/
  stage_firmware_release.ps1 Copies build outputs into a firmware release folder.
  media_uploader.py          Legacy Python convert/build/flash utility.
  convert_video_rgbv.py      Legacy video-to-RGB565 converter.
companion_app/
  CMakeLists.txt
  src/main.cpp               Native portable Windows companion app.
  package_windows.ps1        Stages the portable Windows app folder.
```

The project was created for ESP-IDF v6.x and verified with ESP-IDF v6.0.1.
