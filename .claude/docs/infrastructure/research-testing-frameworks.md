# Research: Testing Methodologies and Frameworks for MeshCore

**Date:** 2026-02-24
**Status:** Research complete, ready for planning

---

## Table of Contents

1. [PlatformIO Native Testing](#1-platformio-native-testing)
2. [Unit Test Frameworks for Embedded C++](#2-unit-test-frameworks-for-embedded-c)
3. [Mocking Frameworks for Embedded](#3-mocking-frameworks-for-embedded)
4. [Testing Strategies for Embedded Projects](#4-testing-strategies-for-embedded-projects)
5. [Testing the Untestable](#5-testing-the-untestable)
6. [CI/CD for Embedded](#6-cicd-for-embedded)
7. [Real-World Examples](#7-real-world-examples)
8. [MeshCore-Specific Analysis](#8-meshcore-specific-analysis)
9. [Recommendations](#9-recommendations)

---

## 1. PlatformIO Native Testing

### Overview

PlatformIO has built-in unit testing support via the `pio test` command. Tests can
run on the local host machine (native), on connected embedded devices, or on
remote devices. The system is configured through `platformio.ini` and follows
a directory convention under `test/`.

**References:**
- [PlatformIO Unit Testing docs](https://docs.platformio.org/en/latest/advanced/unit-testing/index.html)
- [pio test CLI reference](https://docs.platformio.org/en/latest/core/userguide/cmd_test.html)
- [Test Hierarchy](https://docs.platformio.org/en/latest/advanced/unit-testing/structure/hierarchy.html)
- [PlatformIO Labs blog: Unit Testing Part 1](https://piolabs.com/blog/insights/unit-testing-part-1.html)

### How `pio test` Works

1. PlatformIO scans the `test/` directory (configurable via `test_dir`).
2. Each subfolder prefixed with `test_` is an independent test suite.
3. Each test suite compiles as its own executable (needs its own `main()` or
   `setup()/loop()` for Arduino).
4. For **native** tests, the executable runs directly on the host machine.
5. For **embedded** tests, PlatformIO uploads to a connected board, resets it,
   and captures serial output to determine pass/fail.
6. Results are parsed and reported in the terminal or CI output.

### Directory Structure

```
project/
  platformio.ini
  src/                          # firmware source
  test/
    test_utils/                 # native test suite for Utils
      test_utils.cpp
    test_packet/                # native test suite for Packet
      test_packet.cpp
    embedded/
      test_ble_integration/     # on-device test suite
        test_ble.cpp
```

Rules:
- Only folders starting with `test_` are recognized as test suites.
- Files in the root `test/` dir compile with ALL active test suites (shared helpers).
- Nested hierarchies are supported for logical grouping.
- The `test/` root and active test folder are added to the C preprocessor include path.

### Native Platform Configuration

The `native` platform compiles for the host OS using the system's GCC/Clang. No
special toolchain is installed by PlatformIO -- it uses whatever is available.

```ini
[env:native]
platform = native
test_framework = googletest    ; or unity, doctest
build_flags = -std=c++17
test_filter = test_*           ; optional: filter which suites run
```

For projects that mix native and embedded tests:

```ini
[env:native]
platform = native
test_framework = googletest
test_filter = native/*         ; only run tests under test/native/

[env:t1000e_companion_radio_ble]
; ... existing embedded environment ...
test_filter = embedded/*       ; only run tests under test/embedded/
```

### `test_filter` and `test_ignore`

- `test_filter` -- glob pattern matching test suite paths relative to `test_dir`.
  Only matching suites are executed.
- `test_ignore` -- glob pattern for suites to skip.
- Both work in `platformio.ini` or via CLI: `pio test --filter "test_utils*"`

### Supported Test Frameworks (`test_framework`)

| Framework   | Test Types | Platforms                    | Mocking | Notes                                |
|-------------|------------|------------------------------|---------|--------------------------------------|
| Unity       | Any        | Any (including 8-bit AVR)    | No      | Default. Minimal footprint.          |
| GoogleTest  | Any        | Native, ESP8266, ESP32       | Yes     | Full C++ features. Built-in GMock.   |
| Doctest     | Native     | Native only                  | No      | Fastest compile times.               |
| Custom      | Any        | Any                          | Varies  | Implement your own runner.           |

**Key insight:** PlatformIO explicitly supports mixing frameworks in the same
project. Use GoogleTest+GMock for native/host tests (full C++ power) and Unity
for on-device tests (minimal resources).

**References:**
- [Testing Frameworks overview](https://docs.platformio.org/en/latest/advanced/unit-testing/frameworks/index.html)
- [GoogleTest in PlatformIO](https://docs.platformio.org/en/latest/advanced/unit-testing/frameworks/googletest.html)
- [Unity in PlatformIO](https://docs.platformio.org/en/latest/advanced/unit-testing/frameworks/unity.html)
- [Doctest in PlatformIO](https://docs.platformio.org/en/latest/advanced/unit-testing/frameworks/doctest.html)

---

## 2. Unit Test Frameworks for Embedded C++

### Unity (ThrowTheSwitch)

**Language:** Pure C
**Size:** 2 headers + 1 source file
**License:** MIT

Unity is the de facto standard for embedded C testing. It is PlatformIO's default
`test_framework`.

**Pros:**
- Minimal footprint -- runs on 8-bit AVR with kilobytes of RAM.
- Works for both native host testing and on-device testing.
- First-class PlatformIO integration (zero configuration needed).
- Paired with CMock for auto-generated mocks and Ceedling for build automation.
- Well-established in the embedded community.
- Cross-platform and cross-compiler compatible.

**Cons:**
- Pure C -- no native C++ features (no test fixtures as classes, no templates).
- Assertion macros are C-style (`TEST_ASSERT_EQUAL`, `TEST_ASSERT_TRUE`).
- No built-in mocking (requires CMock or manual fakes).
- Test organization requires manual `RUN_TEST()` calls in `main()`.
- Less expressive than C++ frameworks for complex assertions.

**Suitability for MeshCore:** Good for on-device tests. Adequate for simple
native tests. The C-only nature is a limitation for testing C++ classes.

**References:**
- [ThrowTheSwitch.org](https://www.throwtheswitch.org/)
- [Unity GitHub](https://github.com/ThrowTheSwitch/Unity)
- [ThrowTheSwitch comparison](https://www.throwtheswitch.org/comparison-of-unit-test-frameworks)

### Google Test (gtest)

**Language:** C++
**License:** BSD-3

Google Test is the most widely used C++ testing framework. PlatformIO has
first-class support for it via `test_framework = googletest`.

**Pros:**
- Rich assertion library (`EXPECT_EQ`, `ASSERT_THAT`, matchers).
- Built-in mocking via GMock (interface mocking, argument matchers, call verification).
- Test fixtures with setup/teardown as C++ classes.
- Death tests, parameterized tests, typed tests.
- Excellent IDE integration and output formatting.
- Supported on PlatformIO native and ESP32 platforms.
- Huge community and extensive documentation.

**Cons:**
- Large binary size -- not suitable for constrained embedded targets.
- Requires C++14 or later (v1.14+).
- PlatformIO embedded support limited to ESP8266 and ESP32 (NOT nRF52).
- Heavier compile times than Unity or doctest.
- GMock has a learning curve.

**Suitability for MeshCore:** Excellent for native/host tests. Cannot run on
nRF52 targets. Best choice for testing portable logic (mesh routing, packet
handling, crypto, CLI parsing) on the development machine.

**References:**
- [GoogleTest GitHub](https://github.com/google/googletest)
- [GoogleTest in PlatformIO](https://docs.platformio.org/en/latest/advanced/unit-testing/frameworks/googletest.html)
- [PlatformIO gTest/gMock example](https://github.com/Schallbert/PlatformIO_gTestgMock)
- [PlatformIO official examples](https://github.com/platformio/platformio-examples/blob/develop/unit-testing/googletest/platformio.ini)

### Catch2

**Language:** C++ (C++14 for v3, C++11 for v2)
**License:** BSL-1.0

Catch2 is a header-only (v2) or library (v3) testing framework known for its
expressive BDD-style syntax and minimal boilerplate.

**Pros:**
- Natural assertion syntax: `REQUIRE(x == 42)` with automatic expression decomposition.
- BDD-style `SCENARIO`/`GIVEN`/`WHEN`/`THEN` macros.
- Header-only in v2 (single include).
- Auto-registration of test cases (no `RUN_TEST()` boilerplate).
- Micro-benchmarking support.

**Cons:**
- NOT natively supported by PlatformIO's test runner (no `test_framework = catch2`).
  Can be used via custom framework integration but loses `pio test` integration.
- Too large for constrained embedded targets (large binary, heavy templates).
- v3 requires C++14 minimum.
- No built-in mocking.
- PlatformIO issue #2185 requested Catch2 support but it was deprioritized in
  favor of doctest (which is lighter and API-compatible where it matters).

**Suitability for MeshCore:** Not recommended. Lacks PlatformIO integration.
GoogleTest provides equivalent or superior native testing with better PlatformIO
support.

**References:**
- [Catch2 GitHub](https://github.com/catchorg/Catch2)
- [PlatformIO issue #2185](https://github.com/platformio/platformio-core/issues/2185)
- [Catch2 embedded device issue #328](https://github.com/catchorg/Catch2/issues/328)

### CppUTest

**Language:** C++ (but designed for testing C code)
**License:** BSD-3

CppUTest is an xUnit-style framework specifically designed for embedded
development. Featured in James Grenning's "Test Driven Development for
Embedded C."

**Pros:**
- Designed for embedded: small footprint, reduced C++ feature set.
- Built-in memory leak detection.
- CppUMock companion library for mocking.
- Can test both C and C++ code.
- Works on host and some embedded targets.

**Cons:**
- Supported by PlatformIO but less commonly used than Unity or GoogleTest.
- CppUMock is verbose compared to GMock.
- Smaller community than GoogleTest.
- Less modern C++ feel (xUnit-style rather than expressive assertions).
- Requires Ruby for some tooling (Ceedling integration).

**Suitability for MeshCore:** Viable alternative to GoogleTest for native tests.
The embedded-first design philosophy is appealing. However, GoogleTest's PlatformIO
integration and GMock's power make GoogleTest the stronger choice for native
testing.

**References:**
- [CppUTest website](https://cpputest.github.io/)
- [Memfault Interrupt: Unit Testing Basics](https://interrupt.memfault.com/blog/unit-testing-basics)
- [CppUTest nRF52 example](https://github.com/liamw9534/Jumper-nRF52-CppUTest)

### doctest

**Language:** C++11
**License:** MIT

doctest is a fast, lightweight C++ testing framework. PlatformIO has native
support for it via `test_framework = doctest`.

**Pros:**
- Fastest compile times of any C++ testing framework.
- Header-only (single header).
- PlatformIO native support (`test_framework = doctest`).
- Minimal boilerplate, auto-registration of tests.
- API similar to Catch2 but lighter.

**Cons:**
- Native-only in PlatformIO (cannot run on embedded targets).
- No built-in mocking.
- Smaller community than GoogleTest.
- Less powerful assertion matchers than GoogleTest.

**Suitability for MeshCore:** A lighter alternative to GoogleTest for native
tests if compile speed is critical. But GoogleTest's mocking support (GMock) is
a significant advantage that doctest lacks.

**References:**
- [doctest GitHub](https://github.com/doctest/doctest)
- [doctest in PlatformIO](https://docs.platformio.org/en/latest/advanced/unit-testing/frameworks/doctest.html)

### Framework Comparison Summary

| Criterion                  | Unity       | GoogleTest  | Catch2      | CppUTest    | doctest     |
|---------------------------|-------------|-------------|-------------|-------------|-------------|
| PlatformIO support        | First-class | First-class | None        | Supported   | First-class |
| On-device testing         | Yes (any)   | ESP32 only  | No          | Limited     | No          |
| Native host testing       | Yes         | Yes         | Yes*        | Yes         | Yes         |
| Mocking                   | Via CMock   | GMock built-in | None     | CppUMock    | None        |
| C++ friendliness          | Low (C)     | High        | High        | Medium      | High        |
| Binary size               | Tiny        | Large       | Large       | Medium      | Small       |
| Compile speed             | Fast        | Moderate    | Moderate    | Moderate    | Fastest     |
| Maturity/community        | High        | Highest     | High        | Medium      | Medium      |
| Learning curve            | Low         | Medium      | Low         | Medium      | Low         |

*Catch2 requires custom PlatformIO integration.

---

## 3. Mocking Frameworks for Embedded

### CMock (ThrowTheSwitch)

CMock is a companion to Unity. It auto-generates mock implementations by parsing
C header files using Ruby scripts.

**How it works:**
1. Point CMock at a C header file (e.g., `Radio.h`).
2. Ruby scripts parse function declarations.
3. Generated mock files contain `_Expect`, `_Stub`, `_Callback`, `_Ignore` variants
   for each function.
4. Link the mock into your test instead of the real implementation.

**Pros:**
- Auto-generated mocks from headers (no manual writing).
- Rich expectation/verification API (expect with parameters, return values, sequences).
- Tight Unity integration.
- Well-documented for embedded use.

**Cons:**
- Requires Ruby runtime (non-trivial dependency for CI).
- Only works with C function interfaces (not C++ classes/virtual methods).
- Generated code can be large.
- MeshCore uses C++ virtual interfaces, which CMock cannot mock directly.

**Suitability for MeshCore:** Poor fit. MeshCore's abstractions (Radio, PacketManager,
RTCClock, etc.) are C++ abstract classes with virtual methods, not C function APIs.
CMock is designed for C headers.

**References:**
- [CMock GitHub](https://github.com/ThrowTheSwitch/CMock)
- [ThrowTheSwitch CMock docs](https://www.throwtheswitch.org/cmock)

### FFF (Fake Function Framework)

FFF is a header-only micro-framework for creating fake C functions. It is the
simplest mocking approach for embedded C.

**How it works:**
```c
#include "fff.h"
DEFINE_FFF_GLOBALS;

// Create a fake for: int hal_gpio_read(int pin);
FAKE_VALUE_FUNC(int, hal_gpio_read, int);

void test_example(void) {
    hal_gpio_read_fake.return_val = 1;
    // ... call code under test ...
    TEST_ASSERT_EQUAL(1, hal_gpio_read_fake.call_count);
    TEST_ASSERT_EQUAL(5, hal_gpio_read_fake.arg0_val);
}
```

**Pros:**
- Header-only (single `fff.h` file).
- Zero dependencies (no Ruby, no build tools).
- Simple macro-based API.
- Tracks call count, arguments, return values, call sequences.
- Custom return value sequences.
- Works with any test framework (Unity, GoogleTest, etc.).

**Cons:**
- Only fakes C functions (not C++ virtual methods).
- Limited to predefined arity (up to ~20 arguments by default).
- No automatic mock generation from headers.
- Less powerful verification than CMock or GMock.

**Suitability for MeshCore:** Useful for faking low-level C functions (Arduino
APIs like `millis()`, `Serial.print()`, `analogRead()`). Not directly useful
for MeshCore's C++ virtual interfaces, but valuable for the Arduino HAL layer.

**References:**
- [FFF GitHub](https://github.com/meekrosoft/fff)
- [CMock vs FFF comparison](http://www.electronvector.com/blog/cmock-vs-fff-a-comparison-of-c-mocking-frameworks)
- [Nordic DevZone: Mocking nrf_gpio.h with FFF](https://devzone.nordicsemi.com/f/nordic-q-a/54881/mocking-nrf_gpio-h-with-ceedling-unity-fff-fake-function-framework)

### Google Mock (GMock)

GMock is the mocking component of GoogleTest. It mocks C++ interfaces using
class inheritance and virtual methods.

**How it works:**
```cpp
#include <gmock/gmock.h>

// Mock the abstract Radio interface
class MockRadio : public mesh::Radio {
public:
    MOCK_METHOD(int, recvRaw, (uint8_t* bytes, int sz), (override));
    MOCK_METHOD(uint32_t, getEstAirtimeFor, (int len_bytes), (override));
    MOCK_METHOD(bool, startSendRaw, (const uint8_t* bytes, int len), (override));
    MOCK_METHOD(bool, isSendComplete, (), (override));
    MOCK_METHOD(void, onSendFinished, (), (override));
    MOCK_METHOD(bool, isInRecvMode, (), (const, override));
    MOCK_METHOD(float, packetScore, (float snr, int packet_len), (override));
};

TEST(DispatcherTest, SendsPacketWhenRadioReady) {
    MockRadio radio;
    EXPECT_CALL(radio, isInRecvMode()).WillRepeatedly(Return(true));
    EXPECT_CALL(radio, startSendRaw(_, _)).WillOnce(Return(true));
    // ... test code ...
}
```

**Pros:**
- Perfect fit for MeshCore's architecture (C++ abstract classes with virtual methods).
- Rich matchers (`_, Eq, Ne, Lt, Gt, HasSubstr, ElementsAre`, etc.).
- Call verification (times, order, arguments).
- Integrated with GoogleTest assertions.
- Well-documented with large community.
- Actions (`Return`, `SetArgPointee`, `Invoke`, `DoAll`).

**Cons:**
- Requires GoogleTest (not standalone).
- Large binary -- native/host only.
- Cannot mock non-virtual functions or free functions (use FFF for those).
- Learning curve for matchers and expectations.

**Suitability for MeshCore:** Excellent. MeshCore's architecture already uses
abstract interfaces for hardware dependencies:
- `Radio` -- abstract class with virtual `recvRaw()`, `startSendRaw()`, etc.
- `PacketManager` -- abstract class with virtual `allocNew()`, `free()`, etc.
- `MillisecondClock` -- abstract class with virtual `getMillis()`.
- `RTCClock` -- abstract class with virtual `getCurrentTime()`, `setCurrentTime()`.
- `RNG` -- abstract class with virtual `random()`.
- `MeshTables` -- abstract class with virtual `hasSeen()`.
- `MainBoard` -- abstract class with virtual hardware methods.

These interfaces are tailor-made for GMock. This is the recommended mocking
approach for MeshCore native tests.

**References:**
- [GMock for Dummies](https://google.github.io/googletest/gmock_for_dummies.html)
- [GMock Cookbook](https://google.github.io/googletest/gmock_cook_book.html)
- [Schallbert PlatformIO gTest/gMock example](https://github.com/Schallbert/PlatformIO_gTestgMock)

### ArduinoFake

ArduinoFake is a mock library for Arduino APIs, built on FakeIt. It provides
fake implementations of Arduino functions (`digitalWrite`, `analogRead`,
`millis`, `Serial`, etc.) for native testing.

**How it works:**
```cpp
#include <ArduinoFake.h>

When(Method(ArduinoFake(), millis)).Return(1000);
When(Method(ArduinoFake(Serial), println)).AlwaysReturn();

// ... call code under test ...

Verify(Method(ArduinoFake(), digitalWrite).Using(LED_PIN, HIGH)).Once();
```

**Pros:**
- Mocks all standard Arduino APIs.
- Header-only, works with PlatformIO native platform.
- Makes Arduino-dependent code testable on the host.

**Cons:**
- Does NOT cover BLE-specific APIs (Bluefruit, NimBLE).
- Does NOT cover RadioLib.
- Can be fragile -- must stub ALL Arduino methods called, or get segfaults.
- Requires `lib_compat_mode = off` in platformio.ini.
- Adds another dependency.
- FakeIt-based API is different from GMock.

**Suitability for MeshCore:** Limited usefulness. MeshCore's hardware
dependencies are already behind abstract C++ interfaces. For the few places
where Arduino APIs are called directly (Serial, GPIO, analogRead), FFF or
simple manual stubs are simpler than pulling in ArduinoFake.

**References:**
- [ArduinoFake GitHub](https://github.com/FabioBatSilva/ArduinoFake)
- [PlatformIO ArduinoFake example](https://github.com/maxgerhardt/pio-unit-test-mock-ioabstraction)

### Mocking Strategy Recommendation for MeshCore

Given MeshCore's architecture:

1. **GMock** for all abstract C++ interfaces (Radio, PacketManager, Clock, RNG,
   Board, MeshTables). This covers 90% of mocking needs.
2. **FFF** for any remaining C-linkage functions (Arduino `millis()`, `random()`,
   `Serial.print()`) if needed.
3. **Manual fakes** for simple cases (e.g., a `FakeClock` that returns
   configurable values).
4. Skip CMock and ArduinoFake -- they don't fit MeshCore's C++ interface pattern.

---

## 4. Testing Strategies for Embedded Projects

### 4.1 Host-Based (Native) Testing

**What:** Compile and run tests on the development machine (Linux/macOS/Windows)
using `platform = native`.

**Why:** Fast feedback (milliseconds vs seconds for flash+serial), no hardware
needed, CI-friendly, enables rich tooling (coverage, sanitizers, debuggers).

**What to test:**
- Pure logic: packet encoding/decoding, encryption/decryption, hash calculations.
- Data structures: PacketQueue priority ordering, static pool allocation.
- Protocol handling: advert data building/parsing, CLI command parsing.
- Routing decisions: flood/direct routing logic, path manipulation.
- Cryptographic operations: SHA256, AES128, Ed25519 sign/verify, ECDH.
- String utilities: hex conversion, text parsing, formatting.
- State machines: mesh protocol state transitions.

**What NOT to test natively:**
- BLE stack behavior (Bluefruit, NimBLE).
- Radio driver behavior (RadioLib SX1262 interactions).
- Sensor hardware (I2C, SPI).
- Power management (sleep, wake, battery ADC).
- Board-specific GPIO.

**Build approach:**
- Compile only the source file under test plus its dependencies.
- Link against mocks/fakes for hardware interfaces.
- Use `build_src_filter` in the native environment to exclude hardware code.

**References:**
- [Memfault: Unit Testing Basics](https://interrupt.memfault.com/blog/unit-testing-basics)
- [Memfault: Unit Testing with Mocks](https://interrupt.memfault.com/blog/unit-test-mocking)

### 4.2 Hardware-in-the-Loop (HIL) Testing

**What:** Run tests on actual hardware, connected to a CI runner or developer's
machine.

**PlatformIO approach:**
- `pio test -e t1000e_companion_radio_ble` uploads test firmware to a connected
  board and reads serial output.
- PlatformIO Remote can forward test execution to remote machines with connected
  hardware.

**Practicality for MeshCore:**
- Useful for integration tests (BLE pairing, radio TX/RX, sensor reading).
- Requires physical boards connected to CI runners (expensive, fragile).
- Not practical for the initial testing setup -- focus on native tests first.
- Consider for later phases when critical integration paths need validation.

**References:**
- [PlatformIO Labs: Remote Testing](https://piolabs.com/blog/insights/unit-testing-part-3.html)

### 4.3 Emulator/Simulator Testing

#### Renode

Renode is an open-source embedded platform emulator by Antmicro. It is the
most relevant emulator for MeshCore.

**nRF52840 support:**
- Renode has explicit nRF52840 emulation support.
- BLE radio model with multi-node simulation.
- Can run production firmware binaries.
- Supports Wireshark integration for BLE traffic inspection.
- Robot Framework integration for automated test scripting.
- Multi-node mesh simulation with probabilistic packet loss.

**Limitations:**
- Renode's nRF52840 BLE support targets the Zephyr BLE stack, not the Adafruit
  Bluefruit/SoftDevice stack that MeshCore uses.
- SoftDevice S140 is a closed-source binary blob -- Renode cannot emulate it
  directly.
- RadioLib (LoRa radio abstraction) would need custom peripheral models.
- Significant setup effort for diminishing returns at this stage.

**Practicality for MeshCore:** Low priority. The SoftDevice dependency makes
BLE emulation impractical. LoRa radio emulation would require custom Renode
models. Focus on native host testing instead.

#### QEMU

QEMU supports limited Cortex-M emulation (only 2 specific board targets). Not
suitable for nRF52840 or ESP32. Not recommended.

#### Meshtastic Simulator Approach

Meshtastic (a comparable mesh networking project) uses a different strategy:
the Linux native build simulates the LoRa chip by routing packets via TCP
sockets between process instances. This allows multi-node mesh testing without
hardware.

This is relevant as a future possibility for MeshCore integration testing.

**References:**
- [Renode.io](https://renode.io/)
- [Nordic DevZone: BLE on nRF52840 in Renode](https://devzone.nordicsemi.com/nordic/nordic-blog/b/blog/posts/developing-and-testing-ble-products-on-nrf52840-in-renode-and-zephyr)
- [Renode BLE simulation tutorial](https://renode.readthedocs.io/en/latest/tutorials/ble-simulation.html)
- [Renode nRF52840 BLE multi-node script](https://github.com/renode/renode/blob/master/scripts/multi-node/nrf52840-ble-zephyr.resc)
- [Meshtastic interactive simulator](https://meshtastic.org/docs/software/meshtasticator/interactive-sim/)

### 4.4 Static Analysis

PlatformIO integrates two static analysis tools via `pio check`:

#### cppcheck (default)
- Detects undefined behavior, dangerous coding constructs, memory leaks.
- Very few false positives by design.
- Built into PlatformIO (no separate install for `pio check`).
- Inline suppressions: `// cppcheck-suppress warningId`

#### clang-tidy
- Clang-based linter with broader checks (style, performance, modernize, bugprone).
- Can auto-fix some issues.
- Requires clang toolchain.
- Configurable via `.clang-tidy` file.

#### PVS-Studio
- Commercial static analyzer with deep analysis.
- Free for open-source projects.
- PlatformIO integration available.

**Configuration:**
```ini
[env:native]
check_tool = cppcheck, clang-tidy
check_flags =
    cppcheck: --enable=all --suppress=missingInclude
    clang-tidy: --checks=-*,bugprone-*,performance-*,readability-*
```

Run: `pio check -e native` or `pio check --fail-on-defect=high`

**References:**
- [PlatformIO Static Code Analysis](https://docs.platformio.org/en/latest/advanced/static-code-analysis/index.html)
- [cppcheck](https://cppcheck.sourceforge.io/)
- [PlatformIO clang-tidy](https://docs.platformio.org/en/latest/advanced/static-code-analysis/tools/clang-tidy.html)

### 4.5 Fuzz Testing for Protocol Parsers

MeshCore has several protocol parsers that are excellent fuzz testing candidates:
- `Packet::readFrom()` -- deserializes raw bytes into a Packet struct.
- `AdvertDataParser` -- parses advertisement data blobs.
- `Utils::parseTextParts()` -- splits text by separators.
- `Utils::fromHex()` -- hex string to bytes conversion.
- `Utils::MACThenDecrypt()` -- MAC verification + decryption.
- CLI command parsing in `CommonCLI::handleCommand()`.

#### libFuzzer Approach

libFuzzer is a coverage-guided fuzzer integrated into LLVM/Clang. It works by:

1. Writing a fuzz target function:
```cpp
extern "C" int LLVMFuzzerTestOneInput(const uint8_t *data, size_t size) {
    mesh::Packet pkt;
    pkt.readFrom(data, size);  // exercise the parser
    return 0;
}
```

2. Compiling with `-fsanitize=fuzzer,address,undefined`.
3. Running with a seed corpus directory.

**Pros:**
- Finds edge cases that unit tests miss (buffer overflows, integer overflows,
  assertion failures, infinite loops).
- Works well with AddressSanitizer and UBSan for catching memory errors.
- Can run in CI (time-limited).
- Excellent for binary protocol parsers.

**Cons:**
- Requires Clang (not GCC).
- Needs a separate build configuration.
- Cannot directly test hardware-dependent code.

**Practicality for MeshCore:** High value, moderate effort. The packet parser
and advert data parser are perfect fuzz targets. Can be integrated as a
separate PlatformIO native environment or standalone Makefile.

**References:**
- [libFuzzer documentation](https://llvm.org/docs/LibFuzzer.html)
- [Google fuzzing tutorial](https://github.com/google/fuzzing/blob/master/tutorial/libFuzzerTutorial.md)
- [Testing Handbook: libFuzzer](https://appsec.guide/docs/fuzzing/c-cpp/libfuzzer/)

---

## 5. Testing the Untestable

### 5.1 BLE Communication Testing Without Hardware

**Challenge:** MeshCore has two `SerialBLEInterface` implementations (nRF52 and
ESP32) that depend on platform-specific BLE stacks (Bluefruit and NimBLE).

**Approaches:**

1. **Interface-level mocking (recommended):**
   MeshCore already has `BaseSerialInterface` as an abstraction. Test the
   higher-level code (companion radio protocol, CLI dispatch) against a mock
   `BaseSerialInterface` using GMock. The BLE-specific code below the interface
   is thin and best verified by HIL.

2. **Protocol-level testing:**
   Test the BLE serial protocol (command/response framing, fragmentation) in
   native tests by mocking the byte stream. The `SerialBLEInterface` reads/writes
   bytes -- test the protocol logic, not the BLE transport.

3. **Loopback testing:**
   Create a `LoopbackSerialInterface` that connects TX to RX for testing the
   protocol handling without BLE.

### 5.2 Radio/Mesh Protocol Testing

**Challenge:** Mesh routing, packet forwarding, and multi-hop communication
depend on radio TX/RX.

**Approaches:**

1. **Mock Radio class (recommended):**
   MeshCore's `Radio` is an abstract class. Create a `MockRadio` with GMock.
   Feed test packets via `recvRaw()`, verify outbound packets via
   `startSendRaw()`.

   ```cpp
   class MockRadio : public mesh::Radio {
   public:
       MOCK_METHOD(int, recvRaw, (uint8_t*, int), (override));
       MOCK_METHOD(bool, startSendRaw, (const uint8_t*, int), (override));
       MOCK_METHOD(bool, isSendComplete, (), (override));
       MOCK_METHOD(void, onSendFinished, (), (override));
       MOCK_METHOD(bool, isInRecvMode, (), (const, override));
       MOCK_METHOD(uint32_t, getEstAirtimeFor, (int), (override));
       MOCK_METHOD(float, packetScore, (float, int), (override));
   };
   ```

2. **Multi-node simulation:**
   Create a test harness with multiple `Mesh` instances connected via
   `MockRadio` objects. When one mesh node "sends" a packet, the test delivers
   it to other mesh nodes' `recvRaw()`. This tests routing logic, duplicate
   detection, path building, etc. -- all without hardware.

3. **Packet capture/replay:**
   Record real radio packets (hex dumps) from hardware, then replay them in
   native tests to verify routing decisions.

### 5.3 Sensor Driver Testing

**Challenge:** Sensor code interacts with I2C/SPI hardware.

**Approaches:**

1. **HAL abstraction testing:**
   MeshCore's sensor code goes through `SensorManager`. Test the data
   processing logic (unit conversion, formatting, threshold detection)
   separately from the hardware read.

2. **Recorded data testing:**
   Capture real sensor data sequences and replay them in native tests to verify
   processing logic.

3. **Skip low-level testing:**
   Sensor drivers are typically thin wrappers around library calls. The
   risk/reward of mocking I2C is low. Verify by compilation and manual testing.

### 5.4 Power Management Testing

**Challenge:** Sleep/wake cycles, battery monitoring, low-power modes.

**Approaches:**

1. **State machine testing:**
   Test the power management decision logic (when to sleep, when to wake)
   as a state machine with mock inputs.

2. **Mock Board:**
   `MainBoard` is abstract. Mock `getBattMilliVolts()`, `isExternalPowered()`,
   etc. to test power-dependent logic.

---

## 6. CI/CD for Embedded

### GitHub Actions Workflow

A practical CI workflow for MeshCore would include:

```yaml
name: MeshCore CI

on:
  push:
    branches: [dev, main]
  pull_request:
    branches: [dev]

jobs:
  native-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/cache@v4
        with:
          path: |
            ~/.cache/pip
            ~/.platformio/.cache
          key: ${{ runner.os }}-pio-${{ hashFiles('platformio.ini') }}
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      - name: Install PlatformIO
        run: pip install --upgrade platformio
      - name: Run native tests
        run: pio test -e native --verbose

  static-analysis:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      - name: Install PlatformIO
        run: pip install --upgrade platformio
      - name: Run cppcheck
        run: pio check -e native --fail-on-defect=high

  build-check:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        environment:
          - t1000e_companion_radio_ble
          - rak4631_companion_radio_ble
          - heltec_v3_companion_radio_ble
          # add key targets
    steps:
      - uses: actions/checkout@v4
      - uses: actions/cache@v4
        with:
          path: |
            ~/.cache/pip
            ~/.platformio/.cache
          key: ${{ runner.os }}-pio-${{ hashFiles('platformio.ini') }}
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      - name: Install PlatformIO
        run: pip install --upgrade platformio
      - name: Build firmware
        run: pio run -e ${{ matrix.environment }}
```

### Code Coverage

For native tests with coverage reporting:

```ini
[env:native]
platform = native
test_framework = googletest
build_flags =
    -std=c++17
    --coverage
    -lgcov
    -fprofile-abs-path
```

Post-test processing:
```bash
lcov --capture --directory .pio/build/native/ --output-file coverage.info
lcov --remove coverage.info '/usr/*' '*/test/*' '*.pio/*' --output-file coverage.info
genhtml coverage.info --output-directory coverage_report
```

Upload to Codecov or similar in CI:
```yaml
      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          files: coverage.info
```

**References:**
- [PlatformIO GitHub Actions](https://docs.platformio.org/en/stable/integration/ci/github-actions.html)
- [PlatformIO CI/CD with Coverage](https://piolabs.com/blog/insights/cicd-testing-coverage-versioning.html)
- [PlatformIO native coverage blog](https://mber.dev/software/platformio-unit-test-coverage)
- [GitHub Actions PlatformIO coverage](https://blog.leon0399.ru/platformio-coverage-github-actions)

---

## 7. Real-World Examples

### Meshtastic Firmware

Meshtastic is the closest comparable project to MeshCore: a LoRa mesh networking
firmware targeting ESP32 and nRF52, built with PlatformIO.

**Testing approach:**
- Issue #1045 proposed breaking hardware-independent code into `lib/` subdirectories
  for independent testing using PlatformIO's native environment.
- Uses Unity testing framework for PlatformIO integration.
- Has a `Build Native` GitHub Actions workflow for compilation verification.
- Maintains `meshTestic` -- a separate Python-based end-to-end test suite.
- Uses a Linux native build with TCP socket simulation for multi-node testing
  (LoRa chip simulated via TCP forwarding).

**Key takeaway:** Meshtastic follows the same separation strategy -- isolate
hardware-independent logic and test it natively. They haven't implemented
extensive unit tests yet (issue #1045 remains open), making MeshCore an
opportunity to lead in this area.

**References:**
- [Meshtastic firmware GitHub](https://github.com/meshtastic/firmware)
- [Meshtastic unit test issue #1045](https://github.com/meshtastic/firmware/issues/1045)
- [meshTestic e2e test suite](https://github.com/meshtastic/meshTestic)

### PlatformIO gTest/gMock Example (Schallbert)

A reference project demonstrating GoogleTest + GMock with PlatformIO:
- Hardware Abstraction Layer (HAL) enables mock classes.
- Tests validate both inputs (using matchers) and outputs (behavior verification).
- Clean separation between testable logic and hardware access.

**References:**
- [GitHub: Schallbert/PlatformIO_gTestgMock](https://github.com/Schallbert/PlatformIO_gTestgMock)

### PlatformIO Official Examples

PlatformIO maintains official unit testing examples:
- Unity example with embedded testing.
- GoogleTest example with native testing.
- Demonstrates directory structure and platformio.ini configuration.

**References:**
- [platformio-examples: unit-testing](https://github.com/platformio/platformio-examples/blob/develop/unit-testing/googletest/platformio.ini)

### GoogleTest for Embedded (TSprech)

A guide/resource specifically for using GoogleTest on PlatformIO and other
embedded platforms.

**References:**
- [GitHub: TSprech/Googletest-For-Embedded](https://github.com/TSprech/Googletest-For-Embedded)

### maxgerhardt ArduinoFake + PlatformIO Example

Demonstrates native desktop testing with ArduinoFake for mocking Arduino APIs
and IoAbstraction for hardware abstraction.

**References:**
- [GitHub: maxgerhardt/pio-unit-test-mock-ioabstraction](https://github.com/maxgerhardt/pio-unit-test-mock-ioabstraction)

---

## 8. MeshCore-Specific Analysis

### Architecture Advantages for Testing

MeshCore's codebase is well-structured for testing due to its use of abstract
interfaces for hardware dependencies:

| Abstract Interface  | What It Abstracts          | Mockable with GMock? |
|--------------------|----------------------------|----------------------|
| `Radio`            | Packet radio TX/RX         | Yes                  |
| `PacketManager`    | Packet allocation & queues | Yes                  |
| `MillisecondClock` | System tick clock          | Yes                  |
| `RTCClock`         | Real-time clock            | Yes                  |
| `RNG`              | Random number generation   | Yes                  |
| `MeshTables`       | Duplicate packet detection | Yes                  |
| `MainBoard`        | Board hardware (battery, temp, GPIO) | Yes      |
| `CommonCLICallbacks` | CLI action handlers      | Yes                  |
| `SensorManager`    | Sensor data access         | Yes                  |

This is a strong foundation. The `Mesh` class and `Dispatcher` accept these
interfaces via constructor injection -- a textbook dependency injection pattern
that makes testing straightforward.

### High-Value Test Targets (Portable Code)

These modules have no hardware dependencies and can be tested natively:

1. **`Packet` (src/Packet.cpp)**
   - `readFrom()` / `writeTo()` -- serialization/deserialization
   - `calculatePacketHash()` -- hash computation
   - `getRouteType()`, `getPayloadType()` -- header field extraction
   - `getRawLength()` -- length calculation

2. **`Utils` (src/Utils.cpp)**
   - `sha256()` -- hash computation
   - `encrypt()` / `decrypt()` -- AES128
   - `encryptThenMAC()` / `MACThenDecrypt()` -- authenticated encryption
   - `toHex()` / `fromHex()` -- hex conversion
   - `parseTextParts()` -- string parsing
   - `isHexChar()` -- character classification

3. **`Identity` / `LocalIdentity` (src/Identity.cpp)**
   - Construction from hex strings
   - `isHashMatch()` -- hash comparison
   - `verify()` -- Ed25519 signature verification
   - `sign()` -- Ed25519 signing
   - `calcSharedSecret()` -- ECDH
   - `validatePrivateKey()`
   - Serialization (`readFrom()` / `writeTo()`)

4. **`AdvertDataBuilder` / `AdvertDataParser` (src/helpers/AdvertDataHelpers.cpp)**
   - Round-trip encode/decode testing
   - Edge cases (max name length, precision of lat/lon encoding)
   - Invalid input handling

5. **`StrHelper` (src/helpers/TxtDataHelpers.cpp)**
   - `strncpy()` / `strzcpy()` -- safe string copy
   - `ftoa()` / `ftoa3()` -- float to string
   - `isBlank()` -- string classification
   - `fromHex()` -- hex to uint32

6. **`StaticPoolPacketManager` / `PacketQueue` (src/helpers/StaticPoolPacketManager.cpp)**
   - Pool allocation and exhaustion
   - Priority queue ordering
   - Scheduled delivery (time-based)
   - Edge cases (empty pool, full queue)

7. **`CommonCLI::handleCommand()` (src/helpers/CommonCLI.cpp)**
   - CLI command parsing and dispatch
   - Parameter validation
   - Response formatting
   - Requires mocking `CommonCLICallbacks`, `RTCClock`, `MainBoard`, etc.

8. **`Mesh` routing logic (src/Mesh.cpp)**
   - `routeRecvPacket()` -- routing decisions
   - `allowPacketForward()` -- forwarding policy
   - `getRetransmitDelay()` -- timing calculations
   - Requires mocking `Radio`, `PacketManager`, `Clock`, `RNG`, `MeshTables`

9. **`ClientACL` (src/helpers/ClientACL.cpp)**
   - Access control list management

10. **`IdentityStore` (src/helpers/IdentityStore.cpp)**
    - Identity persistence logic (may need filesystem fake)

### Code That Should NOT Be Unit Tested (Initially)

- `SerialBLEInterface` (nrf52 and esp32) -- deep platform coupling
- `ESP32Board` / `NRF52Board` -- hardware initialization
- `TBeamBoard` -- board-specific I2C/GPIO
- Sensor implementations in `variants/` -- hardware I2C
- RadioLib wrapper code in `src/helpers/radiolib/`

### Dependencies That Need Stubbing for Native Build

For native compilation, these Arduino/platform headers need stubs or fakes:

- `<Arduino.h>` -- `millis()`, `delay()`, `Serial`, pin functions
- `<Stream.h>` -- base class for serial communication
- `<SPI.h>`, `<Wire.h>` -- if included transitively
- Crypto library (`<Crypto.h>`, `<AES.h>`, `<SHA256.h>`, `<Ed25519.h>`) --
  the rweather/Crypto library is pure C++ and may compile natively, or may need
  the Arduino-specific parts stubbed.

---

## 9. Recommendations

### Phase 1: Foundation (Immediate)

**Framework choice:**
- **Native tests:** GoogleTest + GMock (`test_framework = googletest`)
- **On-device tests (future):** Unity (`test_framework = unity`)

**Rationale:** MeshCore's C++ abstract interfaces are a perfect match for
GMock. GoogleTest provides the richest assertion/matcher library. PlatformIO
has first-class support. Unity is reserved for future on-device tests where
GoogleTest is too large.

**Initial setup:**
1. Add `[env:native]` to root `platformio.ini`.
2. Create `test/` directory with initial test suites.
3. Start with the easiest, highest-value targets:
   - `test_packet` -- Packet serialization round-trip
   - `test_utils` -- Hex conversion, text parsing
   - `test_advert_data` -- AdvertDataBuilder/Parser round-trip
   - `test_str_helper` -- String utility functions
4. Provide minimal stubs for Arduino APIs (`Stream.h`, `millis()`).

### Phase 2: Core Protocol Testing

**With mocks:**
1. `test_packet_queue` -- StaticPoolPacketManager behavior
2. `test_identity` -- Crypto operations (sign, verify, ECDH)
3. `test_mesh_routing` -- Mesh routing with MockRadio, MockPacketManager, etc.
4. `test_cli` -- CommonCLI command handling with MockCallbacks

### Phase 3: CI/CD Integration

1. GitHub Actions workflow for native tests on every PR.
2. Add `pio check` (cppcheck) for static analysis.
3. Code coverage reporting with gcov/lcov.
4. Build verification for key hardware targets.

### Phase 4: Advanced Testing

1. Fuzz testing for `Packet::readFrom()` and `AdvertDataParser`.
2. Multi-node mesh simulation in native tests.
3. clang-tidy integration.
4. On-device integration tests (if hardware CI runners become available).

### Deferred / Not Recommended

- **Catch2** -- no PlatformIO integration, no advantage over GoogleTest.
- **CppUTest** -- viable but less powerful mocking than GMock.
- **ArduinoFake** -- MeshCore's abstractions make it unnecessary.
- **CMock** -- designed for C functions, not C++ virtual interfaces.
- **Renode emulation** -- SoftDevice/Bluefruit incompatibility makes it impractical.
- **QEMU** -- no nRF52840 or ESP32 support.
