## 语言与正则表达式

<center>![2](./image/image.png){width="500"}</center>

一些定义：
<center>![2](./image/image1.png){width="500"}</center>

能够在语言（集合）上进行的操作：
<center>![2](./image/image2.png){width="500"}</center>

正则表达式：
<center>![2](./image/image3.png){width="500"}</center>
<center>![2](./image/image4.png){width="500"}</center>

## 自动机理论与词法分析器生成器

<center>![2](./image/image5.png){width="300"}</center>
<center>![2](./image/image6.png){width="400"}</center>

### 目标：RE to Lexer
<center>![2](./image/image7.png){width="400"}</center>

**NFA：非确定性有穷自动机（简洁易于理解，便于描述语言 $L(\mathcal{A})$）**

<center>![2](./image/image8.png){width="500"}</center>
其不确定性在于：

- 在同一状态下，看到相同的字符，可以转移到不同的状态
- 没有看到任何字符（看到 $\epsilon$ ），也有可以发生状态的转移


**DFA：确定的有穷自动机（易于判断 $x \in L(\mathcal{A})$ ，适合产生词法分析器）**
<center>![2](./image/image9.png){width="500"}</center>

**约定**：所有没有对应出边的字符默认指向一个“**死状态**”（不可能再被接收）

其确定性在于：

- 只对字符表中的字符做状态转移
- 每个状态下对每个字符都有唯一的状态转移

<center>![2](./image/image10.png){width="400"}</center>

#### Thompson 构造法（RE to NFA）

<center>![2](./image/image11.png){width="300"}</center>

依据 **正则表达式的定义** ，分步处理：
<center>![2](./image/image12.png){width="400"}</center>

分析：
<center>![2](./image/image13.png){width="500"}</center>

例子：
<center>![2](./image/image14.png){width="600"}</center>

#### 子集构造法（NFA to DFA）

<center>![2](./image/image15.png){width="500"}</center>
<center>![2](./image/image16.png){width="500"}</center>
<center>![2](./image/image17.png){width="500"}</center>

#### DFA 最小化（DFA to 最简等价DFA）

DFA 最小化的基本思想：等价的状态可以合并
<center>![2](./image/image18.png){width="500"}</center>

出发点：
<center>![2](./image/image19.png){width="500"}</center>
<center>![2](./image/image20.png){width="300"}</center>
<center>![2](./image/image21.png){width="500"}</center>

DFA 最小化算法总结：
<center>![2](./image/image22.png){width="500"}</center>

算法要在严格定义的 DFA 上执行，**注意** 要先检查是否需要补全“死状态”

#### Kleene 构造法（DFA to RE）

<center>![2](./image/image23.png){width="500"}</center>
<center>![2](./image/image24.png){width="500"}</center>
<center>![2](./image/image25.png){width="500"}</center>
<center>![2](./image/image26.png){width="500"}</center>

### 词法分析器生成

**遵循规则：最前优先匹配，最长优先匹配**

- 根据正则表达式构造相应的 NFA
<center>![2](./image/image27.png){width="500"}</center>

- 合并 NFA，构造完整词法规则的 NFA
<center>![2](./image/image28.png){width="500"}</center>

- 使用子集构造法将 NFA 转化为等价 DFA（并消除“死状态”）
<center>![2](./image/image29.png){width="500"}</center>
<center>![2](./image/image30.png){width="500"}</center>

- 特定于词法分析器的 DFA 最小化（可选）

初始划分不能把所有终止状态笼统地分成一类，需要考虑不同的词法单元，按照识别不同词法单元规约分成不同的类，并补全“死状态”（然后再正常根据 DFA 最小化算法进行处理）
<center>![2](./image/image31.png){width="500"}</center>

- 模拟运行该 DFA，直到无法继续运行为止（输入结束或状态无法转移）
<center>![2](./image/image32.png){width="500"}</center>

## 语法分析器

### 二义性文法（以ANTLR v4工具为例）

例如：stmt："if" expr "then" stmt "else" stmt ; 是一个二义性文法（if-else 的匹配问题）

可以通过改写来消除二义性（将 else 与最近的未匹配的 if 相匹配）：
```antlr
stmt : matched_stmt | open_stmt ;
matched_stmt : "if" expr "then" matched_stmt "else" matched_stmt
             | expr ;
open_stmt : "if" expr "then" stmt
		  | "if" expr "then" matched_stmt "else" open_stmt ;
```
例如：expr：expr "*" expr | expr "+" expr | DIGIT ; 是一个二义性文法（运算符的结合性和优先级）

常识告诉我们乘法的优先级比加法高，并且两者都是左结合的
```antlr
/* 左递归文法 */
expr : expr "+" term | term ;
term : term "*" factor | factor ;
factor : DIGIT ;
```
ANTLR 可以通过算法直接处理二义性文法和具有左公因子的文法，无需人为改写（优先级高的写在前）

ANTLR 中二元运算符默认为左结合，一元运算符默认为右结合，对于右结合的二元运算符，使用关键字 assoc 加以限定，`expr:<assoc = right> expr "^" expr`

### 上下文无关文法

<center>![2](./image/image33.png){width="500"}</center>
<center>![2](./image/image34.png){width="500"}</center>

**一些复杂的例子**

<center>![2](./image/image35.png){width="400"}</center>

**上下文无关文法的表达能力强于正则表达式**

<center>![2](./image/image36.png){width="500"}</center>
<center>![2](./image/image37.png){width="500"}</center>
<center>![2](./image/image38.png){width="400"}</center>
<center>![2](./image/image39.png){width="500"}</center>

**证明一个上下文无关文法没有二义性**

对于 $A → X | Y | Z$，要证明：

- $L(X) \cap L(Y) = \emptyset$
- $L(Y) \cap L(Z) = \emptyset$
- $L(Z) \cap L(X) = \emptyset$

其中，$L(X)$ 表示 $X$ 所定义的语言

故，更一般的，对于 $A → X_1 | X_2 | X_3 | … | X_n$，需要证明$∀ i, j \in [1, n] (i \neq j) \quad L(X_i) \cap L(X_j) = \emptyset$

## 语法分析算法

### LL(1) 语法分析算法

自顶向下的，递归下降的，基于分析预测表的，适用于 LL(1) 文法的 LL(1) 语法分析器

