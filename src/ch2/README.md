<!--
 * @Author: Albresky albre02@outlook.com
 * @Date: 2025-08-16 23:14:02
 * @LastEditors: Albresky albre02@outlook.com
 * @LastEditTime: 2025-08-18 19:43:07
 * @FilePath: /llvm_toy/src/ch2/README.md
 * @Description: LLVM tutorial 2. Kaleidoscope: Implementing a Parser and AST
-->

本节将基于第1节中的词法分析器 lexer，引入编译器中最基本的概念——解析器和抽象语法树（Abstract Syntax Tree, AST）。

AST 用于描述一个语言的行为和结构。它是编译器的核心数据结构之一，能够帮助我们理解程序的语义。通过构建 AST，我们可以更容易地进行语义分析、优化和代码生成等后续步骤。在 `Kaleidoscope` 语言中，我们会设计一个表达式原型 `ExprAST` 来作为 AST 中所有表达式节点的基类：

```c++
// Base class for all AST expression nodes
class ExprAST{
public:
    virtual ~ExprAST() = default;

class NumberExprAST : public ExprAST {
    double Value;
public:
    NumberExprAST(double Val) : Value(Val) {}
};
```

这里的 `NumberExprAST` 继承自 `ExprAST` 用于表示 **数字字面值**。我们可以继续添加其他类型的表达式节点，例如变量、二元运算符等。

```c++
// 变量
class VariableExprAST : public ExprAST {
    std::string Name;
public:
    VariableExprAST(const std::string &Name) : Name(Name) {}
};

// 二元操作符，记录 opcode和左右值
class BinaryExprAST : public ExprAST {
    char Op;
    std::unique_ptr<ExprAST> LHS, RHS;
public:
    BinaryExprAST(char Op, std::unique_ptr<ExprAST> LHS, std::unique_ptr<ExprAST> RHS)
        : Op(Op), LHS(std::move(LHS)), RHS(std::move(RHS)) {}
};

// 函数调用
class CallExprAST: public ExprAST{
    std::string Callee;
    std::vector<std::unique_ptr<ExprAST>> Args;
public:
    CallExprAST(const std::string &Callee, std::vector<std::unique_ptr<ExprAST>> Args)
        : Callee(Callee), Args(std::move(Args)) {}
};
```

以上便是 `Kaleidoscope` 语言中全部的表达式类型。接下来是对函数节点的抽象，通过一个  `PrototypeAST` 和 `FunctionAST` 类来表示函数的原型和定义。

- [函数原型、定义、签名的联系与区别](https://blog.csdn.net/qq_39827640/article/details/129431021)

```c++
class PrototypeAST {
    std::string Name;
    std::vector<std::string> Args;
public:
    PrototypeAST(const std::string &Name, std::vector<std::string> Args)
        : Name(Name), Args(std::move(Args)) {}
};

// 函数定义
class FunctionAST {
    std::unique_ptr<PrototypeAST> Proto;
    std::unique_ptr<ExprAST> Body;
public:
    FunctionAST(std::unique_ptr<PrototypeAST> Proto, std::unique_ptr<ExprAST> Body)
        : Proto(std::move(Proto)), Body(std::move(Body)) {}
};


### Parser 解析器

为构建 AST 我们还需要定义 parser 来解析输入的语言。

比如我们意图解析 `x+y` 的表达式：

```c++
auto LHS = std::make_unique<VariableExprAST>("x");
auto RHS = std::make_unique<VariableExprAST>("y");
auto Result = std::make_unique<BinaryExprAST>('+', std::move(LHS), std::move(RHS));
```

对于 `Kaleidoscope` 语言，这里需要扩充 `lexer` 的方法，用以获取当前和下一个读取到的 token：

```c++
static int CurTok; // 记录 parser 当前正在处理的token
static int getNextToken() {
  return CurTok = gettok();
}
```

此外，一些简单的帮助函数将用于打印 parser 过程中的异常情况：

```c++
std::unique_ptr<ExprAST> LogError(const char *Str) {
  fprintf(stderr, "Error: %s\n", Str);
  return nullptr;
}
std::unique_ptr<PrototypeAST> LogErrorP(const char *Str) {
  LogError(Str);
  return nullptr;
}
```

#### 基础 Expression Parsing

`Kaleidoscope` 中最简单的 parser 就是 `ParseNumberExpr` 负责从tokens 中解析出 `NumberExprAST` 然后递增 token 指针。

```c++
std::unique_ptr<ExprAST> ParseNumberExpr() {
  auto Result = std::make_unique<NumberExprAST>(CurTokVal);
  getNextToken();
  return Result;
}
```

然后是圆括号的匹配，用于从圆括号中解析表达式。该parser可以从表达式中递归提取圆括号中的表达式。

```c++
static std::unique_ptr<ExprAST> ParseParenExpr() {
  getNextToken(); // eat '('
  auto expr = ParseExpression();
  if (!expr) return nullptr;
  if (CurTok != ')') return LogError("Expected ')'");
  getNextToken(); // eat ')'
  return expr;
}
```

进一步将定义如何解析变量和函数调用。

```c++
/// identifierexpr
///   ::= identifier
///   ::= identifier '(' expression* ')'
static std::unique_ptr<ExprAST> ParseIdentifierExpr() {
  std::string IdName = IdentifierStr;

  getNextToken();  // eat identifier.

  if (CurTok != '(') // Simple variable ref.
    return std::make_unique<VariableExprAST>(IdName);

  // Call.
  getNextToken();  // eat (
  std::vector<std::unique_ptr<ExprAST>> Args;
  if (CurTok != ')') {
    while (true) {
      if (auto Arg = ParseExpression())
        Args.push_back(std::move(Arg));
      else
        return nullptr;

      if (CurTok == ')')
        break;

      if (CurTok != ',')
        return LogError("Expected ')' or ',' in argument list");
      getNextToken();
    }
  }

  // Eat the ')'.
  getNextToken();

  return std::make_unique<CallExprAST>(IdName, std::move(Args));
}
```

目前，所有的简单表达式的parse已定义完毕，我们还需要将这些parser封装一层，提供一个统一的接口供外部调用。

```c++
static std::unique_ptr<ExprAST> ParsePrimary() {
  switch (CurTok) {
  case tok_number:
    return ParseNumberExpr();
  case tok_identifier:
    return ParseIdentifierExpr();
  case '(':
    return ParseParenExpr();
  default:
    return LogError("Unknown token");
  }
}

#### 二元 Expression Parsing

下面的二元表达式的parser设计将会显著复杂，因为这里得额外再考虑运算符的优先级和结合率。比如解析 `x + y * z`应该被解析成 `x + (y * z)` 还是 `(x + y) * z`，这是非常关键的问题。这里，`Kaleidoscope` 根据**二元运算符优先级**来判定解析顺序（**运算符优先解析算法**）。

这个算法的核心思想可以概括为：

我有一个左边的表达式（LHS），然后我不断地去看右边的一个个 `[运算符, 右边表达式]` 对。对于每一个新的运算符，我都要和它右边的下一个运算符比一下优先级，来决定**谁应该“先算”**。

下面的代码定义了二元运算符的优先级映射表和获取当前token优先级的函数：
```c++
static std::map<char, int> BinopPrecedence;

static int GetTokPrecedence(){
    if(!isascii(CurTok)) return -1;
    int TokPrec = BinopPrecedence[CurTok];
    return TokPrec <= 0  ? -1 : TokPrec;
}

int main(){
    // Install standard binary operators
    BinopPrecedence['+'] = 10;
    BinopPrecedence['-'] = 10;
    BinopPrecedence['*'] = 20;
    BinopPrecedence['/'] = 20;
    // ...
}
```

对于一个基本表达式例子 `a + b * c - d`，`a` 后面可能跟有一系列 [二元运算符，基本表达式]对，我们通过 `parseExpression` 方法来解析基本表达式：

```c++
static std::unique_ptr<ExprAST> parseExpression(){
    auto LHS = ParsePrimary();  // a
    if(!LHS) return nullptr;
    return ParseBinOpRHS(0, std::move(LHS)); //ParseBinOpRHS(0, a)
}
```

其中，`ParseBinOpRHS` 方法将负责解析后续的二元运算符和基本表达式，二元运算符的优先级和指向当前已解析部分的指针将作为参数传入：

```c++
static std::unique_ptr<ExprAST> ParseBinOpRHS(int ExprPrec, std::unique_ptr<ExprAST> LHS){
    while(true){
        auto tokPrec = GetTokPrecedence();
        if(tokPrec < ExprPrec) return LHS;

        // 获取二元运算符
        int binOp = CurTok;
        getNextToken();  // eat binop

        // Parse the right-hand side.
        auto RHS = ParsePrimary();
        if(!RHS) return nullptr;

        // If the next token is a binary operator that is
        // at least as high precedence as `binOp`, then
        // we need to do another parse.
        while(true){
            auto nextTokPrec = GetTokPrecedence();
            if(nextTokPrec < tokPrec) break;

            // Get the next binary operator.
            int nextBinOp = CurTok;
            getNextToken();  // eat binop

            // Parse the right-hand side.
            auto nextRHS = ParsePrimary();
            if(!nextRHS) return nullptr;

            // Merge the two expressions.
            RHS = std::make_unique<BinaryExprAST>(nextBinOp, std::move(RHS), std::move(nextRHS));
        }

        // Merge the two expressions.
        LHS = std::make_unique<BinaryExprAST>(binOp, std::move(LHS), std::move(RHS));
    }
}
```

如何理解？列个表，拿例子 `a + b * c - d` 跑一遍 `parseExpression()`:


|LHS|RHS|nextRHS|binOp|nextBinOp|
|---|---|---|---|---|
|a|b| |+| |
| ||c||*|
||b*c|d||-|
||b*c-d||||
|a+(b*c)-d| | | | |
||||||


#### 解析剩余部分

现在还需处理函数原型，`Kaleidoscope` 中使用 `extern` 来声明函数，因此我们先要解析这个关键字：

```c++
static std::unique_ptr<PrototypeAST> ParsePrototype(){
    if(CurTok != tok_identifier) return LogErrorP("Expected function name in prototype");
    std::string FuncName = IdentifierStr;
    getNextToken();  // eat identifier

    if(CurTok != '(') return LogErrorP("Expected '(' in prototype");

    // LLVM 官方教程少这行代码 :-G
    getNextToken(); // eat '('

    std::vector<std::string> ArgNames;
    while(getNextToken() == tok_identifier){
        ArgNames.push_back(IdentifierStr);
    }
    if(CurTok!=')') return LogErrorP("Expected ')' in prototype");

    // Eat the ')'.
    getNextToken();

    return std::make_unique<PrototypeAST>(FuncName, std::move(ArgNames));
}
```

现在解析函数定义(包含函数原型和函数体)将十分简单：

```c++
static std::unique_ptr<FunctionAST> ParseDefinition(){
    getNextToken(); // eat def
    auto Proto = ParsePrototype();
    if(!Proto) return nullptr;

    if(auto expr = ParseExpression())
        return std::make_unique<FunctionAST>(std::move(Proto), std::move(expr));
    return nullptr;
}
```

解析`extern`声明过程中，不含函数体的函数，如 `sin`、`cos`：

```c++
static std::unique_ptr<PrototypeAST> ParseExtern(){
    getNextToken(); // eat extern
    return ParsePrototype();
}

此外，还允许解析任意顶级表达式，这里将他定义为匿名无参函数来处理。

顶级表达式示例：
```c++
extern sin;
extern cos;
```

解析器：
```c++
static std::unique_ptr<FunctionAST> ParseTopLevelExpr() {
    if (auto expr = ParseExpression()) {
        // Make an anonymous proto.
        auto Proto = std::make_unique<PrototypeAST>(/* __anon_expr */"", std::vector<std::string>());
        return std::make_unique<FunctionAST>(std::move(Proto), std::move(expr));
    }
    return nullptr;
}
```

### 顶层驱动器

最后，我们为上述 AST 的 parser 构建一个顶层驱动API供调用：

```c++
static void MainLoop(){
    while(true){
        fprintf(stderr, "ready>");
        switch(CurTok){
            case tok_eof:
                return;
            case ';':
                getNextToken();
                break;
            case tok_def:
                HandleDefinition();
                break;
            case tok_extern:
                HandleExtern();
                break;
            default:
                HandleTopLevelExpr();
                break;
        }
    }
}
```

完整代码见 [./parser.cpp](./parser.cpp)。