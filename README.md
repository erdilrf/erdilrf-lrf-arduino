# ERDILRF_LRF

Arduino library for **ERDI 905 nm laser rangefinder modules** — the LR1000E2 UART protocol family.

Install it from the Arduino IDE via **Tools → Manage Libraries…** and search for `ERDILRF_LRF`.

With **PlatformIO**, install it by name from the PlatformIO Registry — verified by actually resolving
and compiling it:

```ini
lib_deps = erdilrf/ERDILRF_LRF
```

```text
Library Manager: Installing erdilrf/ERDILRF_LRF
Library Manager: ERDILRF_LRF@0.2.1 has been installed!
```

Installing straight from Git also works, and tracks the repository rather than a release:

```ini
lib_deps = https://github.com/erdilrf/erdilrf-lrf-arduino.git
```

> **A note on this section's history, because it was wrong twice in opposite directions.**
> An earlier revision gave the registry form *before* the library was in the PlatformIO Registry —
> that failed with `UnknownPackageError`. It was then replaced by the Git form plus a warning that the
> registry form did not work. By the time that warning was published, the registry form *did* work.
> Both revisions asserted something about the registry without re-checking it. As of this revision both
> forms are verified working, and the registry form is given first because it is the one PlatformIO
> users expect.

```cpp
#include <ERDILRF_LRF.h>

ErdilrfLrf lrf(Serial1);

void setup() {
  Serial1.begin(115200);          // default link; see "What the datasheet does not say"
}

void loop() {
  float d = lrf.singleShot();
  if (d >= 0.0f)       Serial.println(d);            // metres
  else if (d > -1.5f)  Serial.println("shot failed"); // module answered, STA != 1
  else                 Serial.println("no answer");   // nothing came back — check wiring
  delay(500);
}
```

## The protocol in one screen

Fixed 8-byte frames. The checksum is the low byte of a sum — and **the two directions use different
ranges**, which is the mistake that stalls most first integrations.

```text
offset   0     1     2      3    4    5     6      7
        0x55  0xAA  FUNC   D1   D2   D3    D4     SUM

host  -> module   SUM = (FUNC + D1 + D2 + D3 + D4) & 0xFF
module -> host    SUM = (0x55 + 0xAA + FUNC + D1 + D2 + D3 + D4) & 0xFF
```

| Command | Transmit | Response |
|---|---|---|
| Single-shot ranging | `55 AA 88 FF FF FF FF 84` | `55 AA 88 STA FF DIS_H DIS_L SUM` |
| Continuous ranging | `55 AA 89 FF FF FF FF 85` | `55 AA 89 STA FF DIS_H DIS_L SUM` |
| Stop ranging | `55 AA 8E FF FF FF FF 8A` | `55 AA 8E STA FF FF FF SUM` |
| Angle measurement † | `55 AA 8A FF FF FF FF 86` | `55 AA 8A STA FF ANG_H ANG_L SUM` |
| Set baud rate | `55 AA TYPE FF FF FF FF SUM` | `55 AA TYPE STA FF FF FF SUM` |

Distance and angle are scaled by 10 — the register holds tenths, so
`((DIS_H << 8) | DIS_L) / 10` gives metres. † Angle is only available on modules with an angle sensor.

### API

| Method | Returns |
|---|---|
| `singleShot(timeoutMs)` | metres, or `-1.0` if the module answered with `STA != 1`, or `-2.0` if no frame arrived |
| `readFrame(frame, timeoutMs)` | `true` on a valid, checksum-checked frame |
| `readPowerOnSelfTest(frame, timeoutMs)` | the unsolicited `55 AA 80` frame — emitted **once, at power-on** |
| `requestSingleShot()` / `requestContinuous()` / `requestStop()` / `requestAngle()` | transmit only |
| `setLdConstantOn(bool)` | transmit only — see the gap below |
| `setBaudRate(typeCode)` | transmit only; takes effect **after the module is restarted** |
| `checksumSend()` / `checksumResponse()` | static; the two different rules |
| `parse()` / `decodeDistance()` / `decodeAngle()` | static; work on captured bytes |

`singleShot()` deliberately returns **three distinguishable outcomes**. Collapsing the last two hides
the difference between *a bad target* and *a bad cable*, which are the two things an integrator
actually needs to tell apart.

## What the datasheet does not say

Four gaps in the manufacturer's manual, and what this library does about each:

1. **The LD constant-on response frame is incomplete** — section 8 prints only `55` for that row.
   This library therefore sends the command and does **not** wait for or validate a reply. The missing
   bytes are not guessed.
2. **Parity and stop bits are not documented.** The manual gives 115200 and 8 data bits; this library
   assumes 8N1 and says so. If a module stays silent, try `SERIAL_8E1` before suspecting the cable —
   a parity mismatch produces *total* silence, which is easy to misread as a dead module.
3. **TXD/RXD are named from the module's point of view**, and no wire colours are documented.
   Module TXD → host RX. Straight-through TXD-to-TXD is the most common silent link.
4. **The enable pin's polarity is configuration-dependent.** The manual says "active high" and, in the
   same row, "compatible with active-low control requirements".

Full detail: [`docs/what-the-datasheet-does-not-say.md`](https://github.com/erdilrf/erdilrf-drivers/blob/main/docs/what-the-datasheet-does-not-say.md)
and [`protocol/PROTOCOL.md`](https://github.com/erdilrf/erdilrf-drivers/blob/main/protocol/PROTOCOL.md)
in the main repository.

## Verification status

**Read this before trusting the library.**

| Checked | How |
|---|---|
| Compiles for AVR | ✅ PlatformIO, `uno` + `nanoatmega328` (ATmega328P: 16-bit `int`, 2 KB RAM, no FPU) with `-Wall -Wextra`, **zero warnings from this driver** |
| Example sketch compiles | ✅ `pio ci --board=uno` — includes `SoftwareSerial` on AVR |
| Transmit frames match the manual | ✅ All six published frames reproduce byte for byte |
| Constants have not drifted | ✅ Cross-checked against a tested Python implementation in the main repository |
| **Behaviour on real hardware** | ❌ **Not verified.** No board, no module. |

Reproduce the build — no system C++ compiler is required, PlatformIO fetches its own toolchain:

```bash
pio run          # see platformio.ini
```

The build project also calls every public member and re-checks the six published checksums **on-target
at runtime**, printing `FAILURES=0` on a healthy build.

## Scope

Civilian industrial optical distance measurement: UAV payloads, mobile robots and AGVs, surveying
instruments, industrial safety and automation, handheld optical devices.

The module is **IEC 60825-1 Class 1**, 905 ± 5 nm. The manufacturer's own manual restricts the product
to civilian applications and excludes defence use.

## Related

* **[erdilrf/erdilrf-drivers](https://github.com/erdilrf/erdilrf-drivers)** — the full protocol
  reference, a zero-dependency Python host driver with 50 tests, wiring notes, and the datasheet-gap
  writeup. This library is developed there and published here so the Arduino Library Manager can index
  it (which requires `library.properties` at the repository root — impossible in a monorepo).

## License

MIT. See `LICENSE`.
