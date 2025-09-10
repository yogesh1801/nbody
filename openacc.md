````markdown
# Overview of OpenACC Implementation

An overview of implementing the direct $N$-body simulation code using OpenACC.

## Implementation Method

* Add directives to the `for` loops that should run on the GPU  
  (If `kernels` does not trigger GPU execution, consider using `parallel`)

   ```c++
   #pragma acc kernels
   #pragma acc loop independent
   for (int32_t ii = 0; ii < num; ii++) {
     ...
   }
````

* Option: Specify the number of threads per block
  (If `kernels` does not trigger GPU execution, consider using `parallel`)

  ```c++
  #pragma acc kernels vector_length(256)
  #pragma acc loop independent
  for (int32_t ii = 0; ii < num; ii++) {
    ...
  }
  ```

* The number of threads per block can also be indicated with the following syntax:

  ```c++
  #pragma acc kernels
  #pragma acc loop independent vector(256)
  for (int32_t ii = 0; ii < num; ii++) {
    ...
  }
  ```

* If **Unified Memory** is not used, add data directives:

  1. Allocate memory on the GPU

     ```c++
     #pragma acc enter data create(ptr[0:num])
     ```

  2. Transfer data from CPU to GPU

     ```c++
     #pragma acc update device(ptr[0:num])
     ```

  3. Transfer data from GPU to CPU

     ```c++
     #pragma acc update host(ptr[0:num])
     ```

  4. Free GPU memory

     ```c++
     #pragma acc exit data delete(ptr[0:num])
     ```

## Implementation Examples

| Source Code                                                                               | Implementation Overview                       | Notes                                     |
| ----------------------------------------------------------------------------------------- | --------------------------------------------- | ----------------------------------------- |
| [cpp/openacc/0\_unified/nbody\_leapfrog2.cpp](/cpp/openacc/0_unified/nbody_leapfrog2.cpp) | Leapfrog method using Unified Memory          |                                           |
| [cpp/openacc/1\_data/nbody\_leapfrog2.cpp](/cpp/openacc/1_data/nbody_leapfrog2.cpp)       | Leapfrog method with explicit data directives |                                           |
| [cpp/openacc/0\_unified/nbody\_hermite4.cpp](/cpp/openacc/0_unified/nbody_hermite4.cpp)   | Hermite method using Unified Memory           | GPU execution disabled for some functions |
| [cpp/openacc/1\_data/nbody\_hermite4.cpp](/cpp/openacc/1_data/nbody_hermite4.cpp)         | Hermite method with explicit data directives  | GPU execution disabled for some functions |

* About partial GPU execution:

  * Some functions failed to work correctly on the GPU, so they were temporarily set to run on the CPU.

    * In CUDA implementations, these functions run correctly on the GPU, suggesting possible implementation mistakes or compiler bugs.
  * The macro `EXEC_SMALL_FUNC_ON_HOST` is enabled (i.e., certain OpenACC directives are commented out).
  * Since these functions are small, their impact on execution time is expected to be minimal.
  * However, this results in extra CPU–GPU data transfers.

## NVIDIA HPC SDK Information

### Compilation & Linking

* Typical compile-time options:

  ```sh
  -acc=gpu -gpu=cc90 -Minfo=accel,opt 
  # Enable OpenACC GPU offloading, optimize for cc90 (NVIDIA H100), 
  # and display compiler messages about offloading/optimizations.

  -acc=gpu -gpu=cc90,mem:unified:nomanagedalloc -Minfo=accel,opt 
  # Recommended setting for Unified Memory on NVIDIA GH200.

  -acc=gpu -gpu=cc90,managed -Minfo=accel,opt 
  # Setting for Unified Memory on non-GH200 systems (x86 CPU + NVIDIA GPU).
  ```

* Typical link-time options:

  ```sh
  -acc=gpu     # Required; without it, the program will not run on the GPU.
  -gpu=mem:unified:nomanagedalloc # Required for Unified Memory (NVIDIA GH200).
  -gpu=managed # Required for Unified Memory (non-GH200: x86 CPU + NVIDIA GPU).
  ```

### Runtime

* Useful environment variables for debugging:

  ```sh
  NVCOMPILER_ACC_NOTIFY=1 ./a.out # Print info each time a kernel is executed on the GPU.
  NVCOMPILER_ACC_NOTIFY=3 ./a.out # Also print CPU–GPU data transfer info.
  NVCOMPILER_ACC_TIME=1 ./a.out   # Print execution time and transfer time.
  ```

```
```
