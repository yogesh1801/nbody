# Overview of Implementation Using the C++17 Standard Language Specification

Introduction to the implementation of an \$N\$-body simulation code (direct method) using the C++17 standard language specification.

## Implementation Method

* Include **algorithm** and **execution** headers:

  ```c++
  #include <algorithm>
  #include <execution>
  ```

* Replace `for` loops to be executed on the GPU with `std::for_each_n`:

  ```c++
  std::for_each_n(std::execution::par, boost::iterators::counting_iterator<int32_t>(0), num, [=](const int32_t ii){
    ...
  });
  ```

  * There are other implementation methods, but for reasonably long loops, the above is the shortest approach.

    * For algorithms where a parallel version is provided, it is enough to simply add `std::execution::par`.

      ```c++
      std::sort(std::execution::par, begin(), end());
      ```

  * Implementation of `counting_iterator`:

    * Possible approaches:

      * Implement your own
      * Use a library such as **thrust**
    * Here, Boost C++ is used:

      * Library usage was chosen to minimize workload
      * Thrust was excluded to maintain portability (to allow compilation with compilers other than NVIDIA HPC SDK).

## Implementation Examples

| Source Code                                                        | Implementation Overview | Notes                                     |
| ------------------------------------------------------------------ | ----------------------- | ----------------------------------------- |
| [cpp/stdpar/nbody\_leapfrog2.cpp](/cpp/stdpar/nbody_leapfrog2.cpp) | Leapfrog method         |                                           |
| [cpp/stdpar/nbody\_hermite4.cpp](/cpp/stdpar/nbody_hermite4.cpp)   | Hermite method          | Some functions disabled for GPU execution |

* Regarding functions disabled for GPU execution:

  * Some functions did not work correctly when executed on the GPU, so they are temporarily executed on the CPU.

    * In the CUDA version, they work correctly on the GPU, so the cause is assumed to be either an implementation error or a compiler bug.
  * The macro `EXEC_SMALL_FUNC_ON_HOST` is enabled (i.e., for some parts `std::execution::seq` is specified instead of `std::execution::par` to suppress parallelization).
  * Since these are small functions, the impact on execution time is considered minimal.
  * However, this results in additional CPU–GPU data transfers.

## Information on NVIDIA HPC SDK

### Compilation and Linking

* Typical compiler options (compilation phase):

  ```sh
  -stdpar=gpu -gpu=cc90 -Minfo=accel,opt,stdpar # Use standard language parallelism for GPU offloading, optimized for cc90 (NVIDIA H100), output compiler messages on GPU offloading and optimizations
  -stdpar=multicore -Minfo=opt,stdpar           # Use standard language parallelism for multicore CPU execution, output compiler messages on optimizations
  ```

* Typical linker options (linking phase):

  ```sh
  -stdpar=gpu # When GPU offloading is enabled
  ```
