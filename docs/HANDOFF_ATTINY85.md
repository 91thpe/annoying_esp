# Handoff: ATtiny85 office annoyance

This is the starting brief for the new repository **`annoying_attiny85`**
and a fresh Claude Code session.

Repo description: *Hidden office noise maker: an ATtiny85 on a mini-breadboard
plays short, hard-to-locate chirps at random 5–15 minute intervals. Plain
avr-gcc in PlatformIO, flashed via an Arduino Mega as ISP.*

This brief Paste it (or commit it as `docs/HANDOFF.md`) at the start of the new
session. It replaces the earlier ESP32-S3/BLE "annoying_esp" project, which
is parked.

## 1. Goal

A tiny, standalone noise maker hidden in an office. It makes a short,
puzzling sound ("what the heck is that?") at random intervals, for example
somewhere between 5 and 15 minutes apart. Each sound is short so nobody can
walk toward it and find the device.

**The target is "annoying and hard to place", not realism.** Insect or
cricket-like rhythms are a starting idea, not a requirement to sound like a
real cricket. Pick pitch and rhythm for maximum annoyance and minimum
locatability.

**Keep it very simple.** No radio, no app, no remote control. All
parameters are compile-time constants at the top of one source file. To
change behavior, edit and reflash.

## 2. Hardware (as stated by the owner)

| Item | Detail |
|------|--------|
| MCU | ATtiny85, bare DIP-8 |
| Final build | ATtiny85 on an adhesive-backed mini-breadboard. **The AliExpress micro-USB ATtiny85 board is not part of the final design** (at most a bench power source during development) |
| Programmer | Arduino Mega running the ArduinoISP sketch (owner built it earlier) |
| Buzzer | Generic 3-wire breadboard buzzer module (SIG / VCC / GND). **Passive, high-triggered:** the owner tested it, and it clicks when SIG goes to 5 V and makes no tone on steady DC |
| Power | USB 5 V via the owner's USB breakout board on the breadboard. On/off is a generic switch in the 5 V line, or simply unplugging a jumper wire. Battery is out of scope |
| Spare parts | 2N3906 PNP transistors (useful now that logic is 5 V) |
| Dev environment | Owner's VS Code + PlatformIO on their own PC |

## 3. Behavior requirements

1. **Power on = running.** There is no arm/disarm switch. The power switch is
   the only control.
2. **Never sound at power-on.** After power-up, wait a quiet period first
   (10 min, confirmed by the owner), then start the random schedule.
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

The owner wants annoying, not authentic. So: put the carrier in the
**~2–3 kHz** band (hardest to localize, and near a typical piezo's loudest
point) rather than at a real cricket's 4.5–5 kHz. Use the insect rhythm
only because rapid, irregular bursts are irritating and hard to place.

| Profile | Recipe | Notes |
|---------|--------|-------|
| **Cricket-ish** (default) | Carrier ~2.5–3 kHz (a real cricket is ~4.5–5 kHz). A chirp is 3–5 syllables at about 20–30 per second (e.g. 15 ms on / 18 ms off). Repeat 1–3 chirps at 2–3 chirps per second | Rhythm borrowed from field crickets (3–5 pulses per chirp, 20–30 Hz pulse rate, 2–3 chirps/s; [Gryllus bimaculatus, PMC](https://pmc.ncbi.nlm.nih.gov/articles/PMC3500458), [Auditory behavior of the cricket](https://link.springer.com/article/10.1007/BF00612706)). Carrier deliberately lowered into the hard-to-locate band |
| **Lone chirp** | One ~60–100 ms beep around 3 kHz | Reminiscent of a smoke detector's low-battery chirp, which is famously maddening. Caveat for an office: people may report it to facilities or start inspecting smoke detectors |
| **Tweet** | Fast upward sweep ~2 → 4 kHz in ~60 ms, 1–2 times | Bird or "phone notification from nowhere" |
| **Tick** | 1–3 very short clicks (1–2 ms pulses) | Barely there. Works with an active buzzer too |

Start with **Cricket-ish** only. Add other profiles one at a time after the
owner has heard the first one.

### 4.3 Active vs passive buzzer

- **Passive** (just a transducer, needs a square wave): any pitch, sweeps,
  all profiles above. More variety, so more annoyance.
- **Active** (built-in oscillator, fixed pitch, typically ~2–4 kHz): only
  on/off timing. Burst rhythms, lone chirps and ticks still work at the
  buzzer's fixed pitch. **The owner has said that's acceptable.** The
  firmware should support both via one compile-time switch
  (`BUZZER_PASSIVE`): passive gets a timer-generated tone, active gets the
  pin switched on/off with the same rhythm tables.
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

- The final build is a bare chip on a breadboard, so the USB board's
  constraints (USB lines on PB3/PB4, LED on PB1) **don't apply**. Only
  **avoid PB5** (RESET).
- **Use PB1 / OC1A (Timer1)** for the buzzer signal: frequency *and* duty
  control, so soft fades are possible later. Timer1 on the ATtiny85 can
  clock from the CPU clock with a wide prescaler range and uses `OCR1C` as
  TOP in CTC mode.
- Fallback: PB0 / OC0A (Timer0, CTC toggle) gives a clean 50% square wave
  with no duty control.
- Tone generation: hardware timer in CTC mode toggling the output pin. No
  bit-banging. Example with Timer0 at 8 MHz, prescaler 8: f = 1 MHz / (2 ·
  (OCR0A + 1)), so 2.7 kHz → OCR0A ≈ 184. Compute the Timer1 equivalent in
  the new session from the datasheet.
- **Breadboard essentials for a bare chip:** 100 nF ceramic capacitor
  directly across VCC (pin 8) and GND (pin 4); about 10 µF bulk on the 5 V
  rail; 10 kΩ pull-up from RESET (pin 1) to VCC. The internal reset pull-up
  is weak, and an office full of switching noise is a good way to find out.
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

Already answered by the owner: a 10-minute quiet period after power-on is
fine; a 5–15-minute interval is fine; no fuse changes; the target is
"annoying", not realism; either buzzer type is acceptable; the final build
is a bare chip on an adhesive mini-breadboard (no USB board); power comes
from a USB breakout board, switched by a generic switch or by pulling a
jumper wire; the repo is `annoying_attiny85`.

Buzzer, resolved: **passive, triggered high** (clicks when SIG goes to 5 V).
Consequences for the design:

- `BUZZER_PASSIVE = 1`; tones come from Timer1 on PB1 as designed.
- **10 kΩ pull-down** from SIG to GND keeps it silent before firmware runs.
- **Idle level is LOW.** Never leave SIG high between sounds: on a magnetic
  passive buzzer, that holds DC through the coil (wasted current, heat, no
  sound). After every tone, disconnect the timer from PB1 and drive it LOW.
- The 2.5–3 kHz carrier is a starting point; the `TEST_MODE` sweep shows
  where this particular buzzer is loudest.

No open hardware questions remain. Confirm the defaults with the owner and
start milestone 1.

Note for the README: **power-on with a pulled jumper.** Plugging a wire in
can bounce power on and off a few times. That's harmless (each bounce just restarts the
quiet period).

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
- **Program the chip on the Mega rig, then move it to the final
  breadboard.** Don't leave the buzzer connected to PB0 or PB1 during
  programming: those are MOSI/MISO (the buzzer will sit on PB1), and the
  module input can corrupt programming. Alternatively, wire a 6-pin ISP
  header on the final breadboard and unplug the buzzer signal wire while
  flashing.

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
