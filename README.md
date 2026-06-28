# Embedded Thermostat System: State Machine Design and Hardware Integration on Raspberry Pi 4B

## Project Summary

This project implements a functional thermostat prototype on a Raspberry Pi 4B. The system reads ambient temperature from an AHT20 sensor over I2C, displays the current time, temperature, and thermostat state on an HD44780 16x2 LCD, and controls two PWM LEDs (red for heat, blue for cool) to indicate whether the system is actively working to reach the setpoint. Three physical buttons allow the user to cycle through thermostat states (off, heat, cool) and adjust the temperature setpoint up or down by one degree. The system also transmits a status update over UART serial every 30 seconds to a downstream temperature server.

The problem it solves is representative of a real embedded control scenario: coordinating sensor input, user input, display output, hardware indicators, and serial communication simultaneously — without any of those concerns blocking the others.

---

## What I Did Particularly Well

The state machine design is clean and well-contained. Using the `python-statemachine` library, the three thermostat states (off, heat, cool) and their transitions are defined declaratively, and all hardware side effects (LED behavior, debug output) are handled in `on_enter` and `on_exit` hooks rather than scattered through the main loop. This keeps state logic easy to follow and modify.

I also identified and resolved a concurrency bug that caused intermittent sensor read failures. The AHT20 sensor was being accessed from both the display management thread and the main thread without coordination. The symptom was silent failures and garbled reads. The fix was a `threading.Lock` (`sensorLock`) that gates all sensor access through a single method (`getFahrenheit()`, line 376), ensuring reads are never concurrent:

```python
def getFahrenheit(self):
    with sensorLock:
        t = thSensor.temperature
        return (((9/5) * t) + 32)
```

Button debounce (`bounce_time=0.2`) was added to all three buttons (lines 463, 470, 477) after observing that physical button presses were registering multiple times and causing the state machine to skip states. This is a common hardware-level issue that software has to compensate for when working with mechanical switches.

---

## Where I Could Improve

The display management thread currently uses `sleep(1)` as its timing mechanism, which drifts over time and isn't suitable for anything requiring precision. A more robust implementation would replace the `Thread` + `sleep` approach with Python's `asyncio`, using `asyncio.sleep()` and async tasks to manage the display loop, serial output timer, and sensor reads concurrently without blocking. This would also eliminate the need for the manual `sensorLock` — async's cooperative multitasking model avoids the shared-state hazard entirely since only one coroutine runs at a time.

The `DEBUG` flag is functional but blunt — a proper logging implementation using Python's `logging` module would allow severity-level filtering and cleaner output without having to remove print statements manually before deployment.

---

## Tools and Resources Added to My Support Network

- **`i2c-tools` / `i2cdetect`** — essential for confirming sensor visibility on the I2C bus before debugging in software. Learned to start here first when hardware isn't behaving.
- **`python-statemachine`** — a clean, declarative way to implement state machines that scales better than if/elif chains as complexity grows.
- **`gpiozero`** — higher-level GPIO abstraction that handles PWM, button events, and cleanup more cleanly than `RPi.GPIO` directly.
- **Threading and locking primitives** (`threading.Thread`, `threading.Lock`) — practical experience with shared hardware state in a concurrent context, which transfers directly to any multi-threaded embedded or systems work.

---

## Transferable Skills

**Concurrent resource management** is the most directly transferable skill from this project. The pattern of identifying shared state, recognizing the race condition, and protecting access with a lock is fundamental — it applies in any language or environment where multiple threads touch the same hardware or data structure. In Rust, for example, this same pattern maps to `Arc<Mutex<T>>`, and the compiler enforces it rather than leaving it to the developer to catch at runtime.

**State machine design** transfers broadly. Thermostats, network protocol handlers, game logic, UI flows — all of these are well-modeled as state machines. The declarative approach used here is more maintainable than ad hoc conditional logic and scales to more complex systems.

**Hardware-software integration and debugging methodology** — specifically, the discipline of confirming hardware visibility (`i2cdetect`) before assuming a software bug — is a habit that saves significant time in any embedded context.

---

## Maintainability, Readability, and Adaptability

The code is organized around a clear class boundary: `ManagedDisplay` owns the LCD hardware, and `TemperatureMachine` owns all thermostat logic, state transitions, and output coordination. Neither class reaches into the other's internals. LED pin assignments and the `DEBUG` flag are defined at the top of the file, making reconfiguration straightforward without hunting through implementation code.

Inline comments explain *why* decisions were made, not just what the code does — for example, the note on `serial.PARITY_NONE` and `serial.STOPBITS_ONE` explaining the serial protocol configuration, and the comment explaining why `board` is not imported a second time for I2C. The `setPoint` default (72°F) is a class-level variable on `TemperatureMachine`, meaning it's one line to change for a different application.

To adapt this system to different hardware, only the pin assignments in `ManagedDisplay.__init__()` and the GPIO numbers passed to `PWMLED` and `Button` would need to change — the logic layer is fully decoupled from the specific wiring.