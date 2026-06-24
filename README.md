# ESP32-C6 LCD Media Viewer

ESP-IDF v6 project for the Waveshare ESP32-C6-LCD-1.3. It compiles media into the firmware image, displays JPEG images on the 240x240 ST7789V2 LCD, plays simple RGB565 video assets, and uses the BOOT button as "next media" after the app has booted.

## Hardware Notes

- LCD: ST7789V2, 240x240, SPI.
- LCD pins from Waveshare demo/schematic: SCLK GPIO7, MOSI GPIO6, CS GPIO14, DC GPIO15, RST GPIO21, backlight GPIO22.
- BOOT button: GPIO9, active low. Holding it during reset still enters download mode; after firmware starts it advances media.
- ESP32-C6 USB is fixed USB Serial/JTAG, so this board cannot expose a custom USB Mass Storage drive. The uploader app uses the USB port for flashing the rebuilt firmware image.

## Media

Raw user media goes in `assets/source/`. The uploader app converts it into generated firmware assets under `assets/firmware/`.

Firmware-supported files:

- `.jpg` / `.jpeg`: baseline JPEG. Keep images at or below 240x240 for best results; larger images are decoded at 1/2, 1/4, or 1/8 scale.
- `.rgbv`: simple raw RGB565 video produced by `tools/convert_video_rgbv.py`.

The files in `assets/firmware/` are linked into the app image, so they count against the board's 4 MB flash. The project uses a custom single-app partition table to leave most of flash available for embedded media.

## LCD Asset Pipeline

The LCD path is intentionally big-endian RGB565 end to end:

- JPEG assets are decoded by the firmware as `JPEG_IMAGE_FORMAT_RGB565` with `.swap_color_bytes = 1`.
- Video assets are converted to `.rgbv` with FFmpeg `format=rgb565be`.
- The ST7789V2 panel init sends `RAMCTRL` (`0xB0`) as `0x00, 0xF0` in `components/media_viewer_display/src/display.c`. ESP-IDF v6's ST7789 driver uses `0xF0` as the big-endian default; setting the little-endian bit caused wrong colors on this board.
- The panel is configured as `LCD_RGB_ELEMENT_ORDER_BGR`. Keep that unless red and blue are swapped.
- `INVON` (`0x21`) is intentional and matches the Waveshare-style ST7789V2 init sequence.

The firmware draws RGB565 data in 20-line chunks and sends a blocking NOP command before decode or video buffers are freed or reused. If an image has the right shape but looks smeared, noisy, or partially corrupt, check this transfer chunking and final wait first. If the geometry is right but colors are psychedelic, check `RAMCTRL`: `0x00, 0xE8` is wrong for the current asset pipeline, while `0x00, 0xF0` is correct.

## Convert And Upload

Drop media into `assets/source/`, plug in the board over USB, then run:

```powershell
python tools\media_uploader.py --port COM7
```

If you omit `--port`, `idf.py` will try to auto-detect the board.

The uploader detects common image/video inputs, converts images to 240x240 JPEG, converts video/GIF files to 240x240 RGB565 `.rgbv`, runs `idf.py reconfigure`, builds, and flashes the firmware over USB. By default, videos are trimmed to 3 seconds at 8 FPS so they fit in flash; use `--max-video-seconds` and `--fps` to change that.

Media fit defaults to `contain`, which preserves the whole image and adds black bars when the source is not square. Use `--fit cover` to fill the LCD by center-cropping, or `--fit stretch` to fill the LCD by distorting the source into a square.

Prepare assets without building or flashing:

```powershell
python tools\media_uploader.py --convert-only
```

Prepare full-screen cropped assets:

```powershell
python tools\media_uploader.py --convert-only --fit cover
```

Convert a desktop video:

```powershell
python tools\convert_video_rgbv.py input.mp4 assets\firmware\output.rgbv --fps 10 --fit cover
```

The converter requires `ffmpeg` in `PATH`. It fits video to 240x240 with the selected `--fit` mode and writes big-endian RGB565 frames.

## Project Layout

```text
main/
  app_main.c                 App loop and high-level state flow.
assets/
  source/                    Raw user media drop folder.
  firmware/                  Generated media compiled into firmware.
components/
  media_viewer_assets/       Build-time embedded media index.
  media_viewer_board/        Waveshare pin map and shared SPI bus setup.
  media_viewer_display/      ST7789V2 LCD and backlight driver.
  media_viewer_input/        BOOT button debounce and press handling.
  media_viewer_media/        JPEG decode and RGB565 video playback.
tools/
  media_uploader.py          Convert, build, and flash workflow.
  convert_video_rgbv.py      Desktop video-to-RGB565 converter.
```

## Build

```powershell
idf.py set-target esp32c6
idf.py build
idf.py flash monitor
```

The project was created for ESP-IDF v6.x and verified with ESP-IDF v6.0.1.
