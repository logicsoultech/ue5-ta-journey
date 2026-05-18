# TROUBLESHOOTING.md

A running log of bugs encountered, hypotheses tested, and solutions found during this learning journey. Maintained as a professional record — both for personal reference and to demonstrate debugging methodology to future employers.

---

## Format

Each entry follows this structure:

```
### [Short title] — UE version · Week · Status
**Symptom:** What was observed
**Hypothesis:** What I thought was causing it
**Tests run:** What I tried (in order)
**Root cause:** What was actually wrong
**Fix:** Exact steps or commands
**Lesson:** What this teaches about the engine
```

---

## Bug log

---

### Shadow and viewport inconsistency — UE 5.7 · Week 1 · Resolved (engine downgrade)

**Symptom:** Viewport shadows, visual effects, and overall scene appearance did not match the tutorial reference. Results were inconsistent regardless of settings adjusted.

**Hypothesis:** Project configuration mismatch or hardware-specific rendering path difference.

**Tests run:**
- Adjusted FPS limit settings
- Changed scalability levels (Low → Epic)
- Manually enabled and disabled ray tracing
- Compared multiple viewport modes

**Root cause:** UE 5.7 introduced rendering pipeline changes that caused visual inconsistencies with tutorial projects authored in earlier engine versions. The scene (Bad Decision Studio) was designed and validated against UE 5.4.

**Fix:** Downgraded from UE 5.7 → UE 5.4. Rebuilt project from scratch in 5.4.

**Lesson:** When following a tutorial project, match the engine version the author used before troubleshooting visual discrepancies. Engine version mismatch is always the first thing to rule out.

---

### Paid decal material (yellow lane lines) — Week 1 · Resolved (material workaround)

**Symptom:** Tutorial required a paid decal asset (yellow line markings) on ArtStation/Marketplace. No budget available.

**Hypothesis:** The color comes from the asset itself — not necessarily.

**Tests run:**
- Inspected what the paid asset actually provided: a white texture mask + color defined in material nodes.

**Root cause:** Not a bug. The asset's color value was configurable via material graph — the white texture was the actual "shape" data; color was a node parameter.

**Fix:**
1. Found a free white decal texture with the correct shape.
2. In Material Editor: added a `Multiply` node between the texture sample and the Base Color output.
3. Connected a `Constant3Vector` (set to yellow: R=1.0, G=0.85, B=0.0) as the second input of the Multiply node.
4. Result: identical visual output to the paid asset.

**Lesson:** In UE5 Material Editor, color is almost never "in" the texture — it is calculated in the graph. Understanding this node relationship (texture × color constant = final color) is fundamental material authoring. This is exactly how professional material functions work.

---

### CineCamera auto-focus not working when attached to camera rig — UE 5.4 · Week 1 · Workaround (partial)

**Symptom:** At tutorial Part 16/19, a CineCamera attached to a camera rig did not auto-focus on the target object when the rig moved. Focus tracking was inactive.

**Hypothesis:** Focus tracking requires the camera to be "active" — simply placing it in the rig is not sufficient.

**Tests run:**
- Checked CineCamera focus settings (Draw Debug Focus Plane, Focus Method set to Tracking)
- Verified the focus actor target was set correctly

**Root cause:** UE 5.4 requires the CineCamera to be initialized as the active camera at least once before tracking becomes live in the editor viewport.

**Fix (workaround):**
1. Select the CineCamera actor in the Outliner.
2. Use the "Pilot" option (right-click → Pilot) to enter the camera's perspective.
3. Press Play, then stop, then exit the camera.
4. Auto-focus tracking now works correctly when the rig moves.

**Known side effect:** A ghosting/shadow artifact appears on objects after this workaround. See next entry for fix.

**Lesson:** UE's sequencer and camera system has initialization states that are not always obvious from the UI. When a camera behavior doesn't activate, "pilot and play once" is a reliable state initializer.

---

### Ghost shadow / motion blur artifact on Stormtroopers — UE 5.4 · Week 1 · Resolved

**Symptom:** After the CineCamera workaround above, Stormtrooper characters displayed a persistent ghost shadow — a faint duplicate image trailing behind moving objects, even when the camera was still.

**Initial hypothesis:** DLSS and Ray Tracing running simultaneously, causing temporal reprojection conflict.

**Output log evidence:**
```
LogDLSSNGXRHI: Destroying NGX DLSS Feature
SrcRect=[0x0->1367x873], DestRect=[0x0->2038x1302]
ScaleX=0.670756, ScaleY=0.670507
NGXDLSSPreset=Default(0)
NGXPerfQuality=MaxQuality(2)
bUseAutoExposure=1
bEnableAlphaUpscaling=1
```

**Tests run (in order, all ineffective):**
1. Disabled Anti-Aliasing (TAA → None) — no change
2. Set monitor percentage to 100% — no change
3. Disabled DLSS via Project Settings — no change
4. Adjusted all DLSS-related parameters — no change
5. Toggled Ray Tracing on/off — no change

**Root cause:** UE 5.4-specific bug in the Shadow Denoiser system. In this version, the denoiser responsible for cleaning shadow noise instead introduces a "ghost" artifact on moving objects — the previous frame's shadow is not correctly discarded.

**Fix:**
```
Console command (temporary, per session):
~ → r.Shadow.Denoiser 0 → Enter

Permanent fix (add to Config/DefaultEngine.ini):
[SystemSettings]
r.Shadow.Denoiser=0
```

**Lesson:** Not all rendering artifacts are caused by the most obvious system (in this case DLSS). Reading the output log gave a clue (DLSS was active) but was ultimately a red herring. Systematic elimination — one variable at a time — is the correct process. The Shadow Denoiser is a separate system from DLSS and TAA; understanding that UE's rendering pipeline has many parallel denoising passes is important for optimization work later.

---

## Config patches applied

| File | Setting | Value | Reason |
|---|---|---|---|
| `Config/DefaultEngine.ini` | `r.Shadow.Denoiser` | `0` | UE 5.4 ghost shadow bug |

---

## Baseline measurements (Week 1)

Scene: Bad Decision Studio (BDS) — UE 5.4

| Metric | Value | Notes |
|---|---|---|
| FPS (editor viewport) | 120 | Not representative of shipped build |
| Frame time | 8.33ms | Editor overhead included |
| Shader Complexity max | 2000 (MaxShaderComplexityCount) | Pink/magenta areas exceed this |
| Total actors | 175 | |
| Hottest area | Left section (magenta) | Likely translucency overdraw |
| Platform floor | Red | Complex layered material |
| Stormtroopers | Green | Efficient shaders |

> **Note:** These numbers are editor viewport measurements only. Packaged build performance will differ significantly. This baseline exists to track relative improvement as optimization techniques are applied in Phase 3.
