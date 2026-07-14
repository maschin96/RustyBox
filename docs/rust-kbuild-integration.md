# Rust Kbuild integration

This document records the build design for linking the experimental Rust
static library into BusyBox. The integration remains opt-in through
`FEATURE_RUST_APPLETS`; a C-only configuration does not invoke Cargo and does
not add Rust objects to the final link.

## Host and target separation

Kconfig and the normal BusyBox host tools continue to run with the host
toolchain. Cargo is invoked only for the target archive required by
`busybox_unstripped`:

```text
Kconfig / generated headers (host tools)
                |
                +-- C objects (CC/CROSS_COMPILE) --------+
                |                                         |
                +-- Rust staticlib (Cargo target) --------+--> final target link
```

The archive is built from `rust/Cargo.toml` with the pinned toolchain and
`panic=abort`. `CARGO_TARGET_DIR=$(objtree)/rust/target` keeps generated Rust
artifacts in the BusyBox output tree. This is important for both `O=` builds
and read-only source trees.

The final link remains owned by Kbuild. Cargo produces only
`libbusybox_rs.a`; it does not link a second executable or choose the target C
linker.

## Architecture and target mapping

BusyBox `ARCH` names are not always Rust target names, and
`CROSS_COMPILE=<prefix>` does not identify a libc ABI. The integration must
therefore use an explicit mapping rather than infer a target from the compiler
prefix alone.

The native path omits `--target`; Cargo then uses the installed host target.
Cross builds must supply a validated Rust target triple and matching linker.
The planned interface is:

```sh
make ARCH=<busybox-arch> CROSS_COMPILE=<tool-prefix> \
  RUST_TARGET=<rust-target-triple> \
  RUST_TARGET_LINKER=<target-linker>
```

The current candidates and validation evidence are recorded in
`rust-cross-compile.md`. A mapping is accepted only after the C compiler target, Rust target,
libc ABI, pointer width, endianness, and final linker have been checked as one
toolchain. Unknown combinations must fail with a clear message instead of
silently producing a host archive.

## Dependency and link ordering

With `FEATURE_RUST_APPLETS=y`, `busybox-all` depends on
`$(objtree)/rust/target/release/libbusybox_rs.a`. The archive is appended to
the normal BusyBox library list so existing C shims and the libbb bridge can
resolve Rust applet entry points during the Kbuild-controlled final link.

The Rust archive must be rebuilt when Rust sources, either Cargo manifest, or
the pinned toolchain file changes. The current conservative `FORCE` dependency
is correct but may be replaced by explicit prerequisites once the source list
is generated reliably.

## Out-of-tree and clean behavior

- In-tree build: Cargo output is written below `rust/target/`.
- `make O=/path ...`: Cargo output is written below
  `/path/rust/target/`; the source tree remains unchanged.
- `make clean`: the Rust target directory in `$(objtree)` must be removed with
  the other generated target artifacts.
- `make mrproper` and `make distclean`: inherit the clean behavior and remove
  configuration generated in the selected object tree.
- A C-only clean build must not require `cargo`, `rustc`, or a Rust target.

## Verification matrix

The integration is considered stable when CI covers all of the following:

| configuration | expected result |
| --- | --- |
| in-tree, `FEATURE_RUST_APPLETS=n` | C-only build succeeds without Cargo |
| in-tree, `FEATURE_RUST_APPLETS=y` | Rust archive is built and linked |
| `O=` build with Rust enabled | archive and BusyBox outputs stay under `O=` |
| clean after Rust build | the object-tree Rust target directory is removed |
| validated glibc cross target | C and Rust objects link and smoke tests run |
| validated musl cross target | C and Rust objects link and smoke tests run |

The first four cases belong to the Kbuild integration. The two cross-target
rows are completed and maintained by the cross-compile matrix in issue #17.
