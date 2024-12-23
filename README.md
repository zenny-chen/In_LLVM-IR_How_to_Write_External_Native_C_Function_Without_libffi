# In LLVM-IR Interpreter Mode How To Write External Native C Functions Without libffi

In LLVM-IR interpreter mode, how to write external native C functions without libffi?

<br />

In LLVM-IR interpreter mode, [**libffi**](https://github.com/libffi/libffi) is required to help invoking external native C/C++ functions because **libffi** can process all kinds of function calling conventions and it is portable.

However, **libffi** is not so easy to build on certain platforms. Fortunately, LLVM-IR interpreter contains a mechanism to pre-install native runtime wrapper functions into the execution engine environment. We can refer to **`llvm/lib/ExecutionEngine/Interpreter/ExternalFunctions.cpp`** for detail.

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



