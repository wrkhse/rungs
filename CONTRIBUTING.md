# Contributing

rungs is a single `no_std` crate with no dependencies. CI runs every check in
[`.github/workflows/check.yml`](.github/workflows/check.yml) on each pull
request and each push to `main`.

## Toolchains

`rust-toolchain.toml` pins Rust 1.100.0 with rustfmt, clippy and the
`wasm32-unknown-unknown` target, the same version as `rust-version` in
`Cargo.toml`. Every Cargo command passes `--locked`. Two jobs use a pinned
nightly: `kani-compiler` uses `nightly-2026-08-21`, the compiler of Kani 0.68,
which rejects some APIs stable in 1.100; `miri` uses `nightly-2026-10-12`,
Chauffeur's pinned nightly, so both projects run the same Miri.

## The checks

The `rust` job denies every Cargo warning through `CARGO_BUILD_WARNINGS`, and
installs cargo-nextest 0.9.151:

```sh
export CARGO_BUILD_WARNINGS=deny

cargo fmt --all -- --check
cargo clippy --all-targets --all-features --locked
cargo clippy --target wasm32-unknown-unknown --locked
cargo build --target wasm32-unknown-unknown --locked
cargo nextest run --all-targets --all-features --locked --no-tests=pass
cargo test --doc --all-features --locked
RUSTDOCFLAGS='-D warnings' cargo doc --all-features --no-deps --locked
```

The other jobs each run one tool. `kani` installs kani-verifier 0.68.0 and
`mutants` installs cargo-mutants 27.1.0 and cargo-nextest 0.9.151:

```sh
# kani-compiler
cargo +nightly-2026-08-21 check --all-targets --all-features --locked

# miri
cargo +nightly-2026-10-12 miri test --all-features --locked

# kani
cargo kani setup
cargo kani -j 4 --output-format terse

# mutants, on main
cargo mutants --timeout 120
```

On a pull request, `mutants` runs only on the lines the change touches under
`src`, through `cargo mutants --in-diff`; on `main` it runs on everything.
`.cargo/mutants.toml` lists the mutants it leaves out, each with its reason:
code built only for Kani, and mutants equivalent to the original.

## What the Kani proofs cover

The harnesses in `src/proofs.rs` check eight properties for every plan of one
to four steps, each step with up to three outcomes and three attempts, from
every progress the plan accepts and through every call. Each property has one
harness per plan size, 32 in all:

- `advance` starts a step exactly when its needs hold.
- A run never stalls.
- Without retries or restarts, each pass of `advance` finishes the next level,
  so `depth()` passes finish a run.
- Between two restarts a step starts at most `attempts` times.
- `resume` accepts exactly the progress the plan can reach.
- Every call keeps the progress reachable.
- A call changes only its own step, and only from the status it expects.
- `restart` moves back only the step and the steps that depend on it.

Under Kani a plan has room for four steps and an outcome set is a byte
(`src/narrow.rs`); the rest of the code is the same. Plans of five to 16
steps run the same code; the tests in `tests/` exercise examples of them, and
the proofs do not reach them.
