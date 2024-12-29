# In LLVM-IR Interpreter Mode How To Write External Native C Functions Without libffi

In LLVM-IR interpreter mode, how to write external native C functions without libffi?

<br />

In LLVM-IR interpreter mode, [**libffi**](https://github.com/libffi/libffi) is required to help invoking external native C/C++ functions because **libffi** can process all kinds of function calling conventions and it is portable.

However, it is not so easy to build **libffi** on certain platforms. Fortunately, LLVM-IR interpreter contains a mechanism to pre-install native runtime wrapper functions into the execution engine environment. We can refer to **`llvm/lib/ExecutionEngine/Interpreter/ExternalFunctions.cpp`** for detail.

```cpp
// int printf(const char *, ...) - a very rough implementation to make output
// useful.
static GenericValue lle_X_printf(FunctionType *FT,
                                 ArrayRef<GenericValue> Args) {
  char Buffer[10000];
  std::vector<GenericValue> NewArgs;
  NewArgs.push_back(PTOGV((void*)&Buffer[0]));
  llvm::append_range(NewArgs, Args);
  GenericValue GV = lle_X_sprintf(FT, NewArgs);
  outs() << Buffer;
  return GV;
}

// int sscanf(const char *format, ...);
static GenericValue lle_X_sscanf(FunctionType *FT,
                                 ArrayRef<GenericValue> args) {
  assert(args.size() < 10 && "Only handle up to 10 args to sscanf right now!");

  char *Args[10];
  for (unsigned i = 0; i < args.size(); ++i)
    Args[i] = (char*)GVTOP(args[i]);

  GenericValue GV;
  GV.IntVal = APInt(32, sscanf(Args[0], Args[1], Args[2], Args[3], Args[4],
                    Args[5], Args[6], Args[7], Args[8], Args[9]));
  return GV;
}

// int scanf(const char *format, ...);
static GenericValue lle_X_scanf(FunctionType *FT, ArrayRef<GenericValue> args) {
  assert(args.size() < 10 && "Only handle up to 10 args to scanf right now!");

  char *Args[10];
  for (unsigned i = 0; i < args.size(); ++i)
    Args[i] = (char*)GVTOP(args[i]);

  GenericValue GV;
  GV.IntVal = APInt(32, scanf( Args[0], Args[1], Args[2], Args[3], Args[4],
                    Args[5], Args[6], Args[7], Args[8], Args[9]));
  return GV;
}

// int fprintf(FILE *, const char *, ...) - a very rough implementation to make
// output useful.
static GenericValue lle_X_fprintf(FunctionType *FT,
                                  ArrayRef<GenericValue> Args) {
  assert(Args.size() >= 2);
  char Buffer[10000];
  std::vector<GenericValue> NewArgs;
  NewArgs.push_back(PTOGV(Buffer));
  NewArgs.insert(NewArgs.end(), Args.begin()+1, Args.end());
  GenericValue GV = lle_X_sprintf(FT, NewArgs);

  fputs(Buffer, (FILE *) GVTOP(Args[0]));
  return GV;
}

static GenericValue lle_X_memset(FunctionType *FT,
                                 ArrayRef<GenericValue> Args) {
  int val = (int)Args[1].IntVal.getSExtValue();
  size_t len = (size_t)Args[2].IntVal.getZExtValue();
  memset((void *)GVTOP(Args[0]), val, len);
  // llvm.memset.* returns void, lle_X_* returns GenericValue,
  // so here we return GenericValue with IntVal set to zero
  GenericValue GV;
  GV.IntVal = 0;
  return GV;
}

static GenericValue lle_X_memcpy(FunctionType *FT,
                                 ArrayRef<GenericValue> Args) {
  memcpy(GVTOP(Args[0]), GVTOP(Args[1]),
         (size_t)(Args[2].IntVal.getLimitedValue()));

  // llvm.memcpy* returns void, lle_X_* returns GenericValue,
  // so here we return GenericValue with IntVal set to zero
  GenericValue GV;
  GV.IntVal = 0;
  return GV;
}

void Interpreter::initializeExternalFunctions()
{
    auto &Fns = getFunctions();
    sys::ScopedLock Writer(Fns.Lock);
    Fns.FuncNames["lle_X_atexit"]       = lle_X_atexit;
    Fns.FuncNames["lle_X_exit"]         = lle_X_exit;
    Fns.FuncNames["lle_X_abort"]        = lle_X_abort;

    Fns.FuncNames["lle_X_printf"]       = lle_X_printf;
    Fns.FuncNames["lle_X_sprintf"]      = lle_X_sprintf;
    Fns.FuncNames["lle_X_sscanf"]       = lle_X_sscanf;
    Fns.FuncNames["lle_X_scanf"]        = lle_X_scanf;
    Fns.FuncNames["lle_X_fprintf"]      = lle_X_fprintf;
    Fns.FuncNames["lle_X_memset"]       = lle_X_memset;
    Fns.FuncNames["lle_X_memcpy"]       = lle_X_memcpy;
}
```

Here's some code snippets from the source file above. We can observe that `Fns` container records all the external functions as needed and all the external C functions have just one signature -- 

```cpp
static GenericValue (FunctionType *FT, ArrayRef<GenericValue> Args)
```

<br />

So we can follow this function signature to put *user-defined* C/C++ functions into this `Fns` container. Below shows how to do it.

```cpp
static GenericValue lle_X_memcpy(FunctionType *FT,
                                 ArrayRef<GenericValue> Args) {
  memcpy(GVTOP(Args[0]), GVTOP(Args[1]),
         (size_t)(Args[2].IntVal.getLimitedValue()));

  // llvm.memcpy* returns void, lle_X_* returns GenericValue,
  // so here we return GenericValue with IntVal set to zero
  GenericValue GV;
  GV.IntVal = 0;
  return GV;
}

// TODO: Add customized external functions "exported" to the running application...

static GenericValue lle_X_strcpy(FunctionType *FT, ArrayRef<GenericValue> Args)
{
    const char *resStr = strcpy((char*)GVTOP(Args[0]), (const char*)GVTOP(Args[1]));
    
    GenericValue GV;
    GV.PointerVal = PointerTy(resStr);
    return GV;
}

static GenericValue lle_X_strlen(FunctionType *FT, ArrayRef<GenericValue> Args)
{
    const size_t len = strlen((char*)GVTOP(Args[0]));

    GenericValue GV;
    GV.IntVal = APInt(64U, len, false);
    return GV;
}

static GenericValue lle_X_abs(FunctionType *FT, ArrayRef<GenericValue> Args)
{
    const uint64_t src = Args[0].IntVal.getLimitedValue();
    const int res = abs(int(src));

    GenericValue GV;
    GV.IntVal = APInt(32U, uint64_t(res), true);
    return GV;
}

void Interpreter::initializeExternalFunctions()
{
    auto &Fns = getFunctions();
    sys::ScopedLock Writer(Fns.Lock);
    Fns.FuncNames["lle_X_atexit"]       = lle_X_atexit;
    Fns.FuncNames["lle_X_exit"]         = lle_X_exit;
    Fns.FuncNames["lle_X_abort"]        = lle_X_abort;

    Fns.FuncNames["lle_X_printf"]       = lle_X_printf;
    Fns.FuncNames["lle_X_sprintf"]      = lle_X_sprintf;
    Fns.FuncNames["lle_X_sscanf"]       = lle_X_sscanf;
    Fns.FuncNames["lle_X_scanf"]        = lle_X_scanf;
    Fns.FuncNames["lle_X_fprintf"]      = lle_X_fprintf;
    Fns.FuncNames["lle_X_memset"]       = lle_X_memset;
    Fns.FuncNames["lle_X_memcpy"]       = lle_X_memcpy;
    
    // TODO: Append the customized function list
    
    Fns.FuncNames["lle_X_strcpy"]       = lle_X_strcpy;
    Fns.FuncNames["lle_X_strlen"]       = lle_X_strlen;
    Fns.FuncNames["lle_X_abs"]          = lle_X_abs;
}
```

Here, I defined three C functions -- `lle_X_strcpy`, `lle_X_strlen` and `lle_X_abs`. And then, put them into `Fns`. Refer to the "TODO"s.

<br />

Next, we can call the external functions directly in our LLVM code. Here's an example.

```cpp
#include "llvm/ADT/APInt.h"
#include "llvm/IR/Verifier.h"
#include "llvm/ExecutionEngine/ExecutionEngine.h"
#include "llvm/ExecutionEngine/GenericValue.h"
#include "llvm/ExecutionEngine/Interpreter.h"
#include "llvm/ExecutionEngine/MCJIT.h"
#include "llvm/IR/Argument.h"
#include "llvm/IR/BasicBlock.h"
#include "llvm/IR/Constants.h"
#include "llvm/IR/DerivedTypes.h"
#include "llvm/IR/Function.h"
#include "llvm/IR/InstrTypes.h"
#include "llvm/IR/Instructions.h"
#include "llvm/IR/LLVMContext.h"
#include "llvm/IR/Module.h"
#include "llvm/IR/Type.h"
#include "llvm/IR/IRBuilder.h"
#include "llvm/Support/Casting.h"
#include "llvm/Support/TargetSelect.h"
#include "llvm/Support/raw_ostream.h"
#include <cstdio>
#include <cstdint>
#include <cstdlib>

#if _MSC_VER
#define _USE_MATH_DEFINES
#include <math.h>
#endif

#include <cmath>
#include <algorithm>
#include <utility>
#include <memory>
#include <string>
#include <vector>

#if _WIN32
#include <Windows.h>
#endif

using namespace llvm;

static auto CreateExternFuncCalleeTestFunction(Module *m, LLVMContext &context) -> Function*
{
    Type* const int32Type = Type::getInt32Ty(context);
    Type* const int64Type = Type::getInt64Ty(context);
    Type* const int8Type = Type::getInt8Ty(context);
    PointerType* const charPtrType = PointerType::get(int8Type, 0U);
    
    // Create the test function and insert it into module M. This function is said
    // to return an int and take an int parameter.
    // Specify the function prototype.
    // Generate: define i32 @testFunc(i32 %argX) {
    FunctionType* const testFuncType = FunctionType::get(Type::getInt32Ty(context),   // the return type
                                                   { Type::getInt32Ty(context) }, // parameter list types
                                                   false    // Indicates not variable argument list
                                                   );

    // Create the function whose name is "testFunc" into the specified module 'M'.
    // The created function has external linkage.
    Function *testFunc = Function::Create(testFuncType, Function::ExternalLinkage, "testFunc", m);

    // Add a basic block to the function.
    BasicBlock *basicBlock = BasicBlock::Create(context, "EntryBlock", testFunc);
    
    IRBuilder<> basicBlockBuilder(basicBlock, basicBlock->begin());
    
    // Get pointers to the constants.
    Value *constZero = ConstantInt::get(int32Type, 0U, true);
    Value *const64 = ConstantInt::get(int32Type, 64U, false);

    // Get pointer to the integer argument of the function...
    auto argListIter = testFunc->arg_begin();   // Get the argument list iterator
    Argument *argX = argListIter + 0;           // Make ArgX points to the first argument
    argX->setName("argX");                      // Give it a nice symbolic name for the parameter
    
    FunctionType* const printfFuncType = FunctionType::get(int32Type, { charPtrType }, true);
    // Generate: declare i32 @printf(ptr, ...)
    Function *printfFunc = Function::Create(printfFuncType, Function::ExternalLinkage, "printf", m);
    
    std::array<char, 15U> str { 'H', 'e', 'l', 'l', 'o', ',', ' ', 'w', 'o', 'r', 'l', 'd', '!', '\n' };
    Constant *strLiteral = ConstantDataArray::get(context, str);
    // Generate: @staticStrPtr = private constant [15 x i8] c"Hello, world!\0A\00", align 16
    GlobalVariable *staticStrPtr = new GlobalVariable(*m, ArrayType::get(int8Type, str.size()),
                                                    true,                           // As a constant
                                                    GlobalValue::PrivateLinkage,    // static storage class
                                                    strLiteral,                     // initial value
                                                    "staticStrPtr");
    staticStrPtr->setAlignment(Align(16U));
    
    std::array<Value*, 1U> strLiteralIndex { constZero };
    // Generate: %strPtr = getelementptr i8, ptr @staticStrPtr, i32 0
    Value *strPtr = basicBlockBuilder.CreateGEP(int8Type, staticStrPtr, strLiteralIndex, "strPtr");
    
    std::array<Value*, 1U> printfArgs { strPtr };
    // Generate: %printfResult = call i32 (ptr, ...) @printf(ptr @staticStrPtr)
    Value *printfResult = basicBlockBuilder.CreateCall(printfFunc, printfArgs, "printfResult");
    
    FunctionType* const strcpyFuncType = FunctionType::get(charPtrType, { charPtrType, charPtrType }, false);
    // Generate: declare ptr @strcpy(ptr, ptr)
    Function *strcpyFunc = Function::Create(strcpyFuncType, Function::ExternalLinkage, "strcpy", m);
    
    // Generate: @srcStr = private unnamed_addr constant [21 x i8] c"Thank you very much!\00", align 1
    Constant *srcStr = basicBlockBuilder.CreateGlobalStringPtr("Thank you very much!", "srcStr");
    // Generate: %dstStr = alloca i8, i32 64, align 1
    Value *dstStr = basicBlockBuilder.CreateAlloca(int8Type, 0U, const64, "dstStr");
    
    std::array<Value*, 2U> strcpyArgs { dstStr, srcStr };
    // Generate: %copiedStr = call ptr @strcpy(ptr %dstStr, ptr @srcStr)
    Value *copiedStr = basicBlockBuilder.CreateCall(strcpyFunc, strcpyArgs, "copiedStr");
    
    FunctionType* const strlenFuncType = FunctionType::get(int64Type, { charPtrType }, false);
    // Generate: declare i64 @strlen(ptr)
    Function *strlenFunc = Function::Create(strlenFuncType, Function::ExternalLinkage, "strlen", m);
    
    std::array<Value*, 1U> strlenArgs { copiedStr };
    // Generate: %strlenResult = call i64 @strlen(ptr %copiedStr)
    Value *strlenResult = basicBlockBuilder.CreateCall(strlenFunc, strlenArgs, "strlenResult");
    
    // Generate:
    // %0 = trunc i64 %strlenResult to i32
    // %sumResult = add i32 %printfResult, %0
    Value *sumResult = basicBlockBuilder.CreateAdd(printfResult, basicBlockBuilder.CreateTrunc(strlenResult, int32Type), "sumResult");
    
    // Generate: %subRes = sub i32 %argX, %sumResult
    Value *subRes = basicBlockBuilder.CreateSub(argX, sumResult, "subRes");
    
    FunctionType* const absFuncType = FunctionType::get(int32Type, { int32Type }, false);
    // Generate: declare i32 @abs(i32)
    Function *absFunc = Function::Create(absFuncType, Function::ExternalLinkage, "abs", m);
    
    std::array<Value*, 1U> absArgs { subRes };
    // Generate: %absRes = call i32 @abs(i32 %subRes)
    Value *absRes = basicBlockBuilder.CreateCall(absFunc, absArgs, "absRes");

    // Generated: ret i32 %absRes
    basicBlockBuilder.CreateRet(absRes);

    return testFunc;
}

// User-defined globals
extern "C" auto MySubFunc(int a, int b) -> int
{
    return a - b;
}

static auto GenerateAndExecuteFunction(const int& n, auto (*createFunctionCallback)(Module*, LLVMContext&) -> Function*, const char *funcName) -> void
{
    LLVMContext Context;

    // Create a module whose ID is "test", to put our function into it.
    std::unique_ptr<Module> Owner(new Module("test", Context));
    Module *M = Owner.get();

    // We are about to create the specified function:
    Function *specifiedFunction = createFunctionCallback(M, Context);

    // Now we going to create JIT which may execute the module later
    std::string errStr;
    // Use `setEngineKind(llvm::EngineKind::Interpreter)` to set the current execution engine to use interpreter mode
    // instead of JIT mode.
    ExecutionEngine *EE = EngineBuilder(std::move(Owner)).setErrorStr(&errStr).create();
    
    if (EE == nullptr)
    {
        errs() << funcName << ": Failed to construct ExecutionEngine: " << errStr
                << "\n";
        return;
    }
    
    // Add user-defined globals to the JIT Execute Engine
    EE->addGlobalMapping("MySubFunc", uint64_t(MySubFunc));
    
    errs() << "verifying... ";

    if (verifyModule(*M))
    {
        errs() << funcName << ": Error constructing function!\n";
        return;
    }

    errs() << funcName << ": OK\n";
    errs() << "We just constructed this LLVM module:\n\n---------\n" << *M;
    errs() << "---------\nstarting " << funcName << "(" << n << ") with JIT...\n";

    // Call the Fibonacci function with argument n:
    std::vector<GenericValue> args(1);
    args[0].IntVal = APInt(32, n);
    
    GenericValue GV = EE->runFunction(specifiedFunction, args);

    // import result of execution
    outs() << "Result: " << GV.IntVal << "\n";
    
    delete EE;
    
    puts("\n");
}

auto main(int argc, const char *argv[]) -> int
{
#if _WIN32
    SetConsoleOutputCP(CP_UTF8);
#endif

    printf("π + e = %f\n\n", M_PI + M_E);

    InitializeNativeTarget();
    InitializeNativeTargetAsmPrinter();

    GenerateAndExecuteFunction(24, CreateExternFuncCalleeTestFunction, "CreateExternFuncCalleeTest");
    
    llvm_shutdown();

    return 0;
}
```

After running the above code, the output info will be as the following:

```llvm
π + e = 5.859874

verifying... CreateExternFuncCalleeTest: OK
We just constructed this LLVM module:

---------
; ModuleID = 'test'
source_filename = "test"
target datalayout = "e-m:o-p270:32:32-p271:32:32-p272:64:64-i64:64-i128:128-f80:128-n8:16:32:64-S128"

@staticStrPtr = private constant [15 x i8] c"Hello, world!\0A\00", align 16
@srcStr = private unnamed_addr constant [21 x i8] c"Thank you very much!\00", align 1

define i32 @testFunc(i32 %argX) {
EntryBlock:
  %printfResult = call i32 (ptr, ...) @printf(ptr @staticStrPtr)
  %dstStr = alloca i8, i32 64, align 1
  %copiedStr = call ptr @strcpy(ptr %dstStr, ptr @srcStr)
  %strlenResult = call i64 @strlen(ptr %copiedStr)
  %0 = trunc i64 %strlenResult to i32
  %sumResult = add i32 %printfResult, %0
  %subRes = sub i32 %argX, %sumResult
  %absRes = call i32 @abs(i32 %subRes)
  ret i32 %absRes
}

declare i32 @printf(ptr, ...)

declare ptr @strcpy(ptr, ptr)

declare i64 @strlen(ptr)

declare i32 @abs(i32)
---------
starting CreateExternFuncCalleeTest(24) with JIT...
Hello, world!
Result: 10
```

So this method is simple. Although we can hardly avoid modifying the LLVM source code, it just works.

