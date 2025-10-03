---
title: SuffixString
categories:
  - hpatch
tags:
  - hpatch
abbrlink: 74360fdb
date: 2025-10-03 09:12:19
---
[TOC]

# HDiffPatch\libHDiffPatch\HDiff\private_diff\suffix_string.cpp

## **后缀数组**

**后缀 (Suffix)**：一个字符串的后缀是指从字符串的某个位置开始到字符串末尾的所有字符组成的子串。
例如，对于字符串 `banana`，它的所有后缀是：
*   `banana` (从位置0开始)
*   `anana`  (从位置1开始)
*   `nana`   (从位置2开始)
*   `ana`    (从位置3开始)
*   `na`     (从位置4开始)
*   `a`      (从位置5开始)

**后缀数组 (Suffix Array)**：后缀数组**不是**存储这些后缀字符串本身，而是存储这些后缀的**起始位置索引**。最关键的一步是，这些索引是按照它们所代表的后缀字符串的**字典序（字母顺序）**进行排序的。

我们来为 `banana` 构建一个后缀数组：

1.  **列出所有后缀及其起始位置：**
    | 起始位置 | 后缀     |
    | :------- | :------- |
    | 0        | `banana` |
    | 1        | `anana`  |
    | 2        | `nana`   |
    | 3        | `ana`    |
    | 4        | `na`     |
    | 5        | `a`      |

2.  **按字典序对后缀进行排序：**
    *   `a`
    *   `ana`
    *   `anana`
    *   `banana`
    *   `na`
    *   `nana`

3.  **构建后缀数组：** 将排序后的后缀对应的**原始起始位置**按顺序列出。
    | 排序后索引 | 原始起始位置 | 对应的后缀 |
    | :--------- | :----------- | :--------- |
    | 0          | **5**        | `a`        |
    | 1          | **3**        | `ana`      |
    | 2          | **1**        | `anana`    |
    | 3          | **0**        | `banana`   |
    | 4          | **4**        | `na`       |
    | 5          | **2**        | `nana`     |

所以，字符串 `banana` 的后缀数组 `SA` 就是 `[5, 3, 1, 0, 4, 2]`。

**后缀数组的魔力在哪里？**

一旦后缀数组被构建好，查找一个子串（比如 `ana`）是否存在于原始字符串中，就变成了一个**二分查找**问题。你可以在这个排好序的后缀列表中通过二分查找快速定位到所有以 `ana` 开头的后缀。由于HDiffPatch要处理的是巨大的旧文件（几百MB甚至GB），这种能力使得它可以在海量数据中进行近乎瞬时的查找，而不是进行缓慢的线性扫描。

## **后缀数组高级构造算法总结**

在 `O(n log n)` 的倍增法成为瓶颈时，我们需要更快的 `O(n)` 算法。这些算法的核心思想都是**分治 (Divide and Conquer)**，但实现方式和巧妙程度各不相同。

| 算法名称           | 核心思想                                   | 优点                                                                | 缺点                                               | 关系与改进                                                                               |
| :----------------- | :----------------------------------------- | :------------------------------------------------------------------ | :--------------------------------------------------- | :--------------------------------------------------------------------------------------- |
| **DC3** (Skew)     | 差量分组 (Difference Cover)，三倍递归        | 理论优雅，是第一个真正意义上的线性时间算法，易于理解和证明。        | **常数巨大**，空间复杂度较高 (`O(n)`)，实际运行速度通常**慢于**常数优化的倍增法。 | 思想上的里程碑，为后续算法提供了“递归解决子问题”的思路。                               |
| **SA-IS**          | 诱导排序 (Induced Sorting)，L/S类型划分        | **常数极小**，空间复杂度低 (`O(n)`)，实现相对简洁，实际运行速度**非常快**。 | 理论证明和理解较为复杂和抽象。                                                       | 对DC3等算法思想的革新，用巧妙的“诱导”代替了复杂的递归和合并，是现代算法的基石。 |
| **DivSufSort**     | 分治与诱导排序的混合，B*/B类型划分             | **常数最小**，是目前公认的**速度最快**的后缀数组构造算法之一，内置多线程支持。 | 算法细节非常复杂，混合了多种高级技巧，难以完全理解。                               | 在SA-IS等算法的基础上，进一步应用了更深层次的优化和工程技巧，达到了性能的极致。       |

---

### **1. DC3 算法 (Difference Cover / Skew Algorithm)**

DC3是第一个被发现的线性时间后缀数组构造算法，具有里程碑式的意义。

*   **实现原理 (三步走):**
    1.  **分 (Divide):** 将所有后缀按照它们的起始位置 `i` 对3取模的结果分为三组：`i mod 3 = 0`, `i mod 3 = 1`, `i mod 3 = 2`。
    2.  **治 (Conquer):**
        *   它巧妙地发现，可以直接对 `mod 3 = 1` 和 `mod 3 = 2` 的后缀组进行**合并排序**。方法是：将每三个字符看作一个“大字符”，形成一个新的、长度为 `2n/3` 的字符串，然后**递归地调用DC3算法**对这个新字符串进行后缀排序。
        *   一旦 `mod 3 != 0` 的后缀排好序，再利用这个结果，通过一次线性扫描和基数排序，**推导出** `mod 3 = 0` 的后缀的正确顺序。
    3.  **合 (Combine):** 最后，使用类似于归并排序的方法，将已排序的 `mod 3 != 0` 组和 `mod 3 = 0` 组合并成最终的、完整的后缀数组。

*   **优缺点:**
    *   **优点:** 算法的递归结构清晰，理论上非常优美，是算法导论等教材中讲解线性时间构造的经典范例。
    *   **缺点:** **常数巨大**。递归调用、多次基数排序以及复杂的合并步骤导致其在实际运行中，往往比经过常数优化的 `O(n log n)` 倍增法还要慢。空间开销也相对较大。

---

### **2. SA-IS 算法 (Suffix Array - Induced Sorting)**

SA-IS算法是对DC3思想的一次巨大革新，它摒弃了复杂的递归合并，引入了极为巧妙的“诱导排序”思想，是目前**理论和实践都极为优秀**的算法。

*   **实现原理:**
    1.  **分类 (Classify):** 首先，对所有后缀进行一次线性扫描，根据其与下一个后缀的大小关系，将它们分为 **L型 (Larger)** 和 **S型 (Smaller)**。
    2.  **识别 LMS 子串:** 在 L/S 序列中，识别出所有 "S型在L型之后" (`...L S...`) 的特殊位置。从这些位置开始的后缀称为 **LMS (Leftmost S-type) 后缀**。
    3.  **递归排序LMS后缀:** 算法**只对这些数量少得多的LMS后缀进行排序**。排序方法是：将每个LMS后缀所代表的子串（称为LMS子串）看作“大字符”，形成一个新字符串，然后**递归地调用SA-IS算法**。
    4.  **诱导排序 (Induced Sort):** 这是算法的精髓。一旦LMS后缀排好序，它们就像“骨架”一样被放置在桶（基数排序的桶）的正确位置。然后：
        *   **第一轮扫描 (从左到右):** 利用已排序的LMS后缀，可以**诱导**出所有L型后缀的正确位置。
        *   **第二轮扫描 (从右到左):** 再次利用已排序的LMS后缀，可以**诱导**出所有S型后缀的正确位置。
        经过这两轮扫描，整个后缀数组就被完整地构建出来了。

*   **优缺点:**
    *   **优点:** 算法常数非常小，空间复杂度低，代码实现相比DC3也更简洁，实际运行速度极快。
    *   **缺点:** “诱导排序”的过程和其正确性的证明相对抽象，初次学习时理解起来有一定难度。

---

### **3. DivSufSort 算法 (HDiffPatch 使用)**

DivSufSort可以看作是SA-IS思想的进一步发展和极致的工程优化。它同样基于分治和诱导排序，但在分类、子问题构造和实现细节上都进行了深度优化。

*   **实现原理 (在 `divsufsort` 函数解析中已详述):**
    1.  **分类:** 将后缀分为 **Type A** 和 **Type B**。
    2.  **识别 B\* 子集:** 从Type B中识别出特殊的 **B\* 后缀**。
    3.  **递归排序 B\* 后缀:** 这是计算最密集的部分，`divsufsort` 在此步骤实现了高效的多线程并行。
    4.  **诱导构建完整数组:** 利用已排序的B\*后缀，通过多轮复杂的扫描和基数排序，诱导出所有Type A后缀的位置，最终构建出完整的后缀数组。

*   **优缺点:**
    *   **优点:** **目前公认的速度最快的后缀数组构造算法之一**。其实现经过了极致的底层优化（比如缓存友好、指令流水线等），并且原生支持多线程，性能卓越。
    *   **缺点:** 算法内部细节极其复杂，混合了多种高级技巧，普通开发者几乎不可能完全理解其论文和源码。它是一个典型的“黑盒”高性能库。

## TSuffixString::resetSuffixString **后缀数组构建与缓存预处理**

这个函数是 `TSuffixString` 类的核心初始化函数，正是它负责执行整个项目中计算最密集、最关键的预处理步骤：**构建（即排序）后缀数组**，并为其建立快速查找缓存。

### **原理与设计思路解析**

`resetSuffixString` 是整个差异发现流程的“准备阶段”。它的设计体现了对性能和内存使用的深度考量。

*   **核心策略：两阶段预处理**
    函数将整个预处理过程清晰地分为两个主要步骤：
    1.  **后缀数组创建 (`_suffixString_create`)**: 这是最核心、计算量最大的部分。它接收原始的旧文件数据，并调用专门的库（`libdivsufsort`）来执行高效的后缀排序，生成基础的后缀数组。
    2.  **加速缓存构建 (`build_cache`)**: 在后缀数组排好序之后，为了让后续的 `lower_bound` 查找能够达到极致的速度，这个步骤会遍历一次已排序的后缀数组，并建立我们之前讨论过的“单字符”和“双字符”前缀的快速查找表。

*   **关键优化：内存自适应 (32/64位选择)**
    *   这是该函数一个非常重要的内存优化设计。后缀数组存储的是索引，如果旧文件的大小不超过4GB，那么用32位整数 (`TInt32`) 来存储索引就足够了。但如果文件超过4GB，则必须使用64位整数 (`TInt`)。
    *   `isUseLargeSA()` 函数就是用来做这个判断的。
    *   **如果文件 > 4GB (`isUseLargeSA()` 为真):** 程序会使用成员变量 `m_SA_large` (一个`std::vector<TInt>`) 来存储64位的后缀数组。
    *   **如果文件 <= 4GB (`isUseLargeSA()` 为假):** 程序会使用成员变量 `m_SA_limit` (一个`std::vector<TInt32>`) 来存储32位的后缀数组。
    *   这种设计可以在处理小文件时，将后缀数组的内存占用**减半**，这是一个非常显著的优化。

*   **性能优化：并行计算**
    *   后缀数组的构建和缓存的建立都是非常耗时的操作。该函数接受一个 `threadNum` 参数，并将它传递给底层的 `_suffixString_create` 和 `build_cache` 函数。
    *   这意味着这两个关键的预处理步骤都是**并行化**的，能够充分利用多核CPU来大幅缩短准备时间。

**执行流程总结：**

1.  接收指向旧文件数据的指针。
2.  判断旧文件大小，决定是使用32位还是64位的后缀数组，以节省内存。
3.  调用 `_suffixString_create`，后者内部再调用 `libdivsufsort` 库，在多个线程上并行执行，完成对所有后缀的排序，并将结果存入选定的32/64位向量中。
4.  调用 `build_cache`，它同样在多个线程上并行执行，遍历刚刚生成的有序后缀数组，创建用于 `lower_bound` 加速的单字符和双字符前缀范围缓存表。
5.  函数执行完毕后，`TSuffixString` 对象就处于“准备就绪”状态，可以用于后续的高速匹配查询。

### **代码解析**

```cpp
/**
 * @brief 重置并构建后缀字符串对象。
 * @param src_begin  指向源数据（旧文件）的起始指针。
 * @param src_end    指向源数据的结束指针。
 * @param threadNum  用于并行计算的线程数。
 */
void TSuffixString::resetSuffixString(const TChar* src_begin,const TChar* src_end,size_t threadNum){
    assert(src_begin<=src_end); // 基本的健全性检查。
    // 保存指向原始数据的指针，供后续比较使用。
    m_src_begin=src_begin;
    m_src_end=src_end;

    // --- 关键优化：根据源数据大小选择32位或64位后缀数组 ---
    if (isUseLargeSA()){ // 如果源数据大于4GB...
        _clearVector(m_SA_limit); // 清理不使用的32位SA向量。
        // 调用创建函数，将排序结果存入64位的 m_SA_large 向量。
        _suffixString_create(m_src_begin,m_src_end,m_SA_large,threadNum);
    }else{ // 如果源数据不大于4GB...
        assert(sizeof(TInt32)==4);
        _clearVector(m_SA_large); // 清理不使用的64位SA向量。
        // 调用创建函数，将排序结果存入32位的 m_SA_limit 向量，节省一半内存。
        _suffixString_create(m_src_begin,m_src_end,m_SA_limit,threadNum);
    }

    // 在后缀数组构建完成后，为其建立加速查找用的缓存表。
    build_cache(threadNum);
}
```

## _suffixString_create **后缀数组构建的底层实现**

这个函数是 `resetSuffixString` 调用的底层工作函数，它直接封装了**后缀数组排序**这一核心算法。该函数通过C++模板和预处理器宏（`#ifdef`）的设计，实现了对不同排序后端库的灵活切换。

### **原理与设计思路解析**

`_suffixString_create` 的职责非常专一：接收原始数据，调用一个高性能的排序算法，然后将排好序的后缀数组索引填充到输出向量 `out_sstring` 中。

*   **核心策略：通过宏切换后端实现**
    *   代码被设计为可以轻松地在三种不同的排序实现之间切换，这在算法研究和性能对比时非常有用。在HDiffPatch的发布版本中，`_SA_SORTBY_DIVSUFSORT` 是默认启用的。
    *   **`_SA_SORTBY_STD_SORT` (标准库排序)**
        *   **实现:** 这是为了测试和对比而存在的“教学版”或“参考版”实现。它首先创建一个从`0`到`size-1`的索引数组，然后使用 `std::sort` 配合一个自定义的比较器 `TSuffixString_compare` 来对这些索引进行排序。
        *   **性能:** `std::sort` 通常是基于快速排序的变种，其平均时间复杂度是 `O(N log N)`。但对于字符串排序，每次比较本身就需要 `O(L)` 时间（`L`是字符串长度），所以总的复杂度会达到 `O(L * N log N)`，这对于后缀排序来说**效率极低**，无法用于生产环境。
    *   **`_SA_SORTBY_SAIS` (SA-IS 算法)**
        *   **实现:** 调用一个名为 `saisxx` 的函数。SA-IS (Suffix Array induced sorting) 是一种先进的后缀数组构造算法，其时间复杂度为线性 `O(N)`。
        *   **性能:** `O(N)` 的复杂度使其比 `std::sort` 快得多，是现代后缀数组构建的主流算法之一。
    *   **`_SA_SORTBY_DIVSUFSORT` (DivSufSort 算法 - HDiffPatch 默认)**
        *   **实现:** 调用 `divsufsort` 或 `divsufsort64` 函数。DivSufSort (Divide-and-conquer Suffix Sort) 是由 Yuta Mori 开发的另一种高性能后缀排序算法。
        *   **性能:** 尽管理论复杂度不是严格的 `O(N)`，但在实践中，DivSufSort 通常被认为是**最快**的后缀数组构造算法之一，尤其是在处理常见的真实世界数据时。它还内置了对**多线程**的支持，可以通过 `threadNum` 参数进一步加速排序过程。HDiffPatch 选择它作为默认实现，正是因为它在速度上的卓越表现。

*   **模板化 (`template<class TSAInt>`)**
    *   这个函数是一个模板函数，`TSAInt` 可以是 `TInt32` (32位整数) 或 `TInt` (64位整数)。
    *   这使得同一个函数体可以被用于处理32位和64位两种不同大小的后缀数组，完美地配合了 `resetSuffixString` 中的内存自适应逻辑。
    *   在 `_SA_SORTBY_DIVSUFSORT` 的实现中，通过 `sizeof(TSAInt)` 来判断应该调用 `divsufsort` (32位版本) 还是 `divsufsort64` (64位版本)，这是模板元编程的一个典型应用。

### **代码解析**

```cpp
/**
 * @brief 调用底层库来创建（即排序）一个后缀数组。
 * @tparam TSAInt      后缀数组索引的整数类型（TInt32 或 TInt）。
 * @param src, src_end 指向源数据的指针。
 * @param out_sstring  输出参数，用于存储生成的后缀数组。
 * @param threadNum    传递给底层排序库的线程数。
 */
template<class TSAInt>
static void _suffixString_create(const TChar* src,const TChar* src_end,
                                 std::vector<TSAInt>& out_sstring,size_t threadNum){
    size_t size=(size_t)(src_end-src);
    if (size<0) // 基础检查
        throw std::runtime_error("suffixString_create() error.");
    
    // 调整输出向量的大小以容纳所有索引。
    out_sstring.resize(size);
    if (size<=0) return; // 如果源数据为空，则无需操作。

#ifdef _SA_SORTBY_STD_SORT
    // --- 后端实现1：使用 std::sort (用于测试，性能差) ---
    // 初始化索引数组 [0, 1, 2, ..., size-1]
    for (TSAInt i=0;i<size;++i)
        out_sstring[i]=i;
    int rt=0;
    try {
        // 使用自定义比较器对索引进行排序。
        std::sort<TSAInt*,const TSuffixString_compare&>(&out_sstring[0],&out_sstring[0]+size,
                                                        TSuffixString_compare(src,src_end));
    } catch (...) {
        rt=-1;
    }
#endif

#ifdef _SA_SORTBY_SAIS
    // --- 后端实现2：使用 SA-IS 算法 ---
    TSAInt rt=saisxx(src,&out_sstring[0],size);
#endif

#ifdef _SA_SORTBY_DIVSUFSORT
    // --- 后端实现3：使用 DivSufSort 算法 (HDiffPatch 默认) ---
    saint_t rt=-1;
    if (sizeof(TSAInt)==8) // 根据模板参数的类型大小，选择调用64位版本
        rt=divsufsort64(src,(saidx64_t*)&out_sstring[0],(saidx64_t)size,(int)threadNum);
    else if (sizeof(TSAInt)==4) // 或调用32位版本
        rt=divsufsort(src,(saidx32_t*)&out_sstring[0],(saidx32_t)size,(int)threadNum);
#endif
   
    // 检查底层排序函数是否成功返回。
   if (rt!=0)
        throw std::runtime_error("suffixString_create() error.");
}
```

## TSuffixString::lower_bound & _lower_bound **优化的后缀数组搜索**

这组函数实现了 `TSuffixString` 类的 `lower_bound` 操作。这可以说是整个 `getBestMatch` 搜索循环中**对性能最关键**的部分。它的目的是在已排序的后缀数组中，找到一个给定字符串可以插入而不破坏排序的第一个位置。该实现不是一个简单的二分查找，而是一个为追求极致速度而设计的多层次、高度复杂的系统。

### **原理与设计思路解析**

其设计基于两个主要原则：在执行搜索之前**积极地缩减搜索空间**，然后对这个缩小的空间使用**高度优化的二分查找算法**。

*   **原则一：多层次搜索空间缩减 (在 `TSuffixString::lower_bound` 中)**
    该函数并非对整个、可能非常庞大的后缀数组进行二分查找，而是应用了一系列越来越精确（但也略微更慢）的过滤器，以戏剧性地缩小需要搜索的范围。

    1.  **第0层 (可选): 布隆过滤器 (`_SSTRING_FAST_MATCH`)**:
        *   这是最快的“提前出局”检查。布隆过滤器是一种概率性数据结构，可以非常迅速地判断一个元素**绝对不**存在于一个集合中。
        *   当函数被调用时，它首先计算输入字符串的哈希值，并检查布隆过滤器。如果过滤器显示该字符串不存在，函数可以立即返回 `-1`，完全不必执行任何搜索。这对于新文件中那些在旧文件中没有任何匹配的部分非常有效。

    2.  **第1层: 双字符范围缓存 (`m_cached2char_range`)**:
        *   这是最有效的搜索空间缩减手段。在 `TSuffixString` 构建期间，代码会预先计算并存储**每一种可能的双字符前缀**（例如 "aa", "ab", "ac"...）在后缀数组中的确切起始和结束位置。
        *   当 `lower_bound` 被调用时，它查看输入字符串的前两个字符，用它们计算出一个索引，然后直接从缓存表中查找到它需要搜索的、非常小的后缀数组**切片**。
        *   **影响:** 举例来说，无需对一个包含1亿个元素的数组进行二分查找，此举可能将搜索空间缩小到仅几千个元素，从而带来巨大的速度提升。

    3.  **第2层: 单字符范围缓存 (`m_cached1char_range`)**:
        *   当双字符缓存未使用或输入字符串太短时，这是一个备选方案。它基于相同的原理，但仅使用第一个字符来缩小搜索范围。它的精确度低于双字符缓存，但仍然远胜于搜索整个数组。

*   **原则二：LCP感知的二分查找 (在 `_lower_bound` 中)**
    即使在搜索空间被缩小后，二分查找本身也经过了高度优化。一个简单的字符串二分查找会一遍又一遍地重复比较相同的前缀。此实现避免了这种情况。
    *   **核心思想:** 它跟踪当前搜索范围的左边界 (`left_eq`) 和右边界 (`right_eq`) 处的**最长公共前缀 (Longest Common Prefix, LCP)**。
    *   当将输入字符串与范围的中间元素进行比较时，它知道前 `min(left_eq, right_eq)` 个字符是**保证匹配**的，因此它可以**跳过**对这些字符的比较，直接从该偏移量之后开始逐字节比较。
    *   每次比较后，它会用刚刚发现的、新的更长的LCP来更新 `left_eq` 或 `right_eq`。这种“LCP感知”或“前缀感知”的方法显著减少了字节比较的次数，尤其是在处理高度重复的数据时。

### **代码解析**

```cpp
// 一个用于32位后缀数组的简单包装器。
static TInt _lower_bound_TInt32(/*...参数...*/) {
    // 它调用主模板函数，并将结果从指针调整为索引。
    return _lower_bound(rbegin,rend,str,str_end,src_begin,src_end,min_eq) - SA_begin;
}

TInt TSuffixString::lower_bound(const TChar* str,const TChar* str_end)const{
    // --- 第0层: 布隆过滤器 (最快检查) ---
#if (_SSTRING_FAST_MATCH>0)
    // 如果启用了快速匹配且布隆过滤器检查未命中...
    if (m_isUsedFastMatch&&(!m_fastMatch.isHit(TFastMatchForSString::getHash(str))))
        return -1; // ...立即返回。字符串绝对不在旧数据中。
    #define kMinStrLen _SSTRING_FAST_MATCH
#else
    #define kMinStrLen 2
#endif

    // --- 第1层 & 第2层: 缓存范围表 (搜索空间缩减) ---
    if ((kMinStrLen>=2)&(m_cached2char_range!=0)){
        // 如果可用且字符串足够长，则使用双字符缓存。
        size_t cc=((size_t)str[1]) | (((size_t)str[0])<<8); // 将前两个字符组合成一个索引。
        size_t r0,r1; // 搜索范围的起始和结束。
        // 从缓存表中查找预先计算好的范围。
        if (isUseLargeSA()){ // 处理64位SA索引。
            r0=((TInt*)m_cached2char_range)[cc]*sizeof(TInt);
            r1=((TInt*)m_cached2char_range)[cc+1]*sizeof(TInt);
        }else{ // 处理32位SA索引。
            r0=((TInt32*)m_cached2char_range)[cc]*sizeof(TInt32);
            r1=((TInt32*)m_cached2char_range)[cc+1]*sizeof(TInt32);
        }
        // 在*急剧缩小*的范围上调用核心二分查找函数。
        return m_lower_bound((TChar*)m_cached_SA_begin+r0,(TChar*)m_cached_SA_begin+r1,
                             str,str_end,m_src_begin,m_src_end,m_cached_SA_begin,2);
    }else if (kMinStrLen>0){
        // 回退到单字符缓存。
        size_t c=str[0];
        return m_lower_bound(m_cached1char_range[c],m_cached1char_range[c+1],
                             str,str_end,m_src_begin,m_src_end,m_cached_SA_begin,1);
    }else{
        return -1; // 在kMinStrLen > 0的情况下不应发生
    }
}

/**
 * @brief 核心的LCP感知二分查找算法。
 * @tparam T 后缀数组的整数类型 (TInt32 或 TInt)。
 */
template <class T>
inline static const T* _lower_bound(const T* rbegin,const T* rend,
                                    const TChar* str,const TChar* str_end,
                                    const TChar* src_begin,const TChar* src_end,
                                    size_t min_eq=0){
#ifdef _SA_MATCHBY_STD_LOWER_BOUND
    // 使用标准库的替代实现 (通常较慢)。
    return std::lower_bound<>(...);
#else
    // 自定义的高性能实现。
    size_t left_eq=min_eq;  // 左边界的LCP。
    size_t right_eq=min_eq; // 右边界的LCP。
    while (size_t len=(size_t)(rend-rbegin)) {
        const T* mid=__select_mid(rbegin,len); // 找到中间元素。
        // 确定对于当前整个范围，有多少前缀字节已知是相等的。
        size_t eq_len=(left_eq<=right_eq)?left_eq:right_eq;
        
        // --- 快进比较 ---
        // 从已知公共前缀*之后*的第一个字节开始比较。
        const TChar* vs=str+eq_len;
        const TChar* ss=src_begin+(*mid)+eq_len;
        bool is_less;
        while (true) { // 逐字节比较循环
            if (vs==str_end) { is_less=false; break; }; // str是mid字符串的前缀
            if (ss==src_end) { is_less=true;  break; }; // mid字符串是str的前缀
            TInt sub=(*ss)-(*vs);
            if (!sub) { // 字节相等
                ++vs; ++ss; ++eq_len; // 继续比较下一个字节
                // 性能启发：如果字符串匹配了8KB，就认为它们足够相等并返回。
                const int kMaxCmpLength_forLimitRangeDiff=1024*8; 
                if (eq_len<kMaxCmpLength_forLimitRangeDiff)
                    continue;
                else
                    return mid;
            }else{ // 字节不同
                is_less=(sub<0);
                break;
            }
        }
        
        // --- 缩小范围并更新LCP ---
        if (is_less){ // mid字符串 < str，所以在右半部分搜索
            left_eq=eq_len; // 新左边界的LCP就是我们刚刚找到的。
            rbegin=mid+1;
        }else{ // str <= mid字符串，所以在左半部分搜索
            right_eq=eq_len; // 新右边界的LCP就是我们刚刚找到的。
            rend=mid;
        }
    }
    return rbegin; // 返回最终位置。
#endif
}
```
