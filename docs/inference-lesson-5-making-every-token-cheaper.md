Inference track, lesson 5 of 8[Back to your prep plan](https://claude.ai/artifact/8m8maPTnDHsbPQWnqjz1bA)

# Making every token cheaper

Two tricks. Quantization stores the model with smaller numbers, so there is less to read on every pass. Speculative decoding lets the big model accept several tokens in one pass.

Speculative decoding: a small model guesses 4 tokens, the big model checks them all in one pass

guess small model's guesskept big model agreeswrong big model disagreesown big model's own token

**19**big-model passes, normal decoding

**5**big-model passes, speculative

**Done.** Same reply as Lesson 1, in 5 big-model passes instead of 19. Each pass checks the guesses and always adds one token of its own.

## Words used in this lesson

Quantization

Storing numbers with fewer bits, such as 8 or 4 instead of 16.

Weights

The model's fixed numbers, read on every pass.

Activations

The numbers that flow through the model while it runs.

W8A8, W4A16

Short names: W = bits for weights, A = bits for activations.

Draft model

A small, fast model that guesses the next few tokens.

Acceptance rate

How often the big model agrees with a guess.

## In this lesson

1. [Fewer bits per number](#bits)
2. [The common formats](#formats)
3. [Which format for which job](#which)
4. [Getting a quantized model](#get)
5. [Speculative decoding](#spec)
6. [How much faster?](#math)
7. [When it helps, and how to turn it on](#when)
8. [Check yourself](#quiz)

1

## Fewer bits per number

Lesson 1 showed that each decode pass reads every weight from memory. Lesson 2 showed that weights take memory away from the KV cache. Both get better if each weight takes fewer bytes.

A 16-bit number can take 65,536 different values. An 8-bit number can take 256, and a 4-bit number only 16. To store a weight in fewer bits, we round it to the nearest value the smaller format can hold. That rounding is the only loss.

Try it: round one weight

Weight value: 0.37

−10+1

**0.370**stored in 8 bits (256 levels)

**0.429**stored in 4 bits (16 levels)

**0.059**4-bit rounding error

Tall ticks are the 16 values a 4-bit number can hold; the orange dot is where the weight lands. Real methods add a scale per group of weights. Smarter tools (GPTQ, AWQ) also choose the rounding to protect the weights that matter most.

Key ideaQuantization rounds each number to fewer levels. Fewer bits means less to read and more room for the KV cache.

2

## The common formats

Formats are named by what gets fewer bits. "W8A8" means 8-bit weights *and* 8-bit activations, so the math itself runs in 8 bits. "W4A16" means 4-bit weights but 16-bit math: weights are unpacked just before they are used.

| Format | 8B model size, roughly | Accuracy (Llama 3.1 study) | On an A100 |
| --- | --- | --- | --- |
| **BF16** (W16A16) | 16 GB | The baseline | Yes |
| **FP8** W8A8 | 8 GB | Essentially lossless | Runs as 8-bit weights, 16-bit math |
| **INT8** W8A8 | 8 GB | 1% to 3% lower when tuned well | Yes, with 8-bit math |
| **INT4** W4A16 (GPTQ, AWQ) | 4 GB | Competitive, close to 8-bit | Yes |

The accuracy column comes from a Red Hat study of over 500,000 evaluations on the Llama 3.1 family. One detail matters for your A100: it has no FP8 math units. vLLM still runs FP8 models on it, but as weight-only (W8A16). You get the memory and decode-speed savings, not faster math. FP8 math needs newer GPUs such as the H100.

Key ideaW = weight bits, A = activation bits. 8-bit is near lossless; 4-bit weights cost a little more accuracy for half the size again.

3

## Which format for which job

Recall Lesson 1. With few requests, decode is limited by *reading* weights, so the smallest weights win. With a big batch, the GPU becomes limited by *math*, so faster 8-bit math wins.

The Red Hat study found the same. W4A16 was the most cost-efficient for one-request-at-a-time use. W8A8 did best for busy servers using continuous batching on high-end GPUs.

Try it: what does each format buy you?

Model

Weight format

Memory vLLM may use

**16.1 GB**weights

**54 GB**left for the KV cache

**127**tokens/s ceiling, one request, A100 80GB

Same math as Lessons 1 and 2: about 2 GB assumed for activations, and the speed ceiling = 2,039 GB/s ÷ weight size. Real files are a little bigger: scale factors are left out, and a few layers, such as the embeddings, usually stay 16-bit.

Key ideaFew users: smallest weights (W4A16). Busy server on new GPUs: 8-bit math (W8A8).

4

## Getting a quantized model

1. **Download one.** Many models already have FP8, INT8 and INT4 versions on Hugging Face. vLLM reads the format from the model's config and loads it.
2. **Make one.** The `llm-compressor` library writes quantized models for vLLM. Simple FP8 needs no sample data; INT8 and INT4 methods use a small set of sample prompts to choose the rounding.
3. **Quick test.** `--quantization fp8` converts a 16-bit model to FP8 as it loads.
4. **Always measure.** Run your own evals before and after. For your attacker model, compare attack success rates, not just general benchmarks.

**Worked example: your attacker model**

Lesson 2 assumed vLLM gets about 24 GB. In 16-bit, about 6 GB is left for the KV cache: roughly 45,000 tokens, so the default 131,072-token limit won't start.

With FP8 weights (8 GB) and an FP8 KV cache, about 14 GB is left, and each token takes half the space. That holds about 213,000 tokens, so even the default limit fits. Check attack quality first.

Key ideaUse a ready-made quantized model when one exists, and prove quality with your own evals.

5

## Speculative decoding

Lesson 1 said decode is limited by reading weights, and the math is tiny. So checking 5 tokens in one pass costs about the same as making 1. Speculative decoding uses this:

1. A small, fast **draft** model guesses the next few tokens.
2. The big model reads all the guesses in *one* pass, the way prefill reads a prompt.
3. It keeps guesses from the start for as long as they match what it would have produced.
4. At the first wrong guess, it uses its own token instead. If all guesses pass, it adds one more of its own.

The paper that introduced it proved that the output follows exactly the same probabilities as normal decoding. It showed a 2 to 3 times speedup on a large model, with identical outputs. You get speed without changing quality.

Key ideaThe draft guesses, the big model checks many tokens in one pass. Every pass yields at least one token, often several.

6

## How much faster?

It depends on two things: how often guesses are right (the acceptance rate), and how many tokens you guess each round. Guessing more helps only while guesses keep being right, and the draft model costs time too.

tokens per pass = (1 − ak+1) ÷ (1 − a)

a = acceptance rate, k = tokens guessed per round. From the speculative decoding paper.

Try it

Acceptance rate: 0.80

Tokens guessed per round: 4

Draft model's cost per guess, vs one big pass: 10%

**3.36**tokens per big pass

**2.4×**faster, after draft cost

**6**best number of guesses here

Speed-up for each number of guesses

12345678

Speed-up = tokens per pass ÷ (1 + guesses × draft cost), the paper's formula. Ideal case: one request, decode limited by memory.

Key ideaHigh acceptance and a cheap draft give big wins. Low acceptance can make it slower than normal.

7

## When it helps, and how to turn it on

Speculative decoding spends spare math to save passes. With few requests the GPU has lots of spare math, so it helps most. With a big batch the GPU is already busy with math (Lesson 1), and the extra checking work competes with real work. The vLLM guide says it is for cutting inter-token latency at low to medium load.

vLLM supports several ways to make guesses:

- EAGLE, MTP

  Small extra heads trained for one model, or built into it. Usually the biggest gains.

  Needs a trained head for your exact model.
- Draft model

  A smaller model of the same family, such as a 1B model drafting for an 8B one.

  Uses extra GPU memory for the second model.
- N-gram

  Guesses by copying text that already appears in the prompt. No extra model.

  Small gains, but good when outputs repeat inputs, like rewriting or summarizing.

Turn it on with one flag. For example: `--speculative-config '{"method": "ngram", "num_speculative_tokens": 4}'`. Then benchmark at your real load (Lesson 8). The gain can disappear when the server is busy.

Key ideaSpeculative decoding helps when the GPU has spare math: low load, latency-focused serving.

## Check yourself

Answer each one first, out loud or in chat, then open the answer.

1Llama 3.1 8B in INT4 (W4A16): roughly how big are the weights, and what is the single-request decode ceiling on an A100 80GB SXM?

Show answer

8.03 billion × 0.5 bytes ≈ 4 GB. Ceiling ≈ 2,039 GB/s ÷ 4 GB ≈ 508 tokens per second, 4 times the 16-bit model's 127.

2You load an FP8 model on your A100. What do you gain, and what don't you gain?

Show answer

You gain half-size weights: more KV cache room and faster decode at low load. You don't gain faster math, because the A100 has no FP8 math units. vLLM runs it as weight-only (W8A16).

3A busy server runs batches of 256 on H100s. W4A16 or W8A8: which helps more?

Show answer

W8A8. At big batch sizes the GPU is limited by math, and W8A8 runs the math in 8 bits. W4A16 still does 16-bit math.

4Acceptance rate 0.8, 4 guesses per round, draft cost 10% of a big pass. Tokens per big pass, and speed-up?

Show answer

(1 − 0.8⁵) ÷ (1 − 0.8) = 0.672 ÷ 0.2 = 3.36 tokens per pass. Speed-up = 3.36 ÷ (1 + 4 × 0.1) = 3.36 ÷ 1.4 = 2.4 times.

5Does speculative decoding change what the model says?

Show answer

No. The method is designed so the output follows exactly the same probabilities as normal decoding. Only speed changes. Tiny floating-point differences are possible in practice.

## What to remember

1. Quantization rounds numbers to fewer bits: less to read per pass, more room for the KV cache.
2. FP8 W8A8 is essentially lossless; INT8 W8A8 loses 1% to 3%; INT4 weights are close to 8-bit.
3. Your A100 runs FP8 as weight-only. FP8 math needs H100-class GPUs.
4. Speculative decoding: a draft guesses, the big model checks many tokens in one pass, with identical outputs.
5. It helps most at low load. Measure at your real load before keeping it.

**Next: Lesson 6, more than one GPU**Splitting a model across GPUs when it doesn't fit. And sharing one GPU with MIG or time-slicing, like your 0.3 share. [Open Lesson 6](https://claude.ai/artifact/NzSi3BVZim2ETkfvjtpwPf)

[Back to Lesson 4](https://claude.ai/artifact/4X7ppjDXD2CJ28WjeJq8MS)

Sources

- [Leviathan et al., Fast Inference from Transformers via Speculative Decoding (ICML 2023)](https://arxiv.org/abs/2211.17192): identical outputs, 2 to 3 times faster, tokens-per-pass and speed-up formulas
- [vLLM docs: speculative decoding](https://docs.vllm.ai/en/stable/features/speculative_decoding/): methods, low-to-medium load guidance, `--speculative-config`
- [Kurtic et al., "Give Me BF16 or Give Me Death"? (ACL 2025)](https://arxiv.org/abs/2411.02355): 500,000+ evaluations, FP8 lossless, INT8 1% to 3%, INT4 competitive, which format for which deployment
- [vLLM docs: FP8](https://docs.vllm.ai/en/v0.11.0/features/quantization/fp8.html): FP8 math needs compute capability 8.9+; Ampere runs FP8 as W8A16 with Marlin; on-the-fly FP8
- [vLLM pull request 5975](https://github.com/vllm-project/vllm/pull/5975): FP8 Marlin for Ampere GPUs