---
title: SA-IS
categories:
  - hpatch
  - SA-IS
tags:
  - hpatch
  - SA-IS
abbrlink: 79755cef
date: 2025-10-04 18:57:17
---
<meta name="referrer" content="no-referrer" />

[TOC]

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/7adb26c2b1f84e639bd26004596bf5c4.png)

## saisxx **(顶层API)**

`saisxx` 是整个 `sais.hxx` 库提供给外部使用的**公共接口函数**。它本身不执行任何复杂的算法逻辑，其主要作用是作为一层**封装**，进行参数的合法性检查，处理一些简单的边界情况，并调用内部的核心实现 `saisxx_private::suffixsort`。

### **原理与设计思路解析**

*   **API 设计:**
    *   函数签名 `template<typename string_type, typename sarray_type, typename index_type>` 表明它是一个高度通用的模板函数。它可以接受任何满足随机访问迭代器 (Random Access Iterator) 条件的类型作为输入字符串 `T` 和输出数组 `SA`。例如，`T` 可以是 `const char*`、`std::string` 或 `std::vector<unsigned char>`；`SA` 可以是 `int*` 或 `std::vector<long>`。
    *   `k = 256` 的默认参数，表明它默认处理的是8位字符（如ASCII, UTF-8字节流）的字符串。

*   **参数校验与断言:**
    *   `assert(...)`: 在函数开头有一系列 `static_assert`（在C++11之前通常用 `assert` 模拟）来在编译期或调试时检查传入的整数类型 `index_type` 和 `savalue_type` 是否满足算法内部的要求（例如，它们必须是有符号整数，并且范围相同）。
    *   `if((n < 0) || (k <= 0)) { return -1; }`: 对输入的字符串长度 `n` 和字符集大小 `k` 进行运行时的合法性检查，确保它们是有效的。

*   **边界情况处理 (Edge Cases):**
    *   `if(n <= 1) { ... }`: 算法对长度为0或1的字符串进行了特殊处理。
        *   如果 `n == 0`，后缀数组为空，直接返回成功。
        *   如果 `n == 1`，后缀数组只有一个元素 `0`，将其设置后返回成功。
    *   处理这些简单的边界情况可以避免让核心的递归算法去处理这些琐碎的、可能引发问题的输入，简化了核心算法的逻辑。

*   **调用核心实现:**
    *   `return saisxx_private::suffixsort(T, SA, 0, n, k, false);`
    *   所有准备工作完成后，它就将参数“透传”给位于 `saisxx_private` 命名空间下的 `suffixsort` 函数。`false` 参数告诉 `suffixsort` 只需要计算后缀数组，而不需要计算BWT。

### **代码与注释**

```cpp
/**
 * @brief (公共API) 在线性时间内构造给定字符串的后缀数组。
 * @tparam string_type   输入字符串的类型 (需要满足随机访问迭代器)。
 * @tparam sarray_type   输出后缀数组的类型 (需要满足随机访问迭代器)。
 * @tparam index_type    用于长度和索引的整数类型。
 * @param T[0..n-1]      输入字符串。
 * @param SA[0..n-1]     用于存储结果的输出后缀数组。
 * @param n              输入字符串的长度。
 * @param k              字符集的大小 (例如 ASCII 为 256)。
 * @return int           成功返回0，如果参数无效则返回-1或-2。
 */
template<typename string_type, typename sarray_type, typename index_type>
int
saisxx(string_type T, sarray_type SA, index_type n, index_type k = 256) {
  // 定义 sarray_type 中存储的值的类型
  typedef typename std::iterator_traits<sarray_type>::value_type savalue_type;

  /* 编译期/调试时断言：确保用于索引和存储的整数类型是有符号的，并且范围匹配。
     算法内部使用负数作为标记，因此必须使用有符号整数。*/
  assert((std::numeric_limits<index_type>::min)() < 0);
  assert((std::numeric_limits<savalue_type>::min)() < 0);
  assert((std::numeric_limits<savalue_type>::max)() == (std::numeric_limits<index_type>::max)());
  assert((std::numeric_limits<savalue_type>::min)() == (std::numeric_limits<index_type>::min)());

  // 运行时参数校验
  if((n < 0) || (k <= 0)) { return -1; }

  // 处理边界情况：长度为0或1的字符串
  if(n <= 1) { 
    if(n == 1) { SA[0] = 0; } // 长度为1的字符串，其后缀数组就是 [0]
    return 0; 
  }

  // 调用私有的核心实现函数
  return saisxx_private::suffixsort(T, SA, 0, n, k, false);
}
```

## suffixsort **SA-IS 三阶段调度器**

`suffixsort` 是 SA-IS 算法的“总指挥”。它本身不执行具体的排序逻辑，而是负责**编排和调度**整个算法的三个核心阶段。它还包含了复杂的**内存管理**策略，以在给定的内存空间内高效地完成排序。

### **原理与设计思路解析**

该函数清晰地展示了 SA-IS 算法的分治流程：`Stage 1` (规约), `Stage 2` (递归), `Stage 3` (诱导)。

*   **`/* stage 1: reduce the problem ... */` (第一阶段：问题规约)**
    *   **目标:** 识别出所有的 LMS (Leftmost S-type) 后缀，对它们进行初步排序，然后将每个 LMS 子串“重命名”为一个新的整数，从而将原始问题**规约 (reduce)** 成一个规模更小、字符集可能也更小的**新问题**。
    *   **内存管理 (`flags`):** 函数开头的大量 `if-else` 和 `flags` 设置，是在进行精密的**内存布局规划**。`SA` 数组是一个巨大的内存块，`suffixsort` 试图将它“一地多用”：一部分用作最终的SA输出，一部分用作递归调用的SA，还有一部分用作基数排序所需的桶数组 `C` 和 `B`。`flags` 变量记录了最终采用的内存布局方案。这是一个极致的内存优化，旨在尽可能避免代价高昂的 `new` 操作。
    *   **调用 `stage1sort`:** 这是第一阶段的实际执行者。它完成LMS后缀的识别、初步排序和重命名工作。
    *   **返回:** `stage1sort` 返回两个关键值：`m` (LMS后缀的总数) 和 `name` (唯一LMS子串的数量)。

*   **`/* stage 2: solve the reduced problem ... */` (第二阶段：递归求解)**
    *   **目标:** 解决第一阶段规约产生的新问题。
    *   **递归条件:** `if(name < m)`。这个条件是递归的**核心**。
        *   如果 `name == m`，意味着所有 LMS 子串都是**唯一**的，它们在第一阶段的初步排序后，其相对顺序就已经完全确定了。因此，**无需递归**，可以直接跳到第三阶段。
        *   如果 `name < m`，意味着存在内容相同的 LMS 子串，它们的相对顺序**尚未确定**。此时，必须通过递归来解决。
    *   **准备递归输入:**
        *   `RA = SA + m + newfs;`: 在 `SA` 数组的后半部分开辟一块空间 `RA`。
        *   `for(...) { RA[j--] = SA[i] - 1; }`: 将第一阶段生成的“新字符串”（即 LMS 子串的名称序列）从 `SA` 的中间部分提取并复制到 `RA` 中。
    *   **递归调用:**
        *   `if(suffixsort(RA, SA, newfs, m, name, false) != 0) ...`: **核心递归调用**。它调用 `suffixsort` 自身，但输入变成了更短的、由“名称”组成的字符串 `RA`。递归的输出（排好序的“名称”的后缀数组）会被写入 `SA` 数组的前 `m` 个位置。
    *   **结果映射:** 递归返回后，需要将排好序的“名称”映射回它们所代表的原始 LMS 后缀。
        *   `for(i=0; i<m; ++i) { SA[i] = RA[SA[i]]; }`: `RA` 在这里被当作一个查找表，`RA[SA[i]]` 能找到第 `i` 小的名称所对应的原始LMS后缀的位置。

*   **`/* stage 3: induce the result ... */` (第三阶段：诱导排序)**
    *   **目标:** 利用在第二阶段已完全排好序的 LMS 后缀“骨架”，构建出完整的后缀数组。
    *   **调用 `stage3sort`:** 这是第三阶段的实际执行者。它接收 `SA` 数组前 `m` 个位置的已排序 LMS 后缀，并执行完整的两轮（一轮L-type，一轮S-type）诱导排序，最终将结果填充到整个 `SA` 数组中。

### **代码与注释**

```cpp
/* saisxx_private namespace */

/**
 * @brief SA-IS 算法的核心调度函数，编排三个主要阶段。
 * @param T      输入字符串。
 * @param SA     工作区和最终输出区。
 * @param fs     可供算法使用的额外栈空间大小。
 * @param n      字符串长度。
 * @param k      字符集大小。
 * @param isbwt  是否计算 BWT。
 * @return int   成功则返回 pidx (BWT的主索引)，失败返回-2。
 */
template<typename string_type, typename sarray_type, typename index_type>
int
suffixsort(string_type T, sarray_type SA,
           index_type fs, index_type n, index_type k,
           bool isbwt) {
  typedef typename std::iterator_traits<string_type>::value_type char_type;
  sarray_type RA, C, B; // RA: 递归数组, C,B: 桶数组
  index_type *Cp, *Bp;
  index_type i, j, m, name, pidx, newfs;
  unsigned flags = 0; // 记录内存布局方案的标志
  char_type c0, c1;

  /* --- stage 1: 规约问题 --- */
  C = B = SA; /* 消除编译器警告 */
  Cp = 0, Bp = 0;

  // --- 复杂的内存布局规划 ---
  // 目标是在 SA+fs 的空间内，为桶 C 和 B 找到位置，尽可能避免 new
  if(k <= 256) { // 小字符集
    try { Cp = new index_type[k]; } catch(...) { Cp = 0; }
    if(Cp == 0) { return -2; }
    if(k <= fs) {
      B = SA + (n + fs - k);
      flags = 1;
    } else {
      try { Bp = new index_type[k]; } catch(...) { Bp = 0; }
      if(Bp == 0) { return -2; }
      flags = 3;
    }
  } else if(k <= fs) { // 大字符集，但空间足够
    C = SA + (n + fs - k);
    if(k <= (fs - k)) {
      B = C - k;
      flags = 0;
    } else if(k <= 1024) {
      try { Bp = new index_type[k]; } catch(...) { Bp = 0; }
      if(Bp == 0) { return -2; }
      flags = 2;
    } else {
      B = C;
      flags = 64 | 8;
    }
  } else { // 空间不足，必须 new
    try { Cp = new index_type[k]; } catch(...) { Cp = 0; }
    if(Cp == 0) { return -2; }
    Bp = Cp;
    flags = 4 | 8;
  }
  // ... 更多基于空间的优化标志设置 ...
  
  {
    std::pair<index_type, index_type> r;
    // 调用第一阶段的执行函数
    if(Cp != 0) {
      if(Bp != 0) { r = stage1sort(T, SA, Cp, Bp, n, k, flags); }
      else { r = stage1sort(T, SA, Cp, B, n, k, flags); }
    } else {
      if(Bp != 0) { r = stage1sort(T, SA, C, Bp, n, k, flags); }
      else { r = stage1sort(T, SA, C, B, n, k, flags); }
    }
    m = r.first;    // LMS 后缀的数量
    name = r.second; // 唯一 LMS 子串的数量
  }
  // 清理第一阶段分配的内存
  if(m < 0) {
    if(flags & (1 | 4)) { delete[] Cp; }
    if(flags & 2) { delete[] Bp; }
    return -2;
  }

  /* --- stage 2: 递归解决子问题 --- */
  if(name < m) { // 如果存在重复的LMS子串，则需要递归
    // ... 清理第一阶段内存 ...
    newfs = (n + fs) - (m * 2); // 计算留给递归调用的额外空间
    // ... 更多内存布局标志 ...

    // 准备递归的输入字符串 RA (即 LMS 子串的名称序列)
    RA = SA + m + newfs;
    for(i = m + (n >> 1) - 1, j = m - 1; m <= i; --i) {
      if(SA[i] != 0) { RA[j--] = SA[i] - 1; }
    }
    // **核心递归调用**
    if(suffixsort(RA, SA, newfs, m, name, false) != 0) { /* ... 错误处理 ... */ return -2; }

    // --- 递归返回后，将结果映射回原始 LMS 后缀 ---
    // 准备查找表 RA
    i = n - 1; j = m - 1; c0 = T[n - 1];
    do { c1 = c0; } while((0 <= --i) && ((c0 = T[i]) >= c1));
    for(; 0 <= i;) {
      do { c1 = c0; } while((0 <= --i) && ((c0 = T[i]) <= c1));
      if(0 <= i) {
        RA[j--] = i + 1;
        do { c1 = c0; } while((0 <= --i) && ((c0 = T[i]) >= c1));
      }
    }
    // 映射结果
    for(i = 0; i < m; ++i) { SA[i] = RA[SA[i]]; }
    
    // ... 为第三阶段重新分配桶内存 ...
  }

  /* --- stage 3: 诱导排序，得出最终结果 --- */
  // 调用第三阶段的执行函数
  if(Cp != 0) {
    if(Bp != 0) { pidx = stage3sort(T, SA, Cp, Bp, n, m, k, flags, isbwt); }
    else { pidx = stage3sort(T, SA, Cp, B, n, m, k, flags, isbwt); }
  } else {
    if(Bp != 0) { pidx = stage3sort(T, SA, C, Bp, n, m, k, flags, isbwt); }
    else { pidx = stage3sort(T, SA, C, B, n, m, k, flags, isbwt); }
  }
  
  // 清理为第三阶段分配的内存
  if(flags & (1 | 4)) { delete[] Cp; }
  if(flags & 2) { delete[] Bp; }

  return pidx;
}
```

## stage1sort **(第一阶段) 问题规约：识别与命名 LMS 子串**

`stage1sort` 是 SA-IS 算法的**问题规约**阶段。它的核心任务是扫描原始字符串，找出所有 LMS (Leftmost S-type) 后缀，对它们进行初步排序，然后将每个 LMS 后缀所代表的**子串**进行“重命名”，最终生成一个规模更小的新问题，交给第二阶段（递归）去解决。

### **原理与设计思路解析**

这个函数本身也包含多个精巧的步骤来完成其复杂的任务。

*   **第一步：识别 LMS 后缀并放入桶中 (初步排序)**
    *   **目标:** 找出所有 LMS 后缀，并利用基数排序的思想，将它们放入各自**首字符**对应的桶中。
    *   **从右到左扫描:** `for(; 0 <= i;)` 循环从字符串的末尾向前扫描。
    *   **L/S 类型判断:** `do { c1 = c0; } while((0 <= --i) && ((c0 = T[i]) >= c1));` 这一段是在判断后缀的 L/S 类型。当循环从 `T[i] >= c1` (L-type) 的状态跳出时，说明遇到了一个 S-type 后缀。
    *   **LMS 识别:** `do { c1 = c0; } while((0 <= --i) && ((c0 = T[i]) <= c1));` 紧接着的这一段是在跳过一个连续的 S-type 区域。当它跳出时，说明又遇到了一个 L-type 后缀。那么，前一个位置 `j = i` 就是一个 `...L S...` 模式的转折点，因此后缀 `j` 是一个 **LMS 后缀**。
    *   **放入桶中:**
        *   `getBuckets(C, B, k, true);` 在扫描前，已经将 `B` 数组设置成了各个桶的**结束边界指针**。
        *   `*b = j; b = SA + --B[c1];`: 当找到一个 LMS 后缀 `j`，其首字符为 `c1` 时，就将 `j` 放入 `c1` 桶当前可用的**末尾位置**，然后将该桶的写指针 `--B[c1]` 向前移动一格。
    *   **结果:** 这一步结束后，`SA` 数组的后半部分（从右向左填充）存储了所有的 LMS 后缀，并且它们已经按照**首字符**进行了排序。

*   **第二步：对 LMS 后缀进行诱导排序 (`LMSsort1` / `LMSsort2`)**
    *   **目标:** 对第一步初步排序的结果进行细化。仅仅按首字符排序是不够的，还需要考虑后续字符。
    *   `LMSsort1` / `LMSsort2` 执行一次局部的、不完整的**诱导排序**。它们的作用是：利用 LMS 后缀之间的诱导关系，对这些 LMS 后缀自身进行一次更精确的排序。这足以将内容完全相同的 LMS 子串**聚集**在一起。

*   **第三步：重命名 (`LMSpostproc1` / `LMSpostproc2`)**
    *   **目标:** 比较排序后相邻的 LMS 子串，为每个**唯一**的子串分配一个整数“名称”。
    *   `LMSpostproc1` / `LMSpostproc2` (post-process) 负责这个过程。
    *   **压缩 SA 数组:** 首先，它将排序后的 LMS 后缀紧凑地移动到 `SA` 数组的**前 `m` 个位置**。
    *   **比较与命名:** 然后，它遍历这 `m` 个 LMS 后缀，逐一比较当前 LMS 子串与上一个 LMS 子串的内容是否相同。
        *   如果不同，就分配一个新的名称 (`++name`)。
        *   如果相同，就使用与上一个相同的名称。
    *   **存储新字符串:** 它巧妙地将这些“名称”存储在 `SA` 数组的**中间部分**（`SA[m + (p >> 1)] = name;`）。这样，一个规模为 `m` 的新问题（由整数名称构成的字符串）就被原地构造出来了。
    *   **返回:** 函数最终返回 LMS 后缀的总数 `m` 和唯一名称的数量 `name`，这两个值将决定第二阶段是否需要递归。

### **代码与注释**

```cpp
/* saisxx_private namespace */

/**
 * @brief (第一阶段) 规约问题：识别、排序并重命名 LMS 子串。
 * @return std::pair<index_type, index_type> 返回 <LMS后缀总数 m, 唯一名称数 name>。
 */
template<typename string_type, typename sarray_type,
         typename bucketC_type, typename bucketB_type,
         typename index_type>
std::pair<index_type, index_type>
stage1sort(string_type T, sarray_type SA,
           bucketC_type C, bucketB_type B,
           index_type n, index_type k, unsigned flags) {
  typedef typename std::iterator_traits<string_type>::value_type char_type;
  sarray_type b;
  index_type i, j, name, m;
  char_type c0, c1;

  // 1. 准备工作：统计字符频率，并计算桶的结束边界
  getCounts(T, C, n, k); 
  getBuckets(C, B, k, true); /* find ends of buckets */
  for(i = 0; i < n; ++i) { SA[i] = 0; } // 清空SA数组

  // 2. 从右到左扫描，识别LMS后缀并放入桶中
  b = SA + n - 1; // b 是总写指针，从最末尾开始
  i = n - 1; j = n; m = 0; c0 = T[n - 1];
  // 从右向左跳过一个 L-type 区域 (T[i] >= T[i+1])
  do { c1 = c0; } while((0 <= --i) && ((c0 = T[i]) >= c1));
  for(; 0 <= i;) {
    // 再从右向左跳过一个 S-type 区域 (T[i] <= T[i+1])
    do { c1 = c0; } while((0 <= --i) && ((c0 = T[i]) <= c1));
    if(0 <= i) { // 如果没到头，说明刚刚越过了一个 L->S 的边界
      *b = j; // j 是上一个LMS后缀的位置，先存入
      // B[c1]是首字符为c1的桶的当前写指针
      b = SA + --B[c1]; 
      j = i; ++m; // j 更新为当前LMS后缀的位置，LMS计数器m加一
      assert(B[c1] != (n - 1));
      // 再次跳过一个 L-type 区域，准备下一次LMS识别
      do { c1 = c0; } while((0 <= --i) && ((c0 = T[i]) >= c1));
    }
  }
  SA[n - 1] = 0; // 清理临时值

  // 3. 对初步排序的LMS后缀进行更精确的诱导排序
  if(1 < m) {
    if(flags & (16 | 32)) { // 根据优化标志选择不同版本的LMSsort
      // ... LMSsort2 and LMSpostproc2 path (更复杂的优化版本) ...
      // ...
      LMSsort2(T, SA, C, B, D, n, k);
      name = LMSpostproc2(SA, n, m);
    } else {
      // 调用基础版本的 LMS 诱导排序
      LMSsort1(T, SA, C, B, n, k, (flags & (4 | 64)) != 0);
      // 对排序后的LMS子串进行重命名
      name = LMSpostproc1(T, SA, n, m);
    }
  } else if(m == 1) { // 只有一个LMS后缀
    *b = j + 1;
    name = 1;
  } else { // 没有LMS后缀
    name = 0;
  }
  
  // 返回LMS后缀总数和唯一名称数
  return std::make_pair(m, name);
}
```

## stage3sort **(第三阶段) 诱导排序：构建完整 SA**

`stage3sort` 是 SA-IS 算法的**最终合成阶段**。在第一阶段规约问题、第二阶段递归求解之后，我们已经得到了一个**完全排好序的 LMS 后缀列表**（存储在 `SA` 数组的前 `m` 个位置）。`stage3sort` 的任务就是利用这个已排序的“骨架”，通过两轮高效的线性扫描，**诱导出所有其他 L-type 和 S-type 后缀的正确位置**，从而构建出最终的、完整的后缀数组。

### **原理与设计思路解析**

这个函数是“诱导排序”思想最纯粹、最直接的体现。它包含两个步骤：首先是准备工作，然后是核心的诱导排序。

*   **第一步：准备阶段 (放置 LMS 后缀骨架)**
    *   **目标:** 将已排序的 LMS 后缀从 `SA` 数组的前 `m` 个位置，移动到它们各自**首字符**对应的桶的**末尾**。这是为后续的诱导扫描做准备，因为诱导过程需要从这些已知的“锚点”开始。
    *   `if(1 < m)`: 只有当存在 LMS 后缀时才需要此步骤。
    *   `getBuckets(C, B, k, 1);`: 计算每个桶的**结束边界**。
    *   `do { ... } while(0 <= i);`: 这个循环从**后向前**遍历已排序的 LMS 后缀列表（位于 `SA[0...m-1]`）。
    *   `SA[--j] = p;`: 对于每个 LMS 后缀 `p`，它被放置到其首字符 `c1` 对应桶的当前可用**末尾位置** `j`。
    *   **结果:** 这一步结束后，`SA` 数组大部分是空的（被 `0` 填充），只有那些已排序的 LMS 后缀被精准地放置在了它们桶的末尾，构成了诱导排序的“骨架”。

*   **第二步：执行完整诱导排序 (`induceSA` 或 `computeBWT`)**
    *   **目标:** 执行两轮线性扫描，填充 `SA` 数组的其余所有空位。
    *   `if(isbwt == false) { induceSA(...) } else { pidx = computeBWT(...) }`: 根据调用者是否需要计算 Burrows-Wheeler Transform (BWT)，选择调用不同的函数。我们主要关注 `induceSA`。
    *   **`induceSA` 的内部工作 (两轮扫描):**
        1.  **第一轮 (诱导 L-type):**
            *   **扫描方向:** **从左到右** (`for(i = 0; i < n; ++i)`) 扫描整个 `SA` 数组。
            *   **诱导逻辑:** 当扫描到一个**已放置**的后缀 `j` (`j > 0`) 时，算法会考察它的**前一个后缀** `j-1`。
            *   如果后缀 `j-1` 是 **L-type** (`T[j-1] >= T[j]`)，算法就会将 `j-1` 放入其**首字符**对应桶的**当前可用开头位置**。
            *   由于是从左到右扫描，桶的写指针也是从左到右移动，这保证了所有 L-type 后缀在其桶内是按字典序排列的。
        2.  **第二轮 (诱导 S-type):**
            *   **扫描方向:** **从右到左** (`for(i = n - 1; 0 <= i; --i)`) 扫描整个 `SA` 数组。
            *   **诱导逻辑:** 当扫描到一个**已放置**的后缀 `j` (`j > 0`) 时，再次考察 `j-1`。
            *   如果后缀 `j-1` 是 **S-type** (`T[j-1] <= T[j]`)，算法就会将 `j-1` 放入其**首字符**对应桶的**当前可用末尾位置**。
            *   由于是从右到左扫描，桶的写指针也是从右向左移动，这保证了所有 S-type 后缀在其桶内也是按字典序排列的。

*   **完成:** 经过这两轮扫描，`SA` 数组中的所有空位都被正确填充，一个完整的、排好序的后缀数组就构建完成了。

### **代码与注释**

```cpp
/* saisxx_private namespace */

/**
 * @brief (第三阶段) 利用已排序的 LMS 后缀，诱导构建出完整的后缀数组。
 * @param T      输入字符串。
 * @param SA     工作区和输出区。SA[0...m-1] 存储着已排序的 LMS 后缀。
 * @param C, B   桶数组。
 * @param n, m, k 字符串长度, LMS 后缀数, 字符集大小。
 * @param flags  优化标志。
 * @param isbwt  是否计算 BWT。
 * @return index_type BWT 的主索引 pidx，如果不计算则为0。
 */
template<typename string_type, typename sarray_type,
         typename bucketC_type, typename bucketB_type, typename index_type>
index_type
stage3sort(string_type T, sarray_type SA, bucketC_type C, bucketB_type B,
           index_type n, index_type m, index_type k,
           unsigned flags, bool isbwt) {
  typedef typename std::iterator_traits<string_type>::value_type char_type;
  index_type i, j, p, q, pidx = 0;
  char_type c0, c1;

  // 根据优化标志，可能需要重新统计字符频率
  if((flags & 8) != 0) { getCounts(T, C, n, k); }

  /* 1. 准备阶段: 将所有已排序的 LMS 后缀放入它们各自桶的末尾 */
  if(1 < m) {
    // 计算桶的结束边界
    getBuckets(C, B, k, 1); /* find ends of buckets */
    i = m - 1; j = n; p = SA[m - 1]; c1 = T[p];
    
    // 从后向前遍历已排序的 LMS 后缀列表 (SA[0...m-1])
    do {
      q = B[c0 = c1]; // q 是 c0 桶的当前写指针
      while(q < j) { SA[--j] = 0; } // 将上一个桶和当前桶之间的空隙填0
      
      // 将所有首字符为 c0 的 LMS 后缀，从 SA[i] 开始向前，放入桶中
      do {
        SA[--j] = p;
        if(--i < 0) { break; }
        p = SA[i];
      } while((c1 = T[p]) == c0);
    } while(0 <= i);
    
    while(0 < j) { SA[--j] = 0; } // 将最前面的空隙填0
  }

  /* 2. 执行完整的诱导排序 */
  if(isbwt == false) {
    // 调用 induceSA 执行两轮诱导扫描，填充整个 SA 数组
    induceSA(T, SA, C, B, n, k, (flags & (4 | 64)) != 0);
  } else {
    // 如果需要，则调用 computeBWT 版本
    pidx = computeBWT(T, SA, C, B, n, k, (flags & (4 | 64)) != 0);
  }

  return pidx;
}
```

## induceSA **核心诱导排序引擎**

`induceSA` 是 SA-IS 算法中**执行最终诱导排序**的核心引擎。它接收一个部分填充（只包含已排序的 LMS 后缀“骨架”）的 `SA` 数组，并通过**两轮方向相反的线性扫描**，高效地、确定性地将其余所有 L-type 和 S-type 后缀填充到它们的正确位置，从而完成整个后缀数组的构建。

### **原理与设计思路解析**

这个函数是“诱导排序”思想最直接的体现。它基于一个关键的推论：**一旦后缀 `i` 的排名确定，其前一个后缀 `i-1` 的排名也可以被高效地推导出来**。

*   **第一轮：诱导 L-type 后缀 (从左到右扫描)**
    *   **`/* compute SAl */`**: 这里的 `SAl` 指的是“S A Left”，即对 L-type 后缀进行排序。
    *   **准备:** `getBuckets(C, B, k, false);` 计算每个桶的**起始边界**，因为我们将从左到右填充。
    *   **扫描:** `for(i = 0; i < n; ++i)` 从头到尾扫描整个 `SA` 数组。
    *   **诱导逻辑:**
        1.  当扫描到 `i` 位置时，取出 `SA[i]` 中存储的后缀索引 `j`。注意，此时的 `j` 可能是负数标记，需要 `~j` 恢复。
        2.  如果 `j > 0`，说明它不是第一个字符的后缀，那么我们就可以考察它的**前一个后缀** `j-1`。
        3.  获取 `j-1` 的首字符 `c0 = T[j-1]` 和 `j` 的首字符 `c1 = T[j]`。
        4.  **判断类型:** 如果 `c0 < c1` (`T[j-1] < T[j]`)，说明后缀 `j-1` 是 **S-type**，我们**忽略**它（S-type 在第二轮处理）。如果 `c0 >= c1` (`T[j-1] >= T[j]`)，说明后缀 `j-1` 是 **L-type**。
        5.  **放置:** 对于 L-type 的后缀 `j-1`，算法会查找 `c0` 桶的**当前写指针 `b`**，并将 `j-1` 放入 `*b` 的位置，然后将写指针 `b++` 向前移动一格。
    *   **为何有效？** 因为我们是从左到右（即按字典序从小到大）扫描 `SA` 数组的。这意味着我们总是先处理排名较小的后缀。对于 L-type 后缀，其排名主要由首字符决定。因此，按照桶的顺序、从左到右填充，可以保证所有 L-type 后缀都被正确地排序。

*   **第二轮：诱导 S-type 后缀 (从右到左扫描)**
    *   **`/* compute SAs */`**: `SAs` 指的是“S A Right”，即对 S-type 后缀进行排序。
    *   **准备:** `getBuckets(C, B, k, true);` 重新计算每个桶的**结束边界**，因为我们将从右到左填充。
    *   **扫描:** `for(i = n - 1; 0 <= i; --i)` 从尾到头扫描整个 `SA` 数组。
    *   **诱导逻辑:**
        1.  与第一轮类似，取出后缀索引 `j`。
        2.  如果 `j > 0`，考察前一个后缀 `j-1`。
        3.  **判断类型:** 如果 `c0 > c1` (`T[j-1] > T[j]`)，说明后缀 `j-1` 是 **L-type**，我们**忽略**它（L-type 在第一轮已经处理完毕）。如果 `c0 <= c1` (`T[j-1] <= T[j]`)，说明后缀 `j-1` 是 **S-type**。
        4.  **放置:** 对于 S-type 的后缀 `j-1`，算法查找 `c0` 桶的**当前写指针 `b`**，并将 `j-1` 放入 `*--b` 的位置（先将写指针 `b` 向左移动一格，再写入）。
    *   **为何有效？** S-type 后缀的排序依赖于其**后续**的后缀。从右到左扫描（即按字典序从大到小）`SA` 数组，可以确保当我们处理后缀 `j` 时，所有比它大的后缀都已就位。这保证了在诱导 `j-1` 时，它的排序依据是可靠的，从而保证了所有 S-type 后缀也被正确排序。

*   **位标记 `~j` 的使用**
    在这个函数中，`~j` 被用来同时编码后缀索引和 L/S 类型信息，以减少一次额外的查表。
    *   `*b++ = ((0 < j) && (T[j - 1] < c1)) ? ~j : j;`: 在第一轮中，如果诱导出的 `j` 本身是 S-type，就用负数标记它。
    *   `*--b = ((j == 0) || (T[j - 1] > c1)) ? ~j : j;`: 在第二轮中，如果诱导出的 `j` 本身是 L-type，就用负数标记它。
    *   最后，`SA[i] = ~j;` 将所有标记恢复为正数索引。

### **代码与注释**

```cpp
/* saisxx_private namespace */

/**
 * @brief (第三阶段核心) 执行完整的诱导排序，构建SA和BWT。
 * @param T      输入字符串。
 * @param SA     工作区和输出区。包含已排序的LMS后缀骨架。
 * @param C, B   桶数组。
 * @param n, k   字符串长度, 字符集大小。
 * @param recount 是否需要重新计算字符频率。
 */
template<typename string_type, typename sarray_type,
         typename bucketC_type, typename bucketB_type, typename index_type>
void
induceSA(string_type T, sarray_type SA, bucketC_type C, bucketB_type B,
         index_type n, index_type k, bool recount) {
  typedef typename std::iterator_traits<string_type>::value_type char_type;
  sarray_type b;
  index_type i, j;
  char_type c0, c1;

  /* --- 第一轮：从左到右，诱导 L-type 后缀 --- */
  if(recount != false) { getCounts(T, C, n, k); }
  getBuckets(C, B, k, false); /* 计算桶的起始边界 */
  j = n - 1;
  b = SA + B[c1 = T[j]];
  // 手动放置最后一个后缀 S[n-1]
  *b++ = ((0 < j) && (T[j - 1] < c1)) ? ~j : j; 

  // 从左到右扫描 SA 数组
  for(i = 0; i < n; ++i) {
    j = SA[i], SA[i] = ~j; // 取出后缀 j 并标记为已处理
    if(0 < j) { // 如果 j 不是第一个字符的后缀
      --j; // 考察前一个后缀 j-1
      c0 = T[j]; // j-1 的首字符
      if(c0 != c1) { // 如果首字符 c0 与上一个不同
        B[c1] = (index_type)(b - SA); // 记录上一个桶的结束位置
        b = SA + B[c1 = c0]; // b 更新为新桶的写指针
      }
      // 如果 T[j-1] < c1(T[j])，则 j 是 S-type，标记为负数；否则是 L-type，保持正数。
      // 注意：这里诱导的是 j，而判断类型的是它的前一个 j-1。
      *b++ = ((0 < j) && (T[j - 1] < c1)) ? ~j : j;
    }
  }

  /* --- 第二轮：从右到左，诱导 S-type 后缀 --- */
  if(recount != false) { getCounts(T, C, n, k); }
  getBuckets(C, B, k, true); /* 计算桶的结束边界 */
  
  // 从右到左扫描 SA 数组
  for(i = n - 1, b = SA + B[c1 = 0]; 0 <= i; --i) {
    if(0 < (j = SA[i])) { // 如果 i 位置有一个已放置的后缀 j
      --j; // 考察前一个后缀 j-1
      c0 = T[j]; // j-1 的首字符
      if(c0 != c1) { // 如果首字符 c0 与上一个不同
        B[c1] = (index_type)(b - SA); // 记录上一个桶的结束位置
        b = SA + B[c1 = c0]; // b 更新为新桶的写指针
      }
      // 如果 T[j-1] > c1(T[j])，则 j 是 L-type，标记为负数；否则是 S-type，保持正数。
      *--b = ((j == 0) || (T[j - 1] > c1)) ? ~j : j;
    } else {
      // 恢复之前在第一轮中被标记为负数的后缀
      SA[i] = ~j;
    }
  }
}
```

## LMSsort1 **(第一阶段) LMS 子串的初步诱导排序**

`LMSsort1` 的任务是在 `stage1sort` 识别出所有 LMS 后缀并将它们（按首字符）放入桶中之后，对这些 LMS 后缀进行一次**更精确的排序**。它本身就是一个**微缩版的诱导排序**，但目的不是构建完整的SA，而是将内容完全相同的 LMS 子串**聚集**在一起，为后续的“重命名”阶段做准备。

### **原理与设计思路解析**

`LMSsort1` 也遵循与 `induceSA` 相同的两轮扫描诱导逻辑，但操作的对象和目标都更加有限。

*   **上下文:**
    *   此时，`SA` 数组的后半部分（从右向左）填充着按**首字符**初步排序的 LMS 后缀。数组的其他部分都是 `0`。

*   **第一轮：诱导 L-type 后缀 (从左到右)**
    *   **`/* compute SAl */`**: 与 `induceSA` 类似，这一步诱导 L-type 后缀。
    *   **扫描:** `for(i = 0; i < n; ++i)` 从左到右扫描 `SA` 数组。
    *   **诱导逻辑:**
        1.  当扫描到一个位置 `i`，如果 `SA[i]` 中有一个**有效的后缀索引 `j`**（此时只有LMS后缀是有效的），它会考察其前一个后缀 `j-1`。
        2.  如果 `j-1` 是 **L-type**，就将 `j-1` 放入其首字符对应桶的**开头**。
    *   **关键区别:** 在 `stage1sort` 的这个阶段，`SA` 数组非常稀疏，大部分都是0。这次扫描的目的不是填充整个SA，而是在 `SA` 的前半部分，临时地、不完整地放置一些由 LMS 后缀诱导出的 L-type 后缀。

*   **第二轮：诱导 S-type 后缀，并**筛选出 LMS 后缀** (从右到左)**
    *   **`/* compute SAs */`**: 这一步诱导 S-type 后缀。
    *   **扫描:** `for(i = n - 1; 0 <= i; --i)` 从右到左扫描 `SA` 数组。
    *   **诱导逻辑:**
        1.  当扫描到一个位置 `i`，如果 `SA[i]` 中有一个有效的后缀索引 `j`（无论是原始的 LMS 还是第一轮诱导出的 L-type），它都会考察其前一个后缀 `j-1`。
        2.  如果 `j-1` 是 **S-type**，就将 `j-1` 放入其首字符对应桶的**末尾**。
    *   **核心目标 (筛选):** SA-IS 算法的一个巧妙之处在于，**只有 LMS 后缀才是 S-type 且其前一个后缀是 L-type**。在第二轮从右向左的诱导过程中，当一个 S-type 后缀 `j-1` 被放置时，如果它的前一个后缀 `j-2` 是 L-type，那么 `j-1` 就是一个 LMS 后缀。通过这次诱导，所有原始的 LMS 后缀会被**重新、但更精确地**排序并放置到 `SA` 的前半部分。
    *   **结果:** 这一轮结束后，所有原始的 LMS 后缀被“提纯”并排序后，紧凑地排列在 `SA` 数组的**开头**部分。其他在扫描过程中产生的临时 L-type 和 S-type 后缀都被丢弃了（被 `SA[i] = 0` 清除）。

### **代码与注释**

```cpp
/* saisxx_private namespace */

/**
 * @brief (第一阶段核心) 对 LMS 后缀进行一次完整的诱导排序，以实现精确分组。
 * @param T      输入字符串。
 * @param SA     工作区。SA的后半部分包含按首字符排序的LMS后缀。
 * @param C, B   桶数组。
 * @param n, k   字符串长度, 字符集大小。
 * @param recount 是否需要重新计算字符频率。
 */
template<typename string_type, typename sarray_type,
         typename bucketC_type, typename bucketB_type, typename index_type>
void
LMSsort1(string_type T, sarray_type SA,
         bucketC_type C, bucketB_type B,
         index_type n, index_type k, bool recount) {
  typedef typename std::iterator_traits<string_type>::value_type char_type;
  sarray_type b;
  index_type i, j;
  char_type c0, c1;

  /* --- 第一轮：从左到右，诱导 L-type 后缀 (临时) --- */
  if(recount != false) { getCounts(T, C, n, k); }
  getBuckets(C, B, k, false); /* 计算桶的起始边界 */
  j = n - 1;
  b = SA + B[c1 = T[j]];
  --j;
  // 手动处理最后一个后缀
  *b++ = (T[j] < c1) ? ~j : j;

  // 从左到右扫描 SA 数组
  for(i = 0; i < n; ++i) {
    if(0 < (j = SA[i])) { // 如果 SA[i] 是一个有效的 LMS 后缀
      // ... (断言检查) ...
      if((c0 = T[j]) != c1) { B[c1] = (index_type)(b - SA); b = SA + B[c1 = c0]; }
      // ... (断言检查) ...
      --j; // 考察前一个后缀 j-1
      // 如果 j-1 是 L-type (T[j-1] >= T[j]=c1)，则将其诱导放置
      *b++ = (T[j] < c1) ? ~j : j; // 用负数标记 S-type
      SA[i] = 0; // 清除原始 LMS 后缀的位置，为第二轮做准备
    } else if(j < 0) { // 如果 SA[i] 是一个被标记的 S-type (来自手动处理)
      SA[i] = ~j; // 恢复
    }
  }

  /* --- 第二轮：从右到左，诱导 S-type 后缀，并筛选出 LMS 后缀 --- */
  if(recount != false) { getCounts(T, C, n, k); }
  getBuckets(C, B, k, true); /* 计算桶的结束边界 */
  
  // 从右到左扫描 SA 数组
  for(i = n - 1, b = SA + B[c1 = 0]; 0 <= i; --i) {
    if(0 < (j = SA[i])) { // 如果 SA[i] 是一个有效的后缀 (此时主要是第一轮诱导出的 L-type)
      // ... (断言检查) ...
      if((c0 = T[j]) != c1) { B[c1] = (index_type)(b - SA); b = SA + B[c1 = c0]; }
      // ... (断言检查) ...
      --j; // 考察前一个后缀 j-1
      // 如果 j-1 是 S-type (T[j-1] <= T[j]=c1)，则将其诱导放置
      // 关键：T[j-1] > c1 判断 j-1 是否是 LMS (前一个是L-type)
      *--b = (T[j] > c1) ? ~(j + 1) : j;
      SA[i] = 0; // 清除临时放置的 L-type 后缀
    }
  }
}
```

## LMSpostproc1 **(第一阶段) LMS 子串的重命名**

`LMSpostproc1` (LMS Post-processing 1) 的任务是对 `LMSsort1` 精确排序后的 LMS 后缀进行**“重命名” (Renaming)**。它会比较相邻的 LMS 子串，为每个**唯一**的子串分配一个整数“名称”，并将这些名称打包成一个新的、更短的字符串。这个新字符串就是下一阶段递归算法的输入。

### **原理与设计思路解析**

这个函数是实现问题规模**规约 (Reduction)** 的关键。它通过两个步骤完成任务：压缩和命名。

*   **第一步：压缩 (Compact)**
    *   **上下文:** `LMSsort1` 执行完毕后，所有排好序的 LMS 后缀被紧凑地放置在 `SA` 数组的**开头**部分，但它们之间可能仍然夹杂着一些被标记的（负数）或无效的（0）条目。
    *   **目标:** 将所有有效的、排好序的 LMS 后缀**无缝地**移动到 `SA` 数组的最前端 `SA[0...m-1]`。
    *   `for(i = 0; (p = SA[i]) < 0; ++i) { SA[i] = ~p; ... }`: 这个循环首先恢复所有被负数标记的 LMS 后缀。
    *   `if(i < m) { for(j = i, ++i;; ++i) { ... } }`: 这个循环则是一个经典的**双指针压缩算法**。指针 `j` 是写指针，指针 `i` 是读指针。它将 `i` 找到的有效 LMS 后缀 `~p` 写入到 `j` 的位置，从而消除所有无效条目。

*   **第二步：存储 LMS 子串长度**
    *   **目标:** 在后续的命名阶段，需要快速比较两个 LMS 子串是否相等。为了进行比较，我们必须知道每个 LMS 子串的**长度**。
    *   `for(; 0 <= i;)`: 这一段代码再次从右到左扫描原始字符串 `T`，用与 `stage1sort` 几乎完全相同的方式重新识别出所有的 LMS 后缀。
    *   `SA[m + ((i + 1) >> 1)] = j - i;`: 这是一个非常巧妙的内存复用技巧。
        *   当识别出一个 LMS 后缀 `i+1` 时，它与前一个 LMS 后缀 `j` 之间的距离 `j-i` 就是 LMS 子串 `T[i+1 ... j]` 的长度。
        *   这个长度信息被存储在了 `SA` 数组的**中间部分** `SA[m + ...] `。
        *   ` (i + 1) >> 1 ` 是一个空间优化，它假设 LMS 后缀的分布大致是每两个字符一个，所以可以用 `p/2` 作为索引来存储与 LMS 后缀 `p` 相关的信息。

*   **第三步：比较与命名 (Compare and Name)**
    *   **目标:** 遍历压缩好的、排好序的 LMS 后缀列表 `SA[0...m-1]`，为每个唯一的 LMS 子串分配一个从 `1` 开始递增的整数“名称”。
    *   `for(i = 0, name = 0, q = n, qlen = 0; i < m; ++i)`: 这个主循环遍历所有 `m` 个 LMS 后缀。
    *   `p = SA[i], plen = SA[m + (p >> 1)], diff = true;`: 获取当前 LMS 后缀 `p` 及其长度 `plen`。
    *   `if((plen == qlen) && ...)`: 比较当前 LMS 子串 `T[p...p+plen-1]` 与**上一个唯一**的 LMS 子串 `T[q...q+qlen-1]`。
    *   `for(j = 0; (j < plen) && (T[p + j] == T[q + j]); ++j) { }`: 逐字节比较。
    *   `if(diff != false) { ++name, q = p, qlen = plen; }`: 如果两个子串**不同**，则增加 `name` 计数器，并将当前子串 `p` 记录为新的“上一个唯一子串”。
    *   `SA[m + (p >> 1)] = name;`: **核心重命名步骤**。在存储 LMS 子串长度的**同一位置**，用计算出的新“名称” `name` **覆盖**掉原来的长度值。

*   **最终结果:**
    *   `LMSpostproc1` 执行完毕后，`SA` 数组的内存布局变为：
        *   `SA[0 ... m-1]`: 存储着紧凑的、排好序的 LMS 后缀的**原始位置**。
        *   `SA[m ... m+(n/2)]`: 存储着一个由整数“名称”构成的、长度为 `m` 的**新字符串**。这个新字符串就是第二阶段递归的输入。
    *   函数返回唯一名称的数量 `name`。

### **代码与注释**

```cpp
/* saisxx_private namespace */

/**
 * @brief (第一阶段后处理) 压缩、计算长度并重命名 LMS 子串。
 * @param T      输入字符串。
 * @param SA     工作区。SA 的开头部分包含 LMSsort1 的排序结果。
 * @param n, m   字符串长度, LMS 后缀数。
 * @return index_type 返回唯一 LMS 子串的数量 (即新问题的字符集大小)。
 */
template<typename string_type, typename sarray_type, typename index_type>
index_type
LMSpostproc1(string_type T, sarray_type SA, index_type n, index_type m) {
  typedef typename std::iterator_traits<string_type>::value_type char_type;
  index_type i, j, p, q, plen, qlen, name;
  char_type c0, c1;
  bool diff;

  assert(0 < n);
  /* 1. 压缩: 将所有已排序的 LMS 后缀紧凑地移动到 SA[0...m-1] */
  // 首先恢复所有被负数标记的 LMS 后缀
  for(i = 0; (p = SA[i]) < 0; ++i) { SA[i] = ~p; assert((i + 1) < n); }
  // 使用双指针法，将有效的 LMS 后缀移动到数组开头
  if(i < m) {
    for(j = i, ++i;; ++i) {
      assert(i < n);
      if((p = SA[i]) < 0) {
        SA[j++] = ~p; SA[i] = 0; // 移动有效后缀，并清除原位置
        if(j == m) { break; } // 当移动满 m 个后，停止
      }
    }
  }

  /* 2. 存储 LMS 子串的长度 */
  i = n - 1; j = n - 1; c0 = T[n - 1];
  // 从右到左重新扫描字符串以识别 LMS 后缀
  do { c1 = c0; } while((0 <= --i) && ((c0 = T[i]) >= c1));
  for(; 0 <= i;) {
    do { c1 = c0; } while((0 <= --i) && ((c0 = T[i]) <= c1));
    if(0 <= i) {
      // 当识别出一个 LMS 后缀 i+1 时，j-i 就是其子串长度
      // 将长度存储在 SA 数组的中间部分
      SA[m + ((i + 1) >> 1)] = j - i; 
      j = i + 1;
      do { c1 = c0; } while((0 <= --i) && ((c0 = T[i]) >= c1));
    }
  }

  /* 3. 比较与重命名 */
  for(i = 0, name = 0, q = n, qlen = 0; i < m; ++i) {
    p = SA[i]; // 当前 LMS 后缀的位置
    plen = SA[m + (p >> 1)]; // 从中间区域取出其长度
    diff = true;

    // 比较当前 LMS 子串 T[p...] 和上一个唯一的 LMS 子串 T[q...]
    if((plen == qlen) && ((q + plen) < n)) {
      for(j = 0; (j < plen) && (T[p + j] == T[q + j]); ++j) { }
      if(j == plen) { diff = false; } // 如果完全相同
    }

    if(diff != false) { // 如果不同
      ++name; // 分配一个新名称
      q = p; qlen = plen; // 更新“上一个唯一子串”
    }
    // 核心重命名：用新的名称覆盖掉之前存储的长度值
    SA[m + (p >> 1)] = name;
  }

  return name; // 返回唯一名称的总数
}
```
