# Engine Notes — UE5 Concepts

Running notes on UE5 concepts encountered during the learning journey. These are personal working notes — not polished documentation.

---

## Shader Complexity view (`Alt+8`)

**What it shows:** How computationally expensive each pixel is to render, expressed as a color heatmap.

**Color scale:**
| Color | Meaning | Approximate instruction count |
|---|---|---|
| Green | Efficient | Low (< ~300) |
| Yellow | Moderate | Medium |
| Orange/Red | Heavy | High |
| Magenta/Pink | Extremely heavy | Exceeds MaxShaderComplexityCount (default: 2000) |

**Important limitation:** This is an editor viewport tool. Numbers do not represent shipped build performance — they represent relative complexity, useful for identifying problem areas, not for benchmarking.

**BDS baseline (Week 1):**
- Platform floor: Red — suspected multi-layer material
- Left section overhead: Magenta — suspected translucency overdraw
- Stormtroopers: Green — efficient shaders

---

## Overdraw

When the GPU renders the same pixel more than once in a single frame, that is overdraw. Each rendering pass costs shader instructions. Multiple transparent or semi-transparent objects layered on top of each other compound: 3 translucent layers = 3 shader passes for the same pixel, even though only the top layer is ultimately visible.

Magenta in Shader Complexity usually indicates severe overdraw, an extremely expensive material, or both combined.

Common causes in UE5:
- Translucent materials (each translucent surface adds a pass)
- Particle systems with many overlapping sprites
- Post-process effects stacking (bloom + volumetric + SSAO)
- Complex material graphs with many texture samples

---

## Material Editor — color is in the graph

Key insight from Week 1 decal workaround:

A texture provides **shape** (the UV mask). Color is a **graph operation** — typically a `Constant3Vector` multiplied against the texture sample before it reaches Base Color. This means:
- The same white texture can produce any color without buying a new asset.
- Changing color at runtime is possible via Material Parameters.
- Understanding this node relationship is prerequisite to any serious material authoring.

Node pattern for recoloring a white texture:
```
TextureSample (white mask) ──┐
                              ├── Multiply ── Base Color
Constant3Vector (RGB color) ──┘
```

---

## Config/DefaultEngine.ini — what it is

UE5 stores per-project persistent engine settings in `Config/DefaultEngine.ini`. Settings written here apply every time the project is opened, without needing to re-enter console commands.

This file should be committed to Git — it carries project-specific fixes and configuration that all team members (or future machines) need.

Syntax: `SettingName=Value` under the relevant `[SectionName]` header.

Console command equivalent uses a space instead of equals sign — the two formats are not interchangeable.

---

## CineCamera — focus tracking initialization

CineCamera auto-focus tracking in UE 5.4 appears to require the camera to be initialized as the active viewport camera at least once per session. The "pilot + play once" sequence triggers this initialization.

Steps:
1. Right-click CineCamera in Outliner → Pilot
2. Press Play
3. Press Stop
4. Exit pilot mode (Eject or press the camera icon again)

This is a known initialization state issue, not a settings problem.

---

## DLSS vs Shadow Denoiser — separate systems

Lesson from the Week 1 ghost shadow investigation:

DLSS (Deep Learning Super Sampling) and the Shadow Denoiser are **separate rendering passes** in UE5's pipeline. The output log showed DLSS activity, which led to a false hypothesis. Disabling DLSS had no effect because the ghost artifact was produced by the Shadow Denoiser, not by temporal reprojection.

Rendering pipeline passes relevant here (simplified):
```
Geometry → Lighting → Shadow Denoiser → DLSS (upscaling) → Post Process → Output
```

Disabling the wrong pass costs debugging time. The correct approach: disable one system at a time, observe, re-enable, move to the next.

The Shadow Denoiser bug in UE 5.4 causes it to retain shadow data from the previous frame for moving objects, producing a visible ghost. Fix: `r.Shadow.Denoiser=0`.
