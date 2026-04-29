# EngGPT2 llama.cpp local notes

This is a local reproducibility note for EngGPT2-16B-A3B support.

## Patch artifact

Apply the saved patch from a clean llama.cpp checkout:

```sh
git apply enggpt2-llama-cpp-support.patch
```

Changed files:

- `convert_hf_to_gguf.py`
- `src/llama-model.cpp`
- `src/models/llama.cpp`

The runtime-critical changes are in the two C++ files. The Python converter
change is only needed to regenerate a GGUF from Hugging Face weights.

The local file `enggpt2-config-for-conversion.json` is the patched Hugging
Face config used during conversion. It changes the architecture/model type to
the Mixtral-compatible path expected by the converter.

`enggpt2-conversion.log` is the last successful conversion log.

## GGUF conversion from Hugging Face

The modified `convert_hf_to_gguf.py` converts a Hugging Face/Transformers
checkpoint, not an already converted MLX runtime directory.

As of 2026-04-29, the EngGPT2 repositories visible under
`https://huggingface.co/robertobissanti` are MLX conversions:

- `robertobissanti/EngGPT2-16B-A3B-MLX`
- `robertobissanti/EngGPT2-16B-A3B-MLX-4bit`
- `robertobissanti/EngGPT2-16B-A3B-MLX-8bit`

Do not use the 4-bit or 8-bit MLX repositories as input for
`convert_hf_to_gguf.py`. They are useful for `mlx-lm`, but they are not the
source format this converter expects.

For a reproducible GGUF conversion, use a full BF16 Hugging Face checkpoint. If
a full HF-style mirror exists under `robertobissanti`, use that. Otherwise use
the original gated source after accepting its license:

```sh
HF_REPO="engineering-group/EngGPT2-16B-A3B"
SRC="$HOME/models/EngGPT2-16B-A3B-HF"
OUT="$HOME/models/enggpt2-bf16.gguf"
```

Install the converter dependencies:

```sh
python3 -m venv .venv-convert
source .venv-convert/bin/activate
pip install -U pip
pip install -r requirements/requirements-convert_hf_to_gguf.txt
```

Download the model. For gated repositories, authenticate first with
`hf auth login` or set `HF_TOKEN`.

```sh
hf download "$HF_REPO" \
  --local-dir "$SRC" \
  --local-dir-use-symlinks False
```

Replace the downloaded config with the conversion config saved in this fork:

```sh
cp enggpt2-config-for-conversion.json "$SRC/config.json"
```

Run the modified converter:

```sh
python convert_hf_to_gguf.py "$SRC" \
  --outfile "$OUT" \
  --outtype bf16
```

Do not use `--remote` for this conversion unless the remote repository already
contains the patched config. The conversion depends on the local
`config.json` override above.

The converter patch does two EngGPT2-specific things:

- writes `self_attn.q_norm.weight` and `self_attn.k_norm.weight` as
  `attn_q_norm` and `attn_k_norm` GGUF tensors;
- maps EngGPT2 expert tensors from `mlp.experts.*.{gate,up,down}_proj.weight`
  to the Mixtral-compatible expert packing path.

Quick sanity check after conversion:

```sh
./build/bin/llama-completion \
  -m "$OUT" \
  -p "test" \
  -n 1 \
  --temp 0 \
  -dev none --fit off -ngl 0 --no-op-offload \
  --single-turn --reasoning off --no-display-prompt --no-warmup
```

The loader should report `291 tensors`. If it fails with `expected 291, got
243`, the running binary is still using an unpatched or stale `libllama`.

## Publishing the GGUF

Publishing `enggpt2-bf16.gguf` or quantized GGUF derivatives is acceptable only
if the repository inherits the original EngGPT license and keeps the same
research/non-commercial scope.

The original `engineering-group/EngGPT2-16B-A3B` repository is gated and uses
the EngGPT Non-Commercial License. A GGUF file contains converted model weights,
so the public repository must preserve attribution, license terms and usage
limits.

Safe artifacts to publish in this llama.cpp fork regardless of model hosting:

- the `llama.cpp` patch;
- the conversion notes;
- the patched conversion config used locally;
- build and runtime instructions;
- links back to the original gated Hugging Face model.

For a public Hugging Face GGUF repository, the model card should include:

- original model: `engineering-group/EngGPT2-16B-A3B`;
- relation: converted GGUF derivative;
- license: EngGPT Non-Commercial License;
- use scope: research/non-commercial only;
- attribution to Engineering Group / EngGPT-Team;
- conversion notes and llama.cpp patch requirement;
- link to this fork and to `enggpt2-repro-notes.md`;
- a clear note that stock Ollama does not run this GGUF unless its bundled
  llama.cpp runner includes the same runtime patch.

## Related MLX manager

The Apple Silicon MLX workflow is tracked separately in:

```text
https://github.com/robertobissanti/enggpt2-mlx-manager
```

That repository manages the MLX variants published under
`robertobissanti/EngGPT2-16B-A3B-MLX*`, starts the patched `mlx-lm` server and
connects it to Open WebUI. It is not part of the GGUF conversion path, but it is
useful as the parallel MLX runtime path while GGUF support depends on this
patched llama.cpp build.

## Build

Static build avoids accidentally loading an older Homebrew `libllama`:

```sh
rm -rf build
cmake -B build -DCMAKE_BUILD_TYPE=Release -DBUILD_SHARED_LIBS=OFF
cmake --build build --target llama-cli llama-server llama-completion -j 8
```

## Run

```sh
./build/bin/llama-cli \
  -m "$HOME/models/enggpt2-bf16.gguf" \
  -p "Spiega in italiano cos'e un trasformatore elettrico." \
  -n 300 \
  --temp 0
```

If using a dynamic build, clear variables that can make the process load an
older Homebrew `libllama`:

```sh
env -u DYLD_LIBRARY_PATH \
    -u DYLD_FALLBACK_LIBRARY_PATH \
    -u DYLD_INSERT_LIBRARIES \
    -u GGML_BACKEND_PATH \
    ./build/bin/llama-completion \
      -m "$HOME/models/enggpt2-bf16.gguf" \
      -p "Spiega in italiano cos'e un trasformatore elettrico." \
      -n 300 \
      --temp 0 \
      -dev none --fit off -ngl 0 --no-op-offload \
      --single-turn --reasoning off --no-display-prompt
```

## Known failure signature

If the loader reports:

```text
wrong number of tensors; expected 291, got 243
```

then the running `libllama` does not include the EngGPT2 Q/K norm loader patch.
The 48 missing tensors are `attn_q_norm` and `attn_k_norm` for 24 layers.

## Ollama

The stock Ollama app can import the GGUF with a Modelfile, but its bundled
runner currently does not include this local llama.cpp patch. A custom Ollama
build needs the same runtime C++ changes in its vendored llama.cpp copy:

- `llama/llama.cpp/src/llama-model.cpp`
- `llama/llama.cpp/src/models/llama.cpp`

The converter patch is not needed by Ollama for running an already generated
GGUF.
