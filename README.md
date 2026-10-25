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

24 GB M5 MacBook Air, Flash-Next Q2_0, 256K context, `--max-memory 17920M`:

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
  experts fit in memory. If you never send images, Splash's own `--language-only` does the
  same job without this flag.

## Build and run

### 1. Requirements

- An M3 or newer Mac, macOS 26.4+.
- Full Xcode (not just the Command Line Tools), opened once so its licence is accepted.
- Homebrew Splash **1.4.0**: `brew install incoai/tap/splash`, then check with
  `splash --version`. The fork has only been tested against 1.4.0; if Homebrew gives you a
  newer Splash, the fork's engine may not match it.

The fork reuses Homebrew Splash's Python server and installer, and your Homebrew install isn't
changed.

### 2. Build

```bash
git clone https://github.com/dlync/splash-24gb.git && cd splash-24gb
xcodebuild -downloadComponent MetalToolchain   # missing by default
make -j4                                       # ~3 min
```

### 3. Make a separate copy of Splash with the fork's engine

Run this from the `splash-24gb` folder. It's safe to re-run, and you should re-run it after
rebuilding or after `brew upgrade splash`:

```bash
FORK=~/splash-24gb-run
SRC="$(brew --prefix splash)/libexec" && [ -d "$SRC" ] && \
rm -rf "$FORK" && cp -R "$SRC" "$FORK" && \
cp build/splash build/splash.metallib build/host-step.metallib "$FORK/engine/" && \
mkdir -p "$FORK/bin" && \
printf '#!/bin/sh\nexport PYTHONDONTWRITEBYTECODE=1\nexec "%s/python/bin/python3" -u "%s/install/launcher.py" "$@"\n' \
  "$FORK" "$FORK" > "$FORK/bin/splash" && \
chmod +x "$FORK/bin/splash" && echo "OK: $FORK/bin/splash"
```

### 4. Run

Stop any stock Splash server first (both use port 8000), then:

```bash
SPLASH_ROUTE_BIAS=2 SPLASH_ROUTE_PROTECT=4 \
SPLASH_DESK_FIRST=1 SPLASH_DESK_OVERLAP=1 \
SPLASH_LAZY_VISION=1 \
~/splash-24gb-run/bin/splash serve --model ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF:Q2_0 \
  --max-memory 17920M --max-context 256K --max-cache-disk 16G
```

- **First run:** it downloads the model and its draft model (several GB). The downloads are
  shared with stock Splash, so you won't get a second copy if you already have them.
- **Check it's the fork:** at startup you should see `FORK: SPLASH_DESK_FIRST on` and
  `FORK: SPLASH_DESK_OVERLAP on`. If you don't, the fork engine isn't running.
- **Using it:** once it prints `Ready`, open <http://127.0.0.1:8000> or point any
  OpenAI-compatible app at it. Agents work the same as upstream, using the fork's launcher,
  e.g. `~/splash-24gb-run/bin/splash opencode`. Press Ctrl+C to stop.
- **`--max-cache-disk 16G`** lets the context spill to the SSD, which is what makes 256K
  possible in 24 GB. It uses up to 16 GB of disk.

All other Splash options and the API work as upstream. See
[README-upstream.md](README-upstream.md).

### Going back / uninstalling

Stock `splash` is untouched and works exactly as before. To remove the fork, delete
`~/splash-24gb-run` and the cloned `splash-24gb` folder.

## Flags

Every runtime feature is an environment variable and **off by default** (one build-time
setting is listed at the end). With none set, the output is the same as Splash 1.4.0.

| Variable | What it does |
|---|---|
| `SPLASH_ROUTE_BIAS=d` | Cache-aware routing: lowers the score of experts not in RAM by `d` (2 recommended). Routing weights still use the real scores. Changes outputs. |
| `SPLASH_ROUTE_PROTECT=n` | Used with `SPLASH_ROUTE_BIAS`: how many of the top experts are always picked normally (4 recommended; 6 if unset; 10 or more turns the preference off). |
| `SPLASH_ROUTE_BIAS_PREFILL=1` | Also apply the routing preference while reading the prompt (off by default). |
| `SPLASH_DESK_FIRST=1` + `SPLASH_DESK_OVERLAP=1` | Desk-first overlap ("desk-first": work on what's already on your desk while the rest is fetched). Set both for the speedup. Doesn't change outputs. |
| `SPLASH_LAZY_VISION=1` | Lazy vision. The first image request after a load is slower and needs ~0.85 GiB free; if that memory isn't free, the request waits until it is. |

Experimental and measurement flags (not needed for the speedup):

| Variable | What it does |
|---|---|
| `SPLASH_PREDICT_BIAS=0`, `SPLASH_PREDICT_TOPK=k`, `SPLASH_AHEAD_LIMIT=n` | Read-ahead tuning. Output unchanged, and no gain on top of desk-first. `SPLASH_PREDICT_BIAS=0` also costs ~50% more SSD reads. |
| `SPLASH_AHEAD_READERS`, `SPLASH_SYNC_PIECE_KIB` | Reader threads / read size. No gain found (my SSD peaks at ~6.5 GB/s). |
| `SPLASH_SPLIT_TAIL=n` | Runs the expert pass in two parts. Used to measure that cost. |
| `SPLASH_ROUTE_TRACE=file` | Expert hit/miss and SSD wait trace. **No size limit**: grows by megabytes per minute, so short runs only. |
| `SPLASH_HIDDEN_LOG=file`, `SPLASH_HIDDEN_LOG_MAX_MB` | Dumps model activations + chosen experts per step. 8 GiB per run by default, and it appends across runs. **Holds data derived from users' prompts. Don't use it on a shared server.** |
| `-DSPLASH_PREFILL_GROUP_CHUNKS=n` (build time) | Prompt-reading group size (default 4). |

## Limits

- One model, one machine (24 GB M5 MacBook Air), one Splash version (1.4.0). Your numbers will
  differ.
- Reading long prompts (~500 tok/s, GPU-bound) isn't faster. Splash's prefix cache makes
  later turns fast.
- Speed drops when the Mac heats up under long runs.

## License

Apache-2.0, the same as upstream (see [LICENSE](LICENSE)). This fork modifies files from
[incoai/splash](https://github.com/incoai/splash) 1.4.0. Changed code is marked with `FORK`
comments; the full list of changes is
[this comparison](https://github.com/incoai/splash/compare/1.4.0...dlync:splash-24gb:fork-main).
Some kernels are adapted from llama.cpp; see [third-party notices](THIRD_PARTY_NOTICES).
Model weights keep their own licences. Splash is a project of incoai.
