---
title: hdiffpatch
categories:
  - hpatch
tags:
  - hpatch
abbrlink: fd53b379
date: 2025-10-03 09:12:19
---
[TOC]

# hdiffi.cpp
## hdiffi & hdiffi_in_mem **内存模式下的补丁生成**

这是 HDiffPatch 工具集中负责**制作补丁**的核心业务逻辑。`hdiffi` 是一个上层封装，而 `hdiffi_in_mem` 实现了将整个文件加载到内存中进行差分的核心流程。

### **原理与设计思路解析**

这部分代码的 overarching goal 是协调整个补丁制作过程，它是一个高层次的控制器，调用底层的差分算法和压缩库来完成任务。

*   **核心策略：内存模式 (In-Memory)**
    *   函数名 `hdiffi_in_mem` 明确指出了其核心设计：**将旧文件和新文件完全读入内存**中，然后再进行比较和差分。
    *   **优点：** 差分算法（特别是基于后缀数组的算法）需要对数据进行大量的、非线性的随机访问。在内存中操作可以极大地提升算法的执行速度，避免了缓慢的磁盘I/O。
    *   **缺点：** 这种模式的**内存消耗巨大**，它要求系统的可用RAM必须能同时容纳整个旧文件和新文件。因此，它不适用于超出内存容量的超大文件。

*   **模块化与可配置性**
    *   代码通过 `TDiffiSets` 结构体和 `compressPlugin` 接口，实现了高度的模块化和可配置性。
    *   **`TDiffiSets`**: 这个结构体像一个“配置单”，它告诉 `hdiffi_in_mem` 要执行哪些操作（`isDoDiff`），是否要生成用于原地更新的补丁（`isDiffForInplacePatch`），以及差分算法的一些内部参数（`matchScore`, `threadNum` 等）。
    *   **`compressPlugin`**: 差分过程和压缩过程是**解耦**的。核心算法生成原始的、未压缩的差分数据，然后将这些数据交给 `compressPlugin` 进行压缩。这种设计使得替换或添加新的压缩算法（如zlib, lzma, zstd）变得非常容易，而无需改动核心差分逻辑。

*   **两种差分模式的调用**
    *   函数根据配置 `diffSets.isDiffForInplacePatch` 来决定调用两个不同的底层差分函数之一：
        1.  `create_lite_diff`: 生成标准的 `lite` 格式补丁。
        2.  `create_inplace_lite_diff`: 生成为 In-place（原地更新）模式设计的补丁。这个函数会执行更复杂的分析，计算出安全更新所需的核心参数 `extraSafeSize` 并将其写入补丁头部。

*   **健壮性设计：补丁自校验 (`isDoPatchCheck`)**
    *   这是一个非常出色的健壮性设计。在生成补丁文件后，代码并**不**假定生成的补丁是完美的。
    *   如果 `isDoPatchCheck` 为真，程序会立刻**加载刚刚生成的补丁文件**，并**模拟一次打补丁的过程**，将内存中的 `oldMem` 应用补丁，然后将结果与内存中的 `newMem` 进行比对。
    *   这个**自测试**步骤可以立即验证补丁的正确性，极大地增加了可靠性，确保分发出去的补丁文件是有效的。

*   **错误处理 (`check` 宏和 `goto clear`)**
    *   代码使用了 `check(condition, error_code, message)` 宏和 `goto clear` 语句，这是一种在C/C++中管理复杂流程和资源清理的经典模式。
    *   无论在哪一步操作（如读文件、写文件、差分）失败，`check` 宏都会将 `result` 设置为相应的错误码，并立即 `goto clear`。
    *   `clear:` 标签后的代码块是**唯一的退出路径**，它负责统一执行资源释放操作（比如关闭文件流 `hpatch_TFileStreamOutput_close`），从而避免了在多个错误分支中重复编写清理代码，保证了资源的正确释放。

### **代码解析**

```cpp
/**
 * @brief 在内存中执行差分操作的核心函数。
 */
static int hdiffi_in_mem(const char* oldFileName,const char* newFileName,const char* outDiffFileName,
                         const hdiffi_TCompress* compressPlugin,const TDiffiSets& diffSets){
    double diff_time0=clock_s(); // 开始计时
    int    result=HDIFFI_SUCCESS; // 初始化返回结果为成功
    int    _isInClear=hpi_FALSE; // 用于防止在清理阶段重复关闭文件的标志
    hpatch_TFileStreamOutput diffData_out; // 文件输出流对象
    hpatch_TFileStreamOutput_init(&diffData_out);
    
    // TAutoMem 是一个RAII封装，用于自动管理内存，确保在函数退出时内存被释放。
    hdiff_private::TAutoMem oldMem(0);
    hdiff_private::TAutoMem newMem(0);

    // --- 1. 加载文件到内存 ---
    if (oldFileName&&(strlen(oldFileName)>0))
        // check宏：如果readFileAll失败，则设置result为HDIFFI_OPENREAD_ERROR，打印消息，并goto clear。
        check(readFileAll(oldMem,oldFileName),HDIFFI_OPENREAD_ERROR,"open oldFile");
    check(readFileAll(newMem,newFileName),HDIFFI_OPENREAD_ERROR,"open newFile");
    printf("oldDataSize : %" PRIu64 "\nnewDataSize : %" PRIu64 "\n",
           (hpatch_StreamPos_t)oldMem.size(),(hpatch_StreamPos_t)newMem.size());

    // --- 2. 执行差分 ---
    if (diffSets.isDoDiff){
        if (diffSets.isDoDiff&&diffSets.isDiffForInplacePatch){
            // 如果是为原地更新生成补丁，打印额外信息。
            printf("\nhdiffi created outDiffFile is ");
            if (isInplaceASets(diffSets.inplaceSets))
                printf("inplaceA");
            else //if (isInplaceBSets(diffSets.inplaceSets))
                printf("inplaceB");
            printf(" format for inpalce-patch!\n");
            printf("  extraSafeSize     : %" PRIu64 "\n",(hpatch_StreamPos_t)diffSets.inplaceSets.extraSafeSize);
            printf("\n");
        }

        std::vector<hpi_byte> outDiffData; // 存储生成的补丁数据。
        try {
            if (diffSets.isDiffForInplacePatch){
                // 调用 in-place 模式的差分函数。
                create_inplace_lite_diff(newMem.data(),newMem.data_end(),oldMem.data(),oldMem.data_end(),
                                         outDiffData,diffSets.inplaceSets,compressPlugin,(int)diffSets.matchScore,
                                         diffSets.isUseBigCacheMatch?true:false,diffSets.threadNum);
            }else{
                // 调用标准 lite 模式的差分函数。
                create_lite_diff(newMem.data(),newMem.data_end(),oldMem.data(),oldMem.data_end(),
                                 outDiffData,compressPlugin,(int)diffSets.matchScore,
                                 diffSets.isUseBigCacheMatch?true:false,0,diffSets.threadNum);
            }
        }catch(const std::exception& e){
            // 捕获C++异常，转换为hdiffi的错误处理流程。
            check(false,HDIFFI_DIFF_ERROR,"diff run error: "+e.what());
        }

        // --- 3. 写入补丁文件 ---
        {
            check(hpatch_TFileStreamOutput_open(&diffData_out,outDiffFileName,outDiffData.size()),
                HDIFFI_OPENWRITE_ERROR,"open out diffFile");
            check(diffData_out.base.write(&diffData_out.base,0,outDiffData.data(),outDiffData.data()+outDiffData.size()),
                HDIFFI_OPENWRITE_ERROR,"write diffFile");
            check(hpatch_TFileStreamOutput_close(&diffData_out),HDIFFI_FILECLOSE_ERROR,"out diffFile close");
        }
        printf("diffDataSize: %" PRIu64 "\n",(hpatch_StreamPos_t)outDiffData.size());
        printf("hdiffi  time: %.3f s\n",(clock_s()-diff_time0));
        printf("  out diff file ok!\n");
    }

    // --- 4. (可选) 补丁自校验 ---
    if (diffSets.isDoPatchCheck){
        double patch_time0=clock_s();
        hdiff_private::TAutoMem diffMem(0);
        {
            printf("\nload diffFile for test by patch:\n");
            check(readFileAll(diffMem,outDiffFileName),
                HDIFFI_OPENREAD_ERROR,"open diffFile for test");
            printf("diffDataSize: %" PRIu64 "\n",(hpatch_StreamPos_t)diffMem.size());
        }
        
        // NOTE: if(0) 块中的代码是禁用的，说明默认使用 hpatchi 的实现来做校验。
        if (0){ 
            // ... (disabled code)
        }else{ // 使用 HPatchLite 库进行校验。
            check(check_lite_diff_by_hpatchi(newMem.data(),newMem.data_end(),oldMem.data(),oldMem.data_end(),
                                             diffMem.data(),diffMem.data_end()),
                  HDIFFI_PATCH_ERROR,"check_lite_diff_by_hpatchi()");
        }
        printf("hpatchi time: %.3f s\n",(clock_s()-patch_time0));
        printf("  patch check diff data ok!\n");
    }

clear: // 统一的清理代码块
    _isInClear=hpi_TRUE;
    check(hpatch_TFileStreamOutput_close(&diffData_out),HDIFFI_FILECLOSE_ERROR,"out diffFile close");
    return result;
}

/**
 * @brief hdiffi 工具的顶层入口函数，主要负责打印信息和调用核心函数。
 */
int hdiffi(const char* oldFileName,const char* newFileName,const char* outDiffFileName,
           const hdiffi_TCompress* compressPlugin,const TDiffiSets& diffSets){
    double time0=clock_s(); // 开始总计时
    // 打印输入输出文件信息。
    std::string fnameInfo=std::string("old : \"")+oldFileName+"\"\n"
                                     +"new : \""+newFileName+"\"\n"
                             +(diffSets.isDoDiff?"out : \"":"test: \"")+outDiffFileName+"\"\n";
    hpatch_printPath_utf8(fnameInfo.c_str());
    
    if (diffSets.isDoDiff) {
        // 打印使用的压缩插件类型。
        const char* compressType="";
        if (compressPlugin->compress) compressType=compressPlugin->compress->compressType();
        printf("hdiffi run with compress plugin: \"%s\"\n",compressType);
    }
    
    int exitCode;
    // 确认配置为内存模式。
    assert(diffSets.isDiffInMem);
    // 调用核心处理函数。
    exitCode=hdiffi_in_mem(oldFileName,newFileName,outDiffFileName,compressPlugin,diffSets);
    
    // 如果执行了差分和校验，打印总耗时。
    if (diffSets.isDoDiff && diffSets.isDoPatchCheck)
        printf("\nall   time: %.3f s\n",(clock_s()-time0));
    return exitCode;
}
```

# pack_uint.h
```cpp
// pack_uint.h 中的 C++ 包装器

// C++ 包装器，将编码结果追加到 std::vector
inline static void packUIntWithTag(std::vector<unsigned char>& out_code,hpatch_StreamPos_t uValue,
                                   int highTag,const int kTagBit){
    unsigned char  codeBuf[hpatch_kMaxPackedUIntBytes];
    unsigned char* codeEnd=codeBuf;
    // 调用底层的C函数，将结果写入一个临时栈缓冲区
    if (!hpatch_packUIntWithTag(&codeEnd,codeBuf+hpatch_kMaxPackedUIntBytes,uValue,highTag,kTagBit))
        throw std::runtime_error("packUIntWithTag() hpatch_packUIntWithTag() error!");
    // 将栈缓冲区的内容插入到 vector 的末尾
    out_code.insert(out_code.end(),codeBuf,codeEnd);
}

// packUIntWithTag 的特例，tag和kTagBit都为0
inline static void packUInt(std::vector<unsigned char>& out_code,hpatch_StreamPos_t uValue){
    packUIntWithTag(out_code,uValue,0,0);
}
```

# diff.cpp
## create_lite_diff **HPatchLite 差分流程总调度器**

这是 HDiffPatch 中负责生成 `HPatchLite` 格式补丁的**总调度函数**。它本身不包含复杂的算法实现，而是像一个“项目经理”，按照固定的流程编排和调用下游的各个核心组件（如 `get_diff` 和 `serialize_lite_diff`）来协同完成整个差分任务。

### **原理与设计思路解析**

这个函数的设计体现了典型的**“关注点分离” (Separation of Concerns)** 原则。它将复杂的差分过程分解为两个主要阶段：**1. 差异发现** 和 **2. 格式序列化**，并分别委托给专门的函数去处理。

*   **执行流程 (Pipeline):**

    1.  **阶段一：差异发现 (`get_diff`)**
        *   `get_diff(diff,covers,...);`
        *   这是整个流程中计算最密集、算法最核心的部分。函数将新旧文件数据以及一些算法调优参数（如 `kMinSingleMatchScore`）传递给 `get_diff`。
        *   `get_diff` 内部会执行我们之前详细分析过的、基于后缀数组的复杂匹配流水线，最终返回一个高度优化的、代表新旧文件匹配关系的**抽象指令集**——`covers` 向量。
        *   `kMinSingleMatchScore-_kMatchScore_optim4bin` 这个细节暗示了内部评分机制的一些微调，可能是为了在处理二进制文件时获得更好的匹配结果而进行的特别优化。

    2.  **阶段二：处理尾部数据 (Sentinel Cover)**
        *   `if (newPosEnd<newSize) covers.push_back(...)`
        *   `get_diff` 找出的 `covers` 可能不会覆盖到新文件的末尾（例如，新文件的最后一部分是全新的内容）。
        *   这几行代码的作用是检查这种情况，并在 `covers` 列表的末尾添加一个长度为 `0` 的**“哨兵” cover**。这个哨兵为后续的序列化函数提供了一个明确的结束点，告诉它：“在最后一个真实 `cover` 和新文件末尾之间的所有数据，都应作为‘新数据’处理。”

    3.  **阶段三：获取 In-place 模式参数**
        *   `if (listener&&listener->getInplacePatchExtraSafeSize){ ... }`
        *   这是实现 **In-place 模式**和**代码复用**的关键。`create_lite_diff` 本身并不关心是否要生成 In-place 补丁。
        *   它通过检查外部传入的 `listener` 接口是否实现了 `getInplacePatchExtraSafeSize` 回调。
        *   如果实现了（通常是 `create_inplace_lite_diff` 函数在调用它时传入的），它就会调用这个回调来获取 `extraSafeSize` 参数，并设置相应的标志位。
        *   这使得同一个差分核心 (`get_diff`) 和序列化逻辑可以被 `lite` 和 `in-place` 两种模式**共享**，极大地提高了代码的复用率。

    4.  **阶段四：格式序列化 (`serialize_lite_diff`)**
        *   `serialize_lite_diff(diff,covers,out_lite_diff,...);`
        *   这是流程的最后一步。它扮演了**“格式化器”**的角色。
        *   它接收 `get_diff` 生成的抽象的 `covers` 列表，以及从 `listener` 获取的 In-place 参数，然后将其严格按照 HPatchLite 格式的规范，转换成最终的二进制字节流，并存入输出缓冲区 `out_lite_diff`。

### **代码解析**

```cpp
/**
 * @brief 创建一个 HPatchLite 格式的补丁。
 * @param newData, newData_end 指向新数据的指针。
 * @param oldData, oldData_end 指向旧数据的指针。
 * @param out_lite_diff        输出参数，用于存储生成的补丁字节流。
 * @param compressPlugin       （可选）压缩插件。
 * @param kMinSingleMatchScore 匹配的最低分数阈值。
 * @param isUseBigCacheMatch   是否使用大的后缀数组缓存。
 * @param listener             （可选）用于高级功能（如in-place）的回调监听器。
 * @param threadNum            用于并行计算的线程数。
 */
void create_lite_diff(const unsigned char* newData,const unsigned char* newData_end,
                      const unsigned char* oldData,const unsigned char* oldData_end,
                      std::vector<hpi_byte>& out_lite_diff,const hdiffi_TCompress* compressPlugin,
                      int kMinSingleMatchScore,bool isUseBigCacheMatch,
                      ILiteDiffListener* listener,size_t threadNum){
    // 定义一个内部调优参数，可能用于优化二进制文件的匹配评分。
    static const int _kMatchScore_optim4bin=6;
    
    // 初始化一个包含新旧文件数据的上下文对象。
    TDiffData diff(newData,newData_end,oldData,oldData_end);
    
    std::vector<TOldCover> covers; // 用于存储找到的最优匹配块（covers）。

    // --- 1. 差异发现阶段 ---
    // 调用核心的差异发现引擎，填充 covers 向量。
    get_diff(diff,covers,kMinSingleMatchScore-_kMatchScore_optim4bin,isUseBigCacheMatch,listener,0,threadNum);

    // --- 2. 处理尾部数据 ---
    size_t oldPosEnd=0;
    size_t newPosEnd=0;
    if (!covers.empty()){
        // 获取最后一个匹配块的结束位置。
        const TOldCover& c=covers.back();
        oldPosEnd=c.oldPos+c.length;
        newPosEnd=c.newPos+c.length;
    }
    const size_t newSize=newData_end-newData;
    // 如果最后一个匹配块没有到达新文件的末尾，则添加一个长度为0的“哨兵”cover。
    if (newPosEnd<newSize)
        covers.push_back(TOldCover(oldPosEnd,newSize,0));

    // --- 3. (可选) 获取 In-place 模式的额外信息 ---
    TInlpacePatchSets inlpacePatchSets={}; // 初始化 in-place 参数集。
    if (listener&&listener->getInplacePatchExtraSafeSize){
        // 通过回调向 listener 查询 in-place 参数。
        if (listener->getInplacePatchExtraSafeSize(listener,&inlpacePatchSets.extraSafeSize))
            inlpacePatchSets.isInplacePatchByExtra=true;
    }
    
    // --- 4. 格式序列化阶段 ---
    // 调用序列化函数，生成最终的补丁字节流。
    serialize_lite_diff(diff,covers,out_lite_diff,compressPlugin,inlpacePatchSets);
}
```

## get_diff **核心差异发现引擎**

这是 HDiffPatch 中负责**发现**两个二进制数据块之间差异的核心算法函数。它不关心最终的补丁格式，只专注于一个目标：找到一组最优的匹配数据块（`covers`），为后续的序列化步骤提供原始“指令”。

### **原理与设计思路解析**

`get_diff` 是一个高度优化、可扩展且复杂的函数，其核心是基于**后缀字符串（Suffix String）**的匹配算法。整个函数可以看作一个可定制的**差异发现流水线 (Pipeline)**。

*   **核心数据结构：后缀字符串 (`TSuffixString`)**
    *   这是整个差异发现过程的基石。后缀字符串（通常通过后缀数组或类似结构实现）是一种经过预处理的数据结构，它允许在`O(logN)`或`O(1)`的时间复杂度内，极快地找到一个字符串（来自新文件）在另一个巨大字符串（旧文件）中的所有匹配项。
    *   **构建成本高昂：** 为旧文件数据构建后缀字符串是一个计算密集且消耗内存的操作。因此，代码设计了一个优化：`get_diff` 可以接受一个外部传入的、已经构建好的 `sstring`。这样，在需要将同一个旧文件与多个不同新文件进行比较的场景下，昂贵的后缀字符串构建过程只需执行一次。如果没有传入，函数会自己创建一个临时的 `_sstring_default`。

*   **差异发现流水线 (Pipeline)**
    `get_diff` 的执行流程不是一步到位，而是分为多个阶段，并且在关键阶段提供了**回调钩子 (Hooks)**，通过 `listener` 接口允许调用者介入和定制算法行为。

    1.  **阶段一：后缀字符串准备**
        *   检查是否需要自行构建后缀字符串。如果需要，则调用 `_sstring_default.resetSuffixString()`，此过程支持多线程以加快速度。

    2.  **阶段二：初次匹配搜索 (`first_search_and_dispose_cover_MT`)**
        *   这是流水线的主工作阶段。它利用构建好的后缀字符串，在“新数据”中滑动，为每个位置在“旧数据”中寻找最佳匹配。
        *   它会生成一个初步的 `covers` 列表。这个过程是多线程的（`_MT`），以充分利用多核CPU。

    3.  **阶段三：(可选) 重新搜索 (`research_cover` Hook)**
        *   在初次搜索后，代码提供了一个 `research_cover` 钩子。
        *   调用者（`listener`）可以检查初步的 `covers` 结果，如果觉得不满意（例如，匹配得不够理想或需要满足某些特殊约束），可以要求算法进行**第二次、更精细的搜索**。
        *   这提供了一个强大的机制，用于根据特定需求（比如 in-place 补丁对 `extraSafeSize` 的要求）来调整和优化匹配结果。

    4.  **阶段四：(可选) 手动插入 (`insert_cover` Hook)**
        *   这个钩子甚至允许 `listener` **手动向 `covers` 列表中插入自定义的匹配项**。
        *   这对于处理一些算法难以发现、但基于领域知识可以确定的匹配非常有用。`listener` 甚至可以修改有效的新旧文件大小，给予了极大的灵活性。

    5.  **阶段五：(可选) 最终整理 (`search_cover_finish` Hook)**
        *   在所有搜索和修改结束后，这个最后的钩子允许 `listener` 对最终的 `covers` 列表进行审查和最终修改，例如移除某些不想要的匹配项。

*   **健壮性与约束**
    *   **`_limitCoverLenth`**: 在流水线的多个阶段后，都会调用此函数，确保所有 `cover` 的长度不超过 `maxCoverLen` 限制。
    *   **`assert_covers_safe`**: 在每次修改 `covers` 列表后，都会调用这个断言函数。它会检查 `covers` 列表的内部一致性（如是否有重叠、是否超出边界），是保证算法正确性的重要调试和安全手段。

### **代码解析**

```cpp
static void get_diff(TDiffData& diff,std::vector<TOldCover>& covers,
                     int kMinSingleMatchScore,
                     bool isUseBigCacheMatch,ICoverLinesListener* listener,
                     const TSuffixString* sstring,size_t threadNum,
                     bool isCanExtendCover=true){
    _out_diff_info("  match covers by suffix string ...\n"); // 打印日志
    // 判断 cover 结构体是32位还是64位版本。
    const bool isCover32=sizeof(*covers.data())==sizeof(hpatch_TCover32);
    if (!isCover32)
        assert(sizeof(*covers.data())==sizeof(hpatch_TCover));
    // 通过 listener 获取对单个 cover 长度的限制。
    const hpatch_StreamPos_t maxCoverLen=(listener&&listener->get_limit_cover_length)?
                                            listener->get_limit_cover_length(listener):kDefaultLimitCoverLen;
    { // 这个花括号限定了 _sstring_default 的生命周期，确保其内存被及时回收。
        TSuffixString _sstring_default(isUseBigCacheMatch);
        // --- 阶段一：后缀字符串准备 ---
        if (sstring==0){ // 如果外部没有提供预先构建好的后缀字符串...
            _out_diff_info("    create suffix string ...\n");
            // ...则自己构建一个。支持多线程。
            _sstring_default.resetSuffixString(diff.oldData,diff.oldData_end,threadNum);
            sstring=&_sstring_default; // 后续使用这个临时的对象。
        }

        // --- 阶段二：初次匹配搜索 ---
        _out_diff_info("    search covers by suffix string ...\n");
        // 调用多线程的初次搜索函数，填充 covers 列表。
        first_search_and_dispose_cover_MT(covers,diff,*sstring,kMinSingleMatchScore,listener,threadNum,isCanExtendCover);
        _limitCoverLenth(covers,maxCoverLen); // 应用长度限制。
        assert_covers_safe(covers,diff.newData_end-diff.newData,diff.oldData_end-diff.oldData); // 检查结果的合法性。

        // --- 阶段三：(可选) 重新搜索 ---
        // Hook点：如果 listener 存在并请求重新搜索...
        if (listener&&listener->search_cover_limit&&
            listener->search_cover_limit(listener,covers.data(),covers.size(),isCover32)){
            _out_diff_info("    research covers by limit ...\n");
            // 创建一个用于重新搜索的上下文对象。
            TDiffResearchCover diffResearchCover(diff,covers,*sstring,kMinSingleMatchScore,
                                    listener->get_max_match_deep?listener->get_max_match_deep(listener):kDefaultMaxMatchDeepForLimit,
                                    isCanExtendCover);
            // 将控制权交给 listener，由它来驱动重新搜索的过程。
            listener->research_cover(listener,&diffResearchCover,covers.data(),covers.size(),isCover32);
            diffResearchCover.researchFinish(); // 完成重搜。
            _limitCoverLenth(covers,maxCoverLen); // 再次应用限制和检查。
            assert_covers_safe(covers,diff.newData_end-diff.newData,diff.oldData_end-diff.oldData);
        }
        
        sstring=0; // 解除对临时对象的引用。
        _sstring_default.clear(); // 显式释放后缀字符串占用的内存。
    }
    
    // --- 阶段四：(可选) 手动插入 ---
    // Hook点：如果 listener 存在并实现了 insert_cover...
    if (listener&&listener->insert_cover){
        TDiffInsertCover diffInsertCover(covers);
        hpatch_StreamPos_t newDataSize=(size_t)(diff.newData_end-diff.newData);
        hpatch_StreamPos_t oldDataSize=(size_t)(diff.oldData_end-diff.oldData);
        // listener 可以手动插入 covers，甚至修改文件大小。
        listener->insert_cover(listener,&diffInsertCover,covers.data(),covers.size(),isCover32,
                               &newDataSize,&oldDataSize);
        diff.newData_end=diff.newData+(size_t)newDataSize;
        diff.oldData_end=diff.oldData+(size_t)oldDataSize;
        _limitCoverLenth(covers,maxCoverLen);
        assert_covers_safe(covers,diff.newData_end-diff.newData,diff.oldData_end-diff.oldData);
    }
    
    // --- 阶段五：(可选) 最终整理 ---
    // Hook点：如果 listener 存在并实现了 search_cover_finish...
    if (listener&&listener->search_cover_finish){
        hpatch_StreamPos_t newDataSize=(size_t)(diff.newData_end-diff.newData);
        hpatch_StreamPos_t oldDataSize=(size_t)(diff.oldData_end-diff.oldData);
        size_t newCoverCount=covers.size();
        // listener 可以对最终的 cover 列表进行过滤或截断。
        listener->search_cover_finish(listener,covers.data(),&newCoverCount,isCover32,
                                      &newDataSize,&oldDataSize);
        check(newCoverCount<=covers.size());
        covers.resize(newCoverCount);
        diff.newData_end=diff.newData+(size_t)newDataSize;
        diff.oldData_end=diff.oldData+(size_t)oldDataSize;
        _limitCoverLenth(covers,maxCoverLen);
        assert_covers_safe(covers,diff.newData_end-diff.newData,diff.oldData_end-diff.oldData);
    }
}
```


## serialize_lite_diff **HPatchLite 格式序列化**

这个函数是**格式化器 (Formatter)**。它接收由 `get_diff` 等上游算法生成的高度抽象的 `covers` 列表，并负责将其**转换 (序列化)** 成 **HPatchLite** 格式的标准二进制字节流。这是从“算法结果”到“最终产品（补丁文件）”的关键一步。

### **原理与设计思路解析**

`serialize_lite_diff` 的核心是将一个补丁所需的所有信息——指令、新数据、差分数据——按照 HPatchLite 格式的规定，紧凑地编码并写入一个字节缓冲区。

*   **核心策略：两阶段构建**
    1.  **数据体构建 (Body Construction):**
        *   算法首先创建一个临时的缓冲区 `buf`。
        *   然后，它遍历 `covers` 列表，将**所有**补丁数据（包括 `cover` 的元数据、`covers` 之间的新增数据、以及 `cover` 区域的差分数据）按照特定的编码方式，**全部写入这个临时的 `buf` 中**。
        *   这个 `buf` 代表了补丁文件中**未压缩的数据体**。

    2.  **头部构建与最终组装 (Header Construction and Final Assembly):**
        *   在数据体 `buf` 构建完成后，算法对其进行**压缩**（如果指定了压缩插件），得到 `compress_buf`。
        *   然后，它开始构建文件头。文件头包含了对数据体的**元数据**描述，比如新文件总大小、数据体压缩前/后的大小、版本号、压缩类型等。
        *   最后，它将**文件头**和**压缩后的数据体 (`compress_buf`)** 依次写入最终的输出缓冲区 `out_diff`，完成整个补丁文件的构建。

*   **数据体 (`buf`) 的详细内容与编码**
    这是理解 HPatchLite 格式的关键。`for` 循环内部详细地展示了每个 `cover` 是如何被编码的：

    1.  **`hpi_packUInt(buf, coverCount)`:** 首先写入 `cover` 的总数。
    2.  **`hpi_packUInt(buf, cover.length)`:** 写入当前 `cover` 的长度。
    3.  **`_getSubDiff(subDiff, ...)`:** 计算差分数据 (`newData - oldData`)。
    4.  **`isNullSubDiff` 优化:** 检查差分数据是否**全为0**。这是一个非常重要的优化。如果全为0，意味着新旧数据在该 `cover` 区域完全相同。
    5.  **`hpi_packUIntWithTag(buf, ...)` (编码 `oldPos`):**
        *   `oldPos` 使用**差量编码**（相对于 `lastOldEnd`）。
        *   `WithTag`: 可变长度整数编码的同时，还打包了**两个标志位**：
            *   **bit0:** `oldPos` 是向前移动 (`+`) 还是向后移动 (`-`)。
            *   **bit1:** `isNullSubDiff` 标志，告诉打补丁程序，这个 `cover` 是否需要进行加法运算。
    6.  **`hpi_packUInt(buf, backNewLen)` (编码 `newPos`):**
        *   `newPos` 也使用差量编码（相对于 `lastNewEnd`）。
    7.  **写入“沟”中数据:**
        *   `if (backNewLen > 0) ...`: 如果当前 `cover` 与上一个 `cover` 之间有间隙（`backNewLen` > 0），这部分就是**纯粹的新增数据**，直接将其原文写入 `buf`。
    8.  **写入差分数据:**
        *   `if (!isNullSubDiff) ...`: **只有在差分不全为0时**，才将差分数据 `subDiff` 写入 `buf`。

*   **文件头 (`out_diff` 的前半部分) 的详细内容**
    *   **魔法标识:** `'hI'`
    *   **压缩类型:** `hpi_compressType_no` 或插件类型。
    *   **打包字节:** 一个字节，通过位运算打包了三个信息：
        *   **版本号:** `kHPatchLite_versionCode` (1) 或 `kHPatchLite_inplaceCode` (2)。
        *   **`newSize` 的字节数:** `newSize` 这个数值本身占多少字节。
        *   **`uncompressSize` 的字节数:** 数据体解压后的大小占多少字节。
    *   **(In-place 模式) `extraSafeSize` 的字节数。**
    *   **可变长度的元数据:** 依次写入 `newSize`, `savedUncompressSize`, 和 (可选的) `extraSafeSize` 的实际数值。

### **代码解析**

```cpp
static void serialize_lite_diff(const TDiffData& diff,const std::vector<TOLDCOVER>& covers,
                                std::vector<TByte>& out_diff,const hdiffi_TCompress* compressPlugin,
                                const TInlpacePatchSets& inlpacePatchSets){
    const TUInt coverCount=(TUInt)covers.size();
    const bool isInplacePatch= inlpacePatchSets.isInplacePatchByExtra;
    std::vector<TByte> subDiff;
    std::vector<TByte> buf; // 临时缓冲区，用于构建未压缩的数据体
    
    // --- 1. 构建数据体 ---
    hpi_packUInt(buf,coverCount); // 写入 cover 总数
    const TUInt newSize=(TUInt)(diff.newData_end-diff.newData);
    {
        TUInt lastOldEnd=0;
        TUInt lastNewEnd=0;
        for (TUInt i=0; i<coverCount; ++i) {
            const TOldCover& cover=covers[i];
            hpi_packUInt(buf, cover.length); // 写入 cover 长度

            _getSubDiff(subDiff,diff,cover); // 计算差分数据
            const TByte isNullSubDiff=_getIs0(subDiff.data(),cover.length)?1:0; // 检查差分是否全为0

            // 编码 oldPos 的差量，并打包标志位
            if ((TUInt)cover.oldPos>=lastOldEnd){
                hpi_packUIntWithTag(buf,(TUInt)(cover.oldPos-lastOldEnd), 0+isNullSubDiff*2,2); // 向前，bit1=isNullSubDiff
            }else{
                hpi_packUIntWithTag(buf,(TUInt)(lastOldEnd-cover.oldPos), 1+isNullSubDiff*2,2); // 向后，bit1=isNullSubDiff
            }

            // 编码 newPos 的差量
            TUInt backNewLen=cover.newPos-lastNewEnd;
            assert(backNewLen>=0);
            hpi_packUInt(buf,(TUInt)backNewLen); 
            
            // 写入两个cover之间的“新数据”
            if (backNewLen>0){
                const TByte* newDataDiff=diff.newData+lastNewEnd;
                pushBack(buf,newDataDiff,newDataDiff+backNewLen);
            }
            // 如果需要，写入差分数据
            if (!isNullSubDiff){
                pushBack(buf,subDiff.data(),subDiff.data()+cover.length);
            }
            
            // 更新状态
            lastOldEnd=cover.oldPos+cover.length;
            lastNewEnd=cover.newPos+cover.length;
        }
        
        TUInt backNewLen=(newSize-lastNewEnd);
        check(backNewLen==0); // 确认已处理到新文件末尾
    }
    
    // --- 2. 压缩数据体 ---
    std::vector<TByte> compress_buf;
    do_compress(compress_buf,buf,compressPlugin->compress);

    // --- 3. 构建文件头并组装 ---
    out_diff.push_back(kHPatchLite_versionType[0]); // 'h'
    out_diff.push_back(kHPatchLite_versionType[1]); // 'I'
    out_diff.push_back(compress_buf.empty()?hpi_compressType_no:compressPlugin->compress_type); // 压缩类型
    
    TUInt savedUncompressSize=compress_buf.empty()?0:buf.size();
    const TByte savedVersionCode=isInplacePatch?kHPatchLite_inplaceCode:kHPatchLite_versionCode;
    // 写入打包字节
    out_diff.push_back((savedVersionCode<<6)|
                       (hpi_getSavedSizeBytes(newSize))|
                       (hpi_getSavedSizeBytes(savedUncompressSize)<<3));
    // 如果是 in-place 模式，写入 extraSafeSize 的长度
    if (isInplacePatch)
        out_diff.push_back(hpi_getSavedSizeBytes(inlpacePatchSets.extraSafeSize));
    
    // 依次写入 newSize, uncompressSize, extraSafeSize 的实际值
    hpi_saveSize(out_diff,newSize);
    hpi_saveSize(out_diff,savedUncompressSize);
    if (isInplacePatch)
        hpi_saveSize(out_diff,inlpacePatchSets.extraSafeSize);
    
    // 写入最终的数据体（压缩过的或原始的）
    pushBack(out_diff,compress_buf.empty()?buf:compress_buf);
}
```

## search_and_dispose_cover **两阶段匹配：原始搜索与最优选择**

这是 `first_search_and_dispose_cover` 内部调用的、单线程的**核心工作单元**。这个函数的设计完美体现了“关注点分离”的原则，将复杂的匹配过程分解为两个清晰、独立的阶段：**1. 原始搜索** 和 **2. 优化处理**。

### **原理与设计思路解析**

这个函数是整个差异发现算法的“引擎室”。它接收一小块新文件数据（由 `diffLimit` 定义范围），并为其在整个旧文件中寻找最佳的匹配指令（`covers`）。

*   **核心策略：两阶段处理 (Two-Phase Processing)**
    这种设计将复杂的任务分解，使得每一阶段的职责都非常单一，易于理解和优化。

    1.  **第一阶段: `_search_cover` (原始匹配生成器 - The Prospector)**
        *   **职责:** 这是算法的“勘探”阶段。它的任务是利用高效的后缀字符串 (`sstring`) 结构，**不加选择地**找出所有可能的、有价值的匹配块。
        *   **产出:** 这一步的产出是一系列“原始”或“候选”的 `covers`。这些 `covers` 可能存在以下问题：
            *   **重叠 (Overlapping):** 多个候选 `cover` 可能覆盖了新文件中的同一区域。
            *   **次优 (Sub-optimal):** 某些短的匹配可能被一个更长的匹配完全包含。
            *   **低质量 (Low-quality):** 某些匹配可能非常短，以至于记录它们的成本（在补丁中的指令大小）高于直接存储新数据的成本。
        *   它将找到的所有候选 `covers` 直接追加到 `covers` 向量的末尾。

    2.  **第二阶段: `_dispose_cover` (最优匹配选择器 - The Jeweler)**
        *   **职责:** 这是算法的“精炼”和“决策”阶段。它接收第一阶段产出的所有原始、混乱的候选 `covers`，并从中挑选出一组**最优的、不重叠的最终 `covers`**。
        *   **算法:** 这一步通常采用**动态规划 (Dynamic Programming)**。它解决的问题可以类比为“加权区间调度问题”：给定一组带有“分数”（`score`，通常与匹配长度和数据稀有度相关）的重叠区间（`covers`），找到一个不重叠的子集，使其总分数最高。
        *   **决策依据 (`kMinSingleMatchScore`):** 在决策过程中，它会使用 `kMinSingleMatchScore` 这个阈值来过滤掉所有“分数”过低的、无价值的匹配，确保最终选出的 `covers` 都是有意义的。
        *   它将处理过的、最优的 `covers` 结果**替换掉** `covers` 向量中原有的那部分原始匹配。

*   **性能优化**
    *   `const size_t cover_begin = covers.size();` 这一行代码非常关键。它在执行搜索前“标记”了当前 `covers` 向量的大小。
    *   `_dispose_cover` 只处理从 `cover_begin` 开始的、由 `_search_cover` **新添加**的这部分 `covers`，而不会影响之前已经处理好的部分。
    *   `if (covers.size() > cover_begin)` 这个检查是一个重要的短路优化。如果 `_search_cover` 没有找到任何候选匹配，那么就完全没有必要执行昂贵的动态规划处理阶段，函数可以直接返回。

### **代码解析**

```cpp
/**
 * @brief 对指定范围的数据块执行“搜索”和“处理”两个阶段的匹配发现。
 * @param covers               输出参数，用于存储最终找到的最优covers。
 * @param diff                 包含新旧文件数据的上下文。
 * @param sstring              预处理过的旧文件后缀字符串，用于快速搜索。
 * @param kMinSingleMatchScore 单个匹配的最低有效分数。
 * @param diffLimit            (可选) 定义了本次搜索只在新文件的哪个范围内进行。
 * @param isCanExtendCover     一个标志，指示是否允许扩展匹配的边界。
 */
static void search_and_dispose_cover(std::vector<TOldCover>& covers,const TDiffData& diff,
                                     const TSuffixString& sstring,int kMinSingleMatchScore,
                                     TDiffLimit* diffLimit,bool isCanExtendCover){
    // 在搜索前，记录下当前 covers 列表的大小。这是一个“书签”，用于标记新添加内容的起点。
    const size_t cover_begin=covers.size();

    // --- 阶段一：原始搜索 ---
    // 调用底层搜索函数，它会利用后缀字符串找到所有可能的匹配项，
    // 并将它们（可能是重叠和未经优化的）追加到 `covers` 向量的末尾。
    _search_cover(covers,diff,sstring,diffLimit,isCanExtendCover);

    // --- 阶段二：优化处理 ---
    // 检查 `_search_cover` 是否找到了任何新的候选匹配。
    if (covers.size()>cover_begin){
        // 如果找到了，则调用处理函数。
        // `_dispose_cover` 会对从 `cover_begin` 开始的这部分原始匹配进行动态规划，
        // 挑选出最优的、不重叠的组合，并用结果替换掉原始匹配。
        _dispose_cover(covers,cover_begin,diff,kMinSingleMatchScore,diffLimit,isCanExtendCover);
    }
}
```

## first_search_and_dispose_cover_MT **多线程并行差异发现**

这是 `get_diff` 内部调用的核心工作函数，专门负责执行**初次**的、计算量最大的匹配搜索。函数名中的 `_MT` (Multi-Threaded) 表明了它的关键特性：**使用多线程来并行处理，以加速差异发现的过程**。

### **原理与设计思路解析**

这个函数是一个典型的**并行任务分发器 (Parallel Task Dispatcher)**。它本身不执行匹配算法，而是负责将庞大的匹配任务**分解**成多个小任务，并将这些小任务分发给多个线程（包括主线程）去执行，最后再将所有线程的结果**汇总**起来。

*   **核心策略：并行工作划分 (Parallel Work Division)**
    *   **并行化的对象：** 算法选择对**新文件 (`newData`)** 进行数据划分。这是因为搜索过程是在新文件上线性进行的，将其切分成多个独立的块是自然且高效的并行化方式。
    *   **任务粒度控制：** 代码并没有简单地将 `newSize / threadNum` 作为任务大小。它实现了一套更智能的启发式策略，以平衡并行开销和效率：
        1.  **并行化阈值 (`kMinParallelSize`)**: 如果新文件太小（小于2MB），则根本不启用多线程，因为创建和管理线程的开销可能会超过并行带来的收益。直接回退到单线程模式。
        2.  **线程数限制 (`maxThreanNum`)**: 即使请求了大量线程，代码也会限制实际创建的线程数，确保每个线程至少能分到一块有意义的数据（1MB），避免了在小文件上创建过多低效的线程。
        3.  **最佳块大小 (`kBestParallelSize`)**: 算法倾向于将新文件切分成大小约为8MB的“工作块”。这种较大的任务粒度可以最大化每个线程的有效计算时间，减少线程间同步的频率。
    *   **动态工作队列：** 通过共享的 `mt_data` 结构体和其中的原子计数器 `workIndex`（虽然未明示，但其用法暗示了原子性），实现了一个动态的工作队列。每个线程完成自己的一个块后，会再去队列中领取下一个可用的块，直到所有块都被处理完毕。

*   **执行模型：主线程参与计算**
    *   该函数采用了高效的 `N-1` 线程创建模型。如果用户请求 `N` 个线程，它只会创建 `N-1` 个新的工作线程。
    *   **主线程**不会闲置等待，而是立即调用工作函数 `_fsearch_and_dispose_cover_thread`，与新创建的线程一起**参与计算**。这确保了所有CPU核心都能被充分利用。

*   **结果聚合：无锁与后处理**
    *   **无锁并行 (`lock-free`)**: 为了最大化性能，每个工作线程都将自己找到的 `covers` 存放在**各自私有的** `threadCovers` 向量中。这避免了在多线程环境下对一个共享的 `covers` 列表进行代价高昂的加锁操作。
    *   **结果合并 (`insert`)**: 当所有工作线程结束后，主线程会依次 `join` 它们，然后将每个线程私有的结果向量 `threadCovers[i]` 中的内容一次性地 `insert` 到主 `covers` 向量的末尾。
    *   **排序与整理 (`tm_collate_covers`)**: 合并后的主 `covers` 向量是无序的（来自不同块的结果混杂在一起），并且在块的边界处可能存在重叠或次优的匹配。`tm_collate_covers` 函数在最后被调用，负责对所有 `covers` 进行排序、去重、以及解决边界冲突，生成一个最终的、干净、有序的匹配列表。

### **代码解析**

```cpp
static void first_search_and_dispose_cover_MT(std::vector<TOldCover>& covers,const TDiffData& diff,
                                              const TSuffixString& sstring,int kMinSingleMatchScore,
                                              ICoverLinesListener* listener,size_t threadNum,bool isCanExtendCover){
// --- 编译时开关：如果不支持多线程，则整个并行逻辑不存在 ---
#if (_IS_USED_MULTITHREAD)
    // 定义并行化的启发式参数
    const size_t kMinParallelSize=1024*1024*2;  // 启用多线程的最小文件大小 (2MB)
    const size_t kBestParallelSize=1024*1024*8; // 倾向的工作块大小 (8MB)
    size_t newSize=diff.newData_end-diff.newData;

    // --- 运行时的并行决策 ---
    // 条件：请求了多线程 & 存在旧数据 & 新文件足够大
    if ((threadNum>1)&&(diff.oldData!=diff.oldData_end)&&(newSize>=kMinParallelSize)){
        // --- 任务划分与设置 ---
        // 限制最大线程数，保证每个线程至少有1MB的工作量
        const size_t maxThreanNum=newSize/(kMinParallelSize/2);
        threadNum=(threadNum<=maxThreanNum)?threadNum:maxThreanNum;
        // 根据最佳块大小计算出需要多少个工作块
        size_t workCount=(newSize+kBestParallelSize-1)/kBestParallelSize;
        // 保证工作块数量不少于线程数，让每个线程都有事可做
        workCount=(threadNum>workCount)?threadNum:workCount;

        const size_t threadCount=threadNum-1; // 实际创建N-1个新线程
        std::vector<std::thread> threads(threadCount);
        // 为每个新线程分配一个私有的结果存储区，以避免加锁
        std::vector<std::vector<TOldCover> > threadCovers(threadCount);
        
        // 设置一个共享的上下文，传递给所有线程
        mt_data_t mt_data;
        mt_data.diff=&diff;
        mt_data.sstring=&sstring;
        mt_data.listener=(listener&&listener->next_search_block_MT)?listener:0;
        mt_data.kMinSingleMatchScore=kMinSingleMatchScore;
        // 计算出每个工作块的实际大小
        mt_data.workBlockSize=(newSize+workCount-1)/workCount;
        mt_data.workIndex=0; // 工作队列的索引，应为原子类型
        mt_data.isCanExtendCover=isCanExtendCover;

        if (mt_data.listener&&listener->begin_search_block)
            listener->begin_search_block(listener,newSize,mt_data.workBlockSize,kPartPepeatSize);

        // --- 线程创建与执行 ---
        for (size_t i=0;i<threadCount;i++)
            // 创建N-1个工作线程
            threads[i]=std::thread(_fsearch_and_dispose_cover_thread,&threadCovers[i],&mt_data);
        
        // 主线程也参与工作，结果直接写入最终的covers列表
        _fsearch_and_dispose_cover_thread(&covers,&mt_data);
        
        // --- 结果聚合 ---
        for (size_t i=0;i<threadCount;i++){
            threads[i].join(); // 等待工作线程结束
            // 将工作线程的结果合并到主列表中
            covers.insert(covers.end(),threadCovers[i].begin(),threadCovers[i].end());
            // 清理工作线程的内存（swap trick）
            { std::vector<TOldCover> tmp; tmp.swap(threadCovers[i]); }
        }
        // 对所有合并后的、无序的covers进行排序和整理
        tm_collate_covers(covers);
    }else // 如果不满足并行条件...
#endif
    {
        // ...则直接调用单线程版本的实现
        first_search_and_dispose_cover(covers,diff,sstring,kMinSingleMatchScore,isCanExtendCover);
    }
}
```

## getBestMatch **后缀数组邻域搜索**

这是 `_search_cover` 调用的底层核心工作函数。它的职责非常专一：对于**新文件中的一个特定位置**，在整个旧文件（由 `TSuffixString` 代表）中，找到**一个最长的匹配串**，并返回其在旧文件中的位置和匹配长度。

### **原理与设计思路解析**

这个函数是 HDiffPatch 性能的关键。它并没有天真地扫描整个旧文件，而是使用了一套基于后缀数组的高度优化的**启发式搜索策略**。

*   **核心策略：局部邻域搜索 (Local Neighborhood Search)**
    1.  **初始命中 (`lower_bound`):** 算法的第一步不是线性扫描，而是利用后缀数组的有序性，通过 `sstring.lower_bound()` 进行**二分查找**。这能以对数时间复杂度 (`O(logN)`) 快速定位到新文件当前位置的字符串在旧文件所有后缀的排序列表中的**第一个匹配位置** (`sai`)。
    2.  **螺旋式探索 (Spiral Search):** 找到初始命中点 `sai` 后，算法并不会停止。它基于一个关键的洞察：**在排序的后缀数组中，字符串内容相似的后缀会聚集在一起**。因此，与 `sai` 位置的后缀最相似、最有可能产生更长匹配的，就是它在数组中的**紧邻邻居**。
        *   代码中的 `for` 循环和 `i = sai + ...` 的计算，实现了一个“螺旋式”或“向外扩展式”的搜索。它会依次检查 `sai`、`sai-1`、`sai+1`、`sai-2`、`sai+2`... 这些 `sai` 周边的位置。
        *   `matchDeep` 参数控制了这个邻域搜索的**半径**或“深度”，即算法愿意从初始命中点向外探索多远。这是一个重要的调优参数，用于在搜索的彻底性和速度之间做权衡。

*   **受限搜索 (`diffLimit`)**
    这个函数可以通过 `diffLimit` 参数进入一种“受限模式”，这通常在 `research_cover` 阶段被使用，目的是寻找满足特定约束的**替代匹配**。
    *   **范围限制:** `kLimitOldPos` 和 `kLimitOldEnd` 可以限制只在旧文件的某个特定范围内寻找匹配。
    *   **回调干预 (`listener->limitCover`):** 这是最强大的功能。在找到一个潜在的匹配后，代码会通过回调函数将这个 `cover` 交给 `listener` 进行审查。`listener` 可以根据自己复杂的规则（例如，是否与现有 `covers` 冲突，是否满足 `in-place` 模式的安全距离等）来**否决**这个匹配或**缩短**其有效长度。这给予了上层逻辑对底层匹配结果的最终控制权。

*   **性能优化**
    *   **有限长度比较 (`getEqualLengthLimit`)**: 在受限模式下，代码首先调用一个只比较有限长度（`kTryEqLenLimit`）的函数。这是一个**短路优化**：如果一个匹配连一个较短的长度都无法满足，就没必要进行完整、可能很慢的比较。只有当它通过了初步检查，或者长度可能超过限制时，才会调用完整的 `getEqualLength`。
    *   **提前退出 (`leftOk`, `rightOk`):** 螺旋搜索一旦在 `sai` 的左侧和右侧都找到了有效的匹配，就会提前 `break` 循环。这避免了在已经找到不错的候选者后，还继续向更远、更不可能的位置进行无效的探索。

### **代码解析**

```cpp
// 在旧文件中，为新文件当前位置的数据寻找最佳匹配的长度和位置。
static TInt getBestMatch(TInt* out_pos,const TSuffixString& sstring,
                         const TByte* newData,const TByte* newData_end,
                         TInt curNewPos,TDiffLimit* diffLimit=0,size_t* out_limitSkip=0){
    // 1. 初始命中：使用二分查找，快速定位到第一个匹配项在后缀数组中的索引(sai)。
    TInt sai=sstring.lower_bound(newData,newData_end);
    if (sai<0) return 0; // 未找到任何匹配。

    // 2. 设置搜索参数：
    // matchDeep 控制邻域搜索的深度（半径）。
    const TInt matchDeep = diffLimit?diffLimit->kMaxMatchDeep:2;
    // kLimitOld... 定义了在旧文件中的有效搜索范围（仅在受限模式下使用）。
    const TInt kLimitOldPos=(TInt)(diffLimit?diffLimit->recoverOldPos:0);
    const TInt kLimitOldEnd=(TInt)(diffLimit?diffLimit->recoverOldEnd:sstring.SASize());
    
    const TByte* src_begin=sstring.src_begin();
    const TByte* src_end=sstring.src_end();
    TInt bestLength= kMinMatchLen -1; // 初始化最佳长度为一个无效值。
    TInt bestOldPos=-1;
    bool leftOk = false; // 标记是否已在sai左侧找到匹配。
    bool rightOk = false; // 标记是否已在sai右侧找到匹配。

    // 3. 螺旋式邻域搜索：
    for (TInt mdi= 0; mdi< matchDeep; ++mdi) {
        // 提前退出优化
        if (mdi&1){ if (rightOk) continue; } 
        else { if (leftOk) continue; }

        // 计算要检查的邻居索引 i: mdi=0->sai, mdi=1->sai-1, mdi=2->sai+1, ...
        TInt i = sai + (1-(mdi&1)*2) * ((mdi+1)/2);
        if ((i<0)|(i>=(src_end-src_begin))) continue; // 边界检查
        
        // 从后缀数组获取该邻居在旧文件中的实际起始位置。
        TInt curOldPos=sstring.SA(i);
        TInt curLength;

        // 4. 计算并评估匹配长度
        #define kTryEqLenLimit (1024*1+17) // 短路优化的比较长度限制
        if (0==diffLimit){ // 标准模式
            curLength=getEqualLength(newData,newData_end,src_begin+curOldPos,src_end);
        }else{ // 受限模式
            // 先进行有限长度的快速比较。
            curLength=getEqualLengthLimit<kTryEqLenLimit>(newData,newData_end,src_begin+curOldPos,src_end);
            if ((kLimitOldPos<=curOldPos)&&(curOldPos<kLimitOldEnd)){
                // 如果这个匹配落在了一个“禁区”内，则跳过它。
                if (curLength==kTryEqLenLimit)
                    *out_limitSkip=kTryEqLenLimit/2; // 提示上层可以安全跳过的距离
                continue;
            }
        }
         
        // 如果当前匹配比已知的最佳匹配要长...
        if (curLength>bestLength){
            if (diffLimit){ // 在受限模式下，需要通过回调让上层listener审查。
                hpatch_TCover cover={(size_t)curOldPos,(size_t)curNewPos,(size_t)curLength};
                hpatch_StreamPos_t hitPos;
                diffLimit->listener->limitCover(diffLimit->listener,&cover,&hitPos);
                if (hitPos<=(size_t)bestLength) // 如果listener否决或缩短了它...
                    continue; // ...则放弃。
                
                // 如果初步匹配长度达到了短路限制，可能实际匹配更长。
                if (hitPos==kTryEqLenLimit){
                    // 进行一次完整的长度比较。
                    curLength=getEqualLength(newData,newData_end,src_begin+curOldPos,src_end);
                    hpatch_TCover cover={(size_t)curOldPos,(size_t)curNewPos,(size_t)curLength};
                    // 再次让listener审查完整长度的匹配。
                    diffLimit->listener->limitCover(diffLimit->listener,&cover,&hitPos);
                }
                curLength=(TInt)hitPos; // 使用listener最终确认的长度。
            }
            // 更新最佳匹配
            bestLength = curLength;
            bestOldPos= curOldPos;
        }

        // 提前退出优化
        if (mdi&1){ rightOk=true; if (leftOk) break; } 
        else { leftOk=true; if (rightOk) break; }
    }

    // 5. 返回结果
    *out_pos=bestOldPos;
    return  bestLength;
}
```

## tryLinkExtend **启发式 Cover 合并**

`tryLinkExtend` 是 `_search_cover` 贪心算法中一个非常激进且强大的优化。当算法找到一个新的匹配 `matchCover` 时，它不会立即接受，而是会调用 `tryLinkExtend` 尝试将这个新的匹配与上一个已接受的 `lastCover` **合并 (merge)** 成一个单一的、更长的 `cover`。

### **原理与设计思路解析**

这个函数的目的是**减少 `cover` 的数量**，从而降低最终补丁文件中控制指令的总大小。它试图将两个在逻辑上相近的 `cover` “缝合”起来。

*   **核心策略：带“桥”的扩展**
    *   **适用场景:** 当两个 `cover` (`lastCover` 和 `matchCover`) 在新文件中靠得很近（它们之间的“间隙”`linkSpaceLength` 小于 `kMaxLinkSpaceLength`）时，这个函数被触发。
    *   **共线情况 (Collinear Case):**
        *   最简单的情况是两个 `cover` 刚好是“共线”的（`isCollinear`），即它们在旧文件中的相对位置和新文件中完全一样。
        *   此时，它们之间的数据（包括 `linkSpaceLength` 间隙和 `matchCover` 本身）可以被看作是 `lastCover` 的一个自然延伸。
        *   `lastCover.Link(matchCover);` 这句代码会直接将 `lastCover` 的长度扩展到包含 `matchCover` 的末尾，完成一次完美的合并。
    *   **非共线情况 (Non-Collinear Case - 核心逻辑):**
        *   这是更有趣的情况。两个 `cover` 不共线，意味着 `matchCover.oldPos` 与 `lastCover` 的期望延伸位置不符。
        *   此时，算法**不**接受 `matchCover` 的 `oldPos`，而是**假设** `lastCover` 可以被“强行”扩展，覆盖掉 `matchCover` 所在的区域。
        *   **成本效益分析:**
            *   `matchCost`: 算法首先计算如果**不**合并，单独存储 `matchCover` 需要的控制成本。
            *   `lastLinkCost`: 然后，它计算如果**强行**将 `lastCover` 延伸到 `matchCover` 的区域（即从 `linkOldPos` 开始），这部分延伸区域的**差分数据成本** (`getRegionRleCost`) 是多少。
            *   `if (lastLinkCost > matchCost) return false;`: 只有当“强行延伸”的差分成本**低于**“新建一个cover”的控制成本时，这个合并尝试才有意义。
        *   **试探性扩展:**
            *   `TInt len=...+(matchCover.length*2/3);`: 如果成本分析通过，算法不会立即将 `lastCover` 延伸到 `matchCover` 的末尾，而是进行一次**试探性**的、保守的扩展（只扩展 `matchCover` 长度的2/3）。
            *   `len+=getEqualLength(...);`: 在此基础上，再从该点开始，尽可能地向后寻找精确匹配，进一步延伸长度。
        *   **边界修正:** `while (...) --len;` 最后，从后向前修正 `len`，确保扩展后的 `lastCover` 的最后一个字节是精确匹配的。

*   **受限搜索 (`diffLimit`) 的角色**
    *   在受限搜索模式下，任何对 `cover` 的修改（包括合并）都必须得到 `listener` 的“批准”。
    *   `if (!diffLimit->listener->limitCover(...)) return false;`
    *   在尝试合并或扩展之前，函数会构造一个代表“合并区域”的 `cover`，并将其交给 `listener`。如果 `listener` 返回 `false`（例如，因为这个合并会违反 `in-place` 模式的安全距离），则 `tryLinkExtend` 会立即放弃合并，返回 `false`。

**总结一下 `tryLinkExtend` 的决策流程：**

1.  两个 `cover` 离得太远吗？是 -> 放弃。
2.  两个 `cover` 是共线的吗？是 -> 直接合并，成功。
3.  (非共线) 合并的差分成本是否低于新建一个 `cover` 的控制成本？否 -> 放弃。
4.  (成本划算) 试探性地扩展 `lastCover` 并寻找最大精确匹配长度。
5.  最终更新 `lastCover.length`，成功。

### **代码解析**

```cpp
// 尝试将 lastCover 扩展以完全替代 matchCover
static bool tryLinkExtend(TOldCover& lastCover,const TOldCover& matchCover,const TDiffData& diff,TDiffLimit* diffLimit){
    if (lastCover.length<=0) return false; // lastCover 必须有效

    // 1. 检查两个 cover 在新文件中的间隙是否过大
    const TInt linkSpaceLength=(matchCover.newPos-(lastCover.newPos+lastCover.length));
    assert(linkSpaceLength>=0);
    if (linkSpaceLength>kMaxLinkSpaceLength)
        return false;

    // 计算如果 lastCover 是共线的，它的延伸部分应该在旧文件的什么位置
    TInt linkOldPos=lastCover.oldPos+lastCover.length+linkSpaceLength;
    if (linkOldPos+matchCover.length>(diff.oldData_end-diff.oldData))
        return false; // 延伸会超出旧文件边界，放弃

    const bool isCollinear=lastCover.isCollinear(matchCover);

    // 在受限模式下，需要先征求 listener 的同意
    if (diffLimit){
        size_t cnewPos=lastCover.newPos+lastCover.length;
        // 构造一个代表“合并区域”的cover
        hpatch_TCover cover={(size_t)(lastCover.oldPos+lastCover.length),cnewPos,
                             (size_t)(linkSpaceLength+(isCollinear?0:matchCover.length))};
        if (!diffLimit->listener->limitCover(diffLimit->listener,&cover,0))
            return false; // listener 否决了合并
    }

    // 2. 如果是共线情况，直接合并并返回
    if (isCollinear){
        lastCover.Link(matchCover);
        return true;
    }

    // --- 3. 非共线情况的处理 ---
    // 成本效益分析
    TInt matchCost=getCoverCtrlCost(matchCover,lastCover); // 新建一个cover的控制成本
    // “强行”延伸的差分成本
    TInt lastLinkCost=(TInt)getRegionRleCost(diff.newData+matchCover.newPos,matchCover.length,diff.oldData+linkOldPos);
    if (lastLinkCost>matchCost)
        return false; // 如果差分成本更高，则不划算，放弃

    // 4. 试探性扩展
    // 初步扩展长度 = lastCover + 间隙 + matchCover的2/3
    TInt len=lastCover.length+linkSpaceLength+(matchCover.length*2/3);
    // 在此基础上，尽可能向后寻找精确匹配
    len+=getEqualLength(diff.newData+lastCover.newPos+len,diff.newData_end,
                        diff.oldData+lastCover.oldPos+len,diff.oldData_end);
    
    // 在受限模式下，再次检查扩展是否越界或被 listener 否决
    if (diffLimit){
        TInt limitLen=diffLimit->newEnd-lastCover.newPos;
        len=len<limitLen?len:limitLen; // 不能超出搜索范围
        TInt safeLen=lastCover.length+linkSpaceLength+matchCover.length;
        if (len>safeLen){ // 如果扩展超过了 matchCover 的原始边界
            hpatch_TCover cover={...};
            hpatch_StreamPos_t hitPos;
            diffLimit->listener->limitCover(diffLimit->listener,&cover,&hitPos);
            len=(TInt)(safeLen+hitPos); // 使用 listener 允许的最终长度
        }
    }

    // 5. 边界修正：从后向前回溯，确保扩展的最后一个字节是匹配的
    while ((len>0) && (diff.newData[lastCover.newPos+len-1]
                       !=diff.oldData[lastCover.oldPos+len-1])) {
        --len;
    }

    // 最终更新 lastCover 的长度，完成合并
    lastCover.length=len;
    return true;
}
```

## _search_cover **贪心算法驱动的原始匹配搜索**

这是 `search_and_dispose_cover` 的第一阶段，也是 HDiffPatch 差异发现算法的“勘探”阶段。它的核心任务是利用后缀字符串，在给定的数据范围内，**贪心地**搜索出所有有价值的原始匹配块（`covers`）。

### **原理与设计思路解析**

这个函数是整个差分引擎的“主力军”，它通过一个循环迭代新文件，并在每个位置尝试寻找最佳匹配。其核心是一个**贪心算法 (Greedy Algorithm)**，并辅以多种复杂的启发式优化。

*   **核心算法：贪心选择**
    *   函数从左到右线性扫描新文件 (`while (newPos <= maxSearchNewPos)`).
    *   在当前的 `newPos` 位置，它调用 `getBestMatch` 来寻找一个在旧文件中能找到的、**最长的**匹配串。
    *   一旦找到一个匹配，它并**不**立即接受，而是进行一次**成本效益分析**。

*   **启发式决策：净收益评分 (Net Score Heuristic)**
    *   这是算法最智能的部分。一个匹配的价值并不仅仅是它的长度。Hpatch 还需要为存储这个匹配的指令（cover的元数据）付出成本。
    *   代码中的 `matchEqLength - getCoverCtrlCost(matchCover, lastCover) < kMinMatchScore` 正是这个决策的核心。
        *   `matchEqLength`: 代表了匹配带来的**收益**（即省去了多少需要直接存储的新数据）。
        *   `getCoverCtrlCost(...)`: 代表了记录这个匹配指令的**成本**。这个成本是动态的，它会考虑当前匹配与上一个匹配 (`lastCover`) 的关系。例如，如果两个匹配在旧文件中的位置相近，编码成本可能就更低。
        *   `kMinMatchScore`: 是一个阈值，只有当**净收益**（收益 - 成本）大于这个阈值时，这个匹配才被认为是“划算的”，才会被接受。

*   **高级优化 (`isCanExtendCover`)**
    当找到一个划算的匹配后，算法并不会简单地将其加入列表，而是会尝试两种高级优化来生成更优的 `cover`：

    1.  **`tryLinkExtend` (Cover 链接/合并):**
        *   这是最激进的优化。它会检查新找到的 `matchCover` 是否能与上一个 `lastCover` **合并**成一个更长的、连续的 `cover`。
        *   这种情况通常发生在两个匹配块在旧文件和新文件中都靠得很近，中间只有少量不匹配的字节。`tryLinkExtend` 会尝试将这两个匹配以及中间的差异“缝合”成一个单一的、更长的 `cover`。这能显著减少需要存储的 `cover` 指令数量，从而减小补丁体积。

    2.  **`tryCollinear` (Cover 共线性调整):**
        *   如果无法合并，算法会尝试这个次级优化。它检查新旧两个 `cover` 是否“共线”（即它们在新旧文件中的相对顺序和间隔都比较规律）。
        *   如果是，它可能会微调 `matchCover` 的边界，使其与前一个 `cover` 的关系更“规整”，以便后续的编码和压缩能更高效。

*   **贪心策略的体现：跳跃式前进**
    *   在接受一个 `cover` 后，扫描指针 `newPos` 会被立即**跳跃**到这个 `cover` 的末尾 (`lastCover.newPos + lastCover.length`)。
    *   这意味着算法**不会**再回头去寻找可能与当前 `cover` 重叠的其他匹配方案。它总是做出“局部最优”的选择，然后继续前进。
    *   函数末尾的注释 `//The selected cover does not allow overlap, which may not be the optimal strategy;` 非常精辟地指出了这一点：这种贪心策略速度极快，但在极少数情况下可能无法找到全局最优解。然而在实践中，它通常能产生非常接近最优解的高质量结果。

### **代码解析**

```cpp
// 在新旧文件之间，寻找合适的匹配块(cover)。
static void _search_cover(std::vector<TOldCover>& covers,const TDiffData& diff,
                          const TSuffixString& sstring,TDiffLimit* diffLimit,bool isCanExtendCover){
    // 如果旧文件为空（后缀字符串为空），则不可能有匹配。
    if (sstring.SASize()<=0) return;
    
    // 确定本次搜索在新文件中的范围 [newPos, newEnd)。
    TInt newPos=diffLimit?diffLimit->newPos:0;
    const TInt newEnd=diffLimit?diffLimit->newEnd:(diff.newData_end-diff.newData);
    // 如果搜索范围太小，直接返回。
    if (newEnd-newPos<=kMinMatchLen) return;
    const TInt maxSearchNewPos=newEnd-kMinMatchLen; // 预计算循环的最大边界。
    const size_t cover_begin=covers.size();

    TOldCover lastCover(0,0,0); // 用于记录上一个被接受的cover。
    
    // --- 贪心搜索主循环 ---
    while (newPos<=maxSearchNewPos) {
        TInt matchOldPos=0;
        size_t limitSkip=1; // 用于getBestMatch返回可以安全跳过的字节数。
        // 调用底层函数，利用后缀字符串寻找当前 newPos 处的最长匹配。
        TInt matchEqLength=getBestMatch(&matchOldPos,sstring,diff.newData+newPos,diff.newData+newEnd,newPos,
                                        diffLimit,&limitSkip);
        
        // 如果找到的匹配太短，则跳过。
        if (matchEqLength<kMinMatchLen){
            newPos+=limitSkip; // 跳到安全位置继续搜索。
            continue;
        }

        TOldCover matchCover(matchOldPos,newPos,matchEqLength);
        
        // --- 启发式决策：成本效益分析 ---
        // 如果匹配的“净收益”（长度 - 编码成本）小于阈值，则放弃该匹配。
        if (matchEqLength-getCoverCtrlCost(matchCover,lastCover)<kMinMatchScore){
            ++newPos; // 只前进一个字节，继续尝试。
            continue;
        }//else matched: 这是一个有价值的匹配。
        
        // --- 高级优化：合并与调整 ---
        if (isCanExtendCover){ // 如果启用了高级优化...
            // 尝试将当前匹配与上一个匹配合并。
            if (tryLinkExtend(lastCover,matchCover,diff,diffLimit)){//use link
                // 如果合并成功，lastCover会被更新，我们只需更新covers列表中的最后一个元素。
                if (covers.size()==cover_begin)
                    covers.push_back(lastCover);
                else
                    covers.back()=lastCover;
            }else{ // 如果无法合并...
                if (covers.size()>cover_begin)
                    // 尝试对新匹配进行共线性微调。
                    tryCollinear(covers.back(),matchCover,diff,diffLimit);
                // 将（可能被微调过的）新匹配加入列表。
                covers.push_back(matchCover);
            }
        }else{ // 如果禁用高级优化，则直接添加。
            covers.push_back(matchCover);
        }

        // --- 更新状态并前进 ---
        lastCover=covers.back(); // 更新 lastCover 状态。
        // 贪心跳跃：将搜索位置 newPos 直接移动到刚接受的 cover 的末尾。
        newPos=std::max(newPos+1,lastCover.newPos+lastCover.length);
    }
}
```

## extend_cover **启发式 Cover 边界扩展**

这个函数是 `_dispose_cover` 优化流水线中的一个关键组件。它的任务是遍历 `covers` 列表，并尝试将每一个 `cover` 的边界**向前后两个方向尽力延伸**。这种扩展不是基于精确匹配，而是基于一种**“模糊”匹配**或**“足够相似”**的启发式规则。

### **原理与设计思路解析**

`extend_cover` 的核心目标是**最大化单个 `cover` 的长度**。一个更长的 `cover` 意味着可以从旧文件中复用更多的数据，从而减少了需要存储在补丁中的“新数据”或“差分数据”，最终达到减小补丁体积的目的。

*   **核心策略：带相似度阈值的模糊扩展**
    *   `_search_cover` 找到的 `cover` 都是**精确匹配**的。然而，在精确匹配区域的紧邻边界，数据通常也是**高度相似**的，可能只有一两个字节不同。
    *   `extend_cover` 正是为了利用这种局部相似性。它不会在遇到第一个不匹配字节时就停止，而是会继续向外探索，只要在一个滑动窗口内的**总体相似度**（由 `kExtendMinSameRatio` 定义）足够高，它就会继续扩展。
    *   这个核心的模糊扩展逻辑由辅助函数 `getCanExtendLength` (此处未显示，但在之前的 `diff.cpp` 概览中已提及) 来实现。

*   **执行流程**

    1.  **确定扩展边界:**
        *   `for (size_t i=cover_begin; ...)`: 函数遍历所有需要处理的 `covers`。
        *   对于当前的 `curCover`，它需要知道自己可以向前和向后扩展的**最大安全范围**。
        *   **向前边界 (`lastNewEnd`):** `curCover` 最多只能向前扩展到上一个 `cover` 的末尾。`lastNewEnd` 变量用于跟踪上一个 `cover` 处理完后的结束位置。
        *   **向后边界 (`newPos_next`):** `curCover` 最多只能向后扩展到下一个 `cover` 的开头。

    2.  **受限搜索下的边界调整 (`diffLimit`):**
        *   在受限搜索模式下，扩展行为必须受到 `listener` 的严格控制。
        *   在向前和向后扩展之前，函数会分别调用 `limitCover_front` 和 `limitCover` 回调。
        *   `listener` 可以根据自己的规则（如 in-place 的安全距离）来**缩减**允许的扩展范围，从而更新 `lastNewEnd` 和 `newPos_next` 的值。这确保了即使在模糊扩展时，也不会违反上层逻辑施加的硬性约束。

    3.  **双向扩展:**
        *   **向前扩展 (`Extend forward`):**
            *   调用 `getCanExtendLength`，传入 `-1` 作为方向，尝试从 `curCover` 的**前边界**向**左**延伸。
            *   如果找到了一个有效的扩展长度 `extendLength_front > 0`，就更新 `curCover` 的 `oldPos`, `newPos` 和 `length` 来包含这个延伸部分。
        *   **向后扩展 (`Extend backward`):**
            *   调用 `getCanExtendLength`，传入 `1` 作为方向，尝试从 `curCover` 的**后边界**向**右**延伸。
            *   如果找到了有效的扩展长度 `extendLength_back > 0`，就直接增加 `curCover.length`。

    4.  **更新状态:**
        *   `lastNewEnd = curCover.newPos + curCover.length;`
        *   在处理完一个 `cover` 后，更新 `lastNewEnd` 的位置，为处理下一个 `cover` 做好准备。

**`getCanExtendLength` 的内部工作原理 (推测):**

*   它会从给定的起始点，按指定方向逐字节比较新旧数据。
*   它会维护一个**滑动窗口**（比如，最近的4个或8个字节）。
*   在每一步，它都会计算这个窗口内的**字节匹配率**。
*   它会记录下匹配率达到峰值时的扩展长度。
*   最终，如果这个峰值匹配率**高于**传入的 `kExtendMinSameRatio` 阈值，它就返回对应的“最佳”扩展长度，否则返回0。

### **代码解析**

```cpp
// 尝试扩展 cover 区域
static void extend_cover(std::vector<TOldCover>& covers,size_t cover_begin,const TDiffData& diff,
                         const TFixedFloatSmooth kExtendMinSameRatio,TDiffLimit* diffLimit=0){
    // lastNewEnd 跟踪上一个 cover 处理完后的边界，作为当前 cover 向前扩展的限制
    TInt lastNewEnd=diffLimit?diffLimit->newPos:0;

    // 遍历所有待处理的 covers
    for (size_t i=cover_begin; i<covers.size(); ++i) {
        TInt newPos_next; // 当前 cover 向后扩展的限制
        if (i+1<covers.size()) // 如果有下一个cover，则限制为下一个cover的开头
            newPos_next=covers[i+1].newPos;
        else // 否则，限制为整个新文件的末尾
            newPos_next=diffLimit?diffLimit->newEnd:(TInt)(diff.newData_end-diff.newData);
        
        TOldCover& curCover=covers[i];

        // 在受限模式下，通过 listener 回调来调整扩展边界
        if (diffLimit){
            // 调整向前边界
            TInt limit_front=std::min(curCover.newPos-lastNewEnd,curCover.oldPos);
            if (limit_front>0){
                hpatch_TCover cover={...};
                hpatch_StreamPos_t lenLimit;
                diffLimit->listener->limitCover_front(diffLimit->listener,&cover,&lenLimit);
                // 使用 listener 返回的允许长度来更新 lastNewEnd
                lastNewEnd=curCover.newPos-(TInt)lenLimit;
            }

            // 调整向后边界
            TInt limit_back=newPos_next-(curCover.newPos+curCover.length);
            // ... 边界检查 ...
            if (limit_back>0){
                hpatch_TCover cover={...};
                hpatch_StreamPos_t lenLimit;
                diffLimit->listener->limitCover(diffLimit->listener,&cover,&lenLimit);
                // 使用 listener 返回的允许长度来更新 newPos_next
                newPos_next=(curCover.newPos+curCover.length)+ (TInt)lenLimit;
            }
        }

        // --- 执行双向扩展 ---

        // 向前扩展 (Extend forward)
        // getCanExtendLength 会返回在满足相似度阈值的情况下，可以向前延伸的最大长度
        TInt extendLength_front=getCanExtendLength(curCover.oldPos-1,curCover.newPos-1,
                                                   -1,lastNewEnd,newPos_next,diff,kExtendMinSameRatio);
        if (extendLength_front>0){
            // 更新 cover 的属性以包含延伸部分
            curCover.oldPos-=extendLength_front;
            curCover.newPos-=extendLength_front;
            curCover.length+=extendLength_front;
        }

        // 向后扩展 (Extend backward)
        TInt extendLength_back=getCanExtendLength(curCover.oldPos+curCover.length,
                                                  curCover.newPos+curCover.length,
                                                  1,lastNewEnd,newPos_next,diff,kExtendMinSameRatio);
        if (extendLength_back>0){
            // 只需增加 length 即可
            curCover.length+=extendLength_back;
        }

        // 更新 lastNewEnd，为处理下一个 cover 做准备
        lastNewEnd=curCover.newPos+curCover.length;
    }
}
```

## select_cover & _select_cover **成本效益驱动的 Cover 选择与合并**

这是 `_dispose_cover` 流水线中**最核心的决策阶段**。在 `_search_cover` 找到了所有可能的原始匹配，并且 `extend_cover` 对它们进行了初步扩展之后，`select_cover` 的任务是对这些候选 `covers` 进行一次彻底的**审查**，决定**哪些该保留，哪些该丢弃，以及哪些可以合并**。

### **原理与设计思路解析**

`select_cover` 的核心是一种**贪心的、基于成本效益分析**的动态规划思想。它通过一次线性扫描，原地（in-place）地构建出一个最终的、不重叠的 `covers` 列表。

*   **核心策略：成本效益分析 (Cost-Benefit Analysis)**
    *   `_search_cover` 和 `extend_cover` 关注的是“技术上”能匹配多长，而 `select_cover` 关注的是“经济上”是否划算。
    *   **`if (!isNeedSave)`** 这个代码块是决策的核心。对于每一个尚未决定是否保留的 `cover` (`covers[i]`)，它会计算一个**净收益分数 (`coverSorce`)**：
        *   `TInt noCoverCost`: 如果**不**使用这个 `cover`，而是将对应的新数据区域当作“新数据”直接存储，其预估的**压缩成本**是多少？
        *   `TInt coverCost`: 如果**使用**这个 `cover`，将对应的新数据区域通过“旧数据 + 差分数据”的方式来表示，其预估的**差分数据压缩成本**是多少？
        *   `getCoverCtrlCost(...)`: 记录这个 `cover` 指令本身（`oldPos`, `newPos`, `length` 的差量编码）需要的**控制成本**是多少？
        *   **`coverSorce = noCoverCost - coverCost - getCoverCtrlCost(...)`**: 这就是这个 `cover` 带来的**净收益**。
        *   `isNeedSave = (coverSorce >= kMinSingleMatchScore)`: 只有当净收益大于等于预设的最小阈值时，这个 `cover` 才被认为是“划算的”，才值得保留。

*   **`TCompressDetect` 的作用**
    *   `nocover_detect` 和 `cover_detect` 是两个 `TCompressDetect` 对象。它们是**熵编码成本估算器**。
    *   它们通过分析已经处理过的数据（通过 `add_chars` 方法），动态地维护一个字符频率模型。
    *   `cost(...)` 方法可以根据这个模型，快速地**估算出**一段给定的数据在压缩后大概会占用多少字节。这使得成本效益分析可以基于一个相当准确的预测，而不是简单的猜测。

*   **合并优化 (Link Merging)**
    `select_cover` 不仅仅是做“保留/丢弃”的决策，它还会主动尝试合并 `covers` 以进一步优化。

    1.  **向前合并 (`Possibility for forward merging`)**:
        *   在决定是否保留 `covers[i]` **之前**，它会先检查 `covers[i]` 是否能与**上一个已经被保留**的 `cover` (`covers[insertIndex-1]`) 进行链接。
        *   如果可以 (`isCanLink`)，它会直接将 `isNeedSave` 设为 `true`，并标记 `isCanLink=true`。这相当于给予了“可合并”的 `cover` 一个优先保留权。

    2.  **向后合并 (`Check possibility for backward link merging`)**:
        *   在检查完向前合并后，它会进行一个更激进的向后扫描。
        *   它会尝试将当前的 `covers[i]` 与**其后的所有**可以连续链接的 `covers` (`covers[j]`) 全部合并成一个巨大的 `cover`。
        *   被合并的 `covers[j]` 会被原地标记为已删除 (`covers[j].length = 0;`)。

*   **原地算法 (In-place Algorithm)**
    *   整个函数通过 `insertIndex` 这个写指针，实现了一个高效的原地过滤和合并算法。
    *   它遍历整个 `[cover_begin, coverSize_old)` 范围的 `covers`。
    *   只有被判定为 `isNeedSave` 的 `cover` 才会被移动到 `covers[insertIndex]` 的位置，或者被合并到 `covers[insertIndex-1]` 中。
    *   `for` 循环结束后，`[cover_begin, insertIndex)` 区域就包含了所有最终被选中的 `covers`。
    *   `covers.resize(insertIndex);` 最后这一句，直接将 `vector` 的大小调整为 `insertIndex`，就完成了对所有无用 `covers` 的删除。

### **代码解析**

```cpp
/**
 * @brief 对一组候选 covers 进行成本效益分析，选择并合并最优的组合。
 */
static void _select_cover(std::vector<TOldCover>& covers,size_t cover_begin,const TDiffData& diff,int kMinSingleMatchScore,
                          TCompressDetect& nocover_detect,TCompressDetect& cover_detect,
                          TDiffLimit* diffLimit,bool isCanExtendCover){
    TOldCover lastCover(0,0,0); // 跟踪上一个被保留的cover
    if (diffLimit)
        lastCover=diffLimit->lastCover_back;
    const size_t coverSize_old=covers.size();
    size_t insertIndex=cover_begin; // 写指针，指向下一个要被保留的cover的存放位置

    // 遍历所有候选 covers
    for (size_t i=cover_begin;i<coverSize_old;++i){
        if (covers[i].length<=0) continue; // 跳过已被标记为删除的
        bool isNeedSave=false;
        bool isCanLink=false;

        // 1. 检查是否可以向前合并
        if (isCanExtendCover&&(!isNeedSave)){
            // 如果能和上一个已保留的cover链接...
            if ((insertIndex>cover_begin)&&(covers[insertIndex-1].isCanLink(covers[i]))){
                if (diffLimit){ // 受限模式下需征求listener同意
                    // ...
                    if (diffLimit->listener->limitCover(diffLimit->listener,&cover,0)){
                        isCanLink=true; isNeedSave=true;
                    }
                }else{ // 普通模式下直接同意
                    isCanLink=true; isNeedSave=true;
                }
            }
        }

        // 2. 检查并执行向后合并
        if (isCanExtendCover&&(i+1<coverSize_old)){
            // 从i+1开始向后扫描，寻找所有可以连续合并的cover
            for (size_t j=i+1;j<coverSize_old; ++j) {
                if (!covers[i].isCanLink(covers[j])) break; // 如果不能链接，则停止
                if (diffLimit){ // 受限模式下需征求listener同意
                    // ...
                    if (diffLimit->listener->limitCover(diffLimit->listener,&cover,0)){
                        covers[i].Link(covers[j]); // 合并
                        covers[j].length=0; // 标记为删除
                    }else{ break; }
                }else{
                    covers[i].Link(covers[j]); // 合并
                    covers[j].length=0; // 标记为删除
                }
            }
        }

        // 3. 成本效益分析，决定是否保留
        if (!isNeedSave){ // 如果没有因为可合并而被优先保留...
            // 估算“不使用cover”的成本
            TInt noCoverCost=nocover_detect.cost(diff.newData+covers[i].newPos,covers[i].length);
            // 估算“使用cover”的差分数据成本
            TInt coverCost=cover_detect.cost(diff.newData+covers[i].newPos,covers[i].length, diff.oldData+covers[i].oldPos);
            // 计算净收益 = 收益 - 成本
            TInt coverSorce=noCoverCost-coverCost-getCoverCtrlCost(covers[i],lastCover);
            isNeedSave=(coverSorce>=kMinSingleMatchScore); // 只有净收益大于阈值才保留
        }
        
        // 4. 执行保留或合并操作
        if (isNeedSave){
            if (isCanLink){ // 如果是向前合并
                covers[insertIndex-1].Link(covers[i]); // 直接扩展上一个已保留的cover
                // 更新成本估算模型
                cover_detect.add_chars(diff.newData+lastCover.newPos+lastCover.length,
                                       covers[insertIndex-1].length-lastCover.length,
                                       diff.oldData+lastCover.oldPos+lastCover.length);
            }else{ // 如果是独立保留
                covers[insertIndex++]=covers[i]; // 移动到写指针位置
                // 更新成本估算模型
                nocover_detect.add_chars(diff.newData+lastCover.newPos+lastCover.length,
                                         covers[i].newPos-(lastCover.newPos+lastCover.length));
                cover_detect.add_chars(diff.newData+covers[i].newPos,covers[i].length,
                                       diff.oldData+covers[i].oldPos);
            }
            lastCover=covers[insertIndex-1]; // 更新 lastCover 状态
        }
    }
    // 5. 调整大小，完成原地删除
    covers.resize(insertIndex);
}

// select_cover 的一个包装器，用于处理 diffLimit==0 的情况
static void select_cover(std::vector<TOldCover>& covers,size_t cover_begin,const TDiffData& diff,
                         int kMinSingleMatchScore,TDiffLimit* diffLimit,bool isCanExtendCover){
    if (diffLimit==0){ // 如果不在受限模式下
        TCompressDetect  nocover_detect; // 创建临时的成本估算器
        TCompressDetect  cover_detect;
        _select_cover(covers,cover_begin,diff,kMinSingleMatchScore,nocover_detect,cover_detect,0,isCanExtendCover);
    }else{ // 如果在受限模式下
        // 使用 diffLimit 中提供的、已经包含了上下文信息的成本估算器
        _select_cover(covers,cover_begin,diff,kMinSingleMatchScore,diffLimit->nocover_detect,
                      diffLimit->cover_detect,diffLimit,isCanExtendCover);
    }
}
```

## _dispose_cover **Cover 优化与选择流水线**

这是 `search_and_dispose_cover` 的“决策”阶段。它接收 `_search_cover` 找到的所有原始、可能重叠的候选 `covers`，然后通过一个**多阶段的优化流水线 (Optimization Pipeline)**，从中筛选、合并和扩展，最终生成一组高质量、不重叠的 `covers`。

### **原理与设计思路解析**

`_dispose_cover` 的核心职责是**最大化 `covers` 带来的总体收益**。它通过一个精心设计的“扩展-选择-再扩展”的流程来实现这一目标。

*   **核心策略：扩展与选择的交替迭代**
    这个函数的设计基于一个重要的观察：`select_cover`（选择）和 `extend_cover`（扩展）这两个操作是**相辅相成**的。
    *   **`extend_cover`** 负责将单个 `cover` 的边界向外延伸，尽可能地增加匹配长度，从而**提高单个 `cover` 的“质量”**。
    *   **`select_cover`** 负责从一堆（可能重叠的）`covers` 中，根据成本效益分析，挑选出最优的组合，并尝试**合并**相邻的 `covers`，从而**优化 `covers` 的“组合”**。

    单纯地先扩展再选择，或者先选择再扩展，都可能无法得到最优解。例如：
    *   如果先选择，可能会因为一个 `cover` 质量不高而被丢弃，从而失去了一个本可以被扩展成高质量 `cover` 的机会。
    *   如果先扩展，可能会导致两个 `cover` 互相延伸从而产生重叠，使得 `select_cover` 在决策时面临更复杂的情况。

    因此，`_dispose_cover` 采用了**三明治结构**的流水线：

    1.  **`extend_cover` (第一次):** 首先，对所有原始 `covers` 进行一次初步的边界扩展。这使得每个 `cover` 的潜力都得到初步的发挥，为后续的选择提供更高质量的候选者。
    2.  **`select_cover` (核心选择):** 然后，在这些经过初步扩展的 `covers` 基础上，进行核心的选择和合并。`select_cover` 会丢弃“不划算”的 `covers`，并合并那些可以连接的，生成一个初步的、不重叠的最优组合。
    3.  **`extend_cover` (第二次):** `select_cover` 的合并操作可能会创造出新的、更大的 `cover`，或者改变 `cover` 之间的间隙。因此，在选择和合并之后，**再次**调用 `extend_cover` 是非常必要的。这次扩展可以在新的、更优的 `cover` 组合的基础上，进一步探索延伸的可能性，从而“锦上添花”，获得最终的优化结果。

*   **对 `isCanExtendCover` 的处理**
    *   代码通过 `isCanExtendCover` 这个布尔标志来控制是否启用这个复杂的“扩展-选择-再扩展”流水线。
    *   如果 `isCanExtendCover` 为 `false`，则算法会跳过所有的 `extend_cover` 步骤，只执行一次核心的 `select_cover`。这提供了一个选项，可以在需要更快速度（但可能牺牲一些补丁大小）的场景下，禁用计算开销较大的边界扩展功能。

*   **`kExtendMinSameRatio` 的动态计算**
    *   `TFixedFloatSmooth kExtendMinSameRatio=kMinSingleMatchScore*36+254;`
    *   `kExtendMinSameRatio` 是传递给 `extend_cover` 的一个关键参数，它代表了在扩展边界时，所能容忍的最低“相似度”。
    *   这个值不是固定的，而是根据 `kMinSingleMatchScore` **动态计算**出来的。这意味着，如果上层算法要求更高的匹配质量（`kMinSingleMatchScore` 很高），那么边界扩展的条件也会变得更苛刻，反之亦然。这使得 `extend_cover` 的行为能够自适应于整体的匹配策略。

### **代码解析**

```cpp
/**
 * @brief 对 _search_cover 找到的原始 covers 进行优化、选择和扩展。
 * @param covers        包含原始 covers 的向量，此函数会对其进行原地修改。
 * @param cover_begin   指示 covers 向量中需要处理部分的起始索引。
 * @param diff          包含新旧文件数据的上下文。
 * @param kMinSingleMatchScore 单个匹配的最低有效分数。
 * @param diffLimit     (可选) 受限搜索的上下文。
 * @param isCanExtendCover 控制是否启用边界扩展功能。
 */
static void _dispose_cover(std::vector<TOldCover>& covers,size_t cover_begin,const TDiffData& diff,
                          int kMinSingleMatchScore,TDiffLimit* diffLimit,bool isCanExtendCover){
    // 如果启用了扩展功能...
    if (isCanExtendCover){
        // 动态计算用于边界扩展的最低相似度阈值
        TFixedFloatSmooth kExtendMinSameRatio=kMinSingleMatchScore*36+254;
        if  (kExtendMinSameRatio<200) kExtendMinSameRatio=200; // 设置下限
        if (kExtendMinSameRatio>800) kExtendMinSameRatio=800; // 设置上限

        // --- 流水线阶段一：初步扩展 ---
        // 尝试扩展所有原始 covers 的边界
        extend_cover(covers,cover_begin,diff,kExtendMinSameRatio,diffLimit);
        
        // --- 流水线阶段二：选择与合并 ---
        // 在初步扩展的基础上，进行成本效益分析，选择最优组合并合并
        select_cover(covers,cover_begin,diff,kMinSingleMatchScore,diffLimit,isCanExtendCover);
        
        // --- 流水线阶段三：再次扩展 ---
        // select_cover 可能会删除或合并 covers，形成新的组合。
        // 在新组合的基础上，再次尝试扩展，以获得最终的优化结果。
        extend_cover(covers,cover_begin,diff,kExtendMinSameRatio,diffLimit);
    }else{ // 如果禁用了扩展功能...
        // ...则只执行核心的选择与合并步骤。
        select_cover(covers,cover_begin,diff,kMinSingleMatchScore,diffLimit,isCanExtendCover);
    }
}
```

## _limitCoverLenth **超长 Cover 分割**

这个函数是一个后处理工具，它的唯一职责是确保 `covers` 列表中的**每一个 `cover` 的长度都不超过一个预设的最大值 `kMaxLen`**。如果一个 `cover` 过长，它会被**分割**成多个符合长度限制的小 `cover`。

### **原理与设计思路解析**

为什么需要限制 `cover` 的最大长度？这通常与**补丁格式的限制**或**应用补丁时的内存策略**有关。

*   **动机:**
    1.  **格式限制:** 某些补丁格式可能使用固定大小的整数来编码 `cover` 的长度，如果长度超过该整数能表示的最大值，就会出错。
    2.  **内存管理:** 在应用补丁时，`hpatch` 可能需要分配一块与 `cover.length` 大小相等的临时缓冲区。如果 `cover` 过长（例如几个GB），就可能导致内存分配失败。
    3.  **流式处理:** 对于流式应用的补丁格式，将一个巨大的操作分解成多个小操作，可能更有利于流水线处理和内存控制。

`_limitCoverLenth` 通过一个高效的**两阶段（Two-Pass）原地算法**来实现分割。

*   **核心策略：两阶段原地分割**

    1.  **第一阶段：计数与预留空间 (Counting and Reservation)**
        *   `for (size_t i=0;i<csize;++i){ ... }`: 算法首先遍历一次 `covers` 列表。
        *   **目的:** 计算出如果对所有超长 `cover` 进行分割，**总共会产生多少个新的 `cover`**。这个数量被累加到 `isize` 中。
        *   `while (clen > kMaxLen)` 和 `_clipLenByLimit`: `_clipLenByLimit` (此处未显示，但其行为可以推断) 是一个辅助函数，它会计算出从一个长度为 `clen` 的 `cover` 中，应该“切下”多长的一块（返回 `clipLen`），并更新 `clen`。这个循环会模拟切割过程，直到 `clen` 不再超长，并在此过程中统计需要新增的 `cover` 数量。
        *   `if (isize==0) return;`: 如果没有超长的 `cover`，则无需任何操作，函数提前返回。
        *   `covers.resize(csize+isize);`: **预留空间**。这是原地算法的关键。它一次性地将 `vector` 的大小扩展到最终所需的大小。
        *   `memmove(covers.data()+isize, ...);`: 将原始的所有 `cover` 数据**整体向后移动 `isize` 个位置**，从而在 `vector` 的**开头**腾出 `isize` 个元素的空闲空间，用于存放后续分割产生的新 `cover`。

    2.  **第二阶段：分割与填充 (Splitting and Filling)**
        *   `for (size_t i=isize; ...)`: 算法现在从**第二个副本**（即被后移的原始 `covers`）的开头开始遍历。
        *   `size_t insertIndex=0;`: `insertIndex` 是一个写指针，从 `0` 开始，指向 `vector` 开头腾出的空闲区域。
        *   `while (clen>kMaxLen){ ... }`: 再次模拟切割过程。
        *   `TInt clipLen=(TInt)_clipLenByLimit(clen,kMaxLen);`: 计算出要切下的第一块的长度。
        *   `covers[insertIndex++]=TOldCover(..., clipLen);`: **创建新的 `cover`**。使用原始 `cover` 的起始位置和计算出的 `clipLen`，在开头的空闲区域创建一个新的小 `cover`。
        *   `covers[i].oldPos+=clipLen; ...`: **更新原始 `cover`**。将被切掉的部分从原始 `cover` 中“移除”，即更新其起始位置和长度。
        *   `covers[insertIndex++]=covers[i];`: 当原始 `cover` `covers[i]` 不再超长后，将这个**剩余的部分**也作为一个 `cover` 写入到 `insertIndex` 的位置。
        *   这个过程不断重复，直到所有原始 `cover` 都被处理完毕。`insertIndex` 最终会恰好等于 `csize+isize`，`isize` 也会减到0。

### **代码解析**

```cpp
/**
 * @brief 遍历 covers 列表，并将所有长度超过 kMaxLen 的 cover 分割成多个小 cover。
 * @param covers    待处理的 cover 列表，会被原地修改。
 * @param kMaxLen   cover 的最大允许长度。
 */
static void _limitCoverLenth(std::vector<TOldCover>& covers,hpatch_StreamPos_t kMaxLen){
    assert((0<kMaxLen)&&(0<(TInt)kMaxLen)&&(kMaxLen==(TInt)kMaxLen)); // 确保 kMaxLen 有效
    size_t csize=covers.size();
    size_t isize=0; // 用于统计需要新增的 cover 数量

    // --- 阶段一：计数与预留空间 ---
    // 第一次遍历：只计数，不修改
    for (size_t i=0;i<csize;++i){
        hpatch_StreamPos_t clen=covers[i].length;
        // 模拟分割过程
        while (clen>kMaxLen){
            ++isize; // 每需要切一次，就增加一个新 cover 的名额
            _clipLenByLimit(clen,kMaxLen); // _clipLenByLimit 会更新 clen
        }
    }
    if (isize==0) return; // 如果没有超长的 cover，提前返回

    // 将 vector 扩容到最终大小
    covers.resize(csize+isize);
    // 将原始数据整体向后移动，在数组开头腾出 isize 个空位
    memmove(covers.data()+isize,covers.data(),sizeof(TOldCover)*csize);

    // --- 阶段二：分割与填充 ---
    size_t insertIndex=0; // 写指针，从数组开头开始
    // 第二次遍历：从后移的数据副本开始读
    for (size_t i=isize;i<covers.size();++i){
        hpatch_StreamPos_t clen=covers[i].length;
        // 对每个超长的 cover 进行实际的分割
        while (clen>kMaxLen){
            --isize;
            // 计算要切下的第一块的长度
            TInt clipLen=(TInt)_clipLenByLimit(clen,kMaxLen);
            assert(insertIndex<i);
            // 在开头的空闲区域创建一个新的、长度为 clipLen 的 cover
            covers[insertIndex++]=TOldCover(covers[i].oldPos,covers[i].newPos,clipLen);
            
            // 更新原始 cover 的属性，相当于“切掉”了前面一部分
            covers[i].oldPos+=clipLen;
            covers[i].newPos+=clipLen;
            covers[i].length-=clipLen;
            assert(covers[i].length==clen); // 确认长度更新正确
        }
        // 将原始 cover 剩余的部分（现在已不超长）也写入
        covers[insertIndex++]=covers[i];
    }
    assert(isize==0); // 最终，所有预留的位置都应被填满
}
```

## _getSubDiff **差分数据计算**

`_getSubDiff` 是 `serialize_lite_diff` 函数在序列化 `cover` 数据时调用的核心计算函数。它的唯一职责是：对于一个给定的 `cover` 区域，计算出**新旧数据之间的逐字节差值**，并将结果存入 `subDiff` 缓冲区。

### **原理与设计思路解析**

这个函数是 HDiffPatch **加法差分 (Additive Diff)** 算法的直接体现。

*   **核心策略：逐字节减法**
    *   **补丁生成时 (Diffing):**
        *   `subDiff[i] = pnew[i] - pold[i];`
        *   这个循环精确地实现了差分公式 `DiffData = NewData - OldData`。它遍历 `cover` 所覆盖的每一个字节，将新文件对应字节的值减去旧文件对应字节的值，得到的差值就是需要存储在补丁中的“差分数据”。
    *   **补丁应用时 (Patching):**
        *   在应用补丁时（如 `hpatch_lite_patch` 中的 `addData` 函数），会执行逆向操作：`NewData = OldData + DiffData`。
        *   `*dst++ += *src++;`
        *   通过将从旧文件中读取的数据与补丁中的差分数据相加，就可以完美地还原出新文件的数据。

*   **为什么这个方法有效？**
    *   正如我们之前讨论过的，文件的大部分内容在版本更新中是**不变或微小改动**的。
    *   当 `pnew[i] == pold[i]` 时，`subDiff[i]` 的结果是 **0**。
    *   当 `pnew[i]` 和 `pold[i]` 的值非常接近时，`subDiff[i]` 的结果是一个**绝对值很小的数**（例如 1, -1, 2, -2 等）。
    *   因此，`_getSubDiff` 计算出的 `subDiff` 向量中会充满大量的 **0 和小数值**。这种数据的**熵 (entropy)** 非常低，非常适合被后续的通用压缩算法（如 zlib, lzma）进行高效压缩。

*   **实现细节**
    *   `subDiff.resize(cover.length);`: 在计算前，函数首先确保 `subDiff` 向量有足够的空间来存储所有差值。
    *   `const TByte* pnew=...`, `const TByte* pold=...`: 为了提高代码可读性和效率，它预先计算出新旧数据块的起始指针。
    *   `for (size_t i=0; ...)`: 一个简单的循环，逐字节执行减法操作。由于这是C++的 `unsigned char`（在 HDiffPatch 中 `TByte` 是其别名），减法会进行**模256算术**，这正是差分算法所需要的。例如，`5 - 10` 的结果是 `251` (即 `-5 mod 256`)。在打补丁时，`10 + 251` 的结果是 `261`，在 `unsigned char` 中溢出后，结果恰好是 `5`，实现了正确的还原。

### **代码解析**

```cpp
/**
 * @brief 计算一个 cover 区域内新旧数据之间的逐字节差分。
 * @param subDiff    [输出] 用于存储差分结果的向量。
 * @param diff       包含新旧文件数据的上下文。
 * @param cover      定义了需要计算差分的区域。
 */
static void _getSubDiff(std::vector<TByte>& subDiff,const TDiffData& diff,const TOldCover& cover){
    // 1. 确保输出向量有足够的空间。
    subDiff.resize(cover.length);
    
    // 2. 获取指向新旧数据块的指针。
    const TByte* pnew=diff.newData+cover.newPos;
    const TByte* pold=diff.oldData+cover.oldPos;
    
    // 3. 逐字节执行减法运算。
    for (size_t i=0;i<subDiff.size();++i)
        subDiff[i]=pnew[i]-pold[i];
}
```
