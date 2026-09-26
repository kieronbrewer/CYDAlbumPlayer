# CYD Album Player (CYD2USB & PlatformIO Edition)

> **Lineage & Acknowledgements:**
> This repository is a fork of **[malaq88/CYDAlbumPlayer](https://github.com/malaq88/CYDAlbumPlayer)**, which was inspired by and built upon **[Sparkadium/Cheap-Yellow-MP3-Player](https://github.com/Sparkadium/Cheap-Yellow-MP3-Player)**.
> All credit for the DAP-style user interface, spectrum analyzer, and audio pipeline belongs to **malaq88** and **Sparkadium**.
> Huge thanks also to the library authors:
> - **Phil Schatzmann** ([ESP32-A2DP](https://github.com/pschatzmann/ESP32-A2DP))
> - **Earle Philhower** ([ESP8266Audio](https://github.com/earlephilhower/ESP8266Audio))
> - **Bodmer** ([TFT_eSPI](https://github.com/Bodmer/TFT_eSPI))
> - **Paul Stoffregen** ([XPT2046_Touchscreen](https://github.com/PaulStoffregen/XPT2046_Touchscreen))

---

## What’s New in This Fork

This fork was created by **[Kieron Brewer](https://github.com/kieronbrewer) although mostly completed by prompting Google's Antigravity tool which wrote the code. ** The work was done to add native support for the newer **Dual-USB "CYD2USB" (ESP32-2432S028R V3)** board variant and enable seamless 1-command builds via PlatformIO:

1. **Dual-USB (CYD2USB / ST7789) Display Support**:
   - Configured out of the box for the **ST7789** display controller found on dual-port boards (USB-C + Micro-USB).
   - Fixed the horizontal mirror issue (`TFT_MAD_MX` added to memory access control) so text and buttons render naturally from left to right.
   - Resolved washed-out/inverted colors (`tft.invertDisplay(false)`) so the DAP dark theme renders with deep blacks and crisp white/accent fonts.
2. **Dual-Pin Backlight Driving (GPIO 21 & GPIO 27)**:
   - Early CYD boards wire backlight to GPIO 21; dual-USB revisions often wire it to GPIO 27. The firmware actively drives both pins HIGH, ensuring the screen illuminates regardless of hardware batch.
3. **Tap-to-Wake & Extended 60s Timeout**:
   - The screen can now be awakened with a **single tap on the touchscreen** (in addition to pressing the physical `BOOT` button on the back).
   - Idle screen timeout extended from 30 seconds to **60 seconds** to allow ample time for Bluetooth pairing.
4. **Native PlatformIO Build (`platformio.ini`)**:
   - No need to manually copy `User_Setup.h` files into Arduino IDE library folders.
   - Pre-configured `huge_app.csv` (3MB partition table) and locked library versions for a 1-click build:
     ```bash
     pio run -t upload
     ```
5. **Fast Track Switching & Near-Gapless Playback (`FastAudioSourceSD`)**:
   - **Eliminated the ~10-second transition delay**: Modern MP3 files contain ID3v2 tags with embedded cover art (often 100 KB to 500 KB+). Previously, the decoder would encounter this non-audio data, lose synchronization, and fall back to single-byte SD card reads over SPI—executing up to 300,000 SPI transactions and moving ~460 MB in RAM byte-by-byte before finding the first audio frame.
   - **Instant ID3v2 Skipping**: Our custom `FastAudioSourceSD` reads the 10-byte ID3 header, computes the 28-bit synchsafe tag length, and seeks directly past the entire tag and cover art in `< 1 ms`.
   - **Clean EOF Truncation**: Detects ID3v1 (`TAG`) and APE tags at the end of the file to stop decoding immediately when audio ends, preventing trailing 1-byte search loops.
   - **4KB RAM Block Buffering**: Replaces single-byte SPI reads with 4096-byte multi-sector chunked reads served from RAM.
   - **Result**: Song transitions and manual track skips happen in **< 0.1s (near-gapless)** instead of pausing in silence for 10 seconds.

---

## Overview

An ESP32 "Cheap Yellow Display" (CYD) music player that:
- Scans your microSD card by albums (folders) and plays `.mp3` / `.wav` tracks.
- Features a **DAP-style player screen** (dark theme, status bar, **16-bar real-time spectrum visualizer**, progress bar, timestamps).
- Acts as a **Bluetooth A2DP Source** to stream audio directly to Bluetooth headphones or speakers.
- **Interactive Bluetooth Picker**: On boot, scans nearby Bluetooth audio sinks and lets you select your speaker/headset directly from the touchscreen.
- Uses the **rear RGB LED** as a Bluetooth connection and playback status indicator.
- **Power saving**: Backlight automatically sleeps after 60 seconds of inactivity; tap the screen or press the `BOOT` button (GPIO 0) to wake it up.

---

## Screenshots

| Album Browser | Playback Screen |
|:---:|:---:|
| ![Album browser](AlbumPlaylist.jpeg) | ![Playback screen](Execution_screen.jpeg) |

---

## Hardware Pinout (ESP32-2432S028R)

| Peripheral | Pins |
| :--- | :--- |
| **Display (ST7789 / HSPI)** | MOSI: `13`, MISO: `12`, SCLK: `14`, CS: `15`, DC: `2`, RST: `-1` |
| **Backlight** | `GPIO 21` and `GPIO 27` (Active HIGH) |
| **Touchscreen (XPT2046)** | MOSI: `32`, MISO: `39`, CLK: `25`, CS: `33`, IRQ: `36` |
| **MicroSD (VSPI)** | CS: `5`, MOSI: `23`, MISO: `19`, SCK: `18` |
| **Rear RGB LED** | Red: `4`, Green: `16`, Blue: `17` (Active LOW) |
| **BOOT Button** | `GPIO 0` (Active LOW) |

---

## Quick Start (PlatformIO)

1. Clone this repository:
   ```bash
   git clone https://github.com/kieronbrewer/CYDAlbumPlayer.git
   cd CYDAlbumPlayer
   ```
2. Connect your CYD board via USB.
3. Build and upload:
   ```bash
   pio run -t upload
   ```
4. (Optional) Open the serial monitor:
   ```bash
   pio device monitor -b 115200
   ```

---

## SD Card Organization

1. Format your microSD card as **FAT32** (cards 64GB+ may require a formatting tool like GUIFormat).
2. Organize your music into folders on the root directory. Each folder represents an **Album**:

```text
SD Card Root (e.g. E:\)
├── Pink Floyd - The Wall\
│   ├── 01 - In the Flesh.mp3
│   ├── 02 - The Thin Ice.mp3
│   └── ...
├── Daft Punk - Discovery\
│   ├── 01 - One More Time.mp3
│   └── ...
└── guara565.raw  (Optional: 200x218 boot splash image)
```

* **Supported Formats**: `.mp3` and `.wav` (up to 32 albums and 300 tracks).
* Tracks inside an album play in alphabetical order by filename. Prefixing track numbers (`01 - `, `02 - `) preserves album track sequence.

---

## RGB Status LED Behavior (Rear of CYD)

| State | LED Pattern |
| :--- | :--- |
| **Bluetooth Scanning / Pairing** | Alternating **Red** and **Blue** blink |
| **Connected & Playing** | Alternating **Green** and **Blue** (~450 ms) |
| **Connected & Paused / Stopped** | LED Off |
