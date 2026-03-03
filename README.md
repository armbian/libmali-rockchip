# libmali-rockchip

Mali GPU userspace driver binaries for Rockchip SoCs, with a meson build system that wraps them with standard library interfaces (GBM, EGL, GLESv1, GLESv2, Wayland EGL, OpenCL, Vulkan).

## Supported SoCs

| GPU | SoCs |
|-----|------|
| valhall-g610 | rk3588 |
| bifrost-g52 | rk3566, rk3568, rk3562, rk3576 |
| bifrost-g31 | px30, rk3326 |
| midgard-t86x | rk3399, rk3399pro |
| midgard-t76x | rk3288 |
| utgard-450 | rk3328, rk3528 |
| utgard-400 | px3se, rk3036, rk3032, rk312x, rk3128h |

## Dependencies

```bash
sudo apt install meson pkg-config \
  libdrm-dev libx11-dev libx11-xcb-dev \
  libxcb-dri2-0-dev libwayland-dev
```

## Building

### Configure

```bash
meson setup build \
  -Dgpu=valhall-g610 \
  -Dversion=g24p0 \
  -Dplatform=x11-wayland-gbm \
  -Darch=aarch64
```

### Build

```bash
ninja -C build
```

### Install to staging directory (for packaging)

```bash
DESTDIR=$(pwd)/pkg ninja -C build install
```

### Install to system

```bash
sudo ninja -C build install
```

## Meson Options

| Option | Default | Description |
|--------|---------|-------------|
| `arch` | auto | Target architecture: `aarch64`, `arm`, `armhf`, etc. |
| `gpu` | midgard-t86x | GPU name (see table above) |
| `version` | r18p0 | Driver version (e.g. `g24p0`, `g29p1`, `r18p0`) |
| `subversion` | none | Driver subversion, or `all` to match any |
| `platform` | gbm | Display platform(s): `gbm`, `x11-gbm`, `wayland-gbm`, `x11-wayland-gbm`, `dummy`, etc. |
| `optimize-level` | O3 | Optimization level: `O0`–`O3`, `Os`, `Ofast`, `Og` |
| `hooks` | true | Enable GBM hook library for extended format support |
| `wrappers` | auto | Build wrapper shared libraries (EGL, GLESv2, etc.) |
| `vendor-package` | false | Install wrappers to `lib/mali/` vendor subdirectory |
| `opencl-icd` | true | Install OpenCL as ICD (`libMaliOpenCL`) instead of `libOpenCL` |
| `wayland-egl` | true | Install `libwayland-egl` wrapper |
| `firmware-dir` | /lib/firmware | Destination for GPU firmware (valhall-g610 only) |
| `with-overlay` | false | Install init overlay scripts (utgard/midgard only) |
| `khr-header` | false | Install `khrplatform.h` header |

## Available Library Variants

The library filename encodes the configuration:

```
libmali-<gpu>-<version>[-<subversion>]-<platform>.so
```

Examples:
- `libmali-valhall-g610-g29p1-x11-wayland-gbm.so` — G610, g29p1, X11 + Wayland + GBM
- `libmali-valhall-g610-g24p0-dummy.so` — G610, g24p0, no display (headless/compute)
- `libmali-bifrost-g52-g24p0-wayland-gbm.so` — G52, g24p0, Wayland + GBM

Binaries live under `lib/<multiarch>/` and `optimize_3/<multiarch>/` (O3-optimized, preferred).

## Building a .deb Package

### Single package (manual)

```bash
# 1. Configure and build
meson setup build -Dgpu=valhall-g610 -Dversion=g29p1 -Dplatform=x11-wayland-gbm -Darch=aarch64
ninja -C build

# 2. Stage the install
DESTDIR=$(pwd)/build/pkg ninja -C build install

# 3. Write control file
mkdir -p build/pkg/DEBIAN
cat > build/pkg/DEBIAN/control << EOF
Package: libmali-valhall-g610-g29p1-x11-wayland-gbm
Version: 29
Architecture: arm64
Maintainer: Your Name <you@example.com>
Provides: libmali
Conflicts: libmali
Replaces: libmali
Depends: libdrm2, libwayland-client0, libwayland-server0, libx11-6, libxcb1, libxcb-dri2-0
Description: Mali GPU User-Space Binary Drivers
 Mali Valhall G610 g29p1 userspace driver with X11, Wayland and GBM support.
EOF

# 4. Build the .deb
dpkg-deb --build build/pkg libmali-valhall-g610-g29p1-x11-wayland-gbm_29_arm64.deb
```

### All packages via debhelper

```bash
# Regenerate debian/targets and debian/control from all libs in lib/
scripts/update_debian.sh

# Build all binary packages
dpkg-buildpackage -b --no-sign
```

To add a new library blob to the debian packaging:

1. Place the `.so` in `lib/<multiarch>/` (and optionally `optimize_3/<multiarch>/`)
2. Filename must follow the convention: `libmali-<gpu>-<version>-<platform>.so`
3. Run `scripts/update_debian.sh` to regenerate `debian/targets` and `debian/control`
4. Build with `dpkg-buildpackage -b --no-sign`

## Adding a New Driver Blob

1. Copy the `.so` into `lib/aarch64-linux-gnu/` (rename to strip any arch suffix)
2. Optionally also place it in `optimize_3/aarch64-linux-gnu/` for the O3 build path
3. Run `scripts/update_debian.sh` to register it in `debian/targets` and `debian/control`
4. Build the deb as shown above
