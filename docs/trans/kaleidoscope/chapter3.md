## 3.1. 第三章简介
欢迎阅读“使用LLVM实现语言”教程的第三章。本章将向您展示如何将第二章中构建的抽象语法树转换为LLVM IR。这将使您了解LLVM的一些工作原理，并展示使用LLVM是多么容易。构建词法分析器和解析器比生成LLVM IR代码要复杂得多。:)

请注意：本章及后续章节的代码要求使用LLVM 3.7或更高版本。LLVM 3.6及之前的版本将无法与之配合使用。另外，请确保您使用的教程版本与您的LLVM版本匹配：如果您使用的是官方LLVM版本，请使用发布版本中包含的文档或在llvm.org发布页面上的文档。

## 3.2. 代码生成设置
为了生成LLVM IR，我们需要一些简单的设置来开始。首先，我们在每个AST类中定义虚拟的代码生成（codegen）方法：

```cpp
/// ExprAST - 所有表达式节点的基类。
class ExprAST {
public:
  virtual ~ExprAST() = default;
  virtual Value *codegen() = 0;
};

/// NumberExprAST - 用于数字字面量（如"1.0"）的表达式类。
class NumberExprAST : public ExprAST {
  double Val;

public:
  NumberExprAST(double Val) : Val(Val) {}
  Value *codegen() override;
};
...
```

codegen()方法用于为该AST节点及其依赖的所有内容生成IR，并返回一个LLVM Value对象。Value是用于表示LLVM中的“静态单赋值（SSA）寄存器”或“SSA值”的类。SSA值的最显著特点是，它的值在相关指令执行时计算，且只有在指令重新执行时（如果有的话）才会获得新值。换句话说，没有办法“更改”SSA值。欲了解更多信息，请阅读静态单赋值（SSA）——一旦您掌握了这些概念，它们真的很自然。

注意，除了在ExprAST类层次结构中添加虚拟方法之外，还可以使用访问者模式或其他方式来建模。再次声明，本教程不会过多关注良好的软件工程实践：对于我们的目的，添加虚拟方法是最简单的。

第二件事是我们需要一个类似于解析器中使用的“LogError”方法，用于报告代码生成过程中发现的错误（例如，使用未声明的参数）：

```cpp
static std::unique_ptr<LLVMContext> TheContext;
static std::unique_ptr<IRBuilder<>> Builder(TheContext);
static std::unique_ptr<Module> TheModule;
static std::map<std::string, Value *> NamedValues;

Value *LogErrorV(const char *Str) {
  LogError(Str);
  return nullptr;
}
```

这些静态变量将在代码生成期间使用。TheContext是一个不透明对象，拥有许多核心LLVM数据结构，例如类型和常量值表。我们不需要详细了解它，只需要一个实例传递给需要它的API。

Builder对象是一个帮助对象，使生成LLVM指令变得容易。IRBuilder类模板的实例跟踪当前插入指令的位置，并具有创建新指令的方法。

TheModule是一个包含函数和全局变量的LLVM构造。在许多方面，它是LLVM IR用来包含代码的顶级结构。它将拥有我们生成的所有IR的内存，这就是为什么codegen()方法返回一个原始的Value*，而不是一个unique_ptr<Value>。

NamedValues映射记录当前作用域中定义的值及其LLVM表示。（换句话说，它是代码的符号表）。在这种形式的Kaleidoscope中，唯一可以引用的是函数参数。因此，生成函数体代码时，函数参数将出现在此映射中。

有了这些基础，我们可以开始讨论如何为每个表达式生成代码。注意，这假设Builder已经设置好以生成代码。现在，我们假设这已经完成，我们将使用它来发出代码。

## 3.3. 表达式代码生成
为表达式节点生成LLVM代码非常简单：我们所有四个表达式节点的注释代码总共不到45行。首先，我们将处理数字字面量：

```cpp
Value *NumberExprAST::codegen() {
  return ConstantFP::get(*TheContext, APFloat(Val));
}
```

在LLVM IR中，数字常量用ConstantFP类表示，该类在内部使用APFloat保存数字值（APFloat具有保存任意精度浮点常量的能力）。这段代码基本上只是创建并返回一个ConstantFP。请注意，在LLVM IR中，所有常量都是唯一的，并且是共享的。因此，API使用“foo::get(...)”习语，而不是“new foo(..)”或“foo::Create(..)”。

```cpp
Value *VariableExprAST::codegen() {
  // 在函数中查找该变量。
  Value *V = NamedValues[Name];
  if (!V)
    LogErrorV("Unknown variable name");
  return V;
}
```

使用LLVM引用变量也很简单。在Kaleidoscope的简单版本中，我们假设变量已经在某处发出，并且其值是可用的。实际上，NamedValues映射中唯一可以存在的值是函数参数。此代码仅检查指定名称是否在映射中（如果没有，则引用了未知变量）并返回其值。在后续章节中，我们将为符号表中的循环诱导变量和局部变量添加支持。

```cpp
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
    // 将布尔值0/1转换为双精度0.0或1.0
    return Builder->CreateUIToFP(L, Type::getDoubleTy(*TheContext),
                                 "booltmp");
  default:
    return LogErrorV("invalid binary operator");
  }
}
```

二元运算符开始变得更有趣。基本思想是我们递归地为表达式的左侧生成代码，然后为右侧生成代码，然后计算二元表达式的结果。在此代码中，我们对操作码进行简单的switch，以创建正确的LLVM指令。

在上面的示例中，LLVM builder类开始显示其价值。IRBuilder知道在哪里插入新创建的指令，您只需指定要创建的指令（例如，使用CreateFAdd）、要使用的操作数（此处为L和R）以及可选地为生成的指令提供名称。

关于LLVM的一件好事是，名称只是一个提示。例如，如果上述代码发出多个“addtmp”变量，LLVM将自动为每个变量提供一个递增的唯一数字后缀。指令的局部值名称纯属可选，但它使阅读IR转储变得容易得多。

LLVM指令受到严格规则的约束：例如，加法指令的左操作数和右操作数必须具有相同的类型，并且加法的结果类型必须与操作数类型匹配。因为Kaleidoscope中的所有值都是双精度的，这使得加法、减法和乘法的代码非常简单。

另一方面，LLVM规定fcmp指令始终返回'i1'值（一位整数）。问题在于Kaleidoscope希望该值为0.0或1.0。为了获得这些语义，我们将fcmp指令与uitofp指令结合使用。该指令通过将输入视为无符号值来将其输入整数转换为浮点值。相比之下，如果我们使用sitofp指令，Kaleidoscope的'<'运算符将根据输入值返回0.0和-1.0。

```cpp
Value *CallExprAST::codegen() {
  // 在全局模块表中查找名称。
  Function *CalleeF = TheModule->getFunction(Callee);
  if (!CalleeF)
    return LogErrorV("Unknown function referenced");

  // 如果参数数量不匹配，报错。
  if (CalleeF->arg_size() != Args.size())
    return LogErrorV("Incorrect # arguments passed");

  std::vector<Value *> ArgsV;
  for (unsigned i = 0, e = Args.size(); i != e; ++i) {
    ArgsV.push_back(Args[i]->codegen());
    if (!ArgsV.back())
      return nullptr;
  }

  return Builder->CreateCall(CalleeF, ArgsV, "calltmp");
}
```

函数调用的代码生成在LLVM中非常简单。上面的代码最初在LLVM模块的符号表中进行函数名称查找。回想一下，LLVM模块是保存我们正在JIT的函数的容器。通过给每个函数赋予用户指定的相同名称，我们可以使用LLVM符号表来解析函数名称。

一旦我们有了要调用的函数，我们递归地生成每个要传递的参数的

代码，并创建一个LLVM调用指令。请注意，LLVM默认使用本机C调用约定，这允许这些调用也调用标准库函数，如“sin”和“cos”，无需额外的努力。

这就完成了我们目前在Kaleidoscope中处理的四个基本表达式的处理。请随意添加更多表达式。例如，通过浏览LLVM语言参考，您会发现其他一些有趣的指令，这些指令非常容易插入我们的基本框架。

## 3.4. 函数代码生成
原型和函数的代码生成必须处理许多细节，这使得它们的代码不如表达式代码生成那么优美，但可以帮助我们说明一些重要点。首先，让我们讨论原型的代码生成：它们既用于函数体，也用于外部函数声明。代码开始如下：

```cpp
Function *PrototypeAST::codegen() {
  // 制作函数类型：double(double,double)等。
  std::vector<Type*> Doubles(Args.size(),
                             Type::getDoubleTy(*TheContext));
  FunctionType *FT =
    FunctionType::get(Type::getDoubleTy(*TheContext), Doubles, false);

  Function *F =
    Function::Create(FT, Function::ExternalLinkage, Name, TheModule.get());
```

这段代码在几行中包含了大量的功能。首先注意，该函数返回一个“Function*”而不是“Value*”。因为“原型”实际上是讨论函数的外部接口（而不是表达式计算的值），所以当代码生成时，它返回相应的LLVM函数是合理的。

对FunctionType::get的调用创建了给定原型应使用的FunctionType。由于Kaleidoscope中的所有函数参数都是双精度的，第一行创建了一个“N”个LLVM双精度类型的向量。然后它使用FunctionType::get方法创建一个函数类型，该类型接受“N”个双精度作为参数，返回一个双精度作为结果，并且不是可变参数（false参数表示这一点）。请注意，LLVM中的类型就像常量一样是唯一的，因此您不会“new”一个类型，而是“get”它。

最后一行实际上创建了与原型对应的IR函数。这指示了类型、链接和名称的使用，以及插入的模块。“外部链接”意味着该函数可以在当前模块之外定义和/或可由模块外的函数调用。传递的名称是用户指定的名称：由于指定了“TheModule”，此名称在“TheModule”的符号表中注册。

```cpp
// 为所有参数设置名称。
unsigned Idx = 0;
for (auto &Arg : F->args())
  Arg.setName(Args[Idx++]);

return F;
```

最后，我们根据原型中给出的名称设置每个函数参数的名称。这一步不是严格必要的，但保持名称一致使IR更具可读性，并允许后续代码直接引用参数的名称，而不是必须在原型AST中查找它们。

此时，我们有一个没有主体的函数原型。这就是LLVM IR表示函数声明的方式。对于Kaleidoscope中的extern语句，这就是我们需要做的全部工作。然而，对于函数定义，我们需要生成代码并附加一个函数体。

```cpp
Function *FunctionAST::codegen() {
  // 首先，检查是否存在先前'extern'声明的函数。
  Function *TheFunction = TheModule->getFunction(Proto->getName());

  if (!TheFunction)
    TheFunction = Proto->codegen();

  if (!TheFunction)
    return nullptr;

  if (!TheFunction->empty())
    return (Function*)LogErrorV("Function cannot be redefined.");
```

对于函数定义，我们首先在TheModule的符号表中搜索该函数的现有版本，以防之前已使用‘extern’语句创建。如果Module::getFunction返回null，则不存在以前的版本，因此我们将从原型生成代码。在任何一种情况下，我们都要断言该函数为空（即还没有主体）才能开始。

```cpp
// 创建一个新的基本块开始插入。
BasicBlock *BB = BasicBlock::Create(*TheContext, "entry", TheFunction);
Builder->SetInsertPoint(BB);

// 在NamedValues映射中记录函数参数。
NamedValues.clear();
for (auto &Arg : TheFunction->args())
  NamedValues[std::string(Arg.getName())] = &Arg;
```

现在我们到了设置Builder的地方。第一行创建一个新的基本块（名为“entry”），插入到TheFunction中。第二行然后告诉builder新指令应该插入到新基本块的末尾。LLVM中的基本块是定义控制流图的重要部分。由于我们没有任何控制流，目前我们的函数只包含一个块。我们将在第5章解决这个问题:).

接下来，我们将函数参数添加到NamedValues映射中（首先清除它）以便VariableExprAST节点可以访问它们。

```cpp
if (Value *RetVal = Body->codegen()) {
  // 完成函数。
  Builder->CreateRet(RetVal);

  // 验证生成的代码，检查一致性。
  verifyFunction(*TheFunction);

  return TheFunction;
}
```

一旦设置了插入点并填充了NamedValues映射，我们调用函数根表达式的codegen()方法。如果没有发生错误，这会发出代码以将表达式计算到入口块中，并返回计算出的值。假设没有错误，我们然后创建一个LLVM ret指令，完成函数。函数构建完成后，我们调用LLVM提供的verifyFunction。此函数对生成的代码执行各种一致性检查，以确定我们的编译器是否正确地完成所有任务。使用此功能很重要：它可以捕捉很多错误。函数完成并验证后，我们返回它。

```cpp
  // 读取主体时出错，删除函数。
  TheFunction->eraseFromParent();
  return nullptr;
}
```

剩下的唯一部分是错误处理。为简单起见，我们通过删除使用eraseFromParent方法生成的函数来处理这一点。这允许用户重新定义他们之前输入错误的函数：如果我们不删除它，它将保留在符号表中，并具有主体，阻止将来的重新定义。

不过，这段代码确实存在一个错误：如果FunctionAST::codegen()方法找到一个现有的IR函数，它不会验证其签名是否与定义的原型匹配。这意味着较早的‘extern’声明将优先于函数定义的签名，这可能导致代码生成失败，例如，如果函数参数的名称不同。有很多方法可以修复此错误，看看您能提出什么！以下是一个测试用例：

```cpp
extern foo(a);     # ok，定义foo。
def foo(b) b;      # 错误：未知变量名。（使用'a'的声明优先）。
```

## 3.5. 驱动程序更改和结论
目前，生成LLVM代码对我们没有太大意义，除了我们可以查看漂亮的IR调用。示例代码将代码生成调用插入到“HandleDefinition”、“HandleExtern”等函数中，然后转储出LLVM IR。这提供了一种查看简单函数的LLVM IR的好方法。例如：

```cpp
ready> 4+5;
Read top-level expression:
define double @0() {
entry:
  ret double 9.000000e+00
}
```

注意解析器如何为我们将顶级表达式转换为匿名函数。这在下一章添加JIT支持时会很有用。还要注意，代码被非常字面地转录，除了IRBuilder执行的简单常量折叠外，没有进行任何优化。我们将在下一章中显式添加优化。

```cpp
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

这显示了一些简单的算术运算。注意与我们用于创建指令的LLVM builder调用的惊人相似之处。

```cpp
ready> def bar(a) foo(a, 4.0) + bar(31337);
Read function definition:
define double @bar(double %a) {
entry:
  %calltmp = call double @foo(double %a, double 4.000000e+00)
  %calltmp1 = call double @bar(double 3.133700e+04)
  %addtmp = fadd double %calltmp, %calltmp1
  ret double %addtmp
}
```

这显示了一些函数调用。注意，如果调用它，该函数将需要很长时间才能执行。在将来，我们将添加条件控制流，以实际使递归有用:).

```cpp
ready> extern cos(x);
Read extern:
declare double @cos(double)

ready> cos(1.234);
Read top-level expression:
define double @1() {
entry:
  %calltmp = call double @cos(double 1.234000e+00)
  ret

 double %calltmp
}
```

这显示了libm的“cos”函数的extern声明，以及对它的调用。

```cpp
ready> ^D
; ModuleID = 'my cool jit'

define double @0() {
entry:
  %addtmp = fadd double 4.000000e+00, 5.000000e+00
  ret double %addtmp
}

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

define double @bar(double %a) {
entry:
  %calltmp = call double @foo(double %a, double 4.000000e+00)
  %calltmp1 = call double @bar(double 3.133700e+04)
  %addtmp = fadd double %calltmp, %calltmp1
  ret double %addtmp
}

declare double @cos(double)

define double @1() {
entry:
  %calltmp = call double @cos(double 1.234000e+00)
  ret double %calltmp
}
```

当您退出当前演示时（通过在Linux上发送EOF，或在Windows上按CTRL+Z并按ENTER），它会转储出生成的整个模块的IR。在这里，您可以看到所有相互引用的函数的大图。

这就结束了Kaleidoscope教程的第三章。接下来，我们将描述如何添加JIT代码生成和优化器支持，以便我们可以实际开始运行代码！