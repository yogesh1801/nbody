# Overview of Solomon Implementation

Introduction to the implementation of an \$N\$-body simulation code (direct method) using [Solomon](https://github.com/ymiki-repo/solomon).

## Overview of Solomon

* Solomon (Simple Off-LOading Macros Orchestrating multiple Notations) is a macro library that unifies the interfaces of GPU directives such as OpenACC and OpenMP target.

  * It enables easy implementation of code that supports both [OpenACC](openacc.md) and [OpenMP target](openmp.md).
  * For an overview of the implementation, see the [Japanese README](https://github.com/ymiki-repo/solomon/blob/main/README_jp.md) in the [Solomon repository](https://github.com/ymiki-repo/solomon).
  * For detailed implementation information, refer to [Miki & Hanawa (2024, IEEE Access, vol. 12, pp. 181644–181665)](https://doi.org/10.1109/ACCESS.2024.3509380).

    * When citing work that uses Solomon, please reference [Miki & Hanawa (2024, IEEE Access, vol. 12, pp. 181644–181665)](https://doi.org/10.1109/ACCESS.2024.3509380).

## Implementation Method

* Include the Solomon header file:

  ```c++
  #include <solomon.hpp>
  ```

* Add directives to the `for` loops you want to execute on the GPU:

  ```c++
  OFFLOAD(AS_INDEPENDENT)
  for(int32_t ii = 0; ii < num; ii++){
    ...
  }
  ```

  * Option: Suggest the number of threads per thread block:

    ```c++
    OFFLOAD(AS_INDEPENDENT, NUM_THREADS(256))
    for(int32_t ii = 0; ii < num; ii++){
      ...
    }
    ```

* If **Unified Memory** is not used, add data directives:

  1. Allocate memory on the GPU:

     ```c++
     MALLOC_ON_DEVICE(ptr[0:num])
     ```

  2. Transfer data from CPU to GPU:

     ```c++
     MEMCPY_H2D(ptr[0:num])
     ```

  3. Transfer data from GPU to CPU:

     ```c++
     MEMCPY_D2H(ptr[0:num])
     ```

  4. Free memory on the GPU:

     ```c++
     FREE_FROM_DEVICE(ptr[0:num])
     ```

## Implementation Examples

| Source Code                                                          | Implementation Overview                               | Notes                                     |
| -------------------------------------------------------------------- | ----------------------------------------------------- | ----------------------------------------- |
| [cpp/solomon/nbody\_leapfrog2.cpp](/cpp/solomon/nbody_leapfrog2.cpp) | Implementation using data directives, Leapfrog method |                                           |
| [cpp/solomon/nbody\_hermite4.cpp](/cpp/solomon/nbody_hermite4.cpp)   | Implementation using data directives, Hermite method  | Some functions disabled for GPU execution |

* Regarding functions disabled for GPU execution:

  * Some functions did not run correctly on the GPU, so they are temporarily executed on the CPU.

    * In the CUDA version, they run correctly on the GPU, so the cause is likely an implementation error or a compiler bug.
  * The macro `EXEC_SMALL_FUNC_ON_HOST` is enabled (some OpenACC directives are commented out).
  * Since these functions are small, the impact on execution time is expected to be minimal.
  * However, this does result in additional CPU–GPU data transfers.

## Compilation Method

* Use compiler options to enable either [OpenACC](openacc.md) or [OpenMP target](openmp.md).
* Specify the path to Solomon (the directory containing `solomon.hpp`) using an option such as `-I/path/to/solomon`.
* Use the following compile-time flags to specify Solomon’s execution mode:

  | Compile Flag                                                       | Backend Used    | Notes                                              |
  | ------------------------------------------------------------------ | --------------- | -------------------------------------------------- |
  | `-DOFFLOAD_BY_OPENACC`                                             | OpenACC         | Uses the `kernels` construct by default            |
  | `-DOFFLOAD_BY_OPENACC -DOFFLOAD_BY_OPENACC_PARALLEL`               | OpenACC         | Uses the `parallel` construct by default           |
  | `-DOFFLOAD_BY_OPENMP_TARGET`                                       | OpenMP target   | Uses the `loop` directive by default               |
  | `-DOFFLOAD_BY_OPENMP_TARGET -DOFFLOAD_BY_OPENMP_TARGET_DISTRIBUTE` | OpenMP target   | Uses the `distribute` directive by default         |
  | *(none)*                                                           | Degenerate mode | Thread parallelism for multicore CPUs using OpenMP |

  * Note: Even if you pass `-DOFFLOAD_BY_OPENACC` as a compile-time flag, if you do not also pass the compiler flag to enable OpenACC (e.g., `-acc` in NVIDIA HPC SDK), `-DOFFLOAD_BY_OPENACC` will automatically be disabled.
