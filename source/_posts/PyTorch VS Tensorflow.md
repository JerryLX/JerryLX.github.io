---
title: PyTorch V.S. Tensorflow
date: 2021-01-16 21:06:00
categories:
- something
tags: 
- AI
- computing framework
---

## **一、PyTorch**

### 高效的C++内核

Pytorch的核心库libtorch 使用C++实现了张量数据结构，GPU和CPU操作以及基本的并行原语。
不需要持有Python全局解释器锁，使用的PyTorch操作组成的运算可在多线程中执行。

### 分离控制和数据流
PyTorch在其控制流和数据流之间保持严格的分离。控制流的解析由运行在CPU上的Python和 C++代码处理，数据流既可以在CPU上也可以在GPU上运行。
此外，PyTorch利用CUDA流机制，使CPU上与GPU上的运算可以并行。

<!--more-->

### 多进程优化
PyTorch将Python multiprocessing 模块扩展为 torch.multiprocessing，该模块会自动将发送到其他进程的张量数据移动到共享内存，而不是通过进程间通道来发送。

### 缓存分配优化
原生CUDAFREE操作会阻塞其调用者直至队列中所有任务结束，为了解决这一瓶颈，PyTorch实现自己的增量分配器

### 内存管理优化
PyTorch依靠引用计数的方案来跟踪每个张量的使用次数，一旦该计数达到零立即释放底层内存。


## **二、Tensorflow**

- 异构环境中的大规模机器学习系统
- 使用数据流图表示“计算”、“状态”、“改变状态的操作”
- 将图中的节点映射到一个集群中的多个机器、一台机器中的多个计算设备（CPU、GPU、TPU等）。


### 设计理念：

- 基于计算原语的数据流图

    在TensorFlow中，数据流图的每个节点是一个独立的操作，如矩阵乘法、卷积等。这使得用户可以轻易地开发新的模型。

- 延期执行

    将执行推迟到整个程序可用之后，能够利用全局信息对执行过程进行优化，提高性能。举例来说：通过任务依赖关系优化并行度。


- 公用的加速器抽象
    - 统一的设备抽象：任意设备(CPU,GPU,TPU……)，只要实现规定的接口，就能够被程序使用。同一个程序在不同设备上不用修改就能充分发挥硬件特性，达到高性能。
    - 统一的数据类型：原始值封装成张量，作为所有设备都能理解的数据交换格式。

### 架构

<img src="framework.jpg" width = "30%"  alt="整体架构" />

- 用户层：用户层包括一些训练和推理、预测的工具库，以及提供了python、c++等客户端API。
- C API：隔离用户层和核心层。
- 核心层：考虑可移植性和性能,核心库用C++实现. 可在Linux. Mac OS X、Windows,、Android、and iOS等各种操作系统运行。 以及x86、ARM等各种架构的CPU，Kepler、Maxwell、Pascal等各种架构GPU上运行. 开源后在其他架构也有实现。
  - Distributed master: 作用是将用户需求翻译成一组任务。能够将计算图根据参与的硬件进行剪枝和划分。另外，由于master能够看到全局的计算步骤，所以能够进行一些标准的优化操作，例如公共表达式消除、常量合并等。 然后协调优化后的子图的执行。
  - Dataflow executor: 作用是处理从master得到的需求. 调度kernel的执行。 Dataflow executor能够将kernel分配到各个设备上。
  - Kernel: 是操作在某种硬件设备上的特定实现，包含200多种操作，包括数学运算，数组复制，控制流，状态管理。
  - 设备和网络: 每个设备实现send和recv接口. CPU和GPU使用cudaMemcpyAsync() API，GPU和GPU之间使用DMA。任务之间传输可以使用多种协议,例如基于TCP的gRPC，基于融合以太网(Converged Ethernet)的RDMA，在GPU和GPU之间也可以使用Collective operations 进行通信优化


### 分布式训练
- 模型并行：网络很大时，由于内存、显存限制，模型的不同部分分散到多个设备并行训练。缺点是层与层之间存在先后约束，并行度较低。
- 数据并行：一般使用数据并行，每个设备训练不同的batch，然后搜集这些梯度进行参数更新。

### 多设备执行
- 任务放置策略：预估张量的大小和运行时间，使用贪心算法选择能够最快完成任务的设备。
- 设备间通信：放置策略确定后，不同设备之间插入send/recv操作，通信方式可以是TCP、gRPC、RDMA等

### Kernel 实现

- Kernel是“操作”在某种硬件设备上的特定实现。
- Kernel(CPU)：基于Eigen::Tensor实现，Eigen是一个C++模板技术，能够生成高效的并发代码。
- Kernel(GPU)：可以使用Eigen技术，也可以直接使用cuBLAS，cuDNN，cuNCCL实现。也可以使用基于OpenCL的计算设备。
- 其他硬件实现：需要实现规定的接口，至少包括：
  - 执行接口
  - 为数据分配内存
  - 与主机内存之间数据传送

