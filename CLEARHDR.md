# ClearHDR Mode Implementation for IMX678

## Overview

ClearHDR (WDMODE=0x10) is Sony's on-chip HDR mode for the IMX678 sensor. It applies
a gradation compression curve internally, producing a single 12-bit RAW frame with
extended dynamic range (~100dB vs ~72dB linear). The MIPI output format is identical
to linear mode — no VI/NVCSI/ISP pipeline changes required.

## What Was Changed

### Driver source (`source/nvidia-oot/drivers/media/i2c/`)

**fr_imx678_mode_tbls.h:**
- Added ClearHDR gradation compression register defines (CCMP1/2_EXP, ACMP1/2_EXP at 0x36E4-0x36EE)
- Added `mode_clearhdr_4k[]` register sequence — 3856x2180 ClearHDR (WDMODE=0x10, VMAX=4500 → 30fps)
- Added `mode_clearhdr_binning[]` register sequence — 1928x1090 ClearHDR with 2x2 binning (60fps)
- Added `IMX678_MODE_CLEARHDR_4K` and `IMX678_MODE_CLEARHDR_BINNING` enum entries
- Added mode_table and frmfmt entries

**fr_imx678.c:**
- Added `imx678_is_clearhdr_mode()` helper
- Added ClearHDR + RAW10 incompatibility check (ClearHDR requires RAW12)
- Added gradation compression curve write in `imx678_set_mode()`:
  - CCMP1=500, ACMP1=0x02, CCMP2=11500, ACMP2=0x06
  - Values cleared to 0 when switching back to linear/DOL modes

### Device tree overlays (`source/hardware/nvidia/t23x/nv-public/overlay/`)

**tegra234-p3767-camera-p3768-fr_imx678-cam0-2lane-overlay.dts:**
- mode10: ClearHDR 4K all-pixel 15fps (2-lane bandwidth limited)
- mode11: ClearHDR 1080p binning 30fps (2-lane)

**tegra234-p3767-camera-p3768-fr_imx678-cam1-4lane-overlay.dts:**
- mode10: ClearHDR 4K all-pixel 30fps (4-lane)
- mode11: ClearHDR 1080p binning 60fps (4-lane)

### ISP override (`isp/`)

**IMX678_IRC650_ClearHDR.isp** (new file):
- Copy of linear ISP with ClearHDR-specific documentation
- Notes on gamma/tone curve adjustments needed for gradation-compressed input
- Needs real-world tuning after testing on hardware

## IMPORTANT: Device Tree Caveat

The p3768 overlay DTS files modified above are for the **NVIDIA reference carrier board**.
The Jetson currently runs a **CTI monolithic DTB**:

    tegra234-orin-nx-cti-NGX024-FSM-IMX678-2CAM.dts

The ClearHDR DT mode entries need to be added to the CTI DTB source for the new modes
to appear in `v4l2-ctl --list-formats-ext`. The kernel module changes (mode tables,
register sequences, gradation curve) are board-independent and will work regardless.

## Frame Rate Reference

| Mode | Resolution | Lanes | Max FPS |
|------|-----------|-------|---------|
| ClearHDR 4K | 3856x2180 | 4 | 30 |
| ClearHDR 4K | 3856x2180 | 2 | ~15 |
| ClearHDR 1080p binning | 1928x1090 | 4 | 60 |
| ClearHDR 1080p binning | 1928x1090 | 2 | 30 |

Formula: `fps = 74,250,000 / (VMAX x HMAX)` where ClearHDR VMAX=4500, HMAX=550 at 891MHz link.

## Gradation Curve Source

Register values (CCMP1/2, ACMP1/2) were obtained from the community IMX678 driver
(RPi port of will127534/imx585-v4l2-driver, with ClearHDR implementation).
See `imx678-v4l2-driver-reference/ABOUT.md` for details.

## Testing

```bash
# Build and deploy to Jetson
./scripts/jetson-build-driver.sh --all

# On Jetson, verify new modes appear
v4l2-ctl --list-formats-ext -d /dev/video0

# Switch ISP to ClearHDR variant
sudo ln -sf /var/nvidia/nvcam/settings/IMX678_IRC650_ClearHDR.isp \
            /var/nvidia/nvcam/settings/camera_overrides.isp

# Test capture (mode index may vary)
gst-launch-1.0 nvarguscamerasrc sensor-id=0 sensor-mode=<clearhdr_mode_index> \
    ! 'video/x-raw(memory:NVMM),width=3856,height=2180,framerate=30/1' \
    ! nvvidconv ! xvimagesink
```

## Future Work

- [ ] Add ClearHDR mode entries to CTI DTB (tegra234-orin-nx-cti-NGX024-FSM-IMX678-2CAM)
- [ ] Tune ISP for gradation-compressed input (gamma, AE target, optical black)
- [ ] Add ClearHDR gain controls (CHDR_DGAIN0_HG, CHDR_AGAIN0_LG/HG) as V4L2 controls
- [ ] Add V4L2_CID_WIDE_DYNAMIC_RANGE toggle for runtime linear/ClearHDR switching
