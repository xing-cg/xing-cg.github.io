---
title: 协程_C语言协程实现
categories:
  - - 操作系统
    - 多线程
tags: 
date: 2023/3/6
updated: 
comments: 
published:
---
# 问题引入：生产和消费

如果此时有一段代码生成数据，另一段代码使用数据，哪一段代码是调用者，哪一段是被调用者呢？

有一段非常简洁的`run-length decompression code`，和一段同样简洁的解析器代码：

```c
    /* Decompression code */
    while (1)
    {
        c = getchar();
        if (c == EOF)
            break;
        if (c == 0xFF)
        {
            len = getchar();
            c = getchar();
            while (len--)
                emit(c);
        }
        else
            emit(c);
    }
    emit(EOF);
```

```c
	/* Parser code */
	while (1)
    {
        c = getchar();
        	break;
        if (isalpha(c))
        {
            do
            {
                add_to_token(c);
                c = getchar();
            } while (isalpha(c));
            got_token(WORD);
        }
        add_to_token(c);
        got_token(PUNCT);
    }
```

第一个片段每调用emit一次就生成一个字符；另一个片段每调用getchar一次就要使用一次字符。如果只能调用emit和getchar来相互传递数据，可以很简单地将两个片段连接在一起来达成目的，即解压器的输出直接传递到解析器。

# 连接两个片段的方案

在现代的操作系统中，可以使用两个进程或者线程之间的管道实现连接，解压器中的emit写入一个管道，解析器中的getchar从这个管道的另一端读取。简单、健壮，但是用线程来解决有点重量级、不够便携。 通常情况下，不需要为如此简单的任务将程序划分为多个线程。

常规的答案(conventional answer)就是重写(rewrite)通信信道的末端中的一个，如此它就成了一个可被调用的函数。下面是对每个示例片段意味着什么的例子。

```c
int decompressor()
{
    static int repchar;
    static int replen;
    if (replen > 0)
    {
        replen--;
        return repchar;
    }
    c = getchar();
    if (c == EOF)
        return EOF;
    if (c == 0xFF)
    {
        replen = getchar();
        repchar = getchar();
        replen--;
        return repchar;
    }
    else
        return c;
}
```

```c
void parser(int c)
{
    static enum { START, IN_WORD } state;
    switch (state)
    {
        case IN_WORD:
            if (isalpha(c))
            {
                add_to_token(c);
                return;
            }
            got_token(WORD);
            state = START;
            /* fall through */
        case START:
            add_to_token(c);
            if (isalpha(c))
                state = IN_WORD;
            else
                got_token(PUNCT);
            break;
    }
}
```

当然，不必重写两个。如果重写解压缩器，以便每次被调用时返回一个字符，则原始解析器代码可以用decompressor替换getchar。相反，如果重写解析器，以便为每个输入的字符都调用一次，则原始解压缩代码可以调用parser而不是emit。如果你是一个贪吃惩罚的人，你只会想将这两个函数重写为被调用者。

这就是重点。与原始函数相比，这两个重写的功能都非常丑陋。当作为调用者而不是被调用者编写时，这里发生的两个过程(process)都更容易阅读。试着去推导解析器能识别的语法，或者解压器能理解的压缩数据格式，仅仅通过阅读代码，你会发现原始的代码很清晰，重写的代码都不太清楚。 如果我们不必将任何一段代码从里到外重写，那就更好。

## Knuth's coroutines

在 The Art of Computer Programming 中，Donald Knuth 提出了解决此类问题的方法。他的回答是完全抛弃栈的概念。不要再将一个进程视为调用者，将另一个进程视为被调用者，而是开始将它们视为平等合作。

实际操作上：用略有不同的原语(primitive: In programming, a fundamental element in a language that can be used to create larger procedures that do the work a programmer wants to do.)替换传统的“调用(call)”原语。 新的“调用”会将返回值保存在堆栈以外的其他位置，然后跳转到另一个保存的返回值中指定的位置。 因此，每次解压缩器发出另一个字符时，它都会保存其程序计数器并跳转到解析器中最后已知的位置 - 每次解析器需要另一个字符时，它都会保存自己的程序计数器并跳转到解压缩器保存的位置。 控制在两个例程(routines)之间来回穿梭(back and forth)。

从理论上讲，这是非常好的，但是实际上只能以汇编语言进行操作，因为没有常用的高级语言支持协程调用的原语(coroutine call primitive)。像C这样的语言完全取决于其基于堆栈的结构，因此，每当控制从任何函数传递到任何其他函数时，都必须有一个是调用者，另一个必须是被调用者。因此，如果想编写便携式代码，则此技术至少与Unix Pipe解决方案一样不切实际。

## Stack-based coroutines

所以我们真正想要的是在C中模仿 Knuth 的协程调用原语的能力。我们必须接受现实，在C级别，一个函数将是调用者，另一个将是被调用者。在调用者中，我们没有问题；我们对原始算法进行code，几乎完全按照编写的方式进行，并且每当它有（或需要）一个字符时，它就会调用另一个函数。

被调用者有全部的难题需要解决。对于被调用者，我们想要一个具有“返回并继续”操作的函数：从函数返回，下次调用它时，从返回语句之后恢复控制。 例如，我们希望能够编写一个函数：

```c
int function()
{
    int i;
    for (int i = 0; i < 10; i++)
    {
        return i; /* won't work, but wouldn't it be nice */
    }
}
```

我们想要对函数进行十次连续调用，返回数字0到9。我们如何实现呢？ 可以使用goto语句将控制转移到函数中的任意点。 所以如果我们使用状态变量，我们可以这样做：

```c
int function()
{
    static int i, state = 0;
    switch (state)
    {
        case 0: goto LABEL0;
        case 1: goto LABEL1;
    }
LABEL0: /* start of function */
    for (i = 0; i < 10; i++)
    {
        state = 1; /* so we will come back to LABEL1 */
        return i;
LABEL1:
        ; /* resume control straight after the return */
    }
}
```

这个方法有效。我们在可能需要恢复控制的点上有一组标签：一个在开始处，一个在每个 return 语句之后。我们有一个状态变量，在函数调用之间保存，它告诉我们下一个需要恢复控制的标签。在任何return之前，我们更新状态变量以指向正确的标签；在任何调用之后，我们都会对变量的值进行切换以找出跳转到的位置。

不过，它仍然很难看。最糟糕的部分是标签集必须手动维护，并且必须在函数体和初始 switch 语句之间保持一致。每次我们添加一个新的返回语句，我们必须发明一个新的标签名称并将其添加到开关中的列表中；每次我们删除 return 语句时，我们都必须删除其对应的标签。 我们刚刚将维护工作量增加了两倍。

## Duff's device

C语言中著名的“达夫装置”利用了这样一个事实，即 case 语句在其匹配的 switch 语句的子块中仍然是合法的。 Tom Duff 使用它来优化输出循环：

```c
switch (count % 8)
{
    case 0: do { *to = *from++;
    case 7:      *to = *from++;
    case 6:      *to = *from++;
    case 5:      *to = *from++;
    case 4:      *to = *from++;
    case 3:      *to = *from++;
    case 2:      *to = *from++;
    case 1:      *to = *from++;
               } while ((count -= 8) > 0);
}
```

我们可以在协程技巧中将其用于稍微不同的用途。我们可以使用 switch 语句来执行跳转，而不是使用 switch 语句来决定执行哪个 goto 语句：

```c
int function(void)
{
    static int i, state = 0;
    switch (state)
    {
        case 0: /* start of function */
            for (i = 0; i < 10; i++)
            {
                state = 1; /* so we will come back to "case 1" */
                return i;
        case 1:
                ; /* resume control straight after the return */
            }
    }
}
```

现在这看起来很有希望。 我们现在要做的就是构建一些精心挑选的宏，我们可以将血淋淋的细节隐藏在看似合理的东西中：（使用`do ... while(0)`来确保`crReturn`直接出现在 if 和 else 之间时不需要大括号）

```c
#define crBegin static int state=0; switch(state) { case 0:
#define crReturn(i,x) do { state=i; return x; case i:; } while (0)
#define crFinish }
int function(void) {
    static int i;
    crBegin;
    for (i = 0; i < 10; i++)
        crReturn(1, i);
    crFinish;
}
```

这几乎正是我们想要的。 我们可以使用`crReturn`以这种方式从函数返回，以便在返回后立即恢复下一次调用的控制。 当然，我们必须遵守一些基本规则（用`crBegin`和`crFinish`包围函数体；如果需要在`crReturn`中保留所有局部变量，则将它们声明为静态的；永远不要将`crReturn`放在显式的`switch`语句中）； 但这些并没有限制我们太多。

剩下的唯一障碍是`crReturn`的第一个参数。 正如我们在上一节中发明新标签时必须避免它与现有标签名称冲突一样，现在我们必须确保`crReturn`的所有状态参数都是不同的。 结果将是相当良性的——编译器会捕获它并且不会让它在运行时做可怕的事情——但我们仍然需要避免这样做。

ANSI C提供了特殊的宏名称`__LINE__`，它表示当前源代码行号。所以我们可以将`crReturn`重写为

```c
#define crReturn(x) do { state=__LINE__; return x; \
                         case __LINE__:; } while (0)
```

如果我们遵守第四条基本规则（永远不要将两个`crReturn`语句放在同一行），那么我们根本就不必担心那些状态参数。

## Evaluation

现在我们有了这个怪物，让我们用它重写我们原来的代码片段。

```c
int decompressor(void) {
    static int c, len;
    crBegin;
    while (1) {
        c = getchar();
        if (c == EOF)
            break;
        if (c == 0xFF) {
            len = getchar();
            c = getchar();
            while (len--)
	        crReturn(c);
        } else
	    crReturn(c);
    }
    crReturn(EOF);
    crFinish;
}
```

```c
void parser(int c) {
    crBegin;
    while (1) {
        /* first char already in c */
        if (c == EOF)
            break;
        if (isalpha(c)) {
            do {
                add_to_token(c);
		crReturn( );
            } while (isalpha(c));
            got_token(WORD);
        }
        add_to_token(c);
        got_token(PUNCT);
	crReturn( );
    }
    crFinish;
}
```

我们已经将解压缩器和解析器都重写为被调用者，完全不需要我们上次这样做时必须进行的大规模重组。**每个函数的结构完全反映了其原始形式的结构**。读者可以推断出解析器识别的语法或解压缩器使用的压缩数据格式，**这比阅读晦涩的状态机代码要容易得多**。一旦您全神贯注于新格式，控制流程就很直观：当解压缩程序有一个字符时，它会使用`crReturn`将其传回给调用者，并等待在需要另一个字符时再次调用。当解析器需要另一个字符时，它使用`crReturn`返回，并等待用参数`c`中的新字符再次调用。

对代码进行了一个小的结构更改：`parser`现在在循环的末尾而不是开始处有它的`getchar`（嗯，相应的`crReturn`），因为当函数运行时第一个字符已经在`c`中进入。我们可以接受这个结构上的小变化，或者如果我们真的对此有强烈的感觉，我们可以指定`parser`需要一个“初始化”调用，然后才能开始向它提供字符。

当然，和以前一样，我们不必使用协程宏重写这两个例程。一个就够了；另一个可以是它的调用者。

我们已经实现了我们打算实现的目标：一种可移植的ANSI C方法，可以在生产者和消费者之间传递数据，而无需将其重写为显式状态机。我们通过将C预处理器与switch语句的一个很少使用的功能相结合来创建隐式状态机(*implicit* state machine)来完成此操作。

## 编码标准问题

当然，这个技巧违反了书中的所有编码标准。尝试在您公司的代码中这样做，可能会受到严厉的批评！在宏中嵌入了不匹配的大括号，在子块中嵌入了用例，以及`crReturn`宏及其可怕的破坏性内容......如果没有因为这种不负责任的编码行为当场被解雇真是个奇迹。应该为自己感到羞耻。

这里的编码标准确实有问题，需要声明。在本文中展示的示例不是很长，也不是很复杂，并且在重写为状态机时仍然可以理解。 但是随着函数越来越长，需要重写的程度越来越大，清晰度的损失也越来越严重。

考虑。 由以下形式的小块构建的函数

```c
    case STATE1:
    /* perform some activity */
    if (condition) state = STATE2; else state = STATE3;
```

对于读者来说，与由以下形式的小块构建的函数没有太大区别

```c
    LABEL1:
    /* perform some activity */
    if (condition) goto LABEL2; else goto LABEL3;
```

一个是调用者，另一个是被调用者，没错，但是函数的视觉结构是相同的，并且它们提供的对其底层算法的洞察力彼此一样小。 那些因为使用我的协程宏而解雇你的人也会因为你用 goto 语句连接的小块构建函数而大声解雇你！这一次他们是对的，因为布置这样的函数会可怕地掩盖算法的结构。

编码标准旨在清晰。 通过将 switch、return 和 case 语句等重要内容隐藏在“混淆”宏中，编码标准会声称您已经模糊了程序的句法结构，并且违反了清晰度要求。但你这样做是为了揭示程序的算法结构，这更可能是读者想知道的！

任何以牺牲算法清晰度为代价坚持句法清晰度的编码标准都应该被重写。 如果您的雇主因使用此技巧而解雇您，请在保安人员将您拖出大楼时反复告诉他们。

# Refinements and Code

在一个严肃的应用程序中，这个玩具协程实现不太可能有用，因为它依赖于静态变量，因此它无法重入或多线程。 理想情况下，在实际应用程序中，您可能希望能够在多个不同的上下文中调用相同的函数，并且在给定上下文中的每次调用中，控制在同一上下文中的最后一次返回之后立即恢复。

这很容易完成。 我们安排了一个额外的函数参数，它是一个指向上下文结构的指针； 我们将所有本地状态和协程状态变量声明为该结构的元素。

这有点难看，因为突然间你不得不使用`ctx->i`作为循环计数器，而你以前只会使用`i`； 实际上，您所有的重要变量都成为协程上下文结构的元素。 但它消除了重入的问题，并且仍然没有影响例程的结构。

（当然，如果 C 只有 Pascal 的 with 语句，我们可以安排宏来使这个间接层也真正透明。遗憾的是，至少 C++ 用户可以通过让他们的协程成为类成员来管理它， 并将其所有局部变量保留在类中，以便范围是隐式的。）

这里包含一个 C 头文件，它将这个协程技巧实现为一组预定义的宏。 文件中定义了两组宏，前缀为 scr 和 ccr。 scr 宏是该技术的简单形式，当您可以使用静态变量时； ccr 宏提供高级可重入表单。 完整文档在头文件本身的注释中给出。

请注意，`Visual C++`版本 6 不喜欢这种协程技巧，因为它的默认调试状态（用于编辑和继续的程序数据库）对`__LINE__`宏做了一些奇怪的事情。 要使用`VC++6`编译使用协程的程序，必须关闭“编辑并继续”。 （在项目设置中，转到“C/C++”选项卡，类别“常规”，设置“调试信息”。选择“用于编辑和继续的程序数据库”以外的任何选项。）

---

参考文献：https://www.chiark.greenend.org.uk/~sgtatham/coroutines.html
