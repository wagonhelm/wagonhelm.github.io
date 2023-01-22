---
layout: post
title: Detecting Cuda Architecture required by CMake using NVCC
excerpt: Documentation on how to detect cuda compute capability from within a CMake script.
categories: [Documentation]
tags: [Cuda, CMake, NVCC, Ubuntu 16.04]
comments: true
---

Recently on the [ARVP](http://arvp.org) team I was assigned with the task of building a cmake script that could detect if Cuda 
is installed and return the compute capability of the graphic card on the computer.  Our current CMake script 
was designed just to work on our Jetson TX2 with a compute capability of 6.1 and we needed our script to work with 
all computers GPU or no GPU.

Luckily CMake has the module [FindCUDA](https://cmake.org/cmake/help/v3.5/module/FindCUDA.html) which offers a lot of help
when trying to detect cuda. The latest versions of CMake have built in macros for detecting the graphic card architecture 
but unfortunately Ubuntu 16.04's default version of CMake (3.5.1) does not support these macros.  This is how we ended up
detecting the Cuda architecture in CMake.

To begin with you need to make a Cuda script to detect the GPU, find the compute capability, and make sure the compute 
capability is greater or equal to the minimum required.  In most general cases a minimum of 3.0 is required.  Here is 
the Cuda script which you can save as `check_cuda.cu`

```cpp
#include <stdio.h>

int main(int argc, char **argv){
    cudaDeviceProp dP;
    float min_cc = 3.0;

    int rc = cudaGetDeviceProperties(&dP, 0);
    if(rc != cudaSuccess) {
        cudaError_t error = cudaGetLastError();
        printf("CUDA error: %s", cudaGetErrorString(error));
        return rc; /* Failure */
    }
    if((dP.major+(dP.minor/10)) < min_cc) {
        printf("Min Compute Capability of %2.1f required:  %d.%d found\n Not Building CUDA Code", min_cc, dP.major, dP.minor);
        return 1; /* Failure */
    } else {
        printf("-arch=sm_%d%d", dP.major, dP.minor);
        return 0; /* Success */
    }
}
```
With this script you can set the minimum compute capability, return an error code if the graphic card is not detected
correctly and determine if the graphic card meets the minimum requirements.  If everything succeeds it will return the 
architecture flag required by NVCC.  You can then add the following to your cmake script that requires the NVCC flag for 
the compute architecture.  The RESULT_VARIABLE will be the value returned from the .cu script (0 if success, 1 if min. 
requirements are not met, or an error code if failure) and OUTPUT_VARIABLE will be the string printed in the .cu script.

```cmake
# Find CUDA
find_package(CUDA)

if (CUDA_FOUND)
  #Get CUDA compute capability
  set(OUTPUTFILE ${CMAKE_CURRENT_SOURCE_DIR}/DIR/TO/OUTPUT/FILE/cuda_script) # No suffix required
  set(CUDAFILE ${CMAKE_CURRENT_SOURCE_DIR}/DIR/TO/CUDA/FILE/check_cuda.cu)
  execute_process(COMMAND nvcc -lcuda ${CUDAFILE} -o ${OUTPUTFILE})
  execute_process(COMMAND ${OUTPUTFILE}
                  RESULT_VARIABLE CUDA_RETURN_CODE
                  OUTPUT_VARIABLE ARCH)

  if(${CUDA_RETURN_CODE} EQUAL 0)
    set(CUDA_SUCCESS "TRUE")
  else()
    set(CUDA_SUCCESS "FALSE")
  endif()

  if (${CUDA_SUCCESS})
    message(STATUS "CUDA Architecture: ${ARCH}")
    message(STATUS "CUDA Version: ${CUDA_VERSION_STRING}")
    message(STATUS "CUDA Path: ${CUDA_TOOLKIT_ROOT_DIR}")
    message(STATUS "CUDA Libararies: ${CUDA_LIBRARIES}")
    message(STATUS "CUDA Performance Primitives: ${CUDA_npp_LIBRARY}")

    set(CUDA_NVCC_FLAGS "${ARCH}")
    add_definitions(-DGPU) #You may not require this

  else()
    message(WARNING ${ARCH})
  endif()
endif()

```

And there you have it, you can now detect the Cuda compute capability using Cmake and set the NVCC flags for building
Cuda related software.
