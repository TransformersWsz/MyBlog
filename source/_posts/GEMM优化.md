---
title: GEMM优化
mathjax: true
toc: true
date: 2024-04-22 02:13:31
updated: 2024-04-22 02:13:31
categories:
- Machine Learning
tags:
- CUDA
- GPU
---
以矩阵相乘的优化为例：

<!--more-->

```c
__global__ void sgemm_V1(
    float * __restrict__ a, float * __restrict__ b, float * __restrict__ c,
    const int M, const int N, const int K) {

    /*
    在我们的例子里，
    dim3 blockDim(BN/TN, BM/TM) = (16, 16)，即一个block中有256个thread
    dim3 gridDim((N + BN - 1) / BN, (M + BM - 1) / BM) = (4，4)，即一共16个block
    */
    const int BM = 128;
    const int BN = 128;
    const int BK = 8;
    const int TM = 8;
    const int TN = 8;

    const int bx = blockIdx.x;
    const int by = blockIdx.y;
    const int tx = threadIdx.x; // thread在对应block内的行id
    const int ty = threadIdx.y; // thread在对应block内的列id
    const int tid = ty * blockDim.x + tx; // thread在对应block中的全局id（从左到右，从上到下，从0开始逐一标）

    /*
    在SMEM上对A和B，分别开辟大小为(BM, BK), (BK, BN)的空间
    对应到图例中，s_a为高亮红，s_b为高亮黄
    */
    __shared__ float s_a[BM][BK];
    __shared__ float s_b[BK][BN];

    /*
    初始化当前thread所维护的C矩阵（确定长度的数组，应该是定义在寄存器上的）
    */
    float r_c[TM][TN] = {0.0};
   
    /*
    例：
    对于tid = 0的thread，以下四个值分别为((0, 0), (0, 0)),
    意味着它负责把s_a(0,0)开始的连续4个数，s_b(0,0)开始的连续4个数，从global memory加载到SMEM

    对于tid = 1的thread，以下四个值分别为((0, 4), (0, 4)),
    意味着它负责把s_a(0,4)开始的连续4个数，s_b(0,4)开始的连续4个数，从global memory加载到SMEM

    对于tid = 2的thread，以下四个值分别为((1, 0), (0, 8))
    此时s_a第一行的8个数已经被前面的thread取完了，所以现在从s_a第二行开始取，s_b第一行没取完，继续进行
   
    对于tid = 18的thread，以下四个值分别为((9, 0), (0, 72))，含义同上
    */
    
    // 当前thread负责把A中的相关数据从global memory加载到SMEM，
    // 这里在计算该thread负责加载的第一个数在s_a中的row
    int load_a_smem_m = tid >> 1;  // tid/2, row of s_a
    // 当前thread负责加载的第一个数在s_a中的col
    int load_a_smem_k = (tid & 1) << 2;  // (tid % 2 == 0) ? 0 : 4, col of s_a
    
    // 当前thread负责把B中的相关数据从global memory加载到SMEM，
    // 这里在计算该thread负责加载的第一个数在s_b中的row
    int load_b_smem_k = tid >> 5;   // tid/32, row of s_b
    // 当前thread负责加载的第一个数在s_b中的col
    int load_b_smem_n = (tid & 31) << 2;  // (tid % 32) * 4, col of s_b

    /*
    例：
    对于tid = 0的thread，以下两个值为(256, 128)，
    表示该thread从s_a上取的第一个数，其位置在A（位于global memory）上的row 256
    该thread从s_b上取的第一个数，其位置在B（位于global memory）上的col 128
   
    对于tid = 18的thread，以下两个值为(265, 200)，道理同上
    */
    int load_a_gmem_m = by * BM + load_a_smem_m;  // global row of a
    int load_b_gmem_n = bx * BN + load_b_smem_n;  // global col of b

    /*
    对每个block，它都要经历K/Bk = 128/8 = 16次循环，每次循环计算一块s_a * s_b的结果
    这也意味着，对每个block内的每个thread，它的外循环也是16次
    */
    for (int bk = 0; bk < (K + BK - 1) / BK; bk++) {    
        /*
        1. 在block的单次循环中，需要把对应的s_a（高亮红）和s_b(高亮黄)从global memory
        加载到SMEM上，因此每个thread负责加载一部分s_a, s_b的数据，最后的__syncthreads()
        是保证thread们在正式计算前，都干完了自己加载的活，即完整的s_a, s_b已被加载到SMEM上
        */
        // 在这次循环中，当前thread从s_a上取的第一个数，其位置在A（位于global memory）上的col，与load_a_gmem_m对应
        int load_a_gmem_k = bk * BK + load_a_smem_k;   // global col of a
        // 在这次循环中，当前thread从s_a上取的第一个数，在A中的地址，即A[load_a_gmem_m][load_a_gmem_k]
        int load_a_gmem_addr = OFFSET(load_a_gmem_m, load_a_gmem_k, K);
        // 从这个地址开始，取出连续的4个数，将其从A所在的global memory上，加载到s_a上
        // 注：采用FLOAT4的好处是便于连续访存。如果存储的4个数在地址上不连续，你就发4条指令。float4的数据类型就只要发1条指令，一起取出
        FLOAT4(s_a[load_a_smem_m][load_a_smem_k]) = FLOAT4(a[load_a_gmem_addr]);
        // 在这次循环中，当前thread从s_b上取的第一个数，其位置在B（位于global memory）上的row，与load_b_gmem_n对应
        int load_b_gmem_k = bk * BK + load_b_smem_k;   // global row of b
        // 在这次循环中，当前thread从s_b上取的第一个数，在B中的地址，即B[load_b_gmem_k][load_b_gmem_n]
        int load_b_gmem_addr = OFFSET(load_b_gmem_k, load_b_gmem_n, N);
        // 同理将相关的数据从global memory加载到SMEM上
        FLOAT4(s_b[load_b_smem_k][load_b_smem_n]) = FLOAT4(b[load_b_gmem_addr]);
        // 在所有thread间做一次同步，保证在下面的计算开始时，s_a, s_b相关的数据已经全部从global memory搬运到SMEM上了
        __syncthreads();

        #pragma unroll
        /*
        2. 在block的单次循环中，每个thread采用split-by-k的方式，
        逐步累加计算当前thread所维护的(TM, TN)块的结果
        */
        // 遍历每一个(渐变红，渐变黄)对，可参见图例
        for (int k = 0; k < BK; k++) {
            #pragma unroll
            for (int m = 0; m < TM; m++) {
                #pragma unroll
                for (int n = 0; n < TN; n++) {
                    int comp_a_smem_m = ty * TM + m;
                    int comp_b_smem_n = tx * TN + n;
                    // 每次从SMEM上，各加载渐变红和渐变黄上的1个元素，到register，然后再计算
                    r_c[m][n] += s_a[comp_a_smem_m][k] * s_b[k][comp_b_smem_n];
                }
            }
        }
        // 做一次同步，保证所有的thread都计算完当前所维护的（TM, TN）块
        __syncthreads();
    }

    #pragma unroll
    /*
    3. 
    此时，所有的block已做完循环，
    我们把当前thread计算出的结果（存放在r_c中，尺寸为(Tm, Tn)）写回
    global memory上的C矩阵对应位置中
    */
    // 遍历当前thread结果矩阵的每一行
    for (int i = 0; i < TM; i++) {
        // 计算该thread结果矩阵的这一行，在C矩阵上对应的全局row
        int store_c_gmem_m = by * BM + ty * TM + i;
        #pragma unroll
        // 以4个数为1组，遍历该thread结果矩阵的每一列
        for (int j = 0; j < TN; j += 4) {
            // 计算这4个数中的第一个数在C矩阵上对应的全局col
            int store_c_gmem_n = bx * BN + tx * TN + j;
            // 将这4个数以FLOAT4写回global memory
            int store_c_gmem_addr = OFFSET(store_c_gmem_m, store_c_gmem_n, N);
            FLOAT4(c[store_c_gmem_addr]) = FLOAT4(r_c[i][j]);
        }
    }
}
```

___

## 参考
- [从啥也不会到Cuda GEMM优化](https://mp.weixin.qq.com/s/YLrsu1KAhzG8gFQ2L-TaMA)
- [传统 CUDA GEMM 不完全指北](https://zhuanlan.zhihu.com/p/584236348)
- [CUDA 矩阵乘法终极优化指南](https://zhuanlan.zhihu.com/p/410278370)