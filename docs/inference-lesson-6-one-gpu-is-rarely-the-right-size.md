Inference track, lesson 6 of 8[Back to your prep plan](https://claude.ai/artifact/8m8maPTnDHsbPQWnqjz1bA)

# One GPU is rarely the right size

Some models are too big for one GPU, so you split them across several. Some are too small for one, so you share it. This lesson covers both.

Try it: split a model across A100 80GB GPUs

Model

Weights

GPUs per copy (`--tensor-parallel-size`)

GPU 01234567

weightsoverhead (3 GB assumed)KV cachenot claimed (10%)

**35.3 GB**weights on each GPU

**411,000**KV cache tokens, all GPUs

**58**tokens/s ceiling, one request

Same math as Lessons 1 and 2, per GPU at 0.9 utilization. Overhead is a little above Lesson 2's 2 GB, because GPUs that talk to each other need extra buffers. The speed ceiling adds up the GPUs' memory bandwidth. It ignores the time they spend talking, so real speed is lower.

## Words used in this lesson

Node

One server, usually with 4 or 8 GPUs inside.

NVLink

A very fast link between GPUs in the same server.

All-reduce

A step where GPUs combine their partial results so all of them get the total.

Tensor parallel

Every layer is split across GPUs.

Pipeline parallel

Groups of layers go on different GPUs, one after another.

Replica

One complete copy of the model serving requests.

MIG

Multi-Instance GPU: cutting one GPU into separate smaller GPUs in hardware.

Time-slicing

Letting several programs take turns on one GPU.

## In this lesson

1. [Three reasons to use more GPUs](#three)
2. [Tensor parallelism](#tp)
3. [Pipeline parallelism](#pp)
4. [Choosing a split](#choose)
5. [Sharing one GPU](#share)
6. [Try it: MIG slices](#mig)
7. [Which do you have?](#yours)
8. [Check yourself](#quiz)

1

## Three reasons to use more GPUs

- **The model doesn't fit.** Llama 3.1 70B in 16-bit needs 141 GB for weights alone. One A100 has 80 GB. You must split it.
- **It fits, but with little room left.** Splitting also splits the KV cache, so more requests fit at once.
- **You need more throughput.** Run more complete copies (replicas) and spread requests across them. Lesson 7 covers this.

The first two need you to split one model. There are two main ways: across the layers' width (tensor parallel) or along their depth (pipeline parallel).

Key ideaSplit a model when it doesn't fit or has no room for the KV cache. Add replicas when you need more throughput.

2

## Tensor parallelism

Tensor parallelism splits *every layer* across the GPUs. With 4 GPUs, each holds a quarter of every weight matrix and a quarter of the attention heads. All four work on every token at the same time.

One layer of Llama 3.1 70B, split across 4 GPUs

GPU 0GPU 1GPU 2GPU 3

**attention heads**1–16

**attention heads**17–32

**attention heads**33–48

**attention heads**49–64

Combine results (all-reduce)

**MLP**¼ of it

**MLP**¼ of it

**MLP**¼ of it

**MLP**¼ of it

Combine results (all-reduce)

The usual design, from Megatron-LM, needs two combine steps per layer. Llama 3.1 70B has 80 layers, so that is 160 combine steps on every pass.

All that combining needs a very fast link. Inside one A100 server, NVLink moves 600 GB/s. A normal PCIe Gen4 slot moves 64 GB/s, almost 10 times less. That is why tensor parallelism is used *inside* one server.

The payoff: weights, KV cache and memory bandwidth are all shared out. So tensor parallelism both fits bigger models and makes each token faster. Turn it on with `--tensor-parallel-size 4`.

Key ideaTensor parallel splits every layer. It is fast, but only works well over NVLink inside one server.

3

## Pipeline parallelism

Pipeline parallelism splits the model by depth. GPU 0 holds layers 1 to 40; GPU 1 holds layers 41 to 80. A token goes through GPU 0 first, then its result moves to GPU 1. Only one small hand-off per stage, so a slower link between servers is fine.

Two stages, two requests (A and B) in flight

t1t2t3t4t5t6 GPU 0\
layers 1–40ABABAB GPU 1\
layers 41–80ABABA

Each square is half a pass. While GPU 1 finishes A's token, GPU 0 starts on B. With only one request, each GPU would sit idle half the time (hatched).

The catch: one request's pass still goes through every stage in turn, so pipeline parallelism doesn't make a single request faster. It needs several requests in flight to keep every stage busy. vLLM's flag is `--pipeline-parallel-size`.

Key ideaPipeline parallel splits by layers. It needs little bandwidth, so it works across servers, but it needs many requests to stay busy.

4

## Choosing a split

The vLLM docs give a simple order to follow:

1. **Fits on one GPU?** Don't split. Use one GPU.
2. **Fits in one server?** Use tensor parallel across its GPUs, for example `--tensor-parallel-size 4`.
3. **Needs several servers?** Tensor parallel inside each server, pipeline parallel across servers. With 2 servers of 8 GPUs: tensor parallel 8, pipeline parallel 2.

Two exceptions from the same docs. If your GPUs have no NVLink, such as L40S cards, use pipeline parallel instead, even inside one server. And if the GPU count doesn't divide the model evenly, pipeline parallel can split layers unevenly.

After it starts, check the "GPU KV cache size" line in the log, as in Lesson 2. If it's too small for your traffic, add GPUs.

Key ideaOne GPU if it fits. Tensor parallel inside a server. Add pipeline parallel across servers.

5

## Sharing one GPU

Now the opposite problem: an 8B model on an 80 GB A100 leaves most of the card unused. Two common ways to share it:

|  | MIG | Time-slicing |
| --- | --- | --- |
| How | Cuts the GPU into up to 7 separate smaller GPUs in hardware | Programs take turns on the whole GPU |
| Memory | Each slice has its own memory | Shared; nothing stops one program from using it all |
| Faults | A crash stays inside its slice | No isolation |
| Compute | Each slice has its own fixed compute | An equal share of time, with no guarantee |
| GPUs | A100, A30, H100 and newer data-center GPUs | Most NVIDIA GPUs, old ones too |

This matters for vLLM. In a MIG slice, vLLM sees a smaller GPU, and `--gpu-memory-utilization` applies to that slice. With time-slicing, vLLM sees the whole card. Its utilization flag then becomes the only budget. It is a per-instance limit that doesn't coordinate with other programs on the GPU. So the flags of all vLLM servers on one card must add up to 1.0 or less. Users also report that starting them one after another works better than starting them together.

Key ideaMIG gives real walls inside the GPU. Time-slicing gives turns, with no walls, so memory budgets are on you.

6

## Try it: MIG slices on an A100 80GB

MIG slice

Llama 3.1 8B weights

A slice name gives its compute (out of 7 parts) and its memory. The math uses the memory in the name. Budget = 0.9 × slice memory, minus weights, minus 2 GB assumed overhead. Weights are parameters × bytes. Real quantized files are a little bigger, because a few layers usually stay 16-bit.

Key ideaPick the smallest slice that leaves enough KV cache room. Or shrink the weights (Lesson 5) to fit a smaller slice.

7

## Which do you have?

Lessons 2 to 5 assumed your attacker model's "0.3 A100 share" means vLLM gets about 24 GB. That is right if it means `--gpu-memory-utilization 0.3` on a shared 80 GB card. It is wrong if it is a MIG slice. Check from inside the pod:

```
nvidia-smi -L
```

If MIG is on, the output lists MIG devices with names like `MIG 2g.20gb`. If it only shows the full GPU, you are not on a MIG slice. You share the whole card, or have it to yourself.

In Kubernetes you can also look at what the pod asks for, but it is less certain. With the GPU Operator's "mixed" MIG setting, slices appear as resources such as `nvidia.com/mig-2g.20gb`. With the "single" setting, slices appear as plain `nvidia.com/gpu`, just like a whole or time-sliced GPU. Time-sliced GPUs may also be renamed `nvidia.com/gpu.shared`. So trust `nvidia-smi -L` first.

Key ideaKnow which kind of share you have. It decides what memory vLLM really gets and who can disturb it.

## Check yourself

Answer each one first, out loud or in chat, then open the answer.

1Llama 3.1 70B in 16-bit on A100 80GB GPUs, utilization 0.9, 3 GB overhead per GPU. What is the smallest tensor-parallel size that leaves room for the KV cache?

Show answer

2 GPUs: 70.6 GB of weights + 3 GB each = 73.6 GB, more than the 72 GB budget. 4 GPUs: 35.3 + 3 = 38.3 GB each, leaving about 34 GB per GPU for the KV cache. So 4. (3 is not allowed: the 64 attention heads must split evenly.)

2Why not run tensor parallel across two servers linked by a normal network?

Show answer

Tensor parallel combines results twice per layer, on every pass: 160 times per pass for a 70B model. That needs NVLink-class speed. Across servers, use pipeline parallel, which only hands off once per stage.

3You have 4 L40S GPUs without NVLink in one server. Tensor or pipeline parallel?

Show answer

Pipeline parallel. The vLLM docs recommend it when GPUs have no NVLink, for higher throughput and less communication.

4Two vLLM servers share a time-sliced A100 80GB, each with utilization 0.5. A teammate adds a third with 0.3. What happens?

Show answer

The budgets add up to 1.3, more than the card has. Time-slicing has no memory walls, so the third server will fail to start or cause out-of-memory errors. Keep the total at 1.0 or less, or use MIG.

5Can Llama 3.1 8B in 16-bit run on a MIG 2g.20gb slice?

Show answer

Not usefully. The budget is 0.9 × 20 = 18 GB. 16 GB of weights plus about 2 GB of overhead leaves nothing for the KV cache. Use FP8 weights, which leave about 7 to 8 GB for the cache, or a 3g.40gb slice.

## What to remember

1. Split a model when it doesn't fit or has no KV room; add replicas for throughput.
2. Tensor parallel splits every layer, needs NVLink, and makes each token faster.
3. Pipeline parallel splits by layers, works across servers, and needs many requests in flight.
4. vLLM's order: one GPU, then tensor parallel in a server, then add pipeline parallel across servers.
5. MIG gives hardware walls; time-slicing doesn't, so `--gpu-memory-utilization` totals must stay at 1.0 or less.

**Next: Lesson 7, serving on Kubernetes**KServe and llm-d, scaling on queue length instead of GPU use, cold starts, and splitting prefill and decode onto different GPUs. [Open Lesson 7](https://claude.ai/artifact/BiKCvaaG54E7RXHftZdQje)

[Back to Lesson 5](https://claude.ai/artifact/EVM9BQ9JuJfBjvBiwY1UHD)

Sources

- [vLLM docs: parallelism and scaling](https://docs.vllm.ai/en/stable/serving/parallelism_scaling/): when to use one GPU, tensor parallel, pipeline parallel; L40S and uneven-split exceptions; KV cache log line
- [Shoeybi et al., Megatron-LM](https://arxiv.org/abs/1909.08053): splitting attention and MLP, one combine step per block
- [NVIDIA: scaling with Megatron](https://developer.nvidia.com/blog/scaling-language-model-training-to-a-trillion-parameters-using-megatron): tensor parallel communication is slow across servers
- [NVIDIA A100 datasheet](https://nvidia.com/content/dam/en-zz/Solutions/Data-Center/a100/pdf/nvidia-a100-datasheet-us-nvidia-1758950-r4-web.pdf): NVLink 600 GB/s, PCIe Gen4 64 GB/s, up to 7 MIG instances
- [NVIDIA: A100 MIG-backed profiles](https://docs.nvidia.com/ai-enterprise/release-8/latest/infra-software/vgpu/reference/ampere-a100.html): 1g.10gb to 7g.80gb and how many of each fit
- [NVIDIA GPU Operator: time-slicing](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/22.9.1/gpu-sharing.html): no memory or fault isolation, no proportional compute
- [NVIDIA GPU Operator, later version](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.3/gpu-sharing.html): `nvidia.com/gpu.shared` naming
- [vLLM forum: utilization is per instance](https://discuss.vllm.ai/t/interpret-process-memory-metrics-available-from-metrics-endpoint/1260) and [starting instances one after another](https://discuss.vllm.ai/t/multiple-vllm-engine-deployment-problem-on-nvidias-sm-engines/1685)