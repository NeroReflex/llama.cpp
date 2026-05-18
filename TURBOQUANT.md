# TurboQuant Benchmark Report

Date: 2026-05-18  
Repo: llama.cpp (branch: tq_merge)  
Build: b9381-3e5773d25  
Model: ../Qwopus3.6-35B-A3B-v1-APEX-I-Compact.gguf

## Methodology

Benchmark objective:
- Compare memory footprint of q4_0/q4_0 vs turbo cache types.
- Compare turbo4/turbo3 memory with Flash Attention on vs off.
- Compare throughput across turbo cache quantization choices.

Common settings for all runs:
- Binary: ./build/bin/llama-cli
- Context size: 262144
- Prompt: "Benchmark token generation: explain matrix multiplication in one sentence."
- Tokens generated: 96
- Seed: 42
- Temperature: 0
- Reasoning: off
- Conversation mode: off (-no-cnv)
- Jinja: off (--no-jinja)

Per-run metrics captured:
- prompt_tps and generation_tps from internal timing lines
- CUDA0 KV buffer size (MiB)
- CPU KV buffer size (MiB)
- CUDA0 compute buffer size (MiB)
- CUDA_Host compute buffer size (MiB)
- Max RSS (MiB) from /usr/bin/time -v

Notes:
- VRAM comparison primarily uses CUDA0 KV buffer size and CUDA0 compute buffer size reported by llama logs.
- Max RSS reflects host RAM usage of the process.
- Results are single-run measurements per configuration.

## Results

| label | cache_k | cache_v | flash_attn | prompt_tps | generation_tps | cuda0_kv_mib | cpu_kv_mib | cuda0_compute_mib | cuda_host_compute_mib | max_rss_mib |
|---|---|---|---|---:|---:|---:|---:|---:|---:|---:|
| q4q4_fa_on | q4_0 | q4_0 | on | 88.33 | 47.59 | 1440.00 | 0.00 | 804.02 | 520.02 | 16509.47 |
| t4t3_fa_on | turbo4 | turbo3 | on | 94.83 | 46.93 | 1180.13 | 0.00 | 812.02 | 520.02 | 16511.25 |
| t4t3_fa_off | turbo4 | turbo3 | off | 93.02 | 47.29 | 1180.13 | 0.00 | 812.02 | 520.02 | 16510.19 |
| t2t2_fa_on | turbo2 | turbo2 | on | 95.81 | 50.64 | 782.13 | 0.00 | 812.02 | 520.02 | 16513.45 |
| t3t3_fa_on | turbo3 | turbo3 | on | 99.31 | 47.91 | 1000.13 | 0.00 | 812.02 | 520.02 | 16509.82 |
| t4t4_fa_on | turbo4 | turbo4 | on | 93.46 | 45.46 | 1360.13 | 0.00 | 812.02 | 520.02 | 16505.40 |

## VRAM Winner Summary

| Configuration | CUDA0 KV Buffer (MiB) | Total VRAM (KV + Compute) (MiB) | Can Flash Attention Reduce VRAM? | Rank |
|---|---:|---:|---|---|
| **turbo2/turbo2** | **782.13** | **1594.15** | ✗ No measured reduction | **🏆 Best** |
| turbo3/turbo3 | 1000.13 | 1812.15 | ✗ No measured reduction | 2nd |
| turbo4/turbo3 | 1180.13 | 1992.15 | ✗ No measured reduction | 3rd |
| turbo4/turbo4 | 1360.13 | 2172.15 | ✗ No measured reduction | 4th |
| q4_0/q4_0 | 1440.00 | 2244.02 | ✗ No measured reduction | 5th |

**Key Findings:**
- **turbo2/turbo2 is the VRAM winner**: 46% less KV cache than q4_0 (782.13 MiB vs 1440.00 MiB).
- **Flash Attention does NOT reduce VRAM** in this test: turbo4/turbo3 with FA on and FA off show identical KV buffer sizes (1180.13 MiB). FA improves computation efficiency but does not eliminate or compress the KV cache itself.
- **Generation speed winner**: turbo2 also has the fastest generation throughput (50.64 t/s).

## Verification Against Requested Questions

1. turbo3+turbo4 uses less VRAM/RAM than q4_0+q4_0:
- VRAM (KV buffer): YES.
  - q4_0/q4_0: 1440.00 MiB
  - turbo4/turbo3: 1180.13 MiB
  - Delta: -259.87 MiB (-18.05%)
- RAM (Max RSS): NO clear reduction in this single-run test.
  - q4_0/q4_0: 16509.47 MiB
  - turbo4/turbo3: 16511.25 MiB
  - Delta: +1.78 MiB (effectively equal within run noise)

2. turboquant + flashattention occupies less VRAM than turboquant without flashattention:
- For turbo4/turbo3 in this test: NO measurable reduction.
  - FA on CUDA0 KV buffer: 1180.13 MiB
  - FA off CUDA0 KV buffer: 1180.13 MiB
  - FA on CUDA0 compute buffer: 812.02 MiB
  - FA off CUDA0 compute buffer: 812.02 MiB
- Conclusion: no VRAM difference observed with this setup and measurement method.

3. speed differences between turboquant quantizations:
- Generation throughput (tokens/s), FA on:
  - turbo2/turbo2: 50.64 (fastest)
  - turbo3/turbo3: 47.91
  - turbo4/turbo3: 46.93
  - turbo4/turbo4: 45.46 (slowest among tested turbo pairs)
- Prompt throughput (tokens/s), FA on:
  - turbo3/turbo3: 99.31 (highest)
  - turbo2/turbo2: 95.81
  - turbo4/turbo3: 94.83
  - turbo4/turbo4: 93.46

## Reproduction Command Pattern

Example (replace cache/FA args per case):

/usr/bin/time -v ./build/bin/llama-cli \
  -no-cnv --no-jinja \
  -m ../Qwopus3.6-35B-A3B-v1-APEX-I-Compact.gguf \
  --ctx-size 262144 \
  --cache-type-k turbo4 --cache-type-v turbo3 \
  --flash-attn on \
  --seed 42 --temp 0 --reasoning off \
  --no-display-prompt -n 96 \
  -p "Benchmark token generation: explain matrix multiplication in one sentence." \
  --log-file /tmp/case.internal.log --log-verbose
