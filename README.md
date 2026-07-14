# BusyBox

> A compact toolbox of Unix utilities in a single executable.

BusyBox provides small implementations of commands such as `cp`, `sed`,
`tar`, `init`, and `ash`. The binary selects an applet from the name used to
invoke it, making BusyBox a practical foundation for embedded Linux systems,
initramfs images, rescue media, installers, and containers.

This fork also explores an incremental, opt-in migration of selected applets
from C to Rust. The established C implementation remains the compatibility
baseline.

## At a glance

- **One binary, many commands** — install applets as links or invoke them
  through `busybox`.
- **Small by design** — select only the applets and features a target needs.
- **Highly portable** — suitable for constrained and purpose-built Linux
  environments.
- **Incremental Rust support** — experimental Rust applets stay behind
  explicit configuration options.

## Contents

- [Build](#build)
- [Use BusyBox](#use-busybox)
- [Install](#install)
- [Configure](#configure)
- [Test](#test)
- [Rust migration](#rust-migration)
- [Repository layout](#repository-layout)
- [Contribute](#contribute)
- [Project resources](#project-resources)

## Build

Local development in this repository uses the build container defined by
`Dockerfile.build` so that toolchains and dependencies remain reproducible.

```sh
docker build -f Dockerfile.build -t busybox-build .
docker run --rm -v "$PWD:/work/src" -w /work/src busybox-build \
  sh -lc 'make defconfig && make -j"$(nproc)"'
```

Check the resulting binary from the same image:

```sh
docker run --rm -v "$PWD:/work/src" -w /work/src busybox-build \
  ./busybox --help
```

For detailed build and installation notes, see [`INSTALL`](INSTALL). Available
configuration and build targets are listed by `make help` inside the container.

## Use BusyBox

Invoke an applet through the BusyBox binary:

```sh
./busybox ls -l /proc
./busybox cat LICENSE
```

BusyBox can also choose an applet from its executable name. For example, a
link named `ls` that points to the BusyBox binary runs the BusyBox `ls` applet.
Running `./busybox` without an applet name lists all applets compiled into the
current binary.

## Install

Install the binary and its configured applet links from the build container:

```sh
docker run --rm -v "$PWD:/work/src" -w /work/src busybox-build \
  sh -lc 'make CONFIG_PREFIX=/work/src/_install install'
```

`CONFIG_PREFIX` selects the installation root. Depending on the configuration,
the install target creates symbolic or hard links to the BusyBox binary. The
generated `busybox.links` file lists the enabled applets and their paths.

## Configure

The selected configuration is stored in `.config`. Common starting points are:

| Target | Purpose |
| --- | --- |
| `defconfig` | Broad, general-purpose configuration |
| `allnoconfig` | Minimal baseline for selecting only required features |
| `allyesconfig` | Nearly all features, mainly for broad build coverage |
| `randconfig` | Randomized configuration for build testing |
| `oldconfig` | Update an existing configuration for a newer source tree |

Run a target through the build container, for example:

```sh
docker run --rm -it -v "$PWD:/work/src" -w /work/src busybox-build \
  make menuconfig
```

Use `make config` in the container when the menu interface is unavailable.
Additional target-specific defaults live in [`configs/`](configs/).

## Test

Run the regular test suite in the build container:

```sh
docker run --rm -v "$PWD:/work/src" -w /work/src busybox-build \
  sh -lc 'make check'
```

When `CONFIG_UNIT_TEST` is enabled, unit tests are compiled as an applet and
can be run with `./busybox unit`. See
[`docs/unit-tests.txt`](docs/unit-tests.txt) for guidance on writing and
running unit tests.

## Rust migration

The Rust work in this fork is intentionally measurable and reversible:

1. Preserve the C build, Kconfig, Kbuild, applet metadata, and test suite as
   the baseline.
2. Enable Rust applets only through explicit opt-in configuration.
3. Compare behavior with the corresponding C implementation.
4. Keep design decisions and verification evidence close to the code for
   human review.

See the [Rust migration guide](docs/rust-migration.md) for scope, architecture,
and acceptance criteria, and the
[migration tracker](docs/rust-migration-tracking.md) for current progress.

## Repository layout

| Path | Contents |
| --- | --- |
| `applets/` | Applet registration and generated metadata |
| `archival/`, `coreutils/`, `networking/`, `shell/`, … | Applet implementations grouped by domain |
| `include/` | Shared headers |
| `libbb/` | Shared BusyBox support library |
| `rust/` | Experimental Rust workspace for opt-in migrations |
| `scripts/` | Build, configuration, and maintenance helpers |
| `testsuite/` | Functional tests |
| `docs/` | Project documentation, design notes, and guides |

## Contribute

Before making a substantial change, read:

- [`docs/contributing.txt`](docs/contributing.txt)
- [`docs/style-guide.txt`](docs/style-guide.txt)
- [`docs/new-applet-HOWTO.txt`](docs/new-applet-HOWTO.txt) when adding an applet

Useful bug reports and patches include the BusyBox version or commit, relevant
`.config` settings, platform and toolchain details, a minimal reproduction, and
the expected behavior. Discuss large applet changes on the BusyBox mailing list
before investing in a large patch.

## Project resources

- [BusyBox home page](https://busybox.net/)
- [Downloads](https://busybox.net/downloads/)
- [Source browser](https://git.busybox.net/busybox/)
- [Source access](https://busybox.net/source.html)
- [Mailing list archives](https://lists.busybox.net/pipermail/busybox/)
- [Bug tracker](https://bugs.busybox.net/)

Questions, bug reports, and patches can be sent to
[`busybox@busybox.net`](mailto:busybox@busybox.net).

## License

BusyBox is distributed under GPL-2.0-only. See [`LICENSE`](LICENSE) for the
complete license text.
