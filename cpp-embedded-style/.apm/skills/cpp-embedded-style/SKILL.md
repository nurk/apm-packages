---
name: cpp-embedded-style
description: >
  Coding style and architecture guidelines for embedded C++ projects targeting
  Arduino-framework microcontrollers on modern AVR cores (megaAVR 0-series,
  AVR-DA/DB/DD, tinyAVR 2-series) via PlatformIO or the Arduino IDE. Covers
  naming conventions, class design, hardware abstraction patterns, comment style,
  and ISR / peripheral access idioms.
  ALWAYS load this skill when: writing any .cpp or .h file, generating a new class,
  adding a method, reviewing code, or when the project uses Arduino, PlatformIO,
  AVR, ATmega, or any AVR embedded C++ framework. Apply without being asked.
---

Apply these guidelines every time you generate or modify C++ code for this project.

---

## 1. File & Header Structure

- One class per `.h` / `.cpp` pair; filename matches the class name exactly (PascalCase).
- Header guards use `#ifndef PROJECT_CLASSNAME_H` / `#define ...` / `#endif //PROJECT_CLASSNAME_H`.
  Replace `PROJECT` with the project name in SCREAMING_SNAKE_CASE.
- Include style: always use angle brackets (`#include <...>`), never double quotes (`#include "..."`).
- Include ordering is **not enforced** — list includes in whatever order is clearest for the file.
- `<Arduino.h>` is **not mandatory**. Only include it when the file directly uses Arduino API
  symbols (`pinMode`, `digitalWrite`, `millis`, Arduino type aliases, etc.). If all required
  symbols are provided by more specific headers (e.g. `<avr/io.h>`, a library header that pulls
  in `<Arduino.h>` transitively, or `<stdint.h>` for fixed-width types), omit `<Arduino.h>`.
- Some architecture headers are already pulled in transitively by `<Arduino.h>` (e.g.
  `<avr/pgmspace.h>`) and do not need to be listed explicitly. Others are **not** included
  transitively and must be listed (e.g. `<util/atomic.h>` for `ATOMIC_BLOCK`). When in doubt,
  add the explicit include — it is always safe to include a header more than once.

### Canonical header file skeleton

```cpp
#ifndef MYPROJECT_MOTORDRIVER_H
#define MYPROJECT_MOTORDRIVER_H

#include <Arduino.h>   // only if Arduino API symbols are used directly

class MotorDriver {
public:
    MotorDriver(uint8_t enPin, uint8_t dirPin);

    void        start();
    void        stop();
    void        setSpeed(uint8_t speed);
    bool        isRunning()  const;

private:
    void applyPwm(uint8_t duty);

    uint8_t enPin;
    uint8_t dirPin;
    bool    running;
    uint8_t currentSpeed;
};

#endif // MYPROJECT_MOTORDRIVER_H
```

Order within the class: deleted specials → constructor(s) → public methods → private methods → private members.

---

## 2. Naming Conventions

| Kind                              | Convention                  | Example                                   |
|-----------------------------------|-----------------------------|-------------------------------------------|
| Classes                           | PascalCase                  | `MotorDriver`, `DisplayController`        |
| Public methods                    | camelCase                   | `setSpeed()`, `turnOn()`                  |
| Private methods                   | camelCase                   | `computePwm()`, `getStorageOffset()`      |
| Private member variables          | camelCase (no trailing `_`) | `isRunning`, `targetSpeed`          |
| `const` members (class-scope)     | SCREAMING_SNAKE_CASE        | `MAX_SPEED`, `BAUD_RATE`                  |
| Local variables                   | camelCase                   | `elapsed`, `rawAdc`, `wasOn`              |
| Function parameters               | camelCase                   | `targetRpm`, `ledPin`                     |
| `#define` pin / hardware constants| SCREAMING_SNAKE_CASE        | `MOTOR_EN`, `STATUS_LED`                  |
| Enum values                       | SCREAMING_SNAKE_CASE        | `IDLE`, `RUNNING`, `ERROR`                |
| ISR vectors                       | MCU names verbatim          | `TCB0_INT_vect`, `TCA0_OVF_vect`, `PORTA_PORT_vect` |

---

## 3. Class Design & Architecture

### Dependency injection via constructor

Pass all hardware references and sibling objects through the constructor — never access globals inside a class.

```cpp
// Good — all dependencies explicit in constructor signature
SensorReader(uint8_t dataPin, uint8_t csPin, SPI& spi, Display& display);

// Bad — silently coupling to a global inside a method
void SensorReader::read() { SPI.transfer(...); }
```

### Constructor initializer lists

Always initialize members in the initializer list, not the constructor body, when possible.

```cpp
SensorReader::SensorReader(const uint8_t dataPin, const uint8_t csPin,
                           SPI& spi, Display& display)
    : dataPin(dataPin), csPin(csPin), spi(spi), display(display) { }
```

### `const` correctness

- Mark every method that does not mutate state as `const`.
- Mark every parameter that will not be modified as `const` **in the `.cpp` definition only**.
  Top-level `const` on value parameters has no effect in a declaration (`.h`) and is flagged by
  Clang-Tidy (`readability-avoid-const-params-in-decls`) — omit it from headers entirely.
  Reference and pointer parameters that must not be modified (`const T&`, `const T*`) carry
  `const` in both the declaration and the definition because it is part of their type.
- Return `const` references where appropriate to prevent unintended mutation.

### Private helpers

Prefer `private static` methods for pure computations that don't touch member state.
Use `private` non-static methods for computations that read (but don't write) member state.

### "save-state, modify, restore" pattern

When an operation must temporarily change hardware state, save and restore the on/running state:

```cpp
void Driver::reconfigure(const Config& cfg) {
    const bool wasRunning = isRunning;
    stop();
    // … apply new configuration to hardware …
    if (wasRunning) start();
}
```

### Single responsibility per class

Each class owns exactly one hardware subsystem or one UI concern.
`main.cpp` wires them together; the classes themselves don't know about each other
unless explicitly injected.

## 4. Hardware Access Idioms

### ISRs

- ISRs live in `main.cpp`, never inside class files.
- Keep ISRs as short as possible: read/toggle/clear the flag, return.
- If an ISR needs to communicate with a class, do it through a `volatile` flag or
  a dedicated method that only touches `volatile` data.

```cpp
// main.cpp
ISR(TCB0_INT_vect) {
    PORTB.OUTTGL  = PIN0_bm;
    TCB0.INTFLAGS = TCB_CAPT_bm;  // clear the interrupt flag
}
```

### Pin initialisation

Use `digitalWriteFast()` for GPIO in setup sequences or tight loops where cycle count matters.
It is a third-party library (`digitalWriteFast` by Watterott / NicksonYap) and must be declared
as a PlatformIO `lib_dep`. Use `digitalWrite()` / `digitalRead()` everywhere else.

### Atomic access for ISR-shared variables

On 8-bit AVR MCUs, reads and writes of any type wider than 8 bits are **not atomic** — an ISR
firing mid-read can corrupt the value. Always protect shared multi-byte variables with
`ATOMIC_BLOCK` from `<avr/atomic.h>`:

```cpp
#include <avr/atomic.h>

// In the ISR (no protection needed — ISRs are non-reentrant by default)
ISR(TCA0_OVF_vect) {
    tickCount++;            // volatile uint16_t — safe to write here
    TCA0.SINGLE.INTFLAGS = TCA_SINGLE_OVF_bm;  // clear the flag
}

// In main-loop / class code — wrap every multi-byte volatile read or write
uint16_t Driver::getTicks() const {
    uint16_t snapshot;
    ATOMIC_BLOCK(ATOMIC_RESTORESTATE) {
        snapshot = tickCount;
    }
    return snapshot;
}
```

Rules:
- Use `ATOMIC_RESTORESTATE` (not `ATOMIC_FORCEON`) so that code called from inside an ISR
  doesn't accidentally re-enable interrupts.
- Single `uint8_t` / `bool` reads and writes are inherently atomic on AVR — no block needed.

### Peripheral register writes — configure-then-enable

Disable the peripheral before changing any configuration register, then write the enable bit last.

```cpp
PERIPH->CTRLA = 0;        // disable first
PERIPH->CTRLB = MODE_gc;  // configure
PERIPH->CCMP  = period;
PERIPH->INTCTRL = CAPT_bm;
PERIPH->CTRLA = CLKSEL_gc; // enable bit intentionally omitted — caller sets it
```

### Hardware interrupt wiring

Prefer direct register writes over `attachInterrupt()` to eliminate overhead and indirection:

```cpp
PORTA.PIN3CTRL |= PORT_ISC_BOTHEDGES_gc;
```

### F() macro — keep strings in flash

Wrap all string literals passed to `Serial`, LCD, or display drivers with `F()`:

```cpp
Serial.println(F("System ready"));
lcd.print(F("Sensor error"));
```

### PROGMEM — keep large const data in flash

On the target AVR cores (megaAVR 0-series, AVR-DA/DB/DD, tinyAVR 2-series) flash is
memory-mapped into the unified data address space, so `PROGMEM` data can be read with
normal array indexing — no `pgm_read_*` helpers required.

Declare large read-only arrays with `PROGMEM` to prevent the linker from copying them
into SRAM at startup:

```cpp
// Data stays in flash; readable like a normal array
// (PROGMEM is available via Arduino.h — no extra include needed)
static const uint8_t SINE_TABLE[256] PROGMEM = { 128, 131, 134, /* … */ };

uint8_t Driver::sineAt(const uint8_t index) const {
    return SINE_TABLE[index];   // direct access — no pgm_read needed
}
```

Rules:
- `PROGMEM` variables must have `static` storage duration (file-scope or `static` local).
- Apply to lookup tables, font data, message strings, and any large read-only array.
- Prefer `PROGMEM` over `F()` for structured data; use `F()` only for inline string literals
  passed directly to `Serial` / display calls.

### Null-pointer guard for optional peripherals

When a peripheral pointer can legitimately be absent, guard every access:

```cpp
if (peripheral != nullptr) peripheral->doSomething();
```

---

## 5. Type Discipline

Use explicit-width types everywhere. Avoid bare `int` or `long` when the width matters.

| Use case                                      | Type        | Reason                                             |
|-----------------------------------------------|-------------|----------------------------------------------------|
| 8-bit register values / pin numbers           | `uint8_t`   | Matches hardware width                             |
| 16-bit register / timer values                | `uint16_t`  | Explicit width; avoids promotion surprises         |
| Counters / values that could exceed 16 bits   | `uint32_t`  | Safe for values that overflow 16 bits on AVR       |
| Values needing full 64-bit range              | `uint64_t`  | Use sparingly; expensive on 8-bit MCUs             |
| Signed differences / deltas                   | `int32_t` / `int64_t` | Transient; cast back to unsigned after clamp |
| Boolean flags                                 | `bool`      | Standard C++; `boolean` is deprecated in modern Arduino cores |
| Signed loop counters / array indices          | `int`       |                                                    |

Use `static_cast<>` rather than C-style casts:

```cpp
const auto period = static_cast<uint16_t>(clkHz / (2UL * targetHz) - 1);
```

---

## 6. Comment Style

### Architecture / rationale block (before a non-trivial function)

```cpp
// Periodic interrupt driver — pin toggled directly in the ISR.
//
// Architecture:
//   • The hardware timer fires an interrupt at 2× the target frequency.
//   • The ISR toggles the output pin via PORT.OUTTGL (~3 cycles).
//   • True 50% duty cycle; full 16-bit CCMP resolution.
//
// Formula:  f = clkHz / (2 × (CCMP + 1))
//
// Returns the actual frequency achieved (may differ slightly due to rounding).
uint32_t Driver::setPeriod(const uint32_t targetHz) {
```

- Bullet items use `•` (not `-` or `*`) inside rationale blocks.
- Formulas are written inline with `×` and `÷`.
- ASCII-aligned tables in comments for precision / range summaries.

### Inline comments

```cpp
PERIPH->CTRLA = clkSel; // ENABLE_bm intentionally omitted — start() sets it
```

- Single space after `//`, sentence-case text.
- Explain *why*, not *what* — the code already says what.

### File-top block (optional)

Use a `/** … */` block at the top of `main.cpp` listing software/hardware revision,
board target, and URLs for all non-standard libraries used:

```cpp
/**
 * Project:  MyProject
 * File:     main.cpp
 * Board:    AVR128DA48 Curiosity Nano
 * Revision: hw=1.0  sw=0.3.1
 *
 * Libraries:
 *   • digitalWriteFast  https://github.com/NicksonYap/digitalWriteFast
 */
```

---

## 7. Formatting

- **Indentation:** 4 spaces, no tabs.
- **Brace style:** K&R — opening brace on the same line as the statement or function
  signature; `else` on the same line as the closing brace of the preceding `if`:

```cpp
if (speed > MAX_SPEED) {
    speed = MAX_SPEED;
} else if (speed == 0) {
    stop();
} else {
    applyPwm(speed);
}
```

Always use braces, even for single-statement bodies. This applies to macro-based
block constructs (`ATOMIC_BLOCK`, `NONATOMIC_BLOCK`, etc.) as well — never collapse
them to a single line:

```cpp
// Correct — body on its own line even though it's a single statement
ATOMIC_BLOCK(ATOMIC_RESTORESTATE) {
    snapshot = sharedValue;
}

// Wrong — collapsed to one line
ATOMIC_BLOCK(ATOMIC_RESTORESTATE) { snapshot = sharedValue; }
```

- **Blank lines:** one blank line between methods in `.cpp`;

- **Multi-line argument lists:** align continuation lines to the column after the opening `(`:

```cpp
SensorReader reader(DATA_PIN,
                    CS_PIN,
                    spi,
                    display);
```

- **Trailing newline:** every file ends with exactly one newline.
- **Line length:** soft limit 160 chars; favour readability over the limit.

- **`auto`:** use only where the type is unambiguous from the right-hand side.
  When the right-hand side is a `static_cast<T>(...)`, `auto` is acceptable because
  the type is fully explicit in the cast — repeating it in the declaration adds no information:

```cpp
// Fine — type is visible in the cast
auto labelW   = static_cast<int16_t>(strlen(scaleBuf) * 6U);
auto barHeight = static_cast<int16_t>((colSums[col] * GRAPH_HEIGHT) / colMax);
```

**Never use `auto` when the type would be implicit** (arithmetic, function return values,
initialiser lists, etc.) and especially not for register-width or hardware-interface types
(`uint8_t`, `uint16_t`, `uint32_t`, register pointers, etc.) — integer-promotion rules can
silently widen a value to `int`, and the difference between an 8-bit and 16-bit result is
invisible at the call site but may corrupt register writes or overflow calculations:

```cpp
// Bad — promoted to int by arithmetic; actual width is invisible
auto mask    = statusReg & 0xFF;
auto ticks   = TCB0.CNT;          // register type is implicit

// Good — width is explicit
uint8_t  mask  = static_cast<uint8_t>(statusReg & 0xFF);
uint16_t ticks = TCB0.CNT;
```

- **Column-aligned declarations and assignments:** within a group of related variable
  declarations or assignments, pad with spaces so that all `=` signs land in the same column.
  Apply this within a logical block; do not force alignment across unrelated groups.

```cpp
// Variable declarations — = signs aligned
volatile bool     pulseDetected = false;
volatile uint32_t totalCount    = 0;
uint32_t          secondsElapsed = 0;

// constexpr constants — = signs aligned
constexpr uint16_t LED_ON_TIME      = 10;
constexpr uint16_t BUZZER_ON_TIME   = 3;
constexpr uint16_t BUZZER_FREQUENCY = 2500;

// Related assignments in a block — = signs aligned
ledIsOn    = true;
ledOnUntil = millis() + LED_ON_TIME;
```

- **`const` placement (west-`const`):** always write `const` before the type, never after:

```cpp
const uint32_t elapsed = millis() - start;   // correct
uint32_t const elapsed = millis() - start;   // wrong
```

- **Range-based `for` loops:** always spell out the element type explicitly — never use `auto`:

```cpp
for (const uint16_t bucket : cpmBuckets) {   // correct
for (auto bucket : cpmBuckets) {             // wrong
```

- **Inline `// NOLINT` suppressions:** when a static-analysis warning is unavoidable (e.g.
  third-party API constraints, AVR-libc header limitations), suppress it inline on the same
  line with a brief explanation after an em-dash:

```cpp
#include <string.h> // NOLINT(*-deprecated-headers) — AVR-libc does not provide <cstring>
Adafruit_SSD1306 display(128, 64, &Wire, -1); // NOLINT(*-interfaces-global-init)
```

---

## 8. Init Function Pattern (`main.cpp`)

Decompose `setup()` into named `initXxx()` helpers — one per subsystem if the init method become too long.
Each helper leaves its subsystem in a safe, known state before returning.

```cpp
void setup() {
    Serial.begin(115200);
    initPins();
    initTimers();
    initI2C();
    initUserInputs();
    ui.refresh();
}
```

No subsystem init function should depend on another having run first, unless the
dependency is explicit and documented with a comment.

