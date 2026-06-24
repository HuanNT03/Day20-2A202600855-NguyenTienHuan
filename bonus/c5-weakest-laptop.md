# Challenge C5 — The "Weakest Laptop" Challenge

## Setup
- **Device**: HP Pavilion
- **CPU**: Intel Core i7-1165G7 (4 physical cores, 8 logical threads)
- **RAM**: 7.5 GB (usable)
- **GPU**: None (CPU-only serving)
- **Model**: `TinyLlama-1.1B-Chat-v1.0-GGUF`
- **Compared Quantizations**: `Q4_K_M` (636.2 MB) vs `Q2_K` (482.5 MB)

---

## Performance & Size Numbers

| Model Quantization | File Size (MB) | Decode Rate (tok/s) | TTFT P50 (ms) | E2E P50 (ms) | RAM Used (MB) |
|:---|:---:|:---:|:---:|:---:|:---:|
| **Q4_K_M** | 636.2 | 37.8 | 144 | 1625 | ~950 |
| **Q2_K** | 482.5 | 38.9 | 260 | 1670 | ~780 |

---

## Quality Evaluation (Hand-Graded on 5 Prompts)

We evaluated both models on 5 common prompts (chat and simple retrieval).

1. **Prompt 1: "Define goodput@SLO in one sentence."**
   - **Q4_K_M**: "Goodput@SLO is the number of requests per second that satisfy both Time-to-First-Token (TTFT) and Time-Per-Output-Token (TPOT) Service Level Objectives." (Grade: **Pass**)
   - **Q2_K**: "Goodput is the number of requests per second of output token count that satisfies TTFT and TPOT and the latency of the SLOs." (Grade: **Borderline/Confusing**)

2. **Prompt 2: "What is PagedAttention?"**
   - **Q4_K_M**: "PagedAttention is a memory management technique that treats KV cache like virtual memory pages, eliminating fragmentation." (Grade: **Pass**)
   - **Q2_K**: "PagedAttention is a cache for KV pages that virtual memory to eliminate fragmentation of cache slots." (Grade: **Fail** - grammatically broken)

3. **Prompt 3: "Explain difference between throughput and goodput."**
   - **Q4_K_M**: "Throughput measures the total volume of data or requests processed, whereas goodput only counts those that meet a specific quality of service or SLO." (Grade: **Pass**)
   - **Q2_K**: "Throughput is total data but goodput is the good data that we get from the server." (Grade: **Pass** - simple but correct)

4. **Prompt 4: "Write a python function to compute factorial."**
   - **Q4_K_M**: Writes correct recursive/iterative function. (Grade: **Pass**)
   - **Q2_K**: Syntax error in loop, forgets indentation on return statement. (Grade: **Fail**)

5. **Prompt 5: "When should I use speculative decoding?"**
   - **Q4_K_M**: "Use speculative decoding when you need to reduce serving latency for single-user scenarios by pairing a small draft model with a larger target." (Grade: **Pass**)
   - **Q2_K**: "Use speculative decoding when you want to make the model run faster with a draft model but not at high batch size." (Grade: **Pass** - correct context)

---

## Conclusion & Decision

On a 7.5 GB RAM ceiling machine:
1. **RAM capacity is not a bottleneck**: The `Q4_K_M` model (636.2 MB) uses under 1 GB of RAM when loaded, leaving over 6 GB of free RAM. 
2. **Speed difference is negligible**: Moving from `Q4_K_M` to `Q2_K` only yields a **~3% speedup** in decode rate (from 37.8 to 38.9 tok/s) because the 1.1B model is already so small that CPU memory bandwidth is not fully saturated compared to larger models.
3. **Quality drop is severe**: The `Q2_K` model frequently produces grammatically incorrect sentences, forgets syntax in code blocks, and exhibits borderline coherence.

**Decision**: For a low-RAM laptop (7.5 GB), **`Q4_K_M` is vastly superior and far more useful** than `Q2_K`. Lowering to 2-bit quantization is not worth the quality loss when the model easily fits in RAM.
