---
title: HPatch
categories:
  - hpatch
tags:
  - hpatch
abbrlink: b701678d
date: 2025-10-04 18:57:17
---
<meta name="referrer" content="no-referrer" />

[TOC]

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/72a0f919146843cba861601b64feb07c.png)

# HDiffPatch\libHDiffPatch\HPatch\patch.c
## packUInt & hpatch_packUIntWithTag **可变长度整数编码**

这组函数实现了一种高效的、类似于 **LEB128 (Little-Endian Base 128)** 或 **VLQ (Variable-length quantity)** 的整数编码方案。其核心目标是：**用更少的字节来表示小数值，用更多的字节来表示大数值**，从而在数据流中实现对数值本身的压缩。

### **原理与设计思路解析**

您提供的注释已经非常精彩地概括了其编码方案，我将在此基础上做更详细的解析。

*   **编码规则 (Encoding Scheme):**
    算法将一个整数 `uValue` 拆分成多个字节进行存储。每个字节都由两部分组成：

    1.  **数据位 (Data Bits):** 每个字节的**低7位**用于存储 `uValue` 的一部分数据。
    2.  **连续标志位 (Continuation Bit):** 每个字节的**最高位 (MSB)** 作为标志位。
        *   如果 MSB 是 `1`，表示后面**还有**字节属于这个整数。
        *   如果 MSB 是 `0`，表示这是这个整数的**最后一个**字节。

*   **带标签的编码 (`hpatch_packUIntWithTag`)**
    HDiffPatch 的实现比标准的 LEB128 更进一步，它允许在第一个字节中**嵌入额外的标志位 (tag)**。

    *   **第一个字节的结构:**
        ```
        Bit:  | 7 | 6 ... (8-kTagBit) | (7-kTagBit) | (6-kTagBit) ... 0 |
        Desc: | C |      highTag      |      C'     |    uValue_part1   |
        ```
        *   **`C'` (Bit `7-kTagBit`):** 这是**主连续标志位**。如果为 `1`，表示后面还有字节。
        *   **`highTag` (Bits `(8-kTagBit)` to `6`):** 这是用户可以自定义的 `kTagBit` 个标志位，用于在上层协议中传递额外信息（例如 `isNullSubDiff` 标志）。
        *   **`uValue_part1`:** 第一个字节中用于存储数值本身的数据位。

    *   **后续字节的结构:**
        ```
        Bit:  |   7   | 6 ... 0 |
        Desc: |   C   | uValue_partN |
        ```
        *   **`C` (Bit 7):** **连续标志位**。
        *   **`uValue_partN`:** 后续字节用于存储数值的数据位。

*   **`hpatch_packUIntWithTag` 的实现逻辑 (大端序输出)**
    这个函数的实现非常巧妙，它生成的是一个**大端序 (Big-Endian)** 的可变长度整数，这意味着数值的**最高有效位**被编码在第一个字节中。

    1.  **分解 (Decomposition):**
        *   `while (uValue > kMaxValueWithTag)`: 循环不断地从 `uValue` 中取出**低7位** (`uValue & ((1<<7)-1)`)，并将其**逆序**存入一个临时的 `codeBuf` 缓冲区中。
        *   `uValue >>= 7`: 将 `uValue` 右移7位，准备处理下一部分。
        *   这个循环结束后，`uValue` 中只剩下最高的一部分数据，而 `codeBuf` 中则逆序存储了所有低位部分。

    2.  **编码第一个字节 (Head Byte):**
        *   `*pcode = (TByte)( (TByte)uValue | (highTag<<(8-kTagBit)) | ... );`
        *   将 `uValue` 的剩余高位部分，与 `highTag` 和主连续标志位 `C'` 通过位运算组合起来，形成第一个字节。

    3.  **编码后续字节 (Tail Bytes):**
        *   `while (codeBuf != codeEnd)`: 循环**从后向前**遍历 `codeBuf`（即按正确的顺序）。
        *   `*pcode = (*codeEnd) | (TByte)(((codeBuf!=codeEnd)?1:0)<<7);`
        *   将 `codeBuf` 中的7位数据与连续标志位 `C` 组合起来，形成后续的字节。

*   **C++ 包装器 (`packUIntWithTag`, `packUInt`)**
    *   `pack_uint.h` 中的 `packUInt...` 函数是对底层C函数 `hpatch_packUIntWithTag` 的C++风格**包装器**。
    *   它们负责处理 `std::vector` 的内存管理，使得调用者可以方便地将编码后的字节流追加到 `vector` 的末尾，而无需手动管理缓冲区指针和大小。`packUInt` 是 `packUIntWithTag` 的一个特例，它默认 `tag` 和 `kTagBit` 都为0。

### **代码解析**

```c
// Variable-length positive integer encoding scheme ...
// (注释详细描述了编码格式)

/**
 * @brief 将一个64位无符号整数 uValue 进行可变长度编码，并允许在第一个字节中嵌入 kTagBit 个高位标志。
 * @param out_code         输入输出参数，指向用于写入的缓冲区指针，函数会推进此指针。
 * @param out_code_end     缓冲区的结束边界，用于安全检查。
 * @param uValue           待编码的整数。
 * @param highTag          要嵌入的高位标志值。
 * @param kTagBit          标志位的数量。
 * @return hpatch_BOOL     成功返回 hpatch_TRUE。
 */
hpatch_BOOL hpatch_packUIntWithTag(TByte** out_code,TByte* out_code_end,
                                   hpatch_StreamPos_t uValue,hpatch_uint highTag,
                                   const hpatch_uint kTagBit){
    TByte*          pcode=*out_code;
    // 计算第一个字节能容纳的最大数值
    const hpatch_StreamPos_t kMaxValueWithTag=((hpatch_StreamPos_t)1<<(7-kTagBit))-1;
    TByte           codeBuf[hpatch_kMaxPackedUIntBytes]; // 临时缓冲区
    TByte*          codeEnd=codeBuf;

    // --- 1. 分解阶段 ---
    // 只要 uValue 还大于第一个字节的容量
    while (uValue>kMaxValueWithTag) {
        // 取出 uValue 的低7位，存入临时缓冲区
        *codeEnd=uValue&((1<<7)-1); ++codeEnd;
        // uValue 右移7位
        uValue>>=7;
    }
    
    // 检查输出缓冲区是否足够大
#ifdef __RUN_MEM_SAFE_CHECK
    if ((out_code_end-pcode)<(1+(codeEnd-codeBuf))) return _hpatch_FALSE;
#endif

    // --- 2. 编码第一个字节 ---
    // 组合：uValue的剩余高位 | 用户自定义的tag | 主连续标志位
    *pcode=(TByte)( (TByte)uValue | (highTag<<(8-kTagBit))
                   | (((codeBuf!=codeEnd)?1:0)<<(7-kTagBit))  );
    ++pcode;

    // --- 3. 编码后续字节 ---
    // 从后向前遍历临时缓冲区（即按正序处理数值的低位部分）
    while (codeBuf!=codeEnd) {
        --codeEnd;
        // 组合：7位数据 | 连续标志位
        *pcode=(*codeEnd) | (TByte)(((codeBuf!=codeEnd)?1:0)<<7);
        ++pcode;
    }

    *out_code=pcode; // 更新外部指针
    return hpatch_TRUE;
}
```
