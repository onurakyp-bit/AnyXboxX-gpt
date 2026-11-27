# NVIDIA stability and performance audit

**Target machine:** Windows, Intel Core i5-11400F (6 cores / 12 logical CPUs), NVIDIA GeForce RTX 4070 SUPER (12 GB), 32 GB system RAM.
**Target game:** GTA VI Xbox Series build 1.0.2.1.
**Audit baseline:** repository `main` as inspected on 2026-11-27.
**Status:** source audit in progress. This document is not a claim that the target machine has passed a build or game test.

## Executive summary

The first goal is not a higher peak FPS number. It is to stop missing graphics, prevent memory exhaustion, and reduce long frame-time spikes. Only then should throughput changes be accepted.

The repository's current Windows + NVIDIA profile is explicitly measured on an RTX 4070 SUPER paired with an i7-13700KF. It is not a measurement for the target i5-11400F. The profile enables both `VICTORY_THIN=1` and `VICTORY_JOIN_PARTS=8`. These options alter command submission and submission batching; they are not simple quality toggles. They must be A/B-tested on the target machine before being treated as universally safe.

## Evidence from the current source

### 1. The current tuned profile is not target-hardware validated

- `crates/anyxboxx-launcher/profiles.toml` labels Windows + NVIDIA as a measured set, but its note identifies the measurement machine as RTX 4070 SUPER + i7-13700KF.
- `crates/anyxboxx/src/run/settings.rs` documents the same measured baseline and the 8/6/4 worker pool.
- The larger 12/8/6 and 12/8/8 worker pools are explicitly measured only on a 24-logical-CPU i7-13700KF. The target i5-11400F has 12 logical CPUs and should not inherit those larger pools.
- The 12 GB VRAM preset in `crates/anyxboxx-launcher/src/options.rs` is explicitly marked as not measured on a 12 GB card. It is a memory-headroom candidate, not a validated optimum.

### 2. Submission modes need controlled testing

- `third_party/victory/crates/victory-gpu/src/list.rs` switches to a thin command-list implementation when `VICTORY_THIN=1`.
- `third_party/victory/crates/victory-gpu/src/host.rs` documents `VICTORY_JOIN_PARTS` as an opt-in batching mechanism. Its notes describe a measured CPU-time reduction in one scene, but also require front-refresh ordering to preserve title-memory correctness.
- The source comments describe trade-offs and correctness constraints. Neither switch should be removed or enabled globally based only on a different machine's benchmark.

### 3. Existing target-machine logs show memory pressure

Previously supplied logs from the target PC reported:
- system RAM near 29.7 / 31.9 GB;
- VRAM near 9.7 / 10.6 GB in the overlay's reported budget;
- low and unstable frame rates;
- missing host pipelines and Wave32 lowering warnings;
- texture queue / staging pressure in an earlier run.

These symptoms point to multiple independent bottlenecks. Reducing texture residency pressure, fixing missing pipelines, and avoiding excessive CPU work must be handled separately.

### 4. Pipeline library correctness matters

`third_party/victory/crates/victory-gpu/src/pipeline/library.rs` documents a Windows memory-mapping lifetime hazard: if the mapped pipeline-library file is replaced while its handle is closed, pages may read as zeros and the D3D12 device can be removed. The current implementation keeps the handle open by default. This behavior must be preserved unless a replacement design has equivalent lifetime guarantees.

## Work plan

### Phase A: establish a reproducible baseline
- Record exact commit, build configuration, driver, GPU budget, game build, resolution, cache state, and scene.
- Separate cold-cache shader compilation from warm-cache gameplay.
- Use a fixed scene and fixed capture interval.
- Track median, 10th percentile, 1% low, frame-time distribution, RAM, VRAM, and dropped draw/pipeline counts.

### Phase B: correctness and stability
- Reproduce the remaining Wave32 lowering failures and identify their shader paths.
- Trace the missing-host-pipeline cases from pipeline creation through draw/dispatch.
- Audit descriptor heap bounds and texture upload / residency eviction paths.
- Verify that the PSO sanitization fix remains covered by focused regression tests.
- Preserve pipeline-library mapping and handle lifetime guarantees.

### Phase C: target-machine tuning
Run controlled A/B tests, one variable group at a time:
1. baseline current Windows + NVIDIA profile;
2. thin submission on/off;
3. joined submission parts at default and the measured value;
4. worker-pool sizes appropriate to 12 logical CPUs;
5. 12 GB VRAM preset versus measured profile reserves;
6. texture budget / maximum texture size;
7. pipeline worker count and cache warm-up.

Do not combine multiple changes in one test. Reject any candidate that improves median FPS but increases missing geometry, texture loss, device removal, or severe frame-time spikes.

### Phase D: acceptance criteria
A candidate is not called complete until:
- the repository builds on Windows in CI;
- Rust tests and repository checks pass;
- no regression is observed in the PSO and launcher profile tests;
- the target machine completes a repeatable warm-cache gameplay session;
- screenshots / logs confirm no new missing geometry or texture failures;
- performance is reported with frame-time percentiles, not just a peak FPS value.

## Known limitations

GitHub Actions can validate source, tests, and Windows compilation. It cannot validate GTA VI gameplay on the target PC's RTX 4070 SUPER and i5-11400F. That final acceptance test requires the user's local machine and game files.

No FPS target is guaranteed. The reference machine in the README is materially faster on the CPU side, and its results must not be presented as results for the target machine.
