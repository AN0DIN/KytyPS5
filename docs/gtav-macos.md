# GTA V on Apple Silicon

Measured results of GTA V (PS5, PPSA04263 01.010.002) on this branch, on one Mac. Not a support promise: other
Macs, macOS versions and game versions were not tested.

## Setup

| | |
|---|---|
| Machine | Apple M5, 32 GB, macOS 26 |
| Emulator | this branch, x86_64 build under Rosetta 2 |
| Vulkan | MoltenVK 1.4.2 (Khronos release) |
| Game | PPSA04263 01.010.002, Performance mode |
| Output | 1920x1080, FSR 1.0 at scale 1.0 |

The game needs a patch that moves its fixed memory mappings to `0x7000000000` and above. On Apple Silicon,
`0x1000000000`-`0x7000000000` is reserved by the system, and without the patch the game exits after about 1.5 s.
The 1080p output, the FSR factor and Performance mode (no ray tracing) are also set by game patches.

The first time the menu appears with an empty cache, Metal compiles shaders for 1.5 to 3.5 minutes. The game is
not frozen during that time.

## Frame rate

Each place is measured with a fixed scripted camera (same view in every run): 10 s to settle, then the median of
5-second fps samples over 30 s.
"Before" = the previous tip of this branch (one run); "after" = this branch with the five changes below (mean of
two runs). Two runs of the same build differ by up to 2.6 fps at 30-50 fps.

| Place | Before (fps) | After (fps) |
|---|---|---|
| Downtown, night / day | 20.2 / 20.4 | 40.5 / 40.6 |
| Strawberry | 20.0 | 36.3 |
| Beach | 20.0 | 33.5 |
| Route 68 | 8.2 | 33.8 |
| Sandy Shores, night / day | 8.8 / 10.0 | 32.9 / 36.9 |
| Grapeseed | 8.6 | 31.0 |
| Paleto Bay, night / day | 16.6 / 16.5 | 38.2 / 41.0 |
| Mount Chiliad | 13.0 | 51.6 |

Below about 15 fps the game's simulation runs slower than real time (0.54-0.66 of real time at 8-10 fps), so
slow-motion video at low fps is expected.

Driving (AI-driven car from Route 68, 30 min, 24.5 km), measured on the development build these changes come
from: median 33.7 fps, minimum 24.8 fps, no crash or freeze.

## What changed

Five commits on top of the previous tip. Effects were measured one at a time during development, each on top of
the previous ones: the three GPU changes at Sandy Shores by switching them on and off within one launch (A B B A,
then B A A B, game camera), the two shader changes between launches with the fixed camera (A B B A).

| Change | Effect |
|---|---|
| Read-backs run on the graphics queue when there is no transfer-only queue family (MoltenVK) | +20-24 % at Sandy Shores |
| Mapped guest buffers are read back directly on an integrated GPU (`KYTY_READBACK_DIRECT=0` turns it off) | +26-31 % at Sandy Shores |
| Indirect draws read their arguments on the GPU when `drawIndirectCount` is missing (MoltenVK) | +44-49 % at Sandy Shores, +43 % while driving in the desert |
| Native round-toward-zero half conversion (same bits as before) | +4-6 % |
| Fused legacy MAD in pixel and compute shaders (macOS) | +65-75 % (pixel), +3-8 % (compute) |

## Known limits

- Memory grows while driving through new areas: resident memory 8.4 -> 11.9 GB in 30 min, with a slope that
  goes down (+140, +100, +80 MB/min per 10 min). A one-hour session has not been measured yet.
- The GPU is no longer saturated everywhere (86-97 % busy in the runs above). At Sandy Shores the emulator's five
  GPU threads each keep a CPU core 94-100 % busy, so they are likely the next limit.
- Fused MADs may differ from the console in the last bit in pixel and compute shaders; vertex shaders and
  depth-exporting pixel shaders stay exact.
