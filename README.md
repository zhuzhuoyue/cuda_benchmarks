
# Cuda benchmarks for unified vs. explicit memory


## Unified memory

Unified memory has been a feature of game consoles for many years. It simplifies game development because it frees the programmer from having to track whether a memory block is on CPU or GPU memory.


Starting with the Pascal architecture, Nvidia also offers [advanced unified memory support](https://devblogs.nvidia.com/unified-memory-cuda-beginners/).


Game consoles can have tighter hardware control, but Nvidia also has the [Jetson product line](https://en.wikipedia.org/wiki/Nvidia_Jetson) with physically unified memory that has been [reported](https://devtalk.nvidia.com/default/topic/1029853/does-unified-memory-and-zero-copy-always-better-than-cudamemcpy-/) to have better performance than explicit memory management.


## Unified memory reputation

There have been several reports, many of them from around 2014, that unified memory has a lower performance in many scenarios. The reports usually do not come with links to actual code.

Recent reports appear to address only specialty scenarios like [8 NVlink GPUs](https://devtalk.nvidia.com/default/topic/1029706/cuda-programming-and-performance/partial-fail-of-peer-access-in-8-volta-gpu-instance-p3-16xlarge-on-aws-gt-huge-slowdown-/).

That report also has no code link.



## Unified memory reality

This benchmark suite attempts to provide actual code so that people can check for themselves. It is incomplete and does not yet address scenarios like IPC or multiple GPUs. Benchmarks that show the superiority of explicit memory management are welcome.


## Examples

The examples were run on Linux with Cuda release 9.2, V9.2.148, using 16GB of host memory and a GeForce 1060 with 6GB of GPU memory.

### simpleManaged vs. simpleMemcpy vs. simpleDMA

This benchmark tests initializing three arrays in host memory, running a kernel and accessing the result. Most of the time is spent in copying, the kernel runtime is negligible.

With N=200000000, explicit memory performs slightly better, but DMA is faster than both:


```
$ ./simpleManaged 200000000
host: MallocManaged: 0.000040
host: init arrays: 0.662895
device: uvm+compute+synchronize: 0.892010
host: access all arrays: 0.929058
host: access all arrays a second time: 0.245681
host: free: 0.176788
total: 2.906544

$ ./simpleMemcpy 200000000
host: MallocHost: 0.706024
host: init arrays: 0.259399
device: malloc+copy+compute: 0.420570
host: access all arrays: 0.239900
host: access all arrays a second time: 0.239795
host: free: 0.350564
total: 2.216320

$ ./simpleDMA 200000000
host: MallocHost: 0.700510
host: init arrays: 0.260276
device: DMA+compute+synchronize: 0.266353
host: access all arrays: 0.241061
host: access all arrays a second time: 0.240792
host: free: 0.349305
total: 2.058358
```

With N=500000000, managed memory has no issues, but explicit memory does not run at all. DMA again performs very well:


```
$ ./simpleManaged 500000000
host: MallocManaged: 0.000043
host: init arrays: 1.632873
device: uvm+compute+synchronize: 2.235518
host: access all arrays: 1.640106
host: access all arrays a second time: 0.607754
host: free: 0.382087
total: 6.498456

$ ./simpleMemcpy 500000000
host: MallocHost: 1.751784
host: init arrays: 0.674096
cudaErrorMemoryAllocation

$ ./simpleDMA 500000000
host: MallocHost: 1.750448
host: init arrays: 0.673640
device: DMA+compute+synchronize: 0.665088
host: access all arrays: 0.607256
host: access all arrays a second time: 0.607619
host: free: 0.882589
total: 5.186704
```

### cuBLAS: gemmManaged vs. gemmMemcpy

This benchmark calls the cublasSgemm() function.

With N=8000, managed memory is considerably faster:

```
$ ./gemmManaged 8000
host: MallocManaged+init: 0.191981
cublasSgemm: 1.430046
host: access all arrays: 0.000080
host: access all arrays a second time: 0.000008
host: free: 0.030062
total: 1.967801

$ ./gemmMemcpy 8000
host: MallocHost+init: 0.236840
cublasSgemm: 3.316726
host: access all arrays: 0.000030
host: access all arrays a second time: 0.000008
host: free: 0.061765
total: 3.928581
```

With N=16000, managed memory is not only considerably faster, but explicit memory performance is catastrophic:

```
$ ./gemmManaged 16000
host: MallocManaged+init: 0.761249
cublasSgemm: 3.317761
host: access all arrays: 0.000105
host: access all arrays a second time: 0.000045
host: free: 0.084146
total: 4.477609

$ ./gemmMemcpy 16000
host: MallocHost+init: 0.940572
cublasSgemm: 35.439908
host: access all arrays: 0.000038
host: access all arrays a second time: 0.000017
host: free: 0.232385
total: 36.929403
```


### cuBlas+Managed vs. cuBlasXt+HostMemory

cuBlasXt handles out-of-core computations for memory that is allocated with cudaMallocHost(). This benchmark compares the cublasSgemm() function running on managed memory vs. the cublasXtSgemm() function running on host allocated memory.

Note that cublasXtSgemm() is designed to run on host allocated memory and handles optimized tiled memory transfers. Also note that cuBlasXt has more functionality (multiple cards), so the slightly worse performance is not surprising.

The point of this comparison, however, is that managed memory performs very well using the standard cuBlas function.

```
$ ./gemmManagedOutOfCore 32000
host: MallocManaged+init: 3.059273
cublasSgemm: 20.510228

$ ./gemmXtOutOfCore 32000
host: MallocHost+init: 3.766991
cublasXtSgemm: 25.617316
```
