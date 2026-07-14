# Rust cross-compile matrix

Rust-enabled cross builds require an explicit target triple. BusyBox `ARCH`
and `CROSS_COMPILE` are insufficient because they do not uniquely identify the
libc ABI. The supported invocation shape is:

```sh
make ARCH=<arch> CROSS_COMPILE=<prefix> \
  RUST_TARGET=<triple> RUST_TARGET_LINKER=<linker> busybox
```

If `CROSS_COMPILE` is set without `RUST_TARGET`, Kbuild stops before compiling.
Native builds continue to omit `--target` and use the pinned host toolchain.

## Candidate mappings

| BusyBox ARCH | glibc Rust target | musl Rust target | status |
| --- | --- | --- | --- |
| `x86_64` | `x86_64-unknown-linux-gnu` | `x86_64-unknown-linux-musl` | candidate |
| `i386` | `i686-unknown-linux-gnu` | `i686-unknown-linux-musl` | candidate |
| `arm` (v7 hard-float) | `armv7-unknown-linux-gnueabihf` | `armv7-unknown-linux-musleabihf` | candidate |
| `arm64` | `aarch64-unknown-linux-gnu` | `aarch64-unknown-linux-musl` | validated as described below |
| `riscv64` | `riscv64gc-unknown-linux-gnu` | `riscv64gc-unknown-linux-musl` | candidate |

Mappings are not automatic. A toolchain prefix may target a different ARM
float ABI, CPU baseline, endianness, or libc. Each candidate becomes supported
only after the matching C sysroot/linker and a final BusyBox smoke test pass.

## Validation snapshot (2026-07-14)

| environment | result | scope |
| --- | --- | --- |
| `aarch64-unknown-linux-gnu` on Debian bookworm arm64 | pass | Rust unit tests, release static archive, Rust-enabled BusyBox link, applet comparison |
| `aarch64-unknown-linux-musl` on Debian bookworm arm64 | pass | pinned Rust toolchain installs the target and builds the release static archive |
| complete musl BusyBox link | blocked | requires a matching musl C compiler/sysroot in the project build image |

The musl archive validation proves that the Rust crate and target are
available; it does not claim a complete mixed C/Rust BusyBox until the matching
musl C link is exercised. This is an explicit toolchain-image gap rather than
falling back to a glibc or host archive.

## Known gaps

- uClibc, Android/Bionic, FDPIC, NOMMU, big-endian, and targets without a
  tiered Rust `std` distribution are unsupported.
- `RUST_TARGET_LINKER` must address the same sysroot and libc as
  `CROSS_COMPILE`; Kbuild cannot verify this mechanically.
- Target-specific CPU flags are not translated from CFLAGS. Add matching
  `RUSTFLAGS=-C target-cpu=...` only after binary compatibility is checked.
- Cross-built applets still require execution under native hardware or an
  explicitly configured emulator for behavior tests.
