---
layout: default
title: Dex2oat流程
nav_order: 2
parent: AndroidRuntime
grand_parent: AOSP
author: Anonymous Committer
---
# Dex2oat流程

## dex2oat 主函数

```mermaid
sequenceDiagram
    participant Main as main函数
    participant Dex2oatMain as Dex2oat函数
    participant Dex2OatObj as Dex2Oat 对象实例

    Main ->>Dex2oatMain: 调用 Dex2oat(argc, argv)
    activate Dex2oatMain

    Dex2oatMain->>Dex2OatObj: 创建并初始化Dex2Oat对象
    activate Dex2OatObj

    Dex2OatObj->Dex2OatObj: ParseArgs(argc, argv)解析命令行参数

    Dex2oatMain->>Dex2oatMain: 初始化Memmap数据结构的静态参数 art::MemMap::Init()

    opt HasProfileInput()存在配置文件输入
        Dex2oatMain->Dex2OatObj: LoadProfile()加载配置文件
    end

    Dex2OatObj->>Dex2OatObj: UpdateCompilerOptionsBasedOnProfile()和InsertCompileOptions更新与插入编译器选项

    Dex2OatObj->>Dex2OatObj: OpenFile()打开输出文件

    Dex2oatMain->Dex2OatObj: 创建ScopedDex2oatReporting初始化错误报告机制

    Dex2OatObj->>Dex2OatObj: 执行 dex2oat->Setup()

    opt DoProfileGuidedOptimizations()执行配置文件引导优化
        Dex2OatObj->>Dex2OatObj: VerifyProfileData()验证配置文件数据
    end

    Dex2OatObj->>Dex2OatObj: DoCompilation执行核心编译
   

    Dex2OatObj ->>   Dex2oatMain : 返回编译结果ReturnCode 
    deactivate Dex2OatObj

    Dex2oatMain ->> Main: 返回编译结果
    deactivate Dex2oatMain
```


## Dex2Oat对象成员方法

### ParseArgs 

解析命令行参数。如果遇到无法识别的选项、无效的数值或参数组合，程序将显示用法错误并退出。因此，如果此方法返回，则表示参数已成功解析。

除了解析命令行参数，在这个函数中还会执行Locks::Init()，如果各种locks没有初始化则会执行初始化。

### Profile

LoadInternal函数

1. OpenSource创建/打开文件源 ProfileSource， ProfileSource包含一个fd_和Memmap成员

2. 调用ProfileSource::Read读取文件头数据，包括magic、version、section_count

3. 获取 section_count

4. 调用ProfileSource::Read读取FileSectionInfo， 返回所有 SectionInfo

5. 处理强制的 dex 文件区域
    - 检查第一个 Section 是否为 kDexFiles
    - ReadDexFilesSection

6. 遍历接下来的section
    - 获取 section_info，不同的类型不同处理
    - kExtraDescriptors  ReadExtraDescriptorsSection
    - kClasses 且 merge_classes标志被设置  ReadClassesSection
    - kMethods  ReadMethodsSection
    - kAggregationCounts 忽略此Section，仅服务器端使用
    - default 跳过未知Section，兼容新版本


###  DoCompilation

```mermaid
sequenceDiagram
    participant Caller as Dex2oat函数
    participant Dex2Oat as Dex2Oat 对象
    participant DexFile
    participant CompilerDriver
    participant OptimizingCompiler
    participant Runtime

    Caller->>Dex2Oat: DoCompilation
    activate Dex2Oat

    Dex2Oat -> DexFile: LoadImageClassDescriptors()
    Note right of Dex2Oat:  加载镜像类描述, <br/>如果有Profile，<br/>则只加载DexFile和<br/>ProfileCompilationInfo的交集
    DexFile -> DexFile: GetClassDescriptor

    Dex2Oat ->> CompilerDriver : Compile()
    CompilerDriver -> CompilerDriver : PrepareDexFilesForOatFile 
    CompilerDriver ->> OptimizingCompiler:  CreateOptimizingCompiler

    activate OptimizingCompiler
    Note over OptimizingCompiler: CompileDexFiles方法

    OptimizingCompiler  ->    Runtime : Runtime::Current()->GetClassLinker()
    opt 非bootimage且非bootImageExtension
        OptimizingCompiler -> ClassLoaderContext : CreateClassLoader
    end 
    OptimizingCompiler -> OptimizingCompiler : InitializeThreadPools();
    OptimizingCompiler -> OptimizingCompiler : PreCompile
    OptimizingCompiler -> OptimizingCompiler : CompileAll(class_loader, dex_files, timings_);
    OptimizingCompiler -> OptimizingCompiler : FreeThreadPools();
    OptimizingCompiler ->> Dex2Oat : 返回 ClassLoader 对象
    deactivate OptimizingCompiler

    Dex2Oat->>Dex2Oat: WriteOutputFiles()
    Note right of Dex2Oat: 将编译结果写入输出文件


    Dex2Oat->>Dex2Oat: FlushOutputFiles()
    Note right of Dex2Oat: 刷新输出文件，保持打开, 后续可能会剥离符号

   
    opt IsImage()
    Note right of Dex2Oat: HandleImage() <br/> 创建 boot.art 并patch oat
    Dex2Oat->>Dex2Oat: CreateImageFile()
    end

    alt IsHost() 不剥离符号快速返回
        Dex2Oat->>Dex2Oat: FlushCloseOutputFiles()
        Dex2Oat->>Dex2Oat: DumpTiming()
        Dex2Oat->>Caller: 返回 kNoFailure
    else
        Dex2Oat->>Dex2Oat: CopyOatFilesToSymbolsDirectoryAndStrip()
        Note right of Dex2Oat: 复制剥离符号的版本到未剥离符号的位置

        Dex2Oat->>Dex2Oat: FlushCloseOutputFiles()
        Note right of Dex2Oat: 刷新关闭文件
        
        Dex2Oat->>Dex2Oat: DumpTiming()
    end
    Dex2Oat->>Caller: 返回 kNoFailure
    deactivate Dex2Oat
```


#### WriteOutputFiles函数

```mermaid
sequenceDiagram
    participant Dex2Oat
    participant MemMap
    Dex2Oat -> MemMap : 遍历每一个opened_dex_files_maps_

    MemMap ->> MemMap:  Sync() 同步 Dex2Dex 转换后的数据
    opt IsImage()
        Dex2Oat ->> Dex2Oat: 如果不是boot image 直接设置 image_base_
        Dex2Oat ->> ImageWriter: 创建新的 ImageWriter 实例
        ImageWriter ->> ImageWriter: PrepareImageAddressSpace() <br/> 为linker patches准备
    end

    Dex2Oat ->> OatWriter: Initialize(driver, verification_results, image_writer, dex_files)
    Note over Dex2Oat: 对于多个Oat文件传入compiler driver, <br/> image writer, dex files初始化OatWriter
    opt !use_existing_vdex_
        OatWriter->>OatWriter: FinishVdexFile(vdex_file, verifier_deps)
        Note over OatWriter: 对于多个Oat/ELF文件生成并写入 VDEX 文件
    end

    loop 循环处理oat_files_ 

        Dex2Oat -> OatWriter : PrepareLayout准备Oat文件布局 

        Dex2Oat -> ElfWriter: PrepareDynamicSection准备ELF动态节信息
       
        opt IsImage()
            Dex2Oat ->> ImageWriter: UpdateOatFileLayout(...)更新ImageWriter的Oat文件布局信息
        end
    end


    loop 循环处理oat_files_ 
        Dex2Oat->>OatWriter: GetDebugInfo()
        Dex2Oat->>ElfWriter: PrepareDebugInfo(debug_info)
        Note over ElfWriter: 准备压缩的DebugInfo

        Dex2Oat->>OatWriter: WriteRodata(rodata_stream)
        Dex2Oat->>ElfWriter: EndRoData(rodata_stream)
        Note over ElfWriter: 写入只读数据段 (.rodata)

        Dex2Oat->>ElfWriter: StartText()
        Dex2Oat->>OatWriter: WriteCode(text_stream)
        Dex2Oat->>ElfWriter: EndText(text_stream)
        Note over ElfWriter: 写入代码段 (.text)

        opt DataImgRelRoSize()
            Dex2Oat->>ElfWriter: StartDataImgRelRo()
            Dex2Oat->>OatWriter: WriteDataImgRelRo(data_img_rel_ro_stream)
            Dex2Oat->>ElfWriter: EndDataImgRelRo(data_img_rel_ro_stream)
            Note over ElfWriter: 写入数据镜像相关只读段 (.data.img.rel.ro)
        end
   
        Dex2Oat->>OatWriter: WriteHeader(elf_writer->GetStream())
        Note over OatWriter: 写入Oat Header

        opt IsImage()
            Dex2Oat->>ImageWriter: UpdateOatFileHeader(...)
            Note over ImageWriter: 更新ImageWriter的Oat文件头信息
        end

        Dex2Oat->>ElfWriter: WriteDynamicSection()
        Note over ElfWriter: 写入ELF动态节

        Dex2Oat->>ElfWriter: WriteDebugInfo(oat_writer->GetDebugInfo())
        Note over ElfWriter: 写入压缩后的调试信息 (等待压缩任务完成)

        Dex2Oat->>ElfWriter: End()
        Note over ElfWriter: 结束ELF文件写入

        Dex2Oat->>File: FlushOutputFile(&vdex_files_[i])
        Dex2Oat->>File: FlushOutputFile(&oat_files_[i])
        Note over File: 刷新VDEX和OAT文件

        OatWriter->>Dex2Oat: Destroy OatWriter
    end

    Dex2Oat-->>Dex2Oat: 返回 true


```