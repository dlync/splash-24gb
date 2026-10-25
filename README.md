# splash-24gb

**A fork of [Splash](https://github.com/incoai/splash) that makes Qwen3.8-Flash-Next decode
faster on a 24 GB Mac: about 30 → 49 tok/s on a MacBook Air.**

Flash-Next is too big for a 24 GB Mac's memory, so Splash keeps only some of the model's
experts in RAM and reads the rest from the SSD while it generates. This fork cuts down how
often it waits on the SSD.

> Tested on one model only: `ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF:Q2_0`. Other models
> haven't been tested and may get no benefit.
> Not affiliated with or endorsed by incoai. The original Splash README is kept as
> [README-upstream.md](README-upstream.md).

## Results

24 GB MacBook Air, Flash-Next Q2_0, 256K context, `--max-memory 17920M`:

| Engine | Decode tok/s | Expert hit rate |
|---|---|---|
| Stock Splash 1.4.0 | ~29–30 | 0.83 |
| + cache-aware routing | ~42 | 0.95 |
| + desk-first overlap | **~49** (45 in a real 10-turn agent session) | 0.95 |

Decode speed is the average over code / tool-call / explanation prompts at temp 0.7, thinking
off, with the machine cool. Only ~6,800 of the model's 24,576 experts fit in RAM.

## How it works

- **Cache-aware routing.** Each token picks 10 experts per layer. The top few are chosen
  exactly as normal. For the rest, experts already in RAM get a small preference, so the model
  needs fewer SSD reads. This changes the exact words the model writes. On a small private
  test set (code with hidden tests, rare facts, math, tool calls) it scored the same as stock,
  but that's a smoke test, not a full benchmark.
- **Desk-first overlap.** In each layer, the GPU starts on the experts already in RAM while the
  missing ones are still loading from the SSD. This doesn't change the output (greedy output is
  token-identical to the engine with it off). It hides about 70% of the SSD waiting.
- **Lazy vision.** Keeps the image encoder out of RAM until the first image request, so more
  experts fit in memory.

## Build and run

Needs an M3 or newer Mac, macOS 26.4+, Xcode, and Homebrew Splash 1.4.0 installed
(`brew install incoai/tap/splash`). The fork reuses its Python server and installer.

```bash
git clone https://github.com/dlync/splash-24gb.git && cd splash-24gb
xcodebuild -downloadComponent MetalToolchain   # missing by default
make -j4                                       # ~3 min
```

Make your own copy of Homebrew's Splash with the fork's engine in it (this leaves the Homebrew
install untouched; the three engine files always go together):

```bash
FORK=~/splash-24gb-run
cp -R "$(brew --prefix splash)/libexec" "$FORK"
cp build/splash build/splash.metallib build/host-step.metallib "$FORK/engine/"
mkdir -p "$FORK/bin"
printf '#!/bin/sh\nexport PYTHONDONTWRITEBYTECODE=1\nexec "%s/python/bin/python3" -u "%s/install/launcher.py" "$@"\n' \
  "$(brew --prefix splash)/libexec" "$FORK" > "$FORK/bin/splash"
chmod +x "$FORK/bin/splash"
```

Then serve with the recommended flags:

```bash
SPLASH_ROUTE_BIAS=2 SPLASH_ROUTE_PROTECT=4 \
SPLASH_DESK_FIRST=1 SPLASH_DESK_OVERLAP=1 \
SPLASH_LAZY_VISION=1 \
~/splash-24gb-run/bin/splash serve --model ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF:Q2_0 \
  --max-memory 17920M
```

All other Splash options and the API work as upstream. See
[README-upstream.md](README-upstream.md).

## Flags

Every feature is behind an environment variable and **off by default**. With none set, the
output is the same as Splash 1.4.0.

| Variable | What it does |
|---|---|
| `SPLASH_ROUTE_BIAS=δ` | Cache-aware routing: lowers the score of experts not in RAM by `δ` (2 recommended). Routing weights still use the real scores. Changes outputs. |
| `SPLASH_ROUTE_PROTECT=n` | How many of the top experts are always picked normally (4 recommended; 6 if unset). |
| `SPLASH_ROUTE_BIAS_PREFILL=1` | Also apply the routing preference while reading the prompt (off by default). |
| `SPLASH_DESK_FIRST=1` + `SPLASH_DESK_OVERLAP=1` | Desk-first overlap. Needs both. Doesn't change outputs. |
| `SPLASH_LAZY_VISION=1` | Lazy vision. The first image request after a load is slower and needs ~0.85 GiB free. |

Experimental and measurement flags (not needed for the speedup):

| Variable | What it does |
|---|---|
| `SPLASH_PREDICT_BIAS=0`, `SPLASH_PREDICT_TOPK=k`, `SPLASH_AHEAD_LIMIT=n` | Read-ahead tuning. Output unchanged, no gain on top of desk-first, ~50% more SSD reads. |
| `SPLASH_AHEAD_READERS`, `SPLASH_SYNC_PIECE_KIB` | Reader threads / read size. No gain found (my SSD peaks at ~6.5 GB/s). |
| `SPLASH_SPLIT_TAIL=n` | Runs the expert pass in two parts. Used to measure that cost. |
| `SPLASH_ROUTE_TRACE=file` | Expert hit/miss and SSD wait trace. **No size limit**: grows by megabytes per minute, so short runs only. |
| `SPLASH_HIDDEN_LOG=file`, `SPLASH_HIDDEN_LOG_MAX_MB` | Dumps model activations + chosen experts per step. 8 GiB per run by default, and it appends across runs. **Holds data derived from users' prompts. Don't use it on a shared server.** |
| `-DSPLASH_PREFILL_GROUP_CHUNKS=n` (build time) | Prompt-reading group size (default 4). |

## Limits

- One model, one machine (24 GB M-series MacBook Air). Your numbers will differ.
- Reading long prompts (~500 tok/s, GPU-bound) isn't faster. Splash's prefix cache makes
  later turns fast.
- Speed drops when the Mac heats up under long runs.

## License

Apache-2.0, the same as upstream (see [LICENSE](LICENSE)). This fork modifies files from
[incoai/splash](https://github.com/incoai/splash) 1.4.0. Changed code is marked with `FORK`
comments, and `git diff 1.4.0` shows every change. Splash is a project of incoai.
