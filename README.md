# sdcpp-portable-linux-x86_64

Prebuilt **Stable-Diffusion.cpp** (`sd-cli`) for **Linux x86_64**, packaged with a
Bundled **glibc 2.44** so it runs on distros whose system glibc is older than 2.38
Debian 12, Ubuntu 22.04, most CI images) — **no compiling required**.

Upstream: https://github.com/leejet/stable-diffusion.cpp
Built from commit `532d2c1` (upstream release `master-1106-532d2c1`).

## What's in the box

| File | Purpose |
|---|---|
| `sd` | Launcher — **use this**, it wires up the bundled glibc |
| `sd-cli` | The actual binary (needs the bundled loader) |
| `glibc/` | `ld-linux-x86-64.so.2`, `libc.so.6`, libm.so.6`, libstdc++.so.6`, libgcc_s.so.1` |
| `libggml*.so`, libstable-diffusion.so`, libwebm.so`, libwebp*.so` | Runtime libs + CPU backends |

## Install

```bash
curl -L 
  https://github.com/natlusniel-code/sdcpp-portable-linux-x86_64/releases/download/v1.0.0/sdcpp-linux-x86_64-glibc244.tar.gz
mkdir -p sdcpp && tar xzf sdcpp.tar.gz -C sdcpp
cd sdcpp && ./sd --list-devices
```

Expected output:

```
load_backend: loaded CPU backend from .../libggml-cpu-zun4.so
CPU     AMD EPYC 9354P 32-Core Processor
```

## Get models

`sd-cli` is only the engine — you supply weights. Smallest working set:

```bash
mkdir -p models
# Wan 2.1 T2V 1.3B, Q2_K quant (~0.55 GB)
curl -L -o models/wan_t2v_q2k.gguf \
  https://huggingface.co/calcuis/wan-1.3b-gguf/resolve/main/wan2.1_t2v_1.3b-q2_k.gguf
# Wan 2.1 VAE (~0.25 GB)
curl -L -o models/wan_vae.safetensors \
  https://huggingface.co/Comfy-Org/Wan_2.1_ComfyUI_repackaged/resolve/main/split_files/vae/wan_2.1_vae.safetensors
# umT5-XXL text encoder, Q3_K_S (~2.9 GB)
curl -L -o models/umt5_q3.gguf \
  https://huggingface.co/city96/umt5-xxl-encoder-gguf/resolve/main/umt5-xxl-encoder-Q3_K_S.gguf
```

## Generate a video

```bash
./sd -M vid_gen \
  --diffusion-model models/wan_t2v_q2k.gguf \
  --vae             models/wan_vae.safetensors \
  --t5xxl           models/umt5_q3.gguf \
  --params-backend disk -t 4 \
  -p "a red fox walking through a snowy forest, cinematic, soft morning light" \
  -n "blurry, low quality, distorted, watermark, text" \
  --cfg-scale 6.0 --sampling-method euler --flow-shift 3.0 \
  -W 480 -H 320 --video-frames 17 --fps 8 \
  -o out/fox.webm
```

Output is WebM (VP9). Convert with `ffmpeg -i out/fox.webm out/fox.mp4` if needed.

## Memory requirements — read this

`--params-backend disk` streams weights from disk instead of holding them in RAM.
It lowers the *weight* footprint but **not** the compute-graph footprint.

| Model | Weights on disk | Practical RAM |
|---|---|---|
| Wan 2.1 T2V 1.3B Q2_K + umT5 Q3 | ~3.7 GB | **8 GB+** |
| Wan 2.1 T2V 1.3B fp16 + umT5 Q8 | ~8.7 GB | 16 GB+ |
| AnimateDiff (SD1.5 Q4 + motion module) | ~2.0 GB | **4 GB+** |

**Verified limit:** on a 2 GB cgroup this binary generates **images** fine
(SD1.5 Q4_0, 256×256) but **OOMs during video** — the AnimateDiff UNet graph
peaks at ~2.3 GB and the Wan T5 encoder graph needs more than 2 GB on its own.
Give it at least 4 GB for AnimateDiff, 8 GB for Wan.

## Notes

- CPU only. No CUDA/Vulkan/Metal in this build.
- `--list-devices` must show `CP5`; if it shows nothing, the `libggml-cpu-*.so`
  files are missing from the working directory.
- The bundled glibc is only used by `./sd`. Do not `export LD_LIBRARY_PATH` to
  `glibc/` globally — it will break system tools.
- Licence: stable-diffusion.cpp is **MIT**. Model weights carry their own licences
  (Wan 2.1: Apache-2.0; SD1.5: CreativeML OpenRAIL-M; umT5: Apache-2.0).
