---
title: trsort
categories:
  - hpatch
  - libdivsufsort
tags:
  - hpatch
  - libdivsufsort
abbrlink: d20370e5
date: 2025-10-03 09:12:19
---
[TOC]

## trsort **Tandem Repeat Sort (串联重复排序)**

`trsort` 是 `divsufsort` 中一个专门用于对**整数数组**进行排序的、高度优化的算法。在 `sort_typeBstar` 的流程中，当计算出所有 B\* 后缀的**排名 (rank)** 并将它们存入 `ISA` 数组后，`trsort` 就被调用来对这个**排名数组**进行最终的排序。

### **原理与设计思路解析**

`trsort` 的名称意为“串联重复排序”，这暗示了它特别擅长处理包含大量**连续重复数值**（即串联重复）的整数数组。这正是 B\* 后缀排名数组的典型特征：许多不同的 B\* 后缀在经过一定深度的比较后，会被赋予相同的排名。

`trsort` 本身是一个**多关键字内省排序 (Multikey Introsort)** 的变体，但它的实现方式与 `sssort` 完全不同，它采用了一种迭代式的、处理“未排序组”的策略。

*   **核心策略：迭代式细化与预算控制**
    算法通过一个主 `for` 循环，不断地对 `ISA` 数组进行**多轮 (pass)** 排序。每一轮都尝试解决上一轮留下的、还未完全排序的“相等组”。

    1.  **分组识别:**
        *   `do { ... } while(first < (SA + n));` 这个内部循环负责在一轮中，遍历整个 `SA` 数组，识别出需要排序的**子数组（分组）**。
        *   `SA` 数组在这里扮演了一个**“分区图”**的角色。负数 `*first` 表示一个已排序或无需排序的区域，可以跳过 `-t` 个元素。正数 `t` 则指向 `ISA` 数组中的一个位置，`SA + ISA[t] + 1` 则定义了这个分组的结束边界 `last`。
        *   通过这种方式，`SA` 数组巧妙地编码了 `ISA` 数组的排序状态。

    2.  **对分组进行内省排序 (`tr_introsort`):**
        *   对于每个识别出的、大小大于1的分组 `[first, last)`，算法会调用 `tr_introsort` 对其进行排序。
        *   `tr_introsort` 是一个与 `ss_mintrosort` 类似但专门为整数排序设计的内省排序函数。

    3.  **多关键字处理 (`ISAd`):**
        *   主 `for` 循环 `for(ISAd = ISA + depth; ...; ISAd += ISAd - ISA)` 是实现多关键字排序的关键。
        *   第一轮，`depth` (通常为1)很小，`ISAd` 指向 `ISA` 数组本身，`tr_introsort` 对排名进行排序。
        *   如果第一轮结束后，仍然存在未完全排序的组（`unsorted > 0`），算法会进入下一轮。
        *   `ISAd += ISAd - ISA` 这一句非常巧妙，它实现了**比较深度的指数级增长**。如果 `ISA` 指向排名的数组，那么 `ISAd` 就指向了“排名的排名”所对应的原始后缀位置，比较它们就相当于比较了更长的子串。这使得算法能够不断地深入，解决那些有很长公共前缀的后缀的排序问题。

*   **预算机制 (`trbudget_t`)**
    *   这是 `trsort` 一个非常独特的设计，用于防止算法在处理恶劣数据时性能退化，是其内省机制的一部分。
    *   `trbudget_init(&budget, tr_ilg(n) * 2 / 3, n);`
        *   `chance`: 初始化一个“机会”或“预算”，它与待排序元素数量的对数成正比。这类似于 `ss_mintrosort` 中的 `limit`。
        *   `remain`, `incval`: 用于控制预算的消耗。
    *   在 `tr_introsort` 内部（此处未显示），每次递归或分区操作都会消耗这个 `budget`。如果 `budget` 在排序一个分组的过程中被耗尽，`tr_introsort` 就会**提前终止**，并返回未排序元素的数量 `budget.count`。
    *   **作用:** 这个机制确保了 `tr_introsort` 不会在一个“困难”的分组上花费过多的时间。它会放弃这个分组，并让 `trsort` 的主循环在**下一轮、更深的 `depth`** 中再去解决它。这是一种比 `ss_mintrosort` 中直接切换到堆排序更灵活的性能控制策略。

### **代码解析**

```c
/*---------------------------------------------------------------------------*/

/*- Function -*/
// 预算结构体初始化
static INLINE
void
trbudget_init(trbudget_t *budget, saidx_t chance, saidx_t incval) {
  budget->chance = chance;
  budget->remain = budget->incval = incval;
}

/**
 * @brief Tandem Repeat Sort: 对整数数组（特别是B*后缀的排名数组ISA）进行排序。
 * @param ISA      待排序的整数数组（B*后缀的排名）。
 * @param SA       用作“分区图”和临时工作空间。
 * @param n        数组大小。
 * @param depth    初始比较深度。
 */
void
trsort(sastore_t *ISA, sastore_t* SA, saidx_t n, saidx_t depth) {
  sastore_t*ISAd; // 指向当前比较深度的“键”数组
  sastore_t*first, *last; // 当前处理的分组的边界
  trbudget_t budget; // 排序预算
  saidx_t t, skip, unsorted;

  // 1. 初始化预算
  trbudget_init(&budget, tr_ilg(n) * 2 / 3, n);

  // 2. 主循环：多轮迭代，不断增加比较深度
  // ISAd += ISAd - ISA 实现了深度的指数增长
  for(ISAd = ISA + depth; -n < *SA; ISAd += ISAd - ISA) {
    first = SA;
    skip = 0;
    unsorted = 0; // 记录本轮结束后，仍未完全排序的元素总数

    // 3. 内循环：遍历SA分区图，识别并排序各个分组
    do {
      if((t = *first) < 0) { // 如果遇到负数，表示一个已排序/跳过区域
        first -= t; skip += t; 
      }
      else { // 遇到正数，表示一个需要处理的分组
        if(skip != 0) { *(first + skip) = skip; skip = 0; }
        // 根据 ISA[t] 确定分组的结束位置
        last = SA + ISA[t] + 1;
        
        if(1 < (last - first)) { // 如果分组大小大于1
          budget.count = 0; // 重置预算计数器
          // 调用内省排序对这个分组进行排序
          tr_introsort(ISA, ISAd, SA, first, last, &budget);
          
          if(budget.count != 0) { // 如果预算耗尽，排序提前终止
            unsorted += budget.count; // 累加未排序的元素数量
          } else { 
            // 如果排序成功，用一个负数标记该区域，以便下一轮快速跳过
            skip = (saidx_t)(first - last); 
          }
        } else if((last - first) == 1) { // 分组大小为1，天然有序
          skip = -1;
        }
        first = last; // 移动到下一个分组的开始
      }
    } while(first < (SA + n));
    
    if(skip != 0) { *(first + skip) = skip; }
    
    // 4. 检查是否完成
    if(unsorted == 0) { break; } // 如果本轮没有任何未排序的元素，则整个排序完成
  }
}
``````