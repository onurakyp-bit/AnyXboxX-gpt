# RTX 4070 SUPER / i5-11400F stability investigation

## Scope

This note records a reproducible investigation plan for a Windows host with an NVIDIA RTX 4070 SUPER (12 GB), an Intel Core i5-11400F (6 cores / 12 threads), and 32 GB of system RAM.

The goal is not to promise a frame-rate target. The immediate acceptance criteria are:

1. The game reaches the open world reliably.
2. Missing or black geometry/textures are reduced without introducing new rendering regressions.
3. Memory use remains stable during a sustained session.
4. Frame-time consistency improves before peak FPS is treated as a success.

No game executable, assets, save, extracted shader, or proprietary content belongs in this repository.

## Baseline evidence

The host's previously shared runtime observations include:

- Intermittent performance around 13 FPS, with visible stutter.
- A prior overlay sample showing system RAM near 29.7 / 31.9 GB and VRAM near 9.7 / 10.6 GB of the reported budget.
- Repeated creation failures for device method slots `0x1c8` and `0x218`. In the vendored `victory-gpu/src/device.rs`, both slots currently point to `unknown_pipeline`, return `E_NOTIMPL`, and clear the output pointer. The source comments confirm that the title requests `ID3D12PipelineState`; the description layout is still unknown.
- Wave32 lowering failures for shader hashes `5dc3b25986622949` and `6f225b641c5b0708`.
- The tuned profile contains a `VICTORY_WAVE_WIDE_ONLY` allowlist that includes both hashes, but it does not set `VICTORY_WAVE_WIDE=1`. In the vendored shader emitter, the allowlist only scopes the explicit wave-wide fallback after that opt-in is enabled; by itself, the list does not enable the fallback. The source also warns that this form puts group barriers inside branches/loops and previously caused device loss in other kernels, so enabling it globally without a controlled test would be unsafe.
- Earlier logs also reported texture-heap pressure and dropped draws when no host pipeline was available.

These are diagnostic observations from different runs, not a single controlled benchmark. The low FPS samples captured in loading/menu periods must not be treated as a representative open-world average.

## Current launcher profile facts

The Windows + NVIDIA profile is the maintainer's measured profile, but its published measurements were made on an RTX 4070 SUPER with an i7-13700KF. They do not establish performance on a 6-core / 12-thread i5-11400F.

The launcher already exposes a separate **12 GB card** preset. It reduces the draw-data and buffer reserves from 1024 MB windows and 1536 MB buffer-copy reserves, with the buffer-copy reserve reduced to 1024 MB and caps the largest texture level at 2048 texels. The launcher documentation explicitly says that this preset has not yet been measured on a 12 GB card. It is therefore a candidate for an A/B test, not a proven fix.

The measured Windows + NVIDIA profile also sets `VICTORY_THIN=1` and `VICTORY_JOIN_PARTS=8`. Do not change these blindly: first confirm the exact code paths and semantics in the vendored VIctory revision, then compare output correctness and frame times independently.

The earlier PSO normalization fix for integer render-target blending (`LogicOpEnable`) and stale `RTFormats` past `NumRenderTargets` is already present in `third_party/victory/crates/victory-gpu/src/pipeline/host.rs`. The remaining `0x1c8` / `0x218` warnings are separate device methods, not a reason to reapply that same PSO fix.

## Controlled test sequence

Keep the game build, save location, scene, camera position, display mode, driver, and cache state constant. Record the first run separately from warm-cache runs.

| Pass | Launcher preset | Output mode | Purpose |
|---|---|---|---|
| A | Fast | 1920x1080 | Reference run |
| B | 12 GB card | 1920x1080 | Check VRAM headroom and texture completeness |
| C | 12 GB card | 1280x720 | Separate render-resolution cost from CPU-side work |
| D | Fast | 1280x720 | Compare preset effect at the same output size |

For each pass, record:

- Median FPS and 1% low if available.
- Median and 95th-percentile frame time.
- CPU total and per-core utilization.
- GPU utilization and GPU frame time.
- Dedicated and shared GPU memory.
- System RAM and process working set.
- Draws submitted, draws dropped for missing host pipelines, and pipeline compilation counts.
- Whether the same geometry and textures are present in the same scene.

Run each warm-cache pass for at least 5 minutes after the scene has settled. Repeat a pass only when its scene or cache state differs.

## Decision rules

- If shared GPU memory rises sharply while dedicated memory approaches its budget, investigate the 12 GB preset and texture residency before CPU scheduling.
- If GPU frame time is low while one or more CPU cores are saturated, investigate host-side copies, read-backs, pipeline waits, and worker scheduling.
- If the frame rate changes but missing geometry increases, reject the change.
- Treat a setting as an optimization only after an A/B result on the same host and scene.
- Do not cherry-pick asynchronous submission, memory-residency, or descriptor changes solely because a different GPU/CPU combination benefited.

## Work still required

- Trace the current VIctory source for every setting in the Windows + NVIDIA profile, especially `VICTORY_THIN` and `VICTORY_JOIN_PARTS`.
- Recover the description layouts for device methods `0x1c8` and `0x218`, then implement pipeline creation with tests. Their current failure is confirmed in `third_party/victory/crates/victory-gpu/src/device.rs`; the layouts are not yet understood.
- Reproduce the two Wave32 lowering failures; test the allowlisted wave-wide fallback for those hashes only, with device-loss monitoring, then add regression tests before changing the default profile.
- Audit texture residency and dropped-draw handling under the 12 GB budget.
- Run the controlled matrix on the target host and publish only measured results.

## Status

This is an investigation plan, not a claim that the above issues are fixed. The host-specific game validation must be run on the target Windows machine; GitHub-hosted CI cannot validate the actual game session or NVIDIA driver behaviour.
