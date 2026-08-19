# CKB On-Chain Script Guide
This guide helps AI agents write on-chain scripts on CKB.

## Quickstart
On a fresh machine, install the prerequisites in this order:
1. `cargo-generate` >= 0.16.0: `cargo install cargo-generate` (required by `make generate`).
2. The RISC-V Rust target: `make prepare` (runs `rustup target add riscv64imac-unknown-none-elf`).
3. Clang: used to build C code. If it is missing on the target machine, ask the user to install it.

Then the typical flow is:
```
make generate CRATE=on-chain-script-1
make build
make test
```

## New on-chain script
Use the following command to create a new script:
```
make generate CRATE=on-chain-script-1
```
The new script will be placed in the `contracts` folder.

`make generate` does more than generating the crate: it also appends the template's test cases to `tests/src/tests.rs` and inserts the new crate into the workspace `Cargo.toml` members list. Do not repeat these two edits by hand.

`TEMPLATE` defaults to `contract`. Other available templates:
- `contract-without-simulator`
- `standalone-contract` (a self-contained project, not tied to a workspace)
- `atomics-contract`
- `stack-reorder-contract` (supports adjusting the stack size, see the Memory section)

Example: `make generate CRATE=my-script TEMPLATE=atomics-contract`.

For native debugging there is also `make generate-native-simulator CRATE=<name>`, which generates a `<name>-sim` crate under `native-simulators/`. This, together with `make coverage`, is the intended use of the `native-simulator` feature; do not enable it for on-chain deployment builds unless explicitly requested.

## Toolchain and target
All crates under `contracts` require `no_std` and are built for the Rust target `riscv64imac-unknown-none-elf` with `-C target-feature=+zba,+zbb,+zbc,+zbs` (see the contract Makefile). This corresponds to ckb-vm's rv64imcb ISA: rv64imac plus the B-extension bit-manipulation subsets enabled by those target features. All crates under `tests` target the native platform and support `std`.

Pin a concrete Rust toolchain version in a `rust-toolchain.toml` at the project root — the workspace template does not ship one, and the templates use edition 2024, which requires a recent toolchain. A good reference is the ckb repository's toolchain file (https://github.com/nervosnetwork/ckb/blob/develop/rust-toolchain.toml, currently channel `1.95.0`); copy a pinned version, e.g. from a ckb release tag, instead of tracking the moving develop branch.

Clang might be used in the toolchain. If it is missing on the target machine, ask the user to install it.

## Memory
The on-chain script runs on [ckb-vm](https://github.com/nervosnetwork/ckb-vm), a RISC-V 64-bit virtual machine, with a total memory of 4M.
There is no need to write a linker script. The default memory layout is sufficient.

With ckb-std's default_alloc! , the heap is a static buffer that lives inside .bss of the ELF — there is no distinct heap area owned by the VM.

By default, it use following configuration:
```
// * 16KB fixed heap
// * 1.2MB(rounded up to be 16-byte aligned) dynamic heap
// * Minimal memory block in dynamic heap is 64 bytes
ckb_std::default_alloc!(16384, 1258306, 64);
```
More information for [buddy-alloc](https://github.com/jjyr/buddy-alloc)

If a script runs out of stack, generate it from the `stack-reorder-contract` template and adjust the stack size with:
```
make run CONTRACT=<name> TASK=adjust_stack_size STACK_SIZE=0x200000
```

## ckb-std
This is a must-use crate for development. Its source code is available at https://github.com/nervosnetwork/ckb-std

It provides the following features:
- ckb syscalls (high-level wrapper)
- argc/argv
- log
- since
- Type ID


Detailed information about syscalls:
- https://github.com/nervosnetwork/rfcs/blob/master/rfcs/0009-vm-syscalls/0009-vm-syscalls.md
- https://github.com/nervosnetwork/rfcs/blob/master/rfcs/0034-vm-syscalls-2/0034-vm-syscalls-2.md
- https://github.com/nervosnetwork/rfcs/blob/master/rfcs/0050-vm-syscalls-3/0050-vm-syscalls-3.md

When using syscalls, first use the `high_level` module. If it doesn't meet your requirements, use the `syscalls` module instead.
When using syscalls, use the predefined types in `ckb-types`, such as Script, WitnessArgs, Transaction, CellInput, CellOutput, etc, instead of parsing molecule from scratch.

The `ckb-types`, `allocator`, and `calc-hash` features are sufficient for most cases. Try them first, then consider adding others. Don't use the `native-simulator` feature unless explicitly requested.

IMPORTANT: Overview of transaction structure:
https://github.com/nervosnetwork/rfcs/blob/master/rfcs/0022-transaction-structure/0022-transaction-structure.md


## Rust
The default language for writing on-chain scripts is Rust. 

IMPORTANT: Unless the user requests otherwise, don't use C. If C is used, refer to [ckb-c-stdlib](https://github.com/nervosnetwork/ckb-c-stdlib) and the [guide](https://github.com/nervosnetwork/ckb-c-stdlib/blob/master/guide.md).


## Log
`ckb_std::debug!` works out of the box: it goes through the debug syscall and needs no feature flag.

If the project wants the standard `log` crate facade (`log::info!` etc.), enable ckb-std's `log` feature (it is disabled by default):
```
ckb-std = { version = "1.1", features = ["log"] }
```

## Molecule
All data structures in transactions (e.g. cell data, witness) should use [molecule](https://github.com/nervosnetwork/molecule). See the related [spec](https://github.com/nervosnetwork/rfcs/blob/master/rfcs/0008-serialization/0008-serialization.md).

By default, generate Rust code via `moleculec` and commit the generated files to git. Install it with `cargo install moleculec`; for reproducible codegen, pin the moleculec version (e.g. `cargo install moleculec --locked --version <x.y.z>`). Keep schema (`.mol`) files in a conventional location such as a `schemas/` directory at the project root. Users can explicitly request compiling it on the fly with [build.rs](https://github.com/nervosnetwork/molecule/blob/master/examples/ci-tests/build.rs).

Don't parse molecule from scratch: use existing types in `ckb-types` or results from `moleculec`. Ignore all clippy warnings on generated rust files from `moleculec`.

Users can explicitly request a different serialization system (e.g. serde).

## Binary size
When a new crate is introduced, make sure it supports `no_std`. Also diff the binary size before and after (e.g. `ls -l build/release/`), and report to users if the size exceeds 100K. Warn users if the final binary size exceed 400K. The limit of binary size is about 500K, because the script binary is published on-chain as the data of a dep cell, and is therefore bounded by transaction and block serialization size limits.

## Writing Tests
Use [ckb-testtool](https://github.com/nervosnetwork/ckb-testtool) to write unit tests. It is important to include failure test case.

`verify_tx` from `ckb-testtool` is enough for most cases. The template also ships a `verify_and_dump_failed_tx` helper in `tests/src/lib.rs` (it is not part of ckb-testtool); don't use it unless explicitly requested.

Besides memory, cycles are the other scarce on-chain resource: keep an eye on the cycles consumed by tests and report unusually large consumption to users.

## Tests
After changes, always do following to verify:
```
make build
make test
```

The workspace Makefile also provides: `make clippy`, `make fmt`, `make checksum` (reproducible-build checksums under `build/`), and `make coverage` (coverage reports based on native simulators, requires `make coverage-install` first).


## Crates
Crates used in on-chain scripts run in a `no_std` environment, so they should use special features.

For each `version = "..."` below, look up the latest published version with `cargo search <crate>` instead of guessing, and note that the resolved version should be double-checked.

- When using blake2b, add `ckb-hash` with the `ckb-contract` feature enabled:
```
ckb-hash = { version = "...", default-features = false, features = ["ckb-contract"]}
```

- When using `sparse-merkle-tree`,  add the `with-blake2b-ref` and `smtc` features:
```
sparse-merkle-tree = { version = "...", default-features = false, features = ["with-blake2b-ref", "smtc"] }
```

- When using `ckb-types`, set `default-features` to false. Note the package is published as `ckb-gen-types` (generated molecule types) and renamed to `ckb-types` so the code can keep using the conventional crate name:
```
ckb-types = { package = "ckb-gen-types", version = "...", default-features = false }
```

In unit tests, you are free to use other features.
