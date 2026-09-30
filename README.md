# Independent Bench: MLX-Serve 26.9.7-dev · Qwen 3.8 Flash-Next mixed 4/8-bit · Mac Studio M5 Ultra

Independent llmprobe run of [mlx-serve](https://github.com/ddalcu/mlx-serve) built from `main` on 2026-09-30, serving [`ddalcu/Qwen3.8-Flash-Next-MLX-Serve-mixed-4-8bit`](https://huggingface.co/ddalcu/Qwen3.8-Flash-Next-MLX-Serve-mixed-4-8bit) (125B-A6B MoE) on a single Mac Studio M5 Ultra 256 GB. Context sweep from 2k to 256k, with a 4-stream concurrency burst at every rung.

**Full credit to [ddalcu](https://github.com/ddalcu)**: mlx-serve, the MLX-Serve app and the Flash-Next mixed 4/8-bit pack are all ddalcu's work. This repo only publishes an independent measurement of it. Dev build, not a release.

## Setup

| | |
|---|---|
| Machine | Mac Studio M5 Ultra (Mac17,15), 256 GB, macOS 27.0.1 |
| Engine | mlx-serve **26.9.7-dev**, commit [`933566c`](https://github.com/ddalcu/mlx-serve/commit/933566c), ReleaseFast |
| Stack | MLX 0.32.3 · mlx-c `56b2d39` · NAX on (M5 neural accelerators) · Xcode 27.0 / Metal Toolchain 27A266a |
| Model | Qwen 3.8 Flash-Next, MLX-Serve mixed 4/8-bit pack (100 GiB) |
| Server flags | `--serve --host 127.0.0.1 --port 11234 --max-concurrent 4` (ctx auto = 262,144) |
| Engine features | KV bf16 · MTP exact · PLD · batched decode |
| Harness | [llmprobe](https://www.npmjs.com/package/llmprobe) 0.6.13, `--bench-only` |
| Sampling | greedy (t=0), thinking on, reasoning medium |
| Runs | warmup + median of 3 per scenario; decode forced to 192 tokens |
| Rungs | `--rungs 2k,4k,8k,16k,32k,64k,128k,250k --concurrency 4` (custom, not comparable to default llmprobe runs) |

## Headline

| Metric | Result |
|---|---|
| Decode | **154.5 tok/s** (143.6–165.8) |
| Prefill (10,404-token prompt) | **4,183 tok/s** |
| TTFT (short prompt) | **174 ms** |
| Speculative decode | **2.3×** (predictable 277.4 · novel 120.5 tok/s · 7.84 tok/step) |
| Prefix cache | 5.9× (546 ms cold → 93 ms warm) |
| Concurrency ×4 (short prompts) | 166.8 tok/s aggregate vs 119.1 alone (0.35 efficiency) |
| Sustained | 154.5 → 151.9 tok/s over 12m 39s (−1.7%, steady) |

## Context sweep

| Context | Decode tok/s | Prefill tok/s | TTFT | tok/step | ×4 total tok/s | ×4 each tok/s | ×4 worst TTFT |
|---|---|---|---|---|---|---|---|
| 2k | 153.2 | 3,274 | 0.6 s | 3.43 | 123.7 | 45.6 | 2.6 s |
| 4k | 152.6 | 2,210 | 1.8 s | 3.62 | 76.7 | 34.4 | 5.9 s |
| 8k | 153.9 | 4,046 | 2.0 s | 3.69 | 57.3 | 31.0 | 10.2 s |
| 16k | 147.6 | 4,351 | 3.7 s | 3.49 | 36.5 | 21.8 | 17.7 s |
| 33k | 137.2 | 4,334 | 7.6 s | 3.25 | 14.3 | 13.9 | 50.7 s |
| 66k | 150.4 | 4,338 | 15.1 s | 3.84 | 11.5 | 16.1 | 65.4 s |
| 131k | 136.7 | 4,262 | 30.8 s | 3.15 | 5.9 | 17.0 | 128.8 s |
| **256k** | **114.4** | **4,114** | **62.3 s** | 3.56 | 2.9 | 12.5 | 261.5 s |

## Observations

- Decode holds ~150 tok/s through 66k and is still 114 tok/s at 256k (−26%).
- Prefill is flat at ~4.1–4.35k tok/s across the whole ladder, so TTFT scales linearly: ~1 minute at 256k.
- Long-context ×4 bursts are marked `serialized` by llmprobe (0.17–0.31 efficiency): aggregate throughput under 4 concurrent long prompts is lower than a single stream. Short-prompt concurrency does scale (0.35 efficiency, 1.4× aggregate).
- For reference only, mlx-serve's own [`benchmarks.md`](https://github.com/ddalcu/mlx-serve/blob/main/benchmarks.md) lists 88 tok/s (MTP) for this model at 26.9.7 on an M4 Max 128 GB. Different hardware and llmprobe settings, so not a like-for-like comparison.

## Caveat

Every context rung reported `answer never used RETRY_BUDGET_MS — context may be unread`. With thinking on and decode capped at 192 tokens, the answer likely never got past the reasoning phase to cite the planted value, so this run does not verify long-context recall. Speed numbers are unaffected.

## Files

- [`raw/llmprobe-output.txt`](raw/llmprobe-output.txt): full console output
- [`raw/llmprobe-report.json`](raw/llmprobe-report.json): llmprobe run record
- [`raw/llmprobe-report-card.html`](raw/llmprobe-report-card.html): llmprobe report card (download and open locally)

## Recipe: re-create for validation

Pinned inputs:

| Input | Pin |
|---|---|
| mlx-serve | `933566c4107f34d459652cd24ca0fd0f4bbf68c6` (main, 2026-09-30) |
| Model | `ddalcu/Qwen3.8-Flash-Next-MLX-Serve-mixed-4-8bit` @ `7eaef0fa82b4c3bf5c64cec60ace4bf48fd271e3` |
| Zig | `0.17.0-dev.2248+3f6a02acd` (staged by `scripts/fetch-zig.sh`) |
| Toolchain | Xcode 27.0 (27A266a) + Metal Toolchain, cmake 4.4, libwebp 1.6.0 |
| Harness | llmprobe 0.6.13 on Node 24 |

### 1. Prerequisites (once)

```bash
# Xcode 27 installed in /Applications; accept its license and fetch the Metal compiler
sudo DEVELOPER_DIR=/Applications/Xcode.app/Contents/Developer xcodebuild -license accept
export DEVELOPER_DIR=/Applications/Xcode.app/Contents/Developer
xcodebuild -downloadComponent MetalToolchain
xcrun -sdk macosx metal --version          # must print a version
brew install cmake webp
```

### 2. Build mlx-serve at the pinned commit

```bash
git clone --recurse-submodules https://github.com/ddalcu/mlx-serve && cd mlx-serve
git checkout 933566c4107f34d459652cd24ca0fd0f4bbf68c6
git submodule update --init --recursive
./scripts/fetch-zig.sh && export PATH="$PWD/.zig-toolchain:$PATH"
./scripts/fetch-llama.sh && ./scripts/build-mlx.sh   # expect "NAX symbol hits" in the output on M5
zig build -Doptimize=ReleaseFast                     # never Debug: 2-4x slower
./zig-out/bin/mlx-serve --version                    # expect 26.9.7-dev · mlx 0.32.3 · "nax on" on M5
```

### 3. Fetch the model at the pinned revision

```bash
huggingface-cli download ddalcu/Qwen3.8-Flash-Next-MLX-Serve-mixed-4-8bit \
  --revision 7eaef0fa82b4c3bf5c64cec60ace4bf48fd271e3 \
  --local-dir ~/.mlx-serve/models/ddalcu/Qwen3.8-Flash-Next-MLX-Serve-mixed-4-8bit
# or: ./zig-out/bin/mlx-serve pull ddalcu/Qwen3.8-Flash-Next-MLX-Serve-mixed-4-8bit (latest revision)
du -sh ~/.mlx-serve/models/ddalcu/Qwen3.8-Flash-Next-MLX-Serve-mixed-4-8bit   # ~101G
```

### 4. Serve

Quiet machine: no other model servers loaded, no builds or large downloads running.

```bash
./zig-out/bin/mlx-serve \
  --model ~/.mlx-serve/models/ddalcu/Qwen3.8-Flash-Next-MLX-Serve-mixed-4-8bit \
  --serve --host 127.0.0.1 --port 11234 --max-concurrent 4
# log should show: "Concurrency: --max-concurrent=4, batched decode on" and "Context size: 262144"
```

### 5. Bench

```bash
npx -y llmprobe@0.6.13 127.0.0.1:11234 \
  -m Qwen3.8-Flash-Next-MLX-Serve-mixed-4-8bit --bench-only \
  --rungs 2k,4k,8k,16k,32k,64k,128k,250k --concurrency 4 \
  --label "Flash-Next mixed4-8 | mlx-serve 26.9.7-dev 933566c | <your machine>" --no-color \
  | tee llmprobe-output.txt
```

Takes about 13 minutes on an M5 Ultra and sends ~3.16M prompt tokens across 76 requests. Compare your `PERFORMANCE` block against [`raw/llmprobe-output.txt`](raw/llmprobe-output.txt). On an M5 Ultra, expect decode within a few percent of 154 tok/s and prefill around 4.1-4.3k tok/s. Other chips will differ. Please open an issue with your numbers and machine.

## Credits

- [**ddalcu**](https://github.com/ddalcu): [mlx-serve](https://github.com/ddalcu/mlx-serve), the MLX-Serve app, the M5 Ultra support and Flash-Next kernels in this build, and the [Flash-Next mixed 4/8-bit pack](https://huggingface.co/ddalcu/Qwen3.8-Flash-Next-MLX-Serve-mixed-4-8bit). All of the performance here is their engineering.
- Qwen team: Qwen 3.8 Flash-Next.
- [llmprobe](https://www.npmjs.com/package/llmprobe): benchmark harness.
- Apple MLX: the underlying array framework.
