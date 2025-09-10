# Overview of OpenMP (target directive) Implementation

Introduction to the implementation of an \$N\$-body simulation code (direct method) using OpenMP (target directive).

## Implementation Method

* Add directives to the `for` loops that you want to execute on the GPU:

  ```c++
  #pragma omp target teams distribute parallel for simd
  for(int32_t ii = 0; ii < num; ii++){
    ...
  }
  ```

  * The `simd` clause is optional.

  * Option: Suggest the number of threads per thread block.

    ```c++
    #pragma omp target teams distribute parallel for simd thread_limit(256)
    for(int32_t ii = 0; ii < num; ii++){
      ...
    }
    ```

  * Implementation using the `loop` clause:

    ```c++
    #pragma omp target teams loop
    for(int32_t ii = 0; ii < num; ii++){
      ...
    }
    ```

    * Option: Suggest the number of threads per thread block.

      ```c++
      #pragma omp target teams loop thread_limit(256)
      for(int32_t ii = 0; ii < num; ii++){
        ...
      }
      ```

* If **Unified Memory** is not used, add data directives:

  1. Allocate memory on the GPU:

     ```c++
     #pragma omp target enter data map(alloc: ptr[0:num])
     ```

  2. Transfer data from CPU to GPU:

     ```c++
     #pragma omp target update to(ptr[0:num])
     ```

  3. Transfer data from GPU to CPU:

     ```c++
     #pragma omp target update from(ptr[0:num])
     ```

  4. Free memory on the GPU:

     ```c++
     #pragma omp target exit data map(delete: ptr[0:num])
     ```

## Implementation Examples

| Source Code                                                                                  | Implementation Overview                                                    | Notes                                     |
| -------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------- | ----------------------------------------- |
| [cpp/openmp/0\_dist/nbody\_leapfrog2.cpp](/cpp/openmp/0_dist/nbody_leapfrog2.cpp)            | Implementation using `distribute` clause, Unified Memory, Leapfrog method  |                                           |
| [cpp/openmp/1\_dist\_data/nbody\_leapfrog2.cpp](/cpp/openmp/1_dist_data/nbody_leapfrog2.cpp) | Implementation using `distribute` clause, data directives, Leapfrog method |                                           |
| [cpp/openmp/a\_loop/nbody\_leapfrog2.cpp](/cpp/openmp/a_loop/nbody_leapfrog2.cpp)            | Implementation using `loop` clause, Unified Memory, Leapfrog method        |                                           |
| [cpp/openmp/b\_loop\_data/nbody\_leapfrog2.cpp](/cpp/openmp/b_loop_data/nbody_leapfrog2.cpp) | Implementation using `loop` clause, data directives, Leapfrog method       |                                           |
| [cpp/openmp/0\_dist/nbody\_hermite4.cpp](/cpp/openmp/0_dist/nbody_hermite4.cpp)              | Implementation using `distribute` clause, Unified Memory, Hermite method   | Some functions disabled for GPU execution |
| [cpp/openmp/1\_dist\_data/nbody\_hermite4.cpp](/cpp/openmp/1_dist_data/nbody_hermite4.cpp)   | Implementation using `distribute` clause, data directives, Hermite method  | Some functions disabled for GPU execution |
| [cpp/openmp/a\_loop/nbody\_hermite4.cpp](/cpp/openmp/a_loop/nbody_hermite4.cpp)              | Implementation using `loop` clause, Unified Memory, Hermite method         | Some functions disabled for GPU execution |
| [cpp/openmp/b\_loop\_data/nbody\_hermite4.cpp](/cpp/openmp/b_loop_data/nbody_hermite4.cpp)   | Implementation using `loop` clause, data directives, Hermite method        | Some functions disabled for GPU execution |

* Regarding functions disabled for GPU execution:

  * Some functions did not run correctly on the GPU, so they are temporarily executed on the CPU.

    * In the CUDA version, they run correctly on the GPU, so the cause is likely an implementation mistake or a compiler bug.
  * Macro `EXEC_SMALL_FUNC_ON_HOST` is enabled (some OpenMP directives are commented out).
  * Since these are small functions, the performance impact is expected to be minimal.
  * However, this results in additional CPU-GPU data transfers.

## Information on NVIDIA HPC SDK

### Compilation and Linking

* Typical compiler options:

  ```sh
  -mp=gpu -gpu=cc90 -Minfo=accel,opt,mp # Offload to GPU using OpenMP, optimized for cc90 (NVIDIA H100), outputs compiler messages on GPU offloading and optimizations
  -mp=gpu -gpu=cc90,mem:unified:nomanagedalloc -Minfo=accel,opt,mp # Recommended settings for Unified Memory on NVIDIA GH200
  -mp=gpu -gpu=cc90,managed -Minfo=accel,opt,mp # Settings for Unified Memory on non-GH200 environments (x86 CPU + NVIDIA GPU)
  ```

* Typical linker options:

  ```sh
  -mp=gpu      # Required, otherwise it won’t run on the GPU
  -gpu=mem:unified:nomanagedalloc # Add this when using Unified Memory (NVIDIA GH200)
  -gpu=managed # Add this when using Unified Memory (non-GH200: x86 CPU + NVIDIA GPU)
  ```

### Runtime

* Useful environment variables for debugging:

  ```sh
  NVCOMPILER_ACC_NOTIFY=1 ./a.out # Prints info every time a kernel runs on the GPU
  NVCOMPILER_ACC_NOTIFY=3 ./a.out # Also prints info on CPU-GPU data transfers
  NVCOMPILER_ACC_TIME=1 ./a.out   # Prints CPU-GPU transfer times and GPU execution time
  ```

---
