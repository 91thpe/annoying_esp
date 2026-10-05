# Handoff: ATtiny85 office cricket

This is the starting brief for a new repository and a fresh Claude Code
session. Paste it (or commit it as `docs/HANDOFF.md`) at the start of the new
session. It replaces the earlier ESP32-S3/BLE "annoying_esp" project, which
is parked.

## 1. Goal

A tiny, standalone noise maker hidden in an office. It makes a short,
puzzling sound ("what the heck is that?") at random intervals, for example
somewhere between 5 and 15 minutes apart. Each sound is short so nobody can
walk toward it and find the device. Cricket-like is the first target sound.

**Keep it very simple.** No radio, no app, no remote control. All
parameters are compile-time constants at the top of one source file. To
change behavior, edit and reflash.

## 2. Hardware (as stated by the owner)

| Item | Detail |
|------|--------|
| MCU | ATtiny85, bare DIP-8 |
| Carrier/power board | AliExpress "ATtiny85 to micro-USB" board with a DIP socket (Digispark-style). Used for USB 5 V power only |
| Programmer | Arduino Mega running the ArduinoISP sketch (owner built it earlier) |
| Buzzer | 3-wire buzzer module (VCC / GND / signal). **Type not yet confirmed**, see §6 |
| Power | USB 5 V. A simple latching power switch in the 5 V line. Battery is out of scope |
| Spare parts | 2N3906 PNP transistors (useful now that logic is 5 V) |
| Dev environment | Owner's VS Code + PlatformIO on their own PC |

## 3. Behavior requirements

1. **Power on = running.** There is no arm/disarm switch. The power switch is
   the only control.
2. **Never sound at power-on.** After power-up, wait a quiet period first
   (default: 10 min, see open questions), then start the random schedule.
   The output pin must be held in its "silent" state from reset onward
   (pull resistor, see §5).
3. **Random interval** uniform in `[INTERVAL_MIN_S, INTERVAL_MAX_S]`
   (defaults 300 and 900 s). The next interval is counted from the end of
   the previous sound. A new random interval is drawn every time.
4. **Short sounds.** One "activation" should last roughly 0.2–1.5 s in total.
   Hard cap: no single activation longer than 3 s, and no tone held longer
   than 500 ms.
5. **Variation.** Small random variation per activation (number of chirps,
   ±2–4% pitch) so it doesn't sound mechanical.
6. **Not the same sequence after every power cycle.** The ATtiny85 has no
   hardware random number generator, so seed the generator properly (§5.4).
7. **Test build.** A compile-time `TEST_MODE` that plays every sound
   profile once, shortly after boot, with short intervals, for tuning at
   the desk. The normal build must never do this.

## 4. Sound design

### 4.1 Why some sounds are hard to locate

- People locate pure tones worst in roughly the **1–3 kHz** band. Below about
  1.5 kHz the brain uses arrival-time differences between the ears; above it,
  loudness differences. Near the crossover neither cue works well
  ([Sound localization, eScholarship](https://escholarship.org/content/qt01k9w5zq/qt01k9w5zq.pdf),
  [Stern, Wang & Brown, ch. 5](https://www.cs.cmu.edu/~rms/BinauralWeb/papers/SternWangBrownChapter.pdf)).
- **Short sounds** give the listener no time to turn their head and compare,
  which is one of the main ways people resolve direction. This is a general
  psychoacoustic principle; I didn't find a specific citation for a duration
  threshold, so tune by ear.
- **Soft onsets** (fade in instead of an instant start) remove the sharp
  transient the ear locks onto. Optional, and only possible on a pin with PWM
  duty control (§5.2).
- **Random, rare timing.** By the time someone stands up, it's over and the
  next one is minutes away.
- Placement matters as much as the signal: behind or inside something, near
  hard reflective surfaces.

### 4.2 Sound profiles (needs a passive buzzer, except where noted)

| Profile | Recipe | Notes |
|---------|--------|-------|
| **Cricket** (default) | Carrier ~4.5–5 kHz. A chirp is 3–5 syllables at about 20–30 per second (e.g. 15 ms on / 18 ms off). Repeat 1–3 chirps at 2–3 chirps per second | Matches real field crickets: about 4.5–5 kHz carrier, 3–5 pulses per chirp, 20–30 Hz pulse rate, 2–3 chirps/s ([Gryllus bimaculatus, PMC](https://pmc.ncbi.nlm.nih.gov/articles/PMC3500458), [Auditory behavior of the cricket](https://link.springer.com/article/10.1007/BF00612706)). A "less authentic but harder to locate" variant moves the carrier down to ~3 kHz |
| **Lone chirp** | One ~60–100 ms beep around 3 kHz | Reminiscent of a smoke detector's low-battery chirp, which is famously maddening. Caveat for an office: people may report it to facilities or start inspecting smoke detectors |
| **Tweet** | Fast upward sweep ~2 → 4 kHz in ~60 ms, 1–2 times | Bird or "phone notification from nowhere" |
| **Tick** | 1–3 very short clicks (1–2 ms pulses) | Barely there. Works with an active buzzer too |

Start with **Cricket** only. Add other profiles one at a time after the
owner has heard the first one.

### 4.3 Active vs passive buzzer

- **Passive** (just a transducer, needs a square wave): any pitch, sweeps,
  all profiles above. **Required for a convincing cricket.**
- **Active** (built-in oscillator, fixed pitch, typically ~2–4 kHz): only
  on/off timing. A cricket rhythm (syllable bursts) is still possible but at
  the buzzer's fixed pitch.
- A piezo is loudest at its resonant frequency (often 2.5–4 kHz). The
  `TEST_MODE` build should include a slow frequency sweep, so the owner can
  hear where it's loudest.

## 5. Technical design

### 5.1 Toolchain

- **PlatformIO in VS Code**, `platform = atmelavr`, `board = attiny85`, **no
  framework** (plain avr-gcc + avr-libc). The Arduino core isn't needed, and
  leaving it out keeps Timer0 free and the code transparent.
- **Don't change fuses.** Factory default is 8 MHz internal RC divided by 8
  = 1 MHz. Switch to 8 MHz at runtime via the `CLKPR` register at the top of
  `main()` and set `board_build.f_cpu = 8000000L`. That avoids fuse writes
  entirely. **Never** set `RSTDISBL` (it locks out ISP programming).
- Starting `platformio.ini` (verify against current PlatformIO docs for
  Arduino-as-ISP):

  ```ini
  [env:attiny85]
  platform = atmelavr
  board = attiny85
  board_build.f_cpu = 8000000L
  upload_protocol = stk500v1
  upload_port = COM3          ; the Mega's port; owner fills in
  upload_speed = 19200
  upload_flags =
      -P$UPLOAD_PORT
      -b$UPLOAD_SPEED
  ```

### 5.2 Pin choice

ATtiny85 DIP pinout: 1 PB5/RESET, 2 PB3, 3 PB4, 4 GND, 5 PB0, 6 PB1,
7 PB2, 8 VCC.

- **Avoid PB3 and PB4** when the chip sits in the USB board. Digispark-style
  boards wire them to USB D−/D+ with zener diodes and a 1.5 kΩ pull-up. This
  is typical for these boards but not verified for this specific one.
- **Avoid PB5** (RESET).
- **Candidates:**
  - **PB1 / OC1A (Timer1):** frequency *and* duty control, so soft fades are
    possible. But Digispark-style boards often have an LED on PB1, which
    would flash with every sound and give the hiding place away. Use PB1
    only if the board has no LED there, or after removing that LED or its
    resistor.
  - **PB0 / OC0A (Timer0, CTC toggle):** clean 50% square wave at any
    frequency, no duty control. The simplest choice.
- Tone generation: hardware timer in CTC mode toggling the output pin. No
  bit-banging. Example: Timer0 at 8 MHz, prescaler 8: f = 1 MHz / (2 ·
  (OCR0A + 1)), so 4.5 kHz → OCR0A ≈ 110.
- **Silent state at reset:** pins are high-impedance until firmware runs.
  Add a 10 kΩ resistor holding the module input at its "off" level (pull-down
  for a high-triggered module, pull-up to 5 V for a low-triggered one). After
  a tone, firmware disconnects the timer from the pin and drives it to the
  off level.

### 5.3 Timing and sleep

- Long waits: watchdog timer interrupt plus power-down sleep (e.g. 1 s or 8 s
  ticks). The watchdog oscillator is only about ±10% accurate, which is fine
  for random minutes.
- Sound timing: busy-wait or Timer-based ms delays are fine during the short
  activation.
- USB-powered, so sleep isn't about battery life. It just keeps the design
  ready if battery comes back.

### 5.4 Randomness

- Combine (a) a boot counter in EEPROM, incremented every power-up, with
  (b) entropy from watchdog-vs-CPU-clock jitter: count CPU timer ticks across
  a few dozen watchdog periods and keep the low bits. (c) An ADC reading of a
  floating or internal channel is optional.
- Feed the result into a small PRNG (xorshift32). Draw intervals with an
  unbiased method (rejection sampling, not plain `%`).
- Chip erase during reflash wipes EEPROM. That's harmless.

### 5.5 Size

8 KB flash and 512 B RAM. The design above should use well under a quarter
of that. Keep sound profiles as small tables in flash (`PROGMEM`).

## 6. Open questions to ask the owner first

1. **Buzzer type.** Photo or markings of the 3-wire module. Quick test:
   connect VCC and GND to 5 V, then touch the signal pin to the level that
   triggers it. A continuous tone means **active**; a single click means
   **passive**. Also find out which level triggers it (high or low); many
   modules with a PNP transistor are **low-triggered**.
2. **USB board LED.** Is there an LED on PB1 (pin 6)? Check for a small LED
   and resistor on the board, or test continuity from socket pin 6.
3. **Quiet period after power-on.** 10 minutes OK?
4. **Interval.** Keep 5–15 minutes?
5. **Repo name** for the new project.
6. **Power switch.** A latching switch in series with USB 5 V is assumed.
   (A momentary "soft power" button needs extra circuitry. Not recommended
   for "very simple".)

## 7. Working agreement for the new session

- **The owner can't connect Claude to their PC.** All hardware work is
  copy-paste between VS Code and the Claude Code web session:
  1. Claude writes code, commits, and pushes to the session branch.
  2. Claude gives the owner **exact** steps: `git pull`, the PlatformIO
     command or button, what output to expect, and what to paste back.
  3. The owner runs them in VS Code and pastes the **full** terminal output.
  4. One step per round. No multi-step leaps without confirmation.
- **Before telling the owner to build:** try to compile in the cloud
  sandbox and report honestly whether it worked. Known sandbox limits from
  the previous project: the PlatformIO registry, docs.m5stack.com,
  espressif.com and dl.google.com were blocked; GitHub, PyPI and pub.dev
  worked. For AVR, try `apt-get install gcc-avr avr-libc` or PlatformIO with
  manual package installs. If nothing works, say so and rely on the owner's
  build.
- **Pure logic is host-testable.** PRNG, interval drawing and sound-profile
  timing tables don't depend on the AVR. Keep them in a header that compiles
  with host gcc, and add a few unit tests.
- **Stop for review** after each milestone: (1) design notes and answers to
  §6, (2) blink/beep "hello" over ISP, (3) cricket in `TEST_MODE`,
  (4) full schedule build, (5) README.
- US English in code, comments, docs and messages. Ask rather than assume.
  Be direct about uncertainty. Never claim something works without running
  it.

## 8. Programming setup (Arduino Mega as ISP)

To be verified with the owner's existing rig. The typical Mega wiring:

| Mega | ATtiny85 pin |
|------|--------------|
| D51 (MOSI) | 5 (PB0) |
| D50 (MISO) | 6 (PB1) |
| D52 (SCK)  | 7 (PB2) |
| D10 (RESET, per ArduinoISP sketch) | 1 (PB5/RESET) |
| 5V | 8 (VCC) |
| GND | 4 (GND) |

- 10 µF capacitor between the Mega's RESET and GND, after uploading
  ArduinoISP, to stop the Mega auto-resetting when avrdude connects.
- The ArduinoISP sketch defaults to 19200 baud and uses D10 for target reset.
  Confirmed from the sketch source.
- **Program the chip on the Mega rig, then move it to the USB board.** Don't
  leave the buzzer connected to PB0 or PB1 during programming: those are
  MOSI/MISO, and the module input can corrupt programming.

## 9. Deliverables in the new repo

```
platformio.ini
src/main.c              setup, schedule loop, sleep
src/sound.c / sound.h   timer tone driver + profiles
src/rng.c / rng.h       seeding + xorshift + unbiased range
src/config.h            all tunable constants
test/                   host unit tests for rng and profile timing
README.md               wiring (ASCII), programming via Mega ISP,
                        build/flash in VS Code, tuning with TEST_MODE
.gitignore
```
