## 2.1. 第二章简介
欢迎阅读“使用LLVM实现语言”教程的第二章。本章将向您展示如何使用第一章构建的词法分析器，来为我们的Kaleidoscope语言构建一个完整的解析器。一旦我们有了解析器，我们将定义并构建一个抽象语法树（AST）。

我们将构建的解析器使用递归下降解析和运算符优先级解析的组合来解析Kaleidoscope语言（后者用于解析二元表达式，前者用于解析其他所有内容）。在开始解析之前，我们先来谈谈解析器的输出：抽象语法树。

## 2.2. 抽象语法树（AST）
程序的AST以一种便于编译器后续阶段（如代码生成）解释的方式捕捉其行为。我们基本上希望为语言中的每个结构都有一个对象，并且AST应该紧密地模拟语言。在Kaleidoscope中，我们有表达式、原型和函数对象。我们先从表达式开始：

```cpp
/// ExprAST - 所有表达式节点的基类。
class ExprAST {
public:
  virtual ~ExprAST() = default;
};

/// NumberExprAST - 用于数字字面量（如"1.0"）的表达式类。
class NumberExprAST : public ExprAST {
  double Val;

public:
  NumberExprAST(double Val) : Val(Val) {}
};
```

上面的代码展示了基类ExprAST和我们用于数字字面量的一个子类的定义。需要注意的是，NumberExprAST类将字面量的数值作为实例变量捕捉。这使得编译器的后续阶段能够知道存储的数值是多少。

现在我们只是创建AST，因此没有有用的访问器方法。很容易添加一个虚方法来漂亮地打印代码，例如。以下是我们在Kaleidoscope语言的基本形式中将使用的其他表达式AST节点定义：

```cpp
/// VariableExprAST - 用于引用变量的表达式类，如"a"。
class VariableExprAST : public ExprAST {
  std::string Name;

public:
  VariableExprAST(const std::string &Name) : Name(Name) {}
};

/// BinaryExprAST - 用于二元运算符的表达式类。
class BinaryExprAST : public ExprAST {
  char Op;
  std::unique_ptr<ExprAST> LHS, RHS;

public:
  BinaryExprAST(char Op, std::unique_ptr<ExprAST> LHS,
                std::unique_ptr<ExprAST> RHS)
    : Op(Op), LHS(std::move(LHS)), RHS(std::move(RHS)) {}
};

/// CallExprAST - 用于函数调用的表达式类。
class CallExprAST : public ExprAST {
  std::string Callee;
  std::vector<std::unique_ptr<ExprAST>> Args;

public:
  CallExprAST(const std::string &Callee,
              std::vector<std::unique_ptr<ExprAST>> Args)
    : Callee(Callee), Args(std::move(Args)) {}
};
```

这部分代码相对直截了当：变量捕捉变量名，二元运算符捕捉它们的操作码（例如'+'），函数调用捕捉函数名以及任何参数表达式的列表。我们的AST的一大优点是它捕捉了语言特性，而没有讨论语言的语法结构。注意，这里没有关于二元运算符的优先级、词法结构等的讨论。

对于我们的基本语言，这些是我们定义的所有表达式节点。因为它没有条件控制流，所以它不是图灵完备的；我们将在后续章节中解决这个问题。接下来我们需要的是一种方式来描述函数的接口，以及描述函数本身：

```cpp
/// PrototypeAST - 该类表示函数的"原型"，
/// 它捕捉函数名及其参数名（因此隐式地表示了函数的参数数量）。
class PrototypeAST {
  std::string Name;
  std::vector<std::string> Args;

public:
  PrototypeAST(const std::string &Name, std::vector<std::string> Args)
    : Name(Name), Args(std::move(Args)) {}

  const std::string &getName() const { return Name; }
};

/// FunctionAST - 该类表示函数定义本身。
class FunctionAST {
  std::unique_ptr<PrototypeAST> Proto;
  std::unique_ptr<ExprAST> Body;

public:
  FunctionAST(std::unique_ptr<PrototypeAST> Proto,
              std::unique_ptr<ExprAST> Body)
    : Proto(std::move(Proto)), Body(std::move(Body)) {}
};
```

在Kaleidoscope中，函数类型仅由其参数的数量决定。由于所有值都是双精度浮点数，因此不需要在任何地方存储每个参数的类型。在一个更激进和现实的语言中，ExprAST类可能会有一个类型字段。

有了这些支撑结构，我们现在可以讨论在Kaleidoscope中解析表达式和函数体。

## 2.3. 解析器基础
现在我们有了要构建的AST，我们需要定义解析器代码来构建它。我们的目标是解析类似“x+y”的字符串（由词法分析器返回为三个标记）成一个可以通过以下调用生成的AST：

```cpp
auto LHS = std::make_unique<VariableExprAST>("x");
auto RHS = std::make_unique<VariableExprAST>("y");
auto Result = std::make_unique<BinaryExprAST>('+', std::move(LHS), std::move(RHS));
```

为了实现这一点，我们将从定义一些基本的辅助例程开始：

```cpp
/// CurTok/getNextToken - 提供一个简单的标记缓冲区。CurTok是解析器正在查看的当前标记。
/// getNextToken从词法分析器读取另一个标记，并使用结果更新CurTok。
static int CurTok;
static int getNextToken() {
  return CurTok = gettok();
}
```

这实现了围绕词法分析器的一个简单的标记缓冲区。这使我们可以提前查看词法分析器返回的标记。我们解析器中的每个函数都会假设CurTok是当前需要解析的标记。

```cpp
/// LogError* - 这些是用于错误处理的小帮助函数。
std::unique_ptr<ExprAST> LogError(const char *Str) {
  fprintf(stderr, "Error: %s\n", Str);
  return nullptr;
}
std::unique_ptr<PrototypeAST> LogErrorP(const char *Str) {
  LogError(Str);
  return nullptr;
}
```

LogError例程是我们的解析器用来处理错误的简单帮助例程。我们解析器中的错误恢复不是最好的，并且对用户不特别友好，但对于我们的教程来说已经足够了。这些例程使在具有各种返回类型的例程中更容易处理错误：它们总是返回空。

有了这些基本的帮助函数，我们可以实现我们的语法的第一个部分：数字字面量。

## 2.4. 基本表达式解析
我们从数字字面量开始，因为它们是最简单的处理。对于语法中的每个产生式，我们将定义一个解析该产生式的函数。对于数字字面量，我们有：

```cpp
/// numberexpr ::= number
static std::unique_ptr<ExprAST> ParseNumberExpr() {
  auto Result = std::make_unique<NumberExprAST>(NumVal);
  getNextToken(); // 消耗数字
  return std::move(Result);
}
```

这个例程非常简单：它预计在当前标记是tok_number标记时被调用。它获取当前的数字值，创建一个NumberExprAST节点，推进词法分析器到下一个标记，最后返回。

其中有一些有趣的方面。最重要的一点是，这个例程会吃掉所有对应于产生式的标记，并返回带有下一个标记（不属于语法产生式）的词法分析器缓冲区。这是递归下降解析器的一种相当标准的方法。为了更好地例示，括号运算符定义如下：

```cpp
/// parenexpr ::= '(' expression ')'
static std::unique_ptr<ExprAST> ParseParenExpr() {
  getNextToken(); // 吃掉(.
  auto V = ParseExpression();
  if (!V)
    return nullptr;

  if (CurTok != ')')
    return LogError("expected ')'");
  getNextToken(); // 吃掉).
  return V;
}
```

这个函数展示了解析器的一些有趣之处：

1）它展示了我们如何使用LogError例程。调用时，该函数期望当前标记是'('标记，但在解析子表达式后，可能没有')'等待。例如，如果用户输入“(4 x”而不是“(4)”，解析器应该发出错误。因为错误可能会发生，解析器需要一种方式来表示它们发生了：在我们的解析器中，错误时返回null。

2）另一个有趣的方面是它使用了递归，调用ParseExpression（我们很快会看到ParseExpression可以调用ParseParenExpr）。这很强大，因为它允许我们处理递归语法，并保持每个产生式非常简单。注意，括号本身并不会构造AST节点。虽然我们可以这样

做，但括号的最重要作用是引导解析器并提供分组。一旦解析器构建了AST，括号就不再需要了。

下一个简单的产生式是处理变量引用和函数调用：

```cpp
/// identifierexpr
///   ::= identifier
///   ::= identifier '(' expression* ')'
static std::unique_ptr<ExprAST> ParseIdentifierExpr() {
  std::string IdName = IdentifierStr;

  getNextToken();  // 吃掉标识符。

  if (CurTok != '(') // 简单变量引用。
    return std::make_unique<VariableExprAST>(IdName);

  // 调用。
  getNextToken();  // 吃掉(
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

  // 吃掉')'。
  getNextToken();

  return std::make_unique<CallExprAST>(IdName, std::move(Args));
}
```

这个例程遵循与其他例程相同的风格。（它期望在当前标记是tok_identifier标记时被调用）。它也有递归和错误处理。一个有趣的方面是它使用前瞻来确定当前标识符是一个独立的变量引用还是一个函数调用表达式。它通过检查标识符之后的标记是否为'('标记来处理这个问题，适当地构造一个VariableExprAST或CallExprAST节点。

现在我们已经有了所有简单的表达式解析逻辑，我们可以定义一个帮助函数将其整合为一个入口点。我们将这一类表达式称为“基本”表达式，原因将在教程后面更加清楚。为了解析一个任意的基本表达式，我们需要确定它是哪种表达式：

```cpp
/// primary
///   ::= identifierexpr
///   ::= numberexpr
///   ::= parenexpr
static std::unique_ptr<ExprAST> ParsePrimary() {
  switch (CurTok) {
  default:
    return LogError("unknown token when expecting an expression");
  case tok_identifier:
    return ParseIdentifierExpr();
  case tok_number:
    return ParseNumberExpr();
  case '(':
    return ParseParenExpr();
  }
}
```

看到这个函数的定义后，为什么我们可以假设各个函数中的CurTok状态就变得更加明显了。这个函数使用前瞻来确定正在检查的是哪种表达式，然后通过函数调用来解析它。

现在基本表达式已经处理完毕，我们需要处理二元表达式。它们稍微复杂一些。

## 2.5. 二元表达式解析
二元表达式更难解析，因为它们通常是模棱两可的。例如，给定字符串“x+y\*z”，解析器可以选择将其解析为“(x+y)\*z”或“x+(y*z)”。根据数学中的常见定义，我们期望后者解析，因为“\*”（乘法）优先于“+”（加法）。

有很多方法可以处理这个问题，但一种优雅而高效的方法是使用运算符优先级解析。这种解析技术使用二元运算符的优先级来指导递归。首先，我们需要一个优先级表：

```cpp
/// BinopPrecedence - 这包含了定义的每个二元运算符的优先级。
static std::map<char, int> BinopPrecedence;

/// GetTokPrecedence - 获取挂起的二元运算符标记的优先级。
static int GetTokPrecedence() {
  if (!isascii(CurTok))
    return -1;

  // 确保这是一个已声明的二元运算符。
  int TokPrec = BinopPrecedence[CurTok];
  if (TokPrec <= 0) return -1;
  return TokPrec;
}

int main() {
  // 安装标准二元运算符。
  // 1是最低优先级。
  BinopPrecedence['<'] = 10;
  BinopPrecedence['+'] = 20;
  BinopPrecedence['-'] = 20;
  BinopPrecedence['*'] = 40;  // 最高。
  ...
}
```

对于Kaleidoscope的基本形式，我们只支持4个二元运算符（显然可以由您来扩展，我们勇敢无畏的读者）。GetTokPrecedence函数返回当前标记的优先级，如果标记不是二元运算符，则返回-1。拥有一个映射使得添加新运算符变得容易，并且明确了算法不依赖于所涉及的具体运算符，但消除映射并在GetTokPrecedence函数中进行比较也很容易。（或者只是使用一个固定大小的数组）。

有了上面的帮助函数定义，我们现在可以开始解析二元表达式了。运算符优先级解析的基本思想是将一个可能有歧义的二元运算符表达式分解成部分。考虑例如表达式“a+b+(c+d)*e*f+g”。运算符优先级解析将其视为由二元运算符分隔的一串基本表达式。因此，它将首先解析前导基本表达式“a”，然后它将看到[+, b] [+, (c+d)] [*, e] [*, f]和[+, g]对。注意，因为括号是基本表达式，二元表达式解析器根本不需要担心嵌套子表达式如(c+d)。

首先，一个表达式是一个基本表达式，后面可能跟着一系列[binop,基本expr]对：

```cpp
/// expression
///   ::= primary binoprhs
///
static std::unique_ptr<ExprAST> ParseExpression() {
  auto LHS = ParsePrimary();
  if (!LHS)
    return nullptr;

  return ParseBinOpRHS(0, std::move(LHS));
}
```

ParseBinOpRHS是为我们解析这对序列的函数。它接受一个优先级和一个指向到目前为止解析的部分的表达式的指针。注意“x”是一个完全有效的表达式：因此，“binoprhs”可以为空，在这种情况下它返回传入的表达式。在上面的例子中，代码传递“a”的表达式到ParseBinOpRHS，并且当前标记是“+”。

传递给ParseBinOpRHS的优先级值表示函数被允许吃掉的最小运算符优先级。例如，如果当前对流是[+, x]并且ParseBinOpRHS被传递了一个优先级为40的值，它将不会消耗任何标记（因为‘+’的优先级只有20）。考虑到这一点，ParseBinOpRHS从以下开始：

```cpp
/// binoprhs
///   ::= ('+' primary)*
static std::unique_ptr<ExprAST> ParseBinOpRHS(int ExprPrec,
                                              std::unique_ptr<ExprAST> LHS) {
  // 如果这是一个二元运算符，找到它的优先级。
  while (true) {
    int TokPrec = GetTokPrecedence();

    // 如果这是一个优先级至少与当前二元运算符绑定的二元运算符，
    // 消耗它，否则我们就完成了。
    if (TokPrec < ExprPrec)
      return LHS;
```

这段代码获取当前标记的优先级，并检查它是否太低。由于我们定义无效标记的优先级为-1，这个检查隐式地知道当标记流中的二元运算符用尽时，对流结束。如果这个检查成功，我们知道该标记是一个二元运算符，并且它将包含在这个表达式中：

```cpp
// 好的，我们知道这是一个二元运算符。
int BinOp = CurTok;
getNextToken();  // 吃掉二元运算符

// 解析二元运算符后的基本表达式。
auto RHS = ParsePrimary();
if (!RHS)
  return nullptr;
```

这样，这段代码吃掉（并记住）二元运算符，然后解析后面的基本表达式。这建立了整个对，第一个对是[+, b]，对于运行中的例子。

现在我们解析了表达式的左边部分和一对RHS序列，我们必须决定表达式的结合方式。特别是，我们可以有“(a+b) binop 未解析”或“a + (b binop 未解析)”。为此，我们查看“binop”的优先级并将其与BinOp的优先级（在这种情况下是‘+’）进行比较：

```cpp
// 如果BinOp与RHS的绑定不如RHS后的运算符紧密，让
// 挂起的运算符将RHS作为其LHS。
int NextPrec = GetTokPrecedence();
if (TokPrec < NextPrec) {
```

如果右边的binop的优先级低于或等于我们当前解析的运算符的优先级，那么

我们知道括号的结合方式为“(a+b) binop …”。在我们的例子中，当前运算符是“+”，下一个运算符是“+”，我们知道它们具有相同的优先级。在这种情况下，我们将创建“a+b”的AST节点，然后继续解析：

```cpp
      ... if body omitted ...
    }

    // 合并LHS/RHS。
    LHS = std::make_unique<BinaryExprAST>(BinOp, std::move(LHS),
                                           std::move(RHS));
  }  // 循环回到while循环的顶部。
}
```

在上面的例子中，这将把“a+b+”变成“(a+b)”并执行while循环的下一次迭代，将“+”作为当前标记。上面的代码将吃掉，记住，并解析“(c+d)”作为基本表达式，这使当前对等于[+, (c+d)]。然后它将评估带有“*”作为基本右边运算符的if条件。在这种情况下，“*”的优先级高于“+”的优先级，因此if条件将被进入。

剩下的关键问题是“if条件如何解析完整的右边表达式”？特别是，为了正确地构建我们的例子的AST，它需要将所有的“(c+d)*e*f”作为RHS表达式变量。这样做的代码出奇地简单（上面两个代码块的代码为了上下文重复）：

```cpp
// 如果BinOp与RHS的绑定不如RHS后的运算符紧密，让
// 挂起的运算符将RHS作为其LHS。
int NextPrec = GetTokPrecedence();
if (TokPrec < NextPrec) {
  RHS = ParseBinOpRHS(TokPrec+1, std::move(RHS));
  if (!RHS)
    return nullptr;
}
// 合并LHS/RHS。
LHS = std::make_unique<BinaryExprAST>(BinOp, std::move(LHS),
                                       std::move(RHS));
}  // 循环回到while循环的顶部。
```

在这一点上，我们知道右边的二元运算符的优先级高于我们当前正在解析的binop。因此，我们知道任何运算符优先级高于“+”的对序列应该一起解析并返回为“RHS”。为此，我们递归调用ParseBinOpRHS函数，指定“TokPrec+1”作为其继续的最小优先级。在上面的例子中，这将导致其返回“(c+d)*e*f”的AST节点作为RHS，然后将其设置为‘+’表达式的RHS。

最后，在while循环的下一次迭代中，“+g”部分被解析并添加到AST。通过这14行非平凡代码，我们正确地处理了完全通用的二元表达式解析。这个代码非常细微，需要仔细理解。我建议用一些复杂的例子运行它，以了解它的工作原理。

这就完成了表达式的处理。到此为止，我们可以将解析器指向一个任意的标记流，并从中构建一个表达式，停在第一个不属于表达式的标记处。接下来，我们需要处理函数定义等。

## 2.6. 解析其余部分
接下来缺少的是函数原型的处理。在Kaleidoscope中，它们既用于'extern'函数声明，也用于函数体定义。处理这段代码是直截了当的，不是很有趣（当你已经处理了表达式）：

```cpp
/// prototype
///   ::= id '(' id* ')'
static std::unique_ptr<PrototypeAST> ParsePrototype() {
  if (CurTok != tok_identifier)
    return LogErrorP("Expected function name in prototype");

  std::string FnName = IdentifierStr;
  getNextToken();

  if (CurTok != '(')
    return LogErrorP("Expected '(' in prototype");

  // 读取参数名列表。
  std::vector<std::string> ArgNames;
  while (getNextToken() == tok_identifier)
    ArgNames.push_back(IdentifierStr);
  if (CurTok != ')')
    return LogErrorP("Expected ')' in prototype");

  // 成功。
  getNextToken();  // 吃掉')'。

  return std::make_unique<PrototypeAST>(FnName, std::move(ArgNames));
}
```

有了这些，一个函数定义就非常简单了，只需一个原型加上一个实现主体的表达式：

```cpp
/// definition ::= 'def' prototype expression
static std::unique_ptr<FunctionAST> ParseDefinition() {
  getNextToken();  // 吃掉def。
  auto Proto = ParsePrototype();
  if (!Proto) return nullptr;

  if (auto E = ParseExpression())
    return std::make_unique<FunctionAST>(std::move(Proto), std::move(E));
  return nullptr;
}
```

此外，我们支持'extern'来声明诸如'sin'和'cos'的函数，并支持用户函数的前向声明。这些'extern'只是没有主体的原型：

```cpp
/// external ::= 'extern' prototype
static std::unique_ptr<PrototypeAST> ParseExtern() {
  getNextToken();  // 吃掉extern。
  return ParsePrototype();
}
```

最后，我们还允许用户输入任意顶级表达式并即时评估。我们通过定义匿名的无参（零参数）函数来处理这一点：

```cpp
/// toplevelexpr ::= expression
static std::unique_ptr<FunctionAST> ParseTopLevelExpr() {
  if (auto E = ParseExpression()) {
    // 创建一个匿名原型。
    auto Proto = std::make_unique<PrototypeAST>("", std::vector<std::string>());
    return std::make_unique<FunctionAST>(std::move(Proto), std::move(E));
  }
  return nullptr;
}
```

现在我们有了所有的部分，让我们构建一个小的驱动程序，让我们实际执行我们构建的代码！

## 2.7. 驱动程序
驱动程序简单地调用所有的解析部分，并带有一个顶级调度循环。这里没有太多有趣的地方，所以我只包括顶级循环。完整代码见“顶级解析”部分。

```cpp
/// top ::= definition | external | expression | ';'
static void MainLoop() {
  while (true) {
    fprintf(stderr, "ready> ");
    switch (CurTok) {
    case tok_eof:
      return;
    case ';': // 忽略顶级分号。
      getNextToken();
      break;
    case tok_def:
      HandleDefinition();
      break;
    case tok_extern:
      HandleExtern();
      break;
    default:
      HandleTopLevelExpression();
      break;
    }
  }
}
```

最有趣的部分是我们忽略了顶级分号。为什么会这样，你问？基本原因是如果你在命令行中输入“4 + 5”，解析器不知道这是你输入的结尾还是不是。例如，在下一行你可以输入“def foo…”，在这种情况下4+5是一个顶级表达式的结尾。或者你可以输入“* 6”，这会继续表达式。使用顶级分号允许你输入“4+5;”，解析器会知道你完成了。

## 2.8. 结论
在不到400行带注释的代码（240行非注释、非空白代码）中，我们完全定义了我们的最小语言，包括一个词法分析器、解析器和AST构建器。完成这些后，可执行文件将验证Kaleidoscope代码，并告诉我们它是否在语法上无效。例如，这里是一个示例交互：

```
$ ./a.out
ready> def foo(x y) x+foo(y, 4.0);
Parsed a function definition.
ready> def foo(x y) x+y y;
Parsed a function definition.
Parsed a top-level expr
ready> def foo(x y) x+y );
Parsed a function definition.
Error: unknown token when expecting an expression
ready> extern sin(a);
ready> Parsed an extern
ready> ^D
$
```

这里有很大的扩展空间。你可以定义新的AST节点，以多种方式扩展语言等。在下一章中，我们将介绍如何从AST生成LLVM中间表示（IR）。