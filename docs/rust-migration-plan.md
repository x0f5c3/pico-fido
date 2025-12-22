# Rust migration and Slint UI plan

## Objectives
- Explore how to run Pico FIDO logic from Rust: either by porting incrementally to Rust or by exposing C bindings that can be called from a Rust binary.
- Add a touchscreen approval flow with a Slint-based UI so user presence/verification can be confirmed on screen instead of (or in addition to) the current button.
- Keep firmware-size and security constraints in mind for RP2040/RP2350 and ESP32-S3 targets.

## Current state snapshot
- C codebase built with CMake, Pico SDK/TinyUSB/TinyCBOR/MbedTLS; CTAP2 flows live under `src/fido` (e.g., `cbor_make_credential.c`, `cbor_get_assertion.c`, `fido.c`, `credential.c`).
- User presence is enforced via a physical button; LED indicators surface state changes.
- Build entry point: `CMakeLists.txt` + `pico_sdk_import.cmake`; tests are Python/pytest driven with Docker helpers.

## Option A: Rust bindings around existing C
1. **Expose a stable C surface**: wrap CTAP operations (init, makeCredential, getAssertion, PIN, storage, LED/presence hooks) into a single header/`static` library target in CMake.
2. **Generate Rust bindings**: use `bindgen` (or `cbindgen` if the C API is authored in C++) in a new Cargo crate (e.g., `pico-fido-sys`) that:
   - Builds the C library via `cmake`/`cc` crate.
   - Exposes safe Rust wrappers that own buffers and translate error codes to `Result`.
3. **Rust host crate**: create a higher-level crate (`pico-fido-rs`) that:
   - Handles USB transport (TinyUSB via C bindings or Rust `usb-device` stack on RP2040/ESP32-S3).
   - Manages storage abstractions (flash/OTP access) behind a trait so C and Rust backends can coexist.
4. **Slint UI layer** (Rust):
   - Use the `slint` crate with the software renderer for MCUs; wire it to the display/touch controller driver (e.g., via `embedded-graphics` frame buffer).
   - Present request details (RPID/user name), show a countdown, and gate CTAP `up` (user presence) on a touch “Approve/Reject”.
   - Propagate touch decisions to the CTAP state machine through a channel/callback the C core can poll.
5. **Build/deploy**:
   - Cross-compile with `cargo build --target thumbv6m-none-eabi` (RP2040) or ESP-IDF target; link the generated `pico-fido-sys` static lib.
   - Keep the original C firmware build intact to allow bisecting issues.

## Option B: Progressive Rust rewrite
1. **Runtime/hal selection**: use `embassy-rp` or `rp2040-hal` (and `esp-idf-hal` for ESP32-S3) with `usb-device` + `usbd-hid` to replicate the HID transport.
2. **Crypto/CBOR**: leverage `p256`, `ed25519-dalek`, and `ciborium`/`serde_cbor`; store secrets with `littlefs2`/`embedded-storage` backed by flash + OTP hooks for RP2350/ESP32-S3 secure regions.
3. **State machine port**: mirror modules (`credential`, `management`, `oath/otp`) as Rust modules with strict ownership for key material; keep C code temporarily as reference tests.
4. **Slint UI**: share the same async executor as USB handling; drive UI redraws on CTAP events and block user-presence futures until touch approval.
5. **Compatibility**: ensure VID/PID configurability and existing vendor profiles remain via build-time features.

## Phased roadmap (recommended)
1. **Setup**: add Rust toolchains/targets, create `pico-fido-sys` crate scaffolding, and ensure CMake builds as a static lib artifact.
2. **FFI bridge**: define/publish the C API surface, wrap it in Rust, and validate parity with a host-side integration test that replays a small CTAP transcript.
3. **UI spike**: prototype a Slint screen (mock display) on host and then on target hardware; prove user-presence gating via touch callback.
4. **Incremental Rust adoption**: move non-hardware logic (CBOR parsing, credential storage, OATH/OTP) into Rust while keeping hardware/USB in C; cut over piece by piece.
5. **Full Rust or hybrid release**: stabilize either the binding-based firmware or the Rust-native stack; document build flags and flashing steps; update tests to run against both modes.

## Testing and validation
- Unit tests in Rust for CBOR/state logic; property tests for credential encoding/decoding.
- Host-loop integration tests that exercise `makeCredential`/`getAssertion` through the FFI boundary.
- On-device smoke tests for USB enumeration, touch approval latency, and power/size regressions.
- UI snapshots (Slint) and golden CTAP transcripts to prevent regressions.

## Risks and mitigations
- **Binary size/performance**: measure with and without Slint; use software renderer with trimmed assets and enable `lto`/`panic = abort`.
- **Secure storage**: keep OTP-bound key handling behind a trait; ensure zeroization on both C and Rust sides.
- **FFI safety**: isolate raw pointers to a thin layer; prefer owned buffers and explicit lifetimes in wrappers.
- **Platform support**: start with RP2350 (secure boot + OTP) where Slint footprint is more realistic; keep RP2040 builds button-only if memory is constrained.
