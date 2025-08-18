<!--
 * @Author: Albresky albre02@outlook.com
 * @Date: 2025-08-16 22:45:26
 * @LastEditors: Albresky albre02@outlook.com
 * @LastEditTime: 2025-08-18 19:55:21
 * @FilePath: /llvm_toy/src/ch1/README.md
 * @Description: LLVM tutorial 1. Kaleidoscope: Kaleidoscope Introduction
-->
LLVM 教程通过引入 `Kaleidoscope` 这个简单而全面的语言来解释其工作流程。本节将介绍该语言的最小集实现 ———— Lexer 词法分析器。

设计好一个语言后，第一件最常用的操作是识别一段文本，这里的首要工作是分析出这段文本中每个 `token` 的含义，这就是 `lexer` 词法分析器的职责。`lexer` 解析到的 token 基本包含一个 token 码和一些元信息 (比如一个数的值)。下面是 token 的码：

```c++
enum Token{
    tok_eof = -1;
    tok_def = -2;
    tok_extern =- 3;
    tok_identifier = -4;
    tok_number = -5;
};

static std::string IdentifierStr;
static double NumVal;
```

而 lexer 的简单实现如下：

```c++
/**
 * 获取下一个 token，返回 token 码
 */
static int gettokk(){
    static int LastChar = ' ';

    // 去除空白字符
    while(isspace(LastChar))
        LastChar = getchar();
    
    // 字母标识符
    if(isalpha(LastChar)){
        IdentifierStr = LastChar;
        while(isalnum((LastChar = getchar())))
            IdentifierStr += LastChar;

        // 解析 token 码
        if(IdentifierStr == "def")
            return tok_def;
        if(IdentifierStr == "extern")
            return tok_extern;
        return tok_identifier;
    // 数字标识符
    } else if(isdigit(LastChar)){
        std::string NumStr;
        do {
            NumStr += LastChar;
            LastChar = getchar();
        } while(isdigit(LastChar));
        NumVal = strtod(NumStr.c_str(), 0);
        return tok_number;
    }
}
```

上面这个词法分析器的实现很简单，可以识别出如 `def`, `extern` 等关键字，也可识别 `9.13` 这样的数字，但是像 `9.13.2025` 等类型的非法数字也会被识别成合法数字，这是由于我们的 lexer 太简单了，嘿嘿。

下面是注释解析和文件 EOF 处理的 lexer 剩余实现：

```c++
if (LastChar == '#') {
  // Comment until end of line.
  do
    LastChar = getchar();
  while (LastChar != EOF && LastChar != '\n' && LastChar != '\r');

  if (LastChar != EOF)
    return gettok();
}

if (LastChar == EOF)
    return tok_eof;

  // Otherwise, just return the character as its ascii value.
  int ThisChar = LastChar;
  LastChar = getchar();
  return ThisChar;
}
```

词法分析器的完整代码见 [./lexer.h](./lexer.h)