# Termux / aarch64 builds

This fork can build a **statically linked aarch64 (arm64) binary** of vaultwarden
that runs on [Termux](https://termux.dev) (Android) as well as on any other
musl/aarch64 Linux.

The workflow lives in [`.github/workflows/termux-build.yml`](.github/workflows/termux-build.yml)
and publishes each build as a GitHub release tagged `termux-<version>`.

## Getting a binary

* Open the **Releases** page of this fork and download
  `vaultwarden-<version>-aarch64-unknown-linux-musl`.
* Or build it yourself: **Actions -> "Build Termux aarch64 binary" -> Run workflow**.
  Leave the version field empty to build the newest upstream release.

### Install in Termux

```sh
pkg install curl
VERSION=1.37.3   # the version you downloaded
curl -LO "https://github.com/majinfanhua/vaultwarden/releases/download/termux-${VERSION}/vaultwarden-${VERSION}-aarch64-unknown-linux-musl"
chmod +x vaultwarden-*-aarch64-unknown-linux-musl
mkdir -p ~/.local/bin
mv vaultwarden-*-aarch64-unknown-linux-musl ~/.local/bin/vaultwarden
~/.local/bin/vaultwarden --version
```

## How the automatic build works

| Trigger | What is built |
|---|---|
| `schedule` (daily 03:00 UTC) | Newest upstream release, **only if** there is no `termux-<version>` release yet |
| `workflow_dispatch` (manual) | The version you type in, or the newest upstream release; source can be upstream or this fork |
| `push` to `main` / `termux-build` | This fork at the pushed commit (version `<newest tag>-<short sha>`) |
| `push` of a `termux-*` tag | This fork at that tag, e.g. `termux-1.37.3` -> release `termux-1.37.3` |

Releases use the `termux-` prefix so they never collide with the upstream
vaultwarden tags in this fork.

To force a fresh build of a version that was already released, delete that
release (or run the workflow manually with **publish release = false** and pick
the artifact from the run summary).

## Why cargo-zigbuild

The binary must be static (`--features sqlite,vendored_openssl`) but a plain
`aarch64-linux-musl-gcc` build is **refused by the Android loader** with

```
CANNOT LINK EXECUTABLE "...": cannot execute: required file not found
```

because Android's own loader is what actually loads the binary, and it needs the
NDK `libc.so` (the syscall stub library that Bionic expects). `cargo-zigbuild`
uses zig as the linker, and zig ships those Android stub libraries (the same
trick the Flutter/Go/Deno Android builds use), so the resulting static-PIE
aarch64 binary starts correctly under Android.

The build also checks for a few things that are known to break on Android:

* the file must be `ARM aarch64` and `static`;
* a warning is emitted when the binary references `__register_frame_info`,
  a libgcc symbol that some Android versions do not export.

## Building locally (same steps as CI)

```sh
rustup target add aarch64-unknown-linux-musl
cargo install cargo-zigbuild --locked
pip install ziglang   # or: python3 -m venv venv && venv/bin/pip install ziglang

export RUSTFLAGS="-C target-feature=+crt-static"
cargo zigbuild --release --locked \
  --target aarch64-unknown-linux-musl \
  --features sqlite,vendored_openssl
```

The result is at `target/aarch64-unknown-linux-musl/release/vaultwarden`.
