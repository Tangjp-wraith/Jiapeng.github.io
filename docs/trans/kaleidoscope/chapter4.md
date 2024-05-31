## 4.1. 第四章简介

欢迎阅读“使用LLVM实现语言”教程的第四章。第一至三章描述了一个简单语言的实现，并添加了生成LLVM IR的支持。本章介绍两种新技术：为您的语言添加优化器支持，以及添加JIT编译器支持。这些附加功能将展示如何为Kaleidoscope语言获得漂亮、高效的代码。

## 4.2. 简单的常量折叠

我们在第三章中的演示优雅且易于扩展。不幸的是，它不会生成出色的代码。然而，IRBuilder在编译简单代码时确实提供了明显的优化：

```
ready> def test(x) 1+2+x;
Read function definition:
define double @test(double %x) {
entry:
        %addtmp = fadd double 3.000000e+00, %x
        ret double %addtmp
}
```

这段代码不是通过解析输入构建的AST的逐字转录。逐字转录的结果如下：

```
ready> def test(x) 1+2+x;
Read function definition:
define double @test(double %x) {
entry:
        %addtmp = fadd double 2.000000e+00, 1.000000e+00
        %addtmp1 = fadd double %addtmp, %x
        ret double %addtmp1
}
```

如上所示，常量折叠是一种非常常见且非常重要的优化：以至于许多语言实现者在其AST表示中实现了常量折叠支持。

使用LLVM，您不需要在AST中支持这一点。由于生成LLVM IR的所有调用都通过LLVM IR builder进行，builder本身会检查在调用它时是否存在常量折叠机会。如果是，它只需进行常量折叠并返回常量，而不是创建指令。

这很简单 :)。实际上，我们建议在生成类似代码时始终使用IRBuilder。它的使用没有“语法开销”（您不必在编译器中到处进行常量检查），在某些情况下（特别是对于具有宏预处理器或使用大量常量的语言），它可以显著减少生成的LLVM IR的数量。

另一方面，IRBuilder的局限在于它在构建代码时内联进行所有分析。如果您使用稍微复杂的示例：

```
ready> def test(x) (1+2+x)*(x+(1+2));
ready> Read function definition:
define double @test(double %x) {
entry:
        %addtmp = fadd double 3.000000e+00, %x
        %addtmp1 = fadd double %x, 3.000000e+00
        %multmp = fmul double %addtmp, %addtmp1
        ret double %multmp
}
```

在这种情况下，乘法的LHS和RHS是相同的值。我们真正想要看到的是生成“tmp = x+3; result = tmp*tmp;”，而不是两次计算“x+3”。

不幸的是，没有任何本地分析能够检测和纠正这一点。这需要两种转换：表达式的重关联（使add的词法相同）和公共子表达式消除（CSE）以删除冗余的add指令。幸运的是，LLVM提供了多种优化，您可以使用这些优化，这些优化以“passes”的形式提供。

## 4.3. LLVM优化Passes

LLVM提供了许多优化passes，它们执行各种不同的操作并具有不同的权衡。与其他系统不同，LLVM不坚持一个优化集适合所有语言和所有情况的错误观念。LLVM允许编译器实现者完全决定使用哪些优化，按什么顺序以及在什么情况下使用。

例如，LLVM支持“全模块”passes，这些passes会跨越尽可能多的代码（通常是一个文件，但如果在链接时运行，这可以是整个程序的一大部分）。它还支持并包括仅对单个函数进行操作的“每函数”passes，而不查看其他函数。有关passes及其运行方式的更多信息，请参见如何编写一个Pass文档和LLVM Passes列表。

对于Kaleidoscope，我们目前正在生成函数，当用户输入时一次生成一个。我们在这种设置中并不追求最终的优化体验，但我们也希望尽可能捕捉到简单快捷的内容。因此，我们将选择在用户输入函数时运行一些每函数优化。如果我们想制作一个“静态Kaleidoscope编译器”，我们将使用我们现在的代码，只是我们会推迟运行优化器，直到整个文件解析完毕。

除了函数和模块passes之间的区别外，passes还可以分为转换和分析passes。转换passes会改变IR，而分析passes会计算其他passes可以使用的信息。为了添加一个转换pass，必须预先注册所有它依赖的分析passes。

为了启动每函数优化，我们需要设置一个FunctionPassManager来保存和组织我们要运行的LLVM优化。一旦我们有了这个，我们可以添加一组要运行的优化。我们需要为每个要优化的模块创建一个新的FunctionPassManager，因此我们将添加到上一章创建的函数（InitializeModule()）中：

```cpp
void InitializeModuleAndManagers(void) {
  // 打开一个新的上下文和模块。
  TheContext = std::make_unique<LLVMContext>();
  TheModule = std::make_unique<Module>("KaleidoscopeJIT", *TheContext);
  TheModule->setDataLayout(TheJIT->getDataLayout());

  // 为模块创建一个新的builder。
  Builder = std::make_unique<IRBuilder<>>(*TheContext);

  // 创建新的pass和分析管理器。
  TheFPM = std::make_unique<FunctionPassManager>();
  TheLAM = std::make_unique<LoopAnalysisManager>();
  TheFAM = std::make_unique<FunctionAnalysisManager>();
  TheCGAM = std::make_unique<CGSCCAnalysisManager>();
  TheMAM = std::make_unique<ModuleAnalysisManager>();
  ThePIC = std::make_unique<PassInstrumentationCallbacks>();
  TheSI = std::make_unique<StandardInstrumentations>(*TheContext,
                                                    /*DebugLogging*/ true);
  TheSI->registerCallbacks(*ThePIC, TheMAM.get());
  ...
```

在初始化全局模块TheModule和FunctionPassManager后，我们需要初始化框架的其他部分。四个AnalysisManagers允许我们添加跨越IR层次结构四个层次的分析passes。PassInstrumentationCallbacks和StandardInstrumentations是pass instrumentation框架所需的，它允许开发人员自定义passes之间发生的事情。

一旦设置好这些管理器，我们使用一系列的“addPass”调用来添加一堆LLVM转换passes：

```cpp
// 添加转换passes。
// 进行简单的“窥孔”优化和位操作优化。
TheFPM->addPass(InstCombinePass());
// 重新关联表达式。
TheFPM->addPass(ReassociatePass());
// 消除公共子表达式。
TheFPM->addPass(GVNPass());
// 简化控制流图（删除不可达块等）。
TheFPM->addPass(SimplifyCFGPass());
```

在这种情况下，我们选择添加四个优化passes。我们选择的这些passes是一组非常标准的“清理”优化，适用于各种代码。我不会深入探讨它们的具体功能，但相信我，它们是一个很好的起点:）。

接下来，我们注册转换passes所使用的分析passes。

```cpp
  // 注册这些转换passes中使用的分析passes。
  PassBuilder PB;
  PB.registerModuleAnalyses(*TheMAM);
  PB.registerFunctionAnalyses(*TheFAM);
  PB.crossRegisterProxies(*TheLAM, *TheFAM, *TheCGAM, *TheMAM);
}
```

一旦设置好PassManager，我们需要使用它。在我们新创建的函数构建后（在FunctionAST::codegen()中），但在返回给客户端之前运行它：

```cpp
if (Value *RetVal = Body->codegen()) {
  // 完成函数。
  Builder.CreateRet(RetVal);

  // 验证生成的代码，检查一致性。
  verifyFunction(*TheFunction);

  // 优化函数。
  TheFPM->run(*TheFunction, *TheFAM);

  return TheFunction;
}
```

如您所见，这非常简单。FunctionPassManager优化并更新LLVM Function*，在某些情况下（希望）改进其主体。这样一来，我们可以再次尝试上面的测试：

```
ready> def test(x) (1+2+x)*(x+(1+2));
ready> Read function definition:
define double @test(double %x) {
entry:
        %addtmp = fadd double %x, 3.000000e+00
       

%multmp = fmul double %addtmp, %addtmp
        ret double %multmp
}
```

正如预期的那样，我们现在得到了优化后的代码，从每次执行此函数中节省了一个浮点加法指令。

LLVM 提供了各种各样的优化，可以在特定情况下使用。关于各种passes的一些文档可用，但并不完整。另一个好的灵感来源是查看Clang运行的passes。`opt` 工具允许您从命令行试验passes，因此您可以查看它们是否有效。

既然我们有了合理的代码从前端出来，让我们谈谈执行它！

## 4.4. 添加JIT编译器

可以对LLVM IR中可用的代码应用各种工具。例如，您可以对其运行优化（如上所述），可以以文本或二进制形式导出它，可以将代码编译为某个目标的汇编文件（.s），或者可以对其进行JIT编译。LLVM IR表示的一个好处是它是许多不同编译器部分之间的“通用货币”。

在本节中，我们将向解释器添加JIT编译器支持。Kaleidoscope的基本思路是让用户像现在这样输入函数体，但立即评估他们输入的顶级表达式。例如，如果他们输入“1 + 2;”，我们应该评估并输出3。如果他们定义一个函数，他们应该能够从命令行调用它。

为此，我们首先准备环境，以便为当前的本地目标创建代码，并声明和初始化JIT。通过调用一些InitializeNativeTarget*函数并添加一个全局变量TheJIT来完成此操作，并在main中初始化它：

```cpp
static std::unique_ptr<KaleidoscopeJIT> TheJIT;
...
int main() {
  InitializeNativeTarget();
  InitializeNativeTargetAsmPrinter();
  InitializeNativeTargetAsmParser();

  // 安装标准二元运算符。
  // 1是最低优先级。
  BinopPrecedence['<'] = 10;
  BinopPrecedence['+'] = 20;
  BinopPrecedence['-'] = 20;
  BinopPrecedence['*'] = 40; // 最高。

  // 初始化第一个标记。
  fprintf(stderr, "ready> ");
  getNextToken();

  TheJIT = std::make_unique<KaleidoscopeJIT>();

  // 现在运行主要的“解释器循环”。
  MainLoop();

  return 0;
}
```

我们还需要为JIT设置数据布局：

```cpp
void InitializeModuleAndPassManager(void) {
  // 打开一个新的上下文和模块。
  TheContext = std::make_unique<LLVMContext>();
  TheModule = std::make_unique<Module>("my cool jit", TheContext);
  TheModule->setDataLayout(TheJIT->getDataLayout());

  // 为模块创建一个新的builder。
  Builder = std::make_unique<IRBuilder<>>(*TheContext);

  // 创建一个新的与其关联的pass管理器。
  TheFPM = std::make_unique<legacy::FunctionPassManager>(TheModule.get());
  ...
```

KaleidoscopeJIT类是专门为这些教程构建的简单JIT，可在LLVM源代码的llvm-src/examples/Kaleidoscope/include/KaleidoscopeJIT.h中找到。在后续章节中，我们将了解它的工作原理并扩展其新功能，但目前我们将其视为给定的。其API非常简单：`addModule`将一个LLVM IR模块添加到JIT中，使其函数可供执行（其内存由ResourceTracker管理）；`lookup`允许我们查找编译代码的指针。

我们可以使用这个简单的API并更改解析顶级表达式的代码如下所示：

```cpp
static ExitOnError ExitOnErr;
...
static void HandleTopLevelExpression() {
  // 将顶级表达式评估为匿名函数。
  if (auto FnAST = ParseTopLevelExpr()) {
    if (FnAST->codegen()) {
      // 创建一个ResourceTracker来跟踪分配给我们匿名表达式的JIT内存——这样我们可以在执行后释放它。
      auto RT = TheJIT->getMainJITDylib().createResourceTracker();

      auto TSM = ThreadSafeModule(std::move(TheModule), std::move(TheContext));
      ExitOnErr(TheJIT->addModule(std::move(TSM), RT));
      InitializeModuleAndPassManager();

      // 在JIT中搜索__anon_expr符号。
      auto ExprSymbol = ExitOnErr(TheJIT->lookup("__anon_expr"));
      assert(ExprSymbol && "Function not found");

      // 获取符号的地址并将其转换为正确的类型（不带参数，返回双精度），以便我们可以将其作为本地函数调用。
      double (*FP)() = ExprSymbol.getAddress().toPtr<double (*)()>();
      fprintf(stderr, "Evaluated to %f\n", FP());

      // 从JIT中删除匿名表达式模块。
      ExitOnErr(RT->remove());
    }
```

如果解析和代码生成成功，下一步是将包含顶级表达式的模块添加到JIT中。通过调用`addModule`来实现此目的，该函数触发模块中所有函数的代码生成，并接受一个ResourceTracker，可用于稍后从JIT中删除模块。模块一旦添加到JIT中，就不能再修改，因此我们通过调用`InitializeModuleAndPassManager()`打开一个新模块来保存后续代码。

一旦将模块添加到JIT中，我们需要获取指向最终生成代码的指针。我们通过调用JIT的`lookup`方法并传递顶级表达式函数的名称`__anon_expr`来实现此目的。由于我们刚刚添加了这个函数，我们断言`lookup`返回了一个结果。

接下来，通过调用符号上的`getAddress()`来获取`__anon_expr`函数的内存地址。回想一下，我们将顶级表达式编译成一个不带参数且返回计算出的双精度的自包含LLVM函数。由于LLVM JIT编译器匹配本地平台ABI，这意味着您可以将结果指针直接转换为该类型的函数指针并直接调用它。这意味着，JIT编译的代码与静态链接到应用程序中的本地机器代码没有区别。

最后，由于我们不支持重新评估顶级表达式，我们在完成后从JIT中删除模块以释放关联的内存。回想一下，我们几行前创建的模块（通过`InitializeModuleAndPassManager`）仍然是打开的，等待新代码的添加。

只需这两个更改，让我们看看Kaleidoscope现在的工作情况！

```
ready> 4+5;
Read top-level expression:
define double @0() {
entry:
  ret double 9.000000e+00
}

Evaluated to 9.000000
```

好吧，这看起来基本上是可行的。函数的转储显示了我们为每个输入的顶级表达式合成的“无参数函数，总是返回双精度”。这展示了非常基本的功能，但我们能做得更多吗？

```
ready> def testfunc(x y) x + y*2;
Read function definition:
define double @testfunc(double %x, double %y) {
entry:
  %multmp = fmul double %y, 2.000000e+00
  %addtmp = fadd double %multmp, %x
  ret double %addtmp
}

ready> testfunc(4, 10);
Read top-level expression:
define double @1() {
entry:
  %calltmp = call double @testfunc(double 4.000000e+00, double 1.000000e+01)
  ret double %calltmp
}

Evaluated to 24.000000

ready> testfunc(5, 10);
ready> LLVM ERROR: Program used external function 'testfunc' which could not be resolved!
```

函数定义和调用也有效，但最后一行出了问题。调用看起来是有效的，那发生了什么？正如您可能猜到的那样，模块是JIT的分配单元，而`testfunc`是包含匿名表达式的同一模块的一部分。当我们从JIT中删除该模块以释放匿名表达式的内存时，我们也删除了`testfunc`的定义。然后，当我们尝试再次调用`testfunc`时，JIT无法再找到它。

修复此问题的最简单方法是将匿名表达式放在与其他函数定义不同的模块中。只要每个被调用的函数都有一个原型，并在调用之前被添加到JIT中，JIT将愉快地跨模块边界解析函数调用。通过将匿名表达式放在不同的模块中，我们可以删除它而不影响其余函数。

事实上，我们将更进一步，将每个函数放在其自己的模块中。这样做允许我们利用KaleidoscopeJIT的一个有用特性，使我们的环境更具REPL特性：函数可以多次添加到JIT中（与模块不同，模块中的每个函数必须具有唯一的定义）。当您在KaleidoscopeJIT中查找符号时，它将始终返回最新的定义：

```
ready> def foo(x

) x + 1;
Read function definition:
define double @foo(double %x) {
entry:
  %addtmp = fadd double %x, 1.000000e+00
  ret double %addtmp
}

ready> foo(2);
Evaluated to 3.000000

ready> def foo(x) x + 2;
define double @foo(double %x) {
entry:
  %addtmp = fadd double %x, 2.000000e+00
  ret double %addtmp
}

ready> foo(2);
Evaluated to 4.000000
```

为了让每个函数都能存在于自己的模块中，我们需要一种方法在我们打开的新模块中重新生成先前的函数声明：

```cpp
static std::unique_ptr<KaleidoscopeJIT> TheJIT;

...

Function *getFunction(std::string Name) {
  // 首先，看看该函数是否已添加到当前模块。
  if (auto *F = TheModule->getFunction(Name))
    return F;

  // 如果没有，请检查我们是否可以从某个现有原型生成声明。
  auto FI = FunctionProtos.find(Name);
  if (FI != FunctionProtos.end())
    return FI->second->codegen();

  // 如果不存在现有原型，则返回null。
  return nullptr;
}

...

Value *CallExprAST::codegen() {
  // 在全局模块表中查找名称。
  Function *CalleeF = getFunction(Callee);

...

Function *FunctionAST::codegen() {
  // 将原型的所有权转移到FunctionProtos映射，但保留一个引用以供下面使用。
  auto &P = *Proto;
  FunctionProtos[Proto->getName()] = std::move(Proto);
  Function *TheFunction = getFunction(P.getName());
  if (!TheFunction)
    return nullptr;
```

为了实现这一点，我们将首先添加一个新的全局变量FunctionProtos，它保存每个函数的最新原型。我们还将添加一个便捷方法`getFunction()`来替换对TheModule->getFunction()的调用。我们的便捷方法在TheModule中搜索现有的函数声明，如果找不到，则从FunctionProtos生成一个新的声明。在CallExprAST::codegen()中，我们只需要替换对TheModule->getFunction()的调用。在FunctionAST::codegen()中，我们需要首先更新FunctionProtos映射，然后调用getFunction()。这样一来，我们就可以在当前模块中为任何先前声明的函数始终获得一个函数声明。

我们还需要更新HandleDefinition和HandleExtern：

```cpp
static void HandleDefinition() {
  if (auto FnAST = ParseDefinition()) {
    if (auto *FnIR = FnAST->codegen()) {
      fprintf(stderr, "Read function definition:");
      FnIR->print(errs());
      fprintf(stderr, "\n");
      ExitOnErr(TheJIT->addModule(
          ThreadSafeModule(std::move(TheModule), std::move(TheContext))));
      InitializeModuleAndPassManager();
    }
  } else {
    // 跳过错误恢复的标记。
    getNextToken();
  }
}

static void HandleExtern() {
  if (auto ProtoAST = ParseExtern()) {
    if (auto *FnIR = ProtoAST->codegen()) {
      fprintf(stderr, "Read extern: ");
      FnIR->print(errs());
      fprintf(stderr, "\n");
      FunctionProtos[ProtoAST->getName()] = std::move(ProtoAST);
    }
  } else {
    // 跳过错误恢复的标记。
    getNextToken();
  }
}
```

在HandleDefinition中，我们添加了两行代码，将新定义的函数转移到JIT中并打开一个新模块。在HandleExtern中，我们只需添加一行代码，将原型添加到FunctionProtos。

**警告**

自LLVM-9起，不允许在单独的模块中重复符号。这意味着您不能像下面显示的那样在Kaleidoscope中重新定义函数。只需跳过此部分。

原因是新的OrcV2 JIT API试图非常接近静态和动态链接器规则，包括拒绝重复符号。要求符号名称唯一允许我们使用（唯一）符号名称作为键来跟踪符号的并发编译。

进行这些更改后，让我们再次尝试我们的REPL（这次我删除了匿名函数的转储，您应该已经了解了:）：

```
ready> def foo(x) x + 1;
ready> foo(2);
Evaluated to 3.000000

ready> def foo(x) x + 2;
ready> foo(2);
Evaluated to 4.000000
```

它起作用了！

即使使用这个简单的代码，我们也能获得一些令人惊讶的强大功能——看看这个：

```
ready> extern sin(x);
Read extern:
declare double @sin(double)

ready> extern cos(x);
Read extern:
declare double @cos(double)

ready> sin(1.0);
Read top-level expression:
define double @2() {
entry:
  ret double 0x3FEAED548F090CEE
}

Evaluated to 0.841471

ready> def foo(x) sin(x)*sin(x) + cos(x)*cos(x);
Read function definition:
define double @foo(double %x) {
entry:
  %calltmp = call double @sin(double %x)
  %multmp = fmul double %calltmp, %calltmp
  %calltmp2 = call double @cos(double %x)
  %multmp4 = fmul double %calltmp2, %calltmp2
  %addtmp = fadd double %multmp, %multmp4
  ret double %addtmp
}

ready> foo(4.0);
Read top-level expression:
define double @3() {
entry:
  %calltmp = call double @foo(double 4.000000e+00)
  ret double %calltmp
}

Evaluated to 1.000000
```

哇，JIT怎么知道sin和cos呢？答案非常简单：KaleidoscopeJIT有一个简单的符号解析规则，用于查找在任何给定模块中不可用的符号：首先它从最近到最旧的所有已添加到JIT中的模块中搜索最新的定义。如果在JIT内部找不到定义，它会回退到在Kaleidoscope进程本身上调用“dlsym("sin")”。由于“sin”在JIT的地址空间中定义，它只是直接修补模块中的调用以调用libm版本的sin。但在某些情况下，这甚至会更进一步：由于sin和cos是标准数学函数的名称，当用常量调用时，常量折叠器将直接将函数调用评估为正确的结果，如上面的“sin(1.0)”所示。

在未来，我们将看到如何调整这个符号解析规则，可以用来启用各种有用的功能，从安全性（限制JIT代码可用的符号集），到基于符号名称的动态代码生成，甚至是延迟编译。

符号解析规则的一个直接好处是我们现在可以通过编写任意C++代码来实现操作来扩展语言。例如，如果我们添加：

```cpp
#ifdef _WIN32
#define DLLEXPORT __declspec(dllexport)
#else
#define DLLEXPORT
#endif

/// putchard - 接受一个double并返回0的putchar。
extern "C" DLLEXPORT double putchard(double X) {
  fputc((char)X, stderr);
  return 0;
}
```

注意，对于Windows，我们需要实际导出这些函数，因为动态符号加载器将使用GetProcAddress查找符号。

现在我们可以使用类似“extern putchard(x); putchard(120);”的代码在控制台上生成简单的输出，这会在控制台上打印一个小写的‘x’（120是‘x’的ASCII码）。类似的代码可以用来在Kaleidoscope中实现文件I/O、控制台输入和许多其他功能。

这完成了Kaleidoscope教程的JIT和优化器章节。目前，我们可以以用户驱动的方式编译一个非图灵完备的编程语言，优化并JIT编译它。接下来我们将研究如何通过控制流结构扩展语言，并在此过程中解决一些有趣的LLVM IR问题。