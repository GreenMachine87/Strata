# Community benchmark: 2x RTX 5060 Ti 16 GB (layer split), IQ3_S at 262K — engine 0.1.41

Measured 2026-10-13 on an always-on production server: this machine serves a
multi-agent coding fleet 24/7, so every number here was taken **with background
load present** (see "Background load"). Three measurement streams:

1. the official suite `tools/strata_bench.py --long --runs 3` against the
   already-running server (light requests, thinking off);
2. a cold-prompt sweep (3 reps at 4,096 / 32,768 / 131,072 target tokens,
   unique salted prompts, `cache_n = 0` verified on every run);
3. a 87-hour live fleet-load audit from the engine's own `/metrics` totals
   (warm, reuse-dominated — kept separate from the cold numbers).

## Hardware and software

- GPU and VRAM: 2x NVIDIA GeForce RTX 5060 Ti, 16 GB each (both GPUs used,
  auto layer split: layers 0-23 on CUDA0, 24-47 + head on CUDA1)
- PCIe: gen 3, width x8 per card (engine probe: 6.9-7.0 GB/s host->device;
  `pcie_frac 0.19`)
- GPU power limit: **150 W enforced** (factory default 180 W) on both cards
- CPU: Intel Xeon E5-2683 v4 @ 2.10 GHz (32 threads, AVX2 only, no AVX-512;
  the engine logs "the expert kernels run on AVX-2")
- RAM: 220 GiB; storage: model weights stream over **NFS v4.2** from a NAS
  (1 GbE); OS: Ubuntu 24.04.4 LTS, kernel 6.8.0-138-generic
- Driver 580.173.02; CUDA 13.0 (source build, nvcc 13.0.88,
  `-DCMAKE_CUDA_ARCHITECTURES=120`)
- Strata engine **0.1.41** (source build at `fb58e0db`); the bench tooling
  (`tools/strata_bench.py`, `tools/needle_bench.py`) comes from the v0.1.43
  checkout (`7695859a`) run against the live 0.1.41 server via `--url`

## Model and configuration

- Model: ISTA-DASLab Qwen3.8-Flash-Next-GSQ-RCO, **IQ3_S**, 262,144-token
  context (native window), `--kv int8 --kv-resident 32768`
- GGUF: `Qwen3.8-Flash-Next-GSQ-RCO-IQ3_S-00001-of-00002.gguf`
  (55.06 GB) + PLE shard `-00002-of-00002.gguf` (28.8 GB); vision:
  `mmproj-Qwen3.8-Flash-Next-BF16.gguf` (907 MB), `--vision`
- Custom pack `ista-iq3_s` — `dense.bin` sha256
  `d2af3d8331e5c7a1985d1d589747230f9d56719a6d33c7cddc9ee6b81cde6b13`
- Learned expert profile `expert-profile-iq3-s.bin` (24,576 ranked pairs)
  sha256 `1b2c1249c2364f1126465d745746ccce283aadb3a5ed1cba927131e5ac1abe26`
- MTP draft: `mtp/rt`, `--spec 4 --spec-min-p 0.5` (engine reports spec 6 with
  lookup 3); expert cache auto: 9,995 slots total (5,667 CUDA0 + 4,328 CUDA1);
  the split search held **10,265 of 24,576 profiled pairs (~97.4% of the
  routed mass)**
- Conversation cache: 48 GiB, 16 slots; env: `STRATA_ADAPT_NOWAIT=1`,
  `STRATA_PREFILL_CPU_SHARE=0` (AVX2-only host)
- Sanitized launch command (home replaced by `~`):

```
~/Strata/build41/strata --serve --pack packs/ista-iq3_s \
  --native /mnt/tank/ai-models/downloads/ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF/IQ3_S/Qwen3.8-Flash-Next-GSQ-RCO-IQ3_S-00001-of-00002.gguf \
  --ple-gguf /mnt/tank/ai-models/downloads/ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF/IQ3_S/Qwen3.8-Flash-Next-GSQ-RCO-IQ3_S-00002-of-00002.gguf \
  --expert-profile data/expert-profile-iq3-s.bin \
  --expert-profile-save data/expert-profile-iq3-s-live.bin --expert-profile-save-every 1 \
  --expert-cache auto --prefill auto --spec 4 --spec-min-p 0.5 --mtp mtp/rt \
  --max-context 262144 --kv int8 --kv-resident 32768 --ple-io ram --vision \
  --remote-expert-opt --conversation-cache-mib 49152 --conversation-cache-slots 16 \
  --layer-split auto
```

Engine start-up lines (layer split, cache coverage): `engine-log-excerpt.txt`.

## Results — official suite (`tools/strata_bench.py --long --runs 3`, medians with ranges)

Speeds are the engine's own timings; TTFT measured by the client over HTTP.
Thinking off; every prompt uniquely tagged (no cache help on fresh prompts).

| Configuration | Actual prompt tokens | Reused | Generated | Runs | Prompt tok/s median (range) | Decode tok/s median (range) | TTFT s median (range) |
| --- | ---: | ---: | ---: | ---: | --- | --- | --- |
| short chat | 46 (43-51) | 0 | 100 | 9 | not measured | 68.3 (54.6-79.3) | 0.68 (0.54-0.97) |
| 4K prompt, fresh | 4019 | 0 | 24 cap | 3 | 619.1 (614.4-627.0) | not measured | 13.38 (6.45-20.69) |
| 16K prompt, fresh | 16058 | 0 | 24 cap | 3 | 1377.3 (1369.7-1393.3) | not measured | 19.39 (11.59-25.43) |
| 32K prompt, fresh | 32049 | 0 | 24 cap | 3 | 1584.8 (1581.3-1606.7) | not measured | 20.31 (20.07-20.39) |
| agent session, follow-up turns | 14478 at last turn | 68.6% hit | up to 64 | 3 | see `results.json` | 69.4 (44.3-105.1) | 6.39 (5.70-57.38) |

TTFT ranges are inflated by queue waits behind fleet traffic (the engine was
never idle during the suite); the engine's own prompt/decode rates are not.
Per-request data: `results.json`. Suite total: 445 s.

## Results — cold-prompt sweep (our harness, 3 reps, `cache_n = 0` verified)

Unique salted whole-word prompts, `ignore_eos`, thinking off, 256-token cap;
engine's own `timings` per run. Full per-run data: `cold-arm-results.json`.

| Target ctx | Actual prompt tokens | Prompt tok/s median [range] | Decode tok/s median [range] | Draft accept | TTFT s (queue-inclusive) |
| ---: | ---: | --- | --- | --- | --- |
| 4,096 | 3,092 | 492.3 [484-500] | 32.6 [15.7-44.2] (43 tok gen) | 0.871 | 19.4-135.9 |
| 32,768 | 23,426-24,596 | 1,521.1 [1505-1523] | 37.7 [35.6-49.1] (41-256 tok gen) | 0.794 | 32.7-69.7 |
| 131,072 | 95,978-98,318 | 1,892.2 [1891-1905] | 50.0 [33.2-54.7] (43-256 tok gen) | 0.576 | 82.5-192.6 |

Caveats, stated plainly: several runs ended at ~41-43 generated tokens (the
same early-stop pattern the banked 0.1.40 arms show), which swings the decode
median; and the fleet was serving concurrently (1-2 requests queued during
the sweep, host loadavg 12-24). The 4K prompt rate (492 t/s) is lower than
the suite's 4K number (619 t/s) because the two harnesses build different
prompts (this one calibrates at 1.178 tok/word against the engine's counter).

## Recall — `tools/needle_bench.py --lengths 32k --depths 10,50,90`

**3/3 found** (32,804-32,805 prompt tokens each; 21.6 / 21.2 / 42.1 s).
Raw JSON: `needle-32k.json`.

## Live fleet-load audit (warm; kept SEPARATE from the cold throughput above)

Engine `/metrics` totals after **87.1 h** of continuous production serving
(snapshot 2026-10-13, same engine session as the runs above):

- 8,522 requests; 703.6M prompt tokens; **96.2% prompt reuse**
- **70.2 t/s lifetime decode** (9.47M output tokens)
- **74.0% speculative accept** (7.63M drafts offered)
- conversation cache: 6,510 parks / 4,869 restores / 562 evictions
- expert cache live: 9,995 slots across both cards

This is the reuse-dominated steady state of a real agent fleet (long
conversations, mostly-cached prompts), not a cold benchmark. Raw snapshot:
`live-audit.json`.

## Comparison bracket (community reports, engine 0.1.41 unless noted)

| Rig | Quant / ctx | Decode t/s (4K / 32K / 128K) | Prompt t/s (4K / 32K / 128K) |
| --- | --- | --- | --- |
| **ours: 2x 5060 Ti 16 GB, gen3 x8, E5-2683 v4, 220 GiB, NFS** | IQ3_S / 262K | cold 32.6 / 37.7 / 50.0 (fleet load); warm lifetime 70.2 under 96% reuse | cold 492 / 1,521 / 1,892; suite 619 / 1,377 (16K) / 1,585 (32K) |
| 2x TITAN RTX 24 GB, E5-2696 v4 (0.1.40.3) | IQ3_S / 262K | 75.4 / 80.2 / 70.6 | 922.6 / 1,642.1 / 1,853.6 |
| 1x 5060 Ti + E5-2680 v4, Win10 | IQ1_M / 262K | 47.5 / 51.0 / 49.1 | 1,179.7 / 1,600.6 / 1,859.5 |
| 1x 5060 Ti + E5-2680 v4, Win10 | Q2_0 / 262K | 68.0 / 73.4 / 68.5 | 782.2 / 1,549.4 / 1,513.8 |
| 1x 5060 Ti WSL2, Ryzen 7 3700X | IQ3_S | 48.2 / 47.7 (short prompts, single runs) | 343 (4K) / 598 (20K) |

Reading: prefill on this dual-16 GB gen3-x8 NFS box **matches the 24 GB TITAN
pair at 128K** (1,892 vs 1,854 t/s) and the single-5060-Ti reports at 32K.
Cold decode sits below the TITAN pair — the TITAN report was measured on an
idle machine, ours ran with a live agent fleet queued in front, and several
cold runs decoded only ~43 tokens. The warm lifetime figure (70.2 t/s at
96.2% reuse over 87 h of real work) is the number this rig actually delivers
day-to-day, and it is well above both single-5060-Ti reports at comparable
quant despite half the per-card VRAM and gen3 x8.

## Background load and limitations

- The agent fleet serves on this server continuously; it was **not** paused
  for these runs (fleet restarts drop live conversations). `/metrics`
  `live.queued` stayed at 0-2 throughout; no arm was aborted.
- Decode rates therefore carry fleet interference; TTFT carries queue waits.
  Engine-reported prompt/decode rates exclude queueing but not GPU sharing.
- GPU power limit 150 W (below the 180 W default) — decode rates are
  power-capped relative to an unlimited card.
- The suite's agent-session decode (69.4 t/s) and the cold sweep's short
  decode medians measure different things (warm reuse vs cold prefill-then-
  decode); both are reported, neither is averaged with the other.

## Files in this folder

- `README.md` — this report
- `results.json` — official suite per-request data (auto-sanitized by the tool)
- `engine-log-excerpt.txt` — engine start-up lines (layer split, coverage)
- `needle-32k.json` — recall results
- `cold-arm-results.json` / `cold-arm-summary.txt` — cold sweep per-run data
- `live-audit.json` — `/metrics` totals snapshot (warm audit)
