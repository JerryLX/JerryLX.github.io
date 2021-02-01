---
title: compute offloading
date: 2021-01-25 17:42:50
categories:
- something
tags:
- 计算任务调度
---

建模方法

- 节点能力：CPU cycles, 可从底层硬件获取
- 任务计算开销：cycles, 可通过profiling方式估计
- 计算时间：开销/节点能力
- 传输数据量：任务大小+参数大小
- 能耗：与任务计算开销正相关
- 总代价：utility = U(transmission_cost, computing_cost)