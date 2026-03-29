---
title: "ARM汇编——Condition Flags and Codes"
date: 2025-03-06T22:06:52+08:00

type: "post"

categories: ["Embedded"]
tags: ["ARM", "Assembly"]

draft: false
---

每一种实用的通用计算架构都存在有条件执行某些代码的机制。 例如，这种机制用于实现 C 语言中的 if 结构，此外还有其他一些不太常见的情况；与许多其他架构一样，Arm 也使用一组标记（flags）来实现条件执行，这些标记存储了有关先前操作的状态信息。
## 案例

请看一段简单的 C 代码：

```c
for (i = 10; i != 0; i--) {
    do_something();
}
```

编译器可按如下方式实现该结构：

```assembly
mov r4, 10
loop_label:
	bl 		do_something
	sub 	r4, r4, #1
	cmp 	r4, #0
	bne 	loop_label
```

最后两条指令尤其值得关注。`cmp`指令将`r4`和`0`进行比较，如果`cmp`的结果是`not equal`（不等于），那么`bne`指令只是一条`b(branch)`条件指令。代码之所以能运行，是因为`cmp`指令设置了一些全局标志，表示操作的各种属性。而`bne`指令实际上只是一个带有 `ne`条件代码后缀的 `b(branch)`指令，通过读取这些标志来决定是否进行分支跳转。



下面的代码实现了一个更有效的解决方案：

```assembly
mov r4, 10
loop_label:
	bl 		do_something
	subs 	r4, r4, #1
	bne 	loop_label
```

在 `sub` 指令添加后缀` s` 会使其根据运算结果自行更新标志。 这个后缀可以添加到许多（但不是所有）算术和逻辑运算中。

文章的后续，我将解释`CPSR条件标志`是什么、它们存储在哪里，以及如何使用条件代码来测试它们。



## The Flags

设置条件标志的最简单方法是使用比较操作，如`cmp`。 这种机制在许多处理器架构中都很常见，而且 `cmp`的语义（如果不考虑细节的话）可能也很相似。此外，我们已经看到，许多指令（如示例中的` sub`）都可以通过添加 `s` 后缀进行修改，并根据结果更新条件标志。 这一切都很好，但标志存储的是什么信息，我们又该如何访问这些信息呢？

附加信息存储在 `APSR（应用处理器状态寄存器）`或 `CPSR（当前处理器状态寄存器）`中，这些标记表示简单的属性，如结果是否为负数，并通过各种组合来检测更高层次的关系，如 "大于 "等。 描述完标记后，我将解释它们如何映射到条件代码（如上一示例中的 `ne`）。

> 在 `Armv7` 上，尽管`APSR`和` CPSR`有不同的名称，但实际是一样的，仅几个条件标志是为`APSR`定义的，其他无法访问，所以无论如何都不应该直接访问`APSP`的其他位，`APSR`本质是对`CPSR`在应用层的精简。
>
> 注意：**`GCC < 4.3.3`版本不支持`APSR`**，如果要访问，则必须在汇编代码中使用`CPSR`。

---

### N: Negative

`N`标志置位：说明指令结果为负

### Z: Zero

`Z`标志置位：说明指令结果为零

### C: Carry (or Unsigned Overflow)

`C`标志置位：说明**无符号运算的结果溢出 32 位寄存器**（该位可用于实现 64 位无符号运算）

### V: (Signed) Overflow

`V`标志置位原理和`C`标志一致都是溢出，但是**有符号运算**。

例如，`0x7fffffff`是32位所表示最大的二进制整数，当`0x7fffffff + 0x7fffffff`会造成有符号溢出，**此时`V`标志置位**，但不是一个无符号溢出/进位 (`0xfffffffe`)，所以`C`标志不置位。



### 标志设置案例

请看下面的例子：

```assembly
ldr r1, =0xffffffff
ldr r2, =0x00000001
adds r0, r1, r2
```

操作计算的结果将会是`0x100000000`最高位会被丢弃，因为32位目标寄存器无法储存，所以最终的真实结果是：`0x00000000`，案例的标志位最终结果：

| Flag  | Explanation                                                  |
| ----- | :----------------------------------------------------------- |
| N = 0 | 结果为0，被认为是正值，因此`N（negative）`位置0              |
| Z = 1 | 结果为0，所以`Z(zero)`位置1                                  |
| C = 1 | 由于运算结果无法容纳 32 位寄存器 最高位被丢弃，所以处理器会将 `C（carry）`位置 1 |
| V = 0 | 从二进制带符号的算术角度，`0xffffffffff`实际上表示`-1`，因此所做的运算实际上是`(-1) + 1 = 0`，该运算显然没有溢出，因此`V（overflow）`位置 0 |

如果你感兴趣，你可以通过`ccdemo`应用程序进行检查，输出类似：

```assembly
$ ./ccdemo adds 0xffffffff 0x1
The results (in various formats):
       Signed:         -1 adds          1 =          0
     Unsigned: 4294967295 adds          1 =          0
  Hexadecimal: 0xffffffff adds 0x00000001 = 0x00000000
Flags:
  N (negative): 0
  Z (zero)    : 1
  C (carry)   : 1
  V (overflow): 0
Condition Codes:
  EQ: 1    NE: 0
  CS: 1    CC: 0
  MI: 0    PL: 1
  VS: 0    VC: 1
  HI: 0    LS: 1
  GE: 1    LT: 0
  GT: 0    LE: 1
```



## 如何读取标志

我们已经搞清楚如何去设置标志，但是去有条件执行一些代码的结果是什么？如果你不能去反应他们，那么去设置这些标志是无意义的。

最常见的测试标志的方式是使用条件执行指令，这个指令和其他结构是相似的，如果你熟悉其他指令，那么你可能识别，



我们已经知道了如何设置标记，但如何才能有条件地执行某些代码呢？ 如果不能对标记做出反应，那么设置标记就毫无意义。 

测试标记的最常用方法是使用条件执行代码。 这种机制与其他体系结构中使用的机制类似，因此，如果你熟悉其他机器，你可能会识别出以下模式，它与 C 的 if/else 结构一一对应：

```assembly
cmp		r0, #20
bhi		do_something_else
do_someing:
	@ This code runs if (r0 <= 20).
	b 	continue		@Prevent do_something_else from executing.
do_something_else:
	@ This code runs if (r0 > 20).
continue:
	@ Other code
```

实际上，将其中一个条件代码附加到指令中，如果条件为真，指令就会执行。 否则，指令什么也不执行，基本上就是一个 `nop`。

下表列出了可用的条件代码，其含义（标志位由`cmp`或`subs`指令设置）以及要测试的标志位：

| Code            | Meaning(for cmp or subs)                              | Flags Tested           |
| --------------- | ----------------------------------------------------- | ---------------------- |
| eq              | Equal                                                 | Z==1                   |
| ne              | Not equal                                             | Z==0                   |
| cs or hs        | Unsigned highter or same (or carry set)               | C==1                   |
| cc or lo        | Unsigned Lower (or carry clear)                       | C==0                   |
| mi              | Negative. The mnemonic stands for "minus"             | N==1                   |
| pl              | Positive or zero. The mnemonic stands for "plus"      | N==0                   |
| vs              | Signed overflow. The mnemonic stands for "V set"      | V==1                   |
| vc              | No signed overflow. The mnemonic stands for "V Clear" | V==0                   |
| hi              | Unsigned higher                                       | （C==1)&& （Z==0）    |
| ls              | Unsigned lower or same                                | （C==0）&& （Z==1）    |
| ge              | Signed greater than or equal                          | N==V                   |
| lt              | Signed less than                                      | N!=V                   |
| gt              | Signed greater than                                   | （Z==0）&& （N==V）    |
| le              | Signed less than or equal                             | （Z==1）\|\|  （N!=V） |
| al(or ommitted) | Always executed                                       | None tested            |

前几个代码的原理非常明显，因为它们测试的是单个标志，但其他代码依赖于特定组合的标志。在实际操作中，你很少需要知道具体发生了什么，助记符掩盖了比较的复杂性。



下面是我之前给出的 for 循环代码示例：

```assembly
mov r4, 10
loop_label:
	bl 		do_something
	subs 	r4, r4, #1
	bne 	loop_label
```

现在应该很容易弄清楚这里到底发生了什么：

- `subs`指令根据`r4 - 1`的结果设置标志。特别是，如果结果是0，`Z`标志位将被置位（`Z=1`），如果结果是其他，`Z`标志将被复位(`Z=0`)。
- `bne`指令只有在`ne`条件为真的情况下才会执行，而只有当`Z=0`时，`ne`条件才为真，所以`bne`会遍历循环，直到`Z`被置1（即此时`r4 = 0`）。



## 专用比较指令

`cmp`指令（我们在第一个案例看到过）可以看做是一条不储存结果的`sub`指令；如果两个操作数相同，减法的结果将为零，因为`eq`和`Z`标志之间存在映射关系。 当然，我们也可以使用假寄存器的`sub`指令，但只有在有空余寄存器的情况下才能这样做。 因此，专用比较指令非常常用。



实际上有四条专用比较指令，它们执行的操作如下表所示：

| Instruction | Description                                           |
| ----------- | ----------------------------------------------------- |
| **`cmp`**   | Works like **`subs`**, but does not store the result. |
| **`cmn`**       | Works like **`adds`**, but does not store the result. |
| **`tst`**   | Works like **`ands`**, but does not store the result. |
| **`teq`**   | Works like **`eors`**, but does not store the result. |

请注意，专门的比较操作不需要后缀 s；它们只更新标志，因此后缀是多余的。



## 结束语

虽然条件标志机制在原理上相当简单，但需要考虑的细节很多，看一些实际例子可能会有所帮助！ 我将在今后的博文中介绍一些实际使用的例子。

**[文章翻译来源]：**[condition-codes-1-condition-flags-and-codes](https://community.arm.com/arm-community-blogs/b/architectures-and-processors-blog/posts/condition-codes-1-condition-flags-and-codes)

更好的翻译不如原文的表达精确，希望大家有能力的情况下，阅读英文原文章！
