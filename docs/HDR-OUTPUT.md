# HDR10 output (opt-in branch)

This branch adds an **HDR10 swapchain and output path** to BOTWRecompAndroid, as an opt-in setting.
Default behaviour is unchanged (SDR, 8-bit).

## Enabling

- In game / start screen menu: **Settings → Driver compatibility → “HDR output (10-bit, HDR10)”** → Automatic / On / Off, then **Restart now**.
- Or edit the ini (`v190/botwrecompandroid.ini`): `hdr = auto | 1 | 0` (default `0`).
- The environment variable `LARECOMP_HDR=1`/`0` still overrides the ini for test runs (e.g. `adb shell am start ... --es env LARECOMP_HDR=1`).

Requirements: a display that accepts HDR10 and a Vulkan driver that offers a 10-bit surface format with
`VK_COLOR_SPACE_HDR10_ST2084_EXT` (the `VK_EXT_swapchain_colorspace` instance extension). When there is no such
surface format the backend logs `HDR10: no 10-bit + ST2084 surface format` and stays SDR; on a driver without the
instance extensions it logs `HDR instance extensions unavailable` and stays SDR.

## What it does

- instance: enables `VK_EXT_swapchain_colorspace` (only when HDR is asked for); device: enables
  `VK_EXT_hdr_metadata` when available.
- swapchain: picks `VK_FORMAT_A2B10G10R10_UNORM_PACK32` (or `R10X6G10X6B10X6A10X6_UNORM_4PACK16`) with
  `VK_COLOR_SPACE_HDR10_ST2084_EXT`, and sets `VkHdrMetadataEXT` (BT.2020 primaries, maxLuminance from
  `LARECOMP_HDR_NITS`, default 420 nits).
- output: `present_hdr.frag` converts the game's SDR output (sRGB) to linear and encodes it as PQ / ST2084 with
  SDR white at 203 nits (`LARECOMP_HDR_SDR_NITS` overrides that reference for experiments).
- Android: the window is set to `ActivityInfo.COLOR_MODE_HDR` when the ini/env asks for HDR.

## Tested on

- **Xiaomi Pad (2410CRP4CC, SM7675 / Adreno 732)**, Android 16, Turnip v26.3.0-R6 through adrenotools:
  `[vk] HDR10 output: format 64, colorspace ST2084, maxLuminance 420 nits, swapchain 3200x2136`,
  `dumpsys display` → `hdrSdrRatio ≈ 2.16`; title screen and world render, 30 fps cap.
- **vivo V2547A (SM8850 / Adreno 840)**, Android 17, Turnip Gen8 through adrenotools:
  `[vk] HDR10 output: format 64, colorspace ST2084, maxLuminance 420 nits, swapchain 3168x1440`,
  `hdrSdrRatio ≈ 5.50`, title screen and Chinese menus render, 30 fps cap, no crash.

## Notes / limitations

- The game's own tonemap is designed for SDR; this puts that SDR image in an HDR10 container (10-bit precision,
  wide-gamut container). It is not a re-graded HDR render.
- Screenshots/`screencap` of the HDR output look washed out: the capture converts PQ to SDR. What the display shows
  is what matters.
- This branch is not a pull request: it is kept on the fork for anyone who wants to try the feature or pick parts of it.
  The `gpu_heap = Automatic` fix for Turnip is separate (PR #103 upstream).
