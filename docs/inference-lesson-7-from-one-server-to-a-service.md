Inference track, lesson 7 of 8[Back to your prep plan](https://claude.ai/artifact/8m8maPTnDHsbPQWnqjz1bA)

# From one server to a service

Real traffic needs several copies of the model. It needs a front door that picks the right copy, and something that adds copies as traffic grows. This lesson covers all three on Kubernetes.

Try it: send 1,200 chat requests to several copies of a model

How requests are sent

Copies

Different system prompts in use

found in cache (hit)not found, prefill again (miss)

**36%**hit rate, round-robin

**97%**hit rate, cache-aware

**32 vs 8**prompts each copy must hold

Each copy's prefix cache fits 12 long prompts (Lesson 4). Each request uses one prompt, picked at random. Cache-aware routing here always sends a prompt to the same copy.

## Words used in this lesson

Replica

One complete copy of the model, in its own pod.

Round-robin

Sending requests to copies in turn: 1, 2, 3, 1, 2, 3.

Endpoint picker (EPP)

A small service that chooses which copy gets each request.

Autoscaler

A controller that adds or removes copies based on a signal.

HPA, KEDA

Kubernetes' built-in autoscaler, and an add-on that feeds it outside signals such as Prometheus metrics.

Cold start

The time a new copy needs before it can answer.

Scale to zero

Removing every copy when there is no traffic.

P/D disaggregation

Running prefill and decode on different GPUs.

## In this lesson

1. [Three layers](#layers)
2. [KServe: deploying the model](#kserve)
3. [Routing that knows the cache](#route)
4. [What to scale on](#scale)
5. [Try it: the autoscaler](#sim)
6. [Cold starts](#cold)
7. [Splitting prefill and decode](#pd)
8. [Your attacker model](#yours)
9. [Check yourself](#quiz)

1

## Three layers

One vLLM server is an engine. A service needs more around it. On Kubernetes, the usual open-source stack has three layers:

1. **vLLM** runs the model in each pod. Everything from Lessons 1 to 6 happens here.
2. **KServe** deploys the model. You describe it in one YAML file, and KServe creates the pods, service and autoscaler.
3. **llm-d** makes the copies work as a team. It adds cache-aware routing, splits prefill and decode, and scales on inference signals.

The vLLM docs list both ways in: KServe with its Hugging Face runtime, or KServe's `LLMInferenceService`, which uses llm-d.

Key ideavLLM is the engine, KServe deploys it, llm-d coordinates the copies.

2

## KServe: deploying the model

KServe has two resources for this. `InferenceService` is the original, built for classic ML models and simple LLM use. `LLMInferenceService` is built for LLMs. It adds llm-d routing, split prefill and decode, and Gateway API support.

KServe can also run in two modes. The choice decides whether a model can scale to zero.

|  | Standard | Knative |
| --- | --- | --- |
| Creates | Normal Deployments, Services and HPAs | Knative services |
| Autoscaling | HPA, or KEDA for custom metrics | Knative, based on requests |
| Scale to zero | No (not from zero on HTTP) | Yes; a component holds requests while a pod starts |
| KServe's advice | Highly recommended for LLMs | For bursty traffic where cost matters most |

Red Hat OpenShift AI deprecated KServe's Serverless (Knative) mode in version 2.25. Version 3 requires moving to RawDeployment, its name for Standard mode.

Key ideaStandard mode is the normal choice for LLMs. It gives up scale to zero.

3

## Routing that knows the cache

A normal load balancer treats requests as small, equal and stateless. LLM requests are none of these. Some run for a minute. And a copy that already holds a request's prefix in its cache can skip most of the prefill (Lesson 4).

Round-robin ignores that. In the demo at the top, try 4 copies and 32 prompts. Every copy sees all 32 prompts, but can only keep 12. So most requests miss and pay for prefill again.

How a request finds its copy

**Gateway**Receives the request, as with any web service

↓ asks

**Endpoint picker (EPP)**1. Filter: drop unhealthy pods and pods with the wrong model. 2. Score: who has this prefix cached? Whose queue is short? 3. Pick the top score.

↓ sends to

pod A

pod B: has the prefix

pod C

This is the Kubernetes Gateway API Inference Extension. An `InferencePool` lists the model pods, and the EPP picks among them. llm-d supplies the EPP.

Sending every prompt to one copy has a risk: a popular prompt can overload it. So llm-d's picker also scores queue length. It sometimes seeds a prompt onto other copies, and it gives up the cache match when the warm copy is much slower.

How much can it matter? In an llm-d test built to show the effect, 8 copies of Qwen3-32B served prompts that share prefixes. Time to first token fell from 35 seconds to 120 milliseconds. Throughput rose 151%.

Key ideaWith many copies and many repeated prompts, route by cache. With few prompts, round-robin is fine.

4

## What to scale on

Out of the box, Kubernetes' autoscaler (HPA) scales on CPU or memory. Neither tracks LLM load, because the work is on the GPU. GPU utilization looks like the fix, but it isn't.

GPU utilization, as `nvidia-smi` reports it, is the share of time *any* kernel is running. vLLM runs kernels non-stop while it decodes even one request. So the number sits near 100% whether there is 1 request or 200.

vLLM exports better signals on its `/metrics` page:

- `vllm:num_requests_waiting`: requests queued, not yet started.
- `vllm:num_requests_running`: requests in the current batch.
- `vllm:kv_cache_usage_perc`: how full the KV cache is.

KEDA reads one of these from Prometheus. By default it sets copies = the total ÷ your target per copy, rounded up. The HPA then adds copies at once, but waits 5 minutes before removing any.

Key ideaScale on what the server is going through: its queue, its batch, its KV cache. Not GPU utilization.

5

## Try it: the autoscaler

30 minutes of traffic: quiet, a rise to 10 requests per second, then quiet again

Scale on

Time for a new copy to be ready

Copies

readystarting (holds a GPU, can't answer)copies the traffic needs

Requests waiting in the queue

Touch or hover the charts to read any minute.

**113**GPU-minutes used

**116**most requests waiting

**29 s**worst wait, roughly

Toy model: each copy runs up to 32 requests at once, and each request takes 8 seconds. So one copy serves 4 requests per second. The HPA checks every 15 seconds, with up to 8 copies. Targets: 80% GPU utilization, 5 waiting per copy, or 24 in flight per copy.

Three things to see:

- **GPU utilization** reads 100% all the time, so it climbs to 8 copies and stays there. Nobody waits, but it uses about twice the GPUs.
- **Waiting requests** works on the way up. But once the copies catch up, the queue is empty, so it removes them. The queue returns and it scales up again: it flaps.
- **In flight** (running + waiting) follows the traffic. It still overshoots on the way up, because new copies take time to start.

Now switch to 4 minutes. Every signal suffers, because no autoscaler can fix a slow start. That is the next step.

Production tools go further. llm-d's workload variant autoscaler watches KV cache use and queue length. It adds copies when spare room runs low, and removes one only if the rest can still cope.

Key ideaA queue tells you that you are already short. Pair it with load in flight or KV cache use.

6

## Cold starts

"Pod is running" is not "model is ready". A new vLLM copy has to do all of this first:

1. **Pull the image.** The vLLM image can be 28 GB. In one test on Google Cloud, the pull alone took about 4 minutes. Nodes that already hold the image skip this.
2. **Get the weights.** 16 GB for an 8B model, from Hugging Face, S3 or a volume.
3. **Load them into GPU memory.** In one Azure test, Llama 3.1 8B loaded in 15.5 s with vLLM's default loader. It took 3.6 s with the Run:ai Model Streamer.
4. **Warm up.** vLLM compiles the model and records CUDA graphs before it reports ready.

Ways to shorten it: keep the image cached on GPU nodes, and keep weights close, on a volume or the node's disk. Stream them in with `--load-format runai_streamer`, which can read straight from S3. And keep at least one copy warm if users can't wait.

Key ideaCold start time decides how fast you can scale. Measure it, then shorten the slowest step.

7

## Splitting prefill and decode

Lesson 4 showed long prompts slowing down everyone else's tokens. Chunked prefill softens this. P/D disaggregation removes it: prompts are processed on some GPUs, and tokens are generated on others.

**Prefill pods**Read the whole prompt. Compute-heavy.

→

**Decode pods**Write the reply, one token at a time. Memory-heavy.

The KV cache moves across, through vLLM's NIXL connector

The KV cache can be big. Llama 3.1 70B uses 320 KiB per token (Lesson 2), so a 10,000-token prompt is about 3.3 GB. At 400 Gb/s that adds about 65 ms to the first token.

A guide on the vLLM blog names the main use: long prompts at high load push the slowest token times (p99 ITL) past target. It needs a fast link: RDMA between servers, or NVLink inside one.

It can also hurt. The transfer adds to every first token. The guide's slow test used two PCIe GPUs without a direct link. Median first-token time was 2.2 s split, versus 0.7 s together. For small models, short prompts or light traffic, keep prefill and decode together.

Key ideaSplit prefill and decode for big models, long prompts and heavy traffic, on fast links. Otherwise, don't.

8

## Your attacker model

One copy of an 8B model on a 0.3 share. What from this lesson applies?

- **Routing:** not yet. With one copy, every request goes to the same cache.
- **Autoscaling:** red-team runs come in bursts, and a queue during a run is fine. If you add copies, scale on requests in flight.
- **Scale to zero:** KServe's Standard mode, the only one left in OpenShift AI 3, can't wake a model from zero on an HTTP request. Keep one copy, or start it before a run.
- **Cold start:** worth measuring for your hardened image. If weights come from S3, try the Run:ai streamer.
- **P/D split:** no. The model is small and the traffic is light.

Key ideaMost of this matters once you run many copies. Learn it now; use it when traffic grows.

## Check yourself

Answer each one first, out loud or in chat, then open the answer.

14 copies, 50 different long system prompts, and each cache fits 12. Roughly what hit rate does round-robin get? And cache-aware routing?

Show answer

Round-robin: each copy sees all 50 but keeps 12, so about 12 ÷ 50 = 24%. Cache-aware: each copy gets about 12 or 13 prompts. Most fit, so the hit rate is high, roughly 90% or more.

2Why can't GPU utilization tell 1 request from 200?

Show answer

It measures the share of time any kernel is running. vLLM keeps kernels running while it decodes even one request, so it reads near 100% either way.

3You scale on `vllm:num_requests_waiting` alone. Under steady traffic, copies go up and down every few minutes. Why?

Show answer

Once the copies keep up, nothing waits, so the signal says "fewer copies". After the 5-minute wait, the HPA removes them, the queue returns, and it scales up again. Scale on requests in flight or KV cache use instead, or add them as a second signal.

4A new copy takes 4 minutes to start, and traffic can double in 1 minute. What do you change?

Show answer

Shorten the start: cache the image on GPU nodes, keep weights close, stream them in. And keep spare room: more minimum copies, or a lower target so scaling starts earlier.

5A 70B model, 10,000-token prompts, and a plain 100 Gb/s network without RDMA. Is splitting prefill and decode across servers a good idea?

Show answer

Probably not. Each prompt's KV cache is about 3.3 GB. At 100 Gb/s (12.5 GB/s) that is about 260 ms added to every first token, before any overhead. The vLLM guide asks for RDMA or NVLink.

## What to remember

1. vLLM runs the model, KServe deploys it, llm-d coordinates many copies.
2. KServe's Standard mode is the usual choice for LLMs, but can't scale to zero.
3. Round-robin wastes prefix caches when there are many prompts; cache-aware routing keeps each prompt on one copy.
4. Scale on requests in flight or KV cache use, not GPU utilization. A queue alone flaps.
5. Cold start limits how fast you can scale. Cache the image, keep weights close, stream them.
6. Split prefill and decode only for big models, long prompts and heavy load, on fast links.

**Next: Lesson 8, benchmarking and cost**How to measure a deployment and read latency percentiles. How much load it takes before you miss targets, and the cost per million tokens. [Open Lesson 8](https://claude.ai/artifact/Bk4dodCHpZEYQH2F6Ld12S)

[Back to Lesson 6](https://claude.ai/artifact/NzSi3BVZim2ETkfvjtpwPf)

Sources

- [vLLM docs: llm-d](https://docs.vllm.ai/en/stable/deployment/integrations/llm-d/) and [KServe](https://docs.vllm.ai/deployment/integrations/kserve.html): what llm-d adds, the two ways to use KServe
- [Red Hat: resources behind LLMInferenceService](https://developers.redhat.com/articles/2026/08/05/trace-kubernetes-resources-llm-d-model-serving): InferenceService versus LLMInferenceService
- [KServe architecture](https://kserve.github.io/website/docs/concepts/architecture), [Standard mode install](https://kserve.github.io/website/docs/admin-guide/kubernetes-deployment) and [control plane](https://kserve.github.io/website/docs/concepts/architecture/control-plane): the two modes, scale to zero, KEDA
- [OpenShift AI release notes](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/html/release_notes/support-removals_relnotes) and [migration article](https://access.redhat.com/articles/7134025): Serverless mode deprecated, move to RawDeployment
- [Red Hat: intelligent inference scheduling with llm-d](https://developers.redhat.com/articles/2026/06/04/intelligent-inference-scheduling-llm-d-red-hat-ai): filter, score, pick; hotspot safeguards; the 35 s to 120 ms test
- [Gateway API Inference Extension](https://gateway-api-inference-extension.sigs.k8s.io/): InferencePool and the endpoint picker
- [What GPU-Util measures](https://itnext.io/100-gpu-utilization-39-of-rated-fp16-ce97ba1b88f0), from NVIDIA's definition
- [KEDA ScaledObject reference](https://keda.sh/docs/2.20/reference/scaledobject-spec) and [HPA API](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/autoscale_apis/horizontalpodautoscaler-autoscaling-v2): polling, average-value targets, 5-minute scale-down window
- [AWS EKS: KEDA on vLLM queue depth](https://docs.aws.amazon.com/eks/latest/userguide/ml-inference-autoscaling-hpa-keda.md)
- [OpenShift AI: autoscaling llm-d deployments](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.5/html/deploy_models_using_distributed_inference_with_llm-d/autoscaling-llmd-model-deployments): the workload variant autoscaler (Technology Preview)
- [vLLM startup on GKE](https://dudeperf3ct.github.io/posts/vllm_startup_load_time): 28 GB image, about 236 s pull
- [Azure: Run:ai Model Streamer](https://devblogs.microsoft.com/azure-sdk/eliminate-llm-cold-starts-load-models-up-to-6x-faster-with-azure-blob-storage-and-runai-model-streamer/): 8B load 15.5 s versus 3.6 s; [AWS EKS](https://docs.aws.amazon.com/eks/latest/userguide/ml-inference-fast-model-loading.html): streaming from S3
- [vLLM blog: a practical guide to disaggregated serving](https://vllm.ai/blog/2026-09-29-disaggregated-serving-guide): when it helps, fast links, 3 GB and 65 ms, the slow PCIe test