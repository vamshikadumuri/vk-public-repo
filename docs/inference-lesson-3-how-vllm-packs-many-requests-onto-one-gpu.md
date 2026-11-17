Inference track, lesson 3 of 8[Back to your prep plan](https://claude.ai/artifact/8m8maPTnDHsbPQWnqjz1bA)

# How vLLM packs many requests onto one GPU

Lesson 1 showed that batching gives more tokens per second. This lesson shows the two ideas that make batching work well in practice: continuous batching and PagedAttention.

Ten requests, four slots. Same work, two ways to schedule it.

**Static batching**done at pass **22**, slots busy **51%**

**Continuous batching**done at pass **15**, slots busy **75%**

first pass (prefill)decode passesslot sits empty

**Done.** Each row is one slot in the batch. Each square is one pass. Continuous batching finishes the same ten requests 7 passes sooner.

## Words used in this lesson

Pass

One run of the model. Each request in the batch gets one new token per pass (Lesson 1).

Batch

The group of requests that share one pass.

Slot

A place for one request in the batch.

Scheduler

The part of vLLM that decides which requests go into each pass.

Block

A small fixed piece of KV cache memory. In vLLM it holds 16 tokens.

Block table

A request's list of which blocks hold its tokens.

Preemption

Pausing a running request to free memory, then restarting it later.

## In this lesson

1. [Why simple batches waste time](#static)
2. [Continuous batching](#continuous)
3. [Why memory was wasted](#reserve)
4. [PagedAttention](#paged)
5. [Sharing blocks](#sharing)
6. [When memory runs out](#preempt)
7. [The two scheduler settings](#knobs)
8. [Check yourself](#quiz)

1

## Why simple batches waste time

The simple way to batch is **static batching**. You take a group of requests, run them together, and wait until all of them finish. Then you start the next group.

The problem is that replies have different lengths. One request may need 2 tokens, another 9. In the top row of the demo above, the short requests finish early, but their slots stay empty until the longest one is done. New requests have to wait outside.

The GPU still reads all the weights on every pass, even when some slots are empty. So empty slots are wasted work.

Key ideaIn static batching, the whole batch waits for its longest reply. The more reply lengths differ, the more slots sit empty.

2

## Continuous batching

**Continuous batching** makes a new decision on every pass. When a request finishes, it leaves at once, and a waiting request takes its slot on the next pass. Nobody waits for the slowest request.

The idea first appeared in a 2022 research system called Orca, under the name *iteration-level scheduling*. vLLM, Hugging Face TGI, TensorRT-LLM and SGLang all use it now.

### What vLLM's scheduler does on every pass

1. Remove requests that just finished, and free their memory.
2. Keep every running request in the batch, so each gets its next token.
3. Use the space that is left to start waiting requests. Long prompts can be split across passes (Lesson 4).
4. Run one pass of the model for the whole batch.

Try it: 32 requests

How much reply lengths differ

Slots in the batch

**39%**slots busy, static

**77%**slots busy, continuous

**2.0×**faster to finish all 32

A toy model: every request uses one slot for as many passes as its reply has tokens. Reply lengths are fixed sample numbers, so you get the same result each time.

When every reply has the same length, both methods tie. The gain comes from the variety in real traffic. Anyscale tested this on an A100 40GB serving the OPT-13B model. Continuous batching plus vLLM's memory handling gave up to 23 times the throughput of simple static batching.

Key ideaContinuous batching picks the batch again on every pass. Finished requests leave at once, and waiting ones join at once.

3

## Why memory was wasted

Continuous batching needs memory for every running request's KV cache (Lesson 2). Before vLLM, servers handled that memory badly.

When a request started, the server did not know how long the reply would be. So it reserved one unbroken piece of memory big enough for the *longest possible* reply. Most of that space was never used.

Three requests, each reserving room for 2,048 tokens

holds real tokensreserved, never usedfree, but in pieces too small to use

Only 22% of this memory holds real tokens. Example numbers for illustration.

There were two kinds of waste. Space reserved but never filled. And small gaps between reservations that were too small for a new request. The vLLM team measured that existing systems wasted 60% to 80% of their KV cache memory this way.

Key ideaReserving for the worst case wastes most of the memory. Less free memory means fewer requests in the batch.

4

## PagedAttention: memory in small blocks

vLLM fixes this with **PagedAttention**. It borrows an old idea from operating systems: virtual memory with pages.

- The KV cache pool is cut into small, equal **blocks**. By default, each vLLM block holds 16 tokens.
- A request gets a new block only when its last block is full.
- The blocks can be anywhere in memory. They do not need to sit next to each other.
- Each request has a **block table**: a list that says which blocks hold its tokens, in order.

Try it: GPU memory with 20 blocks of 4 tokens each

**18**tokens stored

**6**blocks in use

**6**empty token slots, all in last blocks

Requests A, B and C are running. Their blocks are mixed together in memory, and that is fine.

4 tokens per block keeps the picture small; vLLM uses 16 by default. In the block table, "A1→3" means request A's second block is stored in memory block #3 (counting starts at 0).

The only empty space is inside each request's last block. The vLLM team reported that this cuts waste to under 4%. More free memory means more requests fit in the batch, and that is where the speed comes from. In the vLLM paper, this gave 2 to 4 times the throughput of earlier systems such as FasterTransformer and Orca, at the same latency.

One cost: the attention code must look up the block table to find each token's keys and values. vLLM wrote special GPU code for this, which is where the name *PagedAttention* comes from.

Key ideaBlocks are given out as tokens arrive, from anywhere in memory. Waste is less than one block per request.

5

## Sharing blocks

Block tables have a second benefit. Two requests can point to the *same* block. vLLM counts how many requests use each block.

Example: you ask for 3 different replies to one prompt (sampling with `n=3`). The prompt's blocks are stored once and shared. Each reply gets its own blocks only for the new tokens.

One prompt, three replies

Reply 1

P1P2P3R1

Reply 2

P1P2P3R2

Reply 3

P1P2P3R3

In memory

P1 ×3P2 ×3P3 ×3R1R2R3

6 blocks in memory instead of 12. "×3" means three replies use that block.

If one reply needs to write into a shared block, vLLM first copies that block, so the others keep theirs unchanged. This is called **copy-on-write**. Beam search keeps several candidate replies and picks the best one. For sampling and beam search, the vLLM team reported up to 55% less memory and up to 2.2 times the throughput.

The same idea lets many requests share a common start, like one system prompt. That is **prefix caching**, in Lesson 4.

Key ideaA block can belong to many requests. Shared text is stored once.

6

## When memory runs out

vLLM lets a request in based on the memory it needs *now*, not at its longest. This packs in many more requests. The risk: running requests keep growing, and the pool can fill up in the middle of their replies.

When there is no free block for a running request's next token, vLLM **preempts**. It pauses one or more running requests and frees their blocks. Later, when memory is free again, it starts those requests over and computes their KV cache again. In vLLM V1, recomputing is the default way to do this.

The pool is full and request E needs one more block

A

B

C

D

E

F

running requestsF is paused, and its blocks go to E

WARNING ... preempted by PreemptionMode.RECOMPUTE mode because there is not enough KV cache space.

Log text from the vLLM docs, shortened. Newer versions also print a running count, `total_cumulative_preemption_cnt`.

A few preemptions are fine. Many of them slow everything down, because paused requests redo their work. The vLLM docs suggest these fixes:

- Raise `--gpu-memory-utilization`, so the pool is bigger.
- Lower `--max-num-seqs` or `--max-num-batched-tokens`, so fewer requests run at once.
- Raise `--tensor-parallel-size`, so the weights are split across GPUs and more memory is left for the cache.

Key ideaPaging lets vLLM pack requests tightly. If they all grow at once, it pauses some and redoes them later.

7

## The two scheduler settings

Two flags set limits for each pass:

- `--max-num-seqs`: the most requests in one pass.
- `--max-num-batched-tokens`: the most tokens in one pass. This mostly limits how much prompt work fits in a pass (Lesson 4).

Their defaults depend on your vLLM version and setup, so read them from your startup log or `vllm serve --help`.

How many requests actually run at once? It is the smaller of two numbers: `--max-num-seqs`, and how many requests fit in the KV pool.

Try it: how many requests run at once?

KV pool (from Lesson 2)

Average tokens per request, prompt plus reply: 1,500

`--max-num-seqs`

"Old way" = every request reserves room for 8,192 tokens, like the servers in step 3. Averages hide growth: if you are close to the memory limit, expect some preemption.

Key ideaRequests running at once = the smaller of `--max-num-seqs` and (KV pool ÷ tokens per request).

## Check yourself

Answer each one first, out loud or in chat, then open the answer.

1A static batch has three replies of 50, 60 and 400 tokens. After how many passes can the slot of the 50-token reply take a new request? And with continuous batching?

Show answer

Static: after 400 passes, when the whole batch is done. Continuous: right after pass 50. A waiting request joins on the next pass.

2Why does continuous batching help very little when every reply has the same length?

Show answer

All requests finish on the same pass, so no slot is ever left empty. The gain comes from replies of different lengths.

3vLLM uses blocks of 16 tokens. A request holds 1,000 tokens. How many blocks does it use, and how many token slots are empty?

Show answer

1,000 ÷ 16 = 62.5, so 63 blocks. They hold 63 × 16 = 1,008 tokens, so 8 slots are empty, all in the last block.

4Your attacker server's log shows many preemption warnings. Name two fixes.

Show answer

Lower `--max-num-seqs` (or `--max-num-batched-tokens`) so fewer requests run at once. Or make the pool bigger: raise `--gpu-memory-utilization` if your GPU share allows it, or split the model across GPUs with `--tensor-parallel-size`. Lesson 2's fixes also help, such as FP8 KV cache.

5An A100 80GB has a KV pool of 411,529 tokens. Requests average 1,500 tokens. `--max-num-seqs` is 256. How many requests run at once? How many would run if each reserved 8,192 tokens?

Show answer

The pool fits 411,529 ÷ 1,500 = 274 requests, but `--max-num-seqs` caps it at 256. If each reserved 8,192 tokens: 411,529 ÷ 8,192 = 50 requests. Paging runs about 5 times as many.

## What to remember

1. Static batching waits for the longest reply, so slots sit empty.
2. Continuous batching picks the batch again on every pass. Finished requests leave and waiting ones join at once.
3. Old servers reserved memory for the longest possible reply and wasted 60% to 80% of it.
4. PagedAttention gives out 16-token blocks as needed, from anywhere in memory, tracked by a block table. Waste drops under 4%.
5. When the pool fills up, vLLM preempts: it pauses requests and recomputes them later. Watch for it in your logs.

**Next: Lesson 4, prefix caching and chunked prefill**How a shared system prompt is stored once and reused. And how long prompts are split so they don't hold up everyone else's tokens.

[Back to Lesson 2](https://claude.ai/artifact/7jBvi95ykBeCHw6HDrrRrP)

Sources

- [Kwon et al., PagedAttention paper (SOSP 2023)](https://huggingface.co/papers/2309.06180): near-zero waste, sharing, 2 to 4 times the throughput of FasterTransformer and Orca
- [vLLM launch blog (2023)](https://vllm.ai/blog/2023-06-20-vllm): 60% to 80% waste in existing systems, under 4% with paging, block tables, copy-on-write, up to 55% less memory and 2.2 times the throughput for sampling
- [Anyscale: continuous batching](https://anyscale.com/blog/continuous-batching-llm-inference): static batching's problem, Orca as the first system, up to 23 times the throughput on OPT-13B and an A100 40GB
- [vLLM docs: optimization and tuning](https://docs.vllm.ai/en/v0.9.0/configuration/optimization.html): preemption, RECOMPUTE as the V1 default, and the fixes
- [vLLM docs, later version](https://docs.vllm.ai/en/v0.20.0/configuration/optimization/): preemption warning text, decode-first scheduling with chunked prefill
- [PremAI: static vs continuous batching](https://www.premai.io/blog/llm-batching-static-vs-continuous-and-why-it-matters-for-throughput/): which engines use continuous batching
- [vLLM engine arguments](https://vllm.readthedocs.io/en/latest/models/engine_args.html): what `--max-num-seqs` and `--max-num-batched-tokens` limit, 16-token default block size