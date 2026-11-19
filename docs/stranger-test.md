# The stranger cadence

The release's whole claim is that a stranger, given this repository and nothing else, can build
the crate and run its proofs from the README alone. A real agent runs that check on a
cadence, and the weekly `stranger cadence` job fails when the
last run is past 28 days.

What a run is:

1. Clone the repository fresh.
2. Follow the README and nothing else: build, test, and run the proofs.
   The README must name the build and test commands (with any feature a test target needs), the
   bare-metal build with `rustup target add`, the Kani install before the first `cargo kani`, how
   long the proofs take and how to run one harness, what a passing run prints that looks alarming,
   how to run and revert one falsification patch, and how to check the minimum Rust version.
   Each one it lacks is a finding.
3. Note every place the README lied, omitted, or assumed. Those are findings against the README,
   not against the stranger.
4. Record the date on the line below.

Last run: 2026-11-19
- The README gives no `rustup target add` line for the bare-metal build; it only lists target names. The run machine already had them, so the build succeeded, but a stranger would not know the step.
- The README never says how to install Kani or that `cargo kani` is the command; the run guessed `cargo kani`, and it worked (two harnesses, about 12 seconds in total). It gives no timing and no command for one harness.
- The README does not say what a passing proof run prints that looks alarming (for example "5 unreachable" checks in the summary).
- The README says falsification patches exist but not how to apply and revert one; the run inferred `git apply falsifications/<patch>`, then `cargo kani` (one failure, as expected), then `git checkout src` from the patch format, and that worked.
- The README says "Rust 1.88" but gives no command to check it; the run inferred `cargo +1.88 build` and ran only `cargo +1.88 --version`, so it did not verify the minimum version itself from the README.
- Passed as documented: `cargo test` (7 unit tests, 4 doctests) and the three README-named `--target` builds (the run built only x86_64).
