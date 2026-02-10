# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What Is This

IronRDP is a Rust implementation of the Microsoft Remote Desktop Protocol (RDP) with a focus on security. It is a large multi-crate workspace (~42 crates) organized in a tiered architecture. Maintained by Devolutions.

## Build & Development Commands

All automation uses the `cargo xtask` pattern (no Makefile/justfile):

```bash
cargo xtask ci                    # Run full CI locally (must pass before PR)
cargo xtask check fmt             # Check rustfmt formatting
cargo xtask check lints           # Run clippy
cargo xtask check tests           # Compile and run all tests
cargo xtask check typos           # Check for typos (typos-cli)
cargo xtask check locks           # Verify lock files
cargo xtask wasm check            # Validate WASM build
cargo xtask web run               # Dev server for Svelte web client
cargo xtask ffi build             # Build native FFI DLL
cargo xtask ffi bindings          # Generate C# bindings
cargo xtask fuzz run              # Run fuzzing targets
cargo xtask cov report --html     # HTML coverage report
```

Run a single test by name:
```bash
cargo test -p ironrdp-testsuite-core -- test_name
cargo test -p ironrdp-testsuite-extra -- test_name
```

Update snapshot tests: `UPDATE_EXPECT=1 cargo test`

Toolchain: Rust 1.88.0 (see `rust-toolchain.toml`). MSRV for clippy: 1.87.

## Architecture: Tiered Crate System

Read `ARCHITECTURE.md` for full details. The key principle: **the tier determines what a crate is allowed to do**.

### Core Tier (`crates/ironrdp-core`, `ironrdp-pdu`, `ironrdp-connector`, `ironrdp-session`, `ironrdp-graphics`, `ironrdp-svc`, `ironrdp-dvc`, `ironrdp-cliprdr`, `ironrdp-rdpdr`, `ironrdp-rdpsnd`, `ironrdp-input`, `ironrdp-error`, `ironrdp-propertyset`, `ironrdp-rdpfile`, `ironrdp-rdcleanpath`)

Strict invariants:
- **No I/O allowed** — pure protocol logic only
- **Must be `#![no_std]`-compatible** (with `alloc` feature, `std` enabled by default)
- **Must be fuzzed** (targets in `fuzz/`)
- No platform-specific code (`#[cfg(windows)]` etc.)
- No proc-macro dependencies (keep compile times low)
- Minimize monomorphization at API boundaries
- All crates are **API Boundaries** with semver obligations

Key types: `Encode`/`Decode` traits in `ironrdp-core`, `ReadCursor`/`WriteCursor`/`WriteBuf` for no_std encoding.

### Extra Tier (`ironrdp-blocking`, `ironrdp-async`, `ironrdp-tokio`, `ironrdp-futures`, `ironrdp-tls`, `ironrdp-client`, `ironrdp-web`, `ironrdp-cliprdr-native`, `ironrdp-rdpsnd-native`, `ironrdp-cfg`)

Higher-level libraries with relaxed constraints. I/O is allowed here. `ironrdp-client` is the native RDP client binary.

### Internal Tier (never published)

- `ironrdp-testsuite-core` / `ironrdp-testsuite-extra` — all integration tests live here as single binaries organized by module
- `ironrdp-pdu-generators` / `ironrdp-session-generators` — proptest generators
- `ironrdp-fuzzing` — fuzzing oracles
- `xtask` — build automation

**Architectural invariant**: the core test suite must not depend on any extra tier crate.

### Community Tier (`ironrdp-acceptor`, `ironrdp-server`, `ironrdp-mstsgu`)

Community-maintained. Core team will notify maintainers of breakage but may not fix it.

## Code Style (see `STYLE.md`)

Key conventions that differ from defaults:

- **Error messages**: lowercase, no trailing punctuation (`"invalid X.509 certificate"`)
- **Log messages** (tracing): capitalize, no period (`info!("Connect to RDP host")`)
- **Structured tracing fields**: use `error` as field name for errors, not `e`/`err`
- **Size constants**: annotate each field with inline comments (`1 /* Version */ + 2 /* Length */`)
- **Comparisons**: prefer `<` and `<=` over `>` and `>=` (number line order)
- **Invariants**: mark with `INVARIANT:` prefix in comments; state positively (`if !(idx < len)` not `if idx >= len`)
- **Context parameters** come first in function signatures
- **Helper functions**: put nested helpers at end of enclosing function (requires `return`)
- **Avoid single-use helper functions** — use blocks instead
- **Doc comments**: link to MS spec sections using reference-style links
- **Avoid monomorphization** at crate boundaries — use `&mut dyn` inner functions
- **Result types**: always qualify (`crate_name::Result`, not bare `Result`)
- Use `thiserror` in libraries, `anyhow` in binaries
- **Banned in libraries**: `unwrap()`, `panic!()`, `print!`/`println!`/`eprintln!`, `dbg!`, `todo!`

## Formatting

`rustfmt.toml`: max_width=120, imports_granularity="Module", group_imports="StdExternalCrate"

## Testing Philosophy

- **Test at the boundaries** — test features via public API, not internal code
- Tests go in `ironrdp-testsuite-core` (for core tier) or `ironrdp-testsuite-extra` (for extra tier), not alongside source
- **No external resources** — all tests must be perfectly reproducible
- Binary test data in separate files (`*.bin`, `*.bmp`), loaded with `include_bytes!`
- Use `expect-test` for snapshot testing, `rstest` for fixtures, `proptest` for property testing

## Key Workspace Notes

- `ironrdp` (meta crate) only re-exports — it contains no logic
- Workspace dependencies are intentionally limited (see comment in root `Cargo.toml`) — avoid adding workspace deps for published crate dependencies
- `num-derive`/`num-traits` are legacy — avoid new usage
- `diplomat` uses a patched fork (see `[patch.crates-io]`)
- Excluded crates (`ironrdp-client-glutin`, `ironrdp-glutin-renderer`, `ironrdp-replay-client`) have broken compilation
- Linux builds need `libasound2-dev`; Windows needs NASM for cryptography
- Web client is TypeScript/Svelte in `web-client/` (not in Cargo workspace)
- FFI targets .NET via `diplomat` in `ffi/`

## CI Equivalence Invariant

`cargo xtask ci` locally must be logically equivalent to the GitHub Actions CI workflow. If it passes locally, CI should be green.
