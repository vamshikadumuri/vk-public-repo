Inference track, lesson 4 of 8[Back to your prep plan](https://claude.ai/artifact/8m8maPTnDHsbPQWnqjz1bA)

# Don't compute the same prompt twice

Many requests start with the same text. Prefix caching reuses the work already done for it. Chunked prefill stops one long prompt from freezing everyone else's replies.

Six attack prompts. Each one starts with the same 2,048-token system prompt and template, then adds 256 tokens of its own.

**Without prefix caching**computed: **13,824** tokens

**With prefix caching**computed: **3,584** tokens

computed in prefillreused from the cache

**Done.** The shared start is computed once. Prompts 2 to 6 only compute their own 256 tokens: 3.9 times less prefill work.

## Words used in this lesson

Prefix

The start of a prompt. Two prompts "share a prefix" when they begin with the same tokens.

Prefill

Reading the prompt and filling the KV cache for it (Lesson 1).

Block

16 tokens of KV cache (Lesson 3).

Fingerprint

A short code (a hash) that vLLM computes from a block's tokens, so it can find matching blocks quickly.

Cache hit

A block was found already computed, so it is reused.

Chunk

A piece of a long prompt, read in one pass.

Token budget

The most tokens one pass may process, set by `--max-num-batched-tokens`.

## In this lesson

1. [The same text, again and again](#repeat)
2. [How vLLM finds a match](#match)
3. [Fixed text first](#order)
4. [Where the cache lives](#evict)
5. [What it speeds up](#gains)
6. [The cache can leak](#security)
7. [Chunked prefill](#chunked)
8. [Settings to know](#settings)
9. [Check yourself](#quiz)

1

## The same text, again and again

Look at the prompts a real server gets. Many of them start the same way:

- A long **system prompt** sent with every request to the same app.
- **Examples** placed in the prompt to show the model what to do.
- A long **document** that users ask many questions about.
- A **chat**: every new turn resends all the earlier turns.
- Your **red-team runs**: one attack template, hundreds of attack texts.

Without help, prefill computes the KV cache for that shared start every single time. The result is identical each time, so the work is wasted.

Key ideaIf two prompts start with the same tokens, their KV cache for that start is exactly the same. It only needs computing once.

2

## How vLLM finds a match

vLLM calls this **automatic prefix caching**. It works on the 16-token blocks from Lesson 3:

1. When a block is full, vLLM gives it a fingerprint. The fingerprint comes from the block's own tokens *and* the fingerprint of the block before it.
2. When a new request arrives, vLLM computes the fingerprints of its blocks, from the start.
3. It looks each one up. Every match is reused. The first miss ends the search.
4. Prefill runs only for the tokens after the last match.

Because each fingerprint includes the one before it, a block only matches if *everything before it* matches too. And only full blocks are cached, so a partly filled last block is always computed again.

Try it: is the new request a cache hit?

Request 1 (already processed, its blocks are in the cache)

Request 2 (new)

4 tokens per block and one word per token, to keep it small. vLLM uses 16 tokens per block. The fingerprints here are made up for the picture.

Key ideaA cache hit needs the exact same tokens from the very first one. The first difference stops all reuse after it.

3

## Fixed text first

This gives one simple rule for writing prompt templates: **put text that never changes first, and text that changes last.** A date, a user name or a request ID at the top changes the very first block. Then nothing after it can be reused.

#### Cache-hostile order

1. Request ID and timestamp
2. System prompt (2,000 tokens)
3. Attack instructions
4. This attack's text

The first block differs every time. Almost nothing is reused.

#### Cache-friendly order

1. System prompt (2,000 tokens)
2. Attack instructions
3. This attack's text
4. Request ID, if the model must see it

Everything up to the attack text is reused.

Small differences count too. An extra space, a different line ending or a changed chat template gives different tokens, and the cache misses.

Key ideaFixed text first, changing text last. Keep the fixed part byte-for-byte identical.

4

## Where the cache lives

Prefix caching does not need extra memory. It uses the KV pool you already have (Lesson 2):

- While requests use a block, it stays put, and many requests can share it (Lesson 3).
- When no request uses a block anymore, vLLM puts it on the free list *but keeps its contents*. A later request can still get a hit on it.
- When vLLM needs a block, it takes the one at the front of the free list: the one used **least recently**. Only then is the old content gone.

The free list: blocks waiting to be reused

Read left to right: overwritten first (least recently used) → freed most recently.

**#41**old chat

**#7**old chat

**#19**doc Q&A

**#3**attack template

**#28**attack 17

**#12**attack 18

A new attack prompt hits block #3 (dark). vLLM takes it off the free list so it can't be overwritten, and uses it. Block #41, outlined, is the next one to be overwritten.

So a busy server keeps the most recently used prefixes. If memory is tight, an old prefix may be pushed out before it is reused, and the hit rate drops.

Key ideaThe cache is just free memory that hasn't been overwritten yet. It costs no extra space.

5

## What it speeds up

Prefix caching skips prefill work, so it cuts **time to first token** and frees GPU compute for other requests. It does not change decode: every new token still needs its own pass (Lesson 1). The vLLM docs call it "almost a free lunch", because it doesn't change the model's output.

Try it: your workload

Shared start of every prompt: 2,048 tokens

Each prompt's own text: 256 tokens

Prompts sent

**89%**of prompt tokens come from the cache

**119 ms**prefill math per prompt, no cache

**13 ms**with a cache hit

Llama 3.1 8B on an A100 80GB SXM at peak speed, from Lesson 1: about 0.05 ms of math per prompt token, and never less than 7.9 ms per pass. Real systems are slower, but the ratio holds. Only full 16-token blocks count as shared.

Sharing also saves memory while requests run together. Say 100 attacks share a 2,048-token start. Stored once, it takes about 0.27 GB of KV cache. As 100 copies, it would take about 26.8 GB.

Key ideaPrefix caching makes prefill cheaper and saves memory. Decode speed stays the same.

6

## The cache can leak

This matters on any shared server. A cache hit makes the first token arrive sooner. Someone who can send prompts to the *same server* can measure that delay to learn what *other* users sent:

1. A user sends a prompt that starts with something private.
2. An attacker on the same server sends a guess of that start.
3. If the first token comes back unusually fast, the guess was in the cache. The guess was right.

Researchers have shown this kind of attack works against vLLM and SGLang. By default, vLLM shares its prefix cache across *all* requests on a server.

**The fix: `cache_salt`.** Send a salt value with each request, for example one per team or per app. vLLM mixes it into the first block's fingerprint, so only requests with the same salt can share blocks. It is off unless your gateway or app sets it.

Even this fix has had bugs. In CVE-2026-105752, vLLM lost the salt in some multi-turn requests with tool calls, sent through the `/v1/responses` endpoint. Those turns went into the shared cache. Keep vLLM up to date.

Key ideaA shared prefix cache is a shared secret channel. On a multi-tenant server, set `cache_salt` per tenant, and add this to your red-team test plan.

7

## Chunked prefill

Now the second problem. Suppose 16 users are chatting, getting a token every pass. Then someone sends a 32,000-token prompt. If its whole prefill runs in one pass, that pass takes far longer than normal. All 16 users see their reply freeze. This is called a **generation stall**. A 2024 paper, Sarathi-Serve, showed these stalls could last several seconds in vLLM at the time.

The fix is **chunked prefill**. The long prompt is cut into chunks. Each pass first includes every running decode, then fills the rest of the token budget with the next chunk. Decodes never wait for a whole prefill. In vLLM V1 this is always on.

Try it: one long prompt arrives while 16 users are chatting

Length of the long prompt

Token budget per pass (`--max-num-batched-tokens`)

0 ms

normal pass: only the 16 chatspass that also reads part of the long prompt

**1.69 s**longest freeze the 16 chats see

**1.69 s**first token of the long prompt

**1**passes to read the long prompt

Each segment is one pass, drawn to scale. Every chatting user gets one token at the end of each pass, so a wide segment is a freeze. Ideal-hardware numbers for Llama 3.1 8B on an A100 80GB SXM, as in Lesson 1. A normal pass takes 7.9 ms.

The token budget is the dial. The vLLM docs put it simply: a smaller budget, such as 2,048, gives smoother token streams (better inter-token latency), because prefills slow down decodes less. A bigger budget gives a faster first token for long prompts. For the most throughput, the docs suggest a budget above about 8,000 for smaller models on large GPUs.

Key ideaChunked prefill splits long prompts across passes, so running chats keep getting tokens. The token budget trades smooth streaming against first-token speed.

8

## Settings to know

- Prefix caching

  On by default in vLLM V1 for text models. To turn it off, use `--no-enable-prefix-caching`.

  Check it works: the log line "Prefix cache hit rate", or the Prometheus counters `vllm:prefix_cache_queries` and `vllm:prefix_cache_hits`.
- cache_salt

  A field in each request. Requests share cached blocks only if their salts match.

  Set it per tenant on any shared server.
- --prefix-caching-hash-algo

  How fingerprints are made. Since v0.11 the default is SHA-256, which avoids accidental collisions.

  Faster options exist but are weaker.
- --max-num-batched-tokens

  The token budget per pass, and so the chunk size for long prompts.

  Smaller: smoother streaming. Bigger: faster first token and more throughput.

**For your red-team runs**

- Put the system prompt and attack template first, and each attack's text last.
- Keep IDs, dates and run numbers out of the start of the prompt.
- For multi-turn attacks, resend earlier turns exactly as before, so they hit the cache.
- Watch the hit rate. For a run with one shared template, it should be high.
- If you test very long prompts, expect other runs on the same server to stream more slowly while they are read.

## Check yourself

Answer each one first, out loud or in chat, then open the answer.

1Two prompts share their first 100 tokens. Blocks hold 16 tokens. How many tokens can the second prompt reuse?

Show answer

96\. Only full blocks are cached: 100 ÷ 16 = 6 full blocks, which is 96 tokens. The last 4 shared tokens sit in a partly filled block and are computed again.

2Your attack template starts with "Run ID: 8f3a…" and then the 2,000-token system prompt. What will the hit rate be, and what do you change?

Show answer

Close to zero, because the first block differs every time and breaks the chain. Move the run ID to the end, or drop it from the prompt.

3Does prefix caching make tokens stream faster once the reply has started?

Show answer

No. It skips prefill work, so the first token comes sooner. Every later token still needs its own decode pass. It can help indirectly by saving memory, which lets more requests run together.

416 users are chatting when a 32,000-token prompt arrives. Roughly how long do they freeze without chunking? And with a budget of 2,048?

Show answer

Without chunking: one pass of about 32,000 × 0.05 ms ≈ 1.6 seconds at ideal speed. With 2,048: the prompt is read over 16 passes of about 0.1 s each, so each freeze is about 0.1 s.

5Two business units share one vLLM server. How do you stop one from learning the other's prompts through response timing?

Show answer

Send a different `cache_salt` for each unit, for example set by the gateway, so their prefix caches never mix. Keep vLLM updated too, because the salt has had bugs such as CVE-2026-105752.

## What to remember

1. Prompts that start the same have the same KV cache for that start. vLLM reuses it automatically.
2. A block matches only if it and every block before it match. Put fixed text first.
3. Cached blocks are free blocks not yet overwritten, removed least recently used first. No extra memory.
4. On a shared server, the cache can leak prompts through timing. Use `cache_salt`.
5. Chunked prefill splits long prompts so chats keep streaming. `--max-num-batched-tokens` sets the trade-off.

**Next: Lesson 5, making it cheaper**Quantization (8-bit and 4-bit weights, FP8) and speculative decoding: the one trick that beats the decode speed limit from Lesson 1.

[Back to Lesson 3](https://claude.ai/artifact/4Nj8k5bGo7u6JcVmTAFwtj)

Sources

- [vLLM design doc: automatic prefix caching](https://docs.vllm.ai/en/v0.25.0/design/prefix_caching/): fingerprints chained to the block before, full blocks only, LRU free list, cache_salt, "almost a free lunch"
- [vLLM design doc, v0.17](https://docs.vllm.ai/en/v0.17.0/design/prefix_caching/): SHA-256 default hash since v0.11, `--prefix-caching-hash-algo`
- [Packet.ai: vLLM prefix caching](https://packet.ai/blog/vllm-prefix-caching): on by default in V1, `--no-enable-prefix-caching`, hit-rate counters
- [vLLM docs: optimization and tuning](https://docs.vllm.ai/en/v0.11.2/configuration/optimization/): chunked prefill always on in V1, decode-first, token budget trade-off
- [vLLM docs, v0.9.2](https://docs.vllm.ai/en/v0.9.2/configuration/optimization.html): budget above about 8,000 for throughput
- [Agrawal et al., Sarathi-Serve (OSDI 2024)](https://arxiv.org/abs/2403.02310): chunked prefills, stall-free batching, multi-second stalls
- [Pennas et al., PrefixWall (2026)](https://www.alphaxiv.org/abs/2603.10726v2): timing side channels from prefix caching
- [KV cache timing side-channel write-up](https://obormot.github.io/posts/prefix-cache-timing-side-channel/): attacks shown against vLLM and SGLang
- [Tian Pan: cross-tenant leakage](https://tianpan.co/blog/2026-04-10-cross-tenant-data-leakage-llm-infrastructure): cache_salt is opt-in
- [Red Hat: CVE-2026-105752](https://access.redhat.com/security/cve/CVE-2026-105752): cache salt lost in multi-turn tool continuations