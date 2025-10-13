---
title: DivSufSort
categories:
  - hpatch
  - libdivsufsort
tags:
  - hpatch
  - libdivsufsort
abbrlink: 61d60ff1
date: 2025-10-03 09:12:19
---
<meta name="referrer" content="no-referrer" />

[TOC]

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/c8c82feae7364292a32d89af590c5cc6.png)

### **DivSufSort中的后缀类型：A, B, 和 B\* 详解**
- 简单来说，这是一种对所有后缀进行**分类**的巧妙方法，其最终目的是**找出并只排序一个规模小得多的“代表性”后缀子集**，然后再利用这个排好序的子集，通过廉价的线性扫描来“诱导”出所有其他后缀的正确顺序。

---

#### **1. Type A 和 Type B 后缀 (基础分类)**

算法首先将所有后缀（除了最后一个）分为两大类：Type A 和 Type B。分类的依据是**该后缀与它紧邻的下一个后缀的字典序大小关系**。

我们用 `S[i...]` 代表从位置 `i` 开始的后缀。

*   **Type B 后缀 (B for "Bigger-comes-after")**: 如果后缀 `S[i...]` **小于** 后缀 `S[i+1...]`，那么后缀 `i` 就是 **Type B**。
    *   **判断捷径:**
        *   如果 `T[i] < T[i+1]`，那么后缀 `i` **一定是** Type B。
        *   如果 `T[i] == T[i+1]`，那么后缀 `i` 的类型**与后缀 `i+1` 的类型相同**。

*   **Type A 后缀 (A for "Smaller-comes-after")**: 如果后缀 `S[i...]` **大于** 后缀 `S[i+1...]`，那么后缀 `i` 就是 **Type A**。
    *   **判断捷径:**
        *   如果 `T[i] > T[i+1]`，那么后缀 `i` **一定是** Type A。
        *   如果 `T[i] == T[i+1]`，规则同上，类型与后缀 `i+1` 相同。

**注意:** 最后一个后缀 `S[n-1...]` 通常被特殊定义或不参与初步分类。

**示例：**
让我们以字符串 `T = "misisipi"` 为例 (长度n=8)。
最后一个字符 `T[7]='i'` 是哨兵。

| i | T[i] | T[i+1] | T[i] vs T[i+1] | 递归依赖 | **最终类型** |
| :- | :--- | :--- | :--- | :--- | :--- |
| 7 | `i`  | (哨兵)  | -          | (特殊定义为 B) | **Type B**     |
| 6 | `p`  | `i`  | `p > i`    | -          | **Type A**     |
| 5 | `i`  | `p`  | `i < p`    | -          | **Type B**     |
| 4 | `s`  | `i`  | `s > i`    | -          | **Type A**     |
| 3 | `i`  | `s`  | `i < s`    | -          | **Type B**     |
| 2 | `s`  | `i`  | `s > i`    | -          | **Type A**     |
| 1 | `i`  | `s`  | `i < s`    | -          | **Type B**     |
| 0 | `m`  | `i`  | `m > i`    | -          | **Type A**     |

所以，`misisipi` 的类型序列是 `A B A B A B A B`。

#### **2. Type B\* 后缀 (特殊子集)**

Type B\* 后缀是算法真正要处理的“明星”子集。它的定义非常简单：

> **一个 Type B\* 后缀，就是一个 Type B 后缀，并且它紧跟在一个 Type A 后缀之后。**

换句话说，如果后缀 `i-1` 是 Type A，而后缀 `i` 是 Type B，那么后缀 `i` 就是一个 **Type B\*** 后缀。

在代码实现中，`sort_typeBstar` 从右向左扫描，当它从一个 Type A 区域 (`T[i] >= T[i+1]`) 刚刚进入一个 Type B 区域 (`T[i] < T[i+1]`) 时，遇到的第一个 Type B 后缀就是 B\* 后缀。

**继续看示例 `misisipi` (`A B A B A B A B`)**:

| i | 类型       | 前一个(i-1)的类型 | **是否为 B\*** |
| :- | :------- | :--- | :--- |
| 7 | **Type B** | Type A  | **是 (B\*)** |
| 6 | Type A   | Type B  | 否     |
| 5 | **Type B** | Type A  | **是 (B\*)** |
| 4 | Type A   | Type B  | 否     |
| 3 | **Type B** | Type A  | **是 (B\*)** |
| 2 | Type A   | Type B  | 否     |
| 1 | **Type B** | Type A  | **是 (B\*)** |
| 0 | Type A   | -       | 否     |

所以，对于字符串 `misisipi`，它的 B\* 后缀是 `S[1]`, `S[3]`, `S[5]`, `S[7]`。

#### **3. 为什么要这样分类？(算法的策略)**

现在是最关键的部分：这样分类有什么用？

1.  **分 (Divide):**
    *   整个 DivSufSort 算法的核心就是**不直接排序所有 `n` 个后缀**。
    *   它选择**只对数量少得多的 Type B\* 后缀进行排序**。`sort_typeBstar` 函数的唯一目的就是完成这个艰巨的任务。
    *   这个排序过程是递归的：它会把这些 B\* 后缀看作一个新的、更短的字符串，然后递归地调用排序算法来解决这个子问题。这就是代码中 `sssort` (Substring Sort) 的作用。

2.  **治 (Conquer):**
    *   一旦 B\* 后缀这个**子集**被正确排序，它们的位置就相当于在最终的后缀数组中建立了一个**“骨架”**或“锚点”。
    *   接下来的 `construct_SA` 函数（诱导排序阶段）会利用这个排好序的“骨架”，通过几次对数据的**线性扫描**，就能像多米诺骨牌一样，**推导出（诱导出）**所有其他 Type A 和 Type B 后缀的正确排序位置，并将它们填充到骨架的空隙中。
    *   这个“诱导”过程的计算成本远低于从头进行比较排序。

**总结一下：**

| 类型    | 定义                                      | 在算法中的角色                                                              |
| :------ | :---------------------------------------- | :-------------------------------------------------------------------------- |
| **Type A**  | `S[i...] > S[i+1...]`                     | “普通”后缀，在 `sort_typeBstar` 阶段不被直接排序。                            |
| **Type B**  | `S[i...] < S[i+1...]`                     | “普通”后缀，在 `sort_typeBstar` 阶段不被直接排序。                            |
| **Type B\*** | **一个 Type B 后缀，且其前一个是 Type A** | **“代表性”子集**。算法只对这个子集进行复杂的递归排序，作为构建完整SA的**骨架**。 |

通过这种`识别特殊子集 -> 递归排序子集 -> 诱导排序全集`的策略，DivSufSort 成功地将问题规模在每层递归中都大幅缩小，从而实现了卓越的性能。

## divsufsort **C 库入口与核心调度器**

这是 `libdivsufsort` 库的32位版本主入口函数。它本身不包含复杂的算法逻辑，而是一个**顶层调度器 (Top-Level Dispatcher)**，负责参数校验、内存分配，并按照特定顺序调用内部的核心工作函数来完成整个后缀数组的构建过程。

### **原理与设计思路解析**

DivSufSort 算法是一种高效的后缀数组构造算法，其名称意为“分治后缀排序”(Divide-and-Conquer Suffix Sort)。这个顶层函数清晰地展示了该算法宏观上的“三步走”策略。

*   **核心策略：分治与诱导排序 (Divide-and-Conquer and Induced Sorting)**
    这个算法的精髓在于，它**不直接对所有后缀进行排序**，因为这非常慢。相反，它采用了一种更聪明的分治策略：

    1.  **第一步：分类与识别**
        *   算法首先将所有后缀分为不同的类型（在 `divsufsort` 中主要是 "Type A" 和 "Type B"）。
        *   然后，它从 "Type B" 后缀中识别出一个更小的、具有特殊性质的子集，称为 "Type B*" 后缀。

    2.  **第二步：递归排序子集 (`sort_typeBstar`)**
        *   这是算法的“分”阶段。它**只对这个规模小得多的 "Type B*" 后缀子集进行排序**。
        *   这个排序过程是递归的，也就是说，如果这个子集本身还很大，算法会对其再次应用相同的分治策略，直到问题规模小到可以轻松解决。
        *   这个函数是整个过程中计算最密集的部分，也是**多线程 (`threadNum`) 发挥作用的地方**。
        *   它会将排好序的 "Type B*" 后缀的索引，先临时存放在 `SA` 数组的后半部分。同时，它会返回这个子集的大小 `m`。

    3.  **第三步：诱导构建完整数组 (`construct_SA`)**
        *   这是算法的“治”阶段，也是其“诱导排序”思想的体现。
        *   一旦规模较小的 "Type B*" 后缀已经排好序，它们的位置就成了整个后缀数组的“锚点”或“骨架”。
        *   `construct_SA` 函数利用这些已排序的 "Type B*" 后缀作为参考，通过一次或几次对原始数据的线性扫描，就能**高效地、确定性地推导出所有其他类型后缀（"Type A"）的正确排序位置**，并将它们填充到 `SA` 数组的相应位置。
        *   这个“诱导”步骤的复杂度是线性的 `O(n)`，速度极快。

*   **桶的作用 (`bucket_A`, `bucket_B`)**
    *   `bucket_A` 和 `bucket_B` 是用于实现**基数排序 (Radix Sort)** 或 **计数排序 (Counting Sort)** 的临时内存空间。
    *   在排序的各个阶段，算法需要快速地根据后缀的第一个或前几个字符将它们分组。通过预先扫描一遍原始数据，统计每个字符（0-255）的出现次数，就可以确定所有以 'a' 开头的后缀在最终排好序的 `SA` 数组中应该处于哪个范围（“桶”），所有以 'b' 开头的又应该在哪个范围，以此类推。
    *   这两个桶数组就是用来存储这些计数和范围信息的，它们是算法实现高性能的关键辅助数据结构。

*   **对小字符串的特殊处理**
    *   函数开头对 `n <= 2` 的情况进行了硬编码处理。这既是递归算法的**基本情况 (Base Case)**，也是一个简单的性能优化，避免了为极小的输入启动复杂的分配和排序流程。

### **代码解析**

```c
/*- Function -*/

/**
 * @brief divsufsort 32位版本的主入口函数。
 * @param T          输入的原始数据（Text）。
 * @param SA         输出参数，用于存储最终的后缀数组（Suffix Array）。
 * @param n          输入数据的长度。
 * @param threadNum  用于排序的线程数。
 * @return saint_t   成功返回0，失败返回错误码。
 */
saint_t
divsufsort(const sauchar_t *T, sastore_t* SA, saidx_t n,int threadNum) {
  saidx_t *bucket_A, *bucket_B; // 指向桶数组的指针。
  saidx_t m; // 用于存储 Type B* 子集的大小。
  saint_t err = 0; // 初始化错误码为0（成功）。

  /* 1. 参数校验和处理基本情况 */
  if((T == NULL) || (SA == NULL) || (n < 0)) { return -1; } // 空指针或无效长度
  else if(n == 0) { return 0; } // 空字符串
  else if(n == 1) { SA[0] = 0; return 0; } // 单字符字符串
  else if(n == 2) { // 双字符字符串的硬编码排序
    m = (T[0] < T[1]); 
    SA[m ^ 1] = 0, SA[m] = 1; 
    return 0; 
  }

  /* 2. 分配桶内存 */
  // BUCKET_A_SIZE 和 BUCKET_B_SIZE 是预定义的常量（通常是 256 或 256*256）。
  bucket_A = (saidx_t *)malloc(BUCKET_A_SIZE * sizeof(saidx_t));
  bucket_B = (saidx_t *)malloc(BUCKET_B_SIZE * sizeof(saidx_t));

  /* 3. 执行核心排序流程 */
  if((bucket_A != NULL) && (bucket_B != NULL)) { // 确保内存分配成功
    // 步骤一：递归排序 Type B* 子集。这是最耗时的部分，支持多线程。
    m = sort_typeBstar(T, SA, bucket_A, bucket_B, n, threadNum);
    // 步骤二：利用已排序的子集，诱导构建出完整的后缀数组。
    construct_SA(T, SA, bucket_A, bucket_B, n, m);
  } else {
    err = -2; // 内存分配失败错误码
  }

  /* 4. 释放资源 */
  free(bucket_B);
  free(bucket_A);

  return err; // 返回最终状态
}
```

## sort_typeBstar **DivSufSort 核心：B\* 后缀排序**

`sort_typeBstar`。这是整个算法中最复杂、计算最密集的部分。它的唯一目标是：只对数量相对较少的 "Type B\*" 后缀进行正确的排序。

### **原理与设计思路解析**

这个函数本身就是一个完整的多阶段排序算法。它通过一系列精巧的扫描、计数、排序和重编码操作，实现了对特殊子集（B\*后缀）的高效排序。

*   **阶段一：扫描与计数 (Scanning and Counting)**
    *   **目标：** 在一次从右到左的线性扫描中，完成两项任务：
        1.  **识别并存储 B\* 后缀：** 扫描过程中，根据当前字符与前一个字符的大小关系，算法能判断出当前位置 `i` 是不是一个 B\* 后缀的起始点。如果是，就将这个位置索引 `i` **逆序存入 `SA` 数组的末尾**。
        2.  **计数：** 同时，对所有后缀（A, B, B\*）的**前两个字符**进行频率统计，结果存入 `bucket_A` 和 `bucket_B`。这本质上是在为后续的**基数排序**做准备。
    *   **结果：** 扫描结束后，`SA` 数组的后 `m` 个位置（`SA+n-m`）包含了所有 B\* 后缀的**原始位置**（但还未排序），并且我们知道了B\*后缀的总数 `m`。

*   **阶段二：初步排序 (Initial Radix Sort)**
    *   **目标：** 利用第一阶段统计的频率信息，对所有 B\* 后缀进行一次基于其**前两个字符**的基数排序。
    *   **实现：** 首先，`BUCKET_BSTAR` 数组被转换成“桶”的边界指针。然后，代码从后向前遍历存储在 `SA` 数组末尾的 B\* 后缀，根据每个后缀的前两个字符，将它们放入 `SA` 数组前半部分中对应的桶里。
    *   **结果：** 执行完毕后，`SA` 数组的前 `m` 个位置现在存放着 B\* 后缀的**索引**，这些索引已经按照前两个字符排好了序。在同一个桶内的后缀，它们的相对顺序还是乱的。

*   **阶段三：递归排序 (`sssort` - The Core Recursion)**
    *   **目标：** 解决第二阶段留下的问题——对每个桶内部的、前两个字符相同的 B\* 后缀进行**完整排序**。
    *   **`sssort` (Substring Sort):** 这是 `divsufsort` 算法的递归核心。对于一个需要排序的桶，它会比较桶内所有后缀的**后续子串**。如果这些子串通过比较仍然无法分出胜负，它就会将这些子串本身看作一个新的、更小的问题，**递归地调用排序算法**。
    *   **多线程并行 (`_IS_USED_MULTITHREAD`)**: **这正是多线程发挥作用的地方**。代码会将所有需要排序的桶（即大小大于1的桶）组织成一个工作队列。然后创建 `threadNum-1` 个线程，与主线程一起，从队列中领取这些桶并调用 `_sssort_thread` 对它们进行排序。由于不同桶的排序是完全独立的，这个过程可以实现高效的并行化。

*   **阶段四：排名计算与重编码 (Rank Calculation and Renaming)**
    *   **目标：** 将排好序的 B\* 后缀转换成一个新的、更短的字符串，为最终的排序（`trsort`）做准备。
    *   **实现：** 算法遍历已排序的 B\* 后缀，比较相邻两个后缀是否完全相同。
        *   为每个**唯一**的 B\* 后缀分配一个**新的排名（rank）**。
        *   这个过程充满了各种位运算技巧（如 `~SA[i]`），用负数来标记组的边界，从而在 `SA` 数组本身上完成复杂的排名计算，极大地节省了内存。
        *   `ISAb` (Inverse Suffix Array for B\*) 数组被用来存储每个 B\* 后缀（按原始位置索引）所获得的**新排名**。

*   **阶段五：最终排序 (`trsort` - Final Sort of Ranks)**
    *   **目标：** 构建 B\* 后缀的逆后缀数组 (Inverse Suffix Array)。
    *   **`trsort` (Tandem Repeat Sort):** 这是一个专门用于处理整数数组（即刚刚计算出的排名数组）的、高度优化的排序算法。它对 `ISAb` 数组进行排序。
    *   **结果：** `trsort` 执行后，`SA` 数组的前 `m` 个位置现在存储的是**最终排好序的 B\* 后缀的排名**。

*   **阶段六：结果回写与桶边界重计算**
    *   **目标：** 将最终的排序结果（B\* 后缀的正确位置）写回 `SA` 数组，并为下一阶段 `construct_SA` 重新计算桶的边界。
    *   **实现：** 再次扫描数据，并使用 `ISAb` 作为查找表，将每个 B\* 后缀的最终排序位置 `SA[ISAb[--j]]` 确定下来。同时，利用 `bucket_A` 和 `bucket_B`，重新计算出所有 Type A 和 Type B 后缀桶的边界，为 `construct_SA` 的诱导排序做好准备。

### **代码解析**

```c
/* Sorts suffixes of type B*. */
static
saidx_t
sort_typeBstar(const sauchar_t *T, sastore_t* SA,
               saidx_t *bucket_A, saidx_t *bucket_B,
               saidx_t n,int threadNum) {
  sastore_t *PAb, *ISAb;
  saidx_t i, j, k, t, m;
  saint_t c0, c1;

  /* Initialize bucket arrays. */
  // 初始化桶A和桶B，用于后续的计数和基数排序。
  for(i = 0; i < BUCKET_A_SIZE; ++i) { bucket_A[i] = 0; }
  for(i = 0; i < BUCKET_B_SIZE; ++i) { bucket_B[i] = 0; }

  /* Count the number of occurrences of the first one or two characters of each
     type A, B and B* suffix. Moreover, store the beginning position of all
     type B* suffixes into the array SA. */
  // 阶段一：从右到左扫描字符串，进行计数和B*后缀识别。
  for(i = n - 1, m = n, c0 = T[n - 1]; 0 <= i;) {
    /* type A suffix. */
    // T[i] >= T[i+1] 的后缀是 Type A。循环统计。
    do { ++BUCKET_A(c1 = c0); } while((0 <= --i) && ((c0 = T[i]) >= c1));
    if(0 <= i) {
      /* type B* suffix. */
      // 从Type A区域进入Type B区域的第一个后缀是 B* 后缀。
      ++BUCKET_BSTAR(c0, c1); // 根据其前两个字符进行计数。
      SA[--m] = i; // 将B*后缀的起始位置i，逆序存入SA数组的末尾。
      /* type B suffix. */
      // T[i] < T[i+1] 的后缀是 Type B。循环统计。
      for(--i, c1 = c0; (0 <= i) && ((c0 = T[i]) <= c1); --i, c1 = c0) {
        ++BUCKET_B(c0, c1);
      }
    }
  }
  m = n - m; // 计算出B*后缀的总数。
/*
note:
  A type B* suffix is lexicographically smaller than a type B suffix that
  begins with the same first two characters.
*/

  /* Calculate the index of start/end point of each bucket. */
  // 将频率计数转换为用于基数排序的桶边界。
  for(c0 = 0, i = 0, j = 0; c0 < ALPHABET_SIZE; ++c0) {
    t = i + BUCKET_A(c0);
    BUCKET_A(c0) = i + j; /* start point */
    i = t + BUCKET_B(c0, c0);
    for(c1 = c0 + 1; c1 < ALPHABET_SIZE; ++c1) {
      j += BUCKET_BSTAR(c0, c1);
      BUCKET_BSTAR(c0, c1) = j; /* end point */
      i += BUCKET_B(c0, c1);
    }
  }

  if(0 < m) {
    /* Sort the type B* suffixes by their first two characters. */
    // 阶段二：基于前两个字符的初步基数排序。
    PAb = SA + n - m; // PAb指向存储在SA末尾的B*后缀原始位置。
    ISAb = SA + m;    // ISAb指向SA数组中间的临时工作区，用于后续存储逆后缀数组。
    // 从后向前遍历（保证稳定性），将B*后缀放入对应的桶中。
    for(i = m - 2; 0 <= i; --i) {
      t = PAb[i], c0 = T[t], c1 = T[t + 1]; // 获取位置和前两个字符。
      // BUCKET_BSTAR现在是桶的写指针，--后将PAb的索引i放入。
      SA[--BUCKET_BSTAR(c0, c1)] = i;
    }
    t = PAb[m - 1], c0 = T[t], c1 = T[t + 1];
    SA[--BUCKET_BSTAR(c0, c1)] = m - 1;

    /* Sort the type B* substrings using sssort. */
    // 阶段三：对每个桶内部进行递归或并行的深度排序。
#if (_IS_USED_MULTITHREAD)
    if (threadNum>1){
        const saidx_t bufsize = (n - (2 * m)) / (saidx_t)threadNum; // 计算每个线程的缓冲区大小。
        const int threadCount=threadNum-1;
        c0 = ALPHABET_SIZE - 2, c1 = ALPHABET_SIZE - 1, j = m; // 共享的工作队列计数器。
        mt_data_t mt_data; // 设置共享给所有线程的上下文数据。
        mt_data.T=T;
        mt_data.SA=SA;
        mt_data.bucket_B=bucket_B;
        mt_data.PAb=PAb;
        mt_data.bufsize=bufsize;
        mt_data.n=n;
        mt_data.m=m;
        std::vector<std::thread> threads(threadCount);
        sastore_t* buf = SA + m; // 指向排序用的主缓冲区。
        // 创建N-1个工作线程。
        for (int ti=0;ti<threadCount;++ti,buf+=bufsize){
            threads[ti]=std::thread(_sssort_thread,&c0,&c1,&j,buf,&mt_data);
        }
        _sssort_thread(&c0,&c1,&j,buf,&mt_data); // 主线程也参与计算。
        // 等待所有线程结束。
        for (int ti=0;ti<threadCount;++ti)
            threads[ti].join();
    }else
#endif
    { // 单线程版本
        sastore_t* buf = SA + m;
        saidx_t bufsize = n - (2 * m);
        // 遍历所有B*后缀的桶。
        for(c0 = ALPHABET_SIZE - 2, j = m; 0 < j; --c0) {
            for(c1 = ALPHABET_SIZE - 1; c0 < c1; j = i, --c1) {
                i = BUCKET_BSTAR(c0, c1);
                // 如果桶内元素超过1个，则需要进行深度排序。
                if(1 < (j - i)) {
                    sssort(T, PAb, SA + i, SA + j,
                           buf, bufsize, 2, n, *(SA + i) == (m - 1));
                }
            }
        }
    }

    /* Compute ranks of type B* substrings. */
    // 阶段四：计算排名并进行重编码。
    for(i = m - 1; 0 <= i; --i) {
      if(0 <= SA[i]) { // 遇到一个非负数，表示一个新组的开始。
        j = i;
        // 为组内所有相同的B*后缀分配相同的排名i。
        do { ISAb[SA[i]] = i; } while((0 <= --i) && (0 <= SA[i]));
        // 在组的末尾用负数记录组的大小。
        SA[i + 1] = i - j;
        if(i <= 0) { break; }
      }
      j = i;
      // 对于唯一的B*后缀组，用取反操作(~)来标记，并赋予排名。
      do { ISAb[SA[i] = ~SA[i]] = j; } while(SA[--i] < 0);
      ISAb[SA[i]] = j;
    }

    /* Construct the inverse suffix array of type B* suffixes using trsort. */
    // 阶段五：对B*后缀的排名数组进行最终排序。
    trsort(ISAb, SA, m, 1);

    /* Set the sorted order of tyoe B* suffixes. */
    // 阶段六：回写最终排序结果。
    for(i = n - 1, j = m, c0 = T[n - 1]; 0 <= i;) {
      for(--i, c1 = c0; (0 <= i) && ((c0 = T[i]) >= c1); --i, c1 = c0) { }
      if(0 <= i) {
        t = i;
        for(--i, c1 = c0; (0 <= i) && ((c0 = T[i]) <= c1); --i, c1 = c0) { }
        // 使用ISAb（B*排名的逆SA）作为查找表，将B*的原始位置t放入其最终排好序的位置。
        SA[ISAb[--j]] = ((t == 0) || (1 < (t - i))) ? t : ~t;
      }
    }

    /* Calculate the index of start/end point of each bucket. */
    // 为下一阶段 construct_SA 重新计算所有桶的边界。
    BUCKET_B(ALPHABET_SIZE - 1, ALPHABET_SIZE - 1) = n; /* end point */
    for(c0 = ALPHABET_SIZE - 2, k = m - 1; 0 <= c0; --c0) {
      i = BUCKET_A(c0 + 1) - 1;
      for(c1 = ALPHABET_SIZE - 1; c0 < c1; --c1) {
        t = i - BUCKET_B(c0, c1);
        BUCKET_B(c0, c1) = i; /* end point */

        /* Move all type B* suffixes to the correct position. */
        // 将排好序的B*后缀从SA数组的前部移动到其最终归属的桶中（位于SA数组的后部）。
        for(i = t, j = BUCKET_BSTAR(c0, c1);
            j <= k;
            --i, --k) { SA[i] = SA[k]; }
      }
      BUCKET_BSTAR(c0, c0 + 1) = i - BUCKET_B(c0, c0) + 1; /* start point */
      BUCKET_B(c0, c0) = i; /* end point */
    }
  }

  return m; // 返回B*后缀的数量。
}
```

## _sssort_thread **多线程递归排序工作单元**

这是 `divsufsort` 库中负责执行**并行排序**的核心工作线程函数。当 `sort_typeBstar` 决定启用多线程时，它会创建多个线程，每个线程都执行这个函数。`_sssort_thread` 的任务是从一个共享的“工作队列”中不断领取需要排序的“桶”，并调用底层的 `sssort` 函数对它们进行深度排序。

### **原理与设计思路解析**

这个函数是典型的**生产者-消费者**模型中的**消费者**，只不过这里的“生产”是一次性完成的（所有待排序的桶在一开始就已经确定），线程只负责消费。

*   **核心策略：共享工作队列与互斥访问**
    *   **工作队列 (Shared Work Queue):** 待排序的任务（即那些大小大于1的B\*后缀桶）并不是被静态地分配给每个线程。相反，它们构成了一个逻辑上的“工作队列”。这个队列由三个共享的原子变量（通过锁来保证原子性）来表示：`*c0`, `*c1`, 和 `*j`。
        *   `*c0` 和 `*c1`：代表当前正在处理的桶的前两个字符。
        *   `*j`：代表当前桶的结束边界。
        *   `k = BUCKET_BSTAR(d0, d1)`：通过这两个字符，可以查到桶的起始边界。
    *   **互斥锁 (`CHLocker`)**: 由于多个线程需要同时访问和修改 `*c0`, `*c1`, `*j` 这三个共享变量，为了防止竞争条件（race condition），所有对它们的读写操作都必须在一个**临界区 (Critical Section)** 内进行。代码使用了 `CAutoLocker` 这个 RAII 风格的锁来实现：
        *   当一个线程进入 `for(;;)` 循环的 `{...}` 代码块时，`CAutoLocker` 对象被创建，自动获取锁。
        *   线程在这个代码块内安全地读取和修改 `*c0`, `*c1`, `*j`，为自己“领取”下一个要处理的桶 `[k, l)`。
        *   当线程离开这个代码块时，`CAutoLocker` 对象被销毁，自动释放锁，允许其他线程进入。

*   **动态任务分配的优势**
    *   为什么不一开始就把所有桶平分给每个线程呢？因为不同桶的大小（即需要排序的后缀数量）可能相差悬殊。
    *   如果静态分配，一个线程可能很快就完成了它所有的小桶，然后就进入空闲状态；而另一个线程可能还在处理一个巨大的桶。这会导致CPU资源浪费。
    *   通过动态的“领取”模式，可以确保只要还有未排序的桶，所有线程都会保持忙碌，从而实现更好的**负载均衡 (Load Balancing)**。

*   **线程私有与共享数据**
    *   `buf`: 每个线程都会分到一块**私有的**、不重叠的临时工作缓冲区 `buf`。这非常重要，因为它意味着底层的 `sssort` 函数在执行时，不需要任何锁，可以在自己的私有内存上全速运行。
    *   `mt_data`: 这是一个包含了所有线程共享的、**只读**数据的结构体（如指向原始文本T的指针，指向主SA数组的指针等）。由于是只读的，所以访问它不需要加锁。

**执行流程总结：**

1.  线程进入一个无限循环 `for(;;)`。
2.  **获取任务:**
    *   线程尝试获取互斥锁。
    *   获取锁后，它读取共享的队列指针 (`*c0`, `*c1`, `*j`)，为自己寻找下一个大小大于1的桶 `[k, l)`。
    *   找到后，它更新队列指针，将这个桶标记为“已被领取”。
    *   释放锁。
3.  **检查终止:** 如果上一步没有领到任务 (`l == 0`)，说明所有桶都已被处理完毕，线程退出循环。
4.  **执行任务:**
    *   如果领到了任务，线程就调用核心的递归排序函数 `sssort`。
    *   `sssort` 会在线程**私有的缓冲区 `buf`** 上，对共享 `SA` 数组中的 `[k, l)` 这个片段进行深度排序。
5.  循环回到第1步，尝试领取下一个任务。

### **代码解析**

```c
#if (_IS_USED_MULTITHREAD)
#if defined (__cplusplus)
namespace{ // 使用匿名命名空间来限定 mt_data_t 的作用域，避免全局污染
#endif
typedef struct mt_data_t{
    CHLocker            locker;      // 互斥锁，用于保护共享的工作队列指针
    const sauchar_t*    T;           // 指向原始文本（只读共享）
    sastore_t*          SA;          // 指向主SA数组（读写，但sssort只操作指定片段）
    const saidx_t*      bucket_B;    // 指向桶边界数组（只读共享）
    const sastore_t*    PAb;         // 指向B*后缀原始位置数组（只读共享）
    saidx_t             bufsize;     // 线程私有缓冲区的大小
    saidx_t             n;           // 文本总长度
    saidx_t             m;           // B*后缀总数
} mt_data_t;
#if defined (__cplusplus)
}//namespace
#endif

/**
 * @brief 多线程排序的工作函数，每个线程执行此函数。
 * @param c0, c1, j  指向共享的工作队列指针，这些指针会被多个线程修改。
 * @param buf        指向本线程专属的、不重叠的临时工作区。
 * @param mt         指向包含所有共享上下文数据的结构体。
 */
static void _sssort_thread(saint_t* c0,saint_t* c1,saidx_t* j,
                           sastore_t *buf,mt_data_t* mt){
    saidx_t k = 0; // 本线程将要处理的桶的起始位置
    saidx_t l;     // 本线程将要处理的桶的结束位置
    const saidx_t*  bucket_B=mt->bucket_B;

    // 无限循环，直到所有工作被完成
    for(;;) {
        { // 进入临界区，CAutoLocker在作用域开始时加锁，结束时解锁
            CAutoLocker __autoLocker(mt->locker.locker);
            
            // 从共享指针 j 读取当前队列的末尾
            if(0 < (l = *j)) {
                saint_t d0 = *c0, d1 = *c1;
                // 从后向前遍历所有桶，寻找下一个需要处理的（大小>1）
                do {
                    k = BUCKET_BSTAR(d0, d1); // 查找当前(d0, d1)桶的起始位置
                    // 迭代到下一个桶
                    if(--d1 <= d0) {
                        d1 = ALPHET_SIZE - 1;
                        if(--d0 < 0) { break; } // 所有桶都遍历完了
                    }
                } while(((l - k) <= 1) && (0 < (l = k))); // 循环直到找到一个大小>1的桶，或者遍历完
                
                // 更新共享队列指针，将找到的桶 [k, l) 从队列中“取出”
                *c0 = d0, *c1 = d1, *j = k;
            }
        }
        
        // 如果 l == 0，说明没有领到任务，所有工作都完成了
        if(l == 0) { break; }
        
        // 获取到任务 [k, l)，调用核心排序函数
        sastore_t* SA=mt->SA;
        sssort(mt->T, mt->PAb, SA + k, SA + l,
               buf, mt->bufsize, 2, mt->n, *(SA + k) == (mt->m - 1));
    }
}
#endif
```

## construct_SA **诱导排序：构建完整后缀数组**

这是 `divsufsort` 算法的“治”阶段，也是其“诱导排序”思想的完美体现。在 `sort_typeBstar` 费尽心力地将规模较小的 B\* 后缀排好序之后，`construct_SA` 函数接管后续工作。它的任务是利用这个已排序的 B\* 后缀“骨架”，通过**两次高效的线性扫描**，像推倒多米诺骨牌一样，**推导出（诱导出）**所有其他后缀（Type A 和 Type B）的正确排序位置。

### **原理与设计思路解析**

`construct_SA` 的 brilliant idea 在于它利用了一个简单的、确定性的关系：**如果后缀 `S[i...]` 的排序位置已知，那么后缀 `S[i-1...]` 的相对排序位置也可以被高效地推导出来**。

这个函数分为两个主要的诱导排序阶段：

*   **阶段一：诱导排序 Type B 后缀 (从 B\* 推导 B)**
    *   **扫描方向:** **从右到左**扫描 `SA` 数组。
    *   **核心逻辑:**
        1.  算法从 `SA` 数组的末尾开始向前扫描。当它遇到一个**已排序**的后缀 `s` 时（无论是 B\* 还是在上一轮被诱导出的 B），它会考察其**前一个位置** `s-1`。
        2.  如果后缀 `s-1` 恰好是一个 **Type B** 后缀，那么算法就知道后缀 `s-1` 的正确排序位置。
        3.  **如何确定位置？** `s-1` 的第一个字符是 `T[s-1]`，第二个字符是 `T[s]`。利用在 `sort_typeBstar` 结尾处重新计算好的 `bucket_B`，算法可以立即定位到所有以 `(T[s-1], T[s])` 开头的 Type B 后缀应该被放置的“桶”的末尾。
        4.  然后，它将 `s-1` 放入该桶的**当前可用位置**（从右向左填充）。
    *   **为什么从右到左？** 因为 Type B 后缀 (`S[i] < S[i+1]`) 的排序依赖于其**后续**后缀。从右到左扫描 `SA` 数组（即从字典序最大的后缀到最小的），可以确保当我们处理后缀 `s` 时，所有比它大的后缀（包括可能影响 `s-1` 排序的 `s`）都已经被扫描过，从而保证了诱导的正确性。
    *   **结果：** 这一轮扫描结束后，所有 Type B\* 和 Type B 后缀都已经被正确地放置在了 `SA` 数组的后半部分（属于它们的桶中）。

*   **阶段二：诱导排序 Type A 后缀 (从 B 推导 A)**
    *   **扫描方向:** **从左到右**扫描 `SA` 数组。
    *   **核心逻辑:**
        1.  算法从 `SA` 数组的开头开始向后扫描。当它遇到一个**已排序**的 Type B 或 B\* 后缀 `s` 时，它再次考察其**前一个位置** `s-1`。
        2.  如果后缀 `s-1` 恰好是一个 **Type A** 后缀，算法就知道 `s-1` 的正确排序位置。
        3.  **如何确定位置？** `s-1` 的第一个字符是 `T[s-1]`。利用 `bucket_A`，算法可以定位到所有以 `T[s-1]` 开头的 Type A 后缀应该被放置的“桶”的开头。
        4.  然后，它将 `s-1` 放入该桶的**当前可用位置**（从左到右填充）。
    *   **为什么从左到右？** 因为 Type A 后缀 (`S[i] > S[i+1]`) 的排序主要由其**第一个字符**决定。从左到右扫描 `SA` 数组（即从字典序最小的后缀到最大的），可以确保我们总是先放置第一个字符较小的 Type A 后缀，这符合基数排序的逻辑，保证了诱导的正确性。
    *   **结果：** 这一轮扫描结束后，所有 Type A 后缀也被正确地放置在了 `SA` 数组的前半部分。至此，整个 `SA` 数组就完全构建完毕了。

*   **位标记 `~s` 的作用**
    *   在两个阶段中，代码都大量使用了 `*j = ~s`。这个按位取反的标记在这里主要用于**区分**一个后缀是“原始的”（直接从B\*排序结果中来的）还是“被诱导出来的”。在诱导的过程中，它也帮助算法识别后缀的类型（例如，在第二阶段，所有正数 `s` 都是前一阶段诱导出的 Type B，而 `~s` 的后缀可能是 Type A）。

### **代码解析**

```c
/* Constructs the suffix array by using the sorted order of type B* suffixes. */
static
void
construct_SA(const sauchar_t *T, sastore_t* SA,
             saidx_t *bucket_A, saidx_t *bucket_B,
             saidx_t n, saidx_t m) {
  sastore_t *i, *j, *k;
  saidx_t s;
  saint_t c0, c1, c2;

  if(0 < m) { // 只有在存在B*后缀时才需要第一阶段
    /* Construct the sorted order of type B suffixes by using
       the sorted order of type B* suffixes. */
    // --- 阶段一：从右到左，诱导排序 Type B 后缀 ---
    for(c1 = ALPHABET_SIZE - 2; 0 <= c1; --c1) {
      // 从后向前遍历所有以字符 c1 为第二个字符的桶
      for(i = SA + BUCKET_BSTAR(c1, c1 + 1), // i 是B*桶的下界
          j = SA + BUCKET_A(c1 + 1) - 1,   // j 是扫描指针，从A桶的上界开始
          k = NULL, c2 = -1;
          i <= j;
          --j) {
        if(0 < (s = *j)) { // 如果 j 指向一个有效的、已排序的后缀 s
          // ... (断言检查) ...
          *j = ~s; // 标记 s 已被处理
          c0 = T[--s]; // c0 是 s-1 的第一个字符
          
          // 检查 s-1 是否是 Type B (T[s-1] <= T[s])
          // (这里的逻辑通过 c0 != c2 和桶的切换来隐式判断)
          if((0 < s) && (T[s - 1] > c0)) { s = ~s; } // 如果 s-1 是 Type A, 标记为负数
          
          if(c0 != c2) { // 如果 s-1 的首字符 c0 与上一个不同
            if(0 <= c2) { BUCKET_B(c2, c1) =(saidx_t)(k - SA); }
            // k 是指向 (c0, c1) 桶的写指针
            k = SA + BUCKET_B(c2 = c0, c1);
          }
          assert(k < j);
          *k-- = s; // 将 s-1 (可能是负数标记的) 放入桶中
        } else {
          *j = ~s; // 标记已被处理
        }
      }
    }
  }

  /* Construct the suffix array by using
     the sorted order of type B suffixes. */
  // --- 阶段二：从左到右，诱导排序 Type A 后缀 ---
  // 首先，将最后一个后缀（Type A）手动放入正确的桶中
  k = SA + BUCKET_A(c2 = T[n - 1]);
  *k++ = (T[n - 2] < c2) ? ~(n - 1) : (n - 1);

  // 从头开始扫描整个 SA 数组
  for(i = SA, j = SA + n; i < j; ++i) {
    if(0 < (s = *i)) { // 如果 i 指向一个有效的、已排序的后缀 s (必然是Type B)
      // ... (断言检查) ...
      c0 = T[--s]; // c0 是 s-1 的第一个字符
      
      // 检查 s-1 是否是 Type A (T[s-1] > T[s])
      if((s == 0) || (T[s - 1] < c0)) { s = ~s; } // 如果 s-1 是 Type B, 标记为负数

      if(c0 != c2) { // 如果 s-1 的首字符 c0 与上一个不同
        BUCKET_A(c2) = (saidx_t)(k - SA);
        // k 是指向 c0 桶的写指针
        k = SA + BUCKET_A(c2 = c0);
      }
      assert(i < k);
      *k++ = s; // 将 s-1 (可能是负数标记的) 放入桶中
    } else { // 之前被标记过的
      assert(s < 0);
      *i = ~s; // 恢复为正数
    }
  }
}
```
