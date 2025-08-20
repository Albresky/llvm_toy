<!--
 * @Author: Albresky albre02@outlook.com
 * @Date: 2025-08-18 19:57:09
 * @LastEditors: Albresky albre02@outlook.com
 * @LastEditTime: 2025-08-18 23:08:49
 * @FilePath: /llvm_toy/src/ch3/README.md
 * @Description: LLVM tutorial 3. Kaleidoscope: Code Generation to LLVM IR
-->

> 本节起，源码编译依赖 llvm 工具链，环境配置可参考 [./llvm_environment_setup.md](./llvm_environment_setup.md)。下面的内容将逐步说明如何将第2节的AST抽象语法树转换为LLVM IR，这里将极大复用 LLVM 框架。

### CodeGen Setup

我们第一步要做的事情是为我们定义的 AST表达式类添加 codegen 方法：

```c++
class ExprAST {
public:
  virtual ~ExprAST() = default;
  virtual Value *codegen() = 0;
};

/// NumberExprAST - Expression class for numeric literals like "1.0".
class NumberExprAST : public ExprAST {
  double Val;

public:
  NumberExprAST(double Val) : Val(Val) {}
  Value *codegen() override;
};
```

`codegen` 方法用于从该 AST 节点及其依赖项生成IR，且该方法返回一个 LLVM Value 对象，这个对象用于表示 LLVM 中的 `SSA register` 或 `SSA value` [静态单赋值](https://zh.wikipedia.org/wiki/%E9%9D%99%E6%80%81%E5%8D%95%E8%B5%8B%E5%80%BC%E5%BD%A2%E5%BC%8F)类。SSA 值最显著的特点是它们的值在相关指令执行时计算，并且只有在（如果）指令重新执行时才会获得新值。换句话说，没有办法“改变”SSA 值。

LLVM 教程为了简化是通过添加虚拟方法来遍历 AST 的，实际工程中会通过[访问者模式](https://zh.wikipedia.org/wiki/%E8%AE%BF%E9%97%AE%E8%80%85%E6%A8%A1%E5%BC%8F)来遍历。

另外，codegen过程中同样也需要类似 `LogErrorP` 一样的错误处理机制，以便在生成 IR 过程中能够及时捕获并报告错误。

```c++
Value *LogErrorV(const char *Str) {
  LogError(Str);
  return nullptr;
}
```

下面将介绍4个静态全局变量，用以简单说明 LLVM codegen过程中需要用到的组件信息。

```c++
static std::unique_ptr<LLVMContext> TheContext;
static std::unique_ptr<IRBuilder<>> Builder;
static std::unique_ptr<Module> TheModule;
static std::map<std::string, Value *> NamedValues;
```

- `TheContext` 对象是一个包含大量核心 LLVM 数据结构的不可见对象，例如类型和常量值表。我们不需要详细了解它，只需要一个实例传递给需要它的 API。

- `Builder` 对象是一个用于简化 IR 生成的工具类，它提供了一系列方便的方法来创建和插入 LLVM 指令。我们将在 codegen 过程中频繁使用它。

- `TheModule` 对象是一个表示整个 LLVM 模块的容器，在很多方面，它是 LLVM IR 用来包含代码的顶层结构，它包含了所有的**函数**和**全局变量**。我们将在 codegen 过程中将生成的函数和全局变量插入到这个模块中。

- `NamedValues` 是一个映射表，用于存储当前作用域内的所有命名值（变量）。在 codegen 过程中，我们需要查找和插入这些命名值，以便正确生成 IR。`NamedValues` 跟踪当前作用域中定义的值及其 LLVM 表示形式。（换句话说，它是代码的符号表）。在这种 `Kaleidoscope` 形式中，唯一可以引用的是函数参数。因此，在为函数体生成代码时，函数参数将存在于这个映射表中。


### Expr CodeGen 表达式代码生成

从最简单的 `NumberExprAST` 的代码生成开始，它的 codegen 只需创建并返回一个 `ConstantFP`，这是 LLVM IR 中用于表示数值常量的类。其中 `APFloat` 为持有任意精度的浮点常量。

```c++
Value *NumberExprAST::codegen() {
  return ConstantFP::get(*TheContext, APFloat(Val));
}
```

接下来是 `VariableExprAST`，它表示一个变量的引用。它的 codegen 方法需要查找变量名在 `NamedValues` 符号表中的对应值，并返回该值。

```c++
Value *VariableExprAST::codegen() {
  // Look this variable up in the function.
  Value *V = NamedValues[Name];
  if (!V)
    LogErrorV("Unknown variable name");
  return V;
}
```

后续小节中将添加对循环归纳变量、局部变量的支持。

下面是二元操作符的codegen，其基本思路是递归的为目标表达式的左侧生成代码，然后生成右侧代码，最后计算二元表达式的值（中序遍历）。这里根据 opcode创建不同的LLVM指令，调用LLVM的构建器对象Builder，：

```c++
Value *BinaryExprAST::codegen() {
  Value *L = LHS->codegen();
  Value *R = RHS->codegen();
  if (!L || !R)
    return nullptr;

  switch (Op) {
  case '+':
    return Builder->CreateFAdd(L, R, "addtmp");
  case '-':
    return Builder->CreateFSub(L, R, "subtmp");
  case '*':
    return Builder->CreateFMul(L, R, "multmp");
  case '<':
    L = Builder->CreateFCmpULT(L, R, "cmptmp");
    // Convert bool 0/1 to double 0.0 or 1.0
    return Builder->CreateUIToFP(L, Type::getDoubleTy(*TheContext),
                                 "booltmp");
  default:
    return LogErrorV("invalid binary operator");
  }
}
```

注意，LLVM会自动为上述符号中如 `addtmp`、`subtmp` 添加一个递增的唯一数字后缀，以确保每个临时变量的名称都是唯一的。

LLVM 的指令有严格约束：如加法指令的左右操作数必须是相同类型。

函数调用的codegen同样很直接，我们先确定 LLVM 模块的符号表中能够查找到函数名，然后生成对应的调用指令。

```c++
Value *CallExprAST::codegen() {
  // Look up the function in the module.
  Function *F = TheModule->getFunction(Callee);
  if (!F)
    return LogErrorV("Unknown function referenced");

  // Check the function's argument count.
  if (F->arg_size() != Args.size())
    return LogErrorV("Incorrect number of arguments");

  // Generate code for each argument.
  std::vector<Value *> ArgValues;
  for (auto &Arg : Args) {
    Value *ArgValue = Arg->codegen();
    if (!ArgValue)
      return nullptr;
    ArgValues.push_back(ArgValue);
  }

  // Create the call instruction.
  return Builder->CreateCall(F, ArgValues, "calltmp");
}
```


### Func CodeGen 函数代码生成

原型和函数的代码生成必须处理许多细节，这使得它们的代码不如表达式代码生成那么优雅，但允许我们说明一些重要点。首先，让我们谈谈原型的代码生成：它们既用于函数体，也用于外部函数声明。代码以以下内容开始：

```c++
Function *PrototypeAST::codegen() {
  // Make the function type:  double(double,double) etc.
  std::vector<Type*> Doubles(Args.size(),
                             Type::getDoubleTy(*TheContext));
  FunctionType *FT =
    FunctionType::get(Type::getDoubleTy(*TheContext), Doubles, false);

  Function *F =
    Function::Create(FT, Function::ExternalLinkage, Name, TheModule.get());
```

这段代码在几行内包含了大量功能。首先要注意的是，这个函数返回的是 `Function*` 而不是 `Value*`。因为 **原型** 真正讨论的是函数的外部接口（而不是表达式计算出的值），所以当它被代码生成时返回对应的 LLVM Function 是有意义的。

对 `FunctionType::get` 的调用创建了应该用于特定原型的 FunctionType 。由于 `Kaleidoscope` 中所有函数参数的类型都是 `double`，第一行创建了一个包含 `N` 个 LLVM `double` 类型的向量。然后使用 `FunctionType::get` 方法创建一个函数类型，该类型接受 `N` 个 `double` 作为参数，返回一个 `double` 作为结果，并且不是可变参数（`false` 参数表示这一点）。**需要注意的是**，LLVM 中的类型和常量一样是**唯一**的，所以我们不会 `new` 一个类型，而是 `get` 它。

上面最后一行上创建了与原型对应的 IR 函数。这表明要使用的**类型**、**链接**和**名称**，以及要**插入的模块**。外部链接（`Function::ExternalLinkage`） 意味着函数可以定义在当前模块之外，并且/或者可以被模块外的函数调用。传入的名称是用户指定的名称：由于指定了 `TheModule`，此名称注册在 `TheModule` 的符号表中。


最后，我们根据原型中给出的名称设置每个函数参数的名称。这一步并非严格必要，但保持名称一致会使 IR 更易读，并允许后续代码直接通过名称引用参数，而无需在原型 AST 中查找它们。

```c++
unsigned Idx = 0;
for (auto &Arg : F->args())
  Arg.setName(Args[Idx++]);

return F;
```

这时我们已经完成了一个没有函数体的**函数原型**codegen，即对应于`Kaleidoscope` 中 `extern` 声明的语句。接下来是附加了函数体的函数定义的codegen：

```c++
Function *FunctionAST::codegen() {
    // First, check for an existing function from a previous 'extern' declaration.
  Function *TheFunction = TheModule->getFunction(Proto->getName());

  if (!TheFunction)
    TheFunction = Proto->codegen();

  if (!TheFunction)
    return nullptr;

  if (!TheFunction->empty())
    return (Function*)LogErrorV("Function cannot be redefined.");
```

对于函数定义，我们首先在 `TheModule` 的符号表中搜索该函数的现有版本，以防它已经使用 `extern` 语句创建。如果 `Module::getFunction` 返回 null，则不存在之前的版本，因此我们将从原型中生成一个。无论哪种情况，我们都需要在开始之前断言该函数为空（即还没有函数体）。

下面是设置 Builder 的阶段。第一行创建一个新的基本块（命名为 `entry`），并将其插入到 `TheFunction` 中。第二行则告诉 Builder 新的指令应该插入到新基本块的末尾。在 LLVM 中，基本块是定义控制流图的重要部分。由于我们没有控制流，我们的函数在这个阶段只会包含一个块。我们将在第 5 节中解决这个问题：

```c++
// Create a new basic block to start insertion into.
BasicBlock *BB = BasicBlock::Create(*TheContext, "entry", TheFunction);
Builder->SetInsertPoint(BB);

// Record the function arguments in the NamedValues map.
NamedValues.clear();
for (auto &Arg : TheFunction->args())
  NamedValues[std::string(Arg.getName())] = &Arg;
```

接下来我们将函数参数添加到 `NamedValues` 映射中（首先清空它），以便它们能被 `VariableExprAST` 节点访问：
```c++
if (Value *RetVal = Body->codegen()) {
  // Finish off the function.
  Builder->CreateRet(RetVal);

  // Validate the generated code, checking for consistency.
  verifyFunction(*TheFunction);

  return TheFunction;
}
```

一旦插入点设置完成并且 `NamedValues` 映射被填充，我们调用 `codegen()` 方法来处理函数的根表达式。如果没有发生错误，这将生成代码来计算表达式并在入口块中返回计算出的值。假设没有错误，我们接着创建一个 LLVM `ret` 指令，完成函数。一旦函数构建完成，我们调用 `verifyFunction` ，这是由 LLVM 提供的。这个函数对生成的代码进行多种**一致性检查**，以确定我们的编译器是否一切正常。使用这个功能很重要：它能够捕获很多错误。一旦函数完成并验证，我们就可以返回它。

当出现错误时，我们要显式从符号表中删除它：
```c++
  // Error reading body, remove function.
  TheFunction->eraseFromParent();
  return nullptr;
}
```
为简化起见，我们通过简单地删除用 `eraseFromParent` 方法生成的函数来处理这种情况。这允许用户重新定义他们之前错误输入的函数：如果我们不删除它，它将存在于符号表中，并带有函数体，这将阻止未来的重新定义。

上述函数定义的codegen代码存在一个 bug：如果 `FunctionAST::codegen()` 方法找到一个已存在的 IR Function，它不会将其签名与定义自身的原型进行验证。这意味着一个较早的 `extern` 声明将优先于函数定义的签名，这可能导致代码生成失败，例如如果函数参数的命名不同。

### 顶层封装

完整代码中，会将 codegen 调用插入到 `HandleDefinition`、`HandleExtern` 等函数中，然后输出 LLVM IR。

```mlir
ready> 4+5;
Read top-level expression:
define double @0() {
entry:
  ret double 9.000000e+00
}
```

注意解析器如何将顶层表达式转换为匿名函数。这在我们下一章添加 JIT 支持时将很有用。此外请注意代码是逐字转录的，除了 IRBuilder 执行的简单常量折叠外没有进行任何优化。我们将在下一节显式添加优化。


下面展示了一些简单的算术。注意这与我们用来创建指令的 LLVM 构建器调用非常相似：

```mlir
ready> def foo(a b) a*a + 2*a*b + b*b;
Read function definition:
define double @foo(double %a, double %b) {
entry:
  %multmp = fmul double %a, %a
  %multmp1 = fmul double 2.000000e+00, %a
  %multmp2 = fmul double %multmp1, %b
  %addtmp = fadd double %multmp, %multmp2
  %multmp3 = fmul double %b, %b
  %addtmp4 = fadd double %addtmp, %multmp3
  ret double %addtmp4
}
```

