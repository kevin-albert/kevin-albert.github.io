---
layout:     post
title:      "CUDA Programming"
date:       2014-08-29 12:57:00
summary:    "Introduction to CUDA using parallel mergesort, including code samples"
categories: things stuff
---

I’ve been messing with [CUDA](http://www.nvidia.com/object/cuda_home_new.html "CUDA") recently. If you don’t know, CUDA is the programming toolchain designed for using NVIDIA graphics cards for parallel computation. You write host and device code in C++, and use provided functions for moving memory around and invoking GPU kernels.  

To get started, I first downloaded their SDK and compiled some of their samples. Then wrote some completely trivial programs and got them to compile. Next, I decided to try and use CUDA to speed up an actual algorithm, so I chose sorting.

## Algorithm

I chose to try and parallelize a simple bottom-up mergesort. The implementation is simple:

1. Start with two lists: Your input array, and a temp array that’s the same size.
2. Define a width, starting at 2. During each step, width gets multiplied by 2.
3. While width < 2N, sort each width-sized chunk of the list into the temp list. Then switch the pointers of the two lists. This means you can avoid unnecessary allocating / copying.
  * This is the step that happens in parallel. Each thread gets a chunk of the list to sort.
  * Two halves of each chunk are are sorted / merged against each other into the temp array.
4. You end up with one big chunk being sorted into the final list, and you switch input and temp one last time, returning temp

Here’s what it looks like with 4 threads sorting 16 numbers:

```
|                   Input                   |
|-------------------------------------------|
|      8 3 1 9 1 2 7 5 9 3 6 4 2 0 2 5      |
|                                           |
|  Thread1 | Thread2  | Thread3  | Thread4  |
|----------|----------|----------|----------|
| 8 3 1 9  | 1 2 7 5  | 9 3 6 4  | 2 0 2 5  |
|  38 19   |  12 57   |  39 46   |  02 25   |
|---------------------|---------------------|
|       Thread1       |        Thread2      |
|   1398       1257   |   3469       0225   |
|       11235789      |       02234569      |
|-------------------------------------------|
|                  Thread1                  |
|      0 1 1 2 2 2 3 3 4 5 5 6 7 8 9 9      |
|                                           |
```

## Implementation
You can find the source code for my implementation here: [https://github.com/kevin-albert/cuda-mergesort](https://github.com/kevin-albert/cuda-mergesort "CUDA Mergesort").  

The three main functions used in the sort are:  

mergesort kernel (run on the GPU):
```c
__global__ void gpu_mergesort(long* source, long* dest, long size,
                              long width, long slices,
                              dim3* threads, dim3* blocks) {
    unsigned int idx = getIdx(threads, blocks);
    long start = width*idx*slices,
         middle,
         end;
 
    for (long slice = 0; slice < slices; slice++) { 
        if (start >= size)
            break;
 
        middle = min(start + (width >> 1), size);
        end = min(start + width, size);
        gpu_bottomUpMerge(source, dest, start, middle, end);
        start += width;
    }
}
```

gpu_bottomUpMerge (helper kernel):
```c
__device__ void gpu_bottomUpMerge(long* source, long* dest,
                                  long start, long middle,
                                  long end) {
    long i = start;
    long j = middle;
    for (long k = start; k < end; k++) {
        if (i < middle && (j >= end || source[i] < source[j])) {
            dest[k] = source[i];
            i++;
        } else {
            dest[k] = source[j];
            j++;
        }
    }
}
```

the loop that calls the mergesort kernel (run on the CPU):
```c++
for (int width = 2; width < (size << 1); width <<= 1) {
    long slices = size / ((nThreads) * width) + 1;
 
    gpu_mergesort<<<blocksPerGrid, threadsPerBlock>>>
        (A, B, size, width, slices, D_threads, D_blocks);
 
    // Switch the input / output arrays
    A = A == D_data ? D_swp : D_data;
    B = B == D_data ? D_swp : D_data;
}
```

To run, you need the CUDA SDK, add their sample library (I include their `helper_cuda.h` to use the macro `checkCudaErrors()`)

## Actual Performance

Sorting 1 million integers (on my MacBook)

**Built-in sort**
```sh
# takes 2.738 seconds
$ sort -n out.txt 
```

**My CUDA mergesort, with 256 threads**
```sh
# takes 3.871 seconds
$ ./mergesort -v out.txt
```

While I didn’t really expect to beat the system sort, I was curious to see how long the algorithm ran:

| Event                         | Running Time      |
|:------------------------------|:------------------|
| parse argv                    | 16 microseconds   |
| read stdin                    | 1.28 seconds      |
| cudaMalloc device lists       | 122 milliseconds  |
| cudaMemcpy list to device     | 4.47 milliseconds |
| cudaMalloc metadata           | 122 microseconds  |
| cudaMemcpy metadata to device | 1.15 milliseconds |
| call mergesort kernel         | 39 microseconds   |
| call mergesort kernel         | 5 microseconds    |
| call mergesort kernel         | 5 microseconds    |
| call mergesort kernel         | 4 microseconds    |
| call mergesort kernel         | 4 microseconds    |
| call mergesort kernel         | 6 microseconds    |
| call mergesort kernel         | 4 microseconds    |
| call mergesort kernel         | 4 microseconds    |
| call mergesort kernel         | 4 microseconds    |
| call mergesort kernel         | 4 microseconds    |
| call mergesort kernel         | 4 microseconds    |
| call mergesort kernel         | 5 microseconds    |
| call mergesort kernel         | 4 microseconds    |
| call mergesort kernel         | 3 microseconds    |
| call mergesort kernel         | 4 microseconds    |
| call mergesort kernel         | 4 microseconds    |
| call mergesort kernel         | 6 microseconds    |
| call mergesort kernel         | 4 microseconds    |
| call mergesort kernel         | 3 microseconds    |
| call mergesort kernel         | 3 microseconds    |
| cudaMemcpy list back to host  | 2.263 seconds     |
| cudaFree                      | 6.834 milliseconds|
| print to stdout               | 320 milliseconds  |

119 microseconds spent sorting - not too bad. An appropriate follow-up would compare with a similar CPU implementation.  

## Notes
* Don’t use too many threads or your screen will freeze 

