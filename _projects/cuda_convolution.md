---
layout: page
title: GPU-Accelerated CNN Convolution
description: Profiling-driven CUDA optimization of the convolution layers in a small CNN (ECE 408), built around a tensor-core roofline prediction made before the kernel was written and then checked against nsys and Nsight Compute.
img: assets/img/cuda_tensorcore_optime_table.jpeg
importance: 3
category: work
related_publications: false
---

ECE 408 is the applied parallel programming course, and the project is to take the forward convolution layers of a small LeNet-style CNN and make them run fast on an NVIDIA GPU. The convolution is done the way most GPU libraries do it, by unrolling each input patch into a column (im2col) and turning the whole layer into one matrix multiply. Milestone 3 is a sequence of separate optimizations, and each one gets implemented, profiled with nsys, and then judged on whether it actually helped and why. What makes it a real exercise rather than a leaderboard chase is how many textbook optimizations do nothing on this particular workload, and being able to say in advance which ones those would be.

## Predicting the tensor-core result before writing it

The part I care about most is the tensor-core optimization, because I called the result before writing a line of it. Tensor cores are dedicated matmul hardware on each SM. On Ampere a single warp can issue an `mma.sync` that does a 16x16x16 FP16 multiply into an FP32 accumulator in one instruction, which works out to 8192 FMAs per warp per instruction against the SIMT path's one FMA per thread per cycle. The throughput ceiling is enormous, so the textbook move is to reach for them.

A roofline argument said not to bother. These convolution kernels have an arithmetic intensity around 0.25 FLOPs per byte. The crossover on the A40 roofline, the point where a kernel stops being limited by memory bandwidth and starts being limited by compute, sits near 250 FLOPs per byte. The kernel is on the memory-bound side of that line by a factor of roughly a thousand. Putting faster matmul hardware behind something that spends almost all of its time moving bytes cannot help, because the matmul was never the stage it was waiting on. So I wrote down the prediction first: tensor cores would lose on Conv1 and maybe break even on Conv2.

There is a shape problem stacked on top of the roofline one. WMMA fixes the tile at 16 in every dimension, so M, N, and K all have to be multiples of 16 or you pad up and multiply by zeros. Conv1 has `Map_out=4`, which forces the M dimension to 16 with 12 padded rows, so three quarters of every multiply is against zero. Conv2 lines up on M, but its K is `Channel*K*K = 4*7*7 = 196`, which rounds up to 208, a smaller 6% waste inside each dot product. I wrote the kernel as one warp per block, converting FP32 to FP16 on the way into shared memory and accumulating in FP32 so the result stays accurate.

```cpp
// one warp per block; each warp computes a 16x16 output tile
wmma::fragment<wmma::matrix_a, 16, 16, 16, __half, wmma::row_major> a_frag;
wmma::fragment<wmma::matrix_b, 16, 16, 16, __half, wmma::row_major> b_frag;
wmma::fragment<wmma::accumulator, 16, 16, 16, float> c_frag;
wmma::fill_fragment(c_frag, 0.0f);

for (int t = 0; t < num_tiles; t++) {
    // load A and B tiles into shared memory, zero-filling out-of-range
    // elements and converting fp32 -> fp16 on the way in.
    // that zero-fill is also exactly where Conv1's wasted work comes from.
    load_tiles_to_shared(tileA, tileB, t);
    __syncthreads();
    wmma::load_matrix_sync(a_frag, &tileA[0][0], 16);
    wmma::load_matrix_sync(b_frag, &tileB[0][0], 16);
    wmma::mma_sync(c_frag, a_frag, b_frag, c_frag);  // 16x16x16 in one instruction
    __syncthreads();
}
wmma::store_matrix_sync(&tileOut[0][0], c_frag, 16, wmma::mem_row_major);
```

The measured result got the direction right and one detail wrong.

<div class="row justify-content-sm-center">
    <div class="col-sm-10 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/cuda_tensorcore_optime_table.jpeg" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Tensor-core (WMMA) op times against the plain baseline. Both convolutions came out slower, and Conv2 (Op Time 2) was worse than Conv1, which I had not expected.
</div>

Both convolutions were slower than the plain baseline, and Conv2 was worse than Conv1. The Conv1 outcome was the one I had predicted. The Conv2 regression caught me off guard, because I had guessed that layer would at least break even. Nsight Compute explained the miss and corrected my mental model in the same pass. Achieved occupancy was pinned at 33%, capped by register pressure, and the compute breakdown put the tensor pipe at about 3% utilization while the ALU sat near 60% grinding through the integer index math for the unroll. The kernel was bottlenecked on address arithmetic, not on matmul throughput and not purely on memory bandwidth the way the bare roofline number implied. I had written a tensor-core kernel that barely touches the tensor cores. The high-level call held up, since tensor cores were the wrong tool for matmuls this small, but the actual reason was one level more specific than my roofline sketch.

## The optimizations that did not help, and why

The same pattern runs through the rest of the report. Putting the convolution weights in constant memory (op_0) is the standard move for read-only filter data, and I expected a clean win. It helped Conv1, where four mask rows get broadcast to every thread and the constant cache is built for exactly that. It regressed Conv2. At `Map_out=16` the accesses stop being a uniform broadcast and serialize through the constant cache instead, so the change that helped one layer hurt the other. What I took from it is that there is no such thing as a good optimization in the abstract, only one that happens to fit a particular access pattern.

FP16 on the ordinary SIMT path (op_5) came out a few percent slower, for a related reason. Halving the bytes per element only helps if you are bandwidth bound in the way FP16 is meant to fix, and the cache hit rates barely moved, because they depend on the access pattern rather than the element size. Each load also picked up a float-to-half conversion, and packing two halves into a sector left more of each 32-byte memory transaction unused. Half precision only earns its keep once it feeds the tensor cores, which is the thread back to req_1.

## When the metric measures the wrong thing

The streams optimization (req_0) produced the most misleading-looking numbers in the whole report. Splitting the batch into four chunks with one CUDA stream each overlaps the host-to-device copy, the kernel, and the device-to-host copy across the DMA engines and the SMs, using pinned host memory so the transfers do not serialize through a staging buffer. The op-time table for it reads as near zero, around 0.003 ms.

<div class="row justify-content-sm-center">
    <div class="col-sm-10 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/cuda_streams_optime_table.jpeg" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Streams op times reading as roughly 0.003 ms. The kernels are not running in microseconds. The harness is timing a function that, under this structure, is empty.
</div>

Those are not the kernels finishing in microseconds. The course harness times only the `conv_forward_gpu` function, and under the streamed structure that function is empty, because all of the real work has moved into the prolog where the streams get set up and launched. The measurement is pointed at the wrong place. The number that means something is the Layer Time line in the same run, around 463 ms for Conv1 and 398 ms for Conv2. The nsys profile is where the real story lives: the device-to-host transfer was the single most expensive stage at 71 ms, more than the 57.6 ms matrix multiply, so overlapping it hides the stage that was genuinely on the critical path. The streams are not in my final submission, since the assignment rules forbid them there, but the profiling was worth keeping anyway.

Working through all of this changed how I read a profiler. At the start I could launch a kernel and read a wall-clock time, and the rest of the nsys output was noise to me. By the end I was reading occupancy, memory throughput, and stall reasons as one workflow, and more usefully I had learned which numbers to distrust. Nsight Compute's kernel replay can inflate the reported op times by a few hundred times, which is why I take the real timings off the sbatch stdout rather than the profiler's own summary.

If I went back to the tensor-core kernel, the first move would be cutting the per-block index math and packing more warps into each block, so the SMs are not running half empty while the ALU does bookkeeping. That would not turn it into a win on a workload this small, but it would at least let the hardware I asked for do the work I asked it to.

<hr>

[**View the full Milestone 3 writeup (PDF)**]({{ '/assets/pdf/cuda_convolution_m3_report.pdf' | relative_url }}), the complete report with every optimization, its nsys and Nsight Compute profile, and the analysis behind each call.
