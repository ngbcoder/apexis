# Apexis image build

`coreos-assembler` build definition. This turns the design tokens in `design/`
into a bootable Apexis image.

## What is and is not verified

**Verified here:**
- `coreos-assembler` build structure — `manifest.yaml`, `image.yaml`, `overlay.d/`
  (documented at <https://coreos.github.io/coreos-assembler/working>)
- The branding obligations in `plan.md` §3.1 — dropping `fedora-logos`, replacing
  the OS identity, removing Fedora imagery
- The design token values themselves (CI-enforced contrast validation)

**Resolved by the first build (was unverified):**

- `coreos-assembler` *does* run under WSL2 — `/dev/kvm` is present and writable,
  systemd is PID 1, and podman passes KVM through to containers.
- The config structure is valid: `cosa init` located both `manifest.yaml` and
  `image.yaml`.
- `cosa fetch` no longer exists; `build-with-buildah` is the default and fetches
  as part of `build`.

**Still unconfirmed:**

- the base `ref:` for the Fedora Plasma atomic desktop
- the exact `ostree:` keys accepted in `image.yaml`
- the Plasma layout mechanism (see *Plasma layout* below)
- package names for the default app set

Treat the first `cosa build` as the thing that resolves these. Expect to fix
things here rather than be surprised by them.

## Prerequisites

- Linux host with hardware virtualisation (`/dev/kvm` accessible)
- Podman
- Roughly 30 GB free disk

`coreos-assembler` is a container. You are running Fedora tooling inside a
Fedora tooling container; the host OS only needs Podman and KVM.

## Build

```sh
./build.sh
```

Or by hand, if you want to see what each step does:

```sh
podman pull quay.io/coreos-assembler/coreos-assembler

# init the workdir against the local config. Note: `cosa init <path>` treats its
# argument as a clone target, which fails because build/ is not empty — the
# documented local pattern is COREOS_ASSEMBLER_CONFIG_GIT plus /dev/null.
podman run --rm -it --privileged \
  -e COREOS_ASSEMBLER_CONFIG_GIT=/srv/src/config \
  -v "$PWD/.build:/srv:Z" \
  -v "$PWD/build:/srv/src/config:Z" \
  --entrypoint /usr/bin/cosa \
  quay.io/coreos-assembler/coreos-assembler:latest \
  init --force /dev/null

# `cosa fetch` is gone — build-with-buildah is the default and fetches as part
# of the build. Build directly.
podman run --rm -it --privileged \
  -e COREOS_ASSEMBLER_CONFIG_GIT=/srv/src/config \
  -v "$PWD/.build:/srv:Z" \
  -v "$PWD/build:/srv/src/config:Z" \
  --entrypoint /usr/bin/cosa \
  quay.io/coreos-assembler/coreos-assembler:latest \
  build

# boot the resulting image in QEMU
podman run --rm -it --privileged \
  -e COREOS_ASSEMBLER_CONFIG_GIT=/srv/src/config \
  -v "$PWD/.build:/srv:Z" \
  -v "$PWD/build:/srv/src/config:Z" \
  --entrypoint /usr/bin/cosa \
  quay.io/coreos-assembler/coreos-assembler:latest \
  run
```

`cosa` derives the build version by executing `build/versionary`, which emits a
string like `0.1.0.dev.20261011.0`. That becomes the directory name under
`.build/builds/`. It reads `build/build-args.conf` for `VERSION`.

The design tokens are staged into `overlay.d/` by `build.sh` before the build,
so the image and the tokens can never drift apart. CI regenerates tokens and
fails on staleness; `build.sh` copies them in.

## Layout

```
build/
├── build.sh              stages design tokens, then invokes cosa
├── manifest.yaml         OSTree commit definition (packages)
├── image.yaml            disk image output
└── overlay.d/            merged into the OSTree commit, lexicographically
    ├── 00-branding/      remove Fedora identity (trademark obligation)
    ├── 10-plasma/        desktop layout and effects
    └── 20-apexis/        generated design tokens land here
```

`overlay.d/` subdirectories are applied in lexicographic order, which is why
they are numerically prefixed.

## Plasma layout — open

Design Doc §7.1 describes a centred launcher, a floating vertical dock and no
full-width top bar. That is a *shell layout*, not a theme, and §6.4 assumed a
curated desktop could deliver it. It largely can, via Plasma configuration, but
the mechanism for shipping **default** panel layout in an image is not settled
here. Candidates:

- a Plasma 6 layout template in `/usr/share/plasma/layout-templates/apexis/`
- per-user defaults shipped in `/etc/skel/.config/plasma-org.kde.plasma.desktop-appletsrc`

The first is cleaner and more idiomatic; the second is what distributions
historically did and is more predictable. This needs a real Plasma session to
test, so it is deliberately not written speculatively.

## Naming

The image ships as `apexis`, with `osname` set accordingly. Apexis is **not** a
Fedora Spin and carries no Fedora branding — see `plan.md` §3.1. Removal of
`fedora-logos` is a legal obligation, not a stylistic choice.