# ckb-script-templates

This repository keeps a series of CKB script templates that can be inflated via [cargo-generate](https://github.com/cargo-generate/cargo-generate). Those templates enable a native development flow on mainstream Linux, macOS and Windows machines using stock version of latest stable Rust & clang C compiler.

## Noticeable Changes

### Molecule uses bytes crates

 [molecule](https://github.com/nervosnetwork/molecule) starting from 0.8.0, switches to [bytes](https://crates.io/crates/bytes) instead of `Vec` internally to keep data in `no_std` environment. However, bytes would require atomic builtins so as to function, this could lead to unsupported CKB-VM instructions (RISC-V A extension instructions to be precise) being generated. There are 2 ways to solve this issue:

* Enable `dummy-atomic` feature in `ckb-std` crate, also use make sure `FULL_RUSTFLAGS` in the contract makefile is updated so `-a` is included, for example: `FULL_RUSTFLAGS := -C target-feature=+zba,+zbb,+zbc,+zbs,-a $(CUSTOM_RUSTFLAGS)`. Or see [this PR](https://github.com/cryptape/ckb-script-templates/pull/17) for how to change `RUSTFLAGS`.
* Enable `bytes_vec` feature in `molecule` crate

[ckb-gen-types](https://crates.io/crates/ckb-gen-types) starting from 0.117.0 is also affected, since `ckb-gen-types` has upgraded to `molecule` 0.8.0 in this version.

## Usage

### Dependencies

The following dependencies are required for the templates:

* `git`, `make`, `sed`, `bash`, `shasum` and others Unix utilities. Refer to the documentation for your operating systems for how to install them. Chances are your system might already have them.
* `Rust`: latest stable Rust installed via [rustup](https://rustup.rs/) should work. Make sure you have `riscv64` target installed via: `rustup target add riscv64imac-unknown-none-elf`
* `Clang`: make sure you have clang 18+ installed, sample installation steps for selected platforms are:
    + Debian / Ubuntu: `wget https://apt.llvm.org/llvm.sh && chmod +x llvm.sh && sudo ./llvm.sh 18 && rm llvm.sh`
    + Fedora 39+: `sudo dnf -y install clang`
    + Archlinux: `sudo pacman --noconfirm -Syu clang`
    + macOS: `brew install llvm@18`
    + Windows(with [Scoop](scoop install llvm yasm)): `scoop install llvm yasm`
* `cargo-generate`: You can install this via `cargo install cargo-generate`, or follow the steps [here](https://cargo-generate.github.io/cargo-generate/installation.html)

### Creating workspace

To generate a workspace template, use the following command:

```
$ cargo generate gh:cryptape/ckb-script-templates workspace
⚠️   Favorite `gh:cryptape/ckb-script-templates` not found in config, using it as a git repository: https://github.com/cryptape/ckb-script-templates.git
🤷   Project Name: my-first-contract-workspace
🔧   Destination: /tmp/my-first-contract-workspace ...
🔧   project-name: my-first-contract-workspace ...
🔧   Generating template ...
🔧   Moving generated files into: `/tmp/my-first-contract-workspace`...
🔧   Initializing a fresh Git repository
✨   Done! New project created /tmp/my-first-contract-workspace
```

Or you can manually specify the name and skip the prompt:

```
$ cargo generate gh:cryptape/ckb-script-templates workspace --name my-first-contract-workspace
⚠️   Favorite `gh:cryptape/ckb-script-templates` not found in config, using it as a git repository: https://github.com/cryptape/ckb-script-templates.git
🔧   Destination: /tmp/my-first-contract-workspace ...
🔧   project-name: my-first-contract-workspace ...
🔧   Generating template ...
🔧   Moving generated files into: `/tmp/my-first-contract-workspace`...
🔧   Initializing a fresh Git repository
✨   Done! New project created /tmp/my-first-contract-workspace
```

This is probably the only longer command you will deal with when using the templates repository. You can save them as an alias in your shell:

```
$ alias create-ckb-scripts="cargo generate gh:cryptape/ckb-script-templates workspace"
$ create-ckb-scripts
⚠️   Favorite `gh:cryptape/ckb-script-templates` not found in config, using it as a git repository: https://github.com/cryptape/ckb-script-templates.git
🤷   Project Name: my-first-contract-workspace
🔧   Destination: /tmp/my-first-contract-workspace ...
🔧   project-name: my-first-contract-workspace ...
🔧   Generating template ...
🔧   Moving generated files into: `/tmp/my-first-contract-workspace`...
🔧   Initializing a fresh Git repository
✨   Done! New project created /tmp/my-first-contract-workspace
```

### Generating contract crates in the workspace

First, navigate to the workspace directory you just created:

```
$ cd my-first-contract-workspace
```

First thing you can notice, is that we have created a standard Rust workspace project:

```
$ tree .
.
├── Cargo.toml
├── Makefile
├── scripts
│   └── find_clang
└── tests
    ├── Cargo.toml
    └── src
        ├── lib.rs
        └── tests.rs

4 directories, 6 files
```

The only exception here, is that a `Makefile`, together with a `scripts` folder has been created to simplify contract building.

We can use `make generate` to create our first contract:

```
$ make generate
🤷   Project Name: first-contract
🔧   Destination: /tmp/my-first-contract-workspace/contracts/first-contract ...
🔧   project-name: first-contract ...
🔧   Generating template ...
🔧   Moving generated files into: `/tmp/my-first-contract-workspace/contracts/first-contract`...
🔧   Initializing a fresh Git repository
✨   Done! New project created /tmp/my-first-contract-workspace/contracts/first-contract
```

You can also supply the contract create name when executing the make task:

```
$ make generate CRATE=second-contract
🔧   Destination: /tmp/my-first-contract-workspace/contracts/second-contract ...
🔧   project-name: second-contract ...
🔧   Generating template ...
🔧   Moving generated files into: `/tmp/my-first-contract-workspace/contracts/second-contract`...
🔧   Initializing a fresh Git repository
✨   Done! New project created /tmp/my-first-contract-workspace/contracts/second-contract
```

By default, the newly created crate is using [contract](https://github.com/cryptape/ckb-script-templates/tree/main/contract) template, which is put into `contracts` sub-folder, the workspace-level `Makefile` assumes all Rust contracts are stored in `contracts` folder, and treat crates stored in other folders as dependency-only Rust crates.

But chances are you would want to tweak the default settings in certain scenarios:

* Sometimes you want to use a different template as a contract starting point other than `contract` template.
* Sometimes you want to use a different template since you are generating a plain Rust crate, which will also likely be put in a different subfolder other than `contracts`
* **Note**: while you could technically put a Rust contract in a different subfolder other than `contracts`, we don't really recommend this, since the workspace-level `Makefile` is leveraging the convention that all CKB contracts live in `contracts` folder.

You can do this by customizing parameters to `make generate`:

```
$ make generate TEMPLATE=atomics-contract                      # generate a Rust contract crate in contracts subfolder, but use atomics-contract template
$ make DESTINATION=crates                                      # generate a crate in crates folder, and still use the default contract template
$ make generate TEMPLATE=c-wrapper-crate DESTINATION=crates    # generate a crate in crates folder, but use c-wrapper-crate template
```

Ready-to-use templates have been put together for different use cases:

* `contract`: default contract template you should use if no special requirements are needed.
* `stack-reorder-contract`: a contract template that adjusts memory layout so stack lives at lower address, and heap lives at higher address. This way a program would explicitly signal errors when stack space is fully use.
* `c-wrapper-crate`: a crate template that shows how to glue C code in a Rust crate for CKB's contract usage.
* `x64-simulator-crate`: a crate template that contains Rust-only code, but uses [ckb-x64-simulator](https://github.com/nervosnetwork/ckb-x64-simulator) for tests.

There are also deprecated templates kept for historical reasons.

* `atomics-contract`: a contract template that supports atomic builtins without requiring RISC-V A extension. This template allows you to use `log`, `bytes` crate or other code that might deal with atomics. Note that starting from ckb-std v0.16.0, atomic builtins are provided in ckb-std by default. For Rust based scripts, relying the builtin implementation in ckb-std might be better and more smooth idea. We are still keeping this template for now as a hint for a C based solution. It might be removed in future versions.

Certain template might require external modules to be available, for example:

* All C code would require [ckb-c-stdlib](https://github.com/nervosnetwork/ckb-c-stdlib): `git submodule add https://github.com/nervosnetwork/ckb-c-stdlib deps/ckb-c-stdlib`
* `atomics-contract` requires [lib-dummy-atomics](https://github.com/xxuejie/lib-dummy-atomics): `git submodule add https://github.com/xxuejie/lib-dummy-atomics deps/lib-dummy-atomics`
* `stack-reorder-contract` requires [ckb-stack-reorg-bootloader](https://github.com/xxuejie/ckb-stack-reorg-bootloader): `git submodule add https://github.com/xxuejie/ckb-stack-reorg-bootloader deps/ckb-stack-reorg-bootloader`

In future versions, we might leverage [cargo-generate hooks](https://cargo-generate.github.io/cargo-generate/templates/scripting.html) to add submodules automatically, but for now, manual steps are required.

### Build & Test

Note that when you supply the contract crate name, our `Makefile` will be smart enough to automatically insert the crate in to `Cargo.toml` file, so you don't have to edit it manually.

Now you can build the contracts(or adjust parameters):

```
$ make build
$ make build MODE=debug                   # for debug build
$ make build CUSTOM_RUSTFLAGS=""          # release build without debug assertions
$ make build CARGO_ARGS="--verbose"       # release build with `--verbose` attached to cargo command, you can use other arguments accepted by cargo
$ make build CONTRACT=second-contract     # build a single contract
$ make build CLEAN_BUILD_DIR_FIRST=false  # keep old untouched binaries
$ make build CLANG=clang-17               # use a specific clang version to build C code
```

We have prepared a `tests` crate where you can also add contract tests. If you want to run the tests, run the following command:

```
make test
```

The templates provided here, use the same conventions as `ckb-native-build-sample` project, so feel free to refer to the more detailed [usage](https://github.com/xxuejie/ckb-native-build-sample?tab=readme-ov-file#usage) doc in the sample project.

### Reproducible Build

When using this set of templates, we always recommend to use locally installed native versions of LLVM & Rust to build and test your scripts. However, reproducible build is an important part of CKB scripts, which would require locked versions of LLVM & Rust to work. It might not be an easy task when using locally installed versions of compilers.

For the time being, we have prepared a script that does reproducible build via [a docker container image](https://github.com/nervosnetwork/llvm-n-rust). We do want to mention that docker is not necessarily THE way to do reproducible build, nor is it the best way to do reproducible build. There might well be other ways that are better, such as chroot or Nix. It's just that historically, docker has been used in CKB script's build process, and adding a script leveraging docker here, provides an easy solution into the issue.

To do reproducible build, you can use the included script with varying commands:

```
$ ./scripts/reproducible_build_docker               # Clean current repository, used locked LLVM & Rust from a docker container
                                                    # to build all contracts, then test the binaries against a checksum file.

$ ./scripts/reproducible_build_docker --update      # Update the checksum file with new binaries, could be handy when you have
                                                    # made changes to the binaries.

$ ./scripts/reproducible_build_docker --no-clean    # Do not clean intermediate files before building, it is not recommended to
                                                    # use this but when you really know what you are doing, it could help you save
                                                    # some time.

$ ./scripts/reproducible_build_docker --proxy "..." # Setup docker container so it pulls Rust crates using a proxy server
```

By default, the checksum file is stored in `checksums.txt` in the root of the repository. It is strongly recommended that this file is checked into version control, and a CI is setup so reproducible build is always checked in new commits.

### Standalone Contract Crate

In rare cases if you want to simply use a standalone contract crate without a workspace. The [standalone-contract](https://github.com/cryptape/ckb-script-templates/tree/main/standalone-contract) template is prepared for you:

```
$ cargo generate gh:cryptape/ckb-script-templates standalone-contract
⚠️   Favorite `gh:cryptape/ckb-script-templates` not found in config, using it as a git repository: https://github.com/cryptape/ckb-script-templates.git
🤷   Project Name: standalone-first-contract
🔧   Destination: /tmp/standalone-first-contract ...
🔧   project-name: standalone-first-contract ...
🔧   Generating template ...
🔧   Moving generated files into: `/tmp/standalone-first-contract`...
🔧   Initializing a fresh Git repository
✨   Done! New project created /tmp/standalone-first-contract
```

You can then build and test the standalone crate as usual:

```
$ cd standalone-first-contract
$ make build
$ make test
$ make check
$ make clippy
```

The template is tailored built for usage outside of workspace, typically, it is not expected to be used inside a workspace. Feel free to compare it with the default `contract` workspace for differences.

This standalone template also has its own test setup, where in a workspace, a dedicated `tests` crate will handle most of the testing work.

### Native Simulator debug

The `generate-native-simulator` command in the `Makefile` generates a native simulator. It **requires** the `CRATE` parameter to specify an existing subproject. If the parameter is missing or invalid, the command will fail.

```bash
make generate-native-simulator CRATE=<subproject_name>
```

To generate a simulator for the `example_crate` subproject:

```bash
make generate-native-simulator CRATE=example_crate
```

- The `CRATE` parameter must refer to a subproject.
- Missing subprojects will cause the command to fail.

### Code Coverage

The templates include built-in support for code coverage reporting using LLVM's coverage tools.

**Requirements:**
- Only contracts created with the `contract` template support coverage (pure Rust contracts with `native-simulator` feature)
- Other templates (`atomics-contract`, `stack-reorder-contract`, `contract-without-simulator`, `standalone-contract`) do not support coverage
- Native simulators must be generated for each contract you want coverage for
- Only works on x86_64 Linux (not ARM)

**Setup:**

1. Install llvm-tools:
```bash
make coverage-install
```

2. Generate native simulators for your contracts:
```bash
make generate-native-simulator CRATE=<contract_name>
```

**Generate Reports:**
```bash
make coverage        # Text report to console
make coverage-html   # HTML report in target/coverage/html/
make coverage-lcov   # LCOV format for CI integration
```

**Example:**
```bash
# Create a contract using the default 'contract' template
make generate CRATE=my-contract

# Generate native simulator for it
make generate-native-simulator CRATE=my-contract

# Install tools and run coverage
make coverage-install
make coverage
```

Coverage works by running tests with the `native-simulator` feature, which executes contract code natively on x86_64 instead of in CKB-VM. This allows LLVM's coverage instrumentation to track which lines of code are executed.
