---
layout: post
title: "How to speed up your PyTorch training"
tags: pytorch, optimization, performance
crosspost_to_medium: true
---

If you're training PyTorch models, you want to train as fast as possible. This means you can try more things faster and get better results. Let's learn how to speedup your training!

This performance optimization guide is written with cloud-based, multi-machine multi-GPU training in mind, but can be useful even if all you have is a desktop with a single GPU under your bed.


Hardware factors
================

The training speed is fundamentally limited by three factors: CPU, GPU, and network.
- CPU speed. Each training process uses CPUs 1) to receive, parse, preprocess, and batch data examples; 2) to execute some NN operations not supported by GPU; and 3) to postprocess metrics, save checkpoints, and write summaries. In cloud, you can choose how many CPU cores your machine has.
- GPU speed. With DistributedDataParallel (DDP, current SotA for distributed training), we run one training PyTorch process per one GPU. Each training process uses GPU for most of the forward pass, backward pass, and weight update. In synchronized distributed training, GPUs communicate on each step to share gradients. In cloud, you can choose how many GPUs your machine has.
- Network speed. Each training process uses network for data downloading and GPU communication. The "main" process also uploads model checkpoints and summaries. If you're in Google Cloud and store your dataset on GCS, for example, each machine has about 10-100 Gbit/s ingress network which means it can download 1-10 GB/s from GCS. For comparison, my desktop can download from GCS at 100 MB/s. In cloud, you can scale total network bandwidth by adding more machines.

Note that each training process has its whole GPU to itself, but network and CPUs are shared equally between all training processes. So it's useful to think about CPU/GPU ratio or network/GPU ratio.


The typical training step
=========================

- Data loading (uses CPU + network)
    - Download example (bound by network speed, parallelizable per example)
    - Parse, preprocess, augment example (bound by CPU, parallelizable per example)
    - Build a batch (bound by CPU, parallelizable per batch)
- Memcpy batch to GPU (bound by PCI-e, parallelizable per GPU)
- Training (uses GPU + network, parallelizable per batch)
    - Forward pass, backward pass (bound by GPU)
    - Share gradients (bound by network)
    - Update weights (bound by GPU)
- Postprocessing (uses CPU)
- Save checkpoint and summaries (uses CPU)


Realizing the problem and ways to solve it
==========================================

How do you even know the training or dataloading is slow? You can measure the time spent by the training process getting a batch from the dataloader and compare it to the total time of the step. Ideally, all dataloading is offloaded to background processes, so getting the next batch is nearly instantaneous and only take a few percents of total time. Another way is to print, e.g. every 10 batches, how long did it take to process them. That way you can see if you have suspicious pauses, e.g. when a new epoch starts.

We have several ways of making training faster:
- we can *do things in parallel*
- we can *make individual things faster*
- we can *avoid doing some things*.


Parallelization
===============

Ideally, CPU, GPU, and network should be used by the training *simultaneously*, which means each of them should always have something to do. Also, a machine has multiple CPUs, multiple GPUs, and multiple network connections. We want to get to the state where:
- either all CPUs are 100% used
- or all GPUs are 100% used (this is the ultimate goal as in the cloud, GPUs is the most expensive resource of all)
- or the network is 100% used

If that's not the case, e.g. your monitoring charts show CPU < 100%, GPU < 100%, and network < 1 GB/s, it means there is not enough parallelization, i.e. there is some "artificial" software limit or lock preventing full utilization of hardware. For the cloud, this also means we can't scale vertically (by adding more CPUs/GPUs) but only horizontally (by adding more machines), and in fact, might need to downscale vertically if we can't fully use the resources anyway.

![without parallelization](https://www.tensorflow.org/guide/images/data_performance/naive.svg)

![with parallelization](https://www.tensorflow.org/guide/images/data_performance/parallel_interleave.svg)


How to see utilization on your desktop
--------------------------------------

- CPU: I like `htop` (`apt-get install -y htop`). It shows how much each core is utilized with horizontal bars. Press `Shift + H` to hide individual threads. Press `F5` to sort processes by CPU usage.
- GPU: I like `sudo nvtop` ([install](https://github.com/Syllo/nvtop#distribution-specific-installation-process)). It shows graphical history of GPU and memory utilization, as well as processes using GPU. Alternatively, `watch -n 1 nvidia-smi` is okay.
- Network: I like `sudo iftop` (`apt-get install -y iftop`). It shows the incoming/outgoing throughput per connection. The three columns are average bandwidth during the last 2, 10 and 40 seconds.
- To isolate dataloading from the other things, I like to use a separate "dataloading benchmark" script which just fetches batches as fast as possible without engaging GPU. This is to reduce the number of moving parts.


Notes on parallelization: data loading
--------------------------------------

- The main obstacle to dataloading parallelization is Python's GIL: only one stream of computation can progress within a Python process at any moment. The way around it is to have multiple Python processes, or fall to C++ code which is not bound by GIL.
- That's why each training process has its own GPU to feed with data. We shard the dataset into as many pieces as we have GPUs. The processes don't communicate except for sharing gradients, so GPUs can be kept busy in parallel.
    - Note: you can tune this by having more or fewer GPUs. Remember to use `torch.distributed.launch` to enable multi-GPU training, otherwise only one GPU will be used.
- With PyTorch `DataLoader`'s `num_workers > 0`, each training process offloads the dataloading to subprocesses. Each worker subprocess receives one batch worth of example indices, downloads them, preprocesses, and stacks the resulting tensors, and shares the resulting batch with the training process (by pickling into `/dev/shm`). The training process then only has to feed the GPU and postprocess. As a result, GPU computation and CPU computation overlap.
    - Worker subprocesses are also working in parallel, making use of CPU and network. At any moment a process uses either only CPU or only network, so it's even okay to have more processes than you have CPU cores.
    - You can scale this parallelization by increasing `num_workers`, but too many processes exhaust `/dev/shm` space which manifests as "bus error" and training crashes. Remember that if the machine has several GPUs, **each** training process will launch `num_workers` processes!
    - Worker processes are destroyed and recreated every epoch, although there is a way around that by [wrapping the `Sampler` around](https://github.com/pytorch/pytorch/issues/15849#issuecomment-573921048).
    - The workers need to assemble the whole batch, and they sequentially download and process the examples in the batch (sad!). Therefore, in the beginning of the epoch, when all worker processes start dataloading, you might observe a long pause while each of them is assembling their first batch. This pause goes away after the first batch, as long as the dataloading is not slower than training.
- Network requests to GCS are issued in parallel but parallelism is bounded by the underlying connection pool size (by default `10`). Increasing that did not achieve any speedup; likely because the library talking to GCS is in Python and therefore subject to GIL, and therefore unable to make use of more than 10 simultaneous connections.


Side note: Parquet format
-------------------------

- You might consider to use Parquet as your dataset format. Parquet is a columnar format, meaning the records are written on disk not `id1 | name1 | age1 | id2 | name2 | age2`, but rather `id1 | id2 | name1 | name2 | age1 | age2` (however, this holds true only within a "rowgroup" which is a group of several consecutive rows). This means columns of a record can be downloaded and parsed in parallel (unlike e.g. TFRecord format which is just one protobuf blob). You can read Parquet files with PyArrow library, which uses C++-based thread pool to read the columns in parallel. You can tune `OMP_NUM_THREADS` environment variable to increase the size of that thread pool, but too many threads can cause thrashing. Probably stick to default unless you know what you're doing. Then, wrap the resulting dataset in `DataLoader`.
- An alternative reader of Parquet is Petastorm, which in my opinion has made less optimal architectural choices compared to `DataLoader`. Petastorm also has a pool of workers which receive tasks (rowgroup ids), download rowgroups, and perform filtering and transforms (but not batching). There are two implementations of worker pool in Petastorm: thread pool and process pool. You can tune this with `reader_pool_type=` `"thread"` or `"process"`.
    - Thread pool is subject to GIL so there's no CPU parallelization, and all computation happens in the same process that feeds the GPU, but I think network requests are parallelized (Python yields GIL when performing I/O).
    - Process pool parallelizes both CPU and network usage. The worker processes send records back to the main process via ZeroMQ. This means records must be serialized, and the main process needs to spend time deserializing. The fastest way to do that is PyArrow serialization, but it can only serialize certain data types. The alternative is `pickle` serialization which understands all data types but is slower.
    - Also, with Petastorm, the main training process is assembling the batches, which takes some CPU time and can leave GPU underutilized.


Notes on parallelization: memcpy
--------------------------------

- There is a region in RAM called "pinned memory" which is the waiting area for tensors before they can be placed on GPU. For faster CPU-to-GPU transfer, we can copy tensors in the pinned memory region in the background thread, before GPU asks for the next batch. This is available with `pin_memory=True` argument to PyTorch `DataLoader`.
    - Documentation on pinned memory: [pytorch](https://pytorch.org/docs/stable/data.html#memory-pinning), [nvidia](https://devblogs.nvidia.com/how-optimize-data-transfers-cuda-cc/).
- If tensors are placed in pinned memory, it's also possible to copy them to GPU asynchronously, so that CPU is not blocked by the copying operation. This is useful if for some reason you have CPU-based computation between memcpy and kickstarting forward pass. Enable asynchronous memcpy-to-GPU with `your_tensor.to(device, non_blocking=True)`.
    - Documentation on async transfer: [pytorch](https://pytorch.org/docs/stable/notes/cuda.html#use-pinned-memory-buffers), [nvidia](https://devblogs.nvidia.com/how-overlap-data-transfers-cuda-cc/).


Notes on parallelization: training
----------------------------------

- Typical models are composed of sequential layers, so there is not a lot of opportunity to overlap GPU operations. It's theoretically possible to launch kernels in parallel on the same GPU, but in different CUDA streams, which would benefit graph-like models [and maybe even sequential models](https://discuss.pytorch.org/t/how-can-l-run-two-blocks-in-parallel/61618/5) - this seems to be an ongoing effort in PyTorch.
- However, Nvidia Apex library provides optimized [`DistributedDataParallel`](https://nvidia.github.io/apex/parallel.html#apex.parallel.DistributedDataParallel) which overlaps communication (gradient sharing) and backward pass.
- Increasing batch size [doesn't increase parallelism linearly](https://www.pugetsystems.com/labs/hpc/GPU-Memory-Size-and-Deep-Learning-Performance-batch-size-12GB-vs-32GB----1080Ti-vs-Titan-V-vs-GV100-1146/), however there is a fixed cost of memcpy and gradient sharing per batch, so it still makes sense to make the batch as large as possible.
- In case of model parallelism, it's possible to [automatically parallelize even sequential models](https://torchgpipe.readthedocs.io/en/stable/gpipe.html#pipeline-parallelism).


Making things faster and avoiding doing things
==============================================

Suppose you see either CPU, GPU, or network 100% (or almost 100%) used. That saturated resource is the *bottleneck* i.e. the slowest part for which other resources have to wait. Optimizing the slowest part will speed up the whole thing. Optimizing other parts will have no effect. You can use these tips even if the resource is not 100% used, and then save money by downscaling vertically (reducing number of CPUs/GPUs) without losing training speed.

Making CPU computation faster
-----------------------------
- To figure out what is so slow in the computation, you need to profile.
    - I like `py-spy` sampling profiler. It can connect to a running Python process, profile *both Python and native code*, and produce a ["flamegraph" visualization](http://www.brendangregg.com/flamegraphs.html) where the widest part is the slowest method.
    - `pip install py-spy` if it's not yet installed.
    - Figure out the Python process ID to profile. For the training process, check `nvtop` to see which process is using GPU. For the dataloading worker process, pick any of them in `htop`.
    - Do `py-spy record -r 29 -o profile.svg -p <PID> --native`. Wait for 5-10 seconds and `Ctrl+C`. The resulting `profile.svg` image is your profile, open it in your browser.
    - Caveats:
        - Worker processes are killed and recreated every epoch, so start profiling in the beginning of the epoch.
        - `-r` argument is how many samples to take per second. If it's too high, the sampling overhead slows the target Python process down.
        - `--native` argument will profile even C++ code! But if the profiler just crashes, try to remove `--native`.
        - `--subprocesses` will create a combined profile for the main process and all the worker subprocesses, but it's harder to interpret.
- In Parquet, record fields (columns) are written separately from each other, so it's possible to avoid reading columns you don't use for training. Petastorm supports that out of the box.
- Local caching avoids downloading+parsing+preprocessing the same training example twice. The preprocessed example can be saved to disk and loaded from there during the next epochs. Petastorm supports that out of the box. Random-based augmentation still needs to be done outside of caching!

![caching](https://www.tensorflow.org/guide/images/data_performance/cached_dataset.svg)

- Some options to speed up slow Python code:
    - add [`@numba.jit`](https://numba.pydata.org/numba-doc/dev/user/5minguide.html) decorator to a slow method (even with `numpy`) for automatic conversion to machine code (there's more to Numba than that, e.g. parallelization and vectorization)
    - memoize computation + parallelize a for-loop with [`joblib`](https://joblib.readthedocs.io/en/latest/)
    - manually vectorize code with `numpy` operations instead of doing for-loops. Make sure your `numpy` is using `MKL`!
    - parallelize a for-loop with `torch.multiprocessing` (same as `multiprocessing`, but uses `/dev/shm` for communication)
    - use Cython to convert your code into optimized C++ and make it an extension
    - see if there are different implementations of the same logic. `OpenCV` can be faster than `PIL`, etc.
- Some CPU computation (e.g. image preprocessing) can be done much faster on GPU instead! We're working on enabling that with [Nvidia DALI library](https://docs.nvidia.com/deeplearning/sdk/dali-developer-guide/docs/).
- You can override the collation function for a faster one, e.g. consider [`fast_collate` from NVIDIA](https://github.com/NVIDIA/apex/blob/2ca894da7be755711cbbdf56c74bb7904bfd8417/examples/imagenet/main_amp.py#L28).

- Within your training loop, be careful to not use loss variable outside of the loop or use it in such a manner that a reference will be held. If you do that, a separate copy of your model graph will be held in memory, preventing you from using large batch size. [See this for more details](https://pytorch.org/docs/stable/notes/faq.html).


Making network faster
---------------------

- GCS API supports GET-requests that ask only for a range of bytes, but some libraries (`gcsfs` without configuration) inflate that range to 5MB for caching purposes. The Parquet files don't benefit from such a large readahead cache (most columns are super small), and this results in unnecessarily huge incoming traffic and a significant slowdown. Check that global variable `gcsfs.core.GCSFileSystem.default_block_size` is `1024` or less!
- Opening and closing connections is pretty expensive. If you need to download a lot of files from GCS, make sure you're reusing connections.
- Again, caching avoids repeated downloads of the same file. Caching could theoretically be done on filesystem level to persist raw bytes (instead of parsed+preprocessed records).


Making GPU computation faster
-----------------------------

If you maxed out the GPUs, congratulations! You're using the hardware efficiently. Still a few optimization opportunities:

- Use `torch.autograd.profiler` to see how long do the GPU operations take. By default, you would only see the CPU's point of view; with `use_cuda=True` and `num_workers=0`, you will see exact GPU kernel timings.
    - The resulting profiles are JSON files. Open them in [`chrome://tracing`](chrome://tracing) timeline viewer.
    - You might see some kernels, unsupported by GPU, executing on CPU. This is sometimes inefficient because GPU has to send the inputs to CPU and wait for the results to come back. Maybe you can replace it with GPU-supported operations?
    - You might also see kernels that are inefficient on GPU and take super long time to execute. You can try placing them on CPU and get speedup even with the overhead of back-and-forth copying!

![torch_profiler](https://pytorch.org/tutorials/_images/trace_img.png)

- Try setting `torch.backends.cudnn.benchmark = True` in the beginning of training. This chooses [optimal convolution algorithm if the input shapes don't change](https://discuss.pytorch.org/t/what-does-torch-backends-cudnn-benchmark-do/5936).
- "Mixed precision" mode (where some operations work with fp16 inputs but accumulate in fp32) can be [several times faster](https://docs.nvidia.com/deeplearning/performance/mixed-precision-training/index.html#mptrain) than fully fp32 operations, and also allow you to use 2x larger batch size.
    - Try manually converting your inputs to fp16.
    - [Nvidia Apex library](https://github.com/NVIDIA/apex) can automatically search for opportunities to enable mixed precision (AMP). [imagenet example](https://github.com/NVIDIA/apex/blob/master/examples/imagenet/main_amp.py#L157). PyTorch also has some [native AMP support](https://github.com/pytorch/pytorch/issues/25081) (`torch.cuda.amp` exists since Feb 2020).
- Try reordering tensor dimensions. GPUs like NCHW, but "tensor cores" [like NHWC](https://devblogs.nvidia.com/tensor-core-ai-performance-milestones/) and [sizes divisible by 8](https://docs.nvidia.com/deeplearning/sdk/pdf/Deep-Learning-Performance-Guide.pdf).
- In cloud, you can simply replace a GPU with a more powerful one. Be aware that it can be several times more expensive, but not several times faster!


Closing thoughts
================

PyTorch is a powerful tool under active development, and it's useful to learn how it works. Don't be afraid to profile, experiment, and read the source code!
