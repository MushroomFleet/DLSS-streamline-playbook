# 🎮 DLSS Streamline Playbook

**An agent-ready playbook for assessing any game codebase and integrating NVIDIA Streamline with DLSS 4.5 Super Resolution and Frame Generation.**

---

## What is this?

This repository contains a single, structured Markdown playbook, [`dlss-streamline-integration-playbook.md`](./dlss-streamline-integration-playbook.md). It is written to be handed to an AI coding agent such as **Claude Code**, or followed by an engineer, inside a game's repository.

The playbook tells the agent how to:

1. **Assess** an unfamiliar game codebase (engine, graphics API, render graph, motion vectors, jitter, UI passes, swap chain and present flow) without changing anything.
2. **Decide** the right integration path: the official NVIDIA plugin for Unreal Engine or Unity, or a direct **Streamline** integration for custom and in-house engines.
3. **Plan and implement**, in small reviewable steps:
   - Streamline SDK setup and initialization
   - **DLSS Super Resolution / DLAA**
   - **NVIDIA Reflex + PC Latency (PCL) markers**
   - **DLSS Frame Generation and Multi Frame Generation**
   - Settings UI that follows NVIDIA's RTX UI guidelines
4. **Verify** the result with correctness, performance and shipping checklists, plus a table of common failure symptoms and their likely causes.
5. **Prepare** the renderer for **DLSS 5 neural rendering**, so it's straightforward to adopt once the SDK becomes publicly available.

Target APIs are **DirectX 12** and **Vulkan**. DirectX 11 supports Super Resolution only, because Frame Generation requires D3D12 or Vulkan.

---

## Why does this exist?

DLSS is no longer tied to Unreal Engine. NVIDIA's open-source **Streamline** framework lets any engine add DLSS through one integration. Doing that well still means getting many details right: jitter sequences, motion-vector scaling, depth conventions, resource lifetimes, UI separation for frame generation, Reflex markers, and production DLL signing.

This playbook gathers those details into a repeatable, phase-gated workflow:

- **Assessment first.** The agent produces an evidence-backed report (`docs/dlss-assessment.md`) and stops for human approval before changing any code.
- **SDK docs win.** The agent pins a Streamline SDK version and treats NVIDIA's own programming guides as authoritative, recording any differences from the playbook.
- **Safe by default.** Every feature sits behind compile-time and runtime toggles, and the game must still run with Streamline missing, disabled, or on non-NVIDIA hardware.
- **Official integrations only.** No community injectors or DLL-swap hacks.

---

## How to use it

1. Copy `dlss-streamline-integration-playbook.md` into the root of your game repository.
2. Open the repository with Claude Code (or your agent of choice) and start with the assessment phase:

   ```text
   Follow dlss-streamline-integration-playbook.md — Phase 1 (assessment) only.
   Write docs/dlss-assessment.md and stop for review.
   ```

3. Review the assessment, resolve open questions and prerequisites, then approve the next phase:

   ```text
   Proceed with Phase 2 (SDK setup) per the playbook.
   ```

4. Continue phase by phase. Each one ends in a reviewable PR, followed by the verification and maintenance documents.

### Deliverables the agent produces

| Order | Deliverable |
|---|---|
| 1 | `docs/dlss-assessment.md`: codebase assessment and plan (approval gate) |
| 2 | Prerequisite PRs (jitter, motion vectors, UI separation, API port) if needed |
| 3 | Streamline SDK vendoring, build integration, init/shutdown, feature detection |
| 4 | DLSS Super Resolution + settings + dev toggles |
| 5 | Reflex / PCL |
| 6 | DLSS Frame Generation / Multi Frame Generation |
| 7 | `docs/dlss-verification.md`: test results, captures, benchmarks |
| 8 | `docs/dlss-maintenance.md`: SDK versions, update process, known issues |

---

## Playbook contents

| Section | Topic |
|---|---|
| 0 | Ground rules for the agent |
| 1 | Decision tree: UE5 / Unity / custom engine, and API support |
| 2 | Phase 1: Codebase assessment (read-only) |
| 3 | Phase 2: Streamline SDK setup, build integration, application ID |
| 4 | Phase 3: Initialization, hooking mode, feature detection, Vulkan notes |
| 5 | Phase 4: DLSS Super Resolution (placement, jitter, constants, tagging, evaluate) |
| 6 | Phase 5: Reflex/PCL and DLSS Frame Generation |
| 7 | Phase 6: Settings UI and UX |
| 8 | Verification checklist |
| 9 | Common pitfalls |
| 10 | Forward compatibility with DLSS 5 |
| 11 | Agent deliverables |

---

## Requirements & notes

- **Platform:** Windows x64 (Streamline is Windows-only; other platforms should compile the feature out).
- **Hardware for testing:** RTX 20-series or newer for Super Resolution, RTX 40-series or newer for Frame Generation, RTX 50-series for Multi Frame Generation. Also test on one non-NVIDIA GPU.
- **SDK:** download the latest [NVIDIA Streamline SDK](https://github.com/NVIDIA-RTX/Streamline) and the matching DLSS components from [NVIDIA Developer](https://developer.nvidia.com/dlss).
- **Production:** shipping builds need NVIDIA's signed production DLLs and an NVIDIA-issued application ID.
- **DLSS 5:** neural rendering is not part of the public Streamline SDK at the time of writing. Access is arranged through NVIDIA. The playbook covers preparation only.

> This project is an independent community resource and is not affiliated with or endorsed by NVIDIA. NVIDIA, DLSS, Reflex, RTX and Streamline are trademarks of NVIDIA Corporation. Always follow NVIDIA's current SDK documentation and license terms.

---

## 📚 Citation

### Academic Citation

If you use this codebase in your research or project, please cite:

```bibtex
@software{dlss_streamline_playbook,
  title = {DLSS Streamline Playbook: an agent-ready playbook for assessing game codebases and integrating NVIDIA Streamline with DLSS 4.5 Super Resolution and Frame Generation},
  author = {[Drift Johnson]},
  year = {2026},
  url = {https://github.com/MushroomFleet/DLSS-streamline-playbook},
  version = {1.0.0}
}
```

### Donate:

[![Ko-Fi](https://cdn.ko-fi.com/cdn/kofi3.png?v=3)](https://ko-fi.com/driftjohnson)
