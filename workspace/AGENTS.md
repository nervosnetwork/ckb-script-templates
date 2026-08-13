# CKB On-Chain Script Guide
This guide helps AI agents write on-chain scripts on CKB.

## New on-chain script
Use the following command to create a new script:
```
make generate CRATE=on-chain-script-1
```
The new script will be placed in the `contracts` folder.

## Toolchain and target
All crates under `contracts` should target RISC-V (rv64imcb) and require `no_std`. All crates under `tests` target the native platform and support `std`.

Always use rust toolchain from: https://github.com/nervosnetwork/ckb/blob/develop/rust-toolchain.toml

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

IMPORTANT: Overview of transaction structure:
https://github.com/nervosnetwork/rfcs/blob/master/rfcs/0022-transaction-structure/0022-transaction-structure.md


## Rust
The default language for writing on-chain scripts is Rust. 

IMPORTANT: Unless the user requests otherwise, don't use C. If C is used, refer to [ckb-c-stdlib](https://github.com/nervosnetwork/ckb-c-stdlib) and the [guide](https://github.com/nervosnetwork/ckb-c-stdlib/blob/master/guide.md).


## Log
It's recommended to add the `enable_log` feature to the project. It is disabled by default. When it is enabled, enable the `enable_log` feature for ckb-std and log important events.

## Molecule
All data structures in transactions (e.g. cell data, witness) should use [molecule](https://github.com/nervosnetwork/molecule). See the related [spec](https://github.com/nervosnetwork/rfcs/blob/master/rfcs/0008-serialization/0008-serialization.md).

By default, generate Rust code via `moleculec` and commit the generated files to git. Users can explicitly request compiling it on the fly with [build.rs](https://github.com/nervosnetwork/molecule/blob/master/examples/ci-tests/build.rs).

Users can explicitly request a different serialization system (e.g. serde).

## Binary size
When a new crate is introduced, make sure it supports `no_std`. Also diff the binary size before and after, and report to users if the size exceeds 100K. Warn users if the final binary size exceed 400K. The limit of binary size is about 500K.

## Writing Tests
Use [ckb-testtool](https://github.com/nervosnetwork/ckb-testtool) to write unit tests. It is important to include failure test case.

## Tests
After changes, always do following to verify:
```
make build
make test
```

## Crates
Crates used in on-chain scripts run in a `no_std` environment, so they should use special features.


- When using blake2b, add `ckb-hash` with the `ckb-contract` feature enabled:
```
ckb-hash = { version = "???", default-features = false, features = ["ckb-contract"]}
```

- When using `sparse-merkle-tree`,  add the `with-blake2b-ref` and `smtc` features:
```
sparse-merkle-tree = { version = "???", default-features = false, features = ["with-blake2b-ref", "smtc"] }
```

In unit tests, you are free to use other features.
