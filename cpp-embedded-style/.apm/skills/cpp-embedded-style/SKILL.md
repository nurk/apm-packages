---
name: cpp-embedded-style
description: >
  Coding style and architecture guidelines for embedded C++ projects targeting
  Arduino-framework microcontrollers (AVR / megaAVR / SAM / ESP, via PlatformIO
  or the Arduino IDE). Covers naming conventions, class design, hardware
  abstraction patterns, comment style, and ISR / peripheral access idioms.
  ALWAYS load this skill when: writing any .cpp or .h file, generating a new class,
  adding a method, reviewing code, or when the project uses Arduino, PlatformIO,
  AVR, ATmega, ESP, or any embedded C++ framework. Apply without being asked.
---

Apply these guidelines every time you generate or modify C++ code for this project.

---

## 1. File & Header Structure

- One class per `.h` / `.cpp` pair; filename matches the class name exactly (PascalCase).
- Header guards use `#ifndef PROJECT_CLASSNAME_H` / `#define ...` / `#endif //PROJECT_CLASSNAME_H`.
  Replace `PROJECT` with the project name in SCREAMING_SNAKE_CASE.
- Include ordering (each group blank-line separated):
    1. Own header (`#include <ClassName.h>`)
    2. Arduino / framework headers (`<Arduino.h>`, `<Wire.h>`, `<SPI.h>`, …)
    3. Architecture low-level headers (`<avr/io.h>`, `<avr/interrupt.h>`, …)
    4. Third-party library headers
    5. Project-local headers

---

## 2. Naming Conventions

| Kind                              | Convention            | Example                                   |
|-----------------------------------|-----------------------|-------------------------------------------|
| Classes                           | PascalCase            | `MotorDriver`, `DisplayController`        |
| Public methods                    | camelCase             | `setSpeed()`, `turnOn()`                  |
| Private methods                   | camelCase             | `computePwm()`, `getStorageOffset()`      |
| Private member variables          | camelCase             | `isRunning`, `targetSpeed`                |
| `const` members (class-scope)     | SCREAMING_SNAKE_CASE  | `MAX_SPEED`, `BAUD_RATE`                  |
| Local variables                   | camelCase             | `elapsed`, `rawAdc`, `wasOn`              |
| Function parameters               | camelCase             | `targetRpm`, `ledPin`                     |
| `#define` pin / hardware constants| SCREAMING_SNAKE_CASE  | `MOTOR_EN`, `STATUS_LED`                  |
| Enum values                       | SCREAMING_SNAKE_CASE  | `IDLE`, `RUNNING`, `ERROR`                |
| ISR vectors                       | MCU names verbatim    | `TIMER1_COMPA_vect`, `PORTA_PORT_vect`    |

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
    : dataPin_(dataPin), csPin_(csPin), spi_(spi), display_(display) { }
```

### `const` correctness

- Mark every method that does not mutate state as `const`.
- Mark every parameter that will not be modified as `const`.
- Return `const` references where appropriate to prevent unintended mutation.

### Private helpers

Prefer `private static` methods for pure computations that don't touch member state.
Use `private` non-static methods for computations that read (but don't write) member state.

### "save-state, modify, restore" pattern

When an operation must temporarily change hardware state, save and restore the on/running state:

```cpp
void Driver::reconfigure(const Config& cfg) {
    const boolean wasRunning = isRunning_;
    stop();
    // … apply new configuration to hardware …
    if (wasRunning) start();
}
```

### Single responsibility per class

Each class owns exactly one hardware subsystem or one UI concern.
`main.cpp` wires them together; the classes themselves don't know about each other
unless explicitly injected.

---

## 4. Hardware Access Idioms

### ISRs

- ISRs live in `main.cpp`, never inside class files.
- Keep ISRs as short as possible: read/toggle/clear the flag, return.
- If an ISR needs to communicate with a class, do it through a `volatile` flag or
  a dedicated method that only touches `volatile` data.

```cpp
// main.cpp
ISR(TIMER1_COMPA_vect) {
    PORTB.OUTTGL    = PIN0_bm;
    TIMER1.INTFLAGS = CAPT_bm;
}
```

### Pin initialisation (performance-critical paths)

Use `digitalWriteFast()` for output-only GPIO called in setup sequences or tight loops.
Use `digitalWrite()` / `digitalRead()` for general, readable cases.

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

### Null-pointer guard for optional peripherals

When a peripheral pointer can legitimately be absent, guard every access:

```cpp
if (peripheral_ != nullptr) peripheral_->doSomething();
```

---

## 5. Type Discipline

Use explicit-width types everywhere. Avoid bare `int` or `long` when the width matters.

| Use case                                      | Type        | Reason                                             |
|-----------------------------------------------|-------------|----------------------------------------------------|
| 8-bit register values / pin numbers           | `uint8_t`   | Matches hardware width                             |
| 16-bit register / timer values                | `uint16_t`  | Explicit width; avoids promotion surprises         |
| Counters / values that could exceed 16 bits   | `uint32_t`  | Safe on all AVR / 32-bit targets                   |
| Values needing full 64-bit range              | `uint64_t`  | Use sparingly; expensive on 8-bit MCUs             |
| Signed differences / deltas                   | `int32_t` / `int64_t` | Transient; cast back to unsigned after clamp |
| Boolean flags                                 | `boolean`   | Arduino convention                                 |
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

### File-top block

Use a `/** … **/` block at the top of `main.cpp` listing software/hardware revision,
board target, and URLs for all non-standard libraries used.

---

## 7. Formatting

- **Indentation:** 4 spaces, no tabs.
- **Brace style:** K&R — opening brace on the same line as the statement or function signature.
- **Blank lines:** one blank line between methods in `.cpp`; two blank lines between
  unrelated logical sections in `main.cpp`.
- **Multi-line argument lists:** align continuation lines to the column after the opening `(`:

```cpp
SensorReader reader(DATA_PIN,
                    CS_PIN,
                    spi,
                    display);
```

- **Trailing newline:** every file ends with exactly one newline.
- **Line length:** soft limit 100 chars; favour readability over the limit.
- **`auto`:** use only where the type is unambiguous from the right-hand side or
  an explicit cast makes it clear.

---

## 8. Init Function Pattern (`main.cpp`)

Decompose `setup()` into named `initXxx()` helpers — one per subsystem.
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

