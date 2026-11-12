Inference track, lesson 1 of 8[Back to your prep plan](https://claude.ai/artifact/8m8maPTnDHsbPQWnqjz1bA)

# How an LLM answers a prompt

An LLM reads your whole prompt in one go, then writes its answer one token at a time. Watch it happen, then we'll take each part apart.

Prompt

Reply

Each block is one pass through the model19 passes

**Done.** 1 prefill pass read all 10 prompt tokens, then 18 decode passes wrote the reply, one token each.

Prefill: the prompt, all at onceDecode: the reply, one token per pass

## In this lesson

1. [Text becomes tokens](#tokens)
2. [One token per pass](#loop)
3. [Prefill and decode](#phases)
4. [The decode speed limit](#speed-limit)
5. [Batching](#batching)
6. [The numbers you'll be asked](#metrics)
7. [Picking the next token](#sampling)
8. [Check yourself](#quiz)

1

## Text becomes tokens

The model never sees letters. A tokenizer cuts text into **tokens**, chunks that are often a whole word and sometimes part of one, and turns each into a number the model can work with.

One sentence, cut into tokens

**7**tokens for 5 words in this sentence

**128,256**tokens in Llama 3.1's vocabulary

**≈ ¾**of an English word per token, on average

Illustrative split. Each tokenizer cuts a little differently, but long or rare words always break into pieces.

Everything in inference is counted in tokens: prompt tokens in, output tokens out, and price per million tokens. A 1,000-token prompt is roughly 750 words.

Key ideaTokens are the unit of work. Every speed and cost number in this track is per token.

2

## One token per pass

An LLM does exactly one thing: given all the tokens so far, it gives a score to every token it knows. The engine picks one, adds it to the end, and runs the whole model again. This loop is called **autoregressive generation**.

Llama 3.1 8B shown. The loop ends when the model picks its stop token, hits a stop string, or reaches `max_tokens`.

### Watch the input grow

Here are the first four passes of the reply from the top of the page. Each row is one full trip through the model.

new computed this passold reused from the KV cache

Recomputing every earlier token on every pass would be hugely wasteful. So the engine saves each token's attention **keys and values** the first time it computes them. That store is the **KV cache**. From pass 2 on, only the newest token needs fresh work.

The catch is memory. For Llama 3.1 8B the cache costs 128 KiB per token, so one 4,096-token conversation holds 512 MiB. Lesson 2 is all about this.

Key ideaA 300-token answer means 300 passes in a row. They can't run side by side, because each pass needs the token before it.

3

## Prefill and decode

Those passes are not all alike. The first pass, **prefill**, pushes the whole prompt through at once. Every pass after it, **decode**, pushes through a single token. They run into completely different limits.

Each pass has two jobs: stream all the weights out of GPU memory, and do the math with them. Whichever job takes longer sets the speed.

Where the time goes in one pass

**Prefill pass, 1,000-token prompt**Limited by math

Do the math51 ms

Read all weights from memory7.9 ms

**Decode pass, 1 token**Limited by memory

Do the math0.05 ms

Read all weights from memory7.9 ms

01020304050 ms

All bars on one scale. Llama 3.1 8B in 16-bit on an A100 80GB SXM, at the card's peak: 2,039 GB/s of memory bandwidth and 312 TFLOPS. Decode's math is so small it barely shows.

Think of a librarian who has to walk past every shelf to answer each question. For one question, the walk is all the work. Prefill brings a thousand questions on one walk, so the thinking finally dominates.

|  | Prefill | Decode |
| --- | --- | --- |
| Runs | Once per request | Once per output token |
| Tokens per pass | The whole prompt | One |
| Limited by | Math (FLOPs) | Memory bandwidth |
| Sets | Time to first token | Time per output token |
| Grows with | Prompt length | Reply length |

Key ideaPrefill is limited by math. Decode is limited by how fast the weights stream out of GPU memory.

4

## The decode speed limit

Every decode pass reads all the weights, so with plain decoding a single request can't write tokens faster than memory delivers them:

max tokens/s ≈memory bandwidthsize of the weights

Try it: one request, alone on the GPU

GPU

Model

Weight precision

**16.1 GB**weights read per token

**127**tokens/s, best case

**7.9 ms**per token, at least

Same model on each GPU (tap a bar to pick it)

Ceilings, not measurements. Real engines land below them: in a Databricks example, a 7B model used about half of the GPU's bandwidth.

Notice what moves the number: a GPU with more bandwidth, or fewer bytes per weight. Switching to 8-bit weights nearly doubles the ceiling. A GPU with more FLOPs but the same bandwidth would not help at all. The one trick that bends this ceiling is speculative decoding, in Lesson 5.

Key ideaPer-request decode speed ≈ bandwidth ÷ weight bytes. To go faster, buy bandwidth or shrink the weights.

5

## Batching: one read, many tokens

Here is the escape hatch. If 32 requests decode together, the weights are read once per pass and serve all 32 of them. Total output multiplies, while each user's pass gets only a little slower.

It isn't free. Every request in the batch brings its own KV cache, which also has to be read each pass and has to fit in memory.

Llama 3.1 8B, 16-bit, one A100 80GB SXM

Requests decoding together: 32 1, 2, 4 … 256

Tokens in each conversation so far

**3,210**tokens/s in total

**100**tokens/s per user

**4.3 GB**of KV cache in memory

#### Total throughput, tokens/s

#### Speed each user sees, tokens/s

Ideal ceilings from the same bandwidth math, counting weights plus KV cache, with vLLM's default of 90% of GPU memory. Drag across a chart to scrub. Real systems give up more: Databricks measured about 14× the throughput and 4× slower replies at batch size 64 on one A100.

Key ideaBatching trades a little speed per user for a lot of total throughput. Lesson 3 shows how vLLM's scheduler makes that trade on every pass.

6

## The numbers you'll be asked about

Users feel two numbers. Whoever pays for the GPU cares about a third.

- TTFTtime to first token

  The wait from sending a prompt to seeing the first token. Set by prefill, plus any time spent waiting in a queue.
- TPOTtime per output token

  The average gap between tokens after the first. Set by decode. 100 ms per token feels like 10 tokens per second.
- ITLinter-token latency

  Each single gap between two tokens. It averages out to TPOT, but its spread shows jitter that the average hides.
- E2Eend-to-end latency

  The time for the whole reply.
- Throughputtokens/s, all users

  Output tokens per second across every request on the server. This is what sets cost per million tokens.

E2E ≈ TTFT + TPOT × (N − 1)

Build one request's timeline

TTFT: 100 ms

TPOT: 10 ms

Output tokens: 300

Prefill **0.10 s**Decode **2.99 s**Total **3.09 s**

Decode is 97% of the wait. Here, capping `max_tokens` saves far more time than trimming the prompt.

In an interview, ask which number matters before you tune anything. A chat app lives or dies on TTFT. A batch job, like your overnight red-team run, cares about throughput, because nobody watches those replies stream.

Key ideaTTFT comes from prefill, TPOT from decode, and throughput from batching. Know which one your users feel.

7

## Picking the next token

Every pass ends with one score per token in the vocabulary. **Sampling settings** decide how a winner is picked from those scores. Play with them below.

After Decode speed is limited by the model's top candidates are:

Temperature: 0.7

Top-p: 0.90

Your picks will show here

Made-up scores for illustration. Temperature 0 always takes the top token (greedy). Higher temperature flattens the odds. Top-p keeps only the most likely tokens whose chances add up to p.

Sampling is cheap next to the forward pass, so it changes *what* you get, not how fast. For your attacker model, a higher temperature gives more varied attack prompts, and more that miss.

Key ideaSpeed comes from the pass. Sampling only chooses among the scores the pass produced.

## Check yourself

Answer each one out loud or in chat first, then open the answer.

1Why does writing a 2,000-token answer take far longer than reading a 2,000-token prompt?

Show answer

The answer needs 2,000 decode passes in a row, each reading all 16 GB of weights: at least 2,000 × 7.9 ms ≈ 16 s on an A100 80GB SXM. The prompt goes through prefill in one parallel pass, about 0.1 s of math at the card's peak.

2You switch your attacker model from 16-bit to 8-bit weights. Which improves more, TTFT or TPOT?

Show answer

TPOT. Decode reads every weight each pass, so halving the bytes nearly doubles the ceiling, from about 127 to about 254 tokens per second. Prefill is limited by math, so TTFT moves much less.

3A teammate says: "The GPU shows 5% compute use during decode, so it's oversized." What do you tell them?

Show answer

Low compute use is normal in decode: the GPU is waiting on memory, not math. Look at memory-bandwidth use instead, and batch more requests to put the idle compute to work. A smaller card with less bandwidth would make every token slower.

4Your red-team job sends 500 attack prompts overnight. Do you tune for TTFT or throughput?

Show answer

Throughput. Nobody watches those replies stream, so you want the most attacks finished per GPU-hour. Let the engine batch as many prompts as memory allows and accept a slower individual reply.

5Estimate the best single-request decode speed for Llama 3.1 8B in 16-bit on an H100 SXM, which moves 3.35 TB/s.

Show answer

3,350 GB/s ÷ 16.06 GB ≈ 209 tokens per second, or about 4.8 ms per token. Try it in the calculator in step 4.

## What to remember

1. An LLM writes one token per pass, and the passes run one after another.
2. Prefill reads the whole prompt in one pass. It is limited by math and sets TTFT.
3. Decode writes one token per pass. It is limited by memory bandwidth and sets TPOT.
4. One request's ceiling is bandwidth ÷ weight bytes. Batching lifts total throughput past it.
5. Ask which number matters first: TTFT for chat, throughput for batch jobs.

**Next: Lesson 2, KV cache and GPU memory math**How many tokens fit on your GPU, and what `--gpu-memory-utilization` and `--max-model-len` really control in your attacker-model image.

Sources

- [NVIDIA A100 datasheet](https://nvidia.com/content/dam/en-zz/Solutions/Data-Center/a100/pdf/nvidia-a100-datasheet-us-nvidia-1758950-r4-web.pdf): bandwidth and TFLOPS per variant
- [NVIDIA H100 specifications (Colfax)](https://www.colfax-intl.com/nvidia/nvidia-h100): H100 SXM bandwidth and memory
- [Llama 3.1 8B Instruct config](https://huggingface.co/snx999/dev3/blob/main/config.json): layers, KV heads, head size
- [Databricks: LLM inference performance engineering](https://www.databricks.com/blog/llm-inference-performance-engineering-best-practices): TTFT, TPOT, batching measurement, bandwidth use
- [DigitalOcean: LLM inference benchmarking](https://www.digitalocean.com/blog/llm-inference-benchmarking): TPOT vs ITL, latency formula
- [vLLM engine arguments](https://vllm.readthedocs.io/en/latest/models/engine_args.html): `--gpu-memory-utilization` default 0.9