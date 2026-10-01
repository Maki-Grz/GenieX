# Handoff: verify HTP weight-buffer MBUF fix on IQ9 (geniex#1683)

Branch: `fix/htp-weight-buffer-mbuf` (based on `fix/audio-mmproj-htp-session`).
Issue: https://github.com/qcom-ai-hub/geniex/issues/1683

> Internal device access (SSH tunnel host, credentials, on-device paths) is
> intentionally **not** included here since this repo is public
> ([qualcomm/nexa-sdk](https://github.com/qualcomm/nexa-sdk)). Use your
> existing IQ9/QDC access the same way you have for prior `#1677`/`#1683`
> testing.

## What changed

[sdk/plugins/llama_cpp/src/plugin.cpp](sdk/plugins/llama_cpp/src/plugin.cpp)
now defaults `GGML_HEXAGON_MBUF=64` (MiB) unless the caller already set it.
This caps the max contiguous host-buffer mmap size `ggml-hexagon` will
request, so large model-weight buffers (e.g. gemma-4-E4B's ~1018 MiB) get
chunked instead of requiring one ~1 GiB contiguous fastrpc/CMA allocation —
which is what was failing with `ggml-hex: HTP0 buffer mapping failed` /
`fastrpc_mmap failed` on IQ9, independent of the mmproj/LM session-contention
issue `#1677` already fixed.

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

## Test matrix

Register each local model once (`--local-path` must point at a dir holding
exactly one gguf+mmproj pair):

```bash
./geniex pull gemma4-e4b --local-path /data/gguf/e4b --model-hub localfs --model-type vlm
./geniex pull qwen3-asr --local-path /data/gguf/qwen3-asr --model-hub localfs --model-type vlm
```

Then, for each model:

```bash
./geniex infer gemma4-e4b --compute npu --log debug \
  --audio /data/gguf/test-audio.mp3 -p "Transcribe this audio verbatim."
./geniex infer qwen3-asr --compute npu --log debug \
  --audio /data/gguf/test-audio.mp3 -p "Transcribe this audio verbatim."
```

Check:

1. **Before this fix** (`GGML_HEXAGON_MBUF=1024 ./geniex infer ...` as a
   negative control, or test against the parent branch
   `fix/audio-mmproj-htp-session`): expect
   `ggml-hex: HTP0 buffer mapping failed` / `fastrpc_mmap failed` for
   gemma4-e4b specifically (mmproj already loads fine on its own HTP
   session; it's the LM's own weight buffer that fails).
2. **After this fix** (no override, branch `fix/htp-weight-buffer-mbuf`):
   gemma4-e4b should load and produce a correct transcription, same as
   `qwen3-asr` already does.
3. **No regression**: re-run gemma-4-E2B and Qwen3-ASR-1.7B (which worked
   *without* any MBUF override before) to confirm the new default-64 MiB
   cap doesn't break or meaningfully slow them down.

## Report back

For each of the 3 models: pass/fail, any `fastrpc_mmap`/buffer-mapping log
lines, and decode tok/s (from the `--log debug` output) so we can compare
against the pre-fix baseline already on file.
