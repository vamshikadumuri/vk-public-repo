# LLM inference lessons

Nov 11, 2026 · @Vamshi Kadumuri

These lessons build the inference depth Phase 2 needs, one idea at a time, using your attacker model (Llama 3.1 8B on an A100) as the running example.

## Course map

Eight short lessons, each building on the last. Tick one off once you can explain it out loud without notes.

- [ ] **1 · How an LLM serves a request:** prefill, decode, and the latency metrics. Groundwork for week 5.
- [ ] **2 · KV cache and GPU memory math:** how many tokens fit on your GPU, and what `--gpu-memory-utilization` and `--max-model-len` really control. Week 5.
- [ ] **3 · Batching and PagedAttention:** static vs continuous batching, and why vLLM pages its KV cache. Week 5.
- [ ] **4 · Scheduler tricks:** prefix caching and chunked prefill. Week 5.
- [ ] **5 · Making it cheaper:** quantization (AWQ, FP8) and speculative decoding. Week 5.
- [ ] **6 · More than one GPU:** tensor and pipeline parallelism, MIG vs time-slicing. Week 6.
- [ ] **7 · Serving on Kubernetes:** KServe, llm-d, autoscaling on queue depth, split prefill and decode. Week 6.
- [ ] **8 · Benchmarking and cost:** load tests and cost per million tokens. Week 5 proof and week 6 design doc.

## Lesson 1 · How an LLM serves a request

### The big idea

An LLM writes one token at a time, and every token costs one full pass through the model. A token is a chunk of text, roughly three-quarters of an English word. A 300-token answer means at least 300 passes in a row, each waiting on the token before it.

That one fact explains most of inference engineering: why answers stream, why long answers are slow, why batching matters and why the KV cache exists.

&#91;embedded content: request lifecycle · prefill once, then the decode loop\]

The top row runs once and sets TTFT. Every trip around the lower loop adds one token and costs one TPOT, until a stop condition ends the request.

### Prefill and decode

Every request runs in two phases with opposite bottlenecks: prefill is limited by GPU compute, decode by GPU memory bandwidth.

|  | Prefill | Decode |
| --- | --- | --- |
| What it does | Reads the whole prompt in one pass and stores each token's keys and values in the KV cache | Produces one new token per pass, reading the KV cache and adding one entry to it |
| How often | Once per request | Once per output token |
| Tokens per pass | All prompt tokens, in parallel | One per request |
| Bottleneck | Compute (FLOPs) | Memory bandwidth |
| Metric it sets | Time to first token (TTFT) | Time per output token (TPOT) |
| Grows with | Prompt length | Output length, and context length through the KV cache |

The KV cache is why decode never re-reads the whole prompt. Attention needs the keys and values of every earlier token, so the engine computes them once and keeps them in GPU memory. The price is memory: 128 KiB per token for Llama 3.1 8B in FP16, using the formula in Phase 2. A 4,096-token conversation therefore holds 512 MiB. Lesson 2 does this math properly.

### Why decode is slow

With plain decoding, one request alone on an A100 80GB SXM can't generate Llama 3.1 8B in FP16 faster than about 127 tokens per second, however fast the tensor cores are. The ceiling comes from memory, not math:

```latex
\text{max tokens/s for one request} \approx \frac{\text{GPU memory bandwidth}}{\text{bytes of model weights}}
```

Each decode step reads all 16 GB of weights from GPU memory to produce one token. An A100 80GB SXM moves 2,039 GB/s, so a step takes at least 7.9 ms. The math in that step is about 16 GFLOPs, which the A100's 312 TFLOPS finishes in 0.05 ms. The tensor cores sit idle over 99% of the time, waiting on memory. On a 40 GB A100 (1,555 GB/s) the ceiling drops to about 97 tokens per second.

Batching is the fix. When 32 requests decode together, one read of the weights serves 32 tokens instead of one, so throughput climbs while each step slows only a little. Databricks measured this on one A100: batch size 64 raised throughput about 14x while per-request latency grew about 4x. Lesson 3 covers how vLLM manages that trade.

Prefill is the opposite case. A 1,000-token prompt needs about 16 TFLOPs, around 51 ms even at the A100's peak, while the weights are read once. Long prompts cost compute and show up as a slower TTFT.

Rule of thumb: **decode speed follows memory bandwidth, prefill speed follows FLOPs.** It is why 8-bit or 4-bit weights speed up decode: fewer bytes to read per token. Speculative decoding, in Lesson 5, gets past the single-request ceiling by checking several guessed tokens per read of the weights.

### The metrics interviewers ask about

Users feel two numbers, TTFT and TPOT. Whoever pays for the GPU cares about a third, throughput.

- **TTFT (time to first token):** the wait from sending a prompt to the first streamed token. Set by prefill, plus any time spent queued.
- **TPOT (time per output token):** the average gap between tokens after the first. Set by decode. 100 ms per token is about 10 tokens per second.
- **ITL (inter-token latency):** each individual gap between two tokens. Its average matches TPOT, but its spread shows jitter that an average hides. vLLM's serving benchmark reports both.
- **End-to-end latency:** the time for the whole response.
- **Throughput:** output tokens per second across all requests on the server. This is what sets cost per million tokens.

```latex
\text{end-to-end latency} \approx \text{TTFT} + \text{TPOT} \times (N - 1), \quad N = \text{output tokens}
```

Worked example: your attacker model gets a 1,000-token attack prompt and writes a 300-token reply, alone on the GPU. Take TTFT = 0.1 s and TPOT = 10 ms, close to the 7.9 ms ceiling above. End-to-end latency is 0.1 + 0.010 × 299 ≈ 3.1 s, and decode is about 97% of it. Capping output length with `max_tokens` helps far more than trimming the prompt.

In an interview, first ask which metric matters. A chat UI lives or dies on TTFT. A batch red-team run like yours cares about throughput, because nobody watches each reply stream.

### Sampling: picking the next token

Each pass ends with a score (a logit) for every token in the vocabulary, 128,256 of them for Llama 3.1. Sampling turns those scores into one choice:

- **Greedy (temperature 0):** always take the top token. Repeatable, but prone to loops.
- **Temperature:** divides the scores before they become probabilities. Below 1 sharpens toward the top token; above 1 spreads toward variety.
- **Top-k:** sample only from the k highest-scoring tokens.
- **Top-p (nucleus):** sample from the smallest set of tokens whose probabilities add up to p, such as 0.9.

Generation stops at the end-of-sequence token, a stop string, or `max_tokens`. Sampling is cheap next to the forward pass, so it changes what you get, not how fast. For your attacker model, a higher temperature gives more varied attack prompts, at the cost of more off-target ones.

### Check yourself

Answer these in chat in a line or two each. I'll mark them, then we start Lesson 2.

1. Why does a 2,000-token answer take far longer than reading a 2,000-token prompt?
2. You switch your attacker model from FP16 to 8-bit weights. Which improves more, TTFT or TPOT, and why?
3. A teammate says, "The GPU shows 5% compute use during decode, so it's oversized." What do you tell them?
4. Your red-team job sends 500 attack prompts overnight. Do you tune for TTFT or throughput, and why?
5. Estimate the best single-request decode speed for Llama 3.1 8B in FP16 on an H100 SXM, which moves 3.35 TB/s.

## Sources

- [NVIDIA A100 Tensor Core GPU datasheet](https://nvidia.com/content/dam/en-zz/Solutions/Data-Center/a100/pdf/nvidia-a100-datasheet-us-nvidia-1758950-r4-web.pdf): memory bandwidth and TFLOPS per variant
- [LLM Inference Performance Engineering: Best Practices – Databricks](https://www.databricks.com/blog/llm-inference-performance-engineering-best-practices): TTFT, TPOT, throughput, and the batch-size-64 measurement
- [LLM inference benchmarking – DigitalOcean](https://www.digitalocean.com/blog/llm-inference-benchmarking): TPOT vs ITL and the latency formula
