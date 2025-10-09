---
title: tinyuz
categories:
  - hpatch
tags:
  - hpatch
abbrlink: d7123d36
date: 2025-10-04 18:57:17
---
[TOC]
<meta name="referrer" content="no-referrer" />


![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/a9fb8a0cc36b4fdbbf3578ddf4cbdfa1.png)
# tinyuz\compress\tuz_enc.cpp

## tuz_compress **Tuz 数据压缩核心实现**

- 负责执行 Tuz 无损压缩算法，将输入数据流 (`data`) 压缩后写入输出数据流 (`out_code`)。该实现同时支持高效的单线程和多线程压缩模式。

### **原理与设计思路解析**

`tuz_compress` 函数是 Tuz 压缩器的核心，它负责编排整个压缩流程。其设计思想围绕着**分块处理 (Clipping)** 和**并行计算**，以在处理大规模数据时兼顾内存效率和执行速度。

*   **核心压缩策略**
    1.  **写入文件头 (Header First):** 压缩开始时，首先会向输出流写入一个头部。这个头部包含了后续解压所必需的元数据，最主要的是**字典大小 (Dictionary Size)**。
    2.  **数据分块 (Clipping):** 为了有效管理内存并为并行化创造条件，输入数据不会被一次性加载。相反，它会被切分成连续的、大小适中的数据块，称为 "clip"。每个 clip 的大小会根据字典大小进行策略性计算，以平衡压缩率和处理开销。
    3.  **分块压缩:** 真正的压缩逻辑由 `compress_clip` 函数（在本代码片段中被调用，但其定义在别处）执行。该函数对每一个 clip 进行处理，利用一个滑动字典来查找重复数据序列，并将其编码为 (长度, 距离) 对。
    4.  **写入控制码:** 在每个 clip 压缩完成后，会向输出流中写入一个特殊的**控制码** (`outCtrl_clipEnd`)，用于标记数据块的结束。当所有数据都处理完毕后，则写入一个流结束控制码 (`outCtrl_streamEnd`)。
    5.  **动态字典大小优化:** 一个精巧的设计是，在整个压缩过程结束后，函数会检查实际使用的最大字典大小 (`curDictSizeMax`)。如果这个值小于最初设定的字典大小，它会重写文件头，将字典大小更新为这个更精确、更小的值。这可以在不影响解压的前提下，节省几个字节的头部空间。

*   **多线程设计 (Producer-Consumer 模型)**
    当用户指定使用多个线程时，代码会切换到一套精心设计的并行处理模型：
    *   **任务分发与执行:** 主线程将整个数据流划分为多个 clips，并将这些 clips 作为独立的任务。然后启动一个工作线程池，每个线程从任务池中领取一个 clip 进行压缩。
    *   **独立处理:** 每个工作线程都拥有自己独立的字典缓冲区 (`TDictBuf`) 和编码器 (`TTuzCode`)，从而避免了线程间的锁竞争，实现了高度并行的压缩处理。
    *   **有序结果提交:** 由于线程完成任务的顺序是不确定的，代码采用了一个**有序写入**机制 (`finishWork` 函数，在多线程控制器 `TMt` 中实现)。当一个线程完成压缩后，它会尝试提交结果。如果它所处理的 clip 正是下一个需要被写入输出流的块，则直接写入；否则，它会将结果暂存到一个按顺序排序的链表中，等待轮到它时再由其他线程或主线程写入。这个机制确保了即使工作线程乱序完成，最终的压缩产物依然是正确、连续的。

*   **设计优点**
    *   **内存效率:** 分块处理策略避免了将整个文件加载到内存中，使得压缩超大文件成为可能。
    *   **高可伸缩性:** 多线程架构能充分利用现代多核 CPU 的计算能力，在处理大文件时显著提升压缩速度。
    *   **灵活性:** `tuz_TCompressProps` 结构体允许调用者根据需求（例如，速度优先还是压缩率优先）灵活调整字典大小、线程数等关键参数。
    *   **自适应优化:** 压缩结束时对字典大小的动态更新是一个锦上添花的优化，体现了设计的精细。

### **代码解析**

```c
/**
 * @brief Tuz 压缩主函数。
 * @param out_code          压缩数据的输出流。
 * @param data              原始数据的输入流。
 * @param props             压缩参数配置，可为 NULL 使用默认值。
 * @return hpatch_StreamPos_t 返回压缩后的总大小。
 */
hpatch_StreamPos_t tuz_compress(const hpatch_TStreamOutput* out_code,const hpatch_TStreamInput* data,
                                const tuz_TCompressProps* props){
    // --- 1. 参数校验与初始化 ---
    
    // checkv 是一个校验宏，确保输入输出流有效。
    checkv(out_code&&(out_code->write));
    checkv(data&&(data->read));
    if (props){ // 如果用户传入了自定义配置，则校验其合法性。
        checkv((props->dictSize>=1)&(props->dictSize<=tuz_kMaxOfDictSize)); // 字典大小范围
        checkv(props->dictSize==(tuz_size_t)props->dictSize); // 字典大小类型
        checkv(props->maxSaveLength==(tuz_length_t)props->maxSaveLength); // 匹配长度类型
        checkv((props->maxSaveLength>=tuz_kMinOfMaxSaveLength)&&(props->maxSaveLength<=tuz_kMaxOfMaxSaveLength)); // 匹配长度范围
    }
    
    // 若 props 为空，使用默认配置；否则，拷贝一份到 selfProps。
    tuz_TCompressProps selfProps=(props)?*props:tuz_kDefaultCompressProps;
    // 优化：如果设定的字典大小超过了原始数据大小，则将其调整为原始数据大小，因为更大的字典是无用的。
    if (selfProps.dictSize>data->streamSize){
        selfProps.dictSize=(size_t)(data->streamSize);
        if (selfProps.dictSize==0) // 字典大小至少为1，以防数据为空。
            selfProps.dictSize=1;
    }
    
    hpatch_StreamPos_t cur_out_pos=0; // 记录当前在输出流中的写入位置。
    std::vector<tuz_byte> code;       // 一个临时的字节缓冲区，用于暂存生成的压缩码。
    
    // --- 2. 写入文件头 ---
    {//head 作用域块
        assert(code.empty()); // 确保缓冲区是空的。
        // 创建一个编码器实例，它会将编码结果写入 code 缓冲区。
        TTuzCode coder(code,selfProps.isNeedLiteralLine); 
		checkv(selfProps.dictSize==(tuz_size_t)selfProps.dictSize);
        checkv(selfProps.maxSaveLength==(tuz_length_t)selfProps.maxSaveLength);
        // 将字典大小编码并存入 code 缓冲区。
        coder.outDictSize(selfProps.dictSize);
        // 将 code 缓冲区中的头部数据写入到输出流。
        _flush_code(out_code,cur_out_pos,code);
    }

    // --- 3. 计算分块(clip)大小和线程数 ---
    size_t curDictSizeMax=tuz_kMinOfDictSize; // 用于记录压缩全程实际用到的最大字典大小。
    hpatch_StreamPos_t clipSize; // 每个数据块的大小。
    size_t threadNum=(props)?props->threadNum:1; // 从 props 获取线程数，若 props 为空则默认为1。
    { // 作用域块，用于计算 clipSize
        // 这是一个启发式算法，初步将 clipSize 设为字典大小的 1/3 左右。
        clipSize=((hpatch_StreamPos_t)selfProps.dictSize+1)/3;
        // 限制 clipSize 在一个合理的最小和最大值之间。
        if (clipSize<kMinBestClipSize) clipSize=kMinBestClipSize;
        if (clipSize>kMaxBestClipSize) clipSize=kMaxBestClipSize;
        // 根据文件总大小和初步的 clipSize，计算总共需要多少个 clip。
        hpatch_StreamPos_t clipCount=(data->streamSize+clipSize)/clipSize;
        // 再反过来根据 clip 数量，计算出更均匀、能整除的 clipSize。
        clipSize=(data->streamSize+clipCount-1)/clipCount;
        // 优化：线程数不能超过 clip 的数量，否则多余的线程将没有工作。
        if (threadNum>clipCount) threadNum=(size_t)clipCount;
    }
        
    // --- 4. 根据线程数选择执行路径 (多线程或单线程) ---
#if (_IS_USED_MULTITHREAD) // 仅在启用多线程编译时，此代码块有效。
    if (threadNum>1){ // **多线程执行路径**
        TMt mt(out_code,data); // 初始化多线程控制器。
        // 设置控制器的各种状态。
        mt.selfProps=selfProps;
        mt.clipSize=clipSize;
        mt.curWorkClipPos=0;
        mt.curOutedClipPos=0;
        mt.curWritePos=cur_out_pos;
        mt.curDictSizeMax=curDictSizeMax;
        mt.workBufList=0;
        // 创建一个 TWorkBuf 对象池（大小略多于线程数，用于缓冲），并将其放入工作通道中等待线程领取。
        std::vector<TWorkBuf> _codeList;
        _codeList.resize(threadNum+1+threadNum/2);
        for (size_t i=0;i<_codeList.size();++i)
            checkv(mt.work_chan.send(&_codeList[i],true));
        // 启动工作线程池，每个线程执行 _tuz_compress_mt 函数。
        mt.start_threads((int)threadNum,_tuz_compress_mt,&mt,true);

        // 主线程在此等待所有工作线程执行完毕。
        mt.wait_all_thread_end();
        checkv(!mt.is_on_error()); // 检查执行过程中是否有错误发生。
        checkv(mt.curOutedClipPos==data->streamSize); // 确认所有数据都已被处理和输出。
        // 从控制器获取最终的 curDictSizeMax 和 cur_out_pos 状态。
        curDictSizeMax=mt.curDictSizeMax;
        cur_out_pos=mt.curWritePos;
    }else
#endif
    { // **单线程执行路径**
        TDictBuf dict_buf; // 单线程模式下，全程共享一个字典缓冲区。
        // 循环处理每一个 clip，直到整个数据流结束。
        for (hpatch_StreamPos_t clipBegin=0;true;clipBegin+=clipSize) {
            hpatch_StreamPos_t clipEnd=clipBegin+clipSize;
            bool isToStreamEnd=(clipEnd>=data->streamSize);
            if (isToStreamEnd) clipEnd=data->streamSize; // 确保最后一个 clip 不会越界。

            assert(code.empty());
            TTuzCode coder(code,selfProps.isNeedLiteralLine);
            if (clipBegin<clipEnd){
                // 调用核心压缩逻辑处理当前 clip。
                compress_clip(coder,data,clipBegin,clipEnd,selfProps,dict_buf);
            }
            
            // 根据是否为最后一个 clip，向流中输出不同的控制码。
            if (!isToStreamEnd)
                coder.outCtrl_clipEnd();
            else
                coder.outCtrl_streamEnd();
            
            // 跟踪并更新实际使用到的最大字典大小。
            curDictSizeMax=std::max(curDictSizeMax,coder.getCurDictSizeMax());
            // 将当前 clip 压缩后的数据刷新到输出流。
            _flush_code(out_code,cur_out_pos,code);
            if (isToStreamEnd) break; // 如果已到数据末尾，则退出循环。
        }
    }

    // --- 5. 更新文件头中的字典大小 (优化) ---
    {//update dictSize
        checkv(curDictSizeMax<=selfProps.dictSize);
        // 如果实际使用的最大字典大小 小于 文件头中最初记录的值。
        if (curDictSizeMax<selfProps.dictSize){
            assert(code.empty());
            TTuzCode coder(code,selfProps.isNeedLiteralLine);
            // 重新编码一个更小的、精确的字典大小。
            coder.outDictSize(curDictSizeMax);
            hpatch_StreamPos_t dict_out_pos=0; // 设定写入位置为0，即覆盖原文件头。
            _flush_code(out_code,dict_out_pos,code);
        }
    }

    // 返回压缩后的总字节数。
    return cur_out_pos;
}
```

# tinyuz\compress\tuz_enc_private\tuz_enc_clip.cpp
## compress_clip **Tuz 数据块压缩核心逻辑**

- 作为 `tuz_compress` 调用的核心工作函数，负责对单个数据块（clip）执行实际的 LZ77-style 压缩算法。

### **原理与设计思路解析**

`compress_clip` 函数是 Tuz 压缩算法的心脏。它接收一个明确的数据范围（`clipBegin` 到 `clipEnd`），并利用一个“滑动窗口”（由 `TDictBuf` 维护）作为字典来查找和编码重复的数据序列。

*   **核心原理: 滑动窗口压缩 (Sliding Window)**
    函数的核心思想是在一个连续的内存缓冲区中进行操作。这个缓冲区不仅包含当前需要压缩的数据块（clip），还包含了紧接在该数据块之前的、作为“字典”使用的数据。
    
    1.  **数据准备:** 在压缩开始前，函数会创建一个足够大的内存缓冲区 `data_buf`。它首先计算出需要的字典数据的起始位置 `dictBeginPos`，然后将从 `dictBeginPos` 到 `clipEnd` 的所有数据一次性读入这个缓冲区。
    2.  **高效读取:** 为了提高效率，这里有一个重要的优化。如果前一个 clip 处理后留下的字典数据（存储在 `dict_buf` 中）与当前 clip 所需的字典有重叠，它会通过 `memmove` 将这部分重叠数据移动到新缓冲区的起始位置，然后只从输入流中读取剩余的新数据。这避免了对同一数据区域的重复读取。
    3.  **迭代匹配:** 函数使用一个 `TMatch` 对象，从当前 clip 的起始位置开始，逐字节地向后扫描 (`cur` 指针)。在每个位置，`matcher.match()` 会在 `cur` 指针之前的整个字典+已压缩区域中，寻找与当前位置开始的数据序列最长的匹配项。
    4.  **两种编码输出:**
        *   **字面量 (Literals):** 如果在 `cur` 位置**没有找到**足够长的匹配项，`cur` 指针就向后移动一个字节。从上一个匹配结束的位置 (`back` 指针) 到当前 `cur` 指针之间的所有字节，都构成了“字面量”序列，即无法被压缩的原始数据。
        *   **字典引用 (References):** 如果**找到了**一个匹配项，那么之前累积的字面量序列（`back` 到 `cur` 之间的数据）会首先被输出。然后，找到的匹配项会被编码成一个**(匹配长度, 字典距离)**的引用对，并输出到压缩流。最后，`cur` 指针会一次性跳过整个匹配的长度。
    5.  **字典“滑动”:** 当一个 clip 处理完毕后，它本身就变成了历史数据。函数会执行“滑动”操作：保留 `data_buf` 中尾部的、大小等于字典大小的数据，并将其移动到缓冲区的开头。这部分数据将作为**下一个 clip** 的字典。这个过程由 `update dict` 代码块完成，它更新 `dict_buf` 的状态，为下一次调用 `compress_clip` 做好了准备。

*   **设计优点**
    *   **高效性:** 将字典和待压缩数据全部置于一个连续的内存块中，使得匹配查找操作可以极快地进行，避免了频繁的 I/O 操作。
    *   **内存可控:** 滑动窗口机制确保了内存使用量是可控的，其上限大致为“字典大小 + clip大小”，而与被压缩文件的总大小无关。
    *   **模块化:** 将具体的压缩匹配逻辑封装在 `compress_clip` 中，使得上层的 `tuz_compress` 函数可以专注于数据流的划分、多线程调度等宏观任务，实现了关注点分离。

### **代码解析**

```c
namespace _tuz_private{
    
    /**
     * @brief 输出一段连续的、未匹配的字面量数据。
     * @param back            字面量数据的起始指针。
     * @param unmatched_len   字面量数据的长度。
     * @param coder           编码器实例，用于输出。
     * @param props           压缩配置，主要用于获取 maxSaveLength。
     */
    static void _outData(const tuz_byte* back,size_t unmatched_len,
                         _tuz_private::TTuzCode& coder,const tuz_TCompressProps& props){
        // 由于编码器可能对一次性输出的长度有限制(maxSaveLength)，
        // 这里使用循环来确保超长的数据也能被正确分块输出。
        while (unmatched_len){
            size_t len=(unmatched_len<=props.maxSaveLength)?unmatched_len:props.maxSaveLength;
            coder.outData(back,back+len); // 调用编码器输出一个数据块。
            back+=len;
            unmatched_len-=len;
        }
    }
    
/**
 * @brief 对单个数据块 (clip) 执行压缩。
 * @param coder      编码器实例，所有压缩结果（字面量或引用）都通过它输出。
 * @param data       原始数据输入流。
 * @param clipBegin  当前数据块在整个流中的起始位置。
 * @param clipEnd    当前数据块在整个流中的结束位置。
 * @param props      压缩参数配置。
 * @param dict_buf   字典缓冲区，用于在不同 clip 调用之间传递和维护字典数据。
 */
void compress_clip(TTuzCode& coder,const hpatch_TStreamInput* data,hpatch_StreamPos_t clipBegin,
                   hpatch_StreamPos_t clipEnd,const tuz_TCompressProps& props,TDictBuf& dict_buf){
    
    // --- 1. 准备内存缓冲区，用于存放 字典 + 当前clip ---
    std::vector<tuz_byte>& data_buf(dict_buf.dictBuf); // 使用传入的字典缓冲区。
    hpatch_StreamPos_t dictBeginPos; // 字典在整个数据流中的起始位置。
    const size_t dictSizeBack=data_buf.size(); // 记录进入函数时，缓冲区中已有的旧字典数据大小。
    {
        // 计算字典的起始位置。不能早于文件头，最多向前追溯 dictSize 个字节。
        dictBeginPos=(clipBegin<=props.dictSize)?0:(clipBegin-props.dictSize);
        // 计算需要的总内存大小 = (clipEnd - dictBeginPos)。
        hpatch_StreamPos_t _mem_size=clipEnd-dictBeginPos;
        checkv(_mem_size==(size_t)_mem_size); // 确保大小没有溢出 size_t。
        checkv(_mem_size>=data_buf.size()); // 新大小必须不小于旧大小。
        // 将缓冲区调整到所需大小。
        data_buf.resize((size_t)_mem_size);
    }
    
    // --- 2. 从输入流读取数据到缓冲区 (滑动窗口优化) ---
    {//read data
        hpatch_StreamPos_t readPos=dictBeginPos; // 计划从字典起始位置开始读取。
        // 如果上一个 clip 留下的字典数据 (dict_buf.dictEndPos) 在我们要读的范围之内...
        if (dict_buf.dictEndPos>readPos){
            checkv(dict_buf.dictEndPos<=clipBegin);
            size_t movLen=(size_t)(dict_buf.dictEndPos-readPos); // 计算重叠数据的长度。
            checkv(dictSizeBack>=movLen);
            if (dictSizeBack-movLen>0)
                // 【优化】将上一次的旧字典数据中需要保留的部分，移动到缓冲区的正确位置。
                memmove(data_buf.data(),data_buf.data()+dictSizeBack-movLen,movLen);
            readPos=dict_buf.dictEndPos; // 更新读取位置，只需读取后面新的数据即可。
        }
        // 从流中读取剩余需要的数据，填充到缓冲区的后半部分。
        checkv(data->read(data,readPos,data_buf.data()+(size_t)(readPos-dictBeginPos),data_buf.data()+data_buf.size()));
    }
    
    // --- 3. 执行核心的匹配与编码循环 ---
    // 创建匹配器实例，它将在 data_buf 中进行查找。
    TMatch   matcher(data_buf.data(),data_buf.data()+data_buf.size(),coder,props);
    {//match loop
        const tuz_byte* end=data_buf.data()+data_buf.size();
        // `cur` 是当前扫描指针，指向 clip 在缓冲区中的起始位置。
        const tuz_byte* cur=end-(clipEnd-clipBegin);
        // `back` 指针用于标记当前未匹配的字面量序列的开始位置。
        const tuz_byte* back=cur;
        while (cur!=end){ // 循环直到处理完整个 clip。
            const tuz_byte*     matched;
            size_t              match_len;
            // 尝试在当前 `cur` 位置进行匹配。
            if (matcher.match(&matched,&match_len,cur)){
                // **分支1: 找到匹配**
                checkv(matched<cur); // 匹配位置必须在当前位置之前。
                checkv(matched>=data_buf.data()); // 匹配位置不能越界。
                checkv(cur+match_len<=end); // 匹配长度不能越界。
                checkv(match_len>=tuz_kMinDictMatchLen);
                checkv(match_len<=props.maxSaveLength);
                // 计算从上个匹配点到当前点之间的字面量长度。
                const size_t unmatched_len=(cur-back);
                if (unmatched_len>0)
                    _outData(back,unmatched_len,coder,props); // 输出这段字面量。
                
                // 计算匹配位置相对于当前位置的距离（字典位置）。
                size_t dict_pos=(cur-matched)-1;
                checkv(dict_pos<props.dictSize);
                // 调用编码器输出 (长度, 距离) 引用。
                coder.outDict(match_len,dict_pos);
                
                // 向前跳过整个匹配的长度。
                cur+=match_len;
                back=cur; // 重置字面量起始点。
            }else{
                // **分支2: 未找到匹配**
                // 将当前字节视为字面量的一部分，仅移动指针。
                ++cur;
            }
        }
        // 循环结束后，处理可能遗留的最后一段字面量。
        const size_t unmatched_len=(cur-back);
        if (unmatched_len>0)
            _outData(back,unmatched_len,coder,props);
    }
    
    // --- 4. 更新字典，为处理下一个 clip 做准备 (滑动窗口) ---
    { //update dict
        // 计算下一个 clip 需要的字典大小。
        size_t newDictSize=(props.dictSize<=clipEnd)?props.dictSize:(size_t)clipEnd;
        checkv(data_buf.size()>=newDictSize);
        dict_buf.dictEndPos=clipEnd; // 更新字典在流中的结束位置。
        if (data_buf.size()>newDictSize){
            // 将当前处理完的数据的尾部 (即新的字典内容) 移动到缓冲区的开头。
            memmove(data_buf.data(),data_buf.data()+(data_buf.size()-newDictSize),newDictSize);
            // 缩小缓冲区，释放不再需要的内存。
            data_buf.resize(newDictSize);
        }
    }
}

}
```

# tinyuz\decompress\tuz_dec.c

## tuz_TStream_read_dict_size **Tuz 文件头解析：读取字典大小**
### **原理与设计思路解析**

`tuz_TStream_read_dict_size` 函数是任何 Tuz 解压操作的第一步。它的职责单一且至关重要：读取文件最开头的几个字节，并将它们转换成一个整数，这个整数代表了解压过程所需要的字典缓冲区的大小。

*   **固定长度头部 (Fixed-Length Header)**
    Tuz 格式规定，压缩流的起始位置固定存储着字典的大小值。这个值占用的字节数由一个编译时常量 `tuz_kDictSizeSavedBytes` 决定。这种设计使得头部解析非常快速和简单。

*   **I/O 抽象 (I/O Abstraction)**
    为了让解压库能够适应不同的输入源（例如，从磁盘文件、内存缓冲区或网络流读取），此函数的设计并没有直接调用 `fread` 等具体的 I/O 函数。相反，它接受一个函数指针 `read_code` 作为参数。调用者必须提供一个符合 `tuz_TInputStream_read` 签名的函数，`tuz_TStream_read_dict_size` 会通过这个回调函数来完成实际的字节读取。这是一种典型的接口与实现分离的设计，增强了代码的通用性和可移植性。

*   **小端序编码 (Little-Endian Encoding)**
    从文件中读取的字节数组需要被转换成一个多字节的整数。该函数遵循**小端序 (Little-Endian)** 的字节序约定。这意味着字节数组中的第一个字节 (`saved[0]`) 是整数的最低有效位字节 (Least Significant Byte, LSB)，第二个字节 (`saved[1]`) 是次低位，以此类推。

*   **编译时优化 (Compile-Time Optimization)**
    代码中使用了 `#if`/`#elif` 预处理指令，而不是运行时的 `switch` 或 `if` 语句。这是因为字典大小占用的字节数 `tuz_kDictSizeSavedBytes` 是一个在编译时就确定的常量。通过使用预处理器，编译器只会将与该常量值匹配的代码块编译进最终的程序中，从而生成更小、更快的机器码。例如，如果 `tuz_kDictSizeSavedBytes` 被定义为 `4`，那么所有其他分支（`==1`, `==2`, `==3`）的代码都不会出现在最终的可执行文件中。

### **代码解析**

```c
/**
 * @brief 从 Tuz 压缩流的开头读取并解析出字典大小。
 * @param inputStream   输入流的句柄。
 * @param read_code     用于从输入流读取数据的函数指针（回调函数）。
 * @return tuz_size_t   成功则返回解析出的字典大小（一个正整数），失败则返回 0。
 */
tuz_size_t tuz_TStream_read_dict_size(tuz_TInputStreamHandle inputStream,tuz_TInputStream_read read_code){
    // `v` 被初始化为要读取的字节数，这个值之后会被 read_code 函数更新为实际读取的字节数。
    tuz_size_t v=tuz_kDictSizeSavedBytes;
    // `saved` 是一个临时缓冲区，用于存放从流中读取的原始字节。
    tuz_byte   saved[tuz_kDictSizeSavedBytes];
    // 断言，确保调用者提供了一个有效的读取函数。
    assert(read_code!=0);
    
    // 调用回调函数读取数据，并检查是否成功读取了预期的字节数。
    if ((read_code(inputStream,saved,&v))&&(v==tuz_kDictSizeSavedBytes)){
        // --- 核心逻辑：根据预设的字节数，将字节数组（小端序）转换为整数 ---
        
        #if (tuz_kDictSizeSavedBytes==1) // 如果字典大小只用1个字节存储
            v=saved[0];
            assert(v>0); // 字典大小必须为正数。
        #elif (tuz_kDictSizeSavedBytes==2) // 如果用2个字节存储
            // saved[0] 是低8位。
            // saved[1] 左移8位后成为高8位，然后通过按位或(|)与 saved[0] 合并。
            v=saved[0]|(((tuz_size_t)saved[1])<<8);
            // 断言检查：v必须为正，且通过右移操作可以还原出原始的 saved[1] 字节。
            assert((v>0)&(((v>>8)&0xFF)==saved[1]));
        #elif (tuz_kDictSizeSavedBytes==3) // 如果用3个字节存储
            v=saved[0]|(((tuz_size_t)saved[1])<<8)|(((tuz_size_t)saved[2])<<16);
            assert((v>0)&(((v>>8)&0xFF)==saved[1])&((v>>16)==saved[2]));
        #elif (tuz_kDictSizeSavedBytes==4) // 如果用4个字节存储
            v=saved[0]|(((tuz_size_t)saved[1])<<8)|(((tuz_size_t)saved[2])<<16)|(((tuz_size_t)saved[3])<<24);
            assert((v>0)&(((v>>8)&0xFF)==saved[1])&(((v>>16)&0xFF)==saved[2])&((v>>24)==saved[3]));
        #else
        #   error unsupport tuz_kDictSizeSavedBytes // 如果配置了不支持的字节数，编译将失败。
        #endif
        
        // 返回成功转换后的字典大小。
        return v;
    }else{ // 如果读取失败或读取的字节数不足
        return 0; // 返回0表示错误。
    }
}
```

## tuz_TStream_decompress_partial **Tuz 流式解压核心状态机**

- 作为流式解压模型的核心，此函数负责执行实际的解压工作。它的设计精髓在于作为一个**可重入 (Re-entrant) 的状态机**，能够根据提供的输出缓冲区大小，解压部分或全部数据，并在暂停后能从断点处精确恢复。

### **原理与设计思路解析**

`tuz_TStream_decompress_partial` 的设计核心是一个基于 `goto` 和状态变量的、高度优化的主循环。它模拟了一个状态机，可以在几个核心状态之间切换：**“从字典复制”**、**“从输入流复制字面量”**（可选编译）、以及**“解析新指令”**。

*   **状态机设计 (State Machine Design)**
    函数的所有内部状态（例如，还需从字典复制多少字节、上一次的匹配距离是多少等）都保存在 `tuz_TStream* self` 结构体中。这使得函数调用是无状态的，两次调用之间所有的上下文都由 `self` 携带。
    *   **状态1: 从字典复制 (`copyDict_cmp_process`)**: 当 `self->_state.dictType_len > 0` 时，函数处于此状态。它的任务就是循环地从环形字典中读取字节，写入输出区，直到 `dictType_len` 减为0或者输出缓冲区 `cur_out_data` 已满。
    *   **状态2: 复制字面量 (`copyLiteral_cmp_process`)**: 仅在 `tuz_isNeedLiteralLine` 宏启用时存在。当 `self->_state.literalType_len > 0` 时，函数处于此状态。它直接从压缩输入流中读取原始字节，写入输出区。
    *   **状态3: 解析新指令 (`type_process`)**: 当没有任何复制任务时，函数进入此状态。这是状态机的决策中心，它会从位流中读取并解析下一个指令。

*   **部分解压 (Partial Decompression) 的实现**
    此函数的最大特点是能够“干多少活，给多少空间”。
    1.  **输出驱动 (Output-Driven):** 在每个状态的循环中，第一件事就是检查输出缓冲区是否已满 (`cur_out_data < out_data_end`)。
    2.  **暂停与恢复:** 一旦输出缓冲区满了，循环就会通过 `break` 语句退出。因为所有的进度（如 `dictType_len` 剩余长度、输入流的当前位置、位流缓冲区的状态等）都完整地保存在 `self` 结构体中，所以当外部调用者清空或更换输出缓冲区并再次调用此函数时，它能无缝地从上次 `break` 的地方继续执行。
    3.  **指令回退 (`_cache_push_1bit`)**: 一个精巧的设计是，当在“解析新指令”状态下准备处理一个字面量，却发现输出缓冲区已满时，它会调用 `_cache_push_1bit` 将刚刚读出的那个决定性的类型比特“推”回位流缓冲区。这确保了下次函数被调用时，能重新读到这个比特，做出和这次完全一样的决策，保证了状态的一致性。

*   **设计优点**
    *   **极低的内存占用:** 无需一次性为整个解压后的文件分配内存，只需要一个固定大小的字典和一个可由用户控制大小的输出缓冲区。
    *   **高度灵活:** 调用者可以根据系统资源情况，决定每次解压多大的数据块，非常适合内存受限的嵌入式系统或处理流式数据。
    *   **高性能:** 使用 `goto` 来直接跳转到对应的处理逻辑块，避免了传统 `switch` 语句或多次 `if-else` 判断可能带来的开销，使得在主循环内的状态切换非常快。

### **代码解析**

```c
/**
 * @brief 执行部分流式解压。
 * @param self           指向 Tuz 解压流状态机的指针。
 * @param cur_out_data   指向输出缓冲区的起始位置。
 * @param data_size      输入时表示缓冲区大小，函数返回时表示实际解压出的数据大小。
 * @return tuz_TResult   返回解压结果，如 tuz_OK, tuz_STREAM_END 等。
 */
tuz_TResult tuz_TStream_decompress_partial(tuz_TStream* self,tuz_byte* cur_out_data,tuz_size_t* data_size){
    // 计算输出缓冲区的末尾指针，用于边界检查。
    tuz_byte* const out_data_end=cur_out_data+(*data_size);
#ifdef __RUN_MEM_SAFE_CHECK
    const tuz_BOOL isNeedOut=(cur_out_data<out_data_end);
#endif
    // 主解码循环，这是一个无限循环，依靠内部的 break 或 return 退出。
    for(;;){
      // --- 状态1: 正在从字典复制数据 ---
      copyDict_cmp_process:
        if (self->_state.dictType_len){ // 检查是否有待复制的字典数据。
            if (cur_out_data<out_data_end){ // 检查输出缓冲区是否已满。
                const tuz_byte bdata=_dict_read_byte(self); // 从环形字典读取一个字节。
                _dict_write_byte(self,bdata);              // 将该字节写回字典（实现滑动窗口）。
                *cur_out_data++=bdata;                     // 将字节写入输出缓冲区。
                self->_state.dictType_len--;               // 待复制长度减一。
                goto copyDict_cmp_process; // 跳转回循环开始，继续复制下一个字节。
            }else{
                break; // 输出缓冲区已满，退出主循环，暂停解压。
            }
        }

  #if tuz_isNeedLiteralLine // 这部分代码仅在特定编译选项下生效。
      // --- 状态2: 正在从输入流复制字面量行 ---
      copyLiteral_cmp_process:
        if (self->_state.literalType_len){
            if (cur_out_data<out_data_end){
                const tuz_byte bdata=_cache_read_1byte(&self->_code_cache); // 直接从输入流缓存读字节。
                _dict_write_byte(self,bdata); // 写入字典。
                *cur_out_data++=bdata;        // 写入输出。
                self->_state.literalType_len--;
                goto copyLiteral_cmp_process; // 继续复制。
            }else{
                break; // 输出缓冲区已满。
            }
        }
  #endif
  
    // --- 状态3: 读取并解析新的类型码 ---
    type_process:
        {
            // 从位流中读取1个比特作为类型标志。
            if (_cache_read_1bit(self)==tuz_codeType_dict){
                // **分支 A: 类型为 字典引用 或 控制码**
                tuz_size_t saved_len=_cache_unpack_len(self); // 解码长度。
                tuz_size_t saved_dict_pos;
                // 检查是否是“使用上一次距离”的优化编码 (1个比特)。
                if ((self->_state.isHaveData_back)&&(_cache_read_1bit(self))){
                    saved_dict_pos=self->_state.dict_pos_back;
                }else{
                    saved_dict_pos=_cache_unpack_dict_pos(self); // 解码完整的距离。
                    if (saved_dict_pos>tuz_kBigPosForLen) ++saved_len; // 一个特殊优化。
                }
                self->_state.isHaveData_back=tuz_FALSE;

                if (saved_dict_pos){ // **这是一个字典匹配 (距离 > 0)**
                    self->_state.dict_pos_back=saved_dict_pos; // 保存本次距离，以备下次使用。
                    self->_state.dictType_len=saved_len+tuz_kMinDictMatchLen; // 设置待复制总长度。
                    // 将解码出的相对距离转换为在环形字典中的绝对偏移。
                    saved_dict_pos=(self->_dict.dict_size-saved_dict_pos);
                    self->_state.dictType_pos=saved_dict_pos;
                    // 跳转到主循环开头，进入“从字典复制”状态。
                    continue; 
                }else{ // **这是一个控制码 (距离 == 0)**
                  #if tuz_isNeedLiteralLine
                    if (tuz_ctrlType_literalLine==saved_len){
                        self->_state.isHaveData_back=tuz_TRUE;
                        self->_state.literalType_len=_cache_unpack_pos_len(self)+tuz_kMinLiteralLen;
                        continue; // 跳转到主循环开头，进入“复制字面量”状态。
                    }
                  #endif

                    self->_state.dict_pos_back=1; // 重置“上一次距离”。
                    self->_state.type_count=0;    // 控制码后是对齐的，重置位流缓冲。
                    if (tuz_ctrlType_clipEnd==saved_len){ // clip 结束标志。
                        goto type_process; // 忽略，直接去解析下一个指令。
                    }else if (tuz_ctrlType_streamEnd==saved_len){ // 流结束标志。
                        // 更新 data_size 为实际输出的字节数。
                        (*data_size)-=(tuz_size_t)(out_data_end-cur_out_data);
                        return tuz_STREAM_END; // 解压成功结束。
                    }else{
                        // 遇到了未知的控制码，返回错误。
                        return _cache_success_finish(&self->_code_cache)?
                                    tuz_CTRLTYPE_UNKNOW_ERROR:tuz_READ_CODE_ERROR;
                    }
                }
            }else{
                // **分支 B: 类型为 字面量**
                if (cur_out_data<out_data_end){ // 检查输出缓冲区是否已满。
                    const tuz_byte bdata=_cache_read_1byte(&self->_code_cache); // 从输入流读一个字节。
                    _dict_write_byte(self,bdata); // 写入字典。
                    *cur_out_data++=bdata;      // 写入输出。
                    self->_state.isHaveData_back=tuz_TRUE;
                    goto type_process; // 继续解析下一个指令。
                }else{
                    // 输出缓冲区已满，将刚读的类型比特“退回”到位流缓冲。
                    _cache_push_1bit(self,tuz_codeType_data);
                    break; // 暂停解压。
                }
            }
        }
    }//end for

// --- 函数退出前的处理 ---
//return_process:
    {
        assert(cur_out_data==out_data_end); // 正常退出循环时，输出缓冲区必然是满的。
        // 如果输入流已经结束，但还有未完成的操作，说明压缩数据有误。
        if (!_cache_success_finish(&self->_code_cache))
            return tuz_READ_CODE_ERROR;

        // 根据编译时安全检查选项返回结果。
        #ifdef __RUN_MEM_SAFE_CHECK
            return isNeedOut?tuz_OK:tuz_OUT_SIZE_OR_CODE_ERROR;
        #else
            return tuz_OK; // 返回OK，表示成功填满了输出缓冲区。
        #endif
    }
}
```
