# Handoff: verify HTP weight-buffer/ubatch/ctx fix on IQ9 (geniex#1683) — round 2

Branch: `fix/htp-weight-buffer-mbuf` (HEAD now includes a second commit on top
of what you tested, `229d974c`). Issue:
https://github.com/qcom-ai-hub/geniex/issues/1683

Thanks for the round-1 report — it correctly showed the MBUF-only fix was
incomplete. I reproduced the exact failures you saw at the **raw
`llama-cli`/`llama-mtmd-cli` level** (already deployed at `/data/llama.cpp` on
the device) to root-cause them without burning more of your build/deploy
cycles, and landed a second commit. This doc describes what changed and what
to re-verify.

> Internal device access (SSH tunnel host, credentials, on-device paths) is
> intentionally **not** included here since this repo is public
> ([qualcomm/nexa-sdk](https://github.com/qualcomm/nexa-sdk)).

## Root cause found (bisected on-device with raw llama.cpp)

Two more HTP-specific limits beyond the weight-buffer issue the first commit
fixed:

1. **`n_ubatch` ceiling, independent of `n_batch`.** With `GGML_HEXAGON_MBUF=64`
   and `n_ctx=4096`, gemma-4-E4B fails above `n_ubatch=512` and Qwen3-ASR-1.7B
   fails above `n_ubatch=256` — regardless of how high `n_batch` is set
   (confirmed `n_ubatch=1024, n_batch=256` fails identically to
   `n_ubatch=1024, n_batch=2048`; `n_ubatch=512, n_batch=2048` passes). This
   is the bridge's own `n_ubatch=1024` NPU default — too high for both models.
2. **VLM's flat `n_ctx=16384` default is itself too large for HTP**, independent
   of `n_ubatch`: `-c 16384 -ub 256 -b 256` (an otherwise-safe ubatch) still
   fails. The bridge hardcoded 16384 for every VLM regardless of device.

Fix (`sdk/plugins/llama_cpp/src/params.cpp`, `vlm.cpp`):
- NPU `n_ubatch` default lowered from 1024 → 256 (the tighter of the two
  model-specific limits found, so it's safe for both).
- VLM's `n_ctx` default is now device-aware: 4096 on NPU (unchanged 16384 on
  CPU/GPU).

All three raw-llama.cpp repro commands below passed on IQ9 with these values
(`GGML_HEXAGON_MBUF=64`, `n_ctx=4096`, `n_ubatch=256`, `n_batch` unconstrained).

## Build (Linux x86_64 → arm64 Snapdragon cross-compile)

From the repo root, with Docker available:

```bash
git fetch origin fix/htp-weight-buffer-mbuf
git checkout fix/htp-weight-buffer-mbuf

# 1. Build + install the SDK bridge (produces sdk/pkg-geniex/)
docker run --rm -u $(id -u):$(id -g) \
    --volume $(pwd):/workspace \
    --workdir /workspace/sdk \
    -e CCACHE_DIR=/workspace/.ccache \
    --platform linux/amd64 \
    docker.io/qualcomm/geniex-toolchain-linux:v0.1.0 \
    bash -c 'cmake --preset arm64-linux-snapdragon-release -B build-linux . \
      && cmake --build build-linux -j \
      && cmake --install build-linux --prefix pkg-geniex'

# 2. Build the CLI + package it with the SDK artifact into one zip
bazelisk build --config=linux_arm64 //cli:artifact
# -> bazel-bin/cli/artifact.zip
```

See [notes/build.md](notes/build.md) for prerequisites (Bazelisk, Docker
registry login, `.ccache` dir) if this is a fresh checkout on that
workstation.

## Deploy to IQ9

```bash
unzip bazel-bin/cli/artifact.zip -d geniex-artifact
scp -r geniex-artifact <iq9>:/data/geniex-1683/
```

On-device, the extracted dir has `geniex` at the root alongside `libgeniex.so`
and a `llama_cpp/` subfolder (plugin + HTP skel libs). Run everything from
that root with:

```bash
cd /data/geniex-1683/geniex-artifact
export LD_LIBRARY_PATH=.
export GENIEX_PLUGIN_PATH=.
```

Model files (`gemma-4-E4B-it-Q4_0.gguf` + `mmproj-E4B-F16.gguf`,
`Qwen3-ASR-1.7B-Q8_0.gguf` + its mmproj) should already be staged on-device
from prior `#1677` testing — check before re-downloading, `/data` wipes on
device reset.

## What to re-test

Same deployment you already have at `/data/geniex-1683` — just rebuild from
the updated branch HEAD and redeploy:

```bash
git fetch origin fix/htp-weight-buffer-mbuf
git checkout fix/htp-weight-buffer-mbuf
git pull
```

Then rebuild (same two steps as before: docker SDK cross-compile +
`bazelisk build --config=linux_arm64 //cli:artifact`) and redeploy
`artifact.zip` the same way.

Re-run the same 3-model matrix (`gemma4-e4b`, `gemma-4-E2B`, `qwen3-asr`),
already registered via `geniex pull`, with no env overrides — rely on the new
defaults:

```bash
./geniex infer gemma4-e4b --compute npu --log debug \
  -p '/data/gguf/test-audio.mp3 Transcribe this audio verbatim.'
./geniex infer qwen3-asr --compute npu --log debug \
  -p '/data/gguf/test-audio.mp3 Transcribe this audio verbatim.'
./geniex infer gemma-4-e2b --compute npu --log debug \
  -p '/data/gguf/test-audio.mp3 Transcribe this audio verbatim.'
```

Please specifically check:

1. **gemma4-e4b now passes** (this was the main target of #1683).
2. **qwen3-asr** — your round-1 report found this failing even at baseline
   (`MBUF=1024`), which if still true on this unit predates this branch
   entirely. Worth retesting since the new `n_ubatch=256` default might
   incidentally fix it (Qwen3-ASR's own raw-llama.cpp repro needed exactly
   `n_ubatch=256`) — if it still fails, it's very likely the pre-existing,
   unit-specific issue you flagged, not something in scope here.
3. **gemma-4-E2B** — confirm still no regression (decode tok/s may dip
   slightly since `n_ubatch` dropped from 1024→256, affecting prefill chunking
   only; decode itself processes one token at a time so should be largely
   unaffected).
4. **The SIGABRT-on-teardown you found** is in vendored `third-party/llama.cpp`
   (`llama_free()`/`mtmd_free()` during a failed-generate teardown) — this repo's
   [CLAUDE.md](CLAUDE.md) forbids editing third-party code directly, so it's out
   of scope for this branch. If qwen3-asr now passes outright with the new
   defaults, you likely won't hit that path at all for these 3 models. If it's
   still reachable (e.g. qwen3-asr still fails pre-existing), please capture a
   fresh repro + full backtrace so we can decide whether to patch it via
   `sdk/patches/` or file upstream.

## Build/deploy notes you shared (kept for next time)

- Use local disk for `--output_user_root` (NFS home breaks the Bazel JVM server).
- `bazelisk` may need `~/.local/bin` on `PATH`.
- `geniex pull --local-path` needs real regular files, not symlinks — hardlink
  instead.
- `geniex infer` has no `--audio`/`--image` flag — embed the file path directly
  in `-p`, e.g. `-p '/data/gguf/test-audio.mp3 Transcribe this audio verbatim.'`.

## Report back

For each of the 3 models: pass/fail, any `fastrpc_mmap`/buffer-mapping log
lines, and decode tok/s (from the `--log debug` output) so we can compare
against both the round-1 and original pre-fix baselines already on file.
