## 1.1. Kaleidoscope语言

本教程将以一个玩具语言“Kaleidoscope”为例进行演示（取自“意味美丽、形式和视图”）。Kaleidoscope是一种过程式语言，允许定义函数、使用条件语句、进行数学运算等。在整个教程中，我们将扩展Kaleidoscope以支持if/then/else结构、for循环、用户定义的运算符、具有简单命令行接口的JIT编译、调试信息等。

为了保持简单，Kaleidoscope中唯一的数据类型是64位浮点类型（即C语言中的‘double’）。因此，所有的值都是隐式双精度的，语言不需要类型声明。这使得语言具有非常简洁的语法。例如，下面的简单示例计算斐波那契数：

```py
def fib(x)
  if x < 3 then
    1
  else
    fib(x-1) + fib(x-2)

fib(40)
```

我们还允许Kaleidoscope调用标准库函数——LLVM JIT使这变得非常容易。这意味着你可以使用‘extern’关键字在使用函数之前定义它（这对于相互递归的函数也很有用）。例如：

```c
extern sin(arg);
extern cos(arg);
extern atan2(arg1 arg2);

atan2(sin(.4), cos(42))
```

在第6章中包含了一个更有趣的示例，我们编写了一个小的Kaleidoscope应用程序，以不同的放大级别显示曼德布罗特集合。

让我们深入了解这个语言的实现吧！

## 1.2. 词法分析器

在实现一种语言时，首先需要的是处理文本文件并识别其内容的能力。传统的方法是使用“词法分析器”（又称“扫描器”）将输入分解为“标记”。词法分析器返回的每个标记都包含一个标记代码和一些元数据（例如数字的数值）。首先，我们定义可能的标记类型：

```cpp
// 词法分析器返回[0-255]的标记代码，如果是未知字符，否则返回下列已知标记之一。
enum Token {
  tok_eof = -1,

  // 命令
  tok_def = -2,
  tok_extern = -3,

  // 基本类型
  tok_identifier = -4,
  tok_number = -5,
};

static std::string IdentifierStr; // 如果是tok_identifier则填充
static double NumVal;             // 如果是tok_number则填充
```

词法分析器返回的每个标记将是Token枚举值之一，或者是一个‘未知’字符（如‘+’），其返回ASCII值。如果当前标记是标识符，IdentifierStr全局变量将保存标识符的名称。如果当前标记是数字字面值（如1.0），NumVal保存其值。为了简单起见，我们使用全局变量，但这并不是实现真正语言的最佳选择。

词法分析器的实际实现是一个名为gettok的函数。gettok函数用于从标准输入返回下一个标记。其定义如下：

```cpp
/// gettok - 从标准输入返回下一个标记。
static int gettok() {
  static int LastChar = ' ';

  // 跳过所有空白字符。
  while (isspace(LastChar))
    LastChar = getchar();
```

gettok通过调用C的getchar()函数一次读取一个字符。它在识别字符时将它们吃掉，并将最后一个读取但未处理的字符存储在LastChar中。首先，它必须忽略标记之间的空白字符。这可以通过上面的循环完成。

接下来，gettok需要识别标识符和特定的关键字，如“def”。Kaleidoscope通过下面的简单循环实现：

```cpp
if (isalpha(LastChar)) { // 标识符：[a-zA-Z][a-zA-Z0-9]*
  IdentifierStr = LastChar;
  while (isalnum((LastChar = getchar())))
    IdentifierStr += LastChar;

  if (IdentifierStr == "def")
    return tok_def;
  if (IdentifierStr == "extern")
    return tok_extern;
  return tok_identifier;
}
```

注意，这段代码在词法分析标识符时设置了‘IdentifierStr’全局变量。此外，由于语言关键字通过相同的循环匹配，我们在内联处理中处理它们。数字值类似：

```cpp
if (isdigit(LastChar) || LastChar == '.') {   // 数字：[0-9.]+
  std::string NumStr;
  do {
    NumStr += LastChar;
    LastChar = getchar();
  } while (isdigit(LastChar) || LastChar == '.');

  NumVal = strtod(NumStr.c_str(), 0);
  return tok_number;
}
```

这是处理输入的相当直接的代码。当从输入中读取数值时，我们使用C的strtod函数将其转换为存储在NumVal中的数值。注意，这并没有做充分的错误检查：它会错误地读取“1.23.45.67”，并将其处理为“1.23”。可以随意扩展它！接下来我们处理注释：

```cpp
if (LastChar == '#') {
  // 注释直到行尾。
  do
    LastChar = getchar();
  while (LastChar != EOF && LastChar != '\n' && LastChar != '\r');

  if (LastChar != EOF)
    return gettok();
}
```

我们通过跳到行尾来处理注释，然后返回下一个标记。最后，如果输入不匹配上述情况，它可能是一个运算符字符（如‘+’）或文件结束。通过以下代码处理：

```cpp
  // 检查文件结尾。不要吃掉EOF。
  if (LastChar == EOF)
    return tok_eof;

  // 否则，只返回字符的ascii值。
  int ThisChar = LastChar;
  LastChar = getchar();
  return ThisChar;
}
```

通过这些，我们完成了基本的Kaleidoscope语言的词法分析器（词法分析器的完整代码列表在本教程的下一章中提供）。接下来我们将构建一个简单的解析器，使用它来构建抽象语法树。当我们有了这个，我们将包括一个驱动程序，以便你可以一起使用词法分析器和解析器。