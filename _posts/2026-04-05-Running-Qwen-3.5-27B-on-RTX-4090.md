---
layout: post
title: "Running Qwen 3.5 27B on 4090"
date: 2026-04-05
---

## The model

Qwen 3.5 27B has a hybrid attention architecture that makes it unusually efficient on consumer GPUs. Out of its 64 layers:

- **48 layers** use Gated DeltaNet (GDN) — a linear attention variant with fixed-size state. No KV cache needed.
- **16 layers** use standard grouped-query attention (GQA) with 4 KV heads each.

Only 16 of 64 layers need a KV cache. This makes the cache ~4x smaller than a conventional 27B transformer at the same context length.

At 128K context with q4_0 KV quantization, the entire KV cache is about **2 GB**. A standard 27B model would need ~8 GB.

Even better: independent testing shows q4_0 KV cache is **completely lossless** on this architecture (BLEU 1.000 vs FP16). The GDN layers act as error correction between the sparse quantized attention layers.

## The setup

- **GPU:** RTX 4090 (24 GB VRAM)
- **OS:** WSL2 Ubuntu 24.04
- **Backends tested:** Ollama, llama.cpp server
- **Quantizations:** Q4_K_M (16.2 GB), Q5_K_M (18.6 GB)
- **KV cache types:** q8_0, q4_0
- **Context lengths:** 2K, 8K, 32K, 64K, 128K

Each test fills 50% of the context with prompt text and generates 200 tokens. For llama.cpp tests, the server is restarted between context sizes with VRAM fully cleared.

## Results

### Generation speed (tok/s)

| Configuration | 2K | 8K | 32K | 64K | 128K |
|---|---:|---:|---:|---:|---:|
| **llama.cpp Q4_K_M q4_0 KV** | **39.3** | **37.7** | **37.9** | **36.4** | **33.4** |
| llama.cpp Q5_K_M q4_0 KV | 34.2 | 34.1 | 33.1 | 32.0 | 29.9 |
| Ollama Q4_K_M q4_0 KV | 35.8 | 36.4 | 8.5 | 23.3 | 15.5 |
| Ollama Q4_K_M q8_0 KV | 23.3 | 23.3 | 17.9 | 12.1 | 6.3 |

### Prompt processing speed (tok/s)

| Configuration | 2K | 8K | 32K | 64K | 128K |
|---|---:|---:|---:|---:|---:|
| **llama.cpp Q4_K_M q4_0 KV** | 1,409 | **2,203** | **2,467** | **2,302** | **2,121** |
| llama.cpp Q5_K_M q4_0 KV | 1,589 | 2,245 | 2,400 | 2,296 | 2,085 |
| Ollama Q4_K_M q4_0 KV | 1,272 | 1,374 | 198 | 1,526 | 930 |
| Ollama Q4_K_M q8_0 KV | 1,443 | 1,601 | 1,140 | 1,082 | 614 |

### VRAM usage (MB)

| Configuration | 2K | 8K | 32K | 64K | 128K |
|---|---:|---:|---:|---:|---:|
| llama.cpp Q4_K_M q4_0 KV | 19,220 | 19,328 | 19,800 | 20,432 | 21,694 |
| llama.cpp Q5_K_M q4_0 KV | 21,758 | 21,866 | 22,338 | 22,980 | 24,018 |
| Ollama Q4_K_M (any) | ~23,700 | ~23,700 | ~23,700 | ~23,800 | ~24,100 |

## Analysis

### llama.cpp is 5.3x faster than Ollama at 128K

The standout result: llama.cpp Q4_K_M at 128K generates at **33.4 tok/s** vs Ollama's **6.3 tok/s** with q8_0 KV. Even with q4_0 KV, Ollama only manages 15.5 tok/s.

llama.cpp's performance curve is nearly flat — 39 tok/s at 2K down to 33 tok/s at 128K. That's only a 15% drop across a 64x increase in context. Ollama drops 73% over the same range.

### Why Ollama is slow: the auto-allocator

Ollama has a memory scheduler that automatically splits the model between GPU and CPU. Sounds convenient, but it makes bad decisions:

**Q4_K_M with q4_0 KV cache:**
Ollama put 15.5 GiB of model weights on GPU and offloaded **710 MiB to CPU**, even though the model fits entirely in VRAM. Those 710 MiB on CPU create a PCIe bottleneck on every token generation.

At 32K context, Ollama chose an especially bad split and generation dropped to 8.5 tok/s — worse than neighboring context sizes. The auto-allocator's decisions are non-deterministic.

**Q5_K_M:**
Ollama offloaded 833 MiB to 3.4 GiB of weights to CPU depending on context size. Most test configurations timed out at Ollama's 10-minute limit. The model was essentially unusable.

**llama.cpp** lets you control allocation explicitly. All 64 layers stay on GPU. No CPU round-trips, no PCIe bottleneck.

### Ollama pre-allocates, llama.cpp allocates on demand

Ollama fills VRAM to the brim regardless of requested context. Whether you ask for 2K or 128K, it uses ~23.7 GB.

llama.cpp allocates proportionally:

```
2K context:   19.2 GB (model + tiny KV)
128K context: 21.7 GB (model + 2 GB KV)
```

llama.cpp at 128K uses **less VRAM** than Ollama at 2K.

### Q4_K_M vs Q5_K_M: not worth the upgrade

| | Q4_K_M | Q5_K_M |
|---|---|---|
| File size | 16.2 GB | 18.6 GB |
| VRAM at 128K | 21.7 GB | 24.0 GB |
| Speed at 128K | 33.4 tok/s | 29.9 tok/s |
| Headroom at 128K | 2.9 GB | 546 MB |
| Bits per weight | 5.01 | 5.69 |

Q5_K_M barely fits at 128K with 546 MB to spare. On a system where the GPU also drives a display (like WSL2), that's risky. Qwen 3.5 is reportedly robust to quantization — the quality difference at Q4 vs Q5 is minimal.

### q4_0 vs q8_0 KV cache: free performance

On standard transformers, q4_0 KV cache causes measurable degradation. On Qwen 3.5, it's lossless thanks to the hybrid architecture — the GDN layers absorb quantization noise from the 16 attention layers.

The practical impact: switching from q8_0 to q4_0 KV freed ~3.5 GB of VRAM for the same context length, which on Ollama was the difference between working and timing out. On llama.cpp it meant more headroom and slightly faster inference.

## The production config

```ini
# /etc/systemd/system/llama-server.service
[Service]
ExecStart=/home/tianc/bin/llama-server \
    -m /home/tianc/models/Qwen_Qwen3.5-27B-Q4_K_M.gguf \
    --ctx-size 131072 \
    --flash-attn on \
    --cache-type-k q4_0 --cache-type-v q4_0 \
    --parallel 1 --batch-size 2048 --ubatch-size 512 \
    --port 8001 --host 0.0.0.0 --metrics \
    --jinja --chat-template-kwargs '{"enable_thinking":false}'
```

OpenAI-compatible API at `localhost:8001`. 128K context window. 33+ tok/s generation. 2.9 GB VRAM headroom.

## Thinking mode: on vs off

Qwen 3.5 has a built-in "reasoning" mode where it thinks in `<think>` tags before answering. I tested both modes on a real podcast transcript extraction (10,909 prompt tokens, structured JSON output).

| Mode | Gen tok/s | Output tokens | Wall time | Cases extracted |
|---|---:|---:|---:|---:|
| Reasoning OFF | 41.3 | 2,317 | 62.2s | 3 |
| Reasoning ON | 42.8 | 2,054 | 49.2s | 3 |

Both modes extracted identical results: 3 victims, same names, same details, same confidence scores (0.95).

The model didn't actually engage thinking in either mode. When the prompt says "Return ONLY valid JSON, no markdown or explanation," the model skips reasoning entirely — even with `--reasoning on`. The thinking budget goes unused.

For structured extraction tasks with explicit output format constraints, thinking mode makes no difference. It would likely matter more for open-ended analysis or ambiguous matching, but for JSON extraction from transcripts, `--reasoning off` is the right call — it avoids any accidental thinking overhead.

## The display blackout incident

During testing, repeatedly loading and unloading models (filling 24 GB of VRAM 10+ times) caused one of my monitors to go black. The RTX 4090 in WSL2 shares VRAM between Windows display and CUDA compute. Under extreme pressure, the display compositor loses its allocations and can't recover without a reboot.

The fix is also the production config: load the model once at boot, keep it resident. No cycling = no pressure spikes.

## Takeaways

1. **Qwen 3.5's hybrid architecture changes the VRAM math.** The 4x smaller KV cache means 128K context on a 24 GB card with room to spare.
2. **q4_0 KV cache is free on this model.** Lossless compression that saves gigabytes.
3. **Ollama's auto-allocator is the bottleneck, not the GPU.** The same hardware is 5x faster when you control the memory layout yourself.
4. **llama.cpp server has a nearly flat performance curve.** 39 tok/s at 2K, 33 tok/s at 128K. The model is genuinely usable at long context.
5. **Q4_K_M is the sweet spot for 24 GB.** Q5_K_M barely fits and isn't meaningfully better.
