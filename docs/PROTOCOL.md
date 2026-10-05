# annoying_esp BLE protocol

Protocol revision: **1**. This document is the contract between the firmware
(`firmware/`) and the Android app (`app/`). Both sides implement exactly what
is written here. If the code and this document disagree, the code is wrong.

Conventions:

- All multi-byte integers are **little-endian**, unsigned unless stated.
- Offsets are in bytes from the start of the characteristic value.
- "Reserved" bytes are sent as `0x00` and ignored on receipt.
- "Clamp" means: a value below the minimum becomes the minimum, a value above
  the maximum becomes the maximum. Clamping never fails a write.
- Durations in milliseconds are `ms`, in seconds are `s`.

---

## 1. GATT layout

One primary service, three characteristics. All UUIDs share the random base
`65f2xxxx-a1c0-42e0-a59e-1f247a1bb1c9`.

| Name    | UUID                                   | Properties     | Value size |
|---------|----------------------------------------|----------------|------------|
| Service | `65f20001-a1c0-42e0-a59e-1f247a1bb1c9` | primary        | n/a        |
| Config  | `65f20002-a1c0-42e0-a59e-1f247a1bb1c9` | read, write    | 24 bytes   |
| Control | `65f20003-a1c0-42e0-a59e-1f247a1bb1c9` | write          | 1 byte     |
| Status  | `65f20004-a1c0-42e0-a59e-1f247a1bb1c9` | read, notify   | 20 bytes   |

Writes use **Write Request** (with response). Write Command (without response)
is not supported, so the client always learns whether a write was accepted.

### 1.1 Advertising

- Advertising data: flags + complete list of 128-bit service UUIDs (the
  service UUID above). The app filters scans on this UUID.
- Scan response: complete local name, default `annoying_esp`. The name is a
  compile-time setting (`BLE_DEVICE_NAME` in `firmware/include/app_config.h`),
  max 20 characters.
- The name does not fit in the 31-byte advertising packet next to a 128-bit
  UUID, which is why it lives in the scan response. Android requests scan
  responses in active scan mode (the default), so the name still shows up.
- At most **one** central is connected at a time. Advertising stops while
  connected and restarts on disconnect.

### 1.2 MTU

The client SHOULD request an ATT MTU of at least 64 after connecting (the
device accepts up to 247). Both Config (24 bytes) and Status (20 bytes) fit
in a single read at that MTU. With the default MTU of 23:

- Status (20 bytes) still fits in one notification (MTU − 3 = 20).
- Config reads still work (the stack falls back to Read Blob).
- A Config write of 24 bytes needs a long (prepared) write, which the device
  accepts. `flutter_blue_plus` refuses long writes unless `allowLongWrite` is
  set, so the app requests MTU 247 on connect (the `flutter_blue_plus`
  default on Android) and treats a negotiated MTU below 27 as an error.

---

## 2. Security

Goal: a stranger with a BLE scanner in the same office cannot read, change,
or trigger anything.

| Parameter            | Value                                                    |
|----------------------|----------------------------------------------------------|
| Pairing              | LE Secure Connections only (legacy pairing is rejected)  |
| Bonding              | Yes                                                      |
| MITM protection      | Required                                                 |
| Device IO capability | DisplayOnly (`BLE_HS_IO_DISPLAY_ONLY`)                   |
| Association model    | Passkey Entry: the phone user types the device passkey   |
| Passkey              | Static 6-digit number from `firmware/include/secrets.h`  |
| Bond storage         | NVS on the device, OS bond store on the phone            |

The board has no display, so "DisplayOnly" is a deliberate fiction: the
"displayed" passkey is the static one the owner knows. A static passkey is
weaker than a random one (anyone who sniffs one pairing and brute-forces it
offline learns it, and it never rotates), but with LE Secure Connections the
passkey is not exposed to a passive sniffer the way legacy pairing exposed
the TK. It is the practical option without a display. Keep the passkey out of
git and do not reuse a trivial one (`000000`, `123456`); the firmware refuses
to compile with those.

### 2.1 Attribute permissions

| Characteristic  | Read                     | Write                    |
|-----------------|--------------------------|--------------------------|
| Config          | encrypted + authenticated | encrypted + authenticated |
| Control         | n/a                      | encrypted + authenticated |
| Status          | encrypted + authenticated | n/a                      |
| Status CCCD     | stack default            | stack default            |

"Authenticated" means the link is encrypted with a key produced by an
MITM-protected pairing (passkey entry), not "Just Works". An unauthenticated
access is answered by the stack with ATT error `0x05` (Insufficient
Authentication) or `0x0F` (Insufficient Encryption). Android reacts to that
error by starting pairing, which is what produces the passkey dialog on first
connect.

Subscribing to notifications (writing the Status CCCD) may be allowed by the
stack before pairing. That leaks nothing on its own, because the firmware
additionally:

- sends Status notifications only on links that are encrypted and
  authenticated, and
- disconnects any central that has not completed authenticated pairing
  within **30 s** of connecting.

### 2.2 Pairing flow on Android (first connect)

1. The app connects and requests the MTU.
2. The app reads Status. The device answers `0x05`.
3. Android shows the system pairing dialog with a passkey entry field.
4. The user enters the passkey from `secrets.h`.
5. On success both sides store the bond; the app retries the read and
   continues. Later connections encrypt automatically, with no dialog.
6. On a wrong passkey, pairing fails (SMP "Passkey Entry Failed"), the bond is
   not stored, and the device disconnects. The app shows "Pairing failed or
   wrong passkey" and offers retry.

The app may also call `createBond()` explicitly before step 2. Either path
leads to the same system dialog.

### 2.3 Clearing bonds

Phone side: Android Settings → Connected devices → `annoying_esp` → Forget.
The app also offers "Forget device", which removes the bond via
`removeBond()` (the device must then be re-paired).

Device side, either of:

- **Button:** hold the onboard button (GPIO0) for **10 s** while the firmware
  is running. All bonds are deleted and the device reboots. (Do not hold it
  while plugging in or pressing reset: GPIO0 low at reset enters the ROM
  download mode.)
- **Build flag:** build with `-DCLEAR_BONDS_ON_BOOT=1`, flash, boot once,
  then rebuild without it.

If only one side forgets the bond, the next connection fails with an
encryption error. Clear the other side too, then pair again.

---

## 3. Config characteristic

Read returns the active, already-clamped configuration. Write replaces the
whole configuration; partial writes are not supported.

### 3.1 Layout (version 1, 24 bytes)

| Offset | Type | Field                | Range (clamped)  | Default | Notes                                   |
|-------:|------|----------------------|------------------|--------:|-----------------------------------------|
| 0      | u8   | `layout_version`     | must be `1`      | 1       | Write rejected if not 1                 |
| 1      | u8   | `drive_mode`         | 0, 1             | 0       | 0 = DIGITAL, 1 = PWM. Other → 0         |
| 2      | u8   | `pwm_duty_pct`       | 1 … 100          | 50      | PWM only; ignored in DIGITAL            |
| 3      | u8   | `segments`           | 1 … 20           | 3       | Segments per activation                 |
| 4      | u8   | `pulses_per_segment` | 1 … 20           | 2       | Pulses per segment                      |
| 5      | u8   | reserved             |                  | 0       |                                         |
| 6      | u16  | `pulse_ms`           | 10 … 2000        | 80      | ON time per pulse. Hard cap 2000 ms     |
| 8      | u16  | `pulse_gap_ms`       | 10 … 5000        | 120     | OFF time between pulses in a segment    |
| 10     | u16  | `segment_gap_ms`     | 10 … 30000       | 600     | OFF time between segments               |
| 12     | u16  | `pwm_freq_hz`        | 100 … 20000      | 2000    | PWM only; ignored in DIGITAL            |
| 14     | u16  | reserved             |                  | 0       |                                         |
| 16     | u32  | `interval_min_s`     | 60 … 86400       | 300     | Shortest wait between activations       |
| 20     | u32  | `interval_max_s`     | 60 … 86400       | 900     | Longest wait; see rule below            |

Defaults equal "3 segments of 2 short pulses, every 5 … 15 minutes".

### 3.2 Validation rules (applied on the device, in this order)

1. Value length ≠ 24 → reject with ATT `0x0D` (Invalid Attribute Value
   Length). Nothing changes.
2. `layout_version` ≠ 1 → reject with application error `0x80`
   (Unsupported Version). Nothing changes.
3. Clamp every numeric field to its range in the table. Unknown `drive_mode`
   becomes 0 (DIGITAL). Reserved bytes are zeroed.
4. If `interval_max_s` < `interval_min_s`, set `interval_max_s` =
   `interval_min_s`. `min == max` is valid and gives a fixed interval.
5. Activation length cap: if the total activation duration (§5) exceeds
   **120 000 ms**, reduce `segments` by one until it fits or reaches 1, then
   reduce `pulses_per_segment` the same way. (A single pulse is at most
   2000 ms, so this always terminates within the cap.)
6. Store to NVS and apply. The write is answered with success.

Because clamping is silent, the client MUST read Config back after a write and
show the user any field that changed. The app does this on every Apply.

### 3.3 Effect on a running device

- A write never interrupts a running activation; the new pattern applies from
  the next activation.
- If the device is armed and the write changes `interval_min_s` or
  `interval_max_s`, the pending wait is discarded and a new interval is drawn
  from the moment of the write. If the interval range is unchanged, the
  pending wait continues.

---

## 4. Control characteristic

Write a single opcode byte.

| Opcode | Name    | Effect                                                                 |
|-------:|---------|------------------------------------------------------------------------|
| `0x01` | TRIGGER | Start one activation now. The schedule timer keeps running untouched.  |
| `0x02` | STOP    | Abort the running activation immediately; output LOW. No-op if idle.   |
| `0x03` | ARM     | Arm the schedule and draw the first interval from now. No-op if armed. |
| `0x04` | DISARM  | Cancel the schedule timer. Does **not** stop a running activation.     |

Errors:

| Condition                         | ATT error                         |
|-----------------------------------|-----------------------------------|
| Length ≠ 1                        | `0x0D` Invalid Attribute Value Length |
| Unknown opcode                    | `0x81` Unknown Opcode (application) |
| TRIGGER while an activation runs  | `0x82` Busy (application)         |

Interaction rules:

- **ARM/DISARM** persist the armed flag in NVS.
- **STOP during a scheduled activation** while armed: the next interval is
  drawn from the moment of the stop, as if the activation had ended normally.
- **Schedule timer expires during a manual (TRIGGER) activation:** the
  scheduled activation is merged into the running one. It does not start a
  second activation afterward. The next interval is drawn from the end of the
  running activation.
- **Local button long press (≥ 2 s, < 10 s):** same as DISARM + STOP.

---

## 5. Activation pattern

An activation is `S` segments; a segment is `P` pulses. With `S` =
`segments`, `P` = `pulses_per_segment`:

```
for s in 1..S:
    for p in 1..P:
        ON  pulse_ms
        if p < P: OFF pulse_gap_ms
    if s < S: OFF segment_gap_ms
```

The segment gap replaces the pulse gap between the last pulse of one segment
and the first pulse of the next; they are not added. There is no trailing
OFF time after the final pulse.

Total duration:

```
T = S·P·pulse_ms + S·(P−1)·pulse_gap_ms + (S−1)·segment_gap_ms
```

Example with defaults (S=3, P=2, 80/120/600): `3·2·80 + 3·1·120 + 2·600 =
480 + 360 + 1200 = 2040 ms`.

```
ON  ██  ██         ██  ██         ██  ██
OFF   ░░            ░░             ░░
       ^pulse gap ^segment gap
```

### 5.1 Drive modes

- **DIGITAL:** the output pin is driven HIGH during ON and LOW during OFF.
- **PWM:** during ON, the pin carries an LEDC square wave at `pwm_freq_hz`
  with `pwm_duty_pct` duty; during OFF it is LOW (duty 0, then pin LOW).

The buzzer is an **active** module with its own oscillator. Chopping its
supply with PWM mostly changes its average power, so expect little or no
change in pitch, possibly a quieter or rougher tone, or no audible difference
at all. PWM mode is provided for experimentation, not as a pitch control.

### 5.2 Safety

- The output is set LOW as the first action in `setup()` and on any error
  path.
- Independently of the pattern engine, a hardware timer forces the output LOW
  if it has been continuously ON for more than `pulse_ms` cap + 50 ms
  (2050 ms). If that ever fires, the activation is aborted and logged over
  USB serial.

---

## 6. Schedule

- **Armed:** wait a random interval, uniform over the integers
  `[interval_min_s, interval_max_s]`, then run one activation, then draw the
  next interval. The interval is counted from the **end** of the activation.
- Random numbers come from `esp_random()` with rejection sampling (no modulo
  bias).
- **Disarmed:** no scheduled activations. TRIGGER still works.
- **Boot:** the armed flag and Config are restored from NVS. If armed, a fresh
  interval is drawn from boot time. The previous countdown is **not**
  resumed: a power cut always restarts the wait.

---

## 7. Status characteristic

Readable at any time; notified to the subscribed (authenticated) client.

### 7.1 Layout (version 1, 20 bytes)

| Offset | Type | Field                | Notes                                                       |
|-------:|------|----------------------|-------------------------------------------------------------|
| 0      | u8   | `layout_version`     | `1`                                                         |
| 1      | u8   | `state`              | 0 = IDLE, 1 = WAITING, 2 = ACTIVE                           |
| 2      | u8   | `flags`              | bit 0 armed, bit 1 manual activation, bit 2 output ON; rest 0 |
| 3      | u8   | reserved             |                                                             |
| 4      | u32  | `next_activation_s`  | Seconds until the next scheduled activation; `0xFFFFFFFF` = none |
| 8      | u8   | `segment_index`      | 1-based current segment; 0 when not ACTIVE                  |
| 9      | u8   | `segment_count`      | `S` of the running activation; 0 when not ACTIVE            |
| 10     | u8   | `pulse_index`        | 1-based current pulse in the segment; 0 when not ACTIVE      |
| 11     | u8   | `pulses_per_segment` | `P` of the running activation; 0 when not ACTIVE            |
| 12     | u8   | `fw_major`           | Firmware version                                            |
| 13     | u8   | `fw_minor`           |                                                             |
| 14     | u8   | `fw_patch`           |                                                             |
| 15     | u8   | reserved             |                                                             |
| 16     | u32  | `activation_count`   | Activations completed or aborted since boot (manual + scheduled) |

### 7.2 State semantics

| `state`  | Meaning                                                     |
|----------|-------------------------------------------------------------|
| IDLE     | Not armed and no activation running                         |
| WAITING  | Armed, timer counting down, no activation running           |
| ACTIVE   | An activation is running (armed or not; see `flags`)        |

`next_activation_s`:

- Disarmed: `0xFFFFFFFF`.
- Armed and WAITING: remaining seconds, rounded up.
- Armed and ACTIVE with a **manual** activation: remaining seconds of the
  untouched schedule timer (it keeps running; it can reach 0, see §4).
- Armed and ACTIVE with a **scheduled** activation: `0xFFFFFFFF`, because the
  next interval is drawn only when the activation ends.

During a pulse gap, `segment_index`/`pulse_index` keep the values of the pulse
that just ended. During a segment gap, they keep the values of the last pulse
of the segment that just ended.

### 7.3 Notification policy

- Immediately on any change of `state` or `flags`, and after any Config
  write (the countdown may have been redrawn).
- Progress (`segment_index`, `pulse_index`) changes are coalesced to at most
  **10 notifications/s**; the final state of an activation is always sent.
- While connected and not ACTIVE: a heartbeat notification every **5 s**.
  The app counts down locally between notifications and resyncs on each one.

---

## 8. Application error codes

| Code   | Name                | Used by       |
|--------|---------------------|---------------|
| `0x80` | Unsupported Version | Config write  |
| `0x81` | Unknown Opcode      | Control write |
| `0x82` | Busy                | Control write |

Standard ATT errors used: `0x05`, `0x0D`, `0x0F`.

---

## 9. Versioning

- `layout_version` in Config and Status identifies the byte layout. A change
  that moves, resizes, or reinterprets a field increments it.
- New fields may be appended into reserved bytes without a version bump only
  if `0` keeps the old behavior.
- The app refuses to write Config if the Config it read has a
  `layout_version` it does not know, and shows "Firmware too new/old".
- Firmware version (`fw_major.fw_minor.fw_patch`) is informational.

---

## 10. Transport independence

Everything above §1 is BLE-specific; §3–§7 are not. The firmware's command
layer takes the same Config bytes and Control opcodes from any transport, and
produces the same Status bytes. A later Wi-Fi/MQTT transport would publish
the Status blob and accept Config/Control blobs on topics, without changes to
the scheduler or pattern engine. That transport would need its own
authentication; BLE pairing does not cover it.
