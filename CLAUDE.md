# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

PlatformIO/Arduino project template for ESP32 family (ESP32, S2, S3, C6, P4) with GitHub Actions CI/CD. Uses [pioarduino](https://github.com/pioarduino/platform-espressif32) — a community fork of platformio's espressif32 platform that supports newer chips.

## Build commands

```sh
# Build all board variants
pio run

# Build single env (e.g. esp32_c6_n4)
pio run -e esp32_c6_n4

# Upload to connected board
pio run -e esp32_c6_n4 -t upload

# Monitor serial output
pio device monitor -b 115200

# Upload + monitor
pio run -e esp32_c6_n4 -t upload && pio device monitor
```

## Version/release workflow

Releases are triggered by git tags (`v1.0.0` format). CI builds all envs and attaches firmware `.bin` files to the GitHub release.

```sh
# Dry-run: print what tag would be created
python bump_version.py [major|minor|patch]

# Actually tag and push (triggers CI release build)
python bump_version.py --execute [major|minor|patch]
```

Also exposed as PlatformIO custom targets: `bump_version` (dry-run) and `bump_version_execute`.

## Architecture

### Config split

- [platformio.ini](platformio.ini) — board `[env:*]` definitions, each extending a chip-family base
- [extra_configs/base_envs.ini](extra_configs/base_envs.ini) — chip-family bases (`[esp32]`, `[esp32s3]`, `[esp32c6]`, `[esp32p4]`) that board envs `extends =`
- [extra_configs/extra.ini](extra_configs/extra.ini) — reusable flag snippets (`${extra.build_flags_psram}`, `${extra.build_flags_native_usb}`)
- [extra_configs/lib_deps.ini](extra_configs/lib_deps.ini) — global library deps (RadioLib, OneButton, etc. are commented out; uncomment to enable)

### Scripts

- [extra_script.py](extra_script.py) — pre-build: renames firmware to `firmware_<env>_<version>.bin` using `FIRMWARE_VERSION` env var
- [bump_version.py](bump_version.py) — post-build PlatformIO target + standalone CLI for semver tag bumping

### Board customization

- [boards/](boards/) — custom `.json` board definitions (LilyGo, Waveshare boards not in upstream pio)
- [variants/](variants/) — `pins_arduino.h` for boards needing custom pin mappings (TTGO T-Beam, LoRa32, ESP32-P4 M3)
- [lib/config/](lib/config/) — project-specific config headers; added to include path via `-I lib/config`
- [partitions/](partitions/) — custom partition tables (e.g. 16MB OTA + PHY)

### Feature flags (build_flags per board env)

Boards declare hardware via `-D` flags: `HAS_BUTTON`, `HAS_LED`, `HAS_GPS`, `HAS_DISPLAY_SSD1306`, `HAS_PMU`, `USE_SX1276`, `USE_SX1262`, `USE_SX1280`.

### Firmware output

Built to `.pio/build/<env>/firmware_<env>_<version>.bin`. Files with `.factory.` in name (from pioarduino) include bootloader + partition table and are for initial flashing; others are for OTA.

## CI

- [build.yml](.github/workflows/build.yml) — triggers on PR to main or push (non-tag); builds with `FIRMWARE_VERSION=main`
- [build_release.yml](.github/workflows/build_release.yml) — triggers on any tag push; builds and publishes GitHub release with firmware artifacts + CHANGELOG.md as release notes
