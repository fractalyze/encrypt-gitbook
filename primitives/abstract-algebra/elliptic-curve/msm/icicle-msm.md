# ICICLE MSM

ICICLE MSM follows [Pippenger's Algorithm](pippengers-algorithm.md), but introduces a few changes and optimizations for GPU. For a general overview on the process, take a look at Ingonyama's ICICLE docs [here.](https://dev.ingonyama.com/api/cpp/MSM#cuda-backend-msm)

Below is the dependency graph of all the functions in ICICLE MSM.

<figure><img src="../../../../.gitbook/assets/graphviz (2).svg" alt=""><figcaption></figcaption></figure>

* A large part of ICICLE MSM centers around ensuring data is kept and shared between the host and devices as needed with CUDA functions such as `cudaMemcpyAsync`.
* As seen in the diagram above, the leafs of the graph end in "kernel" functions. "Kernel" functions, as known in the GPU space, are designed to be executed by a large number of threads concurrently, thus enabling parallel processing of data.&#x20;

### Generic Configuration

```cpp
struct MSMConfig {
  icicleStreamHandle stream;
  int precompute_factor;
  int c;
  int bitsize;
  int batch_size;
  bool are_points_shared_in_batch;
  bool are_scalars_on_device;
  bool are_scalars_montgomery_form;
  bool are_points_on_device;
  bool are_points_montgomery_form;
  bool are_results_on_device;
  bool is_async;
  ConfigExtension* ext;
};
```

#### precompute\_factor

Number of precomputed doubled points per base point. Default value is 1, which means no pre-computation. Increasing this value decreases online computations and increases static memory footprint. Internally, the virtual amount of memory for storing points is `sizeof(point_t) * msm_size * precompute_factor` as extra `precompute - 1` doubled points should be stored accordingly per point.&#x20;

#### c

Bits per window for the Pippenger's algorithm. If not manually set the default value is $$\mathsf{min}(20, \mathsf{max}(1, \mathsf{degree} - 4))$$. The upper bound 20 is set to prevent bucket memory overflow in large buckets. The value degree - 4 is a chosen optimal value for MSM without pre-computation, according to the[ ICICLE docs](https://dev.ingonyama.com/api/cpp/MSM#choosing-optimal-parameters).&#x20;

#### bitsize

Maximum scalar size in bits. Typically this equals the bitsize of scalar field, but if a better upper bound is known as in the case of condensed scalars, it should be reflected in this variable.&#x20;

#### batch\_size

Number of MSMs to be run in parallel. By passing points and scalars with `batch_size` set to a value larger than 1, we achieve term-based parallelism.&#x20;

#### are\_points\_shared\_in\_batch

This indicates whether all MSMs use the same base points. If this is enabled, the number of base points given multiplied by `batch_size` should equal the number of scalars given.&#x20;

If these values are set to false, each MSM is processed in chunks, so that the stream of PCIe transfer and the other stream of MSM computation overlap.&#x20;

#### is\_async

Controls blocking behavior of MSM function. If set to false, function blocks CPU thread until GPU completes. Otherwise the user must synchronize manually before using the result. Automatically disabled if `are_results_on_device` is set to false.&#x20;

### CUDA-Specific Configuration

```cpp
namespace CudaBackendConfig {
  constexpr const char* CUDA_MSM_IS_BIG_TRIANGLE = "is_big_triangle";
  constexpr const char* CUDA_MSM_LARGE_BUCKET_FACTOR = "large_bucket_factor";
  constexpr const int CUDA_MSM_LARGE_BUCKET_FACTOR_DEFAULT_VAL = 10;
  constexpr const char* CUDA_MSM_NOF_CHUNKS = "nof_chunks";
  constexpr const int CUDA_MSM_NOF_CHUNKS_DEFAULT_VAL = 0;
} // namespace CudaBackendConfig
```

#### is\_big\_triangle

This specifies which method to use for bucket reduction. The difference between the big triangle method and normal reduction is described [below](https://app.gitbook.com/o/vlMwaGV8Ukn8S4gDaGYp/s/rwz1ZAZJtK5FHz4Y1esA/~/changes/35/primitives/abstract-algebra/elliptic-curve/msm/icicle-msm#big-triangle-method). This is automatically enabled when `c == 1` .&#x20;

#### large\_bucket\_factor

This sets the threshold for large bucket optimization. ICICLE internally sorts the buckets by its size and applies different accumulation methods for the buckets with size equal or larger than this factor multiplied by the statistically expected bucket size.&#x20;

When scalars are uniformly distributed, points are equally distributed to each bucket and large buckets typically don't exist. Large buckets exist in two cases:

1. When the scalar distribution isn't uniform.
2. When `c` does not divide the scalar bit-size.

#### nof\_chunks

The number of chunks for chunk processing. If not set, an optimal value is automatically chosen based on GPU memory usage.&#x20;

### [Term-based parallelism](cuzk.md#approach-2-thread-level-msm-partitioning) and Chunk-processing

As mentioned above, with a single set of base points and scalars, we can distribute it into multiple independent MSM computations across different GPU cores simultaneously and add them up.&#x20;

It's always better to use the batch mode instead of running single MSMs in serial as long as there is enough memory available. It supports running a batch of MSMs that share the same points as well as a batch of MSMs that use different points.

Internally, a single large MSM computation is also divided into sequential **chunks**. This approach enables reusing the same GPU resources for different portions of the computation and overlaps the time taken by host-to-device data transfer and actual computation.&#x20;

This strategy achieves two goals:

1. The memory usage peak is kept below the device memory capacity by splitting the problem into small chunks.
2. The actual computation starts earlier instead of waiting for the entire data for MSM transferred to device.&#x20;

Bucket method MSM is applied to each chunk to yield a partial result.&#x20;

```
Timeline ---> 
Stream 1: [Transfer Ch0] [Compute  Ch0] [Transfer Ch2] [Compute  Ch2]
Stream 2: [            ] [Transfer Ch1] [Compute  Ch1] [Transfer Ch3]
```

```
Double Buffer Memory Layout
├── Chunk Buffer 1: [Current chunk data]
├── Chunk Buffer 2: [Next chunk data] 
└── Shared Buckets: [B₁, B₂, ..., Bₘ] (accumulated across chunks)
```

ICICLE MSM estimates the amount of memory needed by the given MSM size, scalar and point types, batch size, and bits per window, and computes the lower bound of number of chunks that makes a sufficiently low amount of memory to be allocated in GPU such that it fits within the memory capacity.&#x20;

#### Required memory computation

Memory is estimated for chunks through the calculations below. `scalars_mem`, `indices_mem` ,  `points_mem` , and `buckets_mem` are estimated memory calculations per MSM. `nof_bms` refers to the number of windows (bucket modules).

```cpp
scalars_mem = sizeof(S) * msm_size * batch_size;
indices_mem = 7 * sizeof(unsigned) * msm_size * batch_size *
              nof_bms; // factor 7 as an estimation for the sorting extra memory.
points_mem = sizeof(A) * msm_size * config.precompute_factor * (config.are_points_shared_in_batch ? 1 : batch_size);
buckets_mem = 4 * sizeof(P) * (1 << c) * batch_size *
              nof_bms_after_precomputation; // factor 3 for the extra memory in the iterative reduction algorithm.
                                            // +1 for large buckets. can be reduced with some optimizations.
unsigned long total_memory;
unsigned long gpu_memory;
cudaMemGetInfo(&gpu_memory, &total_memory); // gpu_memory is the current available memory
if (config.are_points_on_device)
  gpu_memory += points_mem; // if points are already allocated we don't need to take them into account
if (config.are_scalars_on_device)
  gpu_memory += scalars_mem; // if scalars are already allocated we don't need to take them into account
reduced_gpu_memory =
  0.7 * static_cast<double>(gpu_memory); // 0.7 is an arbitrary value in order to have some spare memory
```

The optimal number of chunks is decided such that the temporary gpu memory required for one chunk (`scalars_mem + points_mem + indices_mem`) is lower than the memory available (`reduced_gpu_memory - buckets_mem`). Here, `buckets_mem` is not temporary as buckets are shared across the whole chunks.&#x20;

### Point Assignment

Each thread takes one scalar and executes the kernel function below.&#x20;

#### split\_and\_sort\_scalars

This kernel assigns unique bucket index to each of the buckets exist in the entire MSMs.&#x20;

A given scalar's c-bit digits are extracted to calculate its bucket index and an unique bucket index is given for each MSM instance in batch, window and bucket, like below. Exceptionally, the index of zero buckets are always regarded to be zero.&#x20;

```
unique bucket # = split_scalar == 0 ? 0 :
                (msm_index << (c + bm_bitsize))
                | (target_bm << c)
                |  split_scalar;
```

| MSM # | Window # | Split scalar | unique bucket # |
| ----- | -------- | ------------ | --------------- |
| 0     | 0        | 0            | 0               |
| 0     | 0        | 1            | 1               |
| 0     | 0        | 2            | 2               |
| 0     | 0        | 3            | 3               |
| 0     | 1        | 0            | 0               |
| 0     | 1        | 1            | 5               |
| 0     | 1        | 2            | 6               |
| 0     | 1        | 3            | 7               |
| 1     | 1        | 0            | 0               |
| 1     | 0        | 1            | 9               |
| 1     | 0        | 2            | 10              |
| 1     | 0        | 3            | 11              |
| 1     | 1        | 0            | 0               |
| 1     | 1        | 1            | 13              |
| 1     | 1        | 2            | 14              |
| 1     | 1        | 3            | 15              |

Each thread splits a scalar into digits of c and stores the bucket\_index and the point\_index into the corresponding vectors. Below is a small example of processing a small set of scalars.&#x20;

```
c = 2
points  = [P₀, P₁, P₂]
scalars = [5 , 8 , 13]

MSB → LSB
01 | 01 * P₀
10 | 00 * P₁
11 | 01 * P₂

1. 01 * P₀ :
    1. buckets_indices [1]
    2. point_indices   [0]
2. 00 * P₁ :
    1. buckets_indices [1, 0]
    2. point_indices   [0, 1]
3. 01 * P₂ :
    1. buckets_indices [1, 0, 1]
    2. point_indices   [0, 1, 2]
4. 01 * P₀ :
    1. buckets_indices [1, 0, 1, 5]
    2. point_indices   [0, 1, 2, 0]
5. 10 * P₁ :
    1. buckets_indices [1, 0, 1, 5, 6]
    2. point_indices   [0, 1, 2, 0, 1]
6. 11 * P₂ :
    1. buckets_indices [1, 0, 1, 5, 6, 7]
    2. point_indices   [0, 1, 2, 0, 1, 2]
```

After running the **split** kernel, it sorts the bucket and point index pairs first by `bucket_indices`, then by `point_indices` with `cub::DeviceRadixSort::SortPairs`.

<pre><code><strong>1. buckets_indices   [1, 0, 1, 5, 6, 7]
</strong><strong>2. point_indices     [0, 1, 2, 0, 1, 2]
</strong>→ sorting →
1. buckets_indices   [0, 1, 1, 5, 6, 7]
2. point_indices     [1, 0, 2, 0, 1, 2]
</code></pre>

Now it compresses and encodes these two indices arrays into

```
d_single_bucket_indices  = [0, 1, 5, 6, 7]     # unique bucket indices
d_bucket_sizes           = [1, 2, 1, 1, 1]     # bucket sizes for each index above
d_nof_buckets_to_compute = 5                   # size of d_single_bucket_indices
d_bucket_offsets         = [0, 1, 3, 4, 5]     # d_bucket_sizes incrementally added
```

> [cub(CUDA Unbound)](https://nvidia.github.io/cccl/cub/) provides 2-phase API for Device-wide primitives for allocation and actual operation. For example, DeviceRadixSort needs to be called twice like below, where the first call returns temporary storage size needed for sort in `temp_storage_bytes`. See the [API docs](https://nvidia.github.io/cccl/cub/api/structcub_1_1DeviceRadixSort.html#_CPPv4N3cub15DeviceRadixSortE) for more details.&#x20;
>
> ```cpp
>   CHK_IF_RETURN(cub::DeviceRadixSort::SortPairs(
>     sort_indices_temp_storage, sort_indices_temp_storage_bytes, bucket_indices, d_sorted_bucket_indices,
>     point_indices, d_sorted_point_indices, input_indexes_count, 0, sizeof(unsigned) * 8, stream));
>   CHK_IF_RETURN(cudaMallocAsync(&sort_indices_temp_storage, sort_indices_temp_storage_bytes, stream));
>   CHK_IF_RETURN(cub::DeviceRadixSort::SortPairs(
>     sort_indices_temp_storage, sort_indices_temp_storage_bytes, bucket_indices, d_sorted_bucket_indices,
>     point_indices, d_sorted_point_indices, input_indexes_count, 0, sizeof(unsigned) * 8, stream));
> ```

#### allocate\_and\_sort\_buckets\_by\_size

This sorts `d_bucket_offsets` and `d_single_bucket_indices` by `d_bucket_sizes` in descending order of size. By sorting the buckets by their sizes, the number of buckets with size exceeding some threshold can easily be counted by scanning the sorted array with a cutoff finder.&#x20;

### Bucket Accumulation

Overall, bucket accumulation in ICICLE MSM aims to complete the same process as outlined in Pippenger's: add up all the points in each bucket.

Specially for ICICLE MSM, bucket accumulation is split into two parts: Large bucket accumulation and generic bucket accumulation.

A bucket is considered large if it passes a certain threshold factor. By default, the threshold is 10x the average bucket size. The large bucket accumulation function itself calls 5 separate kernel functions, in which points in buckets are split for accumulation, and then all the split sections are reduced together to one result point per large bucket.

On the other hand, generic bucket accumulation calls a single kernel function for each bucket, meaning that each bucket's accumulation is done on a separate thread.

### Bucket Reduction

#### Big Triangle Method

```
Result = B₁ + (B₁ + B₂) + (B₁ + B₂ + B₃) + ... + (B₁ + B₂ + ... + B꜀)
```

This method is equivalent to the running-sum based bucket reduction method we introduced in [Pippenger's algorithm](pippengers-algorithm.md#id-3.-bucket-reduction). Each window is assigned to a thread and the running sum is computed in parallel.&#x20;

When performing a large MSM, typically big triangle method is not the best choice, because the number of windows is not large enough to fully benefit from parallelism.&#x20;

According to the [ICICLE docs](https://dev.ingonyama.com/api/cpp/MSM#choosing-optimal-parameters), `is_big_triangle` should be `false` in almost all cases as it reduces parallelism. It might provide better results only for very small MSMs (smaller than $$2^8$$) with a large batch size (larger than $$100$$).&#x20;

#### Iterative Reduction Method (default, as long as c > 1)

The Iterative Reduction method uses a iterative approach, progressively reducing the window size $$c$$ by half through multiple stages until reaching $$c = 1$$, so that the original $$W \cdot 2^c$$ total buckets is reduced to final array of $$W \cdot c$$ buckets.&#x20;

1\) Initialize

* $$\mathsf{source\_bits} \leftarrow c$$
* $$\mathsf{source\_windows} \leftarrow W$$
* $$\mathsf{source\_buckets} \leftarrow [W][2^c]$$ group elements

2\) Repeat until $$\mathsf{target\_bits} == 1$$

* $$\mathsf{target\_bits} \leftarrow \mathsf{ceil}(\mathsf{source\_bits} / 2)$$
* $$\mathsf{target\_windows} \leftarrow 2 \cdot \mathsf{source\_windows}$$
* $$\mathsf{target\_buckets\_count} \leftarrow \mathsf{target\_windows} \cdot 2^{\mathsf{target\_bits}}$$
* for ($$j = 1$$ to $$\mathsf{target\_bit\_counts}$$)&#x20;
  * single\_stage\_multi\_reduction\_kernel : Reduce the $$2^c$$ buckets in each windows into half.
  * Above kernel is launched twice per iteration and run in parallel. Each kernel takes a different direction (from bottom windows vs. from top windows) and handles different indices of windows (odd vs. even) so that their executions are independent.&#x20;

In each iteration, a  $$[W][2^c]$$ buckets layout is collapsed into $$[2W][2^{c/2}]$$ buckets, which means each window is split into two windows with half of the bitwidth. Note that each step preserves the total sum, while drastically decreasing the number of buckets.&#x20;

The examples below show a single stage of the overall process, and how a single window is processed during said stage. Note that as aforementioned, in every stage, the number of windows are doubled while the number of buckets decrease by a factor of the current bits\_per\_window c.

```
// First stage of processing all windows

c = 4
Before
(Window 3): [B₀₀₀₀, B₀₀₀₁, ... B₁₁₁₁] // 16 buckets 
(Window 2): [B₀₀₀₀, B₀₀₀₁, ... B₁₁₁₁]
(Window 1): [B₀₀₀₀, B₀₀₀₁, ... B₁₁₁₁]
(Window 0): [B₀₀₀₀, B₀₀₀₁, ... B₁₁₁₁]

Total sum : 
sum(Window 0) + 2^4 * sum(Window 1) + 2^8 * sum(Window 2) + 2^12 * sum(Window 3)

c = 2
After
(Window 7): [B₀₀, B₀₁, B₁₀, B₁₁] // 4 buckets 
(Window 6): [B₀₀, B₀₁, B₁₀, B₁₁] 
(Window 5): [B₀₀, B₀₁, B₁₀, B₁₁] 
(Window 4): [B₀₀, B₀₁, B₁₀, B₁₁] 
(Window 3): [B₀₀, B₀₁, B₁₀, B₁₁] 
(Window 2): [B₀₀, B₀₁, B₁₀, B₁₁] 
(Window 1): [B₀₀, B₀₁, B₁₀, B₁₁] 
(Window 0): [B₀₀, B₀₁, B₁₀, B₁₁] 

Total sum : 
sum(Window 0) + 2^2 * sum(Window 1) + 2^4 * sum(Window 2) + ... + 2^14 * sum(Window 7)

```

```
// For each window at each stage
c = 4
Before (Window 0): [B₀₀₀₀, B₀₀₀₁, B₀₀₁₀, B₀₀₁₁, B₀₁₀₀, B₀₁₀₁, B₀₁₁₀, B₀₁₁₁, B₁₀₀₀, B₁₀₀₁, B₁₀₁₀, B₁₀₁₁, B₁₁₀₀, B₁₁₀₁, B₁₁₁₀, B₁₁₁₁]
sum = 0*B₀₀₀₀ + 1*B₀₀₀₁ + 2*B₀₀₁₀ + 3*B₀₀₁₁ + 4*B₀₁₀₀ + 5*B₀₁₀₁ + 6*B₀₁₁₀ + 7*B₀₁₁₁ + 8*B₁₀₀₀ + 9*B₁₀₀₁ + 10*B₁₀₁₀ + 11*B₁₀₁₁ + 12*B₁₁₀₀ + 13*B₁₁₀₁ + 14*B₁₁₁₀ + 15*B₁₁₁₁

c = 2
After  (Window 1): [B₀₀, B₀₁, B₁₀, B₁₁] // upper 2 bits 
After  (Window 0): [B₀₀, B₀₁, B₁₀, B₁₁] // lower 2 bits
```

$$
\begin{aligned}
&\textbf{Window 0} \\[6pt]
B_{{\color{turquoise}{00}}}^0 &=
  ( B_{{\color{orange}{00}}{\color{turquoise}{00}}} + B_{{\color{orange}{01}}{\color{turquoise}{00}}} ) +
  ( B_{{\color{orange}{10}}{\color{turquoise}{00}}} + B_{{\color{orange}{11}}{\color{turquoise}{00}}} ) \\[6pt]
B_{{\color{turquoise}{01}}}^0 &=
  ( B_{{\color{orange}{00}}{\color{turquoise}{01}}} + B_{{\color{orange}{01}}{\color{turquoise}{01}}} ) +
  ( B_{{\color{orange}{10}}{\color{turquoise}{01}}} + B_{{\color{orange}{11}}{\color{turquoise}{01}}} ) \\[6pt]
B_{{\color{turquoise}{10}}}^0 &=
  ( B_{{\color{orange}{00}}{\color{turquoise}{10}}} + B_{{\color{orange}{01}}{\color{turquoise}{10}}} ) +
  ( B_{{\color{orange}{10}}{\color{turquoise}{10}}} + B_{{\color{orange}{11}}{\color{turquoise}{10}}} ) \\[6pt]
B_{{\color{turquoise}{11}}}^0 &=
  ( B_{{\color{orange}{00}}{\color{turquoise}{11}}} + B_{{\color{orange}{01}}{\color{turquoise}{11}}} ) +
  ( B_{{\color{orange}{10}}{\color{turquoise}{11}}} + B_{{\color{orange}{11}}{\color{turquoise}{11}}} )
\end{aligned}
\kern 3em
\begin{aligned}
&\textbf{Window 1} \\[6pt]
B_{{\color{orange}{00}}}^1 &=
  ( B_{{\color{orange}{00}}{\color{turquoise}{00}}} + B_{{\color{orange}{00}}{\color{turquoise}{01}}} ) +
  ( B_{{\color{orange}{00}}{\color{turquoise}{10}}} + B_{{\color{orange}{00}}{\color{turquoise}{11}}} ) \\[6pt]
B_{{\color{orange}{01}}}^1 &=
  ( B_{{\color{orange}{01}}{\color{turquoise}{00}}} + B_{{\color{orange}{01}}{\color{turquoise}{01}}} ) +
  ( B_{{\color{orange}{01}}{\color{turquoise}{10}}} + B_{{\color{orange}{01}}{\color{turquoise}{11}}} ) \\[6pt]
B_{{\color{orange}{10}}}^1 &=
  ( B_{{\color{orange}{10}}{\color{turquoise}{00}}} + B_{{\color{orange}{10}}{\color{turquoise}{01}}} ) +
  ( B_{{\color{orange}{10}}{\color{turquoise}{10}}} + B_{{\color{orange}{10}}{\color{turquoise}{11}}} ) \\[6pt]
B_{{\color{orange}{11}}}^1 &=
  ( B_{{\color{orange}{11}}{\color{turquoise}{00}}} + B_{{\color{orange}{11}}{\color{turquoise}{01}}} ) +
  ( B_{{\color{orange}{11}}{\color{turquoise}{10}}} + B_{{\color{orange}{11}}{\color{turquoise}{11}}} )
\end{aligned}

\\[1.5em]

\text{sum} = (0*B_{{\color{turquoise}{00}}}^0+ 1*B_{{\color{turquoise}{01}}}^0+ 2*B_{{\color{turquoise}{10}}}^0+ 3*B_{{\color{turquoise}{11}}}^0)
 + 2^2 (0*B_{{\color{orange}{00}}}^1+ 1*B_{{\color{orange}{01}}}^1+ 2*B_{{\color{orange}{10}}}^1+ 3*B_{{\color{orange}{11}}}^1)
$$



The table below shows the incremental progress for each iteration of the algorithm.

|       | c    | Windows | Buckets         |
| ----- | ---- | ------- | --------------- |
| 0     | 16→8 | 16→32   | 1,048,592→8,192 |
| 1     | 8→4  | 32→64   | 8,192→1,024     |
| 2     | 4→2  | 64→128  | 1,024→512       |
| 3     | 2→1  | 128→256 | 512→512         |
| Final | 1    | 256     | 512             |

#### Final Accumulation (Window Reduction)

After reaching c = 1, we have c x W windows and each window holds 2 buckets where one is the zero bucket so can be ignored. Now it runs the weighted sum over the windows, which is identical to the window reduction in Pippenger's algorithm.&#x20;

In ICICLE MSM, window reduction is grouped together with bucket reduction function-wise and is instead named "Final accumulation"; however, its algorithm follows the same logic as Pippenger's with [Double-and-Add](../scalar-multiplication/double-and-add.md) being done on single thread.

### Precomputation

Precomputation stores, per base $$P$$, the sequence  $$[P, 2^{\mathsf{shift}} \cdot P, 2^{2·\mathsf{shift}} \cdot P, ... ]$$ where shift is the scalar bitwidth divided by the precompute factor.

```
unsigned total_nof_bms = (P::SCALAR_FF_NBITS - 1) / c + 1;
unsigned shift = c * ((total_nof_bms - 1) / precompute_factor + 1);
```

&#x20;That is, it precomputes 'precompute factor' number of points which are doubles of the original point P.&#x20;

```
template <typename A, typename P>
__global__ void
precompute_points_kernel(const A* points, int shift, int prec_factor, int count, A* points_out, bool is_montgomery)
{
  int tid = blockIdx.x * blockDim.x + threadIdx.x;
  // ...
  P point = P::from_affine(point_a);
  points_out[tid * prec_factor] = P::to_affine(point);
  for (int i = 1; i < prec_factor; i++) {
    for (unsigned j = 0; j < shift; j++)
      point = P::dbl(point);
    points_out[tid * prec_factor + i] = P::to_affine(point);
  }
}
```

```
c = 2
points  = [P₀, P₁, P₂, P₃]
scalars = [5 , 8 , 13, 5]

MSB → LSB
01 | 01 * P₀
10 | 00 * P₁   
11 | 01 * P₂
01 | 01 * P₃
// number of windows is 2. 
// number of buckets per window is 4.


// Precomputation with factor = 2... 

points_out = [         P₀,        P₁,        P₂,        P₃, 
                (1<<2)*P₀, (1<<2)*P₁, (1<<2)*P₂, (1<<2)*P₃]

Following MSM:
01 * P₀
00 * P₁   
01 * P₂
01 * P₃
01 * (1<<2)*P₀
10 * (1<<2)*P₁
11 * (1<<2)*P₂
01 * (1<<2)*P₃

// The MSM now has # of windows as 1 and # of points as 8. 
```

Instead of the bases points given as input, this expanded `points_out` array is used for the bases throughout the execution. `scalars` array is not expanded and does not affect the static memory usage.&#x20;

This multiplies the points array length by precompute factor, and reduces the number of bucket-modules processed online by that factor. So precomputation with factor 2 doubles the size of memory needed to store the bases points and halves the number of windows.&#x20;

```
const unsigned nof_points = nof_scalars * precompute_factor;
const unsigned nof_bms_per_msm = (total_bms_per_msm - 1) / precompute_factor + 1;
```

During MSM, digits are remapped into groups; the group index selects which precomputed copy to use, and the within-group window index is packed into the bucket key and kernels index into the expanded points array.

When the points are not known in advance we cannot use precomputation. However, in most protocols the points are known in advance and precomputation can be used unless limited by memory. According to the [ICICLE docs](https://dev.ingonyama.com/api/cpp/MSM#choosing-optimal-parameters), usually it's best practice to use maximum precomputation, possibly such that we end up with only a single window combined with larger c.&#x20;

### Provided example configurations

These have been run on the BLS12-377 curve with a NVIDIA GeForce RTX 3090 Ti.&#x20;

<table><thead><tr><th width="113.20703125">MSM size</th><th width="118.41796875">Batch size</th><th width="167.74713134765625">Precompute factor</th><th width="66.053955078125">c</th><th width="205.4112548828125">Memory estimation (GB)</th><th width="179.7896728515625">Actual memory (GB)</th><th width="191.4765625">Single MSM time (ms)</th></tr></thead><tbody><tr><td>10</td><td>1</td><td>1</td><td>9</td><td>0.00227</td><td>0.00277</td><td>9.2</td></tr><tr><td>10</td><td>1</td><td>23</td><td>11</td><td>0.00259</td><td>0.00272</td><td>1.76</td></tr><tr><td>10</td><td>1000</td><td>1</td><td>7</td><td>0.94</td><td>1.09</td><td>0.051</td></tr><tr><td>10</td><td>1000</td><td>23</td><td>11</td><td>2.59</td><td>2.74</td><td>0.025</td></tr><tr><td>15</td><td>1</td><td>1</td><td>11</td><td>0.011</td><td>0.019</td><td>9.9</td></tr><tr><td>15</td><td>1</td><td>16</td><td>16</td><td>0.061</td><td>0.065</td><td>2.4</td></tr><tr><td>15</td><td>100</td><td>1</td><td>11</td><td>1.91</td><td>1.92</td><td>0.84</td></tr><tr><td>15</td><td>100</td><td>19</td><td>14</td><td>6.32</td><td>6.61</td><td>0.56</td></tr><tr><td>18</td><td>1</td><td>1</td><td>14</td><td>0.128</td><td>0.128</td><td>14.4</td></tr><tr><td>18</td><td>1</td><td>15</td><td>17</td><td>0.40</td><td>0.42</td><td>5.9</td></tr><tr><td>22</td><td>1</td><td>1</td><td>17</td><td>1.64</td><td>1.65</td><td>68</td></tr><tr><td>22</td><td>1</td><td>13</td><td>21</td><td>5.67</td><td>5.94</td><td>54</td></tr><tr><td>24</td><td>1</td><td>1</td><td>18</td><td>6.58</td><td>6.61</td><td>232</td></tr><tr><td>24</td><td>1</td><td>7</td><td>21</td><td>12.4</td><td>13.4</td><td>199</td></tr></tbody></table>

Running a batch of 100 ($$< 2^7$$) degree 15 MSM can be approximated to be identical to (or cheaper than) running a degree 22 single MSM.  It turns out that with or without optimal precompute factor, single MSM of degree 22 is faster than running 100 MSMs of degree 15 (84 vs. 68 / 56 vs. 54). Notice that this result may differ depending on curves and devices.&#x20;

## References

* [https://dev.ingonyama.com/api/cpp/msm](https://dev.ingonyama.com/api/cpp/msm)
* [https://github.com/ingonyama-zk/icicle-snark](https://github.com/ingonyama-zk/icicle-snark)

> Written by [Carson Lee](https://app.gitbook.com/u/Hm5RHrPlu2fbxwXnpUgxmvXVTwB3 "mention")  & [Ashley Jeong](https://app.gitbook.com/u/PNJ4Qqiz7kSxSs58UIaauk95AO43 "mention") of Fractalyze
