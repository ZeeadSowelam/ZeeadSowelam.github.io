---
layout: page
title: GPU-Accelerated CNN Convolution
description: Implementing and optimizing the forward pass of a LeNet-5 convolution layer in CUDA (ECE 408), from a CPU baseline to a templated, register-tiled fused kernel that runs 10,000-image inference in about 19 ms.
img: assets/img/cuda_op6_memory_chart.png
importance: 3
category: work
related_publications: false
---

ECE 408 is the applied parallel programming course, and the final project is to implement and optimize the forward pass of a convolution layer in CUDA, then use it to run inference for a modified LeNet-5 on Fashion-MNIST. The input is a batch of 10,000 single-channel 86x86 images, and the optimized kernel runs the two convolution layers, C1 with `Map_out=4` and C3 with `Map_out=16`, both with a 7x7 filter. The convolution is done the way most GPU libraries do it, by unrolling each input patch into a column (im2col) and turning the layer into one matrix multiply. The project runs across three milestones: a CPU baseline, a basic GPU kernel, then unrolling plus kernel fusion, then a stack of optimizations on top of the fused kernel.

There is a hard number attached to it. At batch 10,000, the sum of both layers' op times has to come in at or under 60 ms for full credit, and any layer under 40 ms on its own qualifies for the class competition. That target is the whole point, and it is what the final kernel is built around. Most of this writeup is about that final kernel. The individual-optimization analysis shows up later as the evidence for why the kernel includes what it includes and skips what it skips.

## From a CPU loop to the target

The CPU version is the seven-nested-loop convolution straight out of the textbook, graded on correctness only. On batch 100 it takes roughly 1.5 and 4.1 seconds for the two layers, which is fine, because its only job is to be the thing the GPU results get checked against. The basic GPU kernel from lecture, one output element per thread reading from global memory, brings the sum of op times down to around 70 ms.

Unrolling alone made it worse. Materializing the im2col matrix in global memory and then running a separate tiled matmul over it pushed the unfused version to 124.83 ms at batch 10,000, slower than the naive direct convolution, because every byte of the unrolled matrix gets written out to DRAM and read back. Fusion is what recovers that. Folding the unroll, the matmul, and the output permutation into one kernel, so a patch is unrolled into shared memory and consumed in place without ever touching global memory, lands the fused baseline at 82.71 ms (51.26 ms for Conv1, 31.45 ms for Conv2). That fused kernel is the baseline that every Milestone 3 optimization is measured against.

## The final kernel

The submission is a templated, register-tiled fused convolution. It starts from joint register and shared-memory tiling, the optimization the course numbers op_6, which is thread coarsening pushed about as far as it goes. Instead of one thread computing one output, each thread computes a full column of `U_OUT` outputs in the M dimension and keeps all of them in registers. Each outer iteration loads a short strip of the unrolled K dimension, the mask strip going into shared memory and the input values into per-thread registers, and then every thread does `S_K * U_OUT` fused multiply-adds against that strip. The matmul stops looking like a dot product accumulated tile by tile and starts looking like an outer product, where a mask element loaded once gets reused across all 64 threads in the block and an input value loaded once feeds `U_OUT` separate accumulators.

```cpp
// per-thread accumulator for U_OUT outputs, held entirely in registers
float acc[U_OUT];
#pragma unroll
for (int u = 0; u < U_OUT; u++) acc[u] = 0.0f;

// S_K * U_OUT FMAs per thread per outer iteration, all from registers and shared memory.
// n_reg[s] is reused U_OUT times; Ms[s][u] broadcasts to every thread in the block.
#pragma unroll
for (int s = 0; s < S_K; s++) {
    const float n_val = n_reg[s];
    #pragma unroll
    for (int u = 0; u < U_OUT; u++) {
        acc[u] += Ms[s][u] * n_val;
    }
}
```
<div class="caption">
    The accumulation loop from m3-forward.cu. The whole point of the design is that acc[] stays in registers, which only happens if the compiler can prove nothing aliases it.
</div>

That reuse is visible in the profiler, and it is the reason the kernel is fast. Nsight Compute puts the L1/TEX hit rate at 84.94%, up from 74.49% on the plain fused path, because each loaded element feeds many more FMAs before it gets evicted. Memory throughput climbs to 35.78 GB/s while the memory pipes get less congested, since most requests are now being served out of cache instead of fought over at the DRAM boundary.

<div class="row justify-content-sm-center">
    <div class="col-sm-11 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/cuda_op6_memory_chart.png" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Nsight Compute memory chart for the register-tiled kernel. L1/TEX hit rate is 84.94%, and 442M instructions land on shared memory against the global path, which is the register-and-shared-memory reuse doing its job.
</div>

Register tiling by itself gets the sum of op times to 60.84 ms, just under the full-credit line. Getting from there to the competition tier is a set of refinements stacked on top, each one cheap on its own and most of them required for the tiling to hold up. `__restrict__` and `const` on the read-only pointers (op_1) are not optional here, because the per-thread `acc[U_OUT]` array only stays in registers if the compiler can prove no store aliases it; without the hint it spills to local memory and the win evaporates. `#pragma unroll` (op_2) flattens the inner `S_K * U_OUT` loop into 64 independent FMAs the scheduler can lay out freely. The tile and coarsening sizes come from a parameter sweep (op_3) rather than a guess.

The piece that matters most is templating the kernel on `U_OUT` and `K` and dispatching a different instantiation per layer.

```cpp
// templated on the per-layer output tile U_OUT and kernel size K_T, so
// KK = K_T*K_T (49 here) folds to a compile-time constant and each layer
// compiles to its own machine code.
template <int U_OUT, int K_T>
__global__ void matmul_conv_fused(const float * __restrict__ mask, const float * __restrict__ input,
                                  float * __restrict__ output,
                                  int Batch, int Map_out, int Channel, int Height, int Width);

// Conv1 has Map_out=4, so U=4 means zero wasted rows. Conv2 fills the tile at U=16.
if (Map_out <= 4) {
    matmul_conv_fused<4, 7><<<grid4, blockDim>>>(mask, input, output, Batch, Map_out, Channel, Height, Width);
} else {
    matmul_conv_fused<16, 7><<<grid16, blockDim>>>(mask, input, output, Batch, Map_out, Channel, Height, Width);
}
```
<div class="caption">
    Per-layer dispatch. K is always 7, so KK=49 is a constant the compiler folds, and Conv1 gets a build where the output tile is exactly its four channels.
</div>

In the register-tiled kernel with a fixed `U_OUT=16`, Conv1 was wasting 12 of every 16 output rows, because it only has four output channels and the other twelve were padded with zeros and multiplied anyway. Giving Conv1 its own template instantiation with `U=4`, exactly its `Map_out`, removes that waste entirely. Conv1's op time drops from 36.58 ms to 10.78 ms. Templating on `K=7` also lets the compiler fully unroll the filter loops, which it cannot do while K is a runtime value, and I hoisted the `b/h/w` index math and the div/mod work out of the inner loop, since they depend only on the column and were being recomputed every iteration for nothing.

| Stage | Conv1 | Conv2 | Sum |
| --- | --- | --- | --- |
| M2 fused baseline | 51.26 ms | 31.45 ms | 82.71 ms |
| op_3 (tile 16, coarsen 4) | 49.07 ms | 30.05 ms | 79.12 ms |
| op_6 (register tiling) | 36.58 ms | 24.26 ms | 60.84 ms |
| Final stacked | 10.78 ms | 8.41 ms | 19.19 ms |
| Competition mode | 8.94 ms | 8.80 ms | 17.74 ms |

The final kernel runs the two layers in 19.19 ms at batch 10,000, about 4.3 times faster than the fused baseline, and both layers sit well under the 40 ms competition line.

## What did not make the cut, and why

A few of the optimizations the project asks you to implement do not belong in a kernel for this workload, and the useful part was being able to say so in advance and then confirm it in the profiler.

Tensor cores were the clearest call. They are dedicated matmul hardware, and on Ampere a single warp issues one `mma.sync` that does a 16x16x16 multiply, which is an enormous throughput ceiling, so the textbook move is to reach for them. A roofline argument said not to bother. These convolutions have an arithmetic intensity around 0.25 FLOPs per byte, and the crossover on the A40, the point where a kernel stops being limited by memory bandwidth and starts being limited by compute, sits near 250 FLOPs per byte. The kernel is on the memory-bound side of that line by a factor of roughly a thousand, so faster matmul hardware cannot help something that spends almost all of its time moving bytes. I wrote the prediction down first: tensor cores would lose on Conv1 and maybe break even on Conv2.

<div class="row justify-content-sm-center">
    <div class="col-sm-10 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/cuda_tensorcore_optime_table.jpeg" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Tensor-core op times against the fused baseline. Both layers came out slower, and Conv2 was worse than Conv1, which I had not predicted.
</div>

Both layers came out slower, and Conv2 was worse than Conv1, which I had not seen coming. Nsight Compute explained the miss. Achieved occupancy was pinned at 33% by register pressure, the tensor pipe sat at about 3% utilization, and the ALU ran near 60% grinding through the integer index math for the implicit unroll. I had written a tensor-core kernel that barely touches the tensor cores. The direction of the prediction held, but the actual bottleneck was address arithmetic, one level more specific than the bare roofline number.

Constant memory (op_0) is the standard place to put read-only filter weights, and I expected a clean win. It helped Conv1, where four mask rows broadcast to every thread and the constant cache is built for exactly that pattern. It regressed Conv2, where 16 distinct addresses per warp stop being a broadcast and serialize through the constant cache one address at a time, roughly 16 times the latency of the path the baseline was already using. The same change helped one layer and hurt the other. cuBLAS (op_4) came out about twice as slow as the fused kernel, mostly because it forces the unfused path and the input-unroll write to global memory alone cost more than all of its matmul kernels combined. FP16 on the ordinary SIMT path (op_5) was a few percent slower, because halving the bytes per element only helps a kernel that is bandwidth-bound in the specific way FP16 fixes, and each load picked up a float-to-half conversion on the way in. Half precision only earns its place once it feeds the tensor cores, which is the one path where it pays off.

## Which numbers to trust

The streams optimization (req_0) is required, and it is forbidden in the final submission, because op time is meaningless for it. Splitting the batch across four CUDA streams to overlap the copies with the kernels makes the op-time table read around 0.003 ms. The kernels are not finishing in microseconds. The harness times only the `conv_forward_gpu` function, and under the streamed structure that function is empty, because all the real work has moved into the prolog where the streams get set up. The number that means anything is the layer time, and the nsys profile underneath it, which showed the device-to-host transfer at 71 ms was the most expensive single stage, more than the 57.6 ms matmul.

That is the habit the whole project built, which is reading a profiler instead of a single wall-clock number, and knowing which of its numbers to distrust. Nsight Compute's kernel replay can inflate the reported op times by hundreds of times, so I take real timings off the sbatch stdout rather than the profiler's own summary. The thread running under all of it is that an optimization is not good or bad on its own, it is good or bad for a specific access pattern, and the same line of code that wins on Conv1's broadcast loses on Conv2's sixteen scattered addresses.

<hr>

[**View the full Milestone 3 writeup (PDF)**]({{ '/assets/pdf/cuda_convolution_m3_report.pdf' | relative_url }}), the complete report with every optimization, its nsys and Nsight Compute profile, and the analysis behind each call.
