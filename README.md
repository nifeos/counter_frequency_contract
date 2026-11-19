# counter_frequency_contract

[![check](https://github.com/nifeos/counter_frequency_contract/actions/workflows/ci.yml/badge.svg?branch=main)](https://github.com/nifeos/counter_frequency_contract/actions/workflows/ci.yml?query=branch%3Amain)
[![proofs](https://github.com/nifeos/counter_frequency_contract/actions/workflows/proofs.yml/badge.svg?branch=main)](https://github.com/nifeos/counter_frequency_contract/actions/workflows/proofs.yml?query=branch%3Amain)
[![license](https://img.shields.io/badge/license-MIT%20OR%20Apache--2.0-blue.svg)](https://github.com/nifeos/counter_frequency_contract#license)
[![minimum Rust version](https://img.shields.io/badge/rust-1.88%2B-blue.svg)](https://github.com/nifeos/counter_frequency_contract/blob/main/Cargo.toml)

The page that tells a process how many counter ticks make a second, on the architectures that have
no register that says so (aarch64 reads `CNTFRQ_EL0` and maps no page). `no_std`, no dependencies.
A kernel and the user-mode runtime of its processes share this one definition, so the writer and
the reader cannot drift on the page's layout or its fixed virtual address.

The writer learns the rate once at boot, writes the page once, and maps it read-only into every
process, so no seqlock is necessary. A parent writes a child space's page. An unknown rate is a page
of zeroes, which reads as `None`; a page never carries a fabricated default.

## The contract

- The page is `PAGE_CONTENT_BYTES` (16) bytes: `MAGIC` (the ASCII text `TIMEBAS1`) and then the rate in
  hertz as one little-endian `u64`. A frame without the magic reads as unknown. `VERSION` (1) is the
  digit the magic carries; a layout break bumps both in one change.
- `build_page(hertz)` makes the bytes. `CounterFrequencyPage::new(va)` is `unsafe` and names a mapped page;
  `hertz()` returns `Some(rate)` or `None`. `hertz()` returns the number exactly as the page carries
  it, so a page built for 0 reads as `Some(0)`: the reader applies the band below before it uses
  the rate.
- The fixed address is `PAGE_VIRTUAL_ADDRESS`: `0x0000_7000_0000_0000` on `x86_64`, and `0x0000_0038_0000_0000`
  on `riscv64`, where the low half of the address space ends at `0x40_0000_0000`. Each sits at
  seven-eighths of its low half, far above every other address a program receives.
- A rate is believable only between `MINIMUM_PLAUSIBLE_HERTZ` (100 kilohertz, above the 32.768
  kilohertz watch crystal and far below the slowest real counter) and `MAXIMUM_PLAUSIBLE_HERTZ` (100
  gigahertz). The band exists to catch nonsense, not to police hardware, and `is_plausible(hertz)` is
  its one definition, so each architecture's boot path calls the same check.
- The identity does not change: the magic `TIMEBAS1`, `VERSION` 1, `INTERFACE_ID` 9, the 16-byte layout
  above and both fixed addresses. A layout break ships as a new magic and version together.

## EXAMPLES

```rust
use counter_frequency_contract::{CounterFrequencyPage, build_page, is_plausible};

assert!(is_plausible(62_500_000));
assert!(!is_plausible(1)); // firmware wrote nonsense, and `> 0` would have believed it

let bytes = build_page(1_000_000_000);
// SAFETY: `bytes` is a live buffer of `PAGE_CONTENT_BYTES` for the whole of this block.
let page = unsafe { CounterFrequencyPage::new(bytes.as_ptr() as u64) };
assert_eq!(page.hertz(), Some(1_000_000_000));
```

## What is proven

Two Kani harnesses cover the crate. `hertz_answers_none_exactly_when_the_magic_differs` proves, over
every 16-byte frame, that `hertz` answers `None` exactly when the first eight bytes differ from
`MAGIC`, and otherwise the little-endian `u64` of the next eight. `a_built_page_reads_back_its_rate`
proves that `build_page(x).hertz()` is `Some(x)` for every `u64`. Each has a falsification patch in
`falsifications/` that turns it red. No proof covers the caller's promise that the address names a
mapped page, because Kani cannot see a mapping. Whether the page address lies above every address a
consumer places programs at is the consumer's check, because this crate knows nothing of any
address-space layout.

## Building and testing

```sh
cargo test
rustup target add x86_64-unknown-none aarch64-unknown-none-softfloat riscv64imac-unknown-none-elf
cargo build --lib --target x86_64-unknown-none   # also aarch64-unknown-none-softfloat, riscv64imac-unknown-none-elf
```

Run `rustup target add` once, before the first bare-metal build. The tests drive both halves of the
page without a kernel. The end-to-end test, which compares the number a user process reads with the
one the kernel measured, lives with the kernel that maps the page.

### Running the proofs

Install Kani 0.68.0, the version CI uses, with `cargo install --locked kani-verifier --version
0.68.0`, then `cargo kani setup`, which downloads a model checker. Plain `cargo kani` runs both
harnesses in about 12 seconds. To run one:

```sh
cargo kani --harness verification::a_built_page_reads_back_its_rate --exact
```

A passing run prints lines that look alarming and are expected. Among them are `warning: use of an
unstable feature`, a `trailing semicolon in macro` warning on the `const` assertion, and a list of
unsupported constructs. Some checks read `UNREACHABLE`, in a summary such as `0 of 99 failed (5
unreachable)`.
Trust the last line: `Complete - 2 successfully verified harnesses, 0 failures, 2 total.`

### Running one falsification patch

```sh
git apply falsifications/verification.a_built_page_reads_back_its_rate.patch
cargo kani --harness verification::a_built_page_reads_back_its_rate --exact   # ends VERIFICATION:- FAILED
git checkout -- src
```

The harness name is the part of the patch's file name after `verification.`, and each patch turns
its own harness red.

## BUGS

- The page carries one number for the life of a process, so the contract cannot express a counter
  whose rate changes after the writer writes the page. The writer establishes that its counter keeps one
  rate before it writes the number.
- A process with no mapping at the fixed address faults when it reads the page, by design.
- A process reads the rate once, so a later, better number cannot reach it.

## Minimum Rust version

Rust 1.88. To check it:

```sh
rustup toolchain install 1.88 --profile minimal --target x86_64-unknown-none
cargo +1.88 test
cargo +1.88 build --lib --target x86_64-unknown-none
```

## License

Licensed under either of MIT (LICENSE-MIT) or Apache-2.0 (LICENSE-APACHE), at your option.
