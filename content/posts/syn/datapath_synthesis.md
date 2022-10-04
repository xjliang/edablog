---
title: "Datapath Synthesis"
tags: ["code"]
date: 2020-10-04T15:45:31+08:00
draft: false
---

## Introduction

datapath 在电路设计中是较大的运算元件，因此单独对 datapath 进行优化是非常重要的。

基本的 datapath synthesis 包括两个步骤：

1. datapath inferrencing：识别出 design 中包含的 datapath，存储必要的信息，在整个综合流程中 keep hierarchy，方便单独对其进行优化；

2. datapath implementation selection：在 infer 出 datapath 之后，我们赋予了这些 datapath block 一个抽象的标记，而我们的最终目标是将这些 datapath block 都转换为 gate-level 的netlist，这就需要针对每个特地的 datapath block，将其映射为 gate-level 的 netlist。
3. datapath implementation substitution：替换某个 datapath 的实现。

下面分别来说说这些步骤的目的以及实现方式。

## Datapath inferrencing

这个步骤类似于 synthesis 中的 elaboration，目的是识别出 design 中的 datapath block。

这个识别的实现方式暂时以 Yosys 为例进行说明。

Yosys 这个开源工具主要的流程是：

1. elaborate；
2. techmap;
3. opt;
4. abc opt & mapping;
5. dump final mapped netlist;

其中，techmap 这个步骤可以认为是将一个抽象的 operator（比如 +, -, *, /, ...）替换为一种 hard coded 的 module 实现，这个实现是通过 techmap.v 这个文件来给定的。

我们可以在 techmap 的过程中添加一些操作来标记想要 infer 的 datapath，keep hierarchy。

拿 adder 来举例，techmap 在遇到 \$add 这种 cell 时，通过 techmap.v 中的 attribute 匹配到 \$alu，alu 在往下就是一个具体的 adder instanciation。如果想要 keep adder 的 hierarchy，可以在 techmap 处理 alu 时做出处理，keep hierarchy 并通过添加额外的 attribute 的方式使得在后续的流程中可以找到这些 infer 过的 datapath。

## Datapath Implementation Selection

这个步骤是针对每个 datapath，选择一个合适的 architecture 来实现它。

那么，合适这个概念怎么理解？

一般来说，EDA 综合工具在设计时需要考虑 **PPA**，即 power, performace, area。综合工具需要在这几个指标之间做一个 tradeoff，给出一个符合设计约束且性能功耗都较优的设计方案。设计约束通常由用户给出，比如通过给定 SDC 来描述各个端口的 input_delay, output_delay，clock period 等等。SDC 一般也是 timing constraint 的说明文件。

回到 architecture 的设计问题，一个 datapath 可能有多种 architecture implementation，一种可能的设计方案是将每种实现都通过 Verilog 文件存储，在程序运行过程中载入内存，根据设计约束来选择某个 architecture。

以 adder 为例，传统的 adder architecture 有 ripple carry adder (**RCA**), brent-kung adder (**BK**), Kogge-Stone Adder (**KS**), Sklansky Adder (**SK**)。这些 architecture 的 delay 和 area 都比较明确。

Adder 的设计核心在于 carry chain 怎么处理，如果使用串行的方式，比如 ripple，那么最终的 delay 也会比较差。如果使用并行的方式（parallel prefix adder），最终的 delay 会比较小，而 area 也会比较大。

RCA 是最小的，也是最慢的。

BK, KS, SK 都是 PPA 的经典实现。这三种实现分别对应设计立方体中的三个顶点（wire track, fanout, level)。

## Datapath Implementation Substitution

上面提到了 implementation selection，然而很多决策都不是一蹴而就的，有时候需要在设计过程中对之前的决策进行调整。这时候 keep hierarchy 就起到了重要作用。通过 hierarchy，我们可以明确地知道每个 datapath 的 boundary，也就能够放心地将整个 module 进行替换，不用担心替换带来的功能错误。

一般这个替换操作可以做成一个 API，供流程中的其它步骤进行调用。比如在 timing optimization 的过程中，遇到某个 datapath 在 critical path（关键路径）上，这时候可以考虑将当前的 context 传递给 datapath generator，让其 generate 出一个适合当前 context 的 architecutre，然后通过替换 module 的这个 API 完成替换。

## Summary

本文主要介绍了 datapath synthesis 的基本步骤，重点以 adder architecture 为例展开说明，希望对屏幕前的你有所帮助！