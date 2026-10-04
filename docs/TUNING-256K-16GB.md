# Tuning Qwen3.8-Flash-Next IQ3_S for 256K context on a 16 GB card (measured)

A community result, not an official doc: one PC, one benchmark prompt, real numbers, and what they changed.
The short version: **`--calibrate` + `--spec 6` + the learned expert profile took output at a 257,630-token
context from 17.2 to 43-53.5 tokens/s (2.5-3.1x) on an RTX 5070 Ti 16 GB** - and filled in the IQ3_S 262K
row that [DETAILS.md](DETAILS.md#speed-measured) leaves unmeasured.

## The PC

| Part | What |
| --- | --- |
| GPU | NVIDIA RTX 5070 Ti, 16 GB, driver 610.74, PCIe 5 x16 |
| CPU | Intel Core i7-14700KF (8 P-cores + 12 E-cores, AVX2, no AVX-512) |
| RAM | 96 GB DDR5-5600 (2 sticks, XMP on) |
| Model | Qwen3.8-Flash-Next **IQ3_S** (GSQ-RCO), MTP draft layer, vision encoder on the GPU |
| Engine | 0.1.38 (ready-made), Windows, context 262,144, 8-bit KV with streaming (`--kv-resident 32768`) |

## What was changed

1. **`START-HERE.bat --calibrate`** (setup's own tuner, engine 0.1.19+). It measured this PC and kept:
   - `--pool-workers 13` (the default 19 measured **36.9** tok/s on its short prompts, 13 measured **54.4** - the
     single biggest win. The verify window is as slow as its slowest worker: at 19 threads the E-cores stall
     every window; 13 keeps the work on the P-cores),
   - `--spec-min-p 0.70` (0.5: 41.7, 0.3: 36.6 on the same runs),
   - `--pcie-frac 0.00` (0.2: 37.9, 0.35: 27.9, 0.55: 23.9, 0.75: 20.7 - with a PCIe 5 link and a fast CPU,
     computing a missed expert beats copying it over).
   The calibration is remembered per PC and model in the settings file, so updates keep it.
2. **`--spec 6`** (from 4) in `strata-<model>.json`: with the higher draft floor, the deeper verify windows
   pay off. Draft acceptance on the benchmark: 53-58% before, 64-79% after.
3. **The learned expert profile** (`--expert-profile-save` + `--expert-profile`, #477): the engine saves what
   the adaptive tier learned between requests and starts from it next time. Neutral at steady state, warmer
   starts.

## The benchmark

One 257,630-token prompt (the Strata repository's own source and docs, concatenated), sent through the HTTP
API, greedy, `reasoning_effort: none`, 256-token answers. "Cold" reads the whole prompt first (~2.5 min at
1,600-1,770 tok/s); "warm" reuses the prefix, which is how an agent session actually runs. Single runs -
the docs' ±20% noise applies; every post-tuning run landed between 38.8 and 53.5.

| Configuration | Cold | Warm (prefix reused) |
| --- | ---: | ---: |
| Stock settings (`--spec 4`, no calibration, shipped profile) | 17.5 | 17.2 |
| `--spec 6` + calibrated | 43.0 | 53.5 |
| + learned expert profile (final) | 45.5 | 47.0 |

Prompt reading: 1,468 → 1,744 tok/s. Real agent sessions on the same PC (mixed code, thinking on) ran
20-33 tok/s at 43-65K contexts before the tuning, with 68-84% draft acceptance - a workload the benchmark
understates.

## Why this works

Qwen3.8-Flash-Next routes each token to 10 of 512 experts per layer, so a 16 GB card never holds the model -
it holds the hot experts and streams the rest. Output speed is decided by how rarely a needed expert is
missed and how cheaply a miss is handled:

- **Hybrid-CPU scheduling** (`--pool-workers`): a missed expert is computed on the CPU; a window of tokens
  waits for its slowest worker. Threads on E-cores make every window pay the E-core's price, so fewer,
  faster workers win. The defaults were measured on a 6-core Ryzen with no E-cores.
- **Speculative depth matched to confidence** (`--spec` + `--spec-min-p`): the MTP draft layer proposes
  tokens that are checked in one pass. More drafts are only worth it when the drafter is sure; the
  calibrated floor keeps the acceptance rate up.
- **Compute instead of copy** (`--pcie-frac`): whether a missed expert should travel over PCIe or be
  computed in place depends on the link and the CPU. On PCIe 5 x16 with a 14700KF, in-place wins.
- **KV streaming**: the 256K KV cache (3.1 GiB, int8) lives in pinned RAM, a 32K-cell window in VRAM, so
  the rest of the card stays expert cache. This is why 256K context is possible at all here.
- **The expert cache and the adaptive tier** (the Fiddler-style ideas, already in the engine): hot experts
  stay resident (73-81% hit rate on this PC), next-layer experts stream in while attention runs, and the
  tier swaps in what the workload actually routes. Related art: [Fiddler (DAC 2025)](https://63dac.conference-program.com),
  KTransformers, HOBBIT.

## Reproducing

```
START-HERE.bat --setup --family qwen --model IQ3_S --context 262144 --vision gpu --yes   (or your sizes)
START-HERE.bat --calibrate                     # measures this PC, writes the three settings
```
then set `--spec 6` in `strata-<model>.json` and re-run `--calibrate` once so `--spec-min-p` is measured
with the deeper windows. Compare on your own prompts - a different PC will calibrate differently (a Ryzen
9600 or a PCIe 4 link will not want these exact values; that is the point of the tuner).

Caveat: a `--setup` re-run rewrites the config and drops a manually added `--expert-profile`/`--expert-profile-save`;
re-add them after, or keep them by hand. The calibrated three settings survive re-runs by themselves.
