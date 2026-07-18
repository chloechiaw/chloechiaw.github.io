---
layout: post
title: ""
date: 2026-07-17
categories: jekyll update
permalink: /hardware-lottery/
---

There's been a lot of discourse lately about the lottery—which labs got the good chips, which didn't, who's rationing H100s. And against that backdrop it's a little surprising that some of the most hardware-constrianed shops verseas still make really great open models that hold their own, at least in a benchmar setting. 

I wanted to guess that there weren't that many disticnt problems that we throw kernels at. If we take a look at recent ones:

Deepseek 
- Flash MLA (Attention Variant)
- Deep EP (Comms)

Qwen 
- Chunked GDN (Attention Variant)

Kimi 
- Delta Attention (Attention Variant) 
- latentMoE and Quantile Balancing (Mixture of Experts Routers)


What if you zoom out and label the traits of those operations rather than the operations themselves?  If the whole zoo really does reduce to a handful of shapes, then the map isn't mostly explored — it's mostly empty. For example, attention and a plain matrix multiply. They are similar because every output is a blend of every input, so you walk through the inputs a tile at a time, keep a running total, and never hold the whole thing at once. Attention just slips a softmax into the middle of that walk. 

Other kernel libraries like tokamax from XLA and Quack from Tri Dao also have kernels that cover these three categories. The last column is Mixture of Experts, and the architecture ha spresented an interesting new space forming its own category, mostly due to load balancing. 

# Categorizing the "shapes" of operations

| | Independent | Reduction | Data-dependent |
| :--- | :--- | :--- | :--- |
| **Dependency** | Each output needs one input | Each output needs a bit of every input, combined by an operator where grouping doesn't matter | The work itself depends on the data, unknown until runtime |
| **Examples** | Elementwise, RMSNorm | GEMM, attention | MoE routing, sparse |
| **Data reuse** | None — each byte touched once | High — grows with tile size | Unpredictable |
| **Bound by** | Memory bandwidth | Compute | (the irregularity) |
| **The move** | Fuse into a neighbor, never a standalone kernel | Tile + online softmax | Isolate the irregularity, so more of the work stays regular dense work |

We can visualize these primitives like this: 

![Independent](/Users/chloe/Desktop/chloechiaw.github.io/assets/images/op_independent.png)
![Reduction](/Users/chloe/Desktop/chloechiaw.github.io/assets/images/op_allpairs.png)
![Data-Dependent](/Users/chloe/Desktop/chloechiaw.github.io/assets/images/op_datadependent.png)

Uncharted territory aka possibly new shapes: 

1. Running state (linear attention variants and state-space family)
- post-Mamba-2-influence
- Qwen's Gated DeltaNet layers 
- Recently, Kimi Linear and Kimi Delta Attention in Kimi K3.


Even though there are many varints of linea attention, I'm labelling it under the same primitive. Basic moves are fixed state that's updated per token, with a correction or gate to fight forgetting. This all lives in the same kernel. 

2. Mixture of Experts 
- I think this space will be really intersting! Ccause right now most advancements are like Deepseek Auxiliary loss or quantile balancing for healthy routing.
- ktransformers is a project that experiments with saving memory by just loading more stuff on the cpu (for example, your experts).
- DeepEP, all-to-all done really well for MoE dispatch/combine. 


3. Megakernels

![alt text](megakernel.png)

The core shapes we throw kernels at really do seem to collapse into a surprisingly small set. I'm excited to see if there are new patterns that emerge! 
