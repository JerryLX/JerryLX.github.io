---
title: OneAPI
date: 2021-01-17 08:58:11
categories:
- something
tags:
- Intel
- Programming Model
---

# 一、背景与简介

对于开发者而言，应对包括CPU、GPU、FPGA等多种架构十分挑战。每一种架构都有自己的库以及编程语言，如CUDA。到目前为止，没有普适的且不影响系统性能的编程模型。这意味着每一种架构都需要使用单独的工具，且代码复用困难。

为了提升实现更快的应用程序性能、更高的生产率和更大的创新，OneAPI提供单一的软件抽象层让开发者进行跨不同类型硬件架构的直接编程。其开放、统一的标准保证了跨平台部署的稳定性以及系统的整体性能。

# 二、框架

<img src="framework.jpg" width = "60%"  alt="oneAPI框架" />

## 统一编程语言 (Data Parallel C++, DPC++)

**oneAPI规范的核心是DPC++**，它是基于ISO C++和Khronos SYCL标准构建的开放式跨体系结构语言。DPC++扩展了这些标准，并提供了明确的并行构造和卸载（offload）接口，以支持广泛的计算体系结构和处理器，包括cpu和加速器体系结构。oneAPI平台上可以通过Accelerator接口支持其他语言和编程模型。

其中，SYCL是一个在OpenCL上的C++抽象层，使得用户可以直接用简洁的C++对GPU等进行开发，而无需被OpenCL限制。

<!--more-->

## **库**
OneAPI为计算和数据密集型领域提供库，包括深度学习、科学计算、视频分析和媒体处理。

- **oneDPL (oneAPI DPC++ Library)**
  
  包含如下组成：
  
  - C++标准库的子集
  - 并行 STL 算法的扩展，使其能够运行在oneAPI设备上
  - 其他未在C++标准及SYCL标准中涉及的扩展

- **oneDNN (oneAPI Deep Neural Network Library)**

    <img src="OneDNN.png" width = "60%"  alt="OneDNN" />

    - **Primitive**: 封装特定计算的函子对象，如前向卷积、后向LSTM计算或数据转换操作。

    - **Engine**: 计算设备的抽象（CPU、指定GPU）
    - **Streams**: 封装绑定到特定引擎的执行上下文
    - **Memory**: 将句柄封装到特定引擎上分配的内存、张量维度、数据类型和内存格式—这是张量索引映射到线性内存空间中偏移量的方式。内存对象在执行期间传递给Primitive。

- **oneCCL (oneAPI Collective Communications Library )**

    提供机器学习应用的通讯原语

- **oneDAL (Data Analytics Library)**

    采用批量、在线和分布式计算处理模式，为数据分析的所有阶段（预处理、转换、分析、建模、验证和决策）提供高度优化的算法模块。


- **oneVPL (Video Processing Library)**

    是一个用于视频解码、编码和处理的编程接口，用于在CPU、GPU和其他加速器上构建便携式媒体管道。它在以媒体为中心的工作负载和视频分析工作负载中提供设备发现和选择，并为零拷贝缓冲区共享提供API原语。oneVPL向后和跨体系结构兼容，确保在当前和下一代硬件上实现最佳执行，而无需更改源代码。

- **oneMKL (Math Kernel Library)**

    定义一组用于高性能计算和其他应用程序的基本数学方法。作为oneAPI的一部分，oneMKL的设计允许在各种计算设备上执行：包括CPU、GPU、FPGA和其他加速器。该功能被细分为几个领域：密集线性代数，稀疏线性代数，离散傅立叶变换，随机数发生器和向量数学。
  
##  **硬件抽象层 （Level Zero）**

### 核心组成
Level Zero 核心API针对下列方法提供底层、细粒度、显式的控制：

- 设备发现和切分
- 内存分配、虚拟化、分配
- Kernel运行和调度
- 点对点传输
- 跨进程同步


#### Drivers and Devices

- driver对象表示一个物理设备集合，这个集合所有设备使用的相同的Level-Zero 驱动。
- device对象代表一个支持Level-Zero的物理设备
- Level-Zero启动时，首先调用ZeInit函数加载驱动，进一步发现支持的设备。


<img src="core_device.png" width = "60%"  alt="driver and device" />

##### Contexts

context对象是一个逻辑对象，用于驱动管理所有内存、任务队列、同步对象等等。
- context句柄主要用于创建和管理可能由多个设备使用的资源。例如，可以通过context类实现显式地设备间内存共享。

#### Memory and Images

- Memory: 线性、非格式化的分配，既可以从host上分配也可以在设备上分配。通过虚拟地址访问。
- Images: 非线性、格式化的分配，只能从设备上进行分配。不透明对象，通过句柄访问。

*Memory* 分为三种类型：

- Host：由主机所有，从系统内存分配
- Device：由某个设备所有，从该设备本地内存分配
- Shared：有多个设备/主机共享，可以在主机和1/多个设备间迁移。至少能够被一个host和一个设备访问。

*Images* 用于存储多维和格式定义的内存内容。不支持主机直接访问内容。然而，当其被复制到主机可访问内存分配时，其内容将被隐式解码。格式描述符是格式、类型和swizzle的组合。格式布局描述组件的数量及其相应的位宽度。类型描述了所有这些组件的数据类型，swizzle关联图像组件如何映射到内核的XYZW/RGBA通道。


#### Command Queues and Command Lists


<img src="core_queue.png" width = "60%"  alt="Command Queues and Command Lists" />

-  Command Queues主要与物理设备属性相关联，代表了一个对设备的逻辑输入流，其访问物理设备接近零延迟。
-  Command Lists 主要与主机线城相关联，代表了一个任务执行序列。

#### 同步原语

- Fence 是一个重量级的同步原语，用于向主机传达命令列表执行已经完成。
  - 一个fence与一个Command Queues相关联
  - 只能从设备的Command Queues中发出信号。
  - 在发出信号之前，fence保证设备和主机之间的执行完成和内存一致性。
  - 只有两种状态：未信号化和信号化，且不会隐式复位。
  - 栅栏只能从主机上重置。
  - 栅栏不能跨进程共享。

- Events 用于传达细粒度的主机到设备、设备到主机或设备到设备的依赖关系已经完成。

通过Fence和Events可以实现执行栅栏和内存栅栏，其中fence属于隐式、粗粒度控制而event是显示、细粒度的控制

#### Modules and Kernels
- Modules代表一个单一的翻译单元，由编译在一起的内核组成。
- Kernels代表模块内的内核，将直接从Command List中启动。

<img src="core_module.png" width = "60%"  alt="Modules and Kernels" />

### 工具集
提供底层设备访问：

- 指标报告
- 内核分析和调试

### 系统管理

- 查询加速器资源的性能、电源和运行状况
- 控制加速器资源的性能和功率分布
- 设施维护，如执行硬件诊断、更新固件或重置设备


# 三、实例

## 代码迁移工具

原始CUDA代码
```C++
#include <cuda.h>
#include <stdio.h>

const int vector_size = 256;

__global__ void SimpleAddKernel(float *A, int offset) 
{
  A[threadIdx.x] = threadIdx.x + offset;
 
}int main() 
{
  float *d_A;
  int offset = 10000;

  cudaMalloc( &d_A, vector_size * sizeof( float ) );
  SimpleAddKernel<<<1, vector_size>>>(d_A, offset);

  float result[vector_size] = { };
  cudaMemcpy(result, d_A, vector_size*sizeof(float), cudaMemcpyDeviceToHost);

  cudaFree( d_A );
   
  for (int i = 0; i < vector_size; ++i) {
    if (i % 8 == 0) printf( "\n" );
    printf( "%.1f ", result[i] );
  }

  return 0;
}
```

迁移后代码
```C++
#include <CL/sycl.hpp>
#include <dpct/dpct.hpp>
#include <stdio.h>

const int vector_size = 256;

void SimpleAddKernel(float *A, int offset, sycl::nd_item<3> item_ct1)
{ 
  A[item_ct1.get_local_id(2)] = item_ct1.get_local_id(2) + offset;
 
}int main()
{
  dpct::device_ext &dev_ct1 = dpct::get_current_device();
  sycl::queue &q_ct1 = dev_ct1.default_queue();
  float *d_A;
  int offset = 10000;

  d_A = sycl::malloc_device<float>(vector_size, q_ct1);
  q_ct1.submit([&](sycl::handler &cgh) {
    cgh.parallel_for(sycl::nd_range(sycl::range(1, 1, vector_size),
                                    sycl::range(1, 1, vector_size)),
                     [=](sycl::nd_item<3> item_ct1) {
                       SimpleAddKernel(d_A, offset, item_ct1);
                     });
  });

  float result[vector_size] = { };
  q_ct1.memcpy(result, d_A, vector_size * sizeof(float)).wait();

  sycl::free(d_A, q_ct1);

  for (int i = 0; i < vector_size; ++i) {
    if (i % 8 == 0) printf( "\n" );
    printf( "%.1f ", result[i] );
  }

  return 0;
}
```