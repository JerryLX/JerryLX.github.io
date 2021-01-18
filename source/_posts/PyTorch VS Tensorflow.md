---
title: PyTorch V.S. Tensorflow
date: 2021-01-16 21:06:00
categories:
- something
tags: 
- AI
- computing framework
---

## PyTorch

### 高效的C++内核

Pytorch的核心库libtorch 使用C++实现了张量数据结构，GPU和CPU操作以及基本的并行原语。
不需要持有Python全局解释器锁，使用的PyTorch操作组成的运算可在多线程中执行。

### 分离控制和数据流
PyTorch在其控制流和数据流之间保持严格的分离。控制流的解析由运行在CPU上的Python和 C++代码处理，数据流既可以在CPU上也可以在GPU上运行。
此外，PyTorch利用CUDA流机制，使CPU上与GPU上的运算可以并行。

### 多进程优化
PyTorch将Python multiprocessing 模块扩展为 torch.multiprocessing，该模块会自动将发送到其他进程的张量数据移动到共享内存，而不是通过进程间通道来发送。

### 缓存分配优化
原生CUDAFREE操作会阻塞其调用者直至队列中所有任务结束，为了解决这一瓶颈，PyTorch实现自己的增量分配器

### 内存管理优化
PyTorch依靠引用计数的方案来跟踪每个张量的使用次数，一旦该计数达到零立即释放底层内存。


## Tensorflow
### 目标：
## 二者区别