# Overview of CUDA C++ Implementation

This section introduces the implementation of a direct-method \$N\$-body simulation using **CUDA C++**.

---

## Implementation Method

* Rewrite functions to run on the GPU as `__global__` functions

  * Add `__global__` before the function definition:

    ```c++
    __global__ void func_device(double *__restrict arg0, int32_t *__restrict arg1, ...)
    ```

  * Remove the outermost `for` loop and replace it with the automatically assigned thread ID.

    * In many tutorials, the loop is replaced with an `if` statement, but here we avoid extra conditionals by allocating memory as a multiple of the thread block size.
    * Extra particles should be handled carefully (e.g., by setting their mass to zero) so they do not affect the results.

    ```c++
    // for (int32_t ii = 0; ii < num; ii++) {
    const int32_t ii = blockIdx.x * blockDim.x + threadIdx.x;
    ```

* Functions called from GPU kernels should be marked with `__device__`:

  ```c++
  __device__ void sub_func_device(double *__restrict arg0, int32_t *__restrict arg1, ...)
  ```

* Launch the `__global__` function from the CPU:

  ```c++
  func_device<<<BLOCKSIZE(num, NTHREADS), NTHREADS>>>(arg0, arg1, ...);
  ```

  * `NTHREADS` is the number of threads per block, a key parameter that heavily impacts performance.
  * The number of thread blocks must also be specified. A convenient macro is:

    ```c++
    constexpr int32_t BLOCKSIZE(const int32_t num, const int32_t thread) { 
        return (1 + ((num - 1) / thread)); 
    }
    ```

---

## GPU Memory Management

Two approaches are commonly used:

### 1. Unified Memory

1. Allocate memory:

   ```c++
   cudaMallocManaged((void **)ptr, size_of_array_in_byte);
   ```
2. Free memory:

   ```c++
   cudaFree(ptr);
   ```

### 2. Explicit CPU–GPU Memory Management

1. Allocate memory:

   ```c++
   cudaMalloc((void **)ptr_dev, size_of_array_in_byte);     // GPU memory
   cudaMallocHost((void **)ptr_hst, size_of_array_in_byte); // pinned CPU memory (faster for large transfers)
   ```
2. Transfer data:

   ```c++
   cudaMemcpy(dst_dev, src_hst, size_of_array_in_byte, cudaMemcpyHostToDevice); // CPU → GPU
   cudaMemcpy(dst_hst, src_dev, size_of_array_in_byte, cudaMemcpyDeviceToHost); // GPU → CPU
   ```
3. Free memory:

   ```c++
   cudaFree(ptr_dev);
   cudaFreeHost(ptr_hst);
   ```

---

## Synchronization

* **When `cudaDeviceSynchronize()` is NOT required:**

  * Only the default stream is used
  * Data transfers are performed with synchronous `cudaMemcpy()`

* **When `cudaDeviceSynchronize()` IS required:**

  * Multiple CUDA streams are used
  * Asynchronous CUDA functions are used
  * Unified Memory causes incorrect results when the CPU reads before GPU completion
  * For performance measurement and profiling

```c++
cudaDeviceSynchronize();
```

---

## Example Implementations

| Source Code                                                                                      | Description                                                        |
| ------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------ |
| [cpp/cuda/00\_unified\_base/nbody\_leapfrog2.cu](/cpp/cuda/00_unified_base/nbody_leapfrog2.cu)   | Unified Memory, Leapfrog method                                    |
| [cpp/cuda/01\_unified\_rsqrt/nbody\_leapfrog2.cu](/cpp/cuda/01_unified_rsqrt/nbody_leapfrog2.cu) | Unified Memory with `rsqrt()`, Leapfrog method                     |
| [cpp/cuda/02\_unified\_shmem/nbody\_leapfrog2.cu](/cpp/cuda/02_unified_shmem/nbody_leapfrog2.cu) | Unified Memory with `rsqrt()` and shared memory, Leapfrog method   |
| [cpp/cuda/10\_memcpy\_base/nbody\_leapfrog2.cu](/cpp/cuda/10_memcpy_base/nbody_leapfrog2.cu)     | cudaMemcpy-based, Leapfrog method                                  |
| [cpp/cuda/11\_memcpy\_rsqrt/nbody\_leapfrog2.cu](/cpp/cuda/11_memcpy_rsqrt/nbody_leapfrog2.cu)   | cudaMemcpy-based with `rsqrt()`, Leapfrog method                   |
| [cpp/cuda/12\_memcpy\_shmem/nbody\_leapfrog2.cu](/cpp/cuda/12_memcpy_shmem/nbody_leapfrog2.cu)   | cudaMemcpy-based with `rsqrt()` and shared memory, Leapfrog method |
| [cpp/cuda/00\_unified\_base/nbody\_hermite4.cu](/cpp/cuda/00_unified_base/nbody_hermite4.cu)     | Unified Memory, Hermite method                                     |
| [cpp/cuda/01\_unified\_rsqrt/nbody\_hermite4.cu](/cpp/cuda/01_unified_rsqrt/nbody_hermite4.cu)   | Unified Memory with `rsqrt()`, Hermite method                      |
| [cpp/cuda/02\_unified\_shmem/nbody\_hermite4.cu](/cpp/cuda/02_unified_shmem/nbody_hermite4.cu)   | Unified Memory with `rsqrt()` and shared memory, Hermite method    |
| [cpp/cuda/10\_memcpy\_base/nbody\_hermite4.cu](/cpp/cuda/10_memcpy_base/nbody_hermite4.cu)       | cudaMemcpy-based, Hermite method                                   |
| [cpp/cuda/11\_memcpy\_rsqrt/nbody\_hermite4.cu](/cpp/cuda/11_memcpy_rsqrt/nbody_hermite4.cu)     | cudaMemcpy-based with `rsqrt()`, Hermite method                    |
| [cpp/cuda/12\_memcpy\_shmem/nbody\_hermite4.cu](/cpp/cuda/12_memcpy_shmem/nbody_hermite4.cu)     | cudaMemcpy-based with `rsqrt()` and shared memory, Hermite method  |

---

## Performance Optimization

### Threads per Block

* `NTHREADS` has a major impact on performance.
* Recommended values: multiples of 32, generally ≥128.
* Optimal values depend on problem size, GPU model, and CUDA version.
* In this project:

  * Leapfrog method: best with **single precision** at \$N = 4,194,304\$
  * Hermite method: best with **double precision** at \$N = 4,194,304\$

### Fast Math Instructions

* In \$N\$-body simulations, the **reciprocal square root** is the most critical operation.
* On NVIDIA GPUs, `rsqrtf()` offers significant speedup:

  * Slightly less accurate than `1.0/sqrt()`, but usually acceptable.
  * For double precision, using `rsqrtf()` as a seed with **Newton–Raphson refinement** is faster than computing `1.0 / sqrt()` directly.
  * Example: [Leapfrog](/cpp/cuda/11_memcpy_rsqrt/nbody_leapfrog2.cu), [Hermite](/cpp/cuda/11_memcpy_rsqrt/nbody_hermite4.cu)

### Shared Memory

* On-chip, high-speed but limited-capacity memory accessible by all threads in a block.
* Consistency must be managed manually (e.g., with `__syncthreads()`).

**Static allocation** (recommended, simpler):

```c++
__shared__ double shmem[NTHREADS];
```

* Max capacity: 48 KB per block.
* Example: [Leapfrog](/cpp/cuda/12_memcpy_shmem/nbody_leapfrog2.cu)

**Dynamic allocation** (allocated at kernel launch):

```c++
extern __shared__ double shmem[];  // outside kernel

double *ptr0_shmem = (double *)shmem;
double *ptr1_shmem = (double *)&ptr0_shmem[NTHREADS];

func_device<<<BLOCKSIZE(num, NTHREADS), NTHREADS, allocated_size_in_byte>>>(arg0, arg1, ...);
```

* Example: [Hermite](/cpp/cuda/12_memcpy_shmem/nbody_hermite2.cu)

---

## NVIDIA CUDA SDK Notes

### Compilation & Linking

Typical compiler flags:

```sh
-gencode arch=compute_80,code=sm_80 \
-Xptxas -v,-warn-spills,-warn-lmem-usage \
-lineinfo
```

* Optimized for **CC 8.0 (NVIDIA A100)**
* Outputs useful information on spills and local memory usage
