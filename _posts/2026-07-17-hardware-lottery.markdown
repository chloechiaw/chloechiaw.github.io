---
layout: post
title: "Unrigging the hardware lottery"
date: 2026-07-17
categories: jekyll update
permalink: /hardware-lottery/
---

There's been a lot of discourse lately about the lottery—which places got the good chips, which didn't, who's rationing H100s. And against that backdrop it's a little surprising that some of the most hardware-constrianed shops overseas still make really great open models that hold their own, at least in a benchmark setting!

On the flip side, let's set aside the hardware lottery discussion from the US China export controls/ supply chain/ talk. You can also argue there is a hardware lottery between you as the engineer and the GPU, as it is still really hard to get MFU up even though researchers have really been optimizing every part of the stack.

Realistic MFUs today are so challenging! A dense transformer on a well-tuned stack can reach maybe 40–50% of what the hardware could theoretically do, and Mixture of Experts specifically still lands frighteningly low. 

<img src="/assets/images/haha.png" alt="alt text" width="450">

That made me guess that maybe the territory in performance tricks is pretty unexplored. This might sound a bit strange, cause if we take a look at novel methods in the last year, they seem pretty diverse: 

**DeepSeek**
- [Flash MLA](https://github.com/deepseek-ai/FlashMLA) (attention variant)
- [DeepEP](https://github.com/deepseek-ai/DeepEP) (comms)

**Qwen**
- [Chunked GDN](https://arxiv.org/abs/2412.06464) (attention variant)

**Kimi**
- [Delta Attention](https://arxiv.org/abs/2510.26692) (attention variant)
- latentMoE and quantile balancing (Mixture-of-Experts routers)

But... what if we zoom out and label the traits of those operations rather than the operations themselves? For example, attention and a plain matrix multiply. They are similar because every output is a blend of every input, so you walk through the inputs a tile at a time, keep a running total, and never hold the whole thing at once. Attention just slips a softmax into the middle of that walk. 

While seeing all these recent buzz around new methods, I wonder if a lot of these tricks can collapse into a really small set of basic operations, or primitives. This might seem surprising, since within attention alone there are so many cool tricks — GQA and MQA sharing key/value heads, MLA compressing the KV cache into a latent, sliding-window and block-sparse attention skipping most of the matrix, PagedAttention rethinking how the cache is stored, ring attention splitting the sequence across GPUs. On paper they look like six different ideas. But the actual move each one makes on the GPU is nearly identical: load a tile of queries, stream the keys and values past it a block at a time, keep a running max and running sum (online softmax), accumulate into the output, and never once write the full N×N score matrix to memory. The tricks change *which* keys and values you bother to load, or how you store them, not the walk itself.

Categorizing the "shapes" of operations... 

# Primitives

| | Independent | Reduction | Data-dependent |
| :--- | :--- | :--- | :--- |
| **Dependency** | Each output needs one input | Each output needs a bit of every input, combined by an operator where grouping doesn't matter | The work itself depends on the data, unknown until runtime |
| **Examples** | Elementwise, RMSNorm | GEMM, attention | MoE routing, sparse |
| **Data reuse** | None — each byte touched once | High — grows with tile size | Unpredictable |
| **Bound by** | Memory bandwidth | Compute | (the irregularity) |
| **The move** | Fuse into a neighbor, never a standalone kernel | Tile + online softmax | Isolate the irregularity, so more of the work stays regular dense work |

We can visualize these primitives like this:

![alt text](/assets/images/all_three.png)

Other kernel libraries — [tokamax](https://github.com/openxla/tokamax) from OpenXLA and [Quack](https://github.com/Dao-AILab/quack) from Tri Dao — cover these same three categories too. The last column, data-dependent, is mostly Mixture of Experts, and that architecture has presented an interesting new space forming its own category, mostly because of load balancing.

# Uncharted territory (aka possibly new shapes)
**1. Running state** — linear-attention variants and the state-space family.

- Post-[Mamba-2](https://arxiv.org/abs/2405.21060) influence
- Qwen's [Gated DeltaNet](https://arxiv.org/abs/2412.06464) layers
- Recently, [Kimi Linear](https://arxiv.org/abs/2510.26692) and Kimi Delta Attention in Kimi K3

Even though there are many variants of linear attention, I'm labelling them all under the same primitive. The basic move is a fixed state that's updated per token, with a correction or gate to fight forgetting — and it all lives in the same kernel.

**2. Mixture of Experts.**

- I think this space will be really interesting! Right now most of the advancements are things like DeepSeek's [auxiliary-loss-free load balancing](https://arxiv.org/abs/2408.15664) or quantile balancing for healthy routing.
- [ktransformers](https://github.com/kvcache-ai/ktransformers) is a project that experiments with saving memory by loading more stuff on the CPU (your experts, for example).
- [DeepEP](https://github.com/deepseek-ai/DeepEP) — all-to-all done really well for MoE dispatch/combine.


**3. Megakernels.**

<img src="/assets/images/megakernel.png" alt="Megakernel" width="450"> 
from https://www.cs.cmu.edu/~zhihaoj2/15-779/slides/07-mega-kernel.pdf

The core shapes we throw kernels at might collapse into a surprisingly small set. I'm excited to see what new patterns emerge that can create new "movements" on the GPU. 

