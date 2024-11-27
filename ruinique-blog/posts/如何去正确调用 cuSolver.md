---
title: 如何去正确调用 cuSolver
cover: /1732689719579.png
date: 2024-11-27 14:43:00
categories: ["读研"，"技术"]
author: ruinique
---

## cuSolver 简介

cuSolver 是 NVIDIA 提供的一套线性代数库，包含了一些常用的线性代数计算，比如求解线性方程组、特征值求解等。笔者在开发过程中，往往要和 cuSolver 打交道，但是笔者在这之前对 LAPACK 并不是很熟悉，看着那些 api 就像天书一样，所以在这里总结一下如何去正确调用 cuSolver。

## cuSolver 的组成

cuSolver 分成两个模块，分别是 cuSolver 和 cuSolverMG。cuSolver 则分为三个独立的组件，分别是 cuSolverDN、cuSolverSP 和 suSolverRF。cuSolverDN 用于求解稠密矩阵的线性方程组，cuSolverSP 用于求解稀疏矩阵的线性方程组，cuSolverRF 是一个稀疏重因子化包，在很多问题中，如果只有系数发生变化，而右端项不变，那么可以使用 cuSolverRF 来加速求解。

后面两个 cuSolverSP 和 suSolverRF 暂时不做讨论，因为笔者在开发过程中暂时并没有用到这两个模块~

## cuSolverDN 的简介

cuSolverDN 提供了 QR 分解和选主元的 LU 分解用来分解一般的矩阵。

对于对称/厄尔米特矩阵，cuSolverDN 提供了 Cholesky 分解。

对于对称不定矩阵，cuSolverDN 提供了 LDL 分解。

## 命名的一些约定

cuSolverDN 库提供了两类不同的 API，legacy 和 generic。

对于我们的 legacy API，我们使用 cuSolverDN 的前缀，比如 `cusolverDn<t><Op>`。

<t> 可以是 S、D、C 或 Z，分别代表 `float`、`double`、`cuComplex` 和 `cuDoubleComplex`。

<op> 可以是 Cholesky(`potrf`)、QR (`geqrf`)、LU (`getrf`) 或 LDL (`sytrf`) 。

而对于通用 API，我们在使用的时候和数据类型无关，支持用 64 位整数来定义矩阵和向量的维度，具体调用类似 `cusolverDn<Op>`。

## 其他的一些约定

我们的 cuSolver 的 API 往往是尽可能 async 的，所以我们需要用 `cudaDeviceSynchronize()` 来同步我们的计算。

当然我们还要用 `cudaMemcpy()` 并指定 `cudaMemcpyHostToDevice` 和 `cudaMemcpyDeviceToHost` 来同步我们的数据，这个操作是阻塞的，所以我们不需要额外的同步操作。

在必要时，我们的 cusolver 会用高精度去做迭代的细化。

## 如何使用我们的 cusolver

首先是要链接对应的库，笔者逐渐开始使用 cmake 了，用起来像这样：

```cmake
target_link_libraries(${ProjectName}
    cusolver
    cublas
    cublasLt
    cusparse
)
```

然后由于 `cusolver` 没有提供类似 `cudaMalloc` 的函数，所以需要我们去先调用对应的函数去获取 `buffer_size` 的大小，然后去确定预分配的临时空间大小。

比如我们要去调用函数去完成 LU 分解，具体步骤可以参考如下：

1. 首先我们要创建一个 `cusolverDnHandle_t` 的句柄，类似这样：
```cpp
cusolverDnHandle_t cusolver_handle;
cusolverDnCreate(&cusolver_handle);
```
> 句柄是 NVIDIA 很多官方库的一种设计模式，用于存储线性代数
1. 我们需要先去调用函数 `cusolverDn<t>getrf_bufferSize` 去获取我们的 `buffer_size` 的大小。
2. 然后自己去调用 `cudaMalloc` 去分配我们的 `buffer`。类似这样：
```cpp
cudaMalloc((void**)&buffer, buffer_size);
```
1. 接下来是我发现的一个坑点，居然 `cusolverDngetrf` 是已经 deprecated 的，我们需要去调用 `cusolverDnXgetrf`。结合他的参数列表，我们可以这样去调用：
```cpp
cusolverDnXgetrf(cusolver_handle, m, n, A, lda, buffer, pivots, info);
```

> 这里的 `cusolver_handle` 是我们的句柄，`m` 和 `n` 分别是我们的矩阵的行数和列数，`A` 是我们的矩阵，`lda` 是我们的矩阵的 leading dimension，`buffer` 是我们的临时空间，`pivots` 是我们的置换向量，`info` 是我们的状态码。

接下来，我们又知道对应的 LU 分解的结果会覆盖存储在 `A` 中，所以我们需要去调用 `cudaMemcpy` 来同步我们的数据。我们就可以在 CPU 中看到我们分解后的结果了。

## 某些函数的详细分析

以下是 `cusolverDnXgetrf_bufferSize` 函数参数的清晰表格展示，包括每个参数的含义、作用及是否有默认值：

| **参数顺序** | **参数名称**               | **类型**             | **含义**                        | **作用**                                                                                           | **是否有默认值**                           |
| ------------ | -------------------------- | -------------------- | ------------------------------- | -------------------------------------------------------------------------------------------------- | ------------------------------------------ |
| 1            | `handle`                   | `cusolverDnHandle_t` | cuSolver 上下文句柄             | 管理调用上下文，例如绑定 CUDA 流和当前设备，提供 GPU 资源访问接口。                                | 无，需通过 `cusolverDnCreate()` 显式创建。 |
| 2            | `params`                   | `cusolverDnParams_t` | cuSolver 参数配置               | 包含算法选择及计算配置参数，可以设置优化选项或使用默认配置。                                       | 如果未提供，则使用默认配置。               |
| 3            | `m`                        | `int64_t`            | 矩阵 \( A \) 的行数             | 决定输入矩阵 \( A \) 的行数，用于确定问题规模。                                                    | 无，用户需显式指定。                       |
| 4            | `n`                        | `int64_t`            | 矩阵 \( A \) 的列数             | 决定输入矩阵 \( A \) 的列数，用于确定问题规模。                                                    | 无，用户需显式指定。                       |
| 5            | `dataTypeA`                | `cudaDataType`       | 输入矩阵 \( A \) 的元素数据类型 | 指定矩阵元素类型，如 `CUDA_R_32F`（单精度浮点）或 `CUDA_R_64F`（双精度浮点）。                     | 无，用户需显式指定。                       |
| 6            | `A`                        | `const void*`        | 矩阵 \( A \) 的设备端指针       | 提供输入矩阵的数据地址，要求以列主序（Column Major）存储。                                         | 无，用户需显式提供有效指针。               |
| 7            | `lda`                      | `int64_t`            | 矩阵 \( A \) 的列主元素间距     | 指定列向量的存储间距（以元素为单位），必须满足 \( lda \geq \max(1, m) \)。                         | 无，用户需显式指定。                       |
| 8            | `computeType`              | `cudaDataType`       | 计算时的精度类型                | 指定计算时的精度，例如 `CUDA_R_32F` 或 `CUDA_R_64F`，可与 `dataTypeA` 不同（如支持混合精度运算）。 | 无，用户需显式指定。                       |
| 9            | `workspaceInBytesOnDevice` | `size_t*`            | 设备端工作区大小                | 返回设备端所需的工作区大小（以字节为单位），用户需根据返回值分配 GPU 内存。                        | 无，用户需提供有效指针。                   |
| 10           | `workspaceInBytesOnHost`   | `size_t*`            | 主机端工作区大小                | 返回主机端所需的工作区大小（以字节为单位），如需要在主机端临时存储数据。                           | 无，用户需提供有效指针。                   |

这里可以特别关心一下 `lda` 是怎么起作用的：

- `lda` 是矩阵 \( A \) 的列主元素间距，即矩阵 \( A \) 的列向量的存储间距（以元素为单位）。
- 对于我们一般的矩阵 \( A \) 来说，我们的 `lda` 一般是等于我们的矩阵的行数的。
- 但是如果我们的矩阵 \( A \) 是一个子矩阵，那么我们的 `lda` 就不是我们的矩阵的行数了，而是我们的子矩阵的行数。
- 当然，这也暗含了我们的矩阵 \( A \) 是以列主序存储的。

以下是 `cusolverDnXgetrf` 的参数详细说明表格：

| **参数**                   | **类型**             | **说明**                                                                               | **备注**                                                                  |
| -------------------------- | -------------------- | -------------------------------------------------------------------------------------- | ------------------------------------------------------------------------- |
| `handle`                   | `cusolverDnHandle_t` | cuSolver 句柄，由 `cusolverDnCreate` 创建，用于管理 cuSolver 库的上下文环境。          | 必须有效。                                                                |
| `params`                   | `cusolverDnParams_t` | cuSolver 参数结构体，可配置算法选项，如优化、精度等。传 `nullptr` 时使用默认配置。     | 可选。                                                                    |
| `m`                        | `int64_t`            | 矩阵行数。                                                                             | ≥ 0。                                                                     |
| `n`                        | `int64_t`            | 矩阵列数。                                                                             | ≥ 0。                                                                     |
| `dataTypeA`                | `cudaDataType`       | 矩阵元素的数据类型，例如：`CUDA_R_32F`（单精度）、`CUDA_R_64F`（双精度）。             | 描述输入矩阵类型。                                                        |
| `A`                        | `void*`              | 输入/输出矩阵的设备指针。函数结束后被覆盖为 LU 分解的结果。                            | 大小：`lda * n`。                                                         |
| `lda`                      | `int64_t`            | 矩阵的列主间距，表示矩阵列与列之间的元素偏移量（以元素个数计）。最小值为 `max(1, m)`。 | 确保 `A` 的存储连续性。                                                   |
| `ipiv`                     | `int64_t*`           | 主元数组的设备指针，存储分解过程中的行交换信息。大小为 `min(m, n)`。                   | 分解后返回行交换索引。                                                    |
| `computeType`              | `cudaDataType`       | 计算时使用的精度类型，例如：`CUDA_R_32F`、`CUDA_R_64F`。                               | 可以与 `dataTypeA` 不同，支持混合精度计算。                               |
| `bufferOnDevice`           | `void*`              | 设备端工作区的指针，用于存储临时计算数据。                                             | 需用 `cudaMalloc` 分配，大小由 `cusolverDnXgetrf_bufferSize` 提供。       |
| `workspaceInBytesOnDevice` | `size_t`             | 设备端工作区的大小（字节）。                                                           | 必须大于等于 `cusolverDnXgetrf_bufferSize` 返回值。                       |
| `bufferOnHost`             | `void*`              | 主机端工作区的指针，用于存储辅助计算数据。                                             | 需用 `malloc` 或类似函数分配，大小由 `cusolverDnXgetrf_bufferSize` 提供。 |
| `workspaceInBytesOnHost`   | `size_t`             | 主机端工作区的大小（字节）。                                                           | 必须大于等于 `cusolverDnXgetrf_bufferSize` 返回值。                       |
| `info`                     | `int*`               | 输出 LU 分解的状态信息：                                                               | **值解释**：<br>0：成功<br>>0：矩阵奇异<br><0：参数错误。                 |

这里因为去校验 LU 分解的结果，我们往往也需要调用 `cuBLAS` 的 `cublas<t>gemm()`。这里也介绍一下相关的参数：

| **参数** | **类型**            | **说明**                                                                                                      | **备注**                 |
| -------- | ------------------- | ------------------------------------------------------------------------------------------------------------- | ------------------------ |
| `handle` | `cublasHandle_t`    | cuBLAS 句柄，由 `cublasCreate` 创建，用于管理 cuBLAS 库的上下文环境。                                         | 必须有效。               |
| `transa` | `cublasOperation_t` | 矩阵 \( A \) 的转置操作，可选值为 `CUBLAS_OP_N`（不转置）、`CUBLAS_OP_T`（转置）、`CUBLAS_OP_C`（共轭转置）。 | 通常使用 `CUBLAS_OP_N`。 |
| `transb` | `cublasOperation_t` | 矩阵 \( B \) 的转置操作，可选值为 `CUBLAS_OP_N`（不转置）、`CUBLAS_OP_T`（转置）、`CUBLAS_OP_C`（共轭转置）。 | 通常使用 `CUBLAS_OP_N`。 |
| `m`      | `int`               | 矩阵 \( C \) 的行数。                                                                                         | ≥ 0。                    |
| `n`      | `int`               | 矩阵 \( C \) 的列数。                                                                                         | ≥ 0。                    |
| `k`      | `int`               | 矩阵 \( A \) 的列数和 \( B \) 的行数。                                                                        | ≥ 0。                    |
| `alpha`  | `const void*`       | 标量 \( \alpha \) 的设备指针。                                                                                | 通常为 `1`。             |
| `A`      | `const void*`       | 矩阵 \( A \) 的设备指针。                                                                                     | 大小：`lda * k`。        |
| `lda`    | `int`               | 矩阵 \( A \) 的列主元素间距。                                                                                 | ≥ `max(1, m)`。          |
| `B`      | `const void*`       | 矩阵 \( B \) 的设备指针。                                                                                     | 大小：`ldb * n`。        |
| `ldb`    | `int`               | 矩阵 \( B \) 的列主元素间距。                                                                                 | ≥ `max(1, k)`。          |
| `beta`   | `const void*`       | 标量 \( \beta \) 的设备指针。                                                                                 | 通常为 `0`。             |
| `C`      | `void*`             | 矩阵 \( C \) 的设备指针。                                                                                     | 大小：`ldc * n`。        |
| `ldc`    | `int`               | 矩阵 \( C \) 的列主元素间距。                                                                                 | ≥ `max(1, m)`。          |
