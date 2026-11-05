# DLSS 4.5 via Streamline — Integration Playbook

> **Audience:** an autonomous coding agent (e.g. Claude Code) working inside a game's repository, plus the engineer supervising it.
> **Goal:** assess any game codebase, then plan and implement **NVIDIA Streamline** with **DLSS Super Resolution (DLSS-SR / DLAA)** and **DLSS Frame Generation (DLSS-G, including Multi Frame Generation)**, targeting **DX12 or Vulkan**.
> **Out of scope:** DLSS 5 neural rendering. As of Nov 2026 its Streamline plugin isn't in the public SDK and access goes through NVIDIA. This playbook leaves the pipeline ready for it (see §10).

---

## 0. Ground rules for the agent

1. **The SDK docs are the source of truth, not this playbook.** Function names, struct fields and enum values change between Streamline releases. Before writing code, pin the SDK version (§3.1) and read these from it:
   - `docs/ProgrammingGuide.md`
   - `docs/ProgrammingGuideDLSS.md`
   - `docs/ProgrammingGuideDLSS_G.md`
   - `docs/ProgrammingGuideReflex.md` / PCL guide
   - `docs/ProgrammingGuideManualHooking.md` (if the engine wraps D3D/Vulkan itself)

   Where this playbook disagrees with the SDK, the SDK wins. Record the discrepancy in the report.
2. **Assess first, change nothing.** Phase 1 (§2) is read-only. Produce the assessment report and get human sign-off before touching code.
3. **Small, reversible steps.** Put each feature behind a runtime toggle and a compile-time flag (e.g. `ENABLE_STREAMLINE`). The game must build and run with Streamline absent or disabled, and on non-NVIDIA GPUs.
4. **Never ship community injectors** (OptiScaler, DLL swaps, proxy hacks). This is an official integration only.
5. **If the engine is Unreal or Unity,** stop and use the official NVIDIA plugin (§1). Don't hand-roll Streamline into a stock commercial engine.

---

## 1. Decision tree: which path?

```
Is the game on Unreal Engine 5?
 ├─ Yes → Use NVIDIA's DLSS UE plugin matching the engine version (developer.nvidia.com/dlss).
 │        Integration = enable plugins + project settings + UI. Skip to §7 (UI) and §8 (verification).
 │        If the UE build is heavily modified, also check the NvRTX branch.
Is it on Unity?
 ├─ Yes → Use Unity's built-in DLSS support (HDRP/URP, version-dependent) or NVIDIA's package.
 │        Check what your Unity version supports for Frame Generation before promising it.
Custom / in-house / other engine?
 └─ Yes → Streamline integration (this playbook, §2 onwards).
          Graphics API?
           ├─ D3D12 → Preferred. Full SR + FG support. Interposer or manual hooking.
           ├─ Vulkan → Supported. FG has extra queue/extension requirements; read the Vulkan sections carefully.
           ├─ D3D11 → SR works. FG does NOT work on D3D11. Recommend porting the renderer, or ship SR only.
           └─ OpenGL / Metal / other → Not supported. Report and stop.
```

---

## 2. Phase 1 — Codebase assessment (read-only)

Work through every item. For each one, record **file paths, function names and line references** as evidence. Mark anything you can't determine as `UNKNOWN`, with the question to ask a human.

### 2.1 Project & build
- [ ] Engine name/version, or "custom". Find engine headers, `CMakeLists.txt`, `.uproject`, `ProjectSettings/`, premake/meson/bazel files.
- [ ] Build system and how third-party SDKs are vendored (`external/`, `third_party/`, submodules, vcpkg/conan).
- [ ] Target platforms. Streamline is **Windows x64 only**; consoles, Linux and macOS need the feature compiled out cleanly.
- [ ] Compiler / MSVC toolset and Windows SDK version (Streamline needs a recent Windows 10 SDK).
- [ ] How DLLs are deployed next to the executable (Streamline plugins must ship beside the exe, or at a configured path).

### 2.2 Graphics API & device ownership
- [ ] API(s) in use: grep for `D3D12CreateDevice`, `CreateDXGIFactory`, `vkCreateInstance`, `vkCreateDevice`, `D3D11CreateDevice`.
- [ ] **Where the device and swap chain are created.** List the exact functions. This decides interposer vs manual hooking.
- [ ] Is there an RHI/abstraction layer wrapping the API? (Affects how resources are passed to Streamline.)
- [ ] Swap chain: `IDXGISwapChain` version, flip model, present mode, HDR, VRR/tearing flags, `Present`/`vkQueuePresentKHR` call sites.
- [ ] Queues: graphics/compute/copy queues in use; for Vulkan, queue family indices and how many queues are created.
- [ ] Other overlays or hooks on DXGI/Vulkan (Steam overlay, capture SDKs, anti-cheat). Note them as compatibility risks.

### 2.3 Frame structure & render graph
- [ ] Main loop: simulation/update → render → present. Locate it and draw the order.
- [ ] Render graph or pass list. Identify:
  - [ ] Opaque / G-buffer pass(es)
  - [ ] Lighting
  - [ ] Transparents / particles
  - [ ] **Temporal anti-aliasing (TAA) or existing upscaler** (FSR, XeSS, TSR, custom). This is where DLSS-SR goes.
  - [ ] Post-processing chain. Split it into **pre-upscale** (motion-sensitive: DOF, motion blur at low res, etc.) and **post-upscale** (tonemap?, bloom, film grain, chromatic aberration, vignette).
  - [ ] **UI / HUD rendering**: separate pass? Separate render target? Composited where?
- [ ] Is rendering done at a separate **render resolution** vs **output resolution**? Is dynamic resolution supported?
- [ ] HDR vs LDR at the point TAA runs (DLSS-SR wants pre-tonemap HDR or post-tonemap LDR consistently; record which).

### 2.4 Required inputs for DLSS-SR (check each exists)

| Input | What to look for | Notes |
|---|---|---|
| **Color (render res)** | Scene color before TAA | Must be pre-upscale and jittered |
| **Depth** | Depth buffer at render res | Note: reversed-Z? infinite far plane? linear vs hardware depth |
| **Motion vectors** | Velocity buffer | Screen-space? In pixels or NDC/UV? Render res or output res? Do they include **camera motion** or only object motion? Dilated? Jittered? |
| **Camera jitter** | Sub-pixel projection offset (Halton etc.) | Must exist and be applied to the projection. If not, it must be added |
| **Exposure** | Auto-exposure value or texture | Or use DLSS auto-exposure if the engine has none |
| **Camera matrices** | View, projection, prev-frame equivalents | Needed for constants (§5.4) |
| **Reset/cut signal** | Camera cuts, teleports, scene loads | Needed to reset history |
| **Optional masks** | Reactive/transparency, bias-current-color masks | Particles, water, UI-in-world |

**Missing motion vectors or jitter is the single biggest risk.** If either is absent, flag it as a prerequisite work item, not an integration task.

### 2.5 Required inputs for DLSS-G (Frame Generation)

| Input | Notes |
|---|---|
| **Depth** | Same as SR, usually |
| **Motion vectors** | Same as SR, usually |
| **HUD-less color** | Final color **after** post-processing but **before** UI. Strongly recommended for quality |
| **UI color + alpha** | UI in its own RGBA target, or the ability to make one. Without it, the UI will smear in generated frames |
| **Back buffer** | Swap chain image; Streamline hooks present |
| **Reflex / PCL markers** | Required. Find the simulation start/end, render submit start/end and present call sites |

- [ ] Can UI be rendered to a separate target? If UI is drawn straight into the back buffer, note it as a prerequisite refactor.
- [ ] Any CPU frame limiter, custom VSync or frame pacing logic? It will interact with FG; record it.
- [ ] Any **existing Reflex SDK** integration (`NvLowLatencyVk`, `NvAPI_D3D_SetSleepMode`)? It must be migrated to Streamline's Reflex/PCL.

### 2.6 Settings & UI
- [ ] Where graphics settings live (menus, config files, console vars).
- [ ] Existing upscaler options, so DLSS can slot in beside them.
- [ ] Whether settings are hot-applied or need a restart.

### 2.7 Testing infrastructure
- [ ] Automated tests, replay/benchmark modes, screenshot comparison.
- [ ] A deterministic camera path or demo you can use for before/after captures.

### 2.8 Assessment output

Write `docs/dlss-assessment.md` in the repo with:

1. **Summary**: engine, API, path chosen (§1), go / go-with-prerequisites / no-go.
2. **Evidence table**: each §2 item → status (✅ present / ⚠️ partial / ❌ missing / ❓ unknown) → file:line → notes.
3. **Prerequisites** (work that must happen before Streamline): e.g. "add camera jitter", "render UI to its own target", "port D3D11 → D3D12".
4. **Risks**: overlays, anti-cheat, custom present, multi-GPU, HDR edge cases.
5. **Open questions** for the human.
6. **Proposed plan**, laid out as Phases 2–6 below with estimates.

**Stop here and wait for human approval.**

---

## 3. Phase 2 — SDK setup

### 3.1 Get the SDK
- Download the latest **Streamline SDK** release from NVIDIA (GitHub `NVIDIA-RTX/Streamline` or NVIDIA Developer). Record the exact version in `docs/dlss-assessment.md` and in a `THIRD_PARTY_VERSIONS` file.
- Vendor it under e.g. `third_party/streamline/<version>/`. Keep:
  - `include/` (`sl.h`, `sl_dlss.h`, `sl_dlss_g.h`, `sl_reflex.h`, `sl_pcl.h`, `sl_consts.h`, `sl_helpers*.h`, etc.)
  - `lib/x64/sl.interposer.lib`
  - `bin/x64/` **production** DLLs: `sl.interposer.dll`, `sl.common.dll`, `sl.dlss.dll`, `sl.dlss_g.dll`, `sl.reflex.dll`, `sl.pcl.dll`, plus the NGX model DLLs (`nvngx_dlss.dll`, `nvngx_dlssg.dll`)
  - `bin/x64/development/` (dev builds with logging/overlay) for non-shipping configs only
- **Production builds must use the signed production DLLs.** Streamline verifies signatures; unsigned or modified plugins won't load in production mode.

### 3.2 Build integration
- Add include paths and link `sl.interposer.lib` (Windows x64 only).
- Wrap all Streamline code in `#if ENABLE_STREAMLINE` (or similar) so other platforms compile.
- Copy the right DLL set into the output directory per configuration (dev DLLs for Debug/Dev, production DLLs for Shipping).
- Add the NVIDIA **license/attribution** text the SDK requires to the game's third-party notices.

### 3.3 Application ID
- DLSS needs an **NVIDIA application/project ID** for production (set in `sl::Preferences`). Use the SDK's documented dev/test ID during development, and request a real one from NVIDIA before release. List this as a human action item.

---

## 4. Phase 3 — Initialization & device hookup

### 4.1 Choose the hooking mode
- **Interposer (default, simplest):** link against `sl.interposer` so it proxies `D3D12CreateDevice` / DXGI / Vulkan entry points. Easiest when the engine calls the API directly.
- **Manual hooking (lower overhead, more control):** keep native API calls, then call `slSetD3DDevice` (or the Vulkan equivalent) and `slUpgradeInterface` on the swap chain/factory, as `ProgrammingGuideManualHooking.md` describes. Use this when the engine has its own device wrapper, or other hooks conflict with the interposer.

Record the choice and the reason in the assessment doc.

### 4.2 `slInit` — must happen BEFORE device/swap-chain creation

Pseudocode (check names against your SDK version):

```cpp
sl::Preferences pref{};
pref.showConsole       = isDevBuild;
pref.logLevel          = isDevBuild ? sl::LogLevel::eDefault : sl::LogLevel::eOff;
pref.pathsToPlugins    = nullptr;   // plugins next to exe
pref.numPathsToPlugins = 0;
pref.renderAPI         = sl::RenderAPI::eD3D12;   // or eVulkan
pref.applicationId     = kNvidiaAppId;            // from NVIDIA
pref.engine            = sl::EngineType::eCustom;
pref.engineVersion     = "MyEngine x.y";
pref.projectId         = "<project GUID>";
pref.flags            |= sl::PreferenceFlags::eUseManualHooking; // only if manual hooking
// Request only the features you use:
sl::Feature features[] = { sl::kFeatureDLSS, sl::kFeatureDLSS_G, sl::kFeatureReflex, sl::kFeaturePCL };
pref.featuresToLoad    = features;
pref.numFeaturesToLoad = std::size(features);

if (SL_FAILED(res, slInit(pref, sl::kSDKVersion))) {
    // Log and continue WITHOUT Streamline. Never crash.
    g_streamlineAvailable = false;
}
```

At shutdown, call `slShutdown()` **after** the device and swap chain are released.

### 4.3 Feature support checks
After the device exists, query each feature on the adapter in use (`slIsFeatureSupported` with the adapter LUID). Cache the results:
- DLSS-SR: RTX 20-series and newer.
- DLSS-G: RTX 40-series and newer; **Multi Frame Generation** needs RTX 50-series. Read the supported max from the DLSS-G state/options, never hard-code it.
- Reflex: broadly supported on NVIDIA GPUs.

Settings menus must hide or grey out unsupported options. On AMD/Intel GPUs, everything stays off with no errors.

### 4.4 Vulkan specifics (if applicable)
- Streamline needs specific instance/device extensions and **additional queues** for DLSS-G (and optical flow on some generations). Use the SDK's helpers or documented lists. Don't guess.
- Make sure the engine's queue family allocation leaves room for the extra queues Streamline asks for.

---

## 5. Phase 4 — DLSS Super Resolution

### 5.1 Placement
DLSS-SR **replaces TAA** (and any other upscaler) at the same spot in the frame:

```
[G-buffer] → [Lighting] → [Transparents] → [pre-upscale post] → ★ DLSS-SR ★ → [post-upscale post] → [UI] → [Present]
              (render resolution) ─────────────────────────────────────────┘     (output resolution) ───────────
```

Render resolution comes from DLSS. Don't invent scale factors.

### 5.2 Optimal settings
On resolution/mode change, call `slDLSSGetOptimalSettings(options, settings)` with the output size and chosen mode. Modes:
`eUltraPerformance`, `eMaxPerformance` (Performance), `eBalanced`, `eMaxQuality` (Quality), `eDLAA` (native-res AA), and `eUltraQuality` if your SDK exposes it.

Use the returned optimal render width/height (and min/max for dynamic resolution). Then call `slDLSSSetOptions(viewport, options)`.

Model/preset selection: DLSS 4.5 ships newer transformer-based models. Leave the **preset at default** unless NVIDIA guidance for your SDK version says otherwise; expose overrides only in dev builds.

### 5.3 Jitter
- Required. Use a low-discrepancy sequence (Halton 2,3 is standard), with a **phase count scaled to the upscale ratio**. The DLSS programming guide recommends at least `8 × (outputWidth/renderWidth)²` phases.
- Apply jitter to the projection matrix. Pass the **jitter offset in pixels** to the constants with the right sign convention, and verify it (§8).
- Disable mip bias hacks from TAA, then **set a negative texture LOD bias** based on the render/output ratio: `log2(renderWidth/outputWidth) - 1.0` is the usual starting point (check the guide's current recommendation).

### 5.4 Per-frame constants (`sl::Constants`)
Each frame: get a frame token (`slGetNewFrameToken`), then call `slSetConstants(consts, *frameToken, viewport)`. Fill in, at minimum:

| Field | Meaning / pitfall |
|---|---|
| `cameraViewToClip`, `clipToCameraView` | Projection and inverse. **Without jitter** applied, unless the guide says otherwise for your version |
| `clipToPrevClip`, `prevClipToClip` | Reprojection matrices, current↔previous frame |
| `jitterOffset` | In pixels, render-res space; sign convention matters |
| `mvecScale` | Scales the engine's MVs into the expected space (e.g. `{1/renderW, 1/renderH}` if MVs are in pixels) |
| `cameraPos/Up/Right/Fwd`, `cameraNear/Far`, `cameraFOV`, `cameraAspectRatio` | World-space camera |
| `depthInverted` | `true` for reversed-Z |
| `cameraMotionIncluded` | `true` if MVs already include camera motion (usual for full velocity buffers) |
| `motionVectors3D` | Usually `false` |
| `motionVectorsDilated`, `motionVectorsJittered` | Match how the engine generates MVs |
| `orthographicProjection` | 2D/ortho cameras |
| `reset` | `true` on camera cuts, teleports, level loads, resolution changes |

Constants must be set **every frame** for every viewport, **before** `slEvaluateFeature` and before present (DLSS-G reads them too).

### 5.5 Resource tagging
Tag inputs with `slSetTagForFrame` (or `slSetTag` on older SDKs), per viewport:

- `kBufferTypeScalingInputColor`: render-res color
- `kBufferTypeScalingOutputColor`: output-res target (UAV-capable)
- `kBufferTypeDepth`
- `kBufferTypeMotionVectors`
- `kBufferTypeExposure` (optional; otherwise enable auto-exposure in options)
- Optional masks (bias-current-color / reactive)

For each tag, supply the resource, its **current state/layout**, the **extent** (the sub-rect if the texture is larger than the render size, e.g. under dynamic resolution) and the **lifecycle**:
- `eValidUntilPresent`: the resource persists until present (preferred; lets DLSS-G read it).
- `eOnlyValidNow`: transient/aliased resources that the render graph reuses. Streamline copies them, which costs bandwidth.

### 5.6 Evaluate
At the TAA slot, call:
```cpp
const sl::BaseStructure* inputs[] = { &viewport /*, optional per-call tags */ };
slEvaluateFeature(sl::kFeatureDLSS, *frameToken, inputs, std::size(inputs), cmdList);
```
Afterwards, **restore any pipeline state** your engine assumes (descriptor heaps, root signature, bound pipeline). Streamline changes command-list state.

### 5.7 Lifecycle
- Recreate/resize on resolution or mode change. Call `slFreeResources(kFeatureDLSS, viewport)` when disabling, or when a viewport goes away.
- Split-screen / multiple views: use a separate `sl::ViewportHandle` each.

---

## 6. Phase 5 — Reflex/PCL + DLSS Frame Generation

### 6.1 Reflex & PCL first (required for FG)
1. Enable Reflex via `slReflexSetOptions` (mode `eLowLatency`; offer "On" / "On + Boost" in the UI).
2. Call `slReflexSleep(frameToken)` at the **start of the frame**, before input sampling.
3. Add **PCL markers** with `slPCLSetMarker(marker, frameToken)` at:
   - `eSimulationStart` / `eSimulationEnd`
   - `eRenderSubmitStart` / `eRenderSubmitEnd`
   - `ePresentStart` / `ePresentEnd`
   - `eInputSample` (where input is read), and the PC latency ping if the input pipeline supports it
4. Use the **same frame token** across markers, constants and evaluate calls for one frame.
5. Remove any old standalone Reflex/NvAPI latency code.

Verify with the dev overlay or NVIDIA FrameView before moving on.

### 6.2 Frame Generation tags
In addition to the SR tags (DLSS-G reuses depth/MVs/constants), tag:
- `kBufferTypeHUDLessColor`: final color after post, before UI (output res)
- `kBufferTypeUIColorAndAlpha`: UI layer with alpha (premultiplied, per the guide)
- `kBufferTypeBackbuffer` (optional): helps detect UI that doesn't match HUD-less

Use `eValidUntilPresent` where you can. DLSS-G runs **at present time**, not at a call site.

### 6.3 Options
Use `slDLSSGSetOptions(viewport, options)`:
- `mode`: `eOn` / `eOff`, plus `eAuto` / dynamic mode if your SDK exposes it. DLSS 4.5 adds **Dynamic Multi Frame Generation**; check your SDK version's guide for the exact switch.
- `numFramesToGenerate`: 1 = 2× (classic FG); 2/3 (or higher if supported) = Multi Frame Generation on RTX 50. **Clamp to the max reported** by `slDLSSGGetState`.
- Back buffer format, dimensions and color space must match the swap chain. Update them on resize/HDR toggle.
- Set dynamic-resolution / VRAM-related flags as the guide describes.

### 6.4 Runtime behaviour to handle
- **Check `slDLSSGGetState` every frame or on change**: status, VRAM estimate, min dimensions, max frames. On failure states, turn FG off and inform the UI.
- **Turn FG off automatically** during loading screens, full-screen menus, video playback and pause screens, or wherever the HUD-less/UI split breaks down. Re-enable on return.
- **Camera cuts:** set `constants.reset = true` so generated frames don't interpolate across the cut.
- **VSync / frame limiters:** FG with VSync needs VRR-friendly settings. Document the recommended combination; don't stack a CPU limiter on top of Reflex without testing.
- **Swap chain recreation** (resize, fullscreen toggle, HDR): update options, and possibly turn FG off and on again; follow the guide's resize sequence.
- **Frame rate display:** in-game FPS counters should show rendered and presented FPS separately.

### 6.5 D3D11 note
FG doesn't support D3D11. If the assessment found D3D11, ship SR only and record "port to D3D12 for FG" as a roadmap item.

---

## 7. Phase 6 — Settings UI & UX

Follow NVIDIA's **RTX UI Developer Guidelines** (PDF in the SDK). Minimum set:

| Setting | Values | Notes |
|---|---|---|
| NVIDIA DLSS Super Resolution | Off / DLAA / Quality / Balanced / Performance / Ultra Performance (/ Auto) | Mutually exclusive with other upscalers/TAA modes |
| NVIDIA DLSS Frame Generation | Off / 2× / 3× / 4× (only the multipliers supported) | Grey out with tooltip on unsupported GPUs; requires Reflex |
| NVIDIA Reflex Low Latency | Off / On / On + Boost | Forced On (not Off) when FG is on |
| Sharpening (optional) | Use the engine's own post-sharpen | DLSS's built-in sharpening is deprecated in recent versions |

- Use NVIDIA's naming and trademarks exactly as the guidelines say.
- Persist settings, and validate them on startup against the current GPU's support.
- Expose dev-only toggles (console vars) for: SR on/off, FG on/off, preset override, debug overlay, MV visualisation.

---

## 8. Verification checklist

Run on at least one RTX 40 and one RTX 50 GPU, plus one non-NVIDIA GPU.

### Correctness
- [ ] Game launches, plays and exits cleanly with Streamline **disabled**, **missing DLLs**, and on **AMD/Intel**.
- [ ] Dev overlay / logs show the plugins loaded, the features supported, and no per-frame warnings.
- [ ] **Motion vector check:** pan the camera over a static scene with DLSS-SR at Performance. There should be no swimming, smearing or ghosting. If the image moves "the wrong way", MV sign or scale (`mvecScale`) is wrong.
- [ ] **Jitter check:** static camera and static scene. The image should converge to a sharp result. If it shimmers or blurs, the jitter sign/units or projection matrix (with vs without jitter) is wrong.
- [ ] **Depth check:** if foreground/background disocclusion ghosts, check `depthInverted` and depth format.
- [ ] **Camera cut:** no one-frame ghost after cuts or teleports (`reset` works).
- [ ] **Transparents/particles:** check for ghost trails; add masks if needed.
- [ ] **UI with FG:** no smeared, doubled or warped HUD text in generated frames, on moving text, damage numbers, crosshair or minimap.
- [ ] Every mode switch, resolution change, fullscreen/windowed toggle, HDR toggle and alt-tab works without crashes or leaks (watch VRAM over 30+ minutes).
- [ ] Loading screens, menus, pause and video playback: FG auto-disables or looks correct.

### Performance
- [ ] Capture the same benchmark path for: native+TAA, each DLSS-SR mode, SR+FG 2×, SR+MFG 3×/4×.
- [ ] Record GPU time of the DLSS pass and total frame time, rendered vs presented FPS, and **PC latency** (Reflex overlay / FrameView).
- [ ] Check CPU overhead of tagging/constants (it should be negligible). If `eOnlyValidNow` copies show up, revisit resource lifetimes.

### Shipping
- [ ] Production DLLs are signed and in the build; dev DLLs are absent from shipping packages.
- [ ] A real NVIDIA application ID is set.
- [ ] Third-party notices are updated.
- [ ] The installer/patcher deploys every DLL. A patch must not leave an old `nvngx_*.dll` paired with a new Streamline.

---

## 9. Common pitfalls (quick reference)

| Symptom | Likely cause |
|---|---|
| DLSS never initialises | `slInit` called after device creation; DLLs missing or unsigned in production mode; wrong `renderAPI` |
| Heavy ghosting/smearing in motion | MVs wrong scale/sign/space; `cameraMotionIncluded` wrong; MVs at wrong resolution |
| Static image shimmers/blurry | Jitter missing, wrong sign, or jitter left in the matrices passed as un-jittered |
| Ghosting after cuts | `reset` not set |
| Crash or corruption after evaluate | Engine assumed command-list state Streamline changed; resource state passed wrongly |
| UI smears with FG | No `UIColorAndAlpha`/`HUDLessColor` tags, or UI drawn into the scene before HUD-less capture |
| FG greyed out on RTX 40/50 | Reflex/PCL not active; D3D11; HDR/back buffer mismatch; VRAM state failure (check `slDLSSGGetState`) |
| Stutter with FG | Competing frame limiter/VSync; missing PCL markers; present marker in the wrong place |
| Works in dev, fails in shipping | Production vs development DLL mismatch; app ID not set |

---

## 10. Forward-compatibility: preparing for DLSS 5

DLSS 5 (neural rendering) uses the Streamline pipeline, but the plugin and its docs aren't public yet. Ask NVIDIA developer relations for access. To make adoption cheap later:

- Keep the Streamline integration **feature-agnostic**: one module handles init, frame tokens, constants and tagging. Features plug in as small adapters.
- Make sure the renderer can expose clean **G-buffer data** (albedo, normals, roughness/metalness), depth, MVs and **per-material/region masks** (skin, hair, foliage, UI-in-world, "do not alter" art regions). Expect neural rendering to need richer scene guides than SR.
- Keep art-direction controls in mind: studios will want to limit how much the model reinterprets stylised art.
- Track Streamline SDK releases for a public `sl.dlss_nr`-style plugin and guide.

---

## 11. Agent deliverables, in order

1. `docs/dlss-assessment.md`: Phase 1 report (§2.8). **Wait for approval.**
2. Prerequisite PRs (jitter, MVs, UI separation, API port) if needed, each reviewed separately.
3. PR: Streamline SDK vendoring + build integration + init/shutdown + feature detection (no visual change).
4. PR: DLSS-SR + settings + dev toggles.
5. PR: Reflex/PCL.
6. PR: DLSS-G / MFG + UI tags + auto-disable logic.
7. `docs/dlss-verification.md`: §8 checklist with results, captures and benchmark numbers.
8. `docs/dlss-maintenance.md`: SDK version, how to update DLLs and models, known issues, the DLSS 5 notes from §10.

Each PR description should list: what changed, how to toggle it, how it was tested, and known limitations.
