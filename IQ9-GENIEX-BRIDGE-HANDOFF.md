# geniex-bridge audio/HTP-session fix — handoff to Linux x64 workstation

Branch: `fix/audio-mmproj-htp-session` (already pushed/available — pull it before building).
Goal: build the Linux ARM64 Snapdragon SDK bridge twice (before/after the fix already committed
on this branch — see below for isolating the "before" state), deploy to the IQ9 (QCS9075 EVK)
device, and confirm 3 audio models work end-to-end through the **geniex bridge / CLI**, not raw
`llama-mtmd-cli`.

## Why this handoff exists

Building on the Windows laptop cross-compiles through `docker.io/qualcomm/geniex-toolchain-linux:v0.1.0`
with `--platform linux/amd64`. That image has no `arm64` variant, and this laptop is **ARM64
Windows**, so the entire build (abseil-cpp, sentencepiece, every Rust crate, ggml/llama.cpp, the
Hexagon HTP skels) runs under **QEMU x86_64-on-ARM64 emulation** — extremely slow (still <30% after
~40 min). Your workstation is a native x64 Linux box, so the same container runs natively.

## 1. One-time local environment fixes (apply these before building, not optional)

### CRLF breaks the llama.cpp submodule patches
If this is a fresh clone on a machine with `core.autocrlf=true` (mostly a Windows thing, but
double-check), `sdk/CMakeLists.txt`'s `git apply` of `sdk/patches/*.patch` against
`third-party/llama.cpp` will fail with "patch does not apply" even though the content matches,
because the patch is LF and the checked-out submodule file is CRLF. On a stock Linux workstation
this is very unlikely to bite you, but if it does:
```bash
git config core.autocrlf false
git -C third-party/llama.cpp config core.autocrlf false
git -C third-party/geniex-qairt config core.autocrlf false
git -C third-party/llama.cpp rm --cached -r . -q && git -C third-party/llama.cpp reset --hard HEAD
```

### Docker registry auth
`docker pull docker.io/qualcomm/geniex-toolchain-linux:v0.1.0` goes through the corporate
registry mirror and needs `docker login` first (internal Qualcomm credentials) — I won't run
this for you, just confirm you're logged in.

### Corporate TLS MITM breaks in-container `git clone` (CMake `FetchContent`)
Configuring the SDK triggers a `FetchContent_Populate` of `abseil-cpp` (via sentencepiece, via
the QAIRT plugin), which `git clone`s from github.com **inside the container**. Behind the
corporate proxy this fails with `server verification failed: certificate signer not trusted` —
the container's public CA bundle doesn't include the corp MITM root. Mount your corp CA bundle
and point git/curl at it:
```bash
docker run --rm \
    --volume "$(pwd)":/workspace \
    --volume /path/to/your/corp-ca-bundle.pem:/certs/corp-ca-bundle.pem:ro \
    --workdir /workspace/sdk \
    -e CCACHE_DIR=/workspace/.ccache \
    -e GIT_SSL_CAINFO=/certs/corp-ca-bundle.pem \
    -e SSL_CERT_FILE=/certs/corp-ca-bundle.pem \
    -e CURL_CA_BUNDLE=/certs/corp-ca-bundle.pem \
    docker.io/qualcomm/geniex-toolchain-linux:v0.1.0 \
    bash -c '...'
```
(On a native Linux Docker host you may not need `--platform linux/amd64` at all if the engine is
already amd64 — drop it; it was only needed to force emulation on the ARM64 Windows laptop.)

## 2. Submodules

Unchanged from `origin/main`'s recorded pins — just `git submodule update --init --recursive`
after checking out the branch. Don't run `--remote`; that bumps to each submodule's own latest
upstream tip, not what GenieX's main actually pins:
- `third-party/llama.cpp` → `94256114c229674ef96e76eb2dea596e65b43818`
- `third-party/geniex-qairt` → `43ac75e41b6b448ce007fdd4181b1a7111fbbeaf` (incl. nested
  `tokenizers-cpp/msgpack`, `tokenizers-cpp/sentencepiece`)

## 3. The fix (already committed on this branch — read before re-deriving it)

`sdk/plugins/llama_cpp/`:
- **`src/vlm.cpp`** (`LlamaVlm::create`): when no explicit `vit_device_id` override is given, the
  mmproj (vision/audio encoder) used to default to `selection->front()` — the **same** HTP
  session as the main LM. On IQ9 that hits a `fastrpc_mmap`/session-contention failure as soon as
  the encoder allocates its own compute buffer (see
  `C:\Users\zhic\code\iq9\IQ9-AUDIO-HTP-HANDOFF.md` for the raw llama.cpp-level repro/fix this
  mirrors: `--device HTP0 -mmdev HTP1`). Now calls the new `resolve_vision_device(*selection)`.
- **`include/params.h` / `src/params.cpp`**: new `resolve_vision_device()` — picks a distinct HTP
  device for the mmproj when the LM is on HTP, falling back to sharing the LM's device (with a
  warning) only if no second HTP session is registered.
- **`src/plugin.cpp`** (`LlamaPlugin` ctor): defaults `GGML_HEXAGON_DEVICES=2` (only if the
  caller/environment hasn't already set `GGML_HEXAGON_DEVICES`/`GGML_HEXAGON_NDEV`) so a second
  virtual HTP session actually exists for `resolve_vision_device()` to find. Must happen before
  anything touches the HTP backend registry.

Full diff: `git diff <base>..fix/audio-mmproj-htp-session -- sdk/plugins/llama_cpp`.

## 4. What to build

Linux ARM64 Snapdragon release preset (matches the IQ9 target):
```bash
cd sdk
docker run --rm \
    --volume "$(pwd)/..":/workspace \
    --volume /path/to/corp-ca-bundle.pem:/certs/corp-ca-bundle.pem:ro \
    --workdir /workspace/sdk \
    -e CCACHE_DIR=/workspace/.ccache \
    -e GIT_SSL_CAINFO=/certs/corp-ca-bundle.pem -e SSL_CERT_FILE=/certs/corp-ca-bundle.pem -e CURL_CA_BUNDLE=/certs/corp-ca-bundle.pem \
    docker.io/qualcomm/geniex-toolchain-linux:v0.1.0 \
    bash -c 'cmake --preset arm64-linux-snapdragon-release -B build-linux . \
      && cmake --build build-linux -j \
      && cmake --install build-linux --prefix pkg-geniex'
```

### Build TWICE — once before the fix, once after

To actually demonstrate the regression/fix (per the original ask: reproduce the fastrpc issue
first, then confirm the fix resolves it):

1. **Baseline ("before") build**: `git stash` the 4 files under `sdk/plugins/llama_cpp/` (or
   `git checkout <base-commit> -- sdk/plugins/llama_cpp`) so `vision_device = selection->front()`
   and there's no `GGML_HEXAGON_DEVICES` default — i.e. the pre-fix behavior — then build and
   install to e.g. `pkg-geniex-before/`.
2. **Fixed ("after") build**: `git stash pop` (or re-checkout the branch tip), build again to
   `pkg-geniex/` (or any distinct prefix).

Keep both `pkg-geniex-before/` and `pkg-geniex/` around — you'll deploy each to the device in
turn.

### Also build the CLI (for `geniex infer`)

The CLI links against whichever `sdk/pkg-geniex/` is currently installed (local-SDK mode). Build
it with Bazel targeting `linux_arm64` — see [notes/build.md](notes/build.md) `Build and run the
CLI` section and the `--config=linux_arm64` flag. You'll need Bazelisk in the same container or
natively on the workstation (Bazel doesn't need the toolchain container — it fetches its own
cross toolchain for Go/CGO).

## 5. Deploying + testing on the IQ9 device

Device access: see `C:\Users\zhic\code\iq9\IQ9-AUDIO-HTP-HANDOFF.md` section "Connecting to the
device" — SSH tunnel + `iq9.py` helper (`run`/`script`/`put`/`get`). **The device wipes `/data` on
reset** — always check `ls /data` before assuming anything survived.

Models already fetched to `/data/gguf/` on a prior session (re-pull if the device reset):
- `gemma-4-E2B-it-Q4_0.gguf` + `mmproj-F16.gguf`
- `gemma-4-E4B-it-Q4_0.gguf` + `mmproj-E4B-F16.gguf`
- `Qwen3-ASR-1.7B-Q8_0.gguf` + `mmproj-Qwen3-ASR-1.7B-Q8_0.gguf`
- `test-audio.mp3` sample clip

Deploy whichever `pkg-geniex*/` + the Bazel-built `geniex` CLI binary to e.g. `/data/geniex/`,
then run through the **CLI** (not `llama-mtmd-cli` directly — that's the already-confirmed raw
llama.cpp path from the other handoff doc; this task is validating the **bridge**):

```bash
export LD_LIBRARY_PATH=/data/geniex/lib:/data/geniex/lib/llama_cpp
export ADSP_LIBRARY_PATH=/data/geniex/lib/llama_cpp
export GENIEX_PLUGIN_PATH=/data/geniex/lib
./geniex infer --model-path /data/gguf/gemma-4-E2B-it-Q4_0.gguf \
    --mmproj-path /data/gguf/mmproj-F16.gguf \
    --compute npu \
    -p "Transcribe this audio verbatim. Audio: /data/gguf/test-audio.mp3"
```
(Check `geniex infer --help` on-device for the exact current flag names — `cli/cmd/geniex/infer.go`
wires `MmprojPath`/device/compute-unit flags; adjust if they differ from the sketch above.)

### Step A — confirm the "before" build reproduces the fastrpc issue

Deploy `pkg-geniex-before/`, run all 3 models with audio through `geniex infer --compute npu`.
Expect a `fastrpc_mmap`/buffer-mapping failure (or `GenieXError` surfacing it) on the mmproj's
compute-buffer allocation, since mmproj shares HTP0 with the LM.

### Step B — confirm the "after" (fixed) build works

Deploy `pkg-geniex/` (fixed), re-run the same 3 models + audio prompt. Expect: all 3 transcribe
the sample clip successfully, with bridge logs showing `Using separate HTP session 'HTP1' for
mmproj` (from `resolve_vision_device`'s `GENIEX_LOG_INFO`, visible at `GENIEX_LOG_LEVEL_DEBUG` or
above).

## 6. What to hand back to me

1. Console output from Step A (the reproduced failure) and Step B (all 3 models succeeding),
   ideally full logs (`GENIEX_LOG_LEVEL=debug` or whatever this SDK's env var is — check
   `sdk/include/geniex.h` / logging.h if unsure).
2. The two `pkg-geniex-before/` and `pkg-geniex/` directories (or just confirm pass/fail — logs
   are the important part, binaries can stay on your workstation).
3. Any flag-name corrections needed in `cli/cmd/geniex/infer.go` invocation above.
4. Whether `GGML_HEXAGON_DEVICES=2` default in `plugin.cpp` caused any regression on non-audio
   (LLM-only) NPU runs — quick sanity check with a plain `geniex infer` (no `--mmproj-path`) on
   any text-only GGUF on `--compute npu`, since bumping virtual session count is a registry-wide
   change, not scoped to VLM loads.

Once you report back, I'll finish validating, clean up, and open the PR from
`fix/audio-mmproj-htp-session`.
