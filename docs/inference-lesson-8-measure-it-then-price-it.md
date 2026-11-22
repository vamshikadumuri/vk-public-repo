Inference track, lesson 8 of 8[Back to your prep plan](https://claude.ai/artifact/8m8maPTnDHsbPQWnqjz1bA)

# Measure it, then price it

A good benchmark answers one question: how much traffic can this setup take while still meeting its latency targets? Once you know that, the cost per token is simple division.

Try it: raise the traffic on one A100 until it misses its targets

Workload

Target: 95% of first tokens within

Target: 95% of replies at most this per token

Time to first token (log scale)

Time per output token (log scale)

median (p50)p95targetmost load that meets both

Touch or hover the charts to read each load level.

**7.5**requests/s within both targets

**1,877**output tokens/s at that load

**$0.30**per million output tokens at $2/GPU-hour

A toy model, not a measurement: Llama 3.1 8B in 16-bit on an A100 80GB. It assumes 80% of the GPU's memory speed and 60% of its math speed. Chunked prefill with 2,048 tokens per step, up to 256 requests at once, random arrivals, 1,000 requests per load level.

## Words used in this lesson

Request rate

New requests arriving per second.

Concurrency

Requests in progress at the same moment.

Percentile

p95 = 95% of requests were this fast or faster. p50 is the median.

SLO

Service level objective: a latency target, such as "p95 first token under 1 s".

Goodput

Requests per second that met every target.

Sweep

Running the same test at several load levels.

## In this lesson

1. [The real question](#question)
2. [Read percentiles, not averages](#pct)
3. [Request rate versus concurrency](#rate)
4. [The curve and its knee](#curve)
5. [Traps to avoid](#traps)
6. [Cost per million tokens](#cost)
7. [Write it up](#writeup)
8. [Check yourself](#quiz)
9. [The whole track](#track)

1

## The real question

"How fast is it?" has two very different answers. Look at the chat workload above:

- **Maximum throughput:** push as hard as possible and it serves a little under 9 requests per second. But first tokens then take seconds and replies crawl.
- **Capacity within targets:** with first tokens under 1 s and 50 ms per token, it serves about 7.5.

The second number is the one you plan with. It depends on your targets, so write them down before you test. Then test at many load levels, not one.

Key ideaA benchmark result is "X requests per second *at* these latency targets". A number without targets isn't useful.

2

## Read percentiles, not averages

Lesson 1 named the latency numbers: time to first token (TTFT), time per output token (TPOT), and the gap between tokens (ITL). Add end-to-end latency (E2EL), the whole request.

Each one is a spread, not a single value. An average hides the slow tail. Report percentiles instead:

- **p50** is the typical user.
- **p95** and **p99** are the unlucky 1 in 20 and 1 in 100. That's where queueing shows up first.

In the demo, the p95 line rises well before the median does. `vllm bench serve` reports only p99 by default. Ask for more with `--metric-percentiles 50,95,99`.

Key ideaSet targets on p95 or p99. The median looks fine long after users start to suffer.

3

## Request rate versus concurrency

There are two ways to load a server:

**Request rate (open)**New requests arrive on a clock, whether or not the server keeps up. Like real users.

**Concurrency (closed)**A fixed number of users. Each sends the next request only after its answer.

The two are linked by Little's law: requests in flight = arrival rate × time each request takes. In the demo at 7 requests per second, each chat request takes about 8.4 s, so about 60 are in flight.

The closed style can flatter a server. When it slows down, users wait for answers, so fewer new requests arrive. The vLLM docs warn about this: with a concurrency cap, the real rate can fall below the rate you asked for.

Use request rate to find capacity for user-facing traffic. Use concurrency when there really is a fixed number of clients, such as 16 red-team workers. Note that `vllm bench serve` defaults to an infinite rate: every request at once. That measures maximum throughput, not latency under normal load.

Key ideaOpen load (`--request-rate`) shows queueing honestly. Closed load (`--max-concurrency`) fits a fixed set of workers.

4

## The curve and its knee

Plot latency against load and you get the shape in the demo. It is flat at first, then bends sharply upward. That bend is the knee: past it, requests arrive faster than they finish, and the queue only grows.

Two details in the toy model are worth a second look:

- For chat, the per-token target tends to break first. More requests share each step, and prefill chunks join those steps.
- Long prompts cut capacity hard. Each request has 4 times the prefill work, so first tokens take longer from the start.

Tools can do the sweep for you. GuideLLM's default `sweep` runs one request at a time, then full speed, then 10 fixed rates in between. With `vllm bench serve`, run it once per rate, or pass `--goodput ttft:1000 tpot:50` to count requests that met both targets.

```
guidellm benchmark --target http://localhost:8000 \
  --rate-type sweep --max-seconds 60 \
  --data "prompt_tokens=1000,output_tokens=250"
```

Key ideaFind the knee, then stay below it. Your capacity is where the first p95 target breaks.

5

## Traps to avoid

1. **Unrealistic lengths.** Use your real input and output lengths. They decide everything, as the two workloads in the demo show.
2. **Replies that stop early.** With random prompts the model may stop after a few tokens. `--ignore-eos` makes every reply the length you set.
3. **Wrong prefix sharing.** Random prompts share nothing, so prefix caching never helps. If your traffic shares a system prompt, add it with `--random-prefix-len`.
4. **No warm-up.** `--num-warmups` is 0 by default. Send some requests first, and leave them out of the results.
5. **Runs too short.** Past the knee, results depend on how long you run. Use enough requests, such as 1,000 per level.
6. **Missing details.** Record the vLLM version, every flag and the GPU. Without them nobody can repeat the test, including you.

A full command for one load level:

```
vllm bench serve \
  --model meta-llama/Llama-3.1-8B-Instruct \
  --dataset-name random \
  --random-input-len 1000 --random-output-len 250 \
  --random-prefix-len 2000 --ignore-eos \
  --request-rate 6 --num-prompts 1000 --num-warmups 50 \
  --percentile-metrics ttft,tpot,itl,e2el \
  --metric-percentiles 50,95,99 \
  --goodput ttft:1000 tpot:50 \
  --save-result
```

Key ideaA benchmark is only as good as how closely its traffic matches yours.

6

## Cost per million tokens

Once you know the throughput at your targets, cost is one formula:

cost per million tokens = GPU price per hour ÷ (tokens per second × 3,600) × 1,000,000

Try it with the capacity you found above

GPU price per hour

$2.00

Share of the day the GPU is busy at that load

**$0.30**per million output tokens

**$0.06**per million tokens, input included

**$48**per day for the GPU

A100 80GB rental prices ranged from about $1.19 to $5.07 per GPU-hour on demand in one recent comparison. Use your own internal rate if you have one.

Three things the calculator shows:

- **Idle time is expensive.** A GPU busy a quarter of the day costs 4 times as much per token. Autoscaling (Lesson 7) and sharing (Lesson 6) attack this.
- **Say which tokens you count.** Chat requests here read 4 input tokens per output token. API price lists usually show input and output tokens separately.
- **Your targets set the price.** Looser targets allow more load, so each token costs less.

For your 0.3 share, use 0.3 × the GPU price, and measure throughput on the share itself, not a whole card.

Key ideaCost per token = price ÷ useful throughput. Raise throughput at your targets, and keep the GPU busy.

7

## Write it up

A one-page template. Fill in every line; the empty ones show what you forgot to test.

Goal

Most requests/s with p95 TTFT ≤ 1 s and p95 TPOT ≤ 50 ms

Model

Name and precision (BF16, FP8, INT4)

Hardware

GPU type and count, or the slice (MIG, share)

Server

vLLM version and every flag

Traffic

Input and output lengths, shared prefix, dataset

Load

Request rates tested, requests per rate, warm-ups

Client

Where it ran; streaming on or off

Results

Per rate: TTFT and TPOT p50, p95, p99; output tokens/s; errors

Answer

Capacity X req/s (Y tokens/s); $Z per million output tokens at $P per GPU-hour

Limits

What ran out first: KV cache, math, or queue?

Next

One change to try, and why

Key ideaA benchmark nobody can repeat is an anecdote. Write down everything.

## Check yourself

Answer each one first, out loud or in chat, then open the answer.

1Median TTFT is 0.3 s, but p99 is 4 s. What does that tell you?

Show answer

Most users are fine, but 1 in 100 waits 4 seconds or more. That usually means queueing at busy moments, or long prompts holding others up. You are probably near the knee.

2A test with `--max-concurrency 32` showed great latency. In production, users complain. Why?

Show answer

A concurrency cap is closed load: when the server slows, fewer new requests arrive, so the queue never builds. Real users keep arriving. Retest with `--request-rate` at several rates.

3A GPU costs $2 per hour and gives 1,800 output tokens/s at your targets. What does a million output tokens cost? And if it's busy only a quarter of the day?

Show answer

2 ÷ (1,800 × 3,600) × 1,000,000 ≈ $0.31. Busy a quarter of the day: $0.31 ÷ 0.25 ≈ $1.23.

4Your test used random prompts. Production prompts share a 2,000-token system prompt. Which way is your result off?

Show answer

Too pessimistic. In production, prefix caching skips most of that shared prefill (Lesson 4), so first tokens come sooner. Add `--random-prefix-len 2000`, or use real prompts.

5At the same targets, the toy model's capacity drops from about 7.5 to 1.4 requests per second with long prompts. Why?

Show answer

Each long request has 4,000 prompt tokens to prefill instead of 1,000. Prefill is math-bound (Lesson 1), so each request costs about 3 to 4 times more GPU time. First tokens also start slower.

## What to remember

1. The useful result is capacity at your latency targets, not peak throughput.
2. Set targets on p95 or p99; the median hides the tail.
3. Use request rate for user traffic; concurrency caps hide queueing.
4. Sweep the load and find the knee. Stay below it.
5. Match real lengths and prefixes, use `--ignore-eos`, warm up, record everything.
6. Cost per million tokens = price per hour ÷ (tokens/s × 3,600) × 1,000,000, adjusted for idle time.

[Back to Lesson 7](https://claude.ai/artifact/BiKCvaaG54E7RXHftZdQje)

## The whole track

You've finished the inference track. Here it is in one list, to review before an interview.

1. [How an LLM answers a prompt](https://claude.ai/artifact/USZqxTneUzroaqdQZYrx37)Prefill and decode, TTFT and TPOT, the memory-speed limit
2. [How many tokens fit on your GPU?](https://claude.ai/artifact/7jBvi95ykBeCHw6HDrrRrP)KV cache size, GQA, vLLM's memory budget
3. [How vLLM packs many requests onto one GPU](https://claude.ai/artifact/4Nj8k5bGo7u6JcVmTAFwtj)Continuous batching, PagedAttention, preemption
4. [Don't compute the same prompt twice](https://claude.ai/artifact/4X7ppjDXD2CJ28WjeJq8MS)Prefix caching and chunked prefill
5. [Making every token cheaper](https://claude.ai/artifact/EVM9BQ9JuJfBjvBiwY1UHD)Quantization and speculative decoding
6. [One GPU is rarely the right size](https://claude.ai/artifact/NzSi3BVZim2ETkfvjtpwPf)Tensor and pipeline parallel, MIG and time-slicing
7. [From one server to a service](https://claude.ai/artifact/BiKCvaaG54E7RXHftZdQje)KServe, llm-d routing, autoscaling, cold starts
8. Measure it, then price itBenchmarks, percentiles, capacity and cost (this lesson)

Sources

- [vLLM docs: vllm bench serve](https://docs.vllm.ai/en/v0.32.0/cli/bench/serve/): request rate (default infinite, Poisson), burstiness, concurrency cap warning, percentile defaults, goodput, ignore-eos, warm-ups, prefix length
- [GuideLLM](https://pypi.org/project/guidellm/) and [its GitHub page](https://github.com/vllm-project/guidellm): load profiles, the default sweep
- [Beam: A100 80GB pricing](https://www.beam.cloud/blog/nvidia-a100-80gb-pricing): $1.19 to $5.07 per GPU-hour on demand
- Toy model numbers come from Lessons 1 and 2: A100 80GB SXM memory speed 2,039 GB/s, 312 TFLOPS 16-bit math, 128 KiB of KV cache per token for Llama 3.1 8B