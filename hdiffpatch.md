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

# HDiffPatch\libHDiffPatch\HDiff\diff.cpp
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