---
title: sssort
categories:
  - hpatch
  - libdivsufsort
tags:
  - hpatch
  - libdivsufsort
abbrlink: f26669ed
date: 2025-10-03 09:12:19
---

[toc]
## sssort **递归核心：优化的子串排序算法**

这是 `divsufsort` 算法的**递归心脏**，也是执行实际比较和排序工作的函数。当 `sort_typeBstar` 将 B\* 后缀按照前两个字符分入不同的“桶”后，`sssort` (Substring Sort) 就被调用来对**每一个桶内部**的后缀进行完全、正确的排序。

### **原理与设计思路解析**

`sssort` 不是一个简单的排序函数，它是一个**混合排序 (Hybrid Sort)** 算法，结合了多种排序策略的优点，以在不同规模的数据上都达到最佳性能。它主要围绕一种优化的归并排序变体来实现。

*   **核心策略：非递归的分块归并排序 (Non-Recursive Block Merge Sort)**
    标准的归并排序是递归的，这会带来函数调用的开销。`sssort` 采用了一种**迭代式**、**自底向上**的实现方式，避免了深层递归。

    1.  **分块 (Blocking):**
        *   算法首先将需要排序的区间 `[first, last)` 分成大小为 `SS_BLOCKSIZE` 的小数据块。`SS_BLOCKSIZE` 是一个编译时常量，通常是一个经过优化的值（比如1024）。
        *   对**每一个小数据块内部**，它使用一种更快的排序算法（见下一点）进行**内部排序**。

    2.  **归并 (Merging):**
        *   在所有小块都内部有序之后，算法开始进行一系列的归并操作。
        *   它会成对地合并相邻的块：先将大小为 `SS_BLOCKSIZE` 的块合并成大小为 `2 * SS_BLOCKSIZE` 的有序块，然后将这些块再合并成 `4 * SS_BLOCKSIZE` 的有序块，以此类推，直到整个区间 `[first, last)` 完全有序。
        *   代码中的 `for(b = a, k = SS_BLOCKSIZE, j = i; j & 1; ...)` 和 `for(k = SS_BLOCKSIZE; i != 0; ...)` 这两层循环，正是通过巧妙的位运算来控制这个“滚雪球”式的归并过程。

*   **混合排序策略**

    1.  **块内排序 (`ss_mintrosort`)**:
        *   对于每个 `SS_BLOCKSIZE` 大小的数据块，算法使用 **IntroSort (内省排序)**。
        *   IntroSort 本身就是一种混合算法，它以**快速排序**开始，但在递归深度过深时（有退化为`O(n^2)`的风险），会自动切换到**堆排序**（保证`O(n log n)`的最坏情况）。而在数据规模变得非常小时，它又会切换到**插入排序**（对于小数据最高效）。
        *   `ss_mintrosort` 是 IntroSort 的一个修改版，专门用于比较后缀（通过 `ss_compare`）。

    2.  **归并策略 (`ss_swapmerge`, `ss_inplacemerge`)**:
        *   **`ss_swapmerge`**: 这是主要的归并函数。它需要一个**额外的缓冲区 `buf`** 来辅助归并操作，以达到最高效率。
        *   **`ss_inplacemerge`**: 当可用的外部缓冲区 `buf` 不足时（`bufsize < limit`），算法会回退到一种**原地归并 (In-place Merge)** 策略。原地归并虽然不需要额外的缓冲区，但其速度通常比使用缓冲区的归并要慢。

*   **对 `lastsuffix` 的特殊处理**
    *   `lastsuffix` 是一个标志，用于指示当前正在排序的这个桶是否包含了整个字符串的**最后一个B\*后缀**。
    *   由于这个后缀的比较行为有特殊性（它后面没有更多字符），算法会先**暂时将其排除**在排序之外 (`++first`)。
    *   在整个桶排序完成后，再通过一个简单的线性插入过程，将这个被排除的后缀放回到它在已排序序列中的**正确位置**。

**总结一下 `sssort` 的完整流程：**

1.  处理 `lastsuffix` 的特殊情况。
2.  决定是使用外部缓冲区还是回退到原地归并模式。
3.  将待排序区间划分为 `SS_BLOCKSIZE` 大小的块。
4.  使用 `ss_mintrosort` 对每个小块进行高效的内部排序。
5.  通过一系列迭代式的 `ss_swapmerge` 或 `ss_inplacemerge` 操作，自底向上地将所有已排序的小块合并成一个完全有序的序列。
6.  最后，将 `lastsuffix`（如果存在）插入回正确的位置。

### **代码解析**

```c
/* Substring sort */
void
sssort(const sauchar_t *T, const sastore_t*PA,
       sastore_t *first, sastore_t*last,
       sastore_t *buf, saidx_t bufsize,
       saidx_t depth, saidx_t n, saint_t lastsuffix) {
  sastore_t *a;
#if SS_BLOCKSIZE != 0 // 这个宏控制了是使用分块归并还是直接调用 introsort
  sastore_t *b, *middle, *curbuf;
  saidx_t j, k, curbufsize, limit;
#endif
  saidx_t i;

  // 1. 特殊情况处理：暂时排除最后一个B*后缀
  if(lastsuffix != 0) { ++first; }

#if SS_BLOCKSIZE == 0
  // 如果分块大小为0，则不使用分块归并，直接对整个区间进行内省排序
  ss_mintrosort(T, PA, first, last, depth);
#else
  // 2. 缓冲区决策：决定是使用外部缓冲区还是原地归并
  if((bufsize < SS_BLOCKSIZE) &&
      (bufsize < (last - first)) &&
      (bufsize < (limit = ss_isqrt((saidx_t)(last - first))))) {
    // 如果提供的缓冲区太小，则计算一个 limit，准备使用原地归并
    if(SS_BLOCKSIZE < limit) { limit = SS_BLOCKSIZE; }
    buf = middle = last - limit, bufsize = limit;
  } else {
    // 否则，使用完整的外部缓冲区
    middle = last, limit = 0;
  }

  // 3. 块内排序与初步归并
  for(a = first, i = 0; SS_BLOCKSIZE < (middle - a); a += SS_BLOCKSIZE, ++i) {
    // 对大小为 SS_BLOCKSIZE 的块 a 进行内部排序
#if SS_INSERTIONSORT_THRESHOLD < SS_BLOCKSIZE
    ss_mintrosort(T, PA, a, a + SS_BLOCKSIZE, depth);
#elif 1 < SS_BLOCKSIZE
    ss_insertionsort(T, PA, a, a + SS_BLOCKSIZE, depth);
#endif
    
    // 确定本次归并操作使用的缓冲区
    curbufsize = (saidx_t)(last - (a + SS_BLOCKSIZE));
    curbuf = a + SS_BLOCKSIZE;
    if(curbufsize <= bufsize) { curbufsize = bufsize, curbuf = buf; }
    
    // 迭代地将当前块 a 与之前已合并的、更大的块进行归并
    for(b = a, k = SS_BLOCKSIZE, j = i; j & 1; b -= k, k <<= 1, j >>= 1) {
      ss_swapmerge(T, PA, b - k, b, b + k, curbuf, curbufsize, depth);
    }
  }

  // 4. 处理剩余部分并完成最终归并
  // 对最后一个不足 SS_BLOCKSIZE 的块进行排序
#if SS_INSERTIONSORT_THRESHOLD < SS_BLOCKSIZE
  ss_mintrosort(T, PA, a, middle, depth);
#elif 1 < SS_BLOCKSIZE
  ss_insertionsort(T, PA, a, middle, depth);
#endif

  // 将所有已合并的大块进行最终的归并
  for(k = SS_BLOCKSIZE; i != 0; k <<= 1, i >>= 1) {
    if(i & 1) {
      ss_swapmerge(T, PA, a - k, a, middle, buf, bufsize, depth);
      a -= k;
    }
  }

  // 5. 如果是原地模式，处理被 limit 分离出的部分
  if(limit != 0) {
    // 先对这部分进行内部排序
#if SS_INSERTIONSORT_THRESHOLD < SS_BLOCKSIZE
    ss_mintrosort(T, PA, middle, last, depth);
#elif 1 < SS_BLOCKSIZE
    ss_insertionsort(T, PA, middle, last, depth);
#endif
    // 然后与主体部分进行原地归并
    ss_inplacemerge(T, PA, first, middle, last, depth);
  }
#endif

  // 6. 将之前排除的 lastsuffix 插入回正确位置
  if(lastsuffix != 0) {
    sastore_t PAi[2]; PAi[0] = PA[*(first - 1)], PAi[1] = n - 2;
    for(a = first, i = *(first - 1);
        (a < last) && ((*a < 0) || (0 < ss_compare(T, &(PAi[0]), PA + *a, depth)));
        ++a) {
      *(a - 1) = *a; // 移动元素，为插入腾出空间
    }
    *(a - 1) = i; // 插入
  }
}
```

## ss_compare **DivSufSort 的核心比较函数**

这是 `divsufsort` 算法中**最底层、最核心**的比较函数。在 `sssort` 及其调用的各种排序算法（如 `ss_mintrosort`, `ss_insertionsort`）内部，每当需要判断两个后缀的字典序大小时，最终都会调用这个 `ss_compare` 函数。它的性能直接决定了整个排序过程的效率。

### **原理与设计思路解析**

`ss_compare` 的设计目标是：**尽可能快地**比较两个由 `p1` 和 `p2` 间接指向的后缀。它充满了针对性能的微观优化。

*   **核心功能：多关键字后缀比较**
    *   **`depth` 参数:** 这是实现**多关键字排序 (Multikey Sort)** 的关键。它告诉函数不要从后缀的起始位置开始比较，而是从**偏移 `depth` 之后**的第一个字符开始。这极大地减少了在递归排序中对已知相同前缀的重复比较。
    *   **比较逻辑:** 函数的主体是一个 `for` 循环，它逐字节地比较两个后缀，只要字符相同，就不断向后移动指针 (`++U1, ++U2`)。当遇到第一个不相同的字符，或者其中一个后缀到达末尾时，循环终止。

*   **关键优化：利用 `PA` 数组进行长度限制**
    *   这是一个非常巧妙且不易理解的优化。函数的参数中有一个 `PA` 数组，这个数组存储的是 B\* 后缀的**原始位置**。
    *   `U1n = T + *(p1 + 1) + 2` 和 `U2n = T + *(p2 + 1) + 2` 这两行代码是在**预计算比较的边界**。
    *   **`*(p1 + 1)` 的含义:** 在 `sssort` 的调用上下文中，`p1` 指向一个存储 B\* 后缀在 `PA` 数组中索引的列表。因此，`PA + *p1` 指向 B\* 后缀的原始位置，而 `PA + *(p1 + 1)` 则指向**下一个 B\* 后缀**的原始位置。
    *   **为什么 `+ 2`?** B\* 后缀的定义保证了它们之间的最小距离至少是1（因为B\*前一个是A），并且比较通常关注前两个字符之后的差异。这个 `+2` 提供了一个安全的、经过经验优化的比较上限。
    *   **最终效果：** 这种设计使得比较**不会**无限制地进行到整个字符串的末尾。它实际上是在比较两个 B\* 后缀**直到下一个 B\* 后缀出现之前**的这部分子串。这是一种启发式的**局部比较**，它基于一个假设：如果两个 B\* 后缀在这段“有代表性”的区域内都无法分出胜负，那么它们很可能是相同的，可以留到后续的排名阶段去处理。这避免了在非常长的公共前缀上浪费时间。

*   **返回值的含义**
    *   函数返回一个 `saint_t` (有符号整数)，其值遵循 `strcmp` 的约定：
        *   **> 0:** 后缀1 > 后缀2
        *   **< 0:** 后缀1 < 后缀2
        *   **== 0:** 后缀1 == 后缀2 (在比较范围内)

*   **`INLINE` 关键字**
    *   `INLINE` (通常是 `__inline__` 或 `inline` 的宏) 是给编译器的强烈建议，将这个函数的代码**直接嵌入**到调用它的地方（如 `ss_insertionsort` 的循环内部），而不是通过常规的函数调用（这会有压栈、跳转等开销）。
    *   对于这种在最内层循环中被海量调用的、短小的函数，内联是至关重要的性能优化。

### **代码解析**

```c
/* Compares two suffixes. */
// INLINE 提示编译器进行内联优化
static INLINE
saint_t
ss_compare(const sauchar_t *T,
           const sastore_t*p1, const sastore_t*p2,
           saidx_t depth) {
  const sauchar_t *U1, *U2, *U1n, *U2n; // 定义四个指针用于比较

  // 初始化比较指针：
  // U1, U2: 分别指向两个后缀从 depth 偏移量开始的比较起始位置
  for(U1 = T + depth + *p1,
      U2 = T + depth + *p2,
      // U1n, U2n: 利用 PA 数组计算出比较的“安全”上限
      // *(p1 + 1) 是 p1 指向的 B* 后缀在 PA 数组中的下一个 B* 后缀的索引
      U1n = T + *(p1 + 1) + 2,
      U2n = T + *(p2 + 1) + 2;
      
      // 循环条件：
      // 1. U1, U2 都没有越过各自的比较上限
      // 2. 并且当前指向的字符相同
      (U1 < U1n) && (U2 < U2n) && (*U1 == *U2);
      
      // 如果条件满足，则两个指针都前进一格
      ++U1, ++U2) {
  }

  // 循环结束后，根据指针的状态和指向的字符值来决定最终的比较结果
  return U1 < U1n ? // 如果后缀1没有到达其比较上限...
        (U2 < U2n ? *U1 - *U2 : 1) : // ...再看后缀2。如果也没到上限，则比较当前不同的字符；如果到了，说明后缀2是前缀，后缀1更大。
        (U2 < U2n ? -1 : 0);          // 如果后缀1到达了上限，再看后缀2。如果没到，说明后缀1是前缀，更小；如果都到了，说明它们在比较范围内相等。
}
```

## ss_insertionsort **为小数据组优化的插入排序**

这是 `ss_mintrosort` 在其混合排序策略的最后阶段调用的函数。当待排序的数据块规模变得非常小（小于等于 `SS_INSERTIONSORT_THRESHOLD`）时，`ss_mintrosort` 会放弃复杂的快速排序或堆排序，转而调用这个**高度特化**的插入排序函数。

### **原理与设计思路解析**

虽然名为插入排序，但它并非教科书上简单的实现。它经过了深度定制，以适应 `divsufsort` 算法的内部工作机制，特别是其对**相等后缀组**的处理。

*   **核心策略：标准插入排序框架**
    *   算法的整体结构遵循经典的**插入排序**。它从右到左遍历数组（`for(i = last - 2; ...)`），将每个元素 `t = *i` 看作是待插入的“牌”。
    *   对于每张“牌”`t`，它向右进行一个内部循环（`for(...; 0 < (r = ss_compare(...));)`)，找到 `t` 应该被插入的正确位置。
    *   在寻找位置的过程中，它将所有比 `t` 大的元素都向右移动一格，为 `t` 腾出空间。

*   **关键优化与特殊设计**

    1.  **后缀比较而非整数比较:**
        *   排序的依据不是数组中存储的整数值本身，而是这些整数作为**索引**所指向的**后缀子串**。
        *   所有的比较都通过 `ss_compare(T, PA + t, PA + *j, depth)` 来完成。这个函数会从第 `depth` 个字符开始，比较后缀 `t` 和后缀 `*j` 的字典序。

    2.  **相等后缀的识别与标记 (核心特性):**
        *   这是此函数最关键的特殊之处。它不仅仅在排序，它还在**识别并标记**那些彼此完全相等的后缀。
        *   当 `ss_compare` 返回 `r == 0` 时，意味着后缀 `t` 和后缀 `*j` 是完全相同的。
        *   此时，代码执行 `*j = ~*j;`。这个**按位取反**的操作会将一个非负整数变成一个负数。
        *   这个负数就成了一个**标记**，告诉 `divsufsort` 的上层调用者：“这个位置的后缀 `*j` 与它前一个位置的后缀是相同的”。这个信息对于后续的**排名计算 (`sort_typeBstar`中的阶段四)** 至关重要。这是一种在原地（in-place）传递额外信息的、非常节省内存的技巧。

    3.  **分组移动 (Group Shifting):**
        *   内部的移位循环 `do { *(j - 1) = *j; } while((++j < last) && (*j < 0));` 也不是一个简单的单元素移动。
        *   `(*j < 0)` 这个条件检查的就是上一步留下的“相等标记”。
        *   这个 `do-while` 循环的含义是：“将 `*j` 移到 `*(j-1)`，然后只要下一个元素 `*j` 还是一个标记（负数），就继续移动”。
        *   这实际上实现了**对整个已标记的“相等后缀组”的一次性、连续的移动**，而不是逐个移动组内元素，从而提高了效率。

**为什么对小数据使用插入排序？**

*   **低开销:** 快速排序和堆排序虽然在大数据量上表现优异，但它们的函数调用、分区或建堆操作本身有固定的开销。对于只有十几个元素的小数组，这些开销占比很高。
*   **缓存友好:** 插入排序是线性扫描，具有优秀的**缓存局部性**，在现代CPU上执行效率很高。
*   **对近乎有序的数据高效:** 当数据已经部分有序时，插入排序的性能接近 `O(n)`。在递归排序的后期，小数据块往往已经具有一定的有序性。

### **代码解析**

```c
#if (SS_BLOCKSIZE != 1) && (SS_INSERTIONSORT_THRESHOLD != 1)

/**
 * @brief 对小规模数据组进行插入排序，并标记相等的后缀。
 * @param T, PA      用于后缀比较的上下文。
 * @param first, last  待排序的区间 [first, last)。
 * @param depth      多关键字比较的起始深度。
 */
static
void
ss_insertionsort(const sauchar_t *T, const sastore_t *PA,
                 sastore_t *first, sastore_t *last, saidx_t depth) {
  sastore_t *i, *j;
  saidx_t t;
  saint_t r; // 用于存储比较结果

  // 外层循环：从倒数第二个元素开始，依次向前处理每个元素
  for(i = last - 2; first <= i; --i) {
    t = *i; // t 是当前需要插入的元素（“牌”）
    
    // 内层循环：从 i+1 开始向右，为 t 寻找正确的插入位置 j
    // 循环条件：只要 t 比 j 指向的后缀大 (r > 0)，就继续向右寻找
    for(j = i + 1; 0 < (r = ss_compare(T, PA + t, PA + *j, depth));) {
      
      // 移位操作：将比 t 大的元素（或元素组）向右移动
      do { 
        *(j - 1) = *j; 
      } while((++j < last) && (*j < 0)); // 如果下一个元素也是标记过的（负数），则连续移动
      
      if(last <= j) { break; } // 到达末尾，停止寻找
    }
    
    // 关键：处理相等的情况
    if(r == 0) { 
      // 如果 t 和 *j 指向的后缀相等，则将 *j 标记为负数
      *j = ~*j; 
    }
    
    // 将 t 插入到找到的正确位置
    *(j - 1) = t;
  }
}

#endif /* (SS_BLOCKSIZE != 1) && (SS_INSERTIONSORT_THRESHOLD != 1) */
```

## ss_heapsort & ss_fixdown **最坏情况保障：堆排序**

这是 `ss_mintrosort` 内省排序算法的**第三个组成部分**，也是其性能稳定性的**安全保障**。当 `ss_mintrosort` 检测到快速排序的递归深度过深（有退化为`O(n^2)`的风险）时，它会立即放弃快速排序，转而调用这个 `ss_heapsort` 函数来对当前数据块进行排序。

### **原理与设计思路解析**

这组函数实现了一个经典的**堆排序 (Heapsort)** 算法。堆排序的核心是利用了“堆”这种数据结构的特性。

*   **什么是堆 (Heap)？**
    *   在这里，我们讨论的是**最大堆 (Max-Heap)**。一个最大堆是一个概念上的二叉树，它满足一个性质：**任何父节点的值都大于或等于其子节点的值**。
    *   在数组实现中，如果一个节点的索引是 `i`，那么它的左子节点索引是 `2*i + 1`，右子节点是 `2*i + 2`。

*   **`ss_heapsort` (堆排序主流程)**
    堆排序主要分两个阶段：

    1.  **建堆 (Heapify):**
        *   `for(i = m / 2 - 1; 0 <= i; --i) { ss_fixdown(Td, PA, SA, i, m); }`
        *   这一步是将一个无序的数组 `SA` 转换成一个满足最大堆性质的数组。
        *   它从最后一个非叶子节点 (`m/2 - 1`) 开始，向前遍历到根节点 (`0`)，对每个节点都调用一次 `ss_fixdown`。`ss_fixdown` 会确保以 `i` 为根的子树满足最大堆性质。当这个循环结束后，整个数组就变成了一个最大堆。
        *   此时，数组的第一个元素 `SA[0]` 就是整个数组中的**最大值**（即字典序最大的后缀）。

    2.  **排序 (Sorting):**
        *   `for(i = m - 1; 0 < i; --i) { ... }`
        *   这个循环不断地从堆中取出最大元素，并放到它最终应该在的位置（数组的末尾）。
        *   **步骤 a:** `SWAP(SA[0], SA[i])` (在 `divsufsort` 的实现中，通过变量 `t` 完成了交换)。将当前堆的根（最大元素）与堆的最后一个元素交换。这样，最大的元素就被放到了数组的末尾 `i`，这个位置就**排好序**了。
        *   **步骤 b:** `ss_fixdown(Td, PA, SA, 0, i)`。交换后，新的根节点 `SA[0]` 很可能破坏了堆的性质。因此，需要再次调用 `ss_fixdown`，从根节点开始，向下调整，让新的最大值“浮”到根部，从而在 `[0, i-1]` 的范围内恢复堆的性质。
        *   这个过程不断重复，每次都将当前最大的元素放到已排序部分的开头，堆的大小减一，直到整个数组排序完成。

*   **`ss_fixdown` (向下调整)**
    *   这是维护堆性质的核心辅助函数，也叫 "heapify down" 或 "sift down"。
    *   当一个节点 `i` 的值可能小于其子节点时，`ss_fixdown` 被调用。
    *   它会比较节点 `i` 与其左右子节点的值，找出三者中的最大值。
    *   如果最大的值不在节点 `i`，它就将最大值的子节点与节点 `i` 进行交换。
    *   然后，它会以被交换下去的那个子节点为新起点，**递归地继续向下调整**，直到当前子树重新满足最大堆的性质。

*   **比较的特殊性**
    *   和 `ss_insertionsort` 一样，这里的比较并不是简单的整数比较。
    *   所有大小判断都是通过 `Td[PA[SA[i]]]` 来获取后缀在 `depth` 偏移处的字符值。它只比较**单个字符**来维护堆的性质，这使得堆排序在这个多关键字排序的框架下非常高效。

**为什么需要堆排序？**

*   **最坏情况保障:** 堆排序的时间复杂度稳定在 `O(n log n)`，它没有任何会导致性能退化的“病态”输入。
*   **安全阀:** 在 `ss_mintrosort` 中，它充当了一个“安全阀”。`ss_mintrosort` 可以尽情地使用平均性能极佳的快速排序，一旦发现有性能退化的风险，就立刻切换到堆排序，保证了整个排序过程的性能下限，使得算法既快速又稳健。

### **代码解析**

```c
#if (SS_BLOCKSIZE == 0) || (SS_INSERTIONSORT_THRESHOLD < SS_BLOCKSIZE)

/**
 * @brief 堆排序的“向下调整”辅助函数。
 * @param Td, PA 用于比较的上下文。
 * @param SA     堆所在的数组。
 * @param i      需要向下调整的节点的起始索引。
 * @param size   当前堆的有效大小。
 */
static INLINE
void
ss_fixdown(const sauchar_t *Td, const sastore_t *PA,
           sastore_t* SA, saidx_t i, saidx_t size) {
  saidx_t j, k;
  saidx_t v; // 存储节点i的原始值
  saint_t c, d, e; // 用于存储比较的字符值

  // v是待下沉的元素，c是它的字符值
  for(v = SA[i], c = Td[PA[v]]; 
      (j = 2 * i + 1) < size; // j是左子节点，只要存在子节点就循环
      SA[i] = SA[k], i = k) { // 将选出的较大子节点k的值上移到父节点i，然后继续从k向下看
    
    k = j++; // k先指向左子节点，然后j指向右子节点
    d = Td[PA[SA[k]]]; // d是左子节点的字符值
    
    // 比较左右子节点，找出较大的那个
    if(d < (e = Td[PA[SA[j]]])) { k = j; d = e; }
    
    // 如果父节点比两个子节点都大（或等于），则堆性质已满足，停止下沉
    if(d <= c) { break; }
  }
  // 将原始值v放到最终下沉到的位置
  SA[i] = v;
}

/* Simple top-down heapsort. */
/**
 * @brief 堆排序主函数。
 * @param Td, PA 用于比较的上下文。
 * @param SA     待排序的数组。
 * @param size   数组大小。
 */
static
void
ss_heapsort(const sauchar_t *Td, const sastore_t*PA, sastore_t* SA, saidx_t size) {
  saidx_t i, m;
  saidx_t t;

  m = size;
  // 对数组大小为偶数的情况进行一个微小的预处理，这可能是一个边界情况的优化
  if((size % 2) == 0) {
    m--;
    if(Td[PA[SA[m / 2]]] < Td[PA[SA[m]]]) { SWAP(SA[m], SA[m / 2]); }
  }

  // --- 阶段一：建堆 (Heapify) ---
  // 从最后一个非叶子节点开始，向前对每个节点执行向下调整
  for(i = m / 2 - 1; 0 <= i; --i) { ss_fixdown(Td, PA, SA, i, m); }
  
  // 再次处理偶数大小的边界情况
  if((size % 2) == 0) { SWAP(SA[0], SA[m]); ss_fixdown(Td, PA, SA, 0, m); }
  
  // --- 阶段二：排序 ---
  // 循环n-1次，每次将堆顶（最大元素）换到末尾
  for(i = m - 1; 0 < i; --i) {
    t = SA[0]; SA[0] = SA[i]; // 将堆顶与当前堆的末尾交换
    ss_fixdown(Td, PA, SA, 0, i); // 对大小为i的新堆，从根节点恢复堆性质
    SA[i] = t; // 将取出的最大元素放到已排序的位置
  }
}

#endif
```

## ss_mintrosort **深度优化的内省排序**

这是 `sssort` 内部调用的、用于对中等大小数据块（`SS_BLOCKSIZE`）进行排序的核心算法。函数名 `mintrosort` 意为 **Multikey Introspective Sort** (多关键字内省排序)，它是一种经过深度定制和优化的**内省排序**算法，专门用于对后缀数组进行排序。

### **原理与设计思路解析**

标准的内省排序（IntroSort）结合了快速排序、堆排序和插入排序的优点。`ss_mintrosort` 在此基础上，针对后缀排序的特殊性进行了多项关键优化。

*   **核心策略：混合排序 (Hybrid Sort)**
    这是一个非递归的实现，通过一个手动管理的栈 (`stack`) 来模拟递归。

    1.  **快速排序 (Quicksort) 阶段:**
        *   算法的主体是基于**三路快速排序 (3-Way Quicksort)**，也称为**荷兰国旗问题**的变体。这种排序方式非常适合处理有大量重复元素的场景（比如，很多后缀可能以相同的字符开头）。
        *   它一次划分会将数据分为三部分：`< pivot` (小于主元), `= pivot` (等于主元), `> pivot` (大于主元)。
        *   之后，它只需对 `< pivot` 和 `> pivot` 两部分进行递归排序。对于 `= pivot` 的部分，由于它们的当前字符都相同，只需将它们的**递归深度 `depth` 加一**，然后对这整个部分进行下一轮排序。这被称为**多关键字排序 (Multikey Sort)**，因为它有效地利用了字符串的结构，避免了不必要的重复比较。

    2.  **堆排序 (Heapsort) 阶段 (最坏情况保障):**
        *   快速排序在处理特定数据（如已排序或逆序）时，有退化为 `O(n^2)` 的风险。
        *   `ss_mintrosort` 通过一个 `limit` 计数器来跟踪递归深度。如果 `limit` 减到0（意味着递归太深），算法会立即**切换到堆排序 (`ss_heapsort`)**。
        *   堆排序能保证 `O(n log n)` 的最坏情况时间复杂度，从而避免了快速排序的性能陷阱。

    3.  **插入排序 (Insertion Sort) 阶段 (小数据优化):**
        *   当待排序的数据块大小小于等于一个阈值 `SS_INSERTIONSORT_THRESHOLD` 时，算法会切换到**插入排序 (`ss_insertionsort`)**。
        *   对于小规模数据，插入排序由于其简单的循环结构和良好的缓存局部性，通常比复杂的快速排序或堆排序更快。

*   **关键优化与实现细节**

    1.  **非递归实现 (`stack`):**
        *   为了避免深层递归导致的栈溢出风险，并可能获得更好的性能，该函数使用一个固定大小的数组 `stack` 和 `STACK_PUSH`/`STACK_POP` 宏来手动模拟函数调用栈。这使得算法的控制流变得复杂，但更健壮、更高效。

    2.  **多关键字处理 (`depth`)**:
        *   `depth` 参数是多关键字排序的核心。它告诉比较函数应该从后缀的第 `depth` 个字符开始比较，而不是每次都从头开始。
        *   在处理 `= pivot` 的分区时，只需增加 `depth` 并对整个分区进行下一轮排序，这大大提高了效率。`Td = T + depth;` 这一行就是用于快速定位到比较的起始字符。

    3.  **主元选择 (`ss_pivot`)**:
        *   快速排序的性能严重依赖于主元 (pivot) 的选择。一个好的主元可以将数据均匀地划分。
        *   `ss_pivot` 函数（此处未显示）通常会实现一种健壮的主元选择策略，如**三数取中法 (Median-of-Three)**，以避免选择到最大或最小的元素。

    4.  **复杂的分区逻辑 (`partition`)**:
        *   代码中的 `partition` 部分极其复杂。它不仅实现了三路划分，还进行了大量的交换 (`SWAP`) 操作，以原地方式将 `< pivot`, `= pivot`, `> pivot` 三个部分整理到位的正确位置。
        *   `for(e = first, f = b - s; ...)` 等循环是在执行经典的**块交换 (Block Swap)** 操作，用于将不同的分区移动到最终位置。

### **代码解析**

```c
/* Multikey introsort for medium size groups. */
static
void
ss_mintrosort(const sauchar_t *T, const sastore_t *PA,
              sastore_t *first, sastore_t *last,
              saidx_t depth) {
#define STACK_SIZE SS_MISORT_STACKSIZE
  // 手动管理的递归栈
  struct { sastore_t*a, *b, c; saint_t d; } stack[STACK_SIZE];
  const sauchar_t *Td;
  sastore_t*a, *b, *c, *d, *e, *f;
  saidx_t s, t;
  saint_t ssize;
  saint_t limit; // 用于防止快速排序退化的递归深度限制
  saint_t v, x = 0;

  for(ssize = 0, limit = ss_ilg((saidx_t)(last - first));;) { // 主循环，模拟递归

    // 3. 小数据优化：切换到插入排序
    if((last - first) <= SS_INSERTIONSORT_THRESHOLD) {
#if 1 < SS_INSERTIONSORT_THRESHOLD
      if(1 < (last - first)) { ss_insertionsort(T, PA, first, last, depth); }
#endif
      STACK_POP(first, last, depth, limit); // 从栈中弹出下一个待排序的任务
      continue;
    }

    // Td 指向当前需要比较的字符位置（基于递归深度 depth）
    Td = T + depth;
    
    // 2. 最坏情况保障：切换到堆排序
    if(limit-- == 0) { 
        ss_heapsort(Td, PA, first, (saidx_t)(last - first)); 
        STACK_POP(first, last, depth, limit);
        continue;
    }
    // ... 对已排序的小分区进行特殊处理 ...

    // --- 1. 快速排序阶段 ---
    /* choose pivot */
    // 选择主元
    a = ss_pivot(Td, PA, first, last);
    v = Td[PA[*a]]; // 主元的值（一个字符）
    SWAP(*first, *a); // 将主元放到区域的开头

    /* partition */
    // 极其复杂的三路分区逻辑
    // a, b, c, d 是四个指针，用于将数组划分为：
    // [first...a-1]  (等于 pivot)
    // [a...b-1]      (小于 pivot)
    // [b...c]        (未处理)
    // [c+1...d]      (大于 pivot)
    // [d+1...last-1] (等于 pivot)
    for(b = first; (++b < last) && ((x = Td[PA[*b]]) == v);) { }
    if(((a = b) < last) && (x < v)) {
      for(; (++b < last) && ((x = Td[PA[*b]]) <= v);) {
        if(x == v) { SWAP(*b, *a); ++a; }
      }
    }
    for(c = last; (b < --c) && ((x = Td[PA[*c]]) == v);) { }
    if((b < (d = c)) && (x > v)) {
      for(; (b < --c) && ((x = Td[PA[*c]]) >= v);) {
        if(x == v) { SWAP(*c, *d); --d; }
      }
    }
    for(; b < c;) {
      SWAP(*b, *c);
      for(; (++b < c) && ((x = Td[PA[*b]]) <= v);) {
        if(x == v) { SWAP(*b, *a); ++a; }
      }
      for(; (b < --c) && ((x = Td[PA[*c]]) >= v);) {
        if(x == v) { SWAP(*c, *d); --d; }
      }
    }

    if(a <= d) {
      c = b - 1;
      
      // 使用块交换将三个分区移动到最终位置
      if((s = (saidx_t)(a - first)) > (t = (saidx_t)(b - a))) { s = t; }
      for(e = first, f = b - s; 0 < s; --s, ++e, ++f) { SWAP(*e, *f); }
      if((s = (saidx_t)(d - c)) > (t = (saidx_t)(last - d - 1))) { s = t; }
      for(e = b, f = last - s; 0 < s; --s, ++e, ++f) { SWAP(*e, *f); }

      // a, c, b 现在是三个分区的边界
      a = first + (b - a);
      c = last - (d - c);
      b = (v <= Td[PA[*a] - 1]) ? a : ss_partition(PA, a, c, depth);

      // --- 将新的子任务压入手动管理的栈 ---
      // 这是一个复杂的逻辑，目的是总是先处理较小的分区，以控制栈的深度
      if((a - first) <= (last - c)) {
        if((last - c) <= (c - b)) {
          STACK_PUSH(b, c, depth + 1, ss_ilg((saidx_t)(c - b)));
          STACK_PUSH(c, last, depth, limit);
          last = a;
        } else if((a - first) <= (c - b)) {
          STACK_PUSH(c, last, depth, limit);
          STACK_PUSH(b, c, depth + 1, ss_ilg((saidx_t)(c - b)));
          last = a;
        } else {
          // ... (其他压栈组合) ...
        }
      } else {
        // ... (其他压栈组合) ...
      }
    } else { // 如果所有元素都等于主元
      limit += 1;
      // ...
      depth += 1; // 只需增加深度，对整个分区进行下一轮排序
    }
  }
#undef STACK_SIZE
}
```

## ss_pivot & median helpers **健壮的主元选择**

这是 `ss_mintrosort` 内省排序算法中至关重要的一环。这组函数的核心任务是为快速排序的**分区 (partition)** 操作选择一个高质量的**主元 (pivot)**。一个好的主元选择是避免快速排序退化到 `O(n^2)` 最坏情况的关键。

### **原理与设计思路解析**

快速排序的性能严重依赖于主元的选择。如果每次都选到最大或最小的元素，分区就会极度不平衡，导致算法性能急剧下降。为了解决这个问题，`ss_pivot` 实现了一种**自适应的、基于采样的**主元选择策略。

*   **核心策略：中位数取样 (Median Sampling)**
    该策略的基本思想是：不随机选择主元，而是从待排序的序列中选取少数几个元素，然后用这几个元素的**中位数**作为最终的主元。这个中位数更有可能接近整个序列的真实中位数，从而产生更平衡的划分。

*   **自适应策略 (Adaptive Strategy)**
    `ss_pivot` 函数会根据待排序数组的大小 `t`，动态地选择不同的采样策略，在采样质量和计算开销之间取得平衡：

    1.  **小数组 (`t <= 32`): 三数取中 (`ss_median3`)**
        *   对于非常小的数组，随机选择导致性能退化的概率很低。
        *   算法只取序列的**第一个、中间、最后一个**元素，然后调用 `ss_median3` 找出这三个数的中位数。这是一种开销极小且非常有效的策略。

    2.  **中等数组 (`32 < t <= 512`): 五数取中 (`ss_median5`)**
        *   随着数组增大，从更多样本中选择可以得到更高质量的主元。
        *   算法会从序列中均匀地选取**五个**元素（`first`, `first+t/4`, `middle`, `last-1-t/4`, `last-1`），然后调用 `ss_median5` 找出这五个数的中位数。

    3.  **大数组 (`t > 512`): 九数取中 / 中位数的中位数 (`Median of Medians of Three`)**
        *   对于非常大的数组，需要一种更稳健的策略。这里采用了一种被称为**“中位数的中位数”**的变体。
        *   **步骤 a:** 将数组逻辑上分为**开头、中间、末尾**三个区域。
        *   **步骤 b:** 在每个区域内部，都使用**三数取中** (`ss_median3`) 找出一个局部的中位数。
        *   **步骤 c:** 最后，再对这**三个局部的中位数**调用一次 `ss_median3`，找出最终的主元。
        *   这种方法从9个采样点中选出主元，极大地降低了选出“坏”主元的概率，为大规模数据的分区平衡提供了强有力的保障。

*   **`ss_median3` 和 `ss_median5` 的实现**
    这两个函数通过一系列精心安排的比较和交换 (`SWAP`) 操作，以最少的比较次数找出3个或5个元素的中位数。它们本质上是小型的**排序网络 (Sorting Network)**。

### **代码解析**

```c
/*---------------------------------------------------------------------------*/

/**
 * @brief 从三个给定的元素中找出中位数。
 * @param v1, v2, v3 指向三个元素的指针。
 * @return 指向中位数元素的指针。
 */
static INLINE
sastore_t*
ss_median3(const sauchar_t *Td, const sastore_t*PA,
           sastore_t*v1, sastore_t*v2, sastore_t*v3) {
  sastore_t*t;
  // 比较和交换，类似于对三个元素进行部分排序
  if(Td[PA[*v1]] > Td[PA[*v2]]) { SWAP(v1, v2); }
  if(Td[PA[*v2]] > Td[PA[*v3]]) { // 此时 v2 是 v1,v2,v3 中的最大值
    // 只需再比较 v1 和 v3 即可
    if(Td[PA[*v1]] > Td[PA[*v3]]) { return v1; } // v1 > v3, v3是中位数
    else { return v3; } // v1 <= v3, v1是中位数
  }
  return v2; // v2 <= v3, v2是中位数
}

/**
 * @brief 从五个给定的元素中找出中位数。
 */
static INLINE
sastore_t*
ss_median5(const sauchar_t *Td, const sastore_t*PA,
           sastore_t*v1, sastore_t*v2, sastore_t*v3, sastore_t*v4, sastore_t*v5) {
  sastore_t *t;
  // 通过一系列比较和交换，有效地找出5个数的中位数
  if(Td[PA[*v2]] > Td[PA[*v3]]) { SWAP(v2, v3); }
  if(Td[PA[*v4]] > Td[PA[*v5]]) { SWAP(v4, v5); }
  if(Td[PA[*v2]] > Td[PA[*v4]]) { SWAP(v2, v4); SWAP(v3, v5); }
  if(Td[PA[*v1]] > Td[PA[*v3]]) { SWAP(v1, v3); }
  if(Td[PA[*v1]] > Td[PA[*v4]]) { SWAP(v1, v4); SWAP(v3, v5); }
  if(Td[PA[*v3]] > Td[PA[*v4]]) { return v4; }
  return v3;
}

/**
 * @brief 根据待排序数组的大小，自适应地选择一个高质量的主元。
 * @param first, last 待排序的区间。
 * @return 指向被选为主元的元素的指针。
 */
static INLINE
sastore_t*
ss_pivot(const sauchar_t *Td, const sastore_t*PA, sastore_t*first, sastore_t*last) {
  sastore_t*middle;
  saidx_t t;

  t = (saidx_t)(last - first); // 计算数组大小
  middle = first + t / 2;    // 计算中间位置

  // 自适应策略：
  if(t <= 512) {
    if(t <= 32) {
      // 小数组 (<=32): 三数取中
      return ss_median3(Td, PA, first, middle, last - 1);
    } else {
      // 中等数组 (33-512): 五数取中
      t >>= 2; // t = t / 4
      return ss_median5(Td, PA, first, first + t, middle, last - 1 - t, last - 1);
    }
  }
  // 大数组 (>512): 九数取中 (中位数的中位数)
  t >>= 3; // t = t / 8, 用于在三个区域内采样
  // 1. 找出开头区域的中位数
  first  = ss_median3(Td, PA, first, first + t, first + (t << 1));
  // 2. 找出中间区域的中位数
  middle = ss_median3(Td, PA, middle - t, middle, middle + t);
  // 3. 找出末尾区域的中位数
  last   = ss_median3(Td, PA, last - 1 - (t << 1), last - 1 - t, last - 1);
  // 4. 从这三个中位数中，再找出中位数作为最终的主元
  return ss_median3(Td, PA, first, middle, last);
}
```

## ss_partition **特殊情况下的二路分区**

这是一个非常特殊且高度优化的**分区 (Partition)** 函数，它并不是 `ss_mintrosort` 中主要的三路分区逻辑，而是一个在特定情况下被调用的**二路分区**辅助工具。

### **原理与设计思路解析**

`ss_partition` 的设计目标是在一个**已经部分有序**的、并且**只包含两种类型元素**的序列中，快速地将这两类元素分开。

*   **调用场景:**
    这个函数在 `ss_mintrosort` 的两个特定、相对少见的场景中被调用：
    1.  当 `limit < 0` 时，表明堆排序已经执行过，序列已经**宏观有序**。但可能存在一些**相等的前缀组**需要被进一步细分。
    2.  当三路分区后，所有元素都等于主元 `v` 时，需要根据**下一个字符（`depth+1`）**来对这个等价类进行再次划分。

*   **核心逻辑：Hoare 分区方案的变体**
    该函数实现了类似于**Hoare 分区方案 (Hoare Partition Scheme)** 的思想。
    *   它使用两个指针 `a` 和 `b`，分别从序列的**两端向中间移动**。
    *   指针 `a` 从左向右 (`++a`) 寻找一个“错位”的元素。
    *   指针 `b` 从右向左 (`--b`) 寻找另一个“错位”的元素。
    *   一旦两个指针都找到了错位的元素，并且 `a < b`，就将这两个元素**交换**。
    *   当 `b <= a` 时，两个指针交错或相遇，分区完成。

*   **特殊的比较条件 (`(PA[*a] + depth) >= (PA[*a + 1] + 1)`)**
    *   这是理解此函数最关键、也最费解的部分。它**并不是**在比较后缀的内容。
    *   **上下文:** 调用此函数的序列 `[first, last)` 中的所有后缀，在 `depth` 深度上都具有**相同的前缀**。
    *   **`PA` 数组的含义:** `PA` 数组存储的是 **B\* 后缀的原始位置**。
    *   **比较的真正含义:** 这个条件实际上是在检查一个**结构性属性**。它在判断后缀 `*a` 是否是 B\* 后缀 `*(a+1)` 的**直接前驱**（即 `PA[*a] == PA[*(a+1)] - 1`）。更广义地说，它是在区分两种后缀：
        1.  **第一类:** 那些是另一个 B\* 后缀的直接前驱的后缀。
        2.  **第二类:** 所有其他后缀。
    *   它利用了这个结构性特征来将序列一分为二，这是一种非常高效的划分方式，因为它完全避免了昂贵的字符串比较。

*   **使用按位取反 `~` 进行标记**
    *   `*a = ~*a;`: 当指针 `a` 找到一个满足条件的元素（第一类）时，它会先通过**按位取反**将其标记为负数。
    *   `t = ~*b;`: 在交换之前，它会对从 `b` 位置取出的元素 `*b` 也进行一次取反。
    *   **目的:** 这种标记有两个作用：
        1.  它可能用于在分区结束后，快速地将所有被标记的元素恢复原状 (`if(first < a) { *first = ~*first; }`)。
        2.  更重要的是，这种标记方式与 `ss_insertionsort` 中的标记是兼容的，它们共同构成了 `divsufsort` 内部一套复杂的、用于在原地传递额外信息的编码系统。

### **代码解析**

```c
/**
 * @brief 对一个具有特定结构的部分有序序列进行二路分区。
 * @param PA           指向 B* 后缀原始位置的数组。
 * @param first, last  待分区的区间 [first, last)。
 * @param depth        当前的比较深度。
 * @return sastore_t*  返回分区点指针 a，使得 [first, a) 和 [a, last) 分别属于两类。
 */
static INLINE
sastore_t*
ss_partition(const sastore_t*PA,
             sastore_t*first, sastore_t*last, saidx_t depth) {
  sastore_t *a, *b; // a, b 是左右两个扫描指针
  saidx_t t;

  // 初始化指针 a 和 b，分别在区间的左边界之前和右边界
  for(a = first - 1, b = last;;) {
    
    // 指针 a 从左向右扫描
    // 寻找第一个不满足 (PA[*a] + depth) >= (PA[*a + 1] + 1) 条件的元素
    // 满足条件的元素会被原地标记为负数
    for(; (++a < b) && ((PA[*a] + depth) >= (PA[*a + 1] + 1));) { *a = ~*a; }
    
    // 指针 b 从右向左扫描
    // 寻找第一个满足 (PA[*b] + depth) < (PA[*b + 1] + 1) 条件的元素
    for(; (a < --b) && ((PA[*b] + depth) <  (PA[*b + 1] + 1));) { }
    
    // 如果两个指针交错或相遇，则分区完成
    if(b <= a) { break; }
    
    // 交换 a 和 b 指向的错位元素
    // 注意：在交换前，对 b 指向的元素进行一次取反操作
    t = ~*b;
    *b = *a;
    *a = t;
  }
  
  // 分区结束后，恢复第一个分段中所有被标记的元素
  if(first < a) { *first = ~*first; }

  // 返回分区点
  return a;
}
```

## ss_ilg **快速整数对数计算**

这个函数 `ss_ilg` (Integer Logarithm) 是一个用于快速计算**整数以2为底的对数**（即 `floor(log2(n))`）的辅助函数。在 `ss_mintrosort` 中，这个值被用来确定快速排序的最大允许递归深度 `limit`，从而有效地防止算法性能退化。

### **原理与设计思路解析**

计算对数通常是一个相对耗时的操作。为了在性能敏感的排序算法中避免这种开销，`ss_ilg` 采用了几种高度优化的、基于**预计算**和**位运算**的技巧。

*   **核心策略：查表法 (Lookup Table)**
    该函数的核心是 `lg_table` 这个静态常量数组。
    *   **`lg_table` 的内容:** 这个表预先存储了从0到255这256个整数的以2为底的对数。例如，`lg_table[1]`=0, `lg_table[2]`=1, `lg_table[3]`=1, `lg_table[4]`=2, `lg_table[31]`=4, `lg_table[32]`=5, ...
    *   **优势:** 通过牺牲少量内存（256个整数），可以将一个可能需要循环或复杂指令的对数运算，转换成一次**极快的数组索引操作**。

*   **编译时配置与自适应实现**
    函数通过预处理器宏 `#if`，根据编译时设置的 `SS_BLOCKSIZE`，选择最合适的实现方式：

    1.  **`SS_BLOCKSIZE == 0` (TRSort 模式):**
        *   在这种模式下，算法会调用另一个（此处未显示的）函数 `tr_ilg(n)`。这表明 `trsort` 算法有自己的一套整数对数计算实现，`ss_ilg` 只是直接复用它。

    2.  **`SS_BLOCKSIZE < 256` (小块模式):**
        *   这是最直接的查表法。
        *   由于 `SS_BLOCKSIZE` (即 `ss_mintrosort` 处理的最大数据块) 小于256，那么传入 `ss_ilg` 的 `n`（即 `last - first`）也必然小于256。
        *   因此，可以直接用 `lg_table[n]` 来查找结果，这是一步到位的最快方法。

    3.  **`SS_BLOCKSIZE >= 256` (大块模式 - 默认):**
        *   这是最巧妙的实现。它将一个（通常是16位的）整数 `n` 拆分成**高8位**和**低8位**来处理，从而将256大小的查找表扩展到能处理更大的数。
        *   `return (n & 0xff00) ? ... : ...;`
        *   **`n & 0xff00`:** 这个位运算检查 `n` 的高8位是否不为0。
            *   **如果为真 (e.g., n = 300 = 0x012C):**
                *   说明 `n` 至少是256。那么它的对数至少是8。
                *   `8 + lg_table[(n >> 8) & 0xff]`
                *   ` (n >> 8) & 0xff `: 右移8位，取出高8位的值（对于300，结果是1）。
                *   `lg_table[1]` 的结果是 `0`。
                *   最终结果是 `8 + 0 = 8`，这正是 `floor(log2(300))` 的正确结果。
            *   **如果为假 (e.g., n = 100 = 0x64):**
                *   说明 `n` 小于256。
                *   `0 + lg_table[(n >> 0) & 0xff]`
                *   直接用 `n` 的低8位（也就是 `n` 本身）查表。
                *   `lg_table[100]` 的结果是 `6`，即 `floor(log2(100))` 的正确结果。

**为什么这个函数很重要？**

*   `limit = ss_ilg((saidx_t)(last - first))` 这一行代码在 `ss_mintrosort` 的开头和每次压栈时都会被调用。
*   `limit` 的值直接关系到内省排序何时从快速排序切换到堆排序。
*   一个快速、准确的 `ss_ilg` 实现，是保证 `ss_mintrosort` 能够在不牺牲性能的情况下，又能获得最坏情况安全保障的基石。

### **代码解析**

```c
// 预计算的整数对数查找表，存储了 0-255 的 floor(log2(n)) 的值。
static const int lg_table[256]= {
 -1,0,1,1,2,2,2,2,3,3,3,3,3,3,3,3,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,4,
  5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,
  6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,
  6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,
  7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,
  7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,
  7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,
  7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7
};

/**
 * @brief 快速计算整数 n 以2为底的对数（向下取整）。
 * @param n  输入的整数。
 * @return saint_t floor(log2(n))
 */
static INLINE
saint_t
ss_ilg(saidx_t n) {
#if SS_BLOCKSIZE == 0
  // 模式1：直接调用 trsort 版本的实现
  return tr_ilg(n);
#elif SS_BLOCKSIZE < 256
  // 模式2：对于小数据块，n 必然小于256，直接查表
  return lg_table[n];
#else
  // 模式3：对于大数据块，n 可能大于255
  // 通过检查高8位，将计算扩展到支持16位整数（0-65535）
  return (n & 0xff00) ?              // 检查 n 的高8位是否为非零
          8 + lg_table[(n >> 8) & 0xff] : // 如果是，则结果是 8 + 对高8位的对数
          0 + lg_table[(n >> 0) & 0xff];  // 如果否，则结果直接是对低8位（即n本身）的对数
#endif
}
```

## ss_swapmerge **分治交换归并**

- 这是 `sssort` 中负责**归并**两个已排序子区间的核心算法。它不是一个简单的归并，而是一种高度优化的、基于分治思想的**原地归并 (In-place Merge)** 变体。

### **原理与设计思路解析**

标准归并排序需要 `O(n)` 的额外空间来合并两个子数组。`ss_swapmerge` 的设计目标是在**有限的额外缓冲区 (`buf`)** 的帮助下，尽可能地完成原地归并，以减少内存消耗。它的核心思想来源于经典的**块交换 (Block Swap)** 和**分治归并**算法。

*   **核心策略：找到中点并交换 (Find Median and Swap)**
    算法并没有从头到尾逐个元素比较。相反，它试图找到一个“分割点”，将问题分解成更小的、独立的子问题。

    1.  **寻找分割点:**
        *   `for(m = 0, len = ..., half = ...)` 这个循环是整个算法最精妙的部分。它在两个相邻的已排序子区间 `[first, middle)` 和 `[middle, last)` 的**交界处**，通过**二分查找**，寻找一个最佳的分割点。
        *   它要找到一个位置 `m`，使得 `[middle-m, middle)` 中的所有元素都**大于** `[middle, middle+m)` 中的所有元素。换句话说，`middle-m` 是 `[first, middle)` 中第一个应该被移到 `[middle, last)` 右边的元素，而 `middle+m` 是 `[middle, last)` 中最后一个应该留在原地的元素。
        *   `ss_compare(T, PA + GETIDX(*(middle + m + half)), ...)` 这一行正是在执行二分查找的比较步骤。

    2.  **块交换 (Block Swap):**
        *   一旦找到了这个分割点 `m`，算法就知道 `[middle-m, middle)` 这 `m` 个元素（来自左边区间）和 `[middle, middle+m)` 这 `m` 个元素（来自右边区间）**放错了位置**。
        *   `ss_blockswap(lm, middle, m);` 这一句执行一个**块交换**，将这两个大小为 `m` 的数据块**原地交换**位置。
        *   交换之后，`[first, middle-m)` 和 `[middle+m, last)` 两个外围区域的元素都在它们的最终正确区间内了。

    3.  **分治递归:**
        *   经过一次块交换，问题被**分解**成了两个**更小的、独立的归并子问题**：
            *   归并 `[first, middle-m)` 和 `[middle-m, l)`
            *   归并 `[r, middle+m)` 和 `[middle+m, last)`
        *   算法通过 `STACK_PUSH` 将这两个子问题中的一个（通常是较大的那个）压入**手动管理的栈**中，然后通过修改 `first`, `middle`, `last` 指针，立即**迭代**处理另一个子问题。这就实现了分治递归。

*   **利用有限的缓冲区 (`buf`)**
    *   当分治递归到子问题足够小，小到可以完全装入提供的缓冲区 `buf` 时 (`(last - middle) <= bufsize` 或 `(middle - first) <= bufsize`)，算法就会切换到**最高效的模式**。
    *   它会调用 `ss_mergebackward` 或 `ss_mergeforward`。这两个函数会利用 `buf` 作为临时空间，执行一次**标准的、线性的归并操作**，这比复杂的分治交换要快得多。这正是缓冲区的价值所在。

*   **处理相等后缀 (`MERGE_CHECK` 和 `GETIDX`)**
    *   与 `ss_insertionsort` 类似，归并过程也需要正确处理和**传递**那些被标记为“与前一个相等”的后缀（负数）。
    *   `GETIDX(a)` 宏的作用是在比较前，**临时地**将被标记的负数 `~a` 恢复成正数 `a`，以便能正确地用它作为 `PA` 数组的索引。
    *   `MERGE_CHECK(a, b, c)` 宏在归并完成后被调用，它的任务是检查新形成的有序序列的**边界**处，如果边界两侧的后缀相等，就要正确地设置标记。`check` 变量通过位标志 (`&1`, `&2`, `&4`) 来传递边界的状态信息。

### **代码解析**

```c
/* D&C based merge. (Divide and Conquer based merge) */
static
void
ss_swapmerge(const sauchar_t *T, const sastore_t*PA,
             sastore_t*first, sastore_t*middle, sastore_t*last,
             sastore_t*buf, saidx_t bufsize, saidx_t depth) {
// 宏定义
#define STACK_SIZE SS_SMERGE_STACKSIZE
#define GETIDX(a) ((0 <= (a)) ? (a) : (~(a))) // 获取真实索引（将负数标记转回正数）
#define MERGE_CHECK(a, b, c) /*...*/ // 检查并设置边界的相等标记

  // 手动管理的递归栈
  struct { sastore_t*a, *b, *c; saint_t d; } stack[STACK_SIZE];
  sastore_t*l, *r, *lm, *rm;
  saidx_t m, len, half;
  saint_t ssize;
  saint_t check, next;

  for(check = 0, ssize = 0;;) { // 主循环，模拟递归

    // --- 递归基本情况：切换到线性归并 ---
    // 如果右半部分可以完全装入缓冲区
    if((last - middle) <= bufsize) {
      if((first < middle) && (middle < last)) {
        // 调用标准的回溯归并
        ss_mergebackward(T, PA, first, middle, last, buf, depth);
      }
      MERGE_CHECK(first, last, check); // 检查边界
      STACK_POP(first, middle, last, check); // 弹出下一个任务
      continue;
    }
    // 如果左半部分可以完全装入缓冲区
    if((middle - first) <= bufsize) {
      if(first < middle) {
        // 调用标准的前向归并
        ss_mergeforward(T, PA, first, middle, last, buf, depth);
      }
      MERGE_CHECK(first, last, check);
      STACK_POP(first, middle, last, check);
      continue;
    }

    // --- 分治阶段：寻找分割点 ---
    // 在两个区间的交界处，通过二分查找寻找交换点m
    for(m = 0, len = (saidx_t)(MIN(middle - first, last - middle)), half = len >> 1;
        0 < len;
        len = half, half >>= 1) {
      if(ss_compare(T, PA + GETIDX(*(middle + m + half)),
                       PA + GETIDX(*(middle - m - half - 1)), depth) < 0) {
        m += half + 1;
        half -= (len & 1) ^ 1;
      }
    }

    if(0 < m) { // 如果找到了一个有效的交换点
      // --- 块交换 ---
      lm = middle - m; rm = middle + m;
      ss_blockswap(lm, middle, m); // 交换中间的两个m大小的块

      l = r = middle; next = 0;
      // ... 复杂的边界检查和标记处理 ...
      if(rm < last) {
        if(*rm < 0) {*rm = ~*rm; if(first < lm) { for(; *--l < 0;) { } next |= 4; } next |= 1;} 
        else if(first < lm) { for(; *r < 0; ++r) { } next |= 2;}
      }

      // --- 压栈并迭代 ---
      // 比较两个子问题的大小，总是将较大的子问题压栈，立即处理较小的
      if((l - first) <= (last - r)) {
        STACK_PUSH(r, rm, last, (next & 3) | (check & 4));
        middle = lm; last = l; check = (check & 3) | (next & 4);
      } else {
        if((next & 2) && (r == middle)) { next ^= 6; }
        STACK_PUSH(first, lm, l, (check & 3) | (next & 4));
        first = r; middle = rm; check = (next & 3) | (check & 4);
      }
    } else { // 如果 m=0，说明两个区间已经宏观有序，无需交换
      // 只需检查交界处的元素是否相等
      if(ss_compare(T, PA + GETIDX(*(middle - 1)), PA + *middle, depth) == 0) {
        *middle = ~*middle;
      }
      MERGE_CHECK(first, last, check);
      STACK_POP(first, middle, last, check);
    }
  }
#undef STACK_SIZE
}
```

## ss_mergebackward **利用缓冲区的回溯归并**

这是 `ss_swapmerge` 在其分治递归的**基本情况 (Base Case)** 下调用的高效线性归并函数之一。当待归并的右半部分 `[middle, last)` 的大小足以完全放入提供的缓冲区 `buf` 时，`ss_swapmerge` 会放弃复杂的分治交换，转而调用这个 `ss_mergebackward` 函数。

### **原理与设计思路解析**

这个函数实现了一种**回溯式 (Backward)** 的标准归并算法，并针对 `divsufsort` 的内部数据结构（特别是对相等后缀的负数标记）进行了深度优化。

*   **核心策略：回溯归并 (Merge Backward)**
    1.  **空间准备:**
        *   `ss_blockswap(buf, middle, (saidx_t)(last - middle));`
        *   算法的第一步，也是最关键的一步，是**将右半部分 `[middle, last)` 的全部内容移动到缓冲区 `buf` 中**。
        *   这样一来，原数组中 `[middle, last)` 的区域就**空闲**了出来，可以作为最终合并结果的写入区域。

    2.  **双指针回溯比较:**
        *   算法使用三个指针，都从**右向左**移动：
            *   `c`: 指向左半部分 `[first, middle)` 的**末尾**。
            *   `b`: 指向缓冲区 `buf` (即原右半部分) 的**末尾**。
            *   `a`: 指向最终合并区域 `[first, last)` 的**末尾**，作为**写入指针**。
        *   `for(...;;)` 循环不断地比较 `*c` 和 `*b` 所指向的后缀。
        *   将**较大**的那个元素复制到 `*a` 的位置，然后将对应的源指针（`c` 或 `b`）和写入指针 `a` 都向左移动一格。

    3.  **收尾:**
        *   当其中一个源指针（`c` 或 `b`）到达其区间的开头时，循环终止。
        *   此时，只需将另一个源指针所在区间中所有**剩余**的元素（它们必然是最小的）全部复制到目标区域的剩余位置即可。

*   **为什么是“回溯”？**
    *   因为写入指针 `a` 从 `last-1` 开始，向 `first` 移动。
    *   这种方式的优点在于，写入操作**不会覆盖**尚未被比较过的源数据（因为源数据在写入指针的左边）。这使得归并操作可以在一个几乎原地的空间（只需要一个较小的缓冲区）内完成。

*   **对相等后缀标记的处理**
    *   这是此函数最复杂的部分。代码必须在归并的同时，正确地**处理和传递**那些被标记为负数的相等后缀。
    *   `if(*bufend < 0) ... else ...`: 在开始比较之前，代码会预先检查两个源指针指向的第一个元素是否是负数标记，并用一个位掩码 `x` 记录下来。
    *   `if(x & 1) { do { *a-- = *b, *b-- = *a; } while(*b < 0); x ^= 1; }`: 这是一个典型的处理逻辑。当要移动一个元素 `*b` 时，代码会检查 `x` 的相应位。如果为1，说明 `*b` 是一个相等组的开始。于是，它会进入一个 `do-while` 循环，将**整个连续的负数标记组**一次性地移动过去，然后清除标志位 `x`。
    *   `*a-- = ~*b;`: 当比较发现两个后缀相等时 (`r == 0`)，它会在写入时，将其中一个标记为负数，以保持这个相等关系。

### **代码解析**

```c
static INLINE
void
ss_blockswap(sastore_t*a, sastore_t*b, saidx_t n) {
  saidx_t t;
  for(; 0 < n; --n, ++a, ++b) {
    t = *a, *a = *b, *b = t;
  }
}

/* Merge-backward with internal buffer. */
static
void
ss_mergebackward(const sauchar_t *T, const sastore_t*PA,
                 sastore_t*first, sastore_t*middle, sastore_t*last,
                 sastore_t*buf, saidx_t depth) {
  const sastore_t*p1, *p2; // 指向用于比较的 PA 数组中的元素
  sastore_t*a, *b, *c, *bufend; // a:写指针, b:缓冲区(右半部分)读指针, c:左半部分读指针
  saidx_t t;
  saint_t r; // 比较结果
  saint_t x; // 用于记录边界是否为负数标记的位掩码

  // 1. 准备阶段
  bufend = buf + (last - middle) - 1; // 指向缓冲区的末尾
  // 将右半部分 [middle, last) 的内容移动到缓冲区 buf
  ss_blockswap(buf, middle, (saidx_t)(last - middle));

  x = 0; // 初始化标记
  // 预读取第一个待比较的元素，并检查是否为负数标记
  if(*bufend < 0)       { p1 = PA + ~*bufend;       x |= 1; }
  else                  { p1 = PA +  *bufend; }
  if(*(middle - 1) < 0) { p2 = PA + ~*(middle - 1); x |= 2; }
  else                  { p2 = PA +  *(middle - 1); }
  
  // 2. 主归并循环
  // t 用于保存 a 指向的原始值，因为 a 自身是写入区域
  for(t = *(a = last - 1), b = bufend, c = middle - 1;;) {
    r = ss_compare(T, p1, p2, depth); // 比较左右两个部分的当前最大元素

    if(0 < r) { // 右半部分(buf)的元素更大
      // 如果需要处理负数标记组，则连续移动
      if(x & 1) { do { *a-- = *b, *b-- = *a; } while(*b < 0); x ^= 1; }
      *a-- = *b; // 将较大元素写入
      if(b <= buf) { *buf = t; break; } // 如果缓冲区已空，结束
      *b-- = *a; // 这是一个奇特的交换，用于恢复被 a 覆盖的值
      // 更新 p1 指针，准备下一轮比较
      if(*b < 0) { p1 = PA + ~*b; x |= 1; }
      else       { p1 = PA +  *b; }
    } else if(r < 0) { // 左半部分的元素更大
      // 类似地处理负数标记组
      if(x & 2) { do { *a-- = *c, *c-- = *a; } while(*c < 0); x ^= 2; }
      *a-- = *c, *c-- = *a; // 将较大元素写入
      if(c < first) { // 如果左半部分已空，将缓冲区剩余内容全部复制回去
        while(buf < b) { *a-- = *b, *b-- = *a; }
        *a = *b, *b = t;
        break;
      }
      // 更新 p2 指针
      if(*c < 0) { p2 = PA + ~*c; x |= 2; }
      else       { p2 = PA +  *c; }
    } else { // 两个元素相等
      // 依次移动右半部分和左半部分的元素（组）
      if(x & 1) { do { *a-- = *b, *b-- = *a; } while(*b < 0); x ^= 1; }
      *a-- = ~*b; // 写入时标记为负数
      if(b <= buf) { *buf = t; break; }
      *b-- = *a;
      if(x & 2) { do { *a-- = *c, *c-- = *a; } while(*c < 0); x ^= 2; }
      *a-- = *c, *c-- = *a;
      if(c < first) { // 左半部分已空
        while(buf < b) { *a-- = *b, *b-- = *a; }
        *a = *b, *b = t;
        break;
      }
      // 更新两个指针
      if(*b < 0) { p1 = PA + ~*b; x |= 1; }
      else       { p1 = PA +  *b; }
      if(*c < 0) { p2 = PA + ~*c; x |= 2; }
      else       { p2 = PA +  *c; }
    }
  }
}
```

## ss_inplacemerge **原地归并**

这是 `ss_swapmerge` 在无法使用足够大的外部缓冲区时所采用的**备选策略**。当 `ss_swapmerge` 的分治过程进行到最后，需要合并两个子区间，但剩余的可用缓冲区 `buf` 非常小甚至没有时，它就会调用这个 `ss_inplacemerge` 函数来完成最后的合并工作。

### **原理与设计思路解析**

`ss_inplacemerge` 实现了一种**原地归并 (In-place Merge)** 算法。原地归并的目标是在**不使用或只使用 `O(1)` 额外空间**的情况下，合并两个相邻的有序序列。这通常比使用缓冲区的归并要慢，但能极大地节省内存。

这个函数实现的是一种基于**循环旋转 (Rotation)** 的原地归并算法。

*   **核心策略：逐个插入与旋转**
    算法的整体思路可以看作是，不断地从右半部分 `[middle, last)` 中取出元素，然后为它在左半部分 `[first, middle)` 中找到正确的插入位置，最后通过一次**旋转**操作将元素放到位。

    1.  **选择元素:**
        *   `for(;;)` 主循环从右半部分的**最大元素**开始处理，即 `*(last - 1)`。

    2.  **在左半部分寻找插入点:**
        *   `for(a = first, len = ...)` 这一段是一个**二分查找**。
        *   它的目标是在左半部分 `[first, middle)` 中，找到第一个**大于或等于** `*(last - 1)` 的元素。这个位置 `a` 就是 `*(last - 1)` 应该被插入的地方。

    3.  **执行旋转 (`ss_rotate`):**
        *   一旦找到了插入点 `a`，我们就知道了 `[a, middle)` 这个子区间的元素都比 `*(last - 1)` 小，但它们现在的位置不对，应该被移动到 `*(last - 1)` 的后面。
        *   `ss_rotate(a, middle, last)` 这个函数调用执行一次**块旋转**操作。它会将 `[a, middle)` 和 `[middle, last)` 这两个序列进行三次反转，从而高效地将 `[middle, last)` 移动到 `a` 的位置，并将 `[a, middle)` 移动到 `last` 的位置。
        *   **效果：** 旋转完成后，`*(last-1)`（现在是新序列的某个位置）以及右半部分所有比它小的元素都被正确地归并到了左半部分。

    4.  **缩小问题规模:**
        *   `last -= middle - a;`: 旋转后，我们知道有 `middle - a` 个元素从左半部分移动到了序列的末尾，所以我们将 `last` 指针向左移动这么多，相当于**缩小了待处理问题的规模**。
        *   `middle = a;`: 新的 `middle` 指针就是刚刚找到的插入点 `a`。
        *   然后，`--last` 将已经放到正确位置的最大元素排除在下一次迭代之外。
        *   算法继续在缩小的 `[first, last)` 范围内重复这个过程，直到所有右半部分的元素都被正确地归并进来。

*   **对相等后缀的处理**
    *   与之前的函数一样，它也需要正确处理负数标记。
    *   `if(*(last - 1) < 0) { x = 1; p = PA + ~*(last - 1); }`: 在比较前，先检查元素是否被标记，并获取其真实索引。
    *   `if(r == 0) { *a = ~*a; }`: 在二分查找找到插入点时，如果发现插入点 `a` 的后缀与待插入的后缀相等，就在 `*a` 上设置负数标记。
    *   `if(x != 0) { while(*--last < 0) { } }`: 在缩小 `last` 范围时，如果当前处理的元素是一个相等组的开始，这个循环会跳过整个连续的负数标记组。

### **代码解析**

```c
/*---------------------------------------------------------------------------*/

/**
 * @brief 使用旋转操作，原地归并两个相邻的有序序列。
 * @param T, PA        用于后缀比较的上下文。
 * @param first, middle, last 待归并的区间 [first, middle) 和 [middle, last)。
 * @param depth        多关键字比较的起始深度。
 */
static
void
ss_inplacemerge(const sauchar_t *T, const sastore_t*PA,
                sastore_t*first, sastore_t*middle, sastore_t*last,
                saidx_t depth) {
  const sastore_t*p; // 指向用于比较的 PA 数组中的元素
  sastore_t*a, *b;  // a: 二分查找指针, b: 临时指针
  saidx_t len, half;
  saint_t q, r;     // 比较结果
  saint_t x;        // 相等标记

  // 主循环：不断地将右半部分的最大元素归并到左半部分
  for(;;) {
    // 1. 选择右半部分的最大元素 *(last - 1)
    if(*(last - 1) < 0) { x = 1; p = PA + ~*(last - 1); } // 处理负数标记
    else                { x = 0; p = PA +  *(last - 1); }

    // 2. 在左半部分 [first, middle) 中为 *(last - 1) 寻找插入点 a
    // 这是一个二分查找
    for(a = first, len = (saidx_t)(middle - first), half = len >> 1, r = -1;
        0 < len;
        len = half, half >>= 1) {
      b = a + half;
      // 比较 b 指向的后缀和我们选择的元素 p
      q = ss_compare(T, PA + ((0 <= *b) ? *b : ~*b), p, depth);
      if(q < 0) { // 如果 *b < *p, 说明插入点在右边
        a = b + 1;
        half -= (len & 1) ^ 1;
      } else { // 否则，插入点在左边（或就是b）
        r = q;
      }
    }

    // 3. 如果找到了插入点（即 a < middle），则执行旋转
    if(a < middle) {
      // 如果插入点 a 的后缀与 p 相等，则标记 *a
      if(r == 0) { *a = ~*a; }
      
      // 关键操作：旋转 [a, middle) 和 [middle, last)
      ss_rotate(a, middle, last);
      
      // 4. 缩小问题规模
      last -= middle - a; // 新的 last 指针
      middle = a;         // 新的 middle 指针
      
      // 如果左半部分已空，归并完成
      if(first == middle) { break; }
    }

    --last; // 将已归并的最大元素排除
    // 如果该元素是一个相等组，则跳过整个组
    if(x != 0) { while(*--last < 0) { } }
    
    // 如果右半部分已空，归并完成
    if(middle == last) { break; }
  }
}
```

## ss_blockswap & ss_rotate **底层内存块操作**

这两个函数是 `ss_swapmerge` 和 `ss_inplacemerge` 等更高级算法能够实现**原地 (in-place)** 操作的基石。它们负责高效地移动内存块，而无需分配大量的额外空间。

### **`ss_blockswap`**

#### **原理与设计思路解析**

`ss_blockswap` 的功能非常直接：**交换两个不重叠、大小相同**的内存块。

*   **核心策略：逐元素交换 (Element-wise Swap)**
    *   该函数实现了一个简单的 `for` 循环。
    *   在每次循环中，它交换 `a` 指针和 `b` 指针当前指向的两个元素。
    *   然后两个指针都向前移动一格，重复这个过程 `n` 次，直到两个块的所有元素都被交换完毕。
    *   这是一种最基础、最直接的实现方式。

*   **性能考量**
    *   `INLINE` 关键字建议编译器将这个循环直接内联到调用它的地方，以避免函数调用的开销。
    *   在现代编译器和CPU上，这种简单的循环通常能被很好地优化，可能会被自动向量化（使用SIMD指令）来一次性交换多个元素，从而达到很高的效率。

#### **代码解析**

```c
/**
 * @brief 交换两个大小为 n 的内存块 a 和 b。
 * @param a, b 指向两个内存块的起始位置。
 * @param n    块的大小（元素数量）。
 */
static INLINE
void
ss_blockswap(sastore_t*a, sastore_t*b, saidx_t n) {
  saidx_t t;
  // 循环 n 次
  for(; 0 < n; --n, ++a, ++b) {
    // 使用一个临时变量 t 来完成 a 和 b 指向元素的交换
    t = *a, *a = *b, *b = t;
  }
}
```

---

### **`ss_rotate`**

#### **原理与设计思路解析**

`ss_rotate` 的功能要复杂得多：将一个序列 `[first, last)` 中的两个相邻子序列 `[first, middle)` 和 `[middle, last)` 进行**位置互换**。这等同于标准库中的 `std::rotate`。

*   **核心策略：Gries-Mills 算法 / 三次反转法 (Three Reversals)**
    这个函数实现了一种非常巧妙且高效的原地旋转算法，通常被称为 **Gries-Mills 算法**或**三次反转法**。但这里的具体实现看起来更像是一个基于**块交换**的迭代版本，也被称为**“手摇”算法 (Juggling Algorithm)** 的变体。

    **让我们用一个更简单的三次反转法来理解其原理：**
    要将 `AB` 旋转成 `BA` (其中 `A` 是 `[first, middle)`, `B` 是 `[middle, last)`)，只需三步：
    1.  反转 `A` -> `A^r B`
    2.  反转 `B` -> `A^r B^r`
    3.  反转整个 `A^r B^r` -> `(A^r B^r)^r` -> `B A`

    **而代码中实现的“手摇”算法的原理是：**

    1.  **比较大小:** 比较两个子序列 `A` 和 `B` 的长度。
    2.  **交换短序列:**
        *   如果 `A` 比 `B` 短 (`l < r`)，就将 `A` 与 `B` 的**末尾等长部分**进行块交换。序列变为 `B_prefix A B_suffix` -> `B_suffix A B_prefix`。现在 `A` 已经被移动到了正确的位置群组，问题规模缩小为对 `B_prefix` 和 `B_suffix` 进行旋转。
        *   如果 `B` 比 `A` 短 (`l > r`)，就将 `B` 与 `A` 的**开头等长部分**进行块交换。序列变为 `A_prefix A_suffix B` -> `B A_suffix A_prefix`。现在 `B` 已经被移动到了正确位置，问题规模缩小。
    3.  **迭代:** 在缩小后的序列上重复这个过程，直到其中一个子序列长度为0。

    代码中的 `if(l < r)` 和 `else` 块正是实现了这个逻辑，但它使用了更复杂的、看起来像“冒泡”的 `do-while` 循环来移动元素，而不是直接调用 `ss_blockswap`，这可能是为了处理某些特定的内存访问模式或边界情况。`if(l == r)` 是一个特殊情况的优化，当两个块大小相同时，直接调用一次 `ss_blockswap` 即可完成。

#### **代码解析**

```c
/**
 * @brief 原地旋转序列，将 [first, middle) 和 [middle, last) 互换位置。
 */
static INLINE
void
ss_rotate(sastore_t*first, sastore_t*middle, sastore_t*last) {
  sastore_t*a, *b, t;
  saidx_t l, r;
  l = (saidx_t)(middle - first); // 左半部分 A 的长度
  r = (saidx_t)(last - middle);  // 右半部分 B 的长度

  // 只要两部分都还存在，就继续循环
  for(; (0 < l) && (0 < r);) {
    // 优化：如果两部分等长，一次块交换就解决问题
    if(l == r) { ss_blockswap(first, middle, l); break; }

    if(l < r) { // 如果左半部分 A 更短
      a = last - 1; b = middle - 1; // a, b 分别指向 B 和 A 的末尾
      t = *a; // 保存 B 的最后一个元素
      // 这个循环将 A “冒泡”穿过 B
      do {
        *a-- = *b; *b-- = *a; // 移动元素
        if(b < first) { // 如果 A 的所有元素都移动完了
          *a = t;       // 将保存的 B 的元素放回
          last = a;     // B 的有效范围缩小了
          if((r -= l + 1) <= l) { break; } // 如果 B 的剩余部分不再比 A 长，则跳出内循环
          // 重置指针，准备处理 B 的下一段
          a -= 1; b = middle - 1;
          t = *a;
        }
      } while(1);
    } else { // 如果右半部分 B 更短
      a = first; b = middle; // a, b 分别指向 A 和 B 的开头
      t = *a; // 保存 A 的第一个元素
      // 这个循环将 B “冒泡”穿过 A
      do {
        *a++ = *b; *b++ = *a; // 移动元素
        if(last <= b) { // 如果 B 的所有元素都移动完了
          *a = t;       // 将保存的 A 的元素放回
          first = a + 1;// A 的有效范围缩小了
          if((l -= r + 1) <= r) { break; } // 如果 A 的剩余部分不再比 B 长，则跳出
          // 重置指针，准备处理 A 的下一段
          a += 1; b = middle;
          t = *a;
        }
      } while(1);
    }
  }
}
```

好的，我们来分析 `divsufsort` 中这个用于快速计算整数平方根的函数 `ss_isqrt`。

## ss_isqrt **快速整数平方根**

`ss_isqrt` (Integer Square Root) 是一个高度优化的辅助函数，它的唯一目标是**快速地计算一个整数的平方根并向下取整**（即 `floor(sqrt(x))`）。这个函数在 `sssort` 中被调用，用于决定在无法使用外部缓冲区时，应该采用多大的 `limit` 来进行原地归并。

### **原理与设计思路解析**

计算平方根是一个计算密集型操作，尤其是在一个没有硬件浮点单元支持的环境中。为了避免在性能敏感的排序算法中引入昂贵的数学库函数调用，`ss_isqrt` 采用了多种技巧相结合的策略，包括**查表法、位运算**和**牛顿迭代法**。

*   **核心策略：分段近似与迭代修正**
    该算法的基本思想是，不直接计算精确值，而是：
    1.  **快速估算:** 首先通过查表和位运算，得出一个与真实平方根非常接近的**初始估算值 `y`**。
    2.  **迭代修正:** 然后，使用**牛顿-拉弗森法 (Newton-Raphson method)** 对这个估算值进行一到两次迭代，使其快速收敛到精确的整数平方根。

*   **`lg_table` 和 `sqq_table`：双表驱动**
    该函数依赖两个预计算的查找表：
    *   **`lg_table` (对数表):** 我们之前已经分析过，它用于快速计算 `floor(log2(x))`。在这里，它被用来快速确定整数 `x` 的**最高有效位 (Most Significant Bit)** 的位置，这可以告诉我们 `x` 的大致数量级。
    *   **`sqq_table` (平方根-平方表):**
        *   这个表是 `ss_isqrt` 的核心。`sqq_table[i]` 存储的是 `floor(sqrt(i * 2^6))` 或类似的值，它本质上是一个**对0-255范围内的数进行平方根运算并放大了的定点数表**。
        *   通过将输入 `x` 的最高几位提取出来作为索引去查 `sqq_table`，我们可以得到一个精度相当不错的平方根初始估算值。

*   **执行流程详解**

    1.  **最高有效位确定 (`e`):**
        *   `e = (x & 0xffff0000) ? ... : ...;`
        *   这一大段代码与 `ss_ilg` 的逻辑类似，它通过检查 `x` 的不同字节段是否为零，并结合 `lg_table`，快速地计算出 `x` 的最高有效位所在的比特位置 `e`。例如，如果 `x` 是 1000 (二进制 `1111101000`），那么 `e` 大约是 9。

    2.  **分段处理与初始估算 (`y`):**
        *   **`e >= 16` (大数):**
            *   `x >> ((e - 6) - (e & 1))`: 这是一个复杂的位移操作。它的目的是将 `x` 的最高8个有效位对齐，并用这个结果去查 `sqq_table`。
            *   `<< ((e >> 1) - 7)`: 将查表得到的结果根据 `e` 的值进行相应的左移，将其“放缩”回正确的数量级，得到初始估算值 `y`。
        *   **`e >= 8` (中数):** 逻辑类似，但移位操作不同。
        *   **`e < 8` (小数):** 对于小于256的数，可以直接查 `sqq_table` 并通过右移得到一个足够好的近似值。

    3.  **牛顿法迭代修正:**
        *   `y = (y + 1 + x / y) >> 1;`
        *   这就是**牛顿-拉弗森法**求解平方根 `y = sqrt(x)` 的整数版本迭代公式。其基本思想是：如果 `y` 是 `x` 平方根的一个估算，那么 `x/y` 会是另一个估算（一个偏大，一个偏小），它们的平均值 `(y + x/y) / 2` 会是一个**更好**的估算。
        *   对于大数，代码执行了一到两次这个迭代，这足以让初始估算值快速收敛到精确的整数平方根。

    4.  **最终修正:**
        *   `return (x < (y * y)) ? y - 1 : y;`
        *   由于整数运算的截断误差，最终得到的 `y` 可能是正确的 `floor(sqrt(x))`，也可能比正确值大1。
        *   这一行代码通过一次最终的比较来进行修正，确保返回的是正确的向下取整的结果。

### **代码解析**

```c
// 预计算的平方根查找表。
static const saint_t sqq_table[256] = { /* ... a table of precomputed square root values ... */ };

/**
 * @brief 快速计算整数 x 的平方根并向下取整。
 * @param x 输入的整数。
 * @return saidx_t floor(sqrt(x))
 */
static INLINE
saidx_t
ss_isqrt(saidx_t x) {
  saidx_t y, e;

  // 快捷路径：如果 x 足够大，直接返回已知的上限
  if(x >= (SS_BLOCKSIZE * SS_BLOCKSIZE)) { return SS_BLOCKSIZE; }

  // 1. 快速确定 x 的最高有效位 e (整数对数)
  e = (x & 0xffff0000) ?
        ((x & 0xff000000) ?
          24 + lg_table[(x >> 24) & 0xff] :
          16 + lg_table[(x >> 16) & 0xff]) :
        ((x & 0x0000ff00) ?
           8 + lg_table[(x >>  8) & 0xff] :
           0 + lg_table[(x >>  0) & 0xff]);

  // 2. 分段处理，通过查表和位移得到初始估算值 y
  if(e >= 16) { // for large numbers
    // 提取 x 的高位，查表，然后通过移位放缩回正确量级
    y = sqq_table[x >> ((e - 6) - (e & 1))] << ((e >> 1) - 7);
    
    // 3. 牛顿法迭代修正
    if(e >= 24) { y = (y + 1 + x / y) >> 1; } // 对超大数，迭代两次
    y = (y + 1 + x / y) >> 1; // 所有大数，至少迭代一次
  } else if(e >= 8) { // for medium numbers
    y = (sqq_table[x >> ((e - 6) - (e & 1))] >> (7 - (e >> 1))) + 1;
  } else { // for small numbers (< 256)
    return sqq_table[x] >> 4;
  }

  // 4. 最终修正
  // 检查 y*y 是否过大，并返回正确结果
  return (x < (y * y)) ? y - 1 : y;
}
```
