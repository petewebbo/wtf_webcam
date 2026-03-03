# AMD ISP4 Webcam on EndeavourOS / Arch Linux
> HP ZBook Ultra G1a (Ryzen AI Max) — kernel 6.18+


## Overview

The built-in webcam uses AMD's ISP4 (Image Signal Processor), not a standard USB camera. It requires:
1. A kernel module (`amd_capture`) — out-of-tree, built from upstream patch series
2. A patched libcamera with AMD ISP4 pipeline support

---

## Prerequisites

```bash
sudo pacman -S base-devel linux-headers b4 v4l-utils media-ctl
```

---

## Step 1: Build the kernel module

```bash
git clone https://github.com/sisou/amd-isp4-camera
cd amd-isp4-camera
```

Fix the `sign-file` path for Arch (it's in `/usr/lib/modules`, not `/usr/src/kernels`):

```bash
sed -i 's|/usr/src/kernels/$(KVER)/scripts/sign-file|/usr/lib/modules/$(KVER)/build/scripts/sign-file|' Makefile
```

If Secure Boot is **disabled**, skip signing entirely. Edit the `install` target in `Makefile` to remove the `sign` dependency, then build:

```bash
make setup   # downloads v8 patch series via b4
make -C linux-$(uname -r) build
```

Load dependencies and the module:

```bash
sudo modprobe videodev
sudo modprobe videobuf2_v4l2
sudo modprobe videobuf2_vmalloc
sudo modprobe amd_isp4
sudo insmod linux-$(uname -r)/amd_capture.ko
```

Verify the camera is exposed:

```bash
v4l2-ctl --list-devices
# Should show: amd_isp_capture (platform:amd_isp_capture): /dev/video0
```

Install permanently:

```bash
sudo install -Dm644 linux-$(uname -r)/amd_capture.ko \
  /usr/lib/modules/$(uname -r)/extra/amd_capture.ko
sudo depmod -a

echo -e "videodev\nvideobuf2_v4l2\nvideobuf2_vmalloc\namd_isp4\namd_capture" \
  | sudo tee /etc/modules-load.d/amd-camera.conf
```

> **Note:** Rebuild `amd_capture` after every kernel update by running `make -C linux-$(uname -r) build` again.

---

## Step 2: Build AMD's libcamera fork

The standard `libcamera` package does not include the AMD ISP4 pipeline. Build AMD's fork:

```bash
git clone https://github.com/amd/Linux_ISP_libcamera
cd Linux_ISP_libcamera
```

Configure (the `-Dcpp_args` flag works around missing `#include` issues with GCC 15):

```bash
meson setup build \
  -Dpipelines=uvcvideo,simple,amd/isp4 \
  -Dipas=amd/isp4 \
  -Dcpp_args="-include cstdint -include stdint.h"
```

Patch the pipeline to match the single entity exposed by the v8 kernel driver:

```bash
# The kernel driver only exposes "Preview", not "Video" and "Still"
sed -i 's/static const std::vector<std::string> entityNames{ "Preview", "Video", "Still" };/static const std::vector<std::string> entityNames{ "Preview" };/' \
  src/libcamera/pipeline/amd/isp4/isp4.cpp

sed -i 's/{ supportedEntityNames().at(1), StreamRole::VideoRecording },//' \
  src/libcamera/pipeline/amd/isp4/isp4.cpp

sed -i 's/{ supportedEntityNames().at(2), StreamRole::StillCapture }//' \
  src/libcamera/pipeline/amd/isp4/isp4.cpp
```

Build and install:

```bash
ninja -C build
sudo ninja -C build install
sudo ldconfig
```

Make sure `/usr/local/bin` is in your PATH (add to `~/.zshrc` or `~/.bashrc`):

```bash
export PATH=/usr/local/bin:$PATH
```

---

## Step 3: Verify

```bash
cam --list
# Should show: 1: (ov05c_mipi0)

cam -c 1 -I
# Lists supported resolutions up to 2888x1808 in NV12 and YUYV
```

---

## Troubleshooting

**`amd_isp4` module loads but camera not detected:**
The mainlined `amd_isp4` kernel stub conflicts with `amdgpu` claiming the ISP block. The out-of-tree `amd_capture` module from this guide sidesteps that issue — do not attempt to use `ip_block_mask` to disable amdgpu's ISP, as this causes a boot loop.

**`cam --list` shows no cameras despite module loaded:**
Check that you're running `/usr/local/bin/cam` (AMD fork) and not `/usr/bin/cam` (system libcamera). Verify with `ldd $(which cam) | grep libcamera` — it should show `libcamera.so.0.2`.

**IPA warning (`IPA not functional yet`):**
Expected. The IPA (auto-exposure/white-balance algorithms) is not yet implemented in the AMD fork. The `cam` CLI tool will hang on capture as a result, but the camera works normally in real applications (tested with deepin-camera and Microsoft Teams).
