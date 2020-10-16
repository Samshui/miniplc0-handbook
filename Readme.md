# miniplc0 实验指导书

## 1. 概述

本指导书为 miniplc0 指导书。

由于本指导书还只是一个 beta 版本，因此很可能存在一些错误，如果你发现本书中有以下问题

- 逻辑性/知识性错误
- 表意模糊/错误
- 前后矛盾
- 代码不对应/错误
- 示例输出过时
- ...

欢迎积极联系助教，也可以直接提 issue 甚至 pr，勘误或者单纯的建议都会有一定的加分。

## 2. 编译过程概述

**编译**（compilation）的目的是将指定语言的源代码输入翻译成目标语言输出，通常来说目标语言是某种汇编指令集。

> 粗略地说，一个最简单的编译过程包含了五个顺序执行的主要过程：[词法分析](#词法分析)->[语法分析](#语法分析与语义分析)->[语义分析](#语法分析与语义分析)->[代码优化](#代码优化)->[代码生成](#代码生成)，以及贯穿它们的两个辅助过程：[符号表管理](#符号表管理)和[错误处理](#错误处理)。 
>
> 当存在中间代码时，语义通常会执行中间代码的生成。比较简单的模型下，语义分析可能直接生成目标代码；对于一个多趟的编译过程，可能存在多种中间代码。
>
> 为了降低编译过程的复杂性，存在按阶段划分的编译过程模型：
>
> - 源代码相关的**前端**（front end）：预处理、词法分析、语法分析、语义分析和中间代码生成
> - 源代码和目标机都无关的**中端**（middle end）：机器无关的代码优化
> - 目标机相关的**后端**（back end）：机器相关的代码优化和目标代码生成
>
> LLVM 采用的就是三阶段的编译模型。

实际编程语言的应用场景会更复杂一些，比如[C语言的主要编译过程](https://en.cppreference.com/w/c/language/translation_phases)就有八个阶段；而我们实验中采用的**基于递归下降分析的语法制导分析**则是在语法分析的同时进行语义分析和代码生成。在对简单过程模型的学习后，你可以更深入地去考虑现代编程语言的编译过程，事实上C语言编译过程也是面试经典题。

通过本章节的阅读，你将：

- 了解编译过程的基本组成
- 了解一些编译技术相关的术语
- 了解栈式虚拟机的基本概念

### 2.1 词法分析

> **词法分析**（lexical analysis）是编译器读入源代码文本，并将代码划分为有具体含义的单词序列的过程，这些单词被称为**token**。
>
> 词法分析作为将代码token化的过程，也被称为tokenization；相应地，词法分析器（lexer）也被称为tokenizer。

比如在C语言的编译过程中，一行代码`int a =1;`，经过预处理可能会被视为一组token：`关键字int`、`空白符序列`、`标识符a`、`空白符序列`、`赋值运算符=`、`无符号整数字面量1`、`分号;`。

C语言在词法分析之前还专门会有对源代码的预处理以及将代码划分为逻辑行的过程，而我们的实验文法相对简单，这些过程可能会通过语法分析的一些工具函数代为实现。

#### 2.1.1 最大吞噬规则

通常来说我们的文法会有一定的二义性，但是文法本身已经足够简洁而且实用，这种情况下我们会让编译器遵循**最大吞噬规则**。

在词法分析阶段，最大吞噬可以理解为：**一个token要尽可能多地识别它可以接受的字符**。比如C语言中`returna`不会被理解为`return`和`a`，`inta`也不会被理解为`int`和`a`。

有些时候最大吞噬规则也会有例外，如果本指导书中出现了例外情况，我们会特别指出，其他情况下默认遵循最大吞噬规则。

### 2.2 语法分析与语义分析

> **语法分析**（syntactic analysis）是编译器根据词法分析结果（比如token序列）检查代码是否符合语法规则的过程，通常会生成一棵描述源代码程序语法结构的树。
>
> 语法分析也被称为parsing；相应地，语法分析器也被称为parser。
>
> **语义分析**（semantic analysis）是编译器根据语法分析的结果检查代码是否符合语义规则的过程。此过程一般会构建符号表，并向语法树添加额外的语义信息，有些时候还会执行中间代码的生成。

文法规则定义了编程语言的基本结构，就好像汉语语法中对主谓宾的顺序和组合有各种要求一样。语法分析的目的就是检查是否存在一段代码，它不符合文法规则。

对于无法通过文法规则直接表述的问题，比如变量不应该被重定义，语法分析在这种情况下就会显得鞭长莫及，此时就需要交给语义分析来完成。下面这几种为我们所熟知的C语言编译错误都是通过语义分析发现的：

- 判断变量是否发生了重定义
- 判断是否在指定了返回类型的函数中返回了空值
- 判断是否使用了错误的值类型进行赋值等
- 判断是否对声明为const的常量进行了重新赋值

#### 2.2.1 未定义行为

> **未定义行为（Undefined Behavior）**，简写为UB，指的是源代码中符合文法规则，但是在语义规则中没有相关规定的行为。通常来说，未定义行为不是编译错误，而如何理解未定义行为对应的程序操作（或决定它是不是编译错误），取决于编译器的实现者。

比如C语言中，访问没有初始化过的局部变量，是一种未定义行为：
```clike
int fun() {
    int a;
    printf("%d\n", a); // UB
}
```
C语言标准不要求局部变量被默认初始化为0，通常情况下编译器的实现者也不会，但上面的操作的确符合文法规则。在运行时，上面的程序的输出是不确定的。

由于不确定性的存在，要想编写健壮的程序，我们要尽量避免UB。而设计编程语言时，可以像Java那样给出非常严格的语义规则，也可以像C/C++那样把责任交给程序员。

#### 2.2.2 语法制导翻译

> **语法制导翻译**（syntax-directed translation）可以视为一边进行语法分析一边进行语义分析。

我们实验中采用的是：基于递归下降分析的语法制导翻译。

递归下降分析本身构建语法树的顺序就是之后在语法树上做遍历的顺序，父节点相对于子节点的生成树顺序决定了遍历的顺序（前序、后序等）。

如果让递归下降分析的父节点生成放在所有子节点生成后，并且同时进行语义分析，那么得到的动作指令序列（或中间代码）刚好满足逆波兰式。因为这个特点，这种分析方式和栈式指令集有很好的相性。由于动作序列（或中间代码）此时已经生成，因此在语义规则不太复杂的情况下，甚至可以省略语法树的构建。


### 2.3 符号表管理

> **符号表**是存储了已经被声明过的标识符的具体信息的数据结构。在语义分析阶段构造符号表，并根据分析的需要对符号表进行增加、删除、修改、查询，即是所谓的**符号表管理**。

对于编译型语言，符号表的生命周期往往和语义分析相同，最终得到的目标代码/可执行程序中，通常不会包含有关源代码中名字的信息。

符号表的形态往往取决于实际情况，在我们的mini实验中，符号表只是一个哈希表（[`std::unordered_map`](https://en.cppreference.com/w/cpp/header/unordered_map)）。而C0是拥有多级作用域的语言，如果其采用一趟扫描的编译，通常会采用栈式符号表；而对于多趟扫描的话，树形符号表会更实用一些。

### 2.4 错误处理

> **错误处理**是贯穿整个编译流程的一环，它主要负责对各编译子程序发现的错误进行记录、报告、甚至是纠正。

错误处理是一个可定制度很高的部分，比如下面的C程序：

```c
int main() {
    int a,b;
    int a;
    a = "1";
}
```

你可以简单地报错并终止编译：

```
error! line 3: identifier redeclaration 
```

也可以选择详细地输出错误的信息（位置、相关代码、上下文）并对查错和修改提出一些帮助或建议：

```
error: line 3 col 9: redeclaration of "a":
    int a;
        ^
note: previous declaration of "a" at line 2 col 9:
    int a,b;
        ^
```

还可以选择在报告错误后不终止编译，而是记录下来并跳读代码到可以继续正常编译的地方，以尽可能多地发现有效错误。甚至可以对一些未定义行为提出警告。

不夸张的说，一个优秀的错误处理程序，可以对程序员的错误处理提供极大的帮助。

### 2.5 代码优化

 > **代码优化**是为了提高诸如运行速度和内存占用等的程序性能，对编译的中间结果/目标代码进行一系列优化的行为。

如果你玩过一些策略型解密游戏，他们通常会设置一些挑战，当你采用的操作更少，但是收益更高时，就给你更高的星级评价。代码优化则是让你的程序可以取得更高评价的重要一环。

代码优化的可定制度极高，比如C程序：

```c++
int fun() {
    const int a = 1;
    int b = 2;
    return (a+b)*b+2*a*(a+b);
}
```

你可以进行下列优化：

- `a`是以字面量初始化的常量，你可以在编译期就将所有读取`a`的操作等价理解为读取`1`的；
- `a+b`在中间进行了两次求值，并且过程中`a`和`b`没有进行任何修改，你也可以将其优化为一次求值，该次求值的结果存储到一个中间变量，在第二次需要时直接访问中间变量；
- `fun`运行过程中，`a`和`b`的值都是编译期常量，因此`fun`的返回值也是编译期常量，可直接让`fun`返回`12`以消除求值的开销，甚至将所有对`fun`的调用替代为读取`12`以消除函数调用的开销。

代码优化往往需要结合目标运行平台进行特化。gcc目前是使用了一种语言无关、环境无关的中间语言，编译时gcc先将源代码转换成中间代码，对中间代码进行一系列优化之后再针对具体平台进行特殊的优化，以得到更优的可执行程序。

### 2.6 代码生成

编译的最终目的是将指定语言的源代码文件翻译成目标语言文件，通常来说目标语言是某种指令集。

我们的mini实验和C0实验均采用栈式虚拟机为目标平台。

#### 2.6.1 栈式虚拟机

> **栈式虚拟机**以栈为主要数据结构，执行指令时，操作主要发生在栈上。
>
> 栈式虚拟机的指令的操作数，通常来自栈顶，指令运行产生的数据会压到栈顶。

举一个简单的例子，对于加法运算`3+2+1`，x86可能会这么做：

```assembly
mov $3, %eax    # 将立即数3移入寄存器ax
add $2, %eax    # 将立即数2加到寄存器ax
add $1, %eax    
# 最终结果存储在寄存器ax

# 如果做了激进的优化
mov $6, %eax
```

而栈式虚拟机可能会：

```assembly
PUSH $3    # 将3压到栈顶
           # 此时栈从底到顶依次为： 3
PUSH $2    # 此时栈从底到顶依次为： 3 2
ADD        # 弹出栈顶作为右操作数，再弹出栈顶作为左操作数，两者相加得到的值再压到栈顶
           # 此时栈从底到顶依次为： 5
PUSH $1    # 此时栈从底到顶依次为： 5 1
ADD 
# 最终结果在栈顶，此时栈从底到顶依次为： 6

# 如果做了激进的优化
PUSH $6
```

从这个例子或许可以更直观的看出来：源代码的逆波兰表示和栈式虚拟机指令序列十分相似。

值得一提的是，Java的运行环境JVM也是一种栈式虚拟机，有兴趣的同学可以通过jdk提供的`javap`命令对.class字节码进行反汇编，或许可以给C0实验提供一些思路。

## 3. mini plc0 编译系统

这一部分我们分为两个版本，分别对应 C++ 和 Java 版本的 miniplc0 实验环境。请同学按照自己选择的实验查看相应的文档。

- [C++ 版本请看这里](readme-cpp.md)
- [Java 版本请看这里](readme-java.md)

## 附录A EBNF

Extended Backus-Naur Form（扩展巴科斯范式），是[ISO/IEC 14977](http://standards.iso.org/ittf/PubliclyAvailableStandards/s026153_ISO_IEC_14977_1996(E).zip)接受的一种元语法符号表示法。

由于当前课本采用的类似EBNF的表示法在某些情况下的描述能力很差，因此我们参考了Wikipedia，以[BNF](https://en.wikipedia.org/wiki/Backus%E2%80%93Naur_form)为主，结合了一些[EBNF](https://en.wikipedia.org/wiki/Extended_Backus%E2%80%93Naur_form)和[ABNF](https://en.wikipedia.org/wiki/Augmented_Backus%E2%80%93Naur_form)的语法糖，得出了一种更加严谨的类似EBNF的表示法。

### A.1 BNF

一套文法，由若干条规则组成。

#### A.1.1 规则与终结符


> **约定1.** 规则使用字符序列`::=`分割，在规则左侧（`::=`左侧）出现过的符号称为**非终结符**；**没有在规则左侧出现过、也不是用以描述规则的特殊符号**的其他字符，称为**终结符**。
> 
> 在描述一条规则时，通常我们用一对尖括号`<>`包围非终结符，而用一对引号（单引号`''`或双引号`""`皆可）包围连续的任意长的终结符串（或称为**字面量**）。而没有用尖括号或是引号包围的串，**都是元符号**。


例如下面的**布尔字面量文法**包含了三条规则：

```
<bool literal> ::= <true literal> | <false literal>
<true literal> ::= 'true'
<false literal> ::= 'false'
```

其中：
- `bool literal`、`true literal`和`false literal`在规则左侧出现过，因此是非终结符，可以看到规则中他们被尖括号`<>`包围。
- 字符串`true`到`false`是字面量，可以看到规则中他们被单引号`''`包围。
- 字符串`::=`分隔规则的两侧内容
- 字符`|`表示这条规则的右侧有多种候选项，详见后文

在本附录章节的后续内容中，除了完整写出一条规则的情况，其他情况下非终结符会用尖括号包围（如`<bool literal>`），而终结符串则会直接写出（如`true`）。

> **推导** 规则右侧的内容可以由左侧的非终结符通过推导得到。
> 
> 能够由文法的起始非终结符经过有限次推导得到的终结符串，是符合该文法的字符串。

比如非终结符`<bool literal>`可以推导得出`<true literal>`，也可以推导得出`<false literal>`；非终结符`<true literal>`可以推导得出字符串`true`。


> **约定2.** 规则左侧只能有一个符号，且必须是非终结符。

比如下列都不是一个合法的规则：

- `<A><B> ::= 'ab'`
- `'a' ::= 'b'`
- `<A>'b' ::= 'ab'`

> **约定3.** 规则中的不属于某个符号（非终结符或字面量）的任何空格或换行，都不具有实际含义，只是为了美观和可读性而人为添加的。

比如，规则`<A> ::= 'a'`和规则`<A>::='a'`本质上是完全相同的规则。

再比如，规则`<bool literal> ::= <true literal> | <false literal>`也可以像下面这么写：
```
<bool literal> ::= 
     <true literal> 
    |<false literal>
```

但是规则`<space> ::= ' '`和规则`<space> ::= ''`不是一样的，因为单引号包围的所有内容都被认为是字面量。事实上，使用引号包围字面量的一个原因是为了更直观的描述空格。

> **连接** 如果规则中存在两个相邻的符号A和B（各自可以是非终结符或终结符串），那么这两个符号之间的运算就是连接。
> 
> 在进行推导时，如果符号A能够推导出字符串x，符号B能够推导出字符串y，那么符号A和B的连接AB能够推导出字符串x和y的连接xy。
> 
> 因此规则中两个字面量进行连接，可以表述为一个新的字面量，这个字面量的内容是原来两个字面量的内容的连接。**但是如果本指导书中将两个相邻的字面量用空白符分开，说明我们有意地表示它们是两个不同的token**。

比如：

- 规则`<A> ::= <A> <B>`中`<A>`和`<B>`是相邻的。
- `<A> ::= 'a' 'a'`等价于`<A> ::= 'aa'`
- `<A> ::= 'a' ' ' 'a'`等价于`<A> ::= 'a a'`

请不要忘记，文法中不出现在非终结符或字面量中的空格，只是为了美观而添加的。

> **选择** 选择符指的是规则右侧出现的字符`|`，规则左侧的非终结符可以推导出`|`两侧的任意选项。
> 
> 连接具有比选择更高的运算符优先级。

比如：

- 规则`<zerone> ::= '0' | '1'`表示`<zerone>`可以推导出字面量`0`或`1`。
- 规则`<A> ::= <A><A>|'a'`表示`<A>`可以推导出`<A><A>`或字面量`a`。

> **约定4** 当字面量需要描述双引号`"`时，该字面量必须用一对单引号`''`包围；需要描述单引号`'`时，必须用双引号包围`""`。
> 
> 多个不同种类的引号存在时，以第一个出现的引号为字面量的开始标记，第一个和开始标记同种类的引号为结束标记。

比如规则：
- `<single quote> ::= "'"`说明字面量`'`可以由`<single quote>`推导得出 
- `<double quote> ::= '"'`说明字面量`"`可以由`<double quote>`推导得出 
- `<zero integer literal> ::= '0'` （或`<zero integer literal> ::= "0"`）描述的是整数字面量`0`
- `<zero char literal> ::= "'0'"`描述的是C语言中的字符字面量`'0'`
- `<LF char literal> ::= "'\n'"`描述的是C语言中等值于换行符（line-feed）的字符字面量`'\n'`
- `<string literal> ::= '"' <characters> '"'`描述的是C语言中的形如`"123"`的字符串字面量

> **约定5.** 不出现在非终结符或字面量时，使用字符`ε`表示空串。

比如：

-  `<empty> ::= ε` 表示`<empty>`推导出空串。
-  `<optional A> ::= 'A' | ε`表示`<optional A>`可以推导出字面量`A`，也可以推导出空串
-  `<εAε> ::= εεεε'εaε'εεεε` 等价于`<εAε> ::= 'εaε'`，不等价于`<A> ::= 'a'`

### A.2 EBNF

#### A.2.1 组

> **组** 组指的是规则右侧由一对圆括号`()`包围的符号串，该符号串被视为一个整体

比如`<A> ::= (<A>|ε)'a'`等价于`<A> ::= <A>'a'|'a'`。

#### A.2.2 可选项

> **可选项** 可选项指的是规则右侧由一对方括号`[]`包围的符号串，`[<A>]`等价于`(<A>|ε)`。

比如`<A> ::= [<A>]'a'`等价于`<A> ::= (<A>|ε)'a'`等价于`<A> ::= <A>'a'|'a'`。

#### A.2.3 重复项

> **重复项** 重复项指的是规则右侧由一对花括号`{}`包围的符号串，`{<A>}`等价于无数个`[<A>]`的连接，即`[<A>][<A>][<A>][<A>]...`。

比如`<A> ::= {'a'}`可以推导出空串、`a`、`aa`、由任意多个`a`连接得到的字面量。

### A.3 ABNF

> **约定6.** 文法中使用格式为`%x??`的符号表示ASCII码值为`??`的字符，`%x??-!!`表示ASCII码值在闭区间（包含边界值）`??`到`!!`的任意字符
> 
> **约定6最终没有采用**

此约定目的是便于描述控制字符，或是便于描述可选值域过大的字面量。

比如：
- `<space> ::= ' '` 等价于 `<space> ::= %x20`
- `<zerone> ::= %x30-31` 等价于 `<zerone> ::= %x30 | %x31` 等价于 `<zerone> ::= '0' | '1'`
- `<alpha> ::= %x41-5A | %x61-7A` 表示 `<alpha>`可以推导出任意一个英文字母（无论大小写）
- `<LF> ::= %x0A` 表示 `<LF>`可以推导出值为`0x0A`的换行符；此规则和`<LF literal> ::= '\n'`完全不同，后者推导出的是两个字节长的字面量`\n`

由于不够直观因此对阅读存在一些阻碍（比如为了理解`<space> ::= %x20`，读者需要事先知道ASCII值等于`0x20`的字符是空格），**约定6最终没有采用**，因此实际情况下，可能会用如下表述：

```
<空白符> ::= ' ' | <LF> | <CR> | <HT>
<LF> 是 ASCII值等于0x0A的字符(换行符)
<CR> 是 ASCII值等于0x0D的字符(回车符)
<HT> 是 ASCII值等于0x09的字符(水平制表符)
```

```
<alpha> 可以是ASCII值满足如下任意一个条件的任意字符：
    大于等于0x41(A)且小于等于0x5A(Z)
    大于等于0x61(a)且小于等于0x7A(z)
```

# 4 提交方式

这次试验的提交方式与之前的实验一样，也需要向 OJ 平台提交一个 git 存储库。Java 和 C++ 版都自带了可以正常编译和测试的配置文件，如果你对编译操作做了任何修改（比如修改了 cmake 的参数）请自行修改配置文件。

## 附录D Build with docker

### D.1 Linux

如果还没有装 docker 的话先一键安装

```bash
curl https://get.docker.com | sh
```

#### 只构建不测试

输入下面的命令来使用 dockerfile 自动编译构建一个镜像，并为镜像添加 `<your_tag>` 的标签

```bash
docker build . -t <your_tag>
```

使用镜像来处理本地文件。其中 `<your_params>` 是你运行程序的参数，`<path>` 是你希望处理的文件（夹）的 **绝对路径**。文件将会被映射到容器内 `/tests` 路径中。

```bash
docker run --rm -it -v <path>:/tests <your_tag> <your_params>
#                             ^~~~~~被映射到的路径
```


#### 旧版方法

然后我们从最新的 image 创建一个 container。

```bash
# pull latest image
docker pull lazymio/compilers-env
# -t --tty
# -d --detach
docker run -t -d --name mycontainer lazymio/compilers-env
# open a shell in the container
docker exec -it mycontainer /bin/bash
```

注意从这里开始我们是在 container 内执行指令。

接下来先编译。

```bash
cd ~
git clone https://github.com/BUAA-SE-Compiling/miniplc0-compiler
cd miniplc0-compiler
git submodule update --init --recursive
mkdir build
cd build
cmake ..
make
```

然后如果直接运行可以

```bash
./miniplc0 --help
```

想运行测试可以

```bash
make test
```

### D.2 FAQ

#### D.2.1 docker pull 好慢

换源，具体问 Google。

#### D.2.2 Windows/Mac 怎么办

我建议虚拟机 Ubuntu/Debian，因为这是我觉得最省事的办法，一行 `curl https://get.docker.com | sh` 就完事了。

如果有能力的话可以探索 docker on Windows/Mac 但是有一点需要强调的是：最终我们的评测环境一定是 docker on Linux，尽管因为宿主机环境带来的影响微乎其微，但是对比一下[这里](https://github.com/docker?utf8=%E2%9C%93&q=for&type=&language=)三大平台上 issue 的数量，我们没能力也没信心保证你在 docker on Windows/Mac 的输出一定会和 docker on Linux 输出一致，即使你在 docker on Windows/Mac 测试通过了，我们仍然建议在提交作业前在 docker on Linux 的环境下跑一遍 tests。简单来说，一切都是为了保证输出的一致。

另外根据同学的报告，虚拟机内存小于等于 1G 很可能因为内存不足编译失败，见[讨论](https://github.com/BUAA-SE-Compiling/miniplc0-compiler/issues/4)

此外顺带一句：docker on Windows 需要 HyperV，Windows10 Home 的同学可以歇歇了。

#### D.2.3 container 怎么没有 sshd 呀

首先看[这篇文章](https://jpetazzo.github.io/2014/06/23/docker-ssh-considered-evil/)。

当然如果你非要 sshd 也是可以的，只要

```bash
service sshd restart
```

就 ok 了，至于网络问题请自行探索。
