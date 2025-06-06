---
layout: default
title: LLVM IR相关的类及一些用法
nav_order: 1
parent: 编译
author: Anonymous Committer
---
# LLVM IR相关的类及一些用法

## 模块（llvm::Module）

**`llvm::Module`** 表示一个完整的 LLVM IR 单位（对应一个 `.ll` 或 `.bc` 文件），包含函数、全局变量、别名、符号表等信息。

### 创建与加载

```c++
#include "llvm/IR/Module.h"
#include "llvm/IR/LLVMContext.h"
#include "llvm/IR/IRReader/IRReader.h"
#include "llvm/Support/SourceMgr.h"
#include "llvm/Support/MemoryBuffer.h"

// 从内存缓冲区或文件读取
llvm::LLVMContext Context;
llvm::SMDiagnostic Err;
std::unique_ptr<llvm::Module> M = llvm::parseIRFile("input.ll", Err, Context);
if (!M) {
    Err.print("MyTool", llvm::errs());
    return nullptr;
}
// 或者手动创建空模块
auto ModulePtr = std::make_unique<llvm::Module>("MyModule", Context);
```
主要方法
- 获取模块名称
```cpp
StringRef moduleName = M->getName();  // 返回模块名称
```

### 插入/删除全局变量、函数
```cpp
// 添加全局变量
llvm::Type *Int32Ty = llvm::Type::getInt32Ty(Context);
auto *GV = new llvm::GlobalVariable(
    *M,                            // 所属模块
    Int32Ty,                       // 元素类型
    false,                         // 是否常量
    llvm::GlobalValue::ExternalLinkage,  // 链接类型
    llvm::ConstantInt::get(Int32Ty, 0),  // 初始值
    "myGlobalVar"                  // 名称
);

// 删除某个全局
GV->eraseFromParent();
```

### 遍历函数与全局
```cpp
for (auto &Func : *M) {
    llvm::Function &F = Func;
    // 处理函数 F
}

for (auto &G : M->globals()) {
    llvm::GlobalVariable &Global = G;
    // 处理全局变量 Global
}

```
### 验证模块
```cpp
#include "llvm/IR/Verifier.h"
if (llvm::verifyModule(*M, &llvm::errs())) {
    llvm::errs() << "模块验证失败！\n";
}
```

### 打印 IR
```cpp
M->print(llvm::outs(), nullptr);
```

## 上下文（llvm::LLVMContext）

llvm::LLVMContext 代表一个 LLVM “上下文”，用于存储全局的**单例数据**，如常量唯一化表、类型唯一化表等。通常一个应用只使用一个 LLVMContext，但多线程或隔离场景下可创建多个上下文。

常用场景
- 创建类型
```cpp
llvm::IntegerType *Int32Ty = llvm::Type::getInt32Ty(Context);
llvm::PointerType *PtrToInt32 = llvm::Type::getInt32PtrTy(Context, 0);
```
- 获取函数的上下文
```cpp
llvm::Function *F = /* 某个函数 */;
llvm::LLVMContext &Ctx = F->getContext();
```
- 创建 IRBuilder
```cpp
llvm::IRBuilder<> Builder(Context);
```

## 类型（llvm::Type 及其子类）

LLVM 中所有的类型都派生自 llvm::Type，常见子类包括 IntegerType、PointerType、ArrayType、VectorType、FunctionType、StructType 等。

### 基础类型
- Void 类型
```cpp
llvm::Type *VoidTy = llvm::Type::getVoidTy(Context);
```

- 标签类型（Label，用于基本块标签）
```cpp
llvm::Type *LabelTy = llvm::Type::getLabelTy(Context);
```

- 整数类型
```cpp
llvm::IntegerType *Int1Ty   = llvm::Type::getInt1Ty(Context);
llvm::IntegerType *Int8Ty   = llvm::Type::getInt8Ty(Context);
llvm::IntegerType *Int16Ty  = llvm::Type::getInt16Ty(Context);
llvm::IntegerType *Int32Ty  = llvm::Type::getInt32Ty(Context);
llvm::IntegerType *Int64Ty  = llvm::Type::getInt64Ty(Context);
llvm::IntegerType *Int128Ty = llvm::Type::getInt128Ty(Context);
```

- 浮点数类型
```cpp
llvm::Type *HalfTy   = llvm::Type::getHalfTy(Context);   // 16-bit IEEE 半精度
llvm::Type *BFloatTy = llvm::Type::getBFloatTy(Context); // BFloat16
llvm::Type *FloatTy  = llvm::Type::getFloatTy(Context);  // 32-bit
llvm::Type *DoubleTy = llvm::Type::getDoubleTy(Context); // 64-bit
llvm::Type *FP128Ty  = llvm::Type::getFP128Ty(Context);  // 128-bit
```


### 指针类型

用于表示指向某类型的指针，允许指定地址空间（Address Space）。
```cpp
llvm::PointerType *Int32PtrTy = llvm::Type::getInt32PtrTy(Context, 0);    // 默认地址空间0
llvm::PointerType *FloatPtrTy = llvm::Type::getFloatPtrTy(Context, 0);
llvm::PointerType *DoublePtrTy = llvm::Type::getDoublePtrTy(Context, 1);  // 地址空间1
```
也可以通过更通用的接口：
```cpp
llvm::PointerType *PtrToTy = llvm::PointerType::get(SomeType, AddrSpace);
```
### 数组与向量类型
- 数组类型
```cpp
// 创建一个包含 10 个 i32 元素的数组类型 [10 x i32]
llvm::ArrayType *Array10xI32 = llvm::ArrayType::get(Int32Ty, 10);
```

- 向量类型（SIMD）
```cpp
// 创建一个包含 4 个 float 的向量类型 <4 x float>
llvm::VectorType *Vec4f = llvm::VectorType::get(FloatTy, 4, /*isScalable=*/false);
```


Scalable 向量可用于 Arm SVE 等架构，若为不可伸缩则设置 isScalable=false。

### 函数类型

使用 llvm::FunctionType::get 创建函数签名（返回类型 + 参数列表 + 可变参数标志）：
```cpp
llvm::Type *RetTy = Int32Ty;
std::vector<llvm::Type *> Params = { Int32Ty, Int32Ty };
bool IsVarArg = false;

// 创建 i32 foo(i32, i32) 函数类型
llvm::FunctionType *FuncTy = llvm::FunctionType::get(RetTy, Params, IsVarArg);
```
如果要创建可变参数函数，例如 printf：
```cpp
llvm::FunctionType *PrintfTy = llvm::FunctionType::get(
    Int32Ty,                // 返回 int
    { llvm::Type::getInt8PtrTy(Context) }, // 第一个参数：i8*
    true                    // 可变参数
);
```
### 结构体类型
- 匿名结构体
```cpp
// 创建一个匿名结构体 { i32, float, i8* }
std::vector<llvm::Type *> Elements = { Int32Ty, FloatTy, llvm::Type::getInt8PtrTy(Context) };
llvm::StructType *AnonStruct = llvm::StructType::get(Context, Elements, /*isPacked=*/false);
```

- 命名结构体
```cpp
llvm::StructType *NamedStruct = llvm::StructType::create(Context, "MyStruct");
// 以后再设置元素类型
NamedStruct->setBody(Elements, /*isPacked=*/false);
```

## 值（llvm::Value 及子类）

llvm::Value 是 LLVM IR 的根基类，表示 IR 中的“值”（包括常量、参数、指令、函数、基本块等均直接或间接继承自 Value）。

### 获取类型与名称
- 获取类型
```cpp
llvm::Value *V = /* 某个 Value */;
llvm::Type *Ty = V->getType();
```

- 获取/设置名称
```cpp
llvm::StringRef Name = V->getName();
V->setName("new_name");
```


### 遍历使用者（Uses）

每个 Value 都维护了一个“使用链表”（Use List），记录哪些 User（例如指令、ConstantExpr 等）在使用该值。可通过迭代器遍历：
```cpp
for (auto It = V->use_begin(), End = V->use_end(); It != End; ++It) {
    llvm::Use &UseRef = *It;
    llvm::User *Usr = UseRef.getUser();
    // Usr 是某个使用了 V 的 User（通常是 Instruction 或 ConstantExpr）
}
```
- 获取使用总数
```cpp
unsigned NumUses = V->getNumUses();
```

- 替换所有使用
```cpp
llvm::Value *OldV = /* 旧值 */;
llvm::Value *NewV = /* 新值 */;
OldV->replaceAllUsesWith(NewV);
```

## 全局变量与别名（llvm::GlobalVariable、llvm::GlobalAlias）

### llvm::GlobalVariable

表示模块范围的全局变量，有初始值、链接类型、地址空间、对齐等属性。
```cpp
#include "llvm/IR/GlobalVariable.h"

// 创建一个 i32 全局变量，初始值为 42
llvm::GlobalVariable *GV = new llvm::GlobalVariable(
    *ModulePtr,                      // 所属模块
    Int32Ty,                         // 元素类型
    false,                           // 是否 const
    llvm::GlobalValue::ExternalLinkage, // 链接类型
    llvm::ConstantInt::get(Int32Ty, 42), // 初始值
    "gVar"                           // 名称
);

// 设置对齐
GV->setAlignment(llvm::MaybeAlign(4));

// 访问初始值
llvm::Constant *Init = GV->getInitializer();
```
- 常见方法
```cpp
bool isConstant = GV->isConstant();
GV->setConstant(true);                // 将其设为常量
llvm::Type *ElemTy = GV->getValueType(); // 全局变量的元素类型
unsigned AddrSpace = GV->getAddressSpace();
```

- 删除全局变量
```cpp
GV->eraseFromParent();
```


### llvm::GlobalAlias

用于创建全局别名（例如将某个全局符号重定向到另一个符号）。常见于编译器实现底层库函数重命名。
```cpp
#include "llvm/IR/GlobalAlias.h"

// 假设有一个已有的全局函数 F
llvm::Function *F = /* 某个 llvm::Function* */;

// 创建一个别名 aliasName，指向函数 F
llvm::GlobalAlias *GA = llvm::GlobalAlias::create(
    F->getFunctionType(),           // 别名的函数类型必须与原函数一致
    F->getFunctionType()->getPointerTo(F->getAddressSpace()), // 别名的类型
    llvm::GlobalValue::ExternalLinkage, // 链接类型
    "aliasName",                    // 别名名称
    F,                              // 被别名的目标
    ModulePtr.get()                 // 所属模块
);

// 删除别名
GA->eraseFromParent();
```

## 函数（llvm::Function）

llvm::Function 表示模块中的一个函数，由若干基本块及其指令构成。继承自 GlobalValue 和 Value。

创建函数
```cpp
#include "llvm/IR/Function.h"
#include "llvm/IR/DerivedTypes.h"

// 准备函数类型
llvm::FunctionType *FuncTy = llvm::FunctionType::get(
    /*Result=*/Int32Ty,
    /*Params=*/{Int32Ty, Int32Ty},
    /*IsVarArg=*/false
);

// 创建函数，放入 Module
llvm::Function *F = llvm::Function::Create(
    FuncTy,
    llvm::GlobalValue::ExternalLinkage, // 链接类型
    "addTwoInts",                       // 函数名
    ModulePtr.get()                     // 所属模块
);

// 设置函数参数名称
unsigned Idx = 0;
for (auto &Arg : F->args()) {
    if (Idx == 0) Arg.setName("a");
    else if (Idx == 1) Arg.setName("b");
    ++Idx;
}
```
### 函数属性与调用约定

- 调用约定
```cpp
F->setCallingConv(llvm::CallingConv::C);      // C 调用约定
F->setCallingConv(llvm::CallingConv::Fast);   // fastcall
```

- 添加函数属性
```cpp
// 在函数返回值上添加属性
F->addFnAttr(llvm::Attribute::NoUnwind);
F->addFnAttr(llvm::Attribute::AlwaysInline);
```

- 查询属性
```cpp
bool isInline = F->hasFnAttribute(llvm::Attribute::AlwaysInline);
```


### 遍历基本块与参数
- 遍历基本块
```cpp
for (llvm::BasicBlock &BB : *F) {
    // 处理基本块 BB
}
```
- 或者使用迭代器：
```cpp
for (auto It = F->begin(), End = F->end(); It != End; ++It) {
    llvm::BasicBlock &BB = *It;
}
```
- 获取基本块数量
```cpp
size_t NumBB = F->size();
```

- 遍历参数
```cpp
for (llvm::Argument &Arg : F->args()) {
    // Arg.getName(), Arg.getType() 等
}
```

- 获取所属模块与上下文
```cpp
llvm::Module *ParentMod = F->getParent();
llvm::LLVMContext &Ctx = F->getContext();
```

## 基本块（llvm::BasicBlock）

llvm::BasicBlock 表示函数体内的一段指令序列，以单一入口且无分支跳入。基本块包含若干指令，最后通常以终止指令（如 ret、br、switch）结尾。

### 创建与插入
```cpp
#include "llvm/IR/BasicBlock.h"

// 在函数末尾创建新的基本块
llvm::BasicBlock *BB = llvm::BasicBlock::Create(
    Context,    // LLVMContext&
    "entry",    // 基本块名称（可选）
    F           // 所属函数
);

// 在某个已有基本块之前插入
llvm::BasicBlock *InsertBefore = /* 某个 llvm::BasicBlock* */;
llvm::BasicBlock *BB2 = llvm::BasicBlock::Create(
    Context,
    "middle",
    F,
    InsertBefore
);
```
### 前驱与后继

要遍历基本块的前驱（pred）和后继（succ），需包含头文件 llvm/IR/CFG.h。
- 遍历前驱
```cpp
for (auto PI = llvm::pred_begin(BB), PE = llvm::pred_end(BB); PI != PE; ++PI) {
    llvm::BasicBlock *PredBB = *PI;
    // 处理前驱 PredBB
}
```

- 遍历后继
```cpp
for (auto SI = llvm::succ_begin(BB), SE = llvm::succ_end(BB); SI != SE; ++SI) {
    llvm::BasicBlock *SuccBB = *SI;
    // 处理后继 SuccBB
}
```

- 获取前驱/后继数量
```cpp
unsigned NumPred = llvm::pred_size(BB);
unsigned NumSucc = llvm::succ_size(BB);
```


### 在函数间移动
- 从原函数移除
```cpp
BB->removeFromParent();
```

- 插入到新函数
```cpp
// 假设 NewF 是一个新的 llvm::Function*
BB->insertInto(NewF, /*InsertBefore=*/nullptr); // 插入到 NewF 的末尾
```

- 获取同一函数中前后相邻基本块
```cpp
llvm::BasicBlock *Prev = BB->getPrevNode();  // 若 BB 是第一个基本块，返回 nullptr
llvm::BasicBlock *Next = BB->getNextNode(); // 若 BB 是最后一个基本块，返回 nullptr
```

## 常量（llvm::Constant 及子类）

llvm::Constant 是 Value 的子类，表示 LLVM IR 中的常量，如整型、浮点、全局地址、常量表达式等。

### 整型常量与浮点常量
- 整型常量
```cpp
// 32 位有符号整数常量 123
llvm::ConstantInt *CI32 = llvm::ConstantInt::get(Int32Ty, 123, /*IsSigned=*/true);

// 布尔常量
llvm::ConstantInt *TrueVal = llvm::ConstantInt::getBool(Context, true);
```
- 浮点常量
```cpp
llvm::ConstantFP *FPVal = llvm::ConstantFP::get(Context, llvm::APFloat(3.14f));
```

- 零初始化常量
```cpp
llvm::Constant *ZeroInt = llvm::ConstantInt::get(Int32Ty, 0);
llvm::Constant *NullPtr = llvm::ConstantPointerNull::get(llvm::Type::getInt8PtrTy(Context));
llvm::Constant *ZeroArray = llvm::ConstantAggregateZero::get(Array10xI32);
```


### 常量表达式（ConstantExpr）

ConstantExpr 用于在 IR 常量环境下构造各种操作，而无需指令上下文。例如：
```cpp
#include "llvm/IR/Constants.h"

// 创建一个对指针 varPtr 执行 getelementptr 的常量表达式
llvm::Constant *GEPConst = llvm::ConstantExpr::getGetElementPtr(
    /*PointeeType=*/SomeContainerType,
    /*Ptr=*/SomeGlobalVar,
    /*Indices=*/{ llvm::ConstantInt::get(Int32Ty, 0), llvm::ConstantInt::get(Int32Ty, 2) }
);

// 常量二元运算
llvm::Constant *SumConst = llvm::ConstantExpr::getAdd(
    llvm::ConstantInt::get(Int32Ty, 5),
    llvm::ConstantInt::get(Int32Ty, 7)
);
```

## 指令（llvm::Instruction 及子类）

llvm::Instruction 继承自 User，表示一条 IR 指令。它也是 Value 的子类，因此可作为其他指令的操作数。所有具体的 IR 指令（如 add、load、call、br 等）都派生自 Instruction。

### 操作数访问与修改
- 获取操作数数目
```cpp
llvm::Instruction *I = /* 某条指令 */;
unsigned NumOps = I->getNumOperands();
```
- 访问/设置操作数
```cpp
llvm::Value *Op0 = I->getOperand(0);
I->setOperand(1, NewValue);
```
- 遍历所有操作数
```cpp
for (unsigned i = 0; i < I->getNumOperands(); ++i) {
    llvm::Value *Op = I->getOperand(i);
    // 处理操作数 Op
}
```

### 删除与替换
- 将指令从基本块中移除并返回下一个迭代器
```cpp
auto NextIt = I->eraseFromParent();
// 注意：eraseFromParent 会销毁指令 I 并返回下一个指令的迭代器
```
- 用另一个值替换
```cpp
I->replaceAllUsesWith(NewValue);  // 将 I 的所有使用替换为 NewValue，但 I 本身仍在基本块中，需手动删除
I->eraseFromParent();             // 删除指令 I
```
### 常见指令示例

以下列出部分常见指令的创建与用法示例。

二元运算（llvm::BinaryOperator）
```cpp
#include "llvm/IR/Instructions.h"

// 在基本块 BB 的末尾插入一条 add 指令：%sum = add i32 %lhs, %rhs
llvm::Value *LHS = /* 某个 Value* */;
llvm::Value *RHS = /* 某个 Value* */;
llvm::BinaryOperator *AddInst = llvm::BinaryOperator::Create(
    llvm::Instruction::Add, // Op 类型
    LHS,                    // 左操作数
    RHS,                    // 右操作数
    "sum",                  // 指令名称（可选）
    BB                      // 插入位置：基本块末尾
);
// 其他算数运算：Sub, Mul, UDiv, SDiv, SRem, FRem, ...
```

比较指令（llvm::CmpInst）
整数比较（ICmp）
```cpp
#include "llvm/IR/Instructions.h"

// 创建一条整数比较：%cmp = icmp sgt i32 %a, %b
llvm::CmpInst *ICmpInst = llvm::CmpInst::Create(
llvm::Instruction::ICmp,         // Op 类型
llvm::CmpInst::ICMP_SGT,         // Predicate
LHS,                             // 操作数1
RHS,                             // 操作数2
"cmp",                           // 名称
BB                               // 插入位置
);
```
浮点比较（FCmp）  
```cpp
// %fcmp = fcmp olt double %x, %y
llvm::CmpInst *FCmpInst = llvm::CmpInst::Create(
    llvm::Instruction::FCmp,
    llvm::CmpInst::FCMP_OLT,
    X,
    Y,
    "fcmp",
    BB
);
```
内存操作（alloca、load、store）
alloca 指令（在函数的入口块或栈帧中分配内存）

```cpp
#include "llvm/IR/Instructions.h"


llvm::AllocaInst *AllocaInst = new llvm::AllocaInst(
Int32Ty,         // 分配的类型
0,               // 地址空间（默认为0）
"stackVar",      // 名称
/*插入位置*/ &F->getEntryBlock()  // 一般插入到函数入口块
);

// 分配数组：%arr = alloca [10 x i32]
llvm::Value *ArraySize = llvm::ConstantInt::get(Int32Ty, 10);
llvm::AllocaInst *AllocaArr = new llvm::AllocaInst(
Int32Ty,          // 数组元素类型
0,                // 地址空间
ArraySize,        // 数组大小（元素数量）
"arr",
&F->getEntryBlock()
);
```

getelementptr 指令  
```cpp
#include "llvm/IR/Instructions.h"

// 对指针 ptr 进行 GEP 运算：获取 &ptr[idx]
llvm::Value *IdxConst = llvm::ConstantInt::get(Int32Ty, 5);
llvm::GetElementPtrInst *GEPInst = llvm::GetElementPtrInst::Create(
  /*PointeeType=*/Int32Ty,
  /*Ptr=*/AllocaInst,
  /*IdxList=*/{ IdxConst },
  /*Name=*/"gep",
  /*BB=*/BB
);
```
load 指令
```cpp
llvm::LoadInst *LoadInst = new llvm::LoadInst(
    Int32Ty,         // 加载的类型
    AllocaInst,      // 指向的地址
    "ld",
    BB
);
```
store 指令
```cpp
llvm::StoreInst *StoreInst = new llvm::StoreInst(
    /*Val=*/llvm::ConstantInt::get(Int32Ty, 123),
    /*Ptr=*/AllocaInst,
    /*BB=*/BB
);
```


### PHI 节点（llvm::PHINode）

PHINode 必须出现在基本块的最顶端，用于合并来自不同前驱基本块的值。
```cpp
#include "llvm/IR/Instructions.h"

// 假设 BB 有两个前驱：Pred1 和 Pred2，且它们分别生成值 V1 和 V2
llvm::PHINode *Phi = llvm::PHINode::Create(
    Int32Ty,    // 类型
    2,          // 预留两个 incoming
    "phi",
    BB          // 插入位置：BB 的开头
);

// 设置 incoming 列表
Phi->addIncoming(V1, Pred1);
Phi->addIncoming(V2, Pred2);

// 或者后续动态添加
// Phi->addIncoming(NewV, NewPredBB);
```
访问/修改已有的 incoming
```cpp
unsigned NumIncoming = Phi->getNumIncomingValues();
llvm::Value *IncVal = Phi->getIncomingValue(0);
llvm::BasicBlock *IncBB = Phi->getIncomingBlock(0);

Phi->setIncomingValue(1, SomeValue);
Phi->setIncomingBlock(1, SomeBasicBlock);
```
注意
PHI 节点必须集中出现在基本块顶端。如果在基本块中间插入 PHI，将报错 “PHI nodes not grouped at top of basic block!”。

函数调用（llvm::CallInst）
```cpp
#include "llvm/IR/Instructions.h"

// 假设 f 是一个 llvm::Function*，接受两个 i32，返回 i32
llvm::Function *f = /* 已有函数 */;
llvm::Value *Arg0 = llvm::ConstantInt::get(Int32Ty, 10);
llvm::Value *Arg1 = llvm::ConstantInt::get(Int32Ty, 20);

llvm::CallInst *CallI = llvm::CallInst::Create(
    f->getFunctionType(),
    f,                   // 函数指针
    { Arg0, Arg1 },      // 参数列表
    "call_res",          // 名称
    BB                   // 插入位置
);

// 设置调用约定（可选）
CallI->setCallingConv(llvm::CallingConv::C);
```
在 LLVM 11 及更高版本中，可直接使用静态 Create() 重载：
```cpp
llvm::CallInst *CallI2 = llvm::CallInst::Create(
    /*Callee=*/f,
    /*Args=*/{ Arg0, Arg1 },
    /*Name=*/"callRes2",
    /*InsertAtEnd=*/BB
);
```


### 分支指令（llvm::BranchInst）
- 无条件分支
```cpp
// 构造一条跳转到 IfTrueBB 的无条件分支
llvm::BranchInst *BrUncond = llvm::BranchInst::Create(
    /*IfTrue=*/IfTrueBB,
    /*InsertAtEnd=*/BB
);
```

- 有条件分支
```cpp
// cond 是 i1 类型的条件值
llvm::Value *Cond = /* 某个 i1 值 */;
llvm::BranchInst *BrCond = llvm::BranchInst::Create(
    /*IfTrue=*/ThenBB,
    /*IfFalse=*/ElseBB,
    /*Cond=*/Cond,
    /*InsertAtEnd=*/BB
);
```

- 修改目标
```cpp
BrCond->setSuccessor(0, NewThenBB);
BrCond->setSuccessor(1, NewElseBB);
```


### Switch 指令（llvm::SwitchInst）
```cpp
#include "llvm/IR/Instructions.h"

// value 是 i32 或其他整数类型的值，defaultBB 是默认跳转目标
llvm::Value *Value = /* i32 值 */;
llvm::BasicBlock *DefaultBB = /* BasicBlock* */;
llvm::SwitchInst *SwitchI = llvm::SwitchInst::Create(
    /*Value=*/Value,
    /*Default=*/DefaultBB,
    /*NumCases=*/3,   // 预留3个 case
    /*InsertAtEnd=*/BB
);

// 添加 case: 当 Value 等于 10 时跳转到 CaseBB1
auto *CaseVal1 = llvm::ConstantInt::get(Int32Ty, 10);
SwitchI->addCase(CaseVal1, CaseBB1);

// 遍历所有 case（包括 default）
for (auto CI = SwitchI->case_begin(), CE = SwitchI->case_end(); CI != CE; ++CI) {
    llvm::ConstantInt *OnVal = CI->getCaseValue();
    llvm::BasicBlock *DestBB = CI->getCaseSuccessor();
    // 处理 case
}

// 获取 default case
auto *DefaultCI = SwitchI->getDefaultDest();  // 切换到默认目标
```
### 返回指令（llvm::ReturnInst）
```cpp
#include "llvm/IR/Instructions.h"

// 返回 void
llvm::ReturnInst *RetVoid = llvm::ReturnInst::Create(
    Context,
    /*RetVal=*/nullptr,
    /*InsertAtEnd=*/BB
);

// 返回 int 值
llvm::Value *RetVal = llvm::ConstantInt::get(Int32Ty, 0);
llvm::ReturnInst *RetValInst = llvm::ReturnInst::Create(
    Context,
    RetVal,
    BB
);
```

### 其他常见指令（UnaryOps、CastOps 等）
- 位移与逻辑运算：Shl、LShr、AShr、And、Or、Xor 均由 BinaryOperator::Create 创建。
- 一元运算：Neg 等通过相应静态方法或折叠成 BinaryOperator。
- Cast 指令
```cpp
llvm::Value *IntVal = /* i32 值 */;
// zext i32 -> i64
llvm::Value *ZExt = llvm::CastInst::CreateZExtOrBitCast(
    IntVal,
    Int64Ty,
    "zext",
    BB
);

// trunc i64 -> i32
llvm::Value *Trunc = llvm::CastInst::CreateTruncOrBitCast(
    /*Val=*/SomeInt64Val,
    /*DestTy=*/Int32Ty,
    /*Name=*/"trunc",
    /*InsertAtEnd=*/BB
);
```

- GetElementPtr（GEP）：参见上文。
- Select 指令
```cpp
// %res = select i1 %cond, i32 %val1, i32 %val2
llvm::Value *Cond = /* i1 */;
llvm::Value *ValTrue = /* i32 */;
llvm::Value *ValFalse = /* i32 */;
llvm::SelectInst *SelectI = llvm::SelectInst::Create(
    Cond,
    ValTrue,
    ValFalse,
    "selectRes",
    BB
);

```


## IRBuilder（llvm::IRBuilder<>）

llvm::IRBuilder<> 是构建 LLVM IR 指令的工具类，封装了常见指令的创建，并自动管理插入位置。使用时只需指定当前插入点，调用对应方法即可。

```cpp
#include "llvm/IR/IRBuilder.h"

llvm::IRBuilder<> Builder(Context);

// 设置插入点到基本块 BB 的末尾
Builder.SetInsertPoint(BB);

// 生成二元运算：%sum = add i32 %a, %b
llvm::Value *A = /* i32 */;
llvm::Value *B = /* i32 */;
llvm::Value *Sum = Builder.CreateAdd(A, B, "sum");

// 生成 Alloca
llvm::AllocaInst *Alloca = Builder.CreateAlloca(Int32Ty, nullptr, "localVar");

// 生成 Store
Builder.CreateStore(llvm::ConstantInt::get(Int32Ty, 42), Alloca);

// 生成 Load
llvm::Value *LoadVal = Builder.CreateLoad(Int32Ty, Alloca, "ld");

// 生成 Branch
Builder.CreateCondBr(Builder.CreateICmpSGT(LoadVal, llvm::ConstantInt::get(Int32Ty, 0)), ThenBB, ElseBB);

// 生成 Return
Builder.CreateRet(llvm::ConstantInt::get(Int32Ty, 0));
```
常用方法（仅列举部分）：
- CreateAdd, CreateSub, CreateMul, CreateUDiv, CreateSDiv, CreateFAdd 等算数运算
- CreateICmpEQ, CreateICmpSGT, CreateFCmpOLT 等比较
- CreateAlloca, CreateLoad, CreateStore 等内存操作
- CreateBr, CreateCondBr, CreateSwitch 等分支
- CreatePHI, CreateCall, CreateBitCast, CreateTrunc, CreateZExt 等
- CreateGetElementPtr

IRBuilder 会自动推断类型并创建合适的 Constant，例如：
```cpp
auto *Const5 = Builder.getInt32(5);
```

## 元数据与调试信息（Metadata、llvm::DIBuilder）

LLVM IR 支持附加 元数据（Metadata），可用于调试信息、目标属性、循环信息等。常见调试信息通过 llvm::DIBuilder 构造。

### 普通元数据
- 附加到指令
```cpp
// 假设 I 是一条 Instruction*
llvm::LLVMContext &Ctx = I->getContext();
llvm::MDNode *MD = llvm::MDNode::get(
    Ctx,
    { llvm::MDString::get(Ctx, "my_meta"), llvm::ConstantAsMetadata::get(SomeConst) }
);
I->setMetadata("my.metadata", MD);
```

- 获取元数据
```cpp
if (auto *MD = I->getMetadata("my.metadata")) {
    // 处理 MDNode*
}
```


### 调试信息（DIBuilder）

使用 llvm::DIBuilder 在 IR 中添加调试符号，用于生成 DWARF 信息，便于在调试器中单步调试。
```cpp
#include "llvm/IR/DIBuilder.h"

llvm::DIBuilder DIB(*ModulePtr);

// 创建编译单元
auto *CU = DIB.createCompileUnit(
    llvm::dwarf::DW_LANG_C_plus_plus,   // 语言
    DIB.createFile("example.cpp", "/path/to"), // 源文件
    "MyCompiler",                       // 编译器生产者
    false,                              // 是否优化
    "",                                 // 编译选项
    0                                   // 运行时版本
);

// 创建文件与目录
auto *File = DIB.createFile("example.cpp", "/path/to");

// 创建调试类型信息（例如：int）
auto *DIBasicInt = DIB.createBasicType("int", 32, llvm::dwarf::DW_ATE_signed);

// 在函数前创建子程序类型
auto *SPTy = DIB.createSubroutineType(DIB.getOrCreateTypeArray({DIBasicInt, DIBasicInt}));

// 创建调试子程序（Function）
auto *DIFunction = DIB.createFunction(
    CU,                     // 所属编译单元
    "addTwoInts",           // 函数名称
    "addTwoInts",           // 链接名称
    File,                   // 文件
    10,                     // 行号
    SPTY,                   // 函数类型
    false,                  // isLocalToUnit
    true,                   // isDefinition
    10,                     // 行号
    0,                      // flags
    llvm::DINode::FlagZero  // flags2
);

// 在函数入口创建插入点
llvm::DIScope *DScope = DIFunction;
llvm::DILocation *DILoc = llvm::DILocation::get(Context, 10, 1, DScope);

// 生成调试描述的分配
llvm::AllocaInst *AllocaX = Builder.CreateAlloca(Int32Ty, nullptr, "x");
llvm::DILocalVariable *DIVarX = DIB.createAutoVariable(
    DScope,
    "x",
    File,
    11,
    DIBasicInt
);
DIB.insertDeclare(
    AllocaX,
    DIVarX,
    DIB.createExpression(),
    DILoc,
    Builder.GetInsertBlock()
);

// 完成 DIBuilder 构建
DIB.finalize();
```
注意：调试信息构建较为繁琐，需确保生成的 IR 包含正确的 !dbg 元数据链接。


## 验证与打印 IR

### 验证模块与函数

利用 llvm::Verifier 对模块或函数 Integrity 进行验证，发现 IR 中潜在的问题。
```cpp
#include "llvm/IR/Verifier.h"

// 验证整个模块
if (llvm::verifyModule(*ModulePtr, &llvm::errs())) {
    llvm::errs() << "模块验证失败！\n";
}

// 验证某个函数
for (auto &F : *ModulePtr) {
    if (llvm::verifyFunction(F, &llvm::errs())) {
        llvm::errs() << "函数 " << F.getName() << " 验证失败！\n";
    }
}
```
打印 IR
- 打印到标准输出
```cpp
ModulePtr->print(llvm::outs(), nullptr);
```

- 打印到文件
```cpp
std::error_code EC;
llvm::raw_fd_ostream OS("output.ll", EC, llvm::sys::fs::OF_None);
ModulePtr->print(OS, nullptr);
OS.flush();
``` 

- 以 Bitcode 格式写入

```cpp
#include "llvm/Bitcode/BitcodeWriter.h"

std::error_code EC2;
llvm::raw_fd_ostream BCOS("output.bc", EC2, llvm::sys::fs::OF_None);
llvm::WriteBitcodeToFile(*ModulePtr, BCOS);
BCOS.flush();

```



## Module 操作（链接、写入 Bitcode 等）

### 链接多个模块

使用 llvm::Linker 将若干模块合并成一个。需包含 llvm/Linker/Linker.h。
```cpp
#include "llvm/Linker/Linker.h"

// 假设有模块 ModA, ModB，想要把 ModB 链接进 ModA
llvm::Linker TheLinker(*ModA);
if (TheLinker.linkInModule(std::move(ModB))) {
    llvm::errs() << "模块链接失败！\n";
}
```

写入与读取 Bitcode
- 写入 Bitcode

```cpp
llvm::WriteBitcodeToFile(*ModulePtr, BCOS);
```


- 读取 Bitcode

```cpp
#include "llvm/Bitcode/BitcodeReader.h"
#include "llvm/Support/MemoryBuffer.h"

llvm::Expected<std::unique_ptr<llvm::Module>> ModOrErr =
    llvm::parseBitcodeFile(llvm::MemoryBufferRef(Buf->getMemBufferRef()), Context);
if (!ModOrErr) {
    llvm::errs() << "读取 bitcode 错误: " << llvm::toString(ModOrErr.takeError()) << "\n";
} else {
    auto &LoadedMod = *ModOrErr;
}
```

## 小结
- llvm::Module：表示一个完整的 IR 单元，如何创建、遍历、验证与打印。
- llvm::LLVMContext：表示 LLVM 上下文，存放唯一化表等数据。
- llvm::Type 及其子类：如何获取各种基础类型、指针类型、数组/向量类型、函数类型和结构体类型。
- llvm::Value：所有 IR 值的基类，如何获取类型、名称，遍历使用者并替换。
- llvm::GlobalVariable 与 llvm::GlobalAlias：模块级别的全局符号与别名操作。
- llvm::Function：函数的创建、属性设置、遍历基本块与参数。
- llvm::BasicBlock：创建、前驱/后继遍历以及在函数间移动。
- llvm::Constant 及其子类：整型常量、浮点常量、常量表达式的构造与使用。
- llvm::Instruction 及常见指令：二元运算、比较、alloca、load、store、GEP、PHI、Call、Branch、Switch、Return、Cast、Select 等。
- llvm::IRBuilder<>：高效生成 IR 指令的工具，封装创建指令并自动管理插入位置。
- 元数据与调试信息：普通元数据节点与使用 llvm::DIBuilder 构建 DWARF 调试信息。
- 验证与打印 IR：使用 Verifier 验证模块/函数，打印到标准输出或文件；Bitcode 的写入与读取。
- 模块链接：如何使用 llvm::Linker 将多个模块合并。
