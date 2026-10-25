# splash-24gb: faster expert streaming for Splash on small Macs

splash-24gb is a fork of [incoai/splash](https://github.com/incoai/splash) 1.4.0. It speeds up
decode for large MoE models whose experts don't fit in RAM and are streamed from the SSD.
It was built and tested on a **24 GB MacBook Air** running
`ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF:Q2_0` with the full 256K context. The name comes from the test machine: it should help any Mac where the model is too big for RAM.

Every new feature is behind an environment variable and **off by default**. With no variables
set, the output is the same as upstream 1.4.0. Some internals did change (kernel parameter
structs, slot-table bookkeeping), so the engine binary and its two `.metallib` files must be
replaced together. Don't mix them with upstream files.

## Results (24 GB MacBook Air, Flash-Next Q2_0, 256K context, `--max-memory 17920M`)

| Engine | Decode tok/s | Expert hit rate |
|---|---|---|
| Stock Splash 1.4.0 | ~29–30 | 0.83 |
| + cache-aware routing | ~42 | 0.95 |
| + desk-first overlap | **~49** (45 in a real 10-turn agent session) | 0.95 |

Decode was averaged over code / tool-call / explanation prompts at temp 0.7, thinking off,
with the machine cool (nominal thermal state). Only ~6,800 of the model's 24,576 experts fit
in RAM, so decode is limited by expert reads from the SSD.

## The flags

### Recommended

```bash
SPLASH_ROUTE_BIAS=2 SPLASH_ROUTE_PROTECT=4 \
SPLASH_DESK_FIRST=1 SPLASH_DESK_OVERLAP=1 \
splash serve --model ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF:Q2_0 --max-memory 17920M ...
```

- **`SPLASH_ROUTE_BIAS=δ`, `SPLASH_ROUTE_PROTECT=n`: cache-aware routing.** Each token
  routes to 10 experts per layer. The top `n` are picked exactly as stock. For the remaining
  picks, experts that are *not* already in RAM get their router score lowered by `δ`, so ties
  go to experts that are already loaded. The routing *weights* still come from the raw scores,
  and prefill is unbiased (`SPLASH_ROUTE_BIAS_PREFILL` changes that). Effect: fewer SSD reads.
  On a small private test set (code with hidden tests, rare-fact recall, math, tool calls)
  δ=2 / n=4 scored the same as stock. That's a smoke test, not a full benchmark. **This flag changes outputs:** they are equivalent in my
  tests but not token-identical, and they depend on what's in the expert cache.
- **`SPLASH_DESK_FIRST=1` + `SPLASH_DESK_OVERLAP=1`: desk-first overlap.** In each MoE layer
  during decode, the GPU computes the experts that are already in RAM while the CPU reads the
  missing ones from the SSD, then it computes those. It needs both flags. **Output is
  token-identical** to the flags-off engine (greedy, `--verify-rows all`, at matched prefill
  chunking). In traces it hides about 70% of the SSD wait.
- **`SPLASH_LAZY_VISION=1`.** Keeps the vision tower out of memory until the first image
  request, so expert slots are planned as if the model were text-only. Use it when serving
  without `--language-only`. The first image request after a load is slower (the tower is
  read back in), and it needs free memory for the tower (~0.85 GiB for Flash-Next).

### Experimental / measurement (off by default)

| Variable | What it does |
|---|---|
| `SPLASH_PREDICT_BIAS=0`, `SPLASH_PREDICT_TOPK=k`, `SPLASH_AHEAD_LIMIT=n` | Read-ahead tuning for the next block. Output is unchanged, but they add nothing measurable on top of desk-first and cost about 50% more SSD reads. |
| `SPLASH_AHEAD_READERS`, `SPLASH_SYNC_PIECE_KIB` | Reader thread count / read piece size. No gain found (the SSD peaks at ~6.5 GB/s). |
| `SPLASH_SPLIT_TAIL=n` | Splits the expert pass into two tile ranges (bit-identical). Used to measure the split cost. |
| `SPLASH_ROUTE_TRACE=file` | Per-load expert hit/miss and SSD wait trace. **No size limit**: it grows by megabytes per minute of decoding, so use it for short runs only. |
| `SPLASH_HIDDEN_LOG=file`, `SPLASH_HIDDEN_LOG_MAX_MB` | Dumps MoE inputs + routed experts per decode step (for training predictors). Capped at 8 GiB per run by default, but it appends, so repeated runs keep growing the file. **It holds data derived from users' prompts. Don't enable it on a shared server.** |

## Building

Needs Apple silicon, macOS 26.4+, Xcode, and an installed Homebrew Splash 1.4.0 (for the Python
server and installer).

```bash
xcodebuild -downloadComponent MetalToolchain   # missing by default
make -j4                                       # ~3 min; outputs build/splash + metallibs
```

To run it, copy Homebrew's Splash (`$(brew --prefix splash)/libexec`) to a directory of your own,
then replace its engine with `build/splash`, `build/splash.metallib` and
`build/host-step.metallib`. Don't modify the Homebrew install itself.

## Caveats

- Tested on one machine and one model. Other streamed MoE models should benefit, but I
  haven't measured them.
- Long-prompt prefill (~500 tok/s) is GPU-bound and unchanged by this fork.

## License

Apache-2.0, the same as upstream (see [LICENSE](LICENSE)). This fork modifies files from
[incoai/splash](https://github.com/incoai/splash) 1.4.0; changed code is marked with `FORK`
comments, and `git diff 1.4.0` shows every change. Splash is a project of incoai. This fork
is not affiliated with or endorsed by incoai.
