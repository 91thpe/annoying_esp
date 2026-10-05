# Implementation plan

Short plan for steps 2–4. The BLE contract is in [PROTOCOL.md](PROTOCOL.md).

## Hardware findings

What I confirmed, and from where (docs.m5stack.com, espressif.com and the
PlatformIO registry are blocked from my build sandbox, so I used mirrors on
GitHub and search results):

| Item | Status | Source |
|------|--------|--------|
| Board `m5stack-stamps3` exists | Confirmed: ESP32-S3, 8 MB flash, QIO 80 MHz, `default_8MB.csv`, sets `ARDUINO_USB_MODE=1` | `boards/m5stack-stamps3.json` in platformio/platform-espressif32 and pioarduino; built a test sketch against it |
| G3 broken out | Confirmed by two independent sources | arduino-esp32 variant `m5stack_stamp_s3/pins_arduino.h` (G0–G15, G39–G44, G46); Zephyr `m5stack_stamps3_connectors.dtsi` (header pin 3 → GPIO3). |
| Onboard button | GPIO0, active low | M5Unified (`!gpio_in(GPIO_NUM_0)` for StampS3), Zephyr DTS (`gpio0 0 GPIO_ACTIVE_LOW`) |
| Onboard RGB LED | WS2812 on GPIO21 | M5Unified pin table. Used only for optional status blinks |
| GPIO3 strapping | Harmless with factory eFuses. GPIO3 selects the JTAG source **only if** eFuse `STRAP_JTAG_SEL` is burned; default is 0, so GPIO3 is ignored at boot | ESP32-S3 datasheet (via search result and esp32.com forum thread); ESP-IDF `esp_efuse_table.csv` |
| ESP32-S3FN8, no PSRAM | Consistent with board JSON (8 MB flash, no PSRAM flags) | Board JSON, M5Stack shop listing via search |

What I could **not** confirm:

1. **The eFuse state of your specific chip.** Factory default makes GPIO3 a
   don't-care, but I can't see your board. Check once with
   `espefuse.py --port <port> summary | grep -i jtag` (`STRAP_JTAG_SEL` should
   be `False`).
2. **GPIO current source capability.** From memory, the ESP32-S3 datasheet
   gives about 40 mA source current at the strongest drive setting, with 20 mA
   as the default drive. I could not open the datasheet to verify the exact
   figures.
3. **Your buzzer's trigger threshold and current.** That decides whether
   3.3 V direct drive is OK.
4. **GPIO3 state between reset and `setup()`.** GPIO3 is floating by default
   at reset (per the datasheet strapping table). Firmware drives it LOW as
   early as possible, but for the few hundred ms of ROM boot and bootloader
   it floats. A 10 kΩ pull-down from G3 to GND guarantees silence during boot.
   I'll put this in the README.

## Toolchain decision

- **Platform: pioarduino `platform-espressif32` 55.03.31** (Arduino core 3.3.x
  on ESP-IDF 5.5) instead of the official `platformio/espressif32`, which is
  frozen on Arduino core 2.0.17. Both define `m5stack-stamps3`.
- **NimBLE-Arduino 2.5.1** pinned via its GitHub tag.
- I verified a minimal NimBLE sketch builds for `m5stack-stamps3` with this
  combination (RAM 9.5%, flash 17.5% of the app partition).
- Sandbox caveat: the PlatformIO registry is blocked here, so I hand-installed
  SCons (and will do the same for Unity) to run builds. Your machine will pull
  them normally; nothing in the repo depends on my workaround.

## Firmware layout (`firmware/`)

```
platformio.ini          env:stamps3 (device), env:native (unit tests)
include/
  app_config.h          pin, limits, device name, firmware version
  secrets.example.h     template; copy to secrets.h (gitignored)
lib/core/               pure C++, no Arduino/ESP headers, built in both envs
  config.{h,cpp}        Config struct, defaults, validate/clamp, encode/decode
  pattern.{h,cpp}       activation step generator + total duration
  scheduler.{h,cpp}     armed/waiting logic, interval draw (injected RNG)
  status.{h,cpp}        Status struct + encode/decode
  controller.{h,cpp}    command layer: Command → state changes; owns pattern
                        + scheduler; time and RNG injected; emits outputs
                        through an abstract OutputSink
src/
  main.cpp              wiring only
  engine_task.cpp       FreeRTOS task: queue of commands, waits with timeout
                        = time to next event (no polling, no delay())
  output_hw.cpp         GPIO/LEDC driver + esp_timer max-ON watchdog
  storage_nvs.cpp       Preferences/NVS load/save of Config bytes + armed
  ble_transport.cpp     NimBLE server; translates GATT ↔ Command/Status
  button.cpp            GPIO0 debounce; 2 s = disarm+stop, 10 s = clear bonds
test/test_core/         Unity tests for config, pattern, scheduler, status,
                        controller (fake clock + fake RNG)
```

Key design points:

- **Single owner of state.** Only the engine task touches the controller.
  BLE callbacks and the button post `Command`s to a FreeRTOS queue and return
  immediately. STOP wakes the task at once because it unblocks the queue
  wait.
- **Transport-agnostic.** `ble_transport` depends on the controller's public
  `Command` / `StatusSnapshot` API and the protocol codecs only. A future MQTT
  transport is another adapter beside it.
- **Two-layer safety.** The pattern engine never schedules ON longer than
  `pulse_ms` (≤ 2000). Independently, `output_hw` arms a one-shot `esp_timer`
  on every ON edge that forces LOW at 2050 ms.
- **Random draw** uses `esp_random()` behind an injected function, with
  rejection sampling for an unbiased uniform integer.
- **NVS** stores the Config blob in its wire format (version byte included),
  so firmware upgrades can migrate or fall back to defaults.

## App layout (`app/`)

- Flutter, Android only (`flutter create --platforms=android`).
  `minSdk 23` (Android 6.0): low enough for any phone you'd sideload to, and
  above the floor `flutter_blue_plus` needs.
- Dependencies: `flutter_blue_plus`, `provider`, `shared_preferences`
  (last device, presets), `permission_handler` only if `flutter_blue_plus`'s
  built-in permission flow isn't enough.
- `lib/protocol/`: pure Dart codecs mirroring `PROTOCOL.md`, unit tested with
  the **same byte vectors** as the firmware tests (shared JSON fixture in
  `docs/test_vectors.json`, read by both test suites).
- `lib/ble/`: connection manager (scan by service UUID, MTU request, bonding,
  auto-reconnect with backoff, error mapping to user messages).
- `lib/ui/`: Scan, Control, Settings, Pattern preview (CustomPainter
  timeline driven by the same pattern math as firmware, ported to Dart and
  tested against the same vectors).
- Manifest: `BLUETOOTH_SCAN` with `neverForLocation`, `BLUETOOTH_CONNECT`,
  and `ACCESS_FINE_LOCATION` with `maxSdkVersion="30"`.

## Verification I can and cannot run here

| Command | Expected here |
|---------|---------------|
| `pio run -e stamps3` | Can run (verified with a probe sketch) |
| `pio test -e native` | Can run, using host gcc 13 |
| `flutter analyze`, `flutter test` | Should be able to run: the Flutter SDK and pub.dev are reachable |
| `flutter build apk --debug` | **Likely blocked**: dl.google.com (Android SDK platforms and build-tools) is denied by the sandbox network policy. I'll try; if it fails I'll say so and you build the APK locally (one command, documented in the README) |
