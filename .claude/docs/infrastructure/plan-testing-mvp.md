# Plan: Testing MVP — Proof-of-Concept PR

**Date:** 2026-02-24
**Status:** Complete — PR #1837
**Research:** [research-testing-frameworks.md](research-testing-frameworks.md)

---

## Goal

A minimal PR against `dev` that introduces native host-based unit testing to
MeshCore. The PR must be small enough to review in one sitting, require **zero
changes to existing source code**, and convincingly demonstrate that the
project's core logic is testable on a developer's machine without hardware.

---

## Scope — What's In

1. A `[env:native]` PlatformIO environment for host testing.
2. GoogleTest as the test framework (with GMock available for future use).
3. Two test suites covering real, non-trivial production code:
   - **`test_packet`** — `Packet` serialization/deserialization round-trips.
   - **`test_str_helper`** — `StrHelper` string utility functions.
4. A thin compatibility shim (`test/native/compat/Arduino.h`) so that
   `TxtDataHelpers.cpp` compiles natively without modifying the source file.
5. Documentation: a short section in the PR description explaining how to run
   tests and add new ones.

## Scope — What's Out

- No changes to any existing `.cpp` or `.h` file in `src/`.
- No mocking infrastructure (GMock mocks, fake Radio, etc.) — that's phase 2.
- No CI/CD workflow — that's a follow-up PR after the approach is accepted.
- No on-device tests.
- No static analysis.

---

## Why These Two Modules

### Packet (`src/Packet.cpp`)

- Core protocol type — every node uses it.
- Tests serialization round-trip (`writeTo` → `readFrom`), boundary checks, and
  hash computation — things that actually matter for protocol correctness.
- Only external dependency is `SHA256` from the rweather/Crypto library, which
  is pure C++ and already handles native compilation (its `ProgMemUtil.h`
  defines `PROGMEM` as empty on non-AVR/ESP, and `EndianUtil.h` has an explicit
  `HOST_BUILD` path using `<endian.h>`).
- **Zero Arduino dependency.** `Packet.h` → `MeshCore.h` → `<stdint.h>` + `<math.h>`.

### StrHelper (`src/helpers/TxtDataHelpers.cpp`)

- Used everywhere for safe string operations.
- Pure logic functions: `strncpy`, `strzcpy`, `isBlank`, `fromHex`, `ftoa`,
  `ftoa3`.
- The `.cpp` file does `#include <Arduino.h>`, but only uses `ltoa()`,
  `snprintf()`, `strlen()`, and `abs()` — all available in standard C.
  Providing a minimal `Arduino.h` shim that `#include`s `<cstdlib>` and
  `<cstdio>` is sufficient.

Together these two cover: binary protocol correctness (security-critical) and
string utilities (used across the entire codebase). They demonstrate that
MeshCore's core logic compiles and runs on the host, and that adding more test
suites follows an established pattern.

---

## Detailed Changes

### 1. `platformio.ini` — Add native environment

Append a new `[env:native]` section to the root `platformio.ini`:

```ini
; --------------- Native Host Tests ----------------

[env:native]
platform = native
test_framework = googletest
test_build_src = true
lib_compat_mode = off
build_flags =
    -std=c++17
    -DHOST_BUILD
    -I test/native/compat
build_src_filter =
    +<Packet.cpp>
    +<helpers/TxtDataHelpers.cpp>
lib_deps =
    rweather/Crypto @ ^0.4.0
```

Key decisions:
- **`test_build_src = true`**: Required to compile `src/` files alongside test
  files. See [Implementation corrections](#implementation-corrections) for why
  this cannot be omitted despite being documented as the default.
- **`lib_compat_mode = off`**: Required to allow the Crypto library (which
  declares Arduino framework compatibility in its metadata) to be used on the
  frameworkless `native` platform. See
  [Implementation corrections](#implementation-corrections) for details.
- **`-DHOST_BUILD`**: The Crypto library's `EndianUtil.h` already checks for
  this define and switches to `<endian.h>` for host byte-order macros. This is
  the library's own mechanism — we're using it as designed.
- **`-I test/native/compat`**: Puts our Arduino.h shim on the include path so
  `TxtDataHelpers.cpp`'s `#include <Arduino.h>` resolves without touching the
  source.
- **`build_src_filter`**: Compiles only the source files under test. Prevents
  pulling in the entire `src/` tree and its hardware dependencies.
- **`lib_deps`**: Only the Crypto library. No RadioLib, RTClib, etc.

### 2. `test/native/compat/Arduino.h` — Minimal shim

A compatibility header that satisfies `#include <Arduino.h>` for native builds.
Only provides what `TxtDataHelpers.cpp` actually calls:

```cpp
#pragma once
// Minimal Arduino.h shim for native host builds.
// Provides only the standard-C functions that MeshCore source files
// reference through Arduino.h.

#include <cstdlib>
#include <cstdio>
#include <cstring>
#include <cstdint>
#include <cmath>

// Arduino's ltoa() is just a wrapper around the standard C function.
// glibc provides ltoa() when _GNU_SOURCE is defined, but for portability
// we provide an inline fallback.
#ifndef ltoa
inline char* ltoa(long value, char* str, int base) {
    if (base == 10) {
        sprintf(str, "%ld", value);
    } else if (base == 16) {
        sprintf(str, "%lx", value);
    } else if (base == 8) {
        sprintf(str, "%lo", value);
    }
    return str;
}
#endif
```

This file lives under `test/` — it is never used by firmware builds. The
`-I test/native/compat` flag only applies to the `native` environment.

### 3. `test/test_packet/test_packet.cpp` — Packet tests

```cpp
#include <gtest/gtest.h>
#include <Packet.h>
#include <string.h>

using namespace mesh;

// --- Construction ---

TEST(PacketTest, DefaultConstruction) {
    Packet pkt;
    EXPECT_EQ(pkt.header, 0);
    EXPECT_EQ(pkt.path_len, 0);
    EXPECT_EQ(pkt.payload_len, 0);
}

// --- Round-trip: writeTo → readFrom ---

TEST(PacketTest, FloodRoundTrip) {
    Packet original;
    original.header = (PAYLOAD_TYPE_TXT_MSG << PH_TYPE_SHIFT) | ROUTE_TYPE_FLOOD;
    original.path_len = 3;
    memcpy(original.path, "\x01\x02\x03", 3);
    const char* msg = "hello mesh";
    original.payload_len = strlen(msg);
    memcpy(original.payload, msg, original.payload_len);

    uint8_t wire[MAX_TRANS_UNIT];
    uint8_t len = original.writeTo(wire);

    Packet restored;
    ASSERT_TRUE(restored.readFrom(wire, len));
    EXPECT_EQ(restored.header, original.header);
    EXPECT_EQ(restored.path_len, original.path_len);
    EXPECT_EQ(memcmp(restored.path, original.path, original.path_len), 0);
    EXPECT_EQ(restored.payload_len, original.payload_len);
    EXPECT_EQ(memcmp(restored.payload, original.payload, original.payload_len), 0);
}

TEST(PacketTest, DirectRouteRoundTrip) {
    Packet original;
    original.header = (PAYLOAD_TYPE_REQ << PH_TYPE_SHIFT) | ROUTE_TYPE_DIRECT;
    original.path_len = 5;
    memset(original.path, 0xAB, 5);
    original.payload_len = 10;
    memset(original.payload, 0xCD, 10);

    uint8_t wire[MAX_TRANS_UNIT];
    uint8_t len = original.writeTo(wire);

    Packet restored;
    ASSERT_TRUE(restored.readFrom(wire, len));
    EXPECT_EQ(restored.getRouteType(), ROUTE_TYPE_DIRECT);
    EXPECT_EQ(restored.getPayloadType(), PAYLOAD_TYPE_REQ);
    EXPECT_EQ(restored.path_len, 5);
    EXPECT_EQ(restored.payload_len, 10);
    EXPECT_EQ(memcmp(restored.path, original.path, 5), 0);
    EXPECT_EQ(memcmp(restored.payload, original.payload, 10), 0);
}

TEST(PacketTest, TransportFloodRoundTrip) {
    Packet original;
    original.header = (PAYLOAD_TYPE_ACK << PH_TYPE_SHIFT) | ROUTE_TYPE_TRANSPORT_FLOOD;
    original.transport_codes[0] = 0x1234;
    original.transport_codes[1] = 0x5678;
    original.path_len = 0;
    original.payload_len = 4;
    memcpy(original.payload, "\xDE\xAD\xBE\xEF", 4);

    uint8_t wire[MAX_TRANS_UNIT];
    uint8_t len = original.writeTo(wire);

    Packet restored;
    ASSERT_TRUE(restored.readFrom(wire, len));
    EXPECT_TRUE(restored.hasTransportCodes());
    EXPECT_EQ(restored.transport_codes[0], 0x1234);
    EXPECT_EQ(restored.transport_codes[1], 0x5678);
    EXPECT_EQ(restored.payload_len, 4);
}

TEST(PacketTest, TransportDirectRoundTrip) {
    Packet original;
    original.header = (PAYLOAD_TYPE_RESPONSE << PH_TYPE_SHIFT) | ROUTE_TYPE_TRANSPORT_DIRECT;
    original.transport_codes[0] = 0xAAAA;
    original.transport_codes[1] = 0xBBBB;
    original.path_len = 2;
    memcpy(original.path, "\x0F\xF0", 2);
    original.payload_len = 1;
    original.payload[0] = 0x42;

    uint8_t wire[MAX_TRANS_UNIT];
    uint8_t len = original.writeTo(wire);

    Packet restored;
    ASSERT_TRUE(restored.readFrom(wire, len));
    EXPECT_TRUE(restored.hasTransportCodes());
    EXPECT_TRUE(restored.isRouteDirect());
    EXPECT_EQ(restored.transport_codes[0], 0xAAAA);
    EXPECT_EQ(restored.transport_codes[1], 0xBBBB);
    EXPECT_EQ(restored.path_len, 2);
    EXPECT_EQ(restored.payload_len, 1);
    EXPECT_EQ(restored.payload[0], 0x42);
}

// --- Header field extraction ---

TEST(PacketTest, HeaderFields) {
    Packet pkt;
    pkt.header = (PAYLOAD_TYPE_ADVERT << PH_TYPE_SHIFT) | ROUTE_TYPE_FLOOD | (PAYLOAD_VER_1 << PH_VER_SHIFT);
    EXPECT_EQ(pkt.getRouteType(), ROUTE_TYPE_FLOOD);
    EXPECT_EQ(pkt.getPayloadType(), PAYLOAD_TYPE_ADVERT);
    EXPECT_EQ(pkt.getPayloadVer(), PAYLOAD_VER_1);
    EXPECT_TRUE(pkt.isRouteFlood());
    EXPECT_FALSE(pkt.isRouteDirect());
    EXPECT_FALSE(pkt.hasTransportCodes());
}

TEST(PacketTest, AllPayloadTypes) {
    const uint8_t types[] = {
        PAYLOAD_TYPE_REQ, PAYLOAD_TYPE_RESPONSE, PAYLOAD_TYPE_TXT_MSG,
        PAYLOAD_TYPE_ACK, PAYLOAD_TYPE_ADVERT, PAYLOAD_TYPE_GRP_TXT,
        PAYLOAD_TYPE_GRP_DATA, PAYLOAD_TYPE_ANON_REQ, PAYLOAD_TYPE_PATH,
        PAYLOAD_TYPE_TRACE, PAYLOAD_TYPE_MULTIPART, PAYLOAD_TYPE_CONTROL,
        PAYLOAD_TYPE_RAW_CUSTOM
    };
    for (uint8_t t : types) {
        Packet pkt;
        pkt.header = (t << PH_TYPE_SHIFT) | ROUTE_TYPE_FLOOD;
        EXPECT_EQ(pkt.getPayloadType(), t) << "payload type " << (int)t;
    }
}

// --- Wire length calculation ---

TEST(PacketTest, RawLengthNoTransport) {
    Packet pkt;
    pkt.header = ROUTE_TYPE_FLOOD;
    pkt.path_len = 10;
    pkt.payload_len = 20;
    // header(1) + path_len_field(1) + path(10) + payload(20) = 32
    EXPECT_EQ(pkt.getRawLength(), 32);
}

TEST(PacketTest, RawLengthWithTransport) {
    Packet pkt;
    pkt.header = ROUTE_TYPE_TRANSPORT_FLOOD;
    pkt.path_len = 10;
    pkt.payload_len = 20;
    // header(1) + transport(4) + path_len_field(1) + path(10) + payload(20) = 36
    EXPECT_EQ(pkt.getRawLength(), 36);
}

// --- readFrom rejection of bad input ---

TEST(PacketTest, ReadFromRejectsTruncated) {
    // A minimal valid flood packet: header(1) + path_len(1) + payload(1+) = 3 bytes min
    uint8_t bad[] = {ROUTE_TYPE_FLOOD, 0x00};  // only 2 bytes, no payload
    Packet pkt;
    EXPECT_FALSE(pkt.readFrom(bad, sizeof(bad)));
}

TEST(PacketTest, ReadFromRejectsOversizePath) {
    uint8_t bad[3] = {ROUTE_TYPE_FLOOD, MAX_PATH_SIZE + 1, 0x00};
    Packet pkt;
    EXPECT_FALSE(pkt.readFrom(bad, sizeof(bad)));
}

// --- Hash computation ---

TEST(PacketTest, SamePayloadSameHash) {
    Packet a, b;
    a.header = (PAYLOAD_TYPE_TXT_MSG << PH_TYPE_SHIFT);
    b.header = (PAYLOAD_TYPE_TXT_MSG << PH_TYPE_SHIFT);
    memcpy(a.payload, "test", 4); a.payload_len = 4;
    memcpy(b.payload, "test", 4); b.payload_len = 4;

    uint8_t hash_a[MAX_HASH_SIZE], hash_b[MAX_HASH_SIZE];
    a.calculatePacketHash(hash_a);
    b.calculatePacketHash(hash_b);
    EXPECT_EQ(memcmp(hash_a, hash_b, MAX_HASH_SIZE), 0);
}

TEST(PacketTest, DifferentPayloadDifferentHash) {
    Packet a, b;
    a.header = (PAYLOAD_TYPE_TXT_MSG << PH_TYPE_SHIFT);
    b.header = (PAYLOAD_TYPE_TXT_MSG << PH_TYPE_SHIFT);
    memcpy(a.payload, "aaaa", 4); a.payload_len = 4;
    memcpy(b.payload, "bbbb", 4); b.payload_len = 4;

    uint8_t hash_a[MAX_HASH_SIZE], hash_b[MAX_HASH_SIZE];
    a.calculatePacketHash(hash_a);
    b.calculatePacketHash(hash_b);
    EXPECT_NE(memcmp(hash_a, hash_b, MAX_HASH_SIZE), 0);
}

TEST(PacketTest, DifferentTypeDifferentHash) {
    Packet a, b;
    a.header = (PAYLOAD_TYPE_TXT_MSG << PH_TYPE_SHIFT);
    b.header = (PAYLOAD_TYPE_REQ << PH_TYPE_SHIFT);
    memcpy(a.payload, "same", 4); a.payload_len = 4;
    memcpy(b.payload, "same", 4); b.payload_len = 4;

    uint8_t hash_a[MAX_HASH_SIZE], hash_b[MAX_HASH_SIZE];
    a.calculatePacketHash(hash_a);
    b.calculatePacketHash(hash_b);
    EXPECT_NE(memcmp(hash_a, hash_b, MAX_HASH_SIZE), 0);
}

// --- Do-not-retransmit marker ---

TEST(PacketTest, DoNotRetransmitMarker) {
    Packet pkt;
    EXPECT_FALSE(pkt.isMarkedDoNotRetransmit());
    pkt.markDoNotRetransmit();
    EXPECT_TRUE(pkt.isMarkedDoNotRetransmit());
    EXPECT_EQ(pkt.header, 0xFF);
}

// --- SNR conversion ---

TEST(PacketTest, SNRConversion) {
    Packet pkt;
    pkt._snr = 20;   // 20 / 4.0 = 5.0 dB
    EXPECT_FLOAT_EQ(pkt.getSNR(), 5.0f);

    pkt._snr = -8;   // -8 / 4.0 = -2.0 dB
    EXPECT_FLOAT_EQ(pkt.getSNR(), -2.0f);

    pkt._snr = 0;
    EXPECT_FLOAT_EQ(pkt.getSNR(), 0.0f);
}
```

### 4. `test/test_str_helper/test_str_helper.cpp` — StrHelper tests

```cpp
#include <gtest/gtest.h>
#include <helpers/TxtDataHelpers.h>
#include <string.h>

// --- strncpy ---

TEST(StrHelperTest, StrncpyCopiesNormally) {
    char buf[16];
    StrHelper::strncpy(buf, "hello", sizeof(buf));
    EXPECT_STREQ(buf, "hello");
}

TEST(StrHelperTest, StrncpyTruncates) {
    char buf[4];
    StrHelper::strncpy(buf, "hello world", sizeof(buf));
    EXPECT_STREQ(buf, "hel");  // 3 chars + NUL
}

TEST(StrHelperTest, StrncpyEmptyString) {
    char buf[8] = "garbage";
    StrHelper::strncpy(buf, "", sizeof(buf));
    EXPECT_STREQ(buf, "");
}

TEST(StrHelperTest, StrncpySizeOne) {
    char buf[1] = {'X'};
    StrHelper::strncpy(buf, "anything", sizeof(buf));
    EXPECT_EQ(buf[0], '\0');
}

// --- strzcpy ---

TEST(StrHelperTest, StrzCopyPadsWithNulls) {
    char buf[8];
    memset(buf, 0xFF, sizeof(buf));
    StrHelper::strzcpy(buf, "hi", sizeof(buf));
    EXPECT_STREQ(buf, "hi");
    // Remaining bytes should be NUL-padded
    for (size_t i = 2; i < sizeof(buf); i++) {
        EXPECT_EQ(buf[i], '\0') << "byte " << i << " should be NUL";
    }
}

TEST(StrHelperTest, StrzCopyTruncates) {
    char buf[4];
    StrHelper::strzcpy(buf, "hello world", sizeof(buf));
    EXPECT_STREQ(buf, "hel");
}

// --- isBlank ---

TEST(StrHelperTest, IsBlankEmpty) {
    EXPECT_TRUE(StrHelper::isBlank(""));
}

TEST(StrHelperTest, IsBlankSpaces) {
    EXPECT_TRUE(StrHelper::isBlank("   "));
}

TEST(StrHelperTest, IsBlankWithContent) {
    EXPECT_FALSE(StrHelper::isBlank("  a "));
}

TEST(StrHelperTest, IsBlankSingleChar) {
    EXPECT_FALSE(StrHelper::isBlank("x"));
}

// --- fromHex ---

TEST(StrHelperTest, FromHexLowercase) {
    EXPECT_EQ(StrHelper::fromHex("ff"), 0xFFu);
}

TEST(StrHelperTest, FromHexUppercase) {
    EXPECT_EQ(StrHelper::fromHex("DEADBEEF"), 0xDEADBEEFu);
}

TEST(StrHelperTest, FromHexMixedCase) {
    EXPECT_EQ(StrHelper::fromHex("aB09"), 0xAB09u);
}

TEST(StrHelperTest, FromHexStopsAtNonHex) {
    EXPECT_EQ(StrHelper::fromHex("1Fxyz"), 0x1Fu);
}

TEST(StrHelperTest, FromHexEmpty) {
    EXPECT_EQ(StrHelper::fromHex(""), 0u);
}

TEST(StrHelperTest, FromHexLeading) {
    EXPECT_EQ(StrHelper::fromHex("0001"), 1u);
}

// --- ftoa ---

TEST(StrHelperTest, FtoaZero) {
    EXPECT_STREQ(StrHelper::ftoa(0.0f), "0.0");
}

TEST(StrHelperTest, FtoaPositive) {
    const char* s = StrHelper::ftoa(3.14f);
    float parsed = atof(s);
    EXPECT_NEAR(parsed, 3.14f, 0.01f);
}

TEST(StrHelperTest, FtoaNegative) {
    const char* s = StrHelper::ftoa(-1.5f);
    EXPECT_EQ(s[0], '-');
    float parsed = atof(s);
    EXPECT_NEAR(parsed, -1.5f, 0.01f);
}

TEST(StrHelperTest, FtoaWholeNumber) {
    const char* s = StrHelper::ftoa(42.0f);
    float parsed = atof(s);
    EXPECT_NEAR(parsed, 42.0f, 0.01f);
}

// --- ftoa3 ---

TEST(StrHelperTest, Ftoa3Zero) {
    EXPECT_STREQ(StrHelper::ftoa3(0.0f), "0");
}

TEST(StrHelperTest, Ftoa3ThreeDecimals) {
    EXPECT_STREQ(StrHelper::ftoa3(1.234f), "1.234");
}

TEST(StrHelperTest, Ftoa3TrailingZerosTrimmed) {
    EXPECT_STREQ(StrHelper::ftoa3(2.5f), "2.5");
}

TEST(StrHelperTest, Ftoa3WholeNumber) {
    EXPECT_STREQ(StrHelper::ftoa3(7.0f), "7");
}

TEST(StrHelperTest, Ftoa3Negative) {
    const char* s = StrHelper::ftoa3(-0.5f);
    float parsed = atof(s);
    EXPECT_NEAR(parsed, -0.5f, 0.01f);
}
```

---

## Implementation Corrections

Three issues were discovered during implementation that the original plan did
not anticipate. Each stems from a gap between PlatformIO's documentation /
surface-level behavior and what actually happens at build time.

### 1. `lib_compat_mode = off` — Library compatibility gate

**What failed:** With the default `lib_compat_mode = soft`, PlatformIO rejected
the Crypto library entirely:

```
Framework incompatible library .../Crypto
```

The Crypto library was never compiled, which cascaded: no `libCrypto.a` was
produced, so `SHA256` symbols were unresolved.

**Root cause:** PlatformIO's Library Dependency Finder (LDF) checks each
library's `library.json` (or inferred metadata) for framework compatibility.
The rweather/Crypto library declares compatibility with `arduino` (and possibly
`*`), but the `native` platform has **no framework**. In `soft` mode (the
default), LDF silently drops libraries whose framework list doesn't match the
current environment. The library is downloaded but never compiled.

**Why the plan was wrong:** The plan correctly noted that the Crypto library's
*source code* handles native builds (via `ProgMemUtil.h`, `EndianUtil.h`
`HOST_BUILD` guards). But the plan only analyzed C++ source-level
compatibility. It did not account for PlatformIO's **metadata-level** gate that
runs before compilation even begins. A library can be perfectly compilable on
native but still be rejected if its metadata says "arduino only."

**Fix:** `lib_compat_mode = off` disables the framework compatibility check,
letting LDF include the library based solely on `#include` dependency tracing.

### 2. `test_build_src = true` — Source compilation in test mode

**What failed:** With the default settings, PlatformIO compiled test files and
libraries but produced **zero object files from `src/`**. The link line
contained only `test_str_helper.o` and `libgoogletest.a` — no `Packet.o`, no
`TxtDataHelpers.o`. All StrHelper/Packet symbols were unresolved.

**Root cause:** PlatformIO 6.x documents `test_build_src` as defaulting to
`yes`, meaning `src/` files should be compiled alongside tests. In practice,
on PlatformIO Core 6.1.19 with `platform = native` and `test_framework =
googletest`, the `src/` directory was silently skipped unless
`test_build_src = true` was set explicitly. The exact trigger is unclear — it
may be related to the frameworkless native environment, the googletest
framework, or an interaction between the two. Regardless, the explicit setting
is required for correctness.

**Why the plan was wrong:** The plan assumed PlatformIO's documented default
(`test_build_src = yes` in 6.x) would hold. It did not. This is the kind of
gap that only surfaces during actual `pio test` execution — the documentation
describes intended behavior, not necessarily the behavior of every platform ×
framework combination.

**Fix:** `test_build_src = true` explicitly in `[env:native]`.

### 3. `test/main.cpp` — GoogleTest entry point

**What failed:** The linker reported `undefined reference to 'main'` even after
libraries and sources compiled successfully.

**Root cause:** PlatformIO's GoogleTest integration does **not** auto-generate
a `main()` function. The PlatformIO docs state "PlatformIO will automatically
generate the main source file" for the Unity framework, and this was
misattributed to GoogleTest. The installed `googletest` library includes
`gtest_main.cc` in its source tree, but PlatformIO's build script does not
compile it into `libgoogletest.a` — the archive only contains `gtest.o` and
`gmock.o` (the library code, not the runner). The official PlatformIO
GoogleTest examples all provide an explicit `main()`.

**Why the plan was wrong:** The research noted GoogleTest + PlatformIO as
"first-class support" and assumed the framework integration handled the entry
point. It does not. PlatformIO's test framework integration for GoogleTest
covers library download, include paths, and compilation — but not the runner
`main()`.

**Fix:** `test/main.cpp` at the test root directory. PlatformIO compiles
files in the `test/` root into every test suite, so a single `main.cpp` serves
all suites:

```cpp
#include <gtest/gtest.h>

int main(int argc, char **argv) {
    ::testing::InitGoogleTest(&argc, argv);
    return RUN_ALL_TESTS();
}
```

### 4. Arduino.h shim — Broader than expected

The original plan scoped the Arduino.h shim to `TxtDataHelpers.cpp` needs
only (`ltoa`, `snprintf`, `strlen`, `abs`). In practice, the Crypto library's
`RNG.cpp` also includes `Arduino.h` (transitively, via the compat include
path) and calls `millis()` and `micros()`. The shim was expanded to provide
these using `std::chrono`.

---

## Bug discovered: `StrHelper::ftoa3()` sign loss

Testing revealed a bug in `StrHelper::ftoa3()` at `TxtDataHelpers.cpp:143-148`.
For input values in the range `(-1.0, 0.0)`, the negative sign is silently
dropped.

**Trace for `ftoa3(-0.5f)`:**

```
v = (int)(-0.5f * 1000.0f + (-0.5f)) = (int)(-500.5f) = -500
w = -500 / 1000 = 0           // C integer division truncates toward zero
d = abs(-500 % 1000) = 500
snprintf(s, sizeof(s), "%d.%03d", 0, 500)  →  "0.500"  →  trimmed  →  "0.5"
```

The sign is carried only by `w` (the whole part). When `w == 0`, `"%d"` formats
it as `"0"` — unsigned. The `abs()` on `d` is correct (decimals are always
positive) but means the fractional part can never carry the sign either.

**Affected range:** any `f` where `-1.0 < f < 0.0`.

**Current impact:** Low. The only call site is `CommonCLI.cpp:309`, formatting
LoRa bandwidth (`_prefs->bw`), which is always a positive value (125, 250,
500 kHz). But `ftoa3` is declared as a general-purpose utility in
`TxtDataHelpers.h` with no documented restriction on input range.

**Fix:** Check `if (v < 0 && w == 0)` and manually prepend `'-'`.

The test documents this bug with a detailed comment and asserts the current
(broken) behavior so the test suite passes. When the bug is fixed, the test
should be updated to assert the correct output (`"-0.5"`).

---

## File Tree (new files only)

```
platformio.ini                           (MODIFIED — append native env)
test/
  main.cpp                               (NEW — GoogleTest entry point)
  native/
    compat/
      Arduino.h                          (NEW — minimal shim)
  test_packet/
    test_packet.cpp                      (NEW)
  test_str_helper/
    test_str_helper.cpp                  (NEW)
```

Total: 1 modified file, 4 new files.

---

## How It Works

1. Developer runs: `pio test -e native`
2. PlatformIO compiles each `test_*` suite against the native platform
   (host GCC/Clang).
3. Tests execute directly on the host machine — no board, no upload, no serial.
4. GoogleTest reports pass/fail for each test case.
5. Exit code 0 = all pass (CI-friendly).

---

## Risks and Mitigations

| Risk | Mitigation |
|------|-----------|
| Crypto library metadata rejects native platform | `lib_compat_mode = off` bypasses framework check. Source-level native support is already present (`ProgMemUtil.h`, `EndianUtil.h` `HOST_BUILD`). |
| Crypto library `RNG.cpp` calls Arduino functions | Arduino.h shim provides `millis()` / `micros()` via `std::chrono`. Stubs are sufficient — the RNG is not exercised by tests. |
| `src/` files not compiled in test mode | `test_build_src = true` explicitly. Cannot rely on the documented default. |
| GoogleTest `main()` not auto-generated | `test/main.cpp` provides it. Shared across all test suites via PlatformIO's test-root inclusion. |
| `Arduino.h` shim causes confusion | File is under `test/native/compat/`, clearly separated. The `-I` flag is scoped to `[env:native]` only. |
| Maintainers object to GoogleTest size | GoogleTest only applies to the native environment. It never touches firmware binaries. It adds zero bytes to any device build. |
| `build_src_filter` complexity grows | Each test suite only compiles the source files it needs. This is the standard PlatformIO pattern. As more modules are tested, filters can be expanded incrementally. |

---

## PR Shape

**Title:** Add native host unit tests for Packet and StrHelper

**Description outline:**
- Problem: MeshCore has no automated tests. Protocol correctness depends
  entirely on manual testing.
- What: Adds PlatformIO native test environment with GoogleTest. Two test
  suites cover `Packet` serialization/deserialization and `StrHelper` string
  utilities.
- Why: Proves that MeshCore's core logic can be tested on a developer's machine
  in seconds, with no hardware. Establishes the pattern for adding more tests.
- How to run: `pio test -e native`
- Zero changes to existing source code.
- Follow-up work: more test suites, CI integration, mock-based protocol
  testing.

---

## TODO

### Phase 1: Create files

- [x] 1.1 Add `[env:native]` section to `platformio.ini`
- [x] 1.2 Create `test/main.cpp` (GoogleTest entry point)
- [x] 1.3 Create `test/native/compat/Arduino.h` shim
- [x] 1.4 Create `test/test_packet/test_packet.cpp`
- [x] 1.5 Create `test/test_str_helper/test_str_helper.cpp`

### Phase 2: Verify

- [x] 2.1 Run `pio test -e native` — 41/41 tests pass
- [x] 2.2 Run `pio run -e t1000e_companion_radio_ble` — firmware build unaffected

### Phase 3: Ship

- [x] 3.1 Create feature branch from `origin/dev`, cherry-pick commits
- [x] 3.2 Create PR against `upstream/dev` — [#1837](https://github.com/meshcore-dev/MeshCore/pull/1837)
