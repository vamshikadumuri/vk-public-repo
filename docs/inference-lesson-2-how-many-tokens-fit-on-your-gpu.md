Inference track, lesson 2 of 8[Back to your prep plan](https://claude.ai/artifact/8m8maPTnDHsbPQWnqjz1bA)

# How many tokens fit on your GPU?

Every token a conversation keeps takes GPU memory. Watch the memory fill, then learn the math and the two vLLM flags that decide how many users you can serve.

One A100 80GB serving Llama 3.1 8B, every conversation 4,096 tokens long

Weights

KV cache

Weights **16 GB** Overhead **\~2 GB** KV cache **54 GB** Held back **8 GB**

Each dot is one 4,096-token conversation**100** fit

**Full.** 100 conversations hold 54 GB of KV cache. Conversation 101 (dashed) waits until one finishes.

## In this lesson

1. [What K and V are](#kv)
2. [Why caching pays](#why-cache)
3. [One token's footprint](#size)
4. [How vLLM splits the GPU](#budget)
5. [max_model_len](#max-len)
6. [Plan your server](#planner)
7. [Levers for more room](#levers)
8. [Check yourself](#quiz)

1

## What K and V are

Inside every layer, attention lets each token look back at earlier ones. To do that, each token is turned into three small vectors:

- **Query (Q)**: what this token is looking for.
- **Key (K)**: what this token can be found by.
- **Value (V)**: what this token passes on when it is found.

Think of a search engine: the query is what you type, keys are page titles, values are the pages. The new token's query is matched against every earlier key, and the best matches pass on more of their value.

The new token print looks back (one head, one layer)

Its query (Q) is compared with each earlier key (K). The match decides how much of that token's value (V) flows in.

Earlier tokenKeyHow well it matchesValue

Output: a blend of the values, mostly instructions, a little of the rest.

Made-up match weights for illustration. A real model does this in all 32 layers, for 32 query heads each, on every pass.

Here's the useful part. An earlier token's key and value *never change*. Next pass, and every pass after, they are exactly the same. So the engine keeps them in GPU memory instead of recomputing them. That store is the **KV cache**. Queries are only needed for the newest token, so they are not cached.

Key ideaThe KV cache holds every past token's keys and values, in every layer, so each new token only computes its own.

2

## Why caching pays

Without a cache, every decode pass would push the whole conversation through the model again. With one, each pass does work for just one token and reads the rest from memory.

A 4-token prompt, then 8 decode passes. Each row is a pass, each square a token.

#### Without a KV cache

**60** tokens computed

#### With a KV cache

**11** tokens computed

computed in prefillcomputed in decoderead from the cache

Now at real sizes: a 1,000-token prompt

Reply length: 300 tokens

**344,850**token computations without a cache

**1,299**with a cache

**265×**less work

Without a cache the work grows with the square of the reply length. With one, it grows in a straight line, and memory pays the price.

Key ideaThe KV cache trades memory for compute. That trade is why memory, not math, decides how many users fit.

3

## One token's footprint

Every token in the cache stores a key and a value, in every layer, for every KV head. Multiply it out for Llama 3.1 8B:

**2**K and V

×

**32**layers

×

**8**KV heads

×

**128**numbers per head

×

**2**bytes each (16-bit)

\=

**128 KiB**131,072 bytes per token

One token's KV cache, drawn out: 512 small vectors

8 keys8 values

32 layers, top to bottom

Each square is one head's key or value: 128 numbers × 2 bytes = 256 bytes. 512 squares × 256 bytes = 128 KiB. Every token you keep needs its own copy of this whole grid.

### Why only 8 KV heads?

Llama 3.1 8B has 32 attention heads but only 8 KV heads. Groups of 4 query heads share one key and value. This is **grouped-query attention**. Without it, the cache would be 4 times bigger: 512 KiB per token.

Try other models

Model

KV cache precision

**128 KiB**per token

**512 MiB**one 4,096-token chat

**16 GiB**one 131,072-token chat

32 layers, 8 KV heads, 128 numbers per head.

Key ideaKV per token = 2 × layers × KV heads × head size × bytes. Learn this formula; interviewers ask for it.

4

## How vLLM splits the GPU

vLLM doesn't grow the KV cache on demand. It carves out one big pool when it starts:

1. Claim a fraction of the GPU, set by `--gpu-memory-utilization`. It is 0.9 in most versions; some newer releases default to 0.92, so check `vllm serve --help` for yours.
2. Load the model weights.
3. Run a dummy batch to measure the peak memory for activations, plus CUDA graphs in recent versions.
4. Turn everything that is left into the KV cache, cut into blocks of 16 tokens.
5. Print the result in the startup log.

A100 80GB, Llama 3.1 8B in 16-bit, defaults

16 GB

KV cache, 54 GB

WeightsActivations and CUDA graphs (assumed \~2 GB)KV cacheNot claimed (10%)

INFO Available KV cache memory: **50.24 GiB** INFO GPU KV cache size: **411,529 tokens** INFO Maximum concurrency for 131,072 tokens per request: **3.14x**

Approximate log lines for this setup; the exact numbers depend on your vLLM version and its measured overhead. Note the log uses GiB (2³⁰ bytes).

Read those lines like this. **GPU KV cache size** is the total number of tokens all live requests share. **Maximum concurrency** divides that by the longest allowed request: how many full-length requests could run at once.

Key ideaKV pool = (GPU memory × utilization) − weights − overhead. Every capacity number follows from that one subtraction.

5

## max_model_len: the startup check

`--max-model-len` is the longest prompt plus reply one request may have. If you don't set it, vLLM takes the model's own limit: 131,072 tokens for Llama 3.1.

At startup, vLLM checks that at least one request of that full length fits in the KV pool. For Llama 3.1 8B that one request needs 131,072 × 128 KiB = 16 GiB. If the pool is smaller, the server stops with an error.

A real failure: Llama 3.1 8B on a smaller memory budget

One request at max_model_len**131,072 tokens**

What the KV pool could hold**30,544 tokens**

ValueError: The model's max seq len (131072) is larger than the maximum number of tokens that can be stored in KV cache (30544).

From a user report on NVIDIA's forum. The full message ends by suggesting two fixes: raise `gpu_memory_utilization` or lower `max_model_len`.

Two things surprise people. First, lowering `--max-model-len` does not shrink the memory vLLM takes. The pool stays the same size; only how many full-length requests fit at once changes. Second, recent vLLM versions may shrink the limit for you and log "Auto-fit max_model_len" instead of failing, so read your startup log.

And vLLM doesn't set aside a full max-length slot for every request. Each one only uses blocks for the tokens it actually has. Lesson 3 shows how.

Key ideamax_model_len is a promise you make to every request. Promise only what your traffic needs, and the server starts and packs more users.

6

## Plan your server

Put it all together. Pick a setup, and the planner does the same subtraction vLLM does at startup, then tells you whether the server would start.

GPU memory

`--gpu-memory-utilization`: 0.90

Model

Weight precision

`--kv-cache-dtype`

`--max-model-len`

Activations and CUDA graphs (assumption): 2.0 GB

One full-length request, same scale

**411,529**tokens in the KV pool

**3.14×**full-length requests at once

**100**4,096-token chats at once

Decimal GB except in the log lines. The overhead is a guess; replace it with the real number from your startup log.

**Worked example: your attacker model**

Suppose your 0.3 share means vLLM gets 30% of an 80 GB A100, about 24 GB. Take away 16 GB of weights and \~2 GB of overhead and about 6 GB is left: roughly 45,000 tokens of KV cache.

Left at the model's default of 131,072 tokens, that server would not start. Set `--max-model-len 8192` and it starts with room for about 5 attack conversations at full length at once. Tap "Your attacker model" above to see it, then compare with your real startup log.

Key ideaCapacity planning is one subtraction and one division. Do it before you deploy, then check it against the log.

7

## Levers for more room

When the pool is too small, these are your options, roughly in the order to try them.

- --max-model-len

  Set it to what your traffic really needs. The server starts, and more full-length requests fit at once.

  Cost: longer requests are rejected. The pool size doesn't change.
- --kv-cache-dtype fp8

  Stores keys and values in 8 bits, roughly doubling the tokens that fit.

  Cost: a small accuracy drop; in vLLM's tests, 0.7 to 2 points on reasoning without calibration. Run your own evals.
- --gpu-memory-utilization

  Lets vLLM claim more of the GPU for the pool.

  Cost: less safety headroom. Risky when other processes share the GPU, as with your fractional share.
- 8-bit or 4-bit weights

  Shrinks the weights and hands the freed memory to the pool: 8 GB more for an 8B model at 8-bit.

  Cost: check quality on your evals. Lesson 5.
- Prefix caching

  Requests that start with the same text, like a shared system prompt, store that prefix once. Recent vLLM versions turn it on by default; older ones need `--enable-prefix-caching`.

  Only helps when prompts share a prefix. Lesson 4.
- --tensor-parallel-size

  Splits the weights and the KV heads across several GPUs.

  Cost: more GPUs and traffic between them. Lesson 6.

Key ideaShrink the promise (max_model_len), then the bytes (fp8 KV, smaller weights), then add hardware.

## Check yourself

Answer each one out loud or in chat first, then open the answer.

1Llama 3.1 70B has 80 layers, 8 KV heads and 128 numbers per head. How much KV cache does one token take in 16-bit?

Show answer

2 × 80 × 8 × 128 × 2 = 327,680 bytes, or 320 KiB per token: 2.5 times the 8B model, because it has 2.5 times the layers.

2Your attacker pod fails at startup with "max seq len (131072) is larger than the maximum number of tokens that can be stored in KV cache". What do you change first?

Show answer

Lower `--max-model-len` to what attack conversations actually need, such as 8,192. Raising `--gpu-memory-utilization` is risky on a shared GPU, and it can't add memory your share doesn't have. FP8 KV cache is the next option if you need longer contexts.

3Grouped-query attention makes the KV cache 4 times smaller. Why doesn't it make single-request decode 4 times faster?

Show answer

At batch size 1, each pass reads 16 GB of weights but only about 0.5 GB of KV cache for a 4,096-token chat. The weights dominate. A smaller cache pays off with long contexts and big batches, where KV reads and memory become the limit.

4How many 4,096-token conversations fit on one A100 80GB serving Llama 3.1 8B in 16-bit, at utilization 0.9 with 2 GB of overhead?

Show answer

Pool = 72 − 16.06 − 2 ≈ 53.9 GB. One conversation = 4,096 × 131,072 bytes ≈ 0.54 GB. 53.9 ÷ 0.54 ≈ 100 conversations.

5100 attack prompts run at once, and all share the same 2,000-token system prompt. How much KV cache does that shared prefix take, and what feature cuts it?

Show answer

100 × 2,000 × 128 KiB ≈ 26 GB, the same text stored 100 times. Prefix caching stores it once (about 0.26 GB) and skips its prefill too. That's Lesson 4.

## What to remember

1. The KV cache keeps every past token's keys and values, so each pass computes only the new token.
2. KV per token = 2 × layers × KV heads × head size × bytes: 128 KiB for Llama 3.1 8B in 16-bit.
3. vLLM's pool = GPU memory × utilization − weights − overhead, fixed at startup.
4. `--max-model-len` must fit once in the pool, or the server won't start. Lowering it doesn't shrink memory use.
5. For more room: lower max_model_len, use FP8 KV, shrink weights, then add GPUs.

**Next: Lesson 3, Batching and PagedAttention**Why vLLM never reserves a full max_model_len slot per request, and how continuous batching keeps the GPU full on every pass.

[Back to Lesson 1](https://claude.ai/artifact/USZqxTneUzroaqdQZYrx37)

Sources

- [Llama 3.1 8B Instruct config](https://huggingface.co/snx999/dev3/blob/main/config.json): 32 layers, 32 attention heads, 8 KV heads, head size 128, 131,072-token limit
- [Llama 3.1 70B config](https://huggingface.co/PatronusAI/Llama-3.1-70B-Instruct-AWQ-INT4/blob/main/config.json): 80 layers, 8 KV heads, head size 128
- [vLLM engine arguments](https://vllm.readthedocs.io/en/latest/models/engine_args.html): utilization default 0.9, block size 16, KV dtype follows the model
- [Particula: debugging vLLM KV cache OOM](https://particula.tech/blog/vllm-kv-cache-oom-debugging): how the pool is computed, the 0.92 default in newer releases, FP8 KV accuracy
- [vLLM production tuning (BestHub)](https://www.besthub.dev/articles/vllm-production-tuning-the-max-num-seqs-gpu-memory-concurrency-triad-62834e8a5b4e): startup log lines
- [NVIDIA forum: Llama 3.1 8B max seq len error](https://forums.developer.nvidia.com/t/problem-with-installation-of-llama-3-1-8b-nim/305569)
- [vLLM forum: max_model_len vs GPU memory](https://discuss.vllm.ai/t/max-model-len-vs-gpu-memory-usage/2724) and [one max-length request must fit](https://discuss.vllm.ai/t/vllm-v1-forces-me-to-pre-allocate-a-huge-non-reclaimable-gpu-kv-cache-for-long-contexts-and-none-of-the-current-offload-or-quantization-options-solve-the-resulting-vram-bloat-without-crippling-speed/1502)
- [NVIDIA forum: Auto-fit max_model_len log line](https://forums.developer.nvidia.com/t/cant-figure-out-why-a-4-bit-model-is-using-so-much-vram-in-vllm/375602)