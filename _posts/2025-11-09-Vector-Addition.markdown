---
layout: post
title: "Plain Vector Addition on a B200"
date: 2025-09-12 00:38:17 -0800
categories: jekyll update
permalink: /vec-add/
---

Over the weekend, I was trying to see if I could use CUDA to speed up vector addition. This is the result: 

#4/465 and one person yassa9 is absolutely mogging everyone with Triton. 
Substantial GFLOPs difference compared to everyone else (2x more GFLOPs reached than my 445.366/)

## Vector addition results across all GPUs
I was working on optimizing a memory-bound kernel for vector addition on NVIDIA H100 and B200 GPUs. The task was just to add corresponding elements from two input arrays and write the results to an output array, so very simple! At first I just played around with low hanging fruit such as block size (since the max is 1024 threads per block on an H100) to see what the performance benefits were but then tried to use more logical approaches such as coalesced memory access. I am still learning, so this is a novice worklog describing all the things tried! The language I chose was CUDA, but Tensara lets you submit kernels in Triton, CuTe DSL, etc.

## Initial Approach: Multiple Elements Per Thread
My first implementation had each thread handle 8 elements (4 from each input array):

```cpp
__global__ void vectorAdd(float *d_a, float *d_b, float *d_output, int n) {
    int idx = blockIdx.x * blockDim.x + threadIdx.x;
    int base = idx * 8;  // Each thread handles 8 elements

    if (base + 7 < n) {
        // Load 4 floats at once using float4
        float4 a1 = reinterpret_cast<float4*>(d_a)[base/4];
        float4 a2 = reinterpret_cast<float4*>(d_a)[base/4 + 1];

        float4 b1 = reinterpret_cast<float4*>(d_b)[base/4];
        float4 b2 = reinterpret_cast<float4*>(d_b)[base/4 + 1];

        // Compute sums
        float4 result1, result2;
        result1.x = a1.x + b1.x;
        result1.y = a1.y + b1.y;
        result1.z = a1.z + b1.z;
        result1.w = a1.w + b1.w;

        result2.x = a2.x + b2.x;
        result2.y = a2.y + b2.y;
        result2.z = a2.z + b2.z;
        result2.w = a2.w + b2.w;

        // Write back
        reinterpret_cast<float4*>(d_output)[base/4] = result1;
        reinterpret_cast<float4*>(d_output)[base/4 + 1] = result2;
    }
}
```

This approach used CUDA's float4 vector type to load 4 floats at once, maximizing the data loaded per instruction. Since CUDA vector types max out at 4 elements, loading 8 elements required two float4 loads per thread.

## The Memory Access Pattern Problem
Here's where things went wrong. With this approach, the memory access pattern looked like:

Noncoalesced Access Pattern:
Thread 0: indices 0-7
Thread 1: indices 8-15
Thread 2: indices 16-23
Thread 3: indices 24-31
Within a warp (32 threads executing together), threads were accessing memory in strides of 8 floats, not consecutively. This meant:
Thread 0 loads bytes 0-31 (8 floats)
Thread 1 loads bytes 32-63 (8 floats)
Thread 2 loads bytes 64-95 (8 floats)

This is a problem because threads in a warp access memory in strides of 8 floats, not consecutively. This causes multiple memory transactions instead of one coalesced transaction, wasting precious memory bandwidth.

## Attempt #1: Adjusting Block Size
My first optimization attempt was just to see how block size tuning impacted GFLOPs.
Testing different thread counts per block:
256 threads/block (baseline)
128 threads/block (allows more blocks per SM, often good for memory-bound operations)
512 threads/block (higher occupancy to hide memory latency)
1024 threads/block (maximum occupancy)
Results on H100:
512 and 1024 threads/block: Lower GFLOPS
128 and 256 threads/block: Similar performance
There wasn’t too much of a difference between changing the : poor memory coalescing.

Memory Coalescing
Coalesced Access (Optimal):
Thread 0: index 0
Thread 1: index 1
Thread 2: index 2
...
Thread 31: index 31

All 32 threads in a warp access consecutive memory addresses → 1 memory transaction
Noncoalesced Access (My Original Code):
Thread 0: indices 0-7
Thread 1: indices 8-15
Thread 2: indices 16-23


Threads access data in strides → Multiple memory transactions
Consecutive threads have to access consecutive memory locations

Attempt #2: Coalesced Memory Access (H100)
I rewrote the kernel to ensure coalesced access:

```cpp
__global__ void vectorAddCoalesced(float *d_a, float *d_b, float *d_output, int n) {
    int idx = blockIdx.x * blockDim.x + threadIdx.x;

    if (idx < n) {
        // Each thread handles ONE element
        // Thread i accesses index i → coalesced!
        d_output[idx] = d_a[idx] + d_b[idx];
    }
}
```

Result: Same GFLOPS and latency as the non-coalesced version with 4 elements per thread.
Wait, what? After all that work, performance was identical? This seemed counterintuitive until I realized: even with coalescing, each thread was only doing 1 operation, reducing computational intensity.
Turns out you can combine both strategies: coalesced memory access and having each thread process multiple elements:
**global** void vectorAddOptimized(float *d_a, float *d_b, float _d_output, int n) {
int tid = blockIdx.x _ blockDim.x + threadIdx.x;
int stride = blockDim.x \* gridDim.x;

    // Process 8 elements per thread, but with coalesced access
    for (int i = 0; i < 8; i++) {
        int idx = tid + i * stride;
        if (idx < n) {
            d_output[idx] = d_a[idx] + d_b[idx];
        }
    }

}

Instead of each thread accessing 8 consecutive elements (stride of 8), threads now access elements separated by stride (total number of threads).

This ensures:
First iteration: Thread 0 → index 0, Thread 1 → index 1, ... (coalesced)
Second iteration: Thread 0 → index 0+stride, Thread 1 → index 1+stride, ... (coalesced)

Each thread still processes 8 elements total (“computational intensity”, although it’s just vector addition so not thatttt intense)


| Implementation              | Memory Pattern | GFLOPS (B200)     | Notes                       |
| --------------------------- | -------------- | ----------------- | --------------------------- |
| Original (8 elem/thread)    | Noncoalesced   | Baseline          | Stride-8 access             |
| Single element/thread       | Coalesced      | ~Same as baseline | Low computational intensity |
| Optimized (8 elem, strided) | Coalesced      | Best              | Both coalescing + intensity |
