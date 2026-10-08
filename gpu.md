
## 1. The two phases of LLM inference

| Phase | What happens | Bottleneck | Metric it controls | Hardware spec that matters |
|---|---|---|---|---|
| **Prefill** | Model reads the whole prompt at once | Compute (math) | **TTFT**: Time To First Token | FLOPS |
| **Decode** | Model generates one token at a time, reading all weights for each token | Memory bandwidth | **TPOT / ITL**: Time Per Output Token | Memory bandwidth (TB/s) |

> Most chat and agent workloads are decode-heavy, so **bandwidth usually matters more than FLOPS for inference.**

---

## 2. Core terminology

| Term | Meaning | Why it matters to you |
|---|---|---|
| **VRAM** | Memory on the GPU itself | Model weights, KV cache, and activations must all fit here |
| **HBM** (HBM3, HBM3e, HBM4) | Stacked high-bandwidth memory next to the GPU die | Very fast, very expensive; found on data center GPUs |
| **GDDR** (GDDR6, GDDR7) | Conventional graphics memory | Cheaper, slower; found on workstation cards (L40S, RTX PRO) |
| **Memory bandwidth** | Speed of moving data from VRAM to compute (TB/s) | Sets the speed limit for token generation |
| **FLOPS / TFLOPS / PFLOPS** | Math operations per second (10¹² / 10¹⁵) | Matters for prefill, training, and big batches |
| **Dense vs Sparse FLOPS** | Sparse ≈ 2× dense, using a trick few workloads use | **Always compare dense numbers** |
| **Tensor cores / Matrix cores** | AI math units (NVIDIA / AMD) | AI FLOPS figures refer to these |
| **Quantization** | Converting a model to fewer bits | Cuts memory, raises speed, may cost some quality |
| **KV cache** | Cached attention data for every token in context | Grows with **users × context length**; often the real VRAM consumer |
| **Batch size / Concurrency** | Requests processed together | More batch = more total throughput, slightly slower per user |
| **Throughput** | Total tokens/sec across all users | Capacity metric |
| **Latency** | Speed experienced by one user | User-experience metric |
| **TDP** | Power drawn per GPU (watts) | Drives power and cooling requirements |
| **MIG** | Multi-Instance GPU: slice one GPU into isolated parts | Good for small models or multi-tenant setups |

---

## 3. Precision / data types

| Format | Bytes per parameter | 70B model size | Quality | Typical use |
|---|---|---|---|---|
| FP32 | 4 | 280 GB | Reference | Rarely used for LLMs now |
| BF16 / FP16 | 2 | 140 GB | Full | Training, high-quality inference |
| FP8 | 1 | 70 GB | Near-lossless | Production standard (Hopper and newer) |
| INT4 / FP4 (NVFP4, MXFP4) | 0.5 | 35 GB | Usually good; test on your tasks | Aggressive inference (native on Blackwell, MI355X) |

> When a vendor quotes FLOPS, ask **"at which precision?"** FP4 ≈ 2× FP8 ≈ 4× BF16.

---

## 4. GPU form factors

| Form factor | How it connects | GPU-to-GPU speed | Cost | Best for |
|---|---|---|---|---|
| **PCIe card** | Standard server slot | Slow (PCIe) | Lower | One model per GPU, many copies (data parallel) |
| **SXM** (NVIDIA HGX board) | 8 GPUs on a baseboard, NVLink | Very fast | Higher | Models split across GPUs, training |
| **OAM** (AMD) | 8 GPUs on a baseboard, Infinity Fabric | Very fast | Higher | Same as SXM |
| **Rack-scale** (e.g. NVL72) | 72 GPUs as one NVLink domain | Fastest | Highest; liquid-cooled rack | Very large models, frontier-scale work |

---

## 5. Multi-GPU parallelism

| Type | How it splits work | Network demand | Typical placement |
|---|---|---|---|
| **Tensor Parallel (TP)** | Each layer split across GPUs | Very high | Inside one server (NVLink) |
| **Pipeline Parallel (PP)** | Different layers on different GPUs | Medium | Across servers is OK |
| **Data Parallel (DP)** | Full model copies serving different requests | Very low | Anywhere; easiest way to scale inference |
| **Expert Parallel (EP)** | MoE experts on different GPUs | High | Inside a server or rack |

---

## 6. Interconnect and networking

| Term | Layer | What it is | Typical speed |
|---|---|---|---|
| **NVLink / NVSwitch** | Scale-up (inside server/rack) | NVIDIA GPU-to-GPU link | ~0.9–3.6 TB/s per GPU, by generation |
| **Infinity Fabric** | Scale-up | AMD GPU-to-GPU link | ~0.5–1 TB/s per GPU |
| **PCIe Gen5 / Gen6** | Host connection | GPU to CPU / NIC | ~64 / 128 GB/s (x16) |
| **InfiniBand** | Scale-out (between servers) | Low-latency AI network fabric | 400–800 Gb/s per port |
| **RoCE / Spectrum-X / Ultra Ethernet** | Scale-out | Ethernet-based AI networking | 400–800 Gb/s per port |
| **RDMA / GPUDirect** | Scale-out | GPUs read remote memory without the CPU | Reduces latency and CPU load |

> Note the units: NVLink is quoted in **GB/s (bytes)**, network in **Gb/s (bits)**. 800 Gb/s ≈ 100 GB/s.

---

## 7. Current GPU landscape (late 2026)

| GPU | Vendor | Generation | VRAM | Bandwidth | Form factor | Positioning |
|---|---|---|---|---|---|---|
| L40S | NVIDIA | Ada | 48 GB GDDR6 | ~0.86 TB/s | PCIe | Small models, embeddings, vision |
| RTX PRO 6000 Blackwell | NVIDIA | Blackwell | 96 GB GDDR7 | ~1.8 TB/s | PCIe | Mid-size models, air-cooled, strong value |
| H100 | NVIDIA | Hopper | 80 GB HBM3 | 3.35 TB/s | SXM / PCIe | Workhorse, prices falling |
| H200 | NVIDIA | Hopper | 141 GB HBM3e | 4.8 TB/s | SXM | Excellent inference value |
| B200 | NVIDIA | Blackwell | 192 GB HBM3e | ~8 TB/s | SXM | Native FP4 |
| B300 | NVIDIA | Blackwell Ultra | 288 GB HBM3e | 8 TB/s | SXM | Current flagship shipping in volume |
| Rubin (VR200) | NVIDIA | Vera Rubin | 288 GB HBM4 | 22 TB/s | SXM / rack | Next generation, allocation-limited |
| MI300X | AMD | CDNA 3 | 192 GB HBM3 | 5.3 TB/s | OAM | Previous gen, good memory/$ |
| MI325X | AMD | CDNA 3 | 256 GB HBM3e | 6 TB/s | OAM | Memory refresh |
| MI355X | AMD | CDNA 4 | 288 GB HBM3e | 8 TB/s | OAM | Competes with B200/B300, lower price |

Supporting details: the B300 draws 1,400W with 15 PFLOPS dense FP4, and DGX B300 systems ship with 8–12 week lead times. Rubin partners start in H2 2026, but allocation queues are expected to push real deliveries into 2027. One price reference puts a single B300 at about $53,000 and an 8-GPU DGX B300 at $300,000–$500,000.

---

## 8. NVIDIA vs AMD

| Factor | NVIDIA | AMD |
|---|---|---|
| Software stack | CUDA: industry default | ROCm: much improved, more edge cases |
| Framework support (vLLM, SGLang) | Everything works first | Well supported now |
| Memory per dollar | Lower | Often higher |
| Availability | High demand, lead times | Often easier to get |
| Risk for a new team | Lower | Moderate |

---

## 9. Sizing formulas

| What | Formula | Example |
|---|---|---|
| **Weights memory** | parameters × bytes per parameter | 70B × 1 byte (FP8) = 70 GB |
| **KV cache per token** | 2 × layers × kv_heads × head_dim × bytes | Llama-3-70B: 2×80×8×128×2 ≈ 0.33 MB |
| **KV cache total** | per-token × context × concurrent users | 0.33 MB × 8K × 40 users ≈ 105 GB |
| **Total VRAM** | weights + KV cache + 10–20% overhead | 70 + 105 + 20 ≈ 195 GB |
| **Max single-user speed** | bandwidth ÷ model bytes | H100: 3,350 ÷ 70 ≈ 48 tokens/s ceiling |
| **Training memory** | roughly 4–8× inference memory per parameter | Gradients + optimizer states |

---

## 10. Quick sizing reference (inference, FP8 weights, moderate concurrency)

| Model size | Weights (FP8) | Example GPU options |
|---|---|---|
| 7–8B | ~8 GB | 1× L40S, 1× RTX PRO 6000 |
| 30–32B | ~32 GB | 1× RTX PRO 6000, 1× H100 |
| 70B | ~70 GB | 1× H200, 1× B200/B300, 2× RTX PRO 6000 |
| 120B (MoE) | ~120 GB | 1× B200/B300, 2× H100/H200 |
| 400B+ / large MoE (DeepSeek-class) | 400–700 GB | 8× H200, 4–8× B200/B300, 8× MI355X |

> Add KV cache on top. High concurrency or long context (64K+) can double the requirement.

---

## 11. Workload definition checklist (fill in before talking to vendors)

| Question | Example answer |
|---|---|
| Which models? | Llama 70B, Qwen 32B, embedding model |
| Precision? | FP8 |
| Peak concurrent users? | 50 |
| Typical / max context length? | 8K / 32K |
| Target TTFT? | < 1 second |
| Target tokens/sec per user? | ≥ 30 |
| Inference only, or fine-tuning too? | Inference + occasional LoRA fine-tuning |
| Redundancy needed? | N+1 |
| Growth expectation (12–24 months)? | 2× users |

---

## 12. Data center considerations

| Area | What to know | Question for facilities |
|---|---|---|
| **Power** | 8× B300 server ≈ 14 kW; enterprise racks often 5–15 kW total | Max kW per rack? |
| **Cooling** | B200/B300/MI355X-class often requires direct-to-chip liquid cooling | Do we support liquid cooling (CDUs, coolant loops)? |
| **Storage** | Fast NVMe for loading models; parallel FS (Weka, VAST, DDN) for training | Capacity and throughput needs? |
| **Host server** | System RAM ≥ total VRAM; enough CPU cores | Spec of host CPU and RAM? |
| **Networking** | 400–800 Gb/s per GPU for training clusters | Switch capacity and cabling? |

---

## 13. Software stack

| Layer | Options | Purpose |
|---|---|---|
| Serving engine | vLLM, SGLang, TensorRT-LLM, NVIDIA NIM | Runs the model efficiently, handles batching |
| Orchestration | Kubernetes + GPU Operator, Slurm | Schedules workloads on GPUs |
| Monitoring | NVIDIA DCGM, Prometheus, Grafana | GPU health, utilization, temperature |
| Licensing | NVIDIA AI Enterprise (per GPU, yearly) | Enterprise support for NVIDIA software |

---

## 14. Questions to ask infrastructure vendors

| # | Question | What a good answer looks like |
|---|---|---|
| 1 | Which GPU, form factor, and count per node? | Specific model + SXM/PCIe + count |
| 2 | Dense FLOPS at which precision? | Dense, precision stated (not sparse FP4) |
| 3 | VRAM and bandwidth per GPU? | Exact GB and TB/s |
| 4 | Intra-node and inter-node interconnect? | NVLink gen + InfiniBand/RoCE speed per GPU |
| 5 | Power per node and rack, cooling type? | kW figures + air/liquid + facility requirements |
| 6 | Delivery lead time? | Committed date, not "subject to allocation" |
| 7 | Can we run a PoC with our model? | Yes, with measured TTFT and tokens/sec at our concurrency |
| 8 | What software and licenses are included? | Drivers, K8s integration, NVAIE years |
| 9 | Warranty and failed-GPU replacement time? | Clear SLA, local on-site support |
| 10 | Upgrade path to next generation? | Chassis/rack compatibility stated |

---

If you share your actual models, users, and context lengths, I can fill in the sizing tables for your case.

Sources:
- [NVIDIA GPU Roadmap 2026-2030 (VRLA Tech)](https://vrlatech.com/nvidia-gpu-roadmap-2026-2030/)
- [Blackwell vs Rubin: Should I Wait? (VRLA Tech)](https://vrlatech.com/blackwell-vs-rubin-should-i-wait/)
- [Rent NVIDIA B300 GPU: 2026 Guide (Cyfuture)](https://cyfuture.cloud/blog/?p=75373)
- [AMD Instinct MI355X (SemiAnalysis InferenceX)](https://inferencex.semianalysis.com/chips/mi355x)
