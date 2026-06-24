# ESP32-C6 LCD Media Viewer

ESP-IDF v6 project for the Waveshare ESP32-C6-LCD-1.3. The firmware displays JPEG images and simple RGB565 video clips on the 240x240 ST7789V2 LCD, and uses the BOOT button as "next media" after the app has booted.

The current design keeps user media out of the application binary. The board runs a prebuilt firmware image, and the companion app creates a separate `media.bin` package that is flashed into a dedicated data partition.

## Hardware Notes

- LCD: ST7789V2, 240x240, SPI.
- LCD pins from Waveshare demo/schematic: SCLK GPIO7, MOSI GPIO6, CS GPIO14, DC GPIO15, RST GPIO21, backlight GPIO22.
- BOOT button: GPIO9, active low. Holding it during reset still enters download mode; after firmware starts it advances media.
- ESP32-C6 USB is fixed USB Serial/JTAG, so this board cannot expose a custom USB Mass Storage drive. The companion app uses the USB port for serial flashing.

## Media Workflow

End-user workflow:

1. Run the portable Windows companion app.
2. Add image, GIF, or video files.
3. Pick the fit mode: `contain`, `cover`, or `stretch`.
4. Select the ESP32-C6 COM port.
5. Press Flash.

The app converts media to device-ready files, builds a `media.bin` package, downloads or uses bundled prebuilt firmware files, and flashes:

```text
0x0000   bootloader.bin
0x8000   partition-table.bin
0x10000  esp32_c6_lcd_media_viewer.bin
0x170000 media.bin
```

The device firmware reads the `media` partition at runtime. Press BOOT to advance to the next item.

Animated GIFs and videos are auto-fit to the available flash budget. The app starts from the selected FPS and video-seconds values, then lowers per-file frame count when needed so the final `media.bin` fits the `media` partition without slowing the selected playback FPS.

The output selector near the Convert/Flash buttons controls where `media.bin` is saved: the default app data folder, the same folder as the EXE, or a custom folder. The app first looks for `settings.ini` next to the EXE for portable mode; if it is not present, it falls back to `%LOCALAPPDATA%\ESP32MediaViewerCompanion\settings.ini`. The Settings selector can switch between those two storage locations.

Optimized GIF frames are composited before conversion, and firmware video playback schedules frame timing from the start of each frame so LCD transfer time does not get added to the requested delay.

## Partition Layout

```text
# Name,   Type, SubType, Offset,  Size
nvs,      data, nvs,     0x9000,  0x6000
phy_init, data, phy,     0xf000,  0x1000
factory,  app,  factory, 0x10000, 0x160000
media,    data, 0x40,    0x170000,0x290000
```

The `media` partition is a custom data partition. Its package format is defined in `components/media_viewer_assets/include/media_package_format.h` and is shared by the firmware and companion app.

## LCD Asset Pipeline

The LCD path is intentionally big-endian RGB565 end to end:

- JPEG assets are decoded by the firmware as `JPEG_IMAGE_FORMAT_RGB565` with `.swap_color_bytes = 1`.
- RGB565 video frames are stored big-endian.
- The ST7789V2 panel init sends `RAMCTRL` (`0xB0`) as `0x00, 0xF0` in `components/media_viewer_display/src/display.c`. ESP-IDF v6's ST7789 driver uses `0xF0` as the big-endian default; setting the little-endian bit caused wrong colors on this board.
- The panel is configured as `LCD_RGB_ELEMENT_ORDER_BGR`. Keep that unless red and blue are swapped.
- `INVON` (`0x21`) is intentional and matches the Waveshare-style ST7789V2 init sequence.

If an image has the right shape but looks smeared, noisy, or partially corrupt, check transfer chunking and the final display wait first. If the geometry is right but colors are psychedelic, check `RAMCTRL`: `0x00, 0xE8` is wrong for the current asset pipeline, while `0x00, 0xF0` is correct.

## Native Companion App

The companion app is a native C++20 Win32 application in `companion_app/`. It does not require Python, .NET, Git, ESP-IDF, or user-installed FFmpeg on the end user's machine.

Current runtime dependencies:

- Windows Imaging Component for image and GIF decoding.
- Windows Media Foundation for common video decoding.
- A bundled `tools/esptool.exe` or `esptool.exe` next to the app for flashing.
- Either bundled firmware files under `firmware/` next to the app, or downloadable GitHub release assets.

Expected portable folder layout:

```text
ESP32MediaViewerCompanion.exe
tools/
  esptool.exe
firmware/
  firmware_package.json
  bootloader.bin
  partition-table.bin
  esp32_c6_lcd_media_viewer.bin
```

If the `firmware/` folder is absent, the app tries to download assets from the latest GitHub release:

```text
https://github.com/d0773d/esp32-media-viewer/releases/latest/download/
```

## Build Firmware

```powershell
idf.py set-target esp32c6
idf.py build
```

Stage the prebuilt firmware release payload:

```powershell
powershell -ExecutionPolicy Bypass -File tools\stage_firmware_release.ps1
```

The staged release files are written to:

```text
dist/firmware-release/
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
  media_viewer_board/        Waveshare pin map and shared SPI bus setup.
  media_viewer_display/      ST7789V2 LCD and backlight driver.
  media_viewer_input/        BOOT button debounce and press handling.
  media_viewer_media/        JPEG decode and RGB565 video playback.
firmware_release/
  firmware_package.json      Release manifest consumed by the companion app.
tools/
  stage_firmware_release.ps1 Copies build outputs into dist/firmware-release.
  media_uploader.py          Legacy Python convert/build/flash utility.
  convert_video_rgbv.py      Legacy video-to-RGB565 converter.
companion_app/
  CMakeLists.txt
  src/main.cpp               Native portable Windows companion app.
  package_windows.ps1        Stages the portable Windows app folder.
```

The project was created for ESP-IDF v6.x and verified with ESP-IDF v6.0.1.
