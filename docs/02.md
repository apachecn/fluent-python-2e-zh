<link href="Styles/Style00.css" rel="stylesheet" type="text/css"> <link href="Styles/Style01.css" rel="stylesheet" type="text/css"> 

# 第一章。Python 数据模型

> Guido 对语言设计美学的感觉是惊人的。我见过许多优秀的语言设计者，他们可以编写出理论上很漂亮但没人会使用的语言，但是 Guido 是少数几个可以编写出理论上不太漂亮但却能带来快乐的语言的人之一。
> 
> Jim Hugunin，Jython 的创建者，AspectJ 的共同创建者，的架构师。net DLR[1]

Python 最好的品质之一是它的一致性。在使用 Python 一段时间后，您就能够开始对新特性做出明智而正确的猜测。

但是，如果你在 Python 之前学过另一种面向对象的语言，你可能会觉得用`len(collection)`代替`collection.len()`很奇怪。这个明显的奇怪现象只是冰山一角，如果正确理解，它就是我们所说的*蟒蛇*的关键。冰山被称为 Python 数据模型，它是我们用来使我们自己的对象与最地道的语言特性很好地配合的 API。

你可以把数据模型想象成 Python 作为框架的描述。它形式化了语言本身的构造块的接口，比如序列、函数、迭代器、协同程序、类、上下文管理器等等。

当使用一个框架时，我们花费大量的时间编写框架调用的方法。当我们利用 Python 数据模型来构建新的类时，也会发生同样的情况。Python 解释器调用特殊方法来执行基本的对象操作，通常由特殊语法触发。 特殊方法名总是以双下划线开头和结尾。例如，`__getitem__`特殊方法支持语法`obj[key]`。为了评估`my_collection[key]`，解释器调用`my_collection.__getitem__(key)`。

当我们希望我们的对象支持基本的语言结构并与之交互时，我们实现特殊的方法，例如:

*   收集

*   属性访问

*   迭代(包括使用`async for`的异步迭代)

*   运算符重载

*   函数和方法调用

*   字符串表示和格式

*   使用`await`进行异步编程

*   对象创建和销毁

*   使用`with`或`async with`语句的托管上下文

# 魔术和邓德

术语*魔法方法*是特殊方法的俚语，但是我们如何谈论像`__getitem__`这样的特殊方法呢？我从作家兼教师史蒂夫·霍尔登那里学会了说“dunder-getitem”。“Dunder”是“前后双下划线”的快捷方式这就是为什么特殊方法也被称为 *dunder 方法*的原因。Python 语言参考文献中的[【词法分析】](https://fpy.li/1-3)一章警告说，“*在任何上下文中，任何不遵循明确记录的使用的`__*__`名称的*使用都可能会在没有警告的情况下遭到破坏。”

# 本章的新内容

这一章与第一版相比几乎没有什么变化，因为它是对 Python 数据模型的介绍，这是相当稳定的。最显著的变化是:

*   支持异步编程和其他新特性的特殊方法，添加到“特殊方法概述”的表格中。

*   图 1-2 展示了“集合 API”中特殊方法的使用，包括 Python 3.6 中引入的`collections.abc.Collection`抽象基类。

此外，在这里和整个第二版中，我采用了 Python 3.6 中引入的 *f-string* 语法，它比旧的字符串格式符号更易读，也更方便:方法 `str.format()`和操作符`%`。

###### 小费

仍然使用`my_fmt.format()`的一个原因是`my_fmt`的定义必须在代码中不同于格式化操作需要发生的地方。例如，当`my_fmt`有多行并且最好在常量中定义时，或者当它必须来自配置文件或数据库时。这些都是真实的需求，但并不经常发生。

# 一副巨蟒牌

示例 1-1 很简单，但是它展示了实现两个特殊方法`__getitem__`和`__len__`的强大功能。

##### 示例 1-1：作为一系列扑克牌的一副牌

```
import collections

Card = collections.namedtuple('Card', ['rank', 'suit'])

class FrenchDeck:
    ranks = [str(n) for n in range(2, 11)] + list('JQKA')
    suits = 'spades diamonds clubs hearts'.split()

    def __init__(self):
        self._cards = [Card(rank, suit) for suit in self.suits
                                        for rank in self.ranks]

    def __len__(self):
        return len(self._cards)

    def __getitem__(self, position):
        return self._cards[position]
```

首先要注意的是使用`collections.namedtuple`来构造一个简单的类来表示各个卡片。我们使用`namedtuple`来构建对象类，这些对象只是没有定制方法的属性束，比如数据库记录。在本例中，我们使用它来为一副牌中的牌提供一个很好的表示，如控制台会话所示:

```
>>> beer_card = Card('7', 'diamonds')
>>> beer_card
Card(rank='7', suit='diamonds')
```

但是这个例子的重点是`FrenchDeck`类。它很短，但是很有力。首先，像任何标准的 Python 集合一样，一副牌通过返回其中牌的数量来响应 `len()`函数:

```
>>> deck = FrenchDeck()
>>> len(deck)
52
```

由于采用了`__getitem__`方法，从一副牌中读取特定的牌(比如第一张或最后一张)很容易:

```
>>> deck[0]
Card(rank='2', suit='spades')
>>> deck[-1]
Card(rank='A', suit='hearts')
```

我们是否应该创建一个随机选择卡片的方法？不需要。Python 已经有了从序列中获取随机项的函数:`random.choice`。我们可以在 deck 实例中使用它:

```
>>> from random import choice
>>> choice(deck)
Card(rank='3', suit='hearts')
>>> choice(deck)
Card(rank='K', suit='spades')
>>> choice(deck)
Card(rank='2', suit='clubs')
```

我们刚刚看到了使用特殊方法利用 Python 数据模型的两个优点:

*   对于标准操作，类的用户不必记住任意的方法名。(“如何获取物品数量？是`.size()`、`.length()`，还是什么？”)

*   更容易从丰富的 Python 标准库中受益，避免重复发明轮子，比如`random.choice`函数。

但事情会变得更好。

因为我们的`__getitem__`委托给了`self._cards`的 `[]`操作符，所以我们的卡牌自动支持切片。下面是我们如何查看一副全新牌中的前三张牌，然后从索引 12 开始，一次跳过 13 张牌，只挑选 a:

```
>>> deck[:3]
[Card(rank='2', suit='spades'), Card(rank='3', suit='spades'),
Card(rank='4', suit='spades')]
>>> deck[12::13]
[Card(rank='A', suit='spades'), Card(rank='A', suit='diamonds'),
Card(rank='A', suit='clubs'), Card(rank='A', suit='hearts')]
```

仅仅通过实现`__getitem__`特殊方法，我们的卡片组也是可迭代的:

```
>>> for card in deck:  # doctest: +ELLIPSIS
...   print(card)
Card(rank='2', suit='spades')
Card(rank='3', suit='spades')
Card(rank='4', suit='spades')
...
```

我们也可以反过来迭代甲板:

```
>>> for card in reversed(deck):  # doctest: +ELLIPSIS
...   print(card)
Card(rank='A', suit='hearts')
Card(rank='K', suit='hearts')
Card(rank='Q', suit='hearts')
...
```

# 文档测试中的省略

只要 可能，我就从 [`doctest`](https://fpy.li/doctest) 中提取本书中的 Python 控制台清单以确保准确性。当输出太长时，省略的部分用省略号(`...`)标记，就像前面代码的最后一行一样。在这种情况下，我使用了`# doctest: +ELLIPSIS`指令让 doctest 通过。如果您正在交互式控制台中尝试这些示例，您可以完全省略 doctest 注释。

迭代通常是隐式的。如果一个集合没有 `__contains__`方法，那么`in`操作符会进行顺序扫描。典型的例子:`in`与我们的`FrenchDeck`类一起工作，因为它是可迭代的。看看这个:

```
>>> Card('Q', 'hearts') in deck
True
>>> Card('7', 'beasts') in deck
False
```

排序怎么样？一种常见的纸牌排名系统是按等级(a 为最高)，然后是花色，依次是黑桃(最高)、红心、方块和梅花(最低)。下面是一个根据该规则对牌进行排序的函数，对于梅花 2 返回`0`，对于黑桃 a 返回`51`:

```
suit_values = dict(spades=3, hearts=2, diamonds=1, clubs=0)

def spades_high(card):
    rank_value = FrenchDeck.ranks.index(card.rank)
    return rank_value * len(suit_values) + suit_values[card.suit]
```

给定`spades_high`，我们现在可以按照等级递增的顺序列出我们的牌组:

```
>>> for card in sorted(deck, key=spades_high):  # doctest: +ELLIPSIS
...      print(card)
Card(rank='2', suit='clubs')
Card(rank='2', suit='diamonds')
Card(rank='2', suit='hearts')
... (46 cards omitted)
Card(rank='A', suit='diamonds')
Card(rank='A', suit='hearts')
Card(rank='A', suit='spades')
```

虽然`FrenchDeck`隐式地继承了`object`类，但是它的大部分功能不是继承的，而是来自于利用数据模型和组合。通过实现特殊的方法`__len__`和`__getitem__`，我们的`FrenchDeck`表现得像一个标准的 Python 序列，允许它受益于核心语言特性(例如，迭代和切片)和标准库，如使用`random.choice`、、`reversed`、和`sorted`的例子所示。多亏了组合，`__len__`和`__getitem__`实现可以将所有工作委托给一个`list`对象`self._cards`。

# 洗牌怎么样？

到目前为止，一个`FrenchDeck`不能被混洗，因为它是*不可变的*:卡片和它们的位置不能被改变，除非违反封装并直接处理`_cards`属性。在第 13 章的中，我们将通过添加一行`__setitem__`方法来解决这个问题。

# 如何使用特殊方法

关于特殊方法，首先要知道的是它们应该由 Python 解释器调用，而不是由你来调用。你不写`my_object.__len__()`。您编写`len(my_object)`，如果`my_object`是用户定义类的实例，那么 Python 调用您实现的`__len__`方法。

但是解释器在处理像`list`、`str`、`bytearray`这样的内置类型或者像 NumPy 数组这样的扩展时会走捷径。用 C 编写的 Python 可变大小集合包括一个名为`PyVarObject`的结构 [2] ，其中有一个`ob_size`字段保存集合中的项数。因此，如果`my_object`是其中一个内置的实例，那么`len(my_object)`将检索`ob_size`字段的值，这比调用一个方法要快得多。

通常，特殊的方法调用是隐式的。例如，语句`for i in x:`实际上导致了对`iter(x)`的调用，而后者可能会调用`x.__iter__()`(如果有的话),或者使用`x.__getitem__()`,就像在`FrenchDeck`的例子中一样。

通常，您的代码不应该有很多对特殊方法的直接调用。除非您正在进行大量的元编程，否则您应该更多地实现特殊的方法，而不是显式地调用它们。用户代码经常直接调用的唯一特殊方法是`__init__`在你自己的`__init__`实现中调用超类的初始化器。

如果需要调用一个特殊的方法，通常最好调用相关的内置函数(如`len`、`iter`、`str`等)。).这些内置调用相应的特殊方法，但通常提供其他服务，并且对于内置类型来说，比方法调用更快。例如，参见第 17 章中的“使用具有可调用性的 ITER”。

在下一节中，我们将看到特殊方法的一些最重要的用途:

*   模拟数字类型

*   对象的字符串表示

*   对象的布尔值

*   实现集合

## 模拟数字类型

几个 特殊方法允许用户对象响应`+`等操作符。我们将在第 16 章中更详细地介绍这一点，但这里我们的目标是通过另一个简单的例子进一步说明特殊方法的使用。

我们将实现一个类来表示二维向量——也就是像数学和物理中使用的欧几里得向量(见图 1-1 )。

###### 小费

内置的`complex`类型可以用来表示二维向量，但是我们的类可以扩展来表示 *n* 维向量。我们将在第 17 章中讨论这个问题。

![2D vectors](Images/flpy_0101.png)

###### 图 1-1。二维向量加法的例子；Vector(2，4) + Vector(2，1)产生 Vector(4，5)。

我们将通过编写一个模拟的控制台会话来开始为这样一个类设计 API，稍后我们可以将它用作 doctest。以下代码片段测试了图 1-1 中的矢量加法:

```
>>> v1 = Vector(2, 4)
>>> v2 = Vector(2, 1)
>>> v1 + v2
Vector(4, 5)
```

注意操作符`+`如何产生一个新的`Vector`，以友好的格式显示在控制台上。

`abs`内置函数返回整数和浮点数的绝对值，以及`complex`数字的大小，所以为了一致，我们的 API 也使用`abs`来计算一个向量的大小:

```
>>> v = Vector(3, 4)
>>> abs(v)
5.0
```

我们

```
>>> v * 3
Vector(9, 12)
>>> abs(v * 3)
15.0
```

示例 1-2 是一个`Vector`类，通过使用特殊的 方法`__repr__`、`__abs__`、`__add__`和`__mul__`，实现了刚才描述的操作。

##### 示例 1-2：一个简单的二维向量类

```
"""
vector2d.py: a simplistic class demonstrating some special methods

It is simplistic for didactic reasons. It lacks proper error handling,
especially in the ``__add__`` and ``__mul__`` methods.

This example is greatly expanded later in the book.

Addition::

 >>> v1 = Vector(2, 4)
 >>> v2 = Vector(2, 1)
 >>> v1 + v2
 Vector(4, 5)

Absolute value::

 >>> v = Vector(3, 4)
 >>> abs(v)
 5.0

Scalar multiplication::

 >>> v * 3
 Vector(9, 12)
 >>> abs(v * 3)
 15.0

"""

import math

class Vector:

    def __init__(self, x=0, y=0):
        self.x = x
        self.y = y

    def __repr__(self):
        return f'Vector({self.x!r}, {self.y!r})'

    def __abs__(self):
        return math.hypot(self.x, self.y)

    def __bool__(self):
        return bool(abs(self))

    def __add__(self, other):
        x = self.x + other.x
        y = self.y + other.y
        return Vector(x, y)

    def __mul__(self, scalar):
        return Vector(self.x * scalar, self.y * scalar)
```

除了熟悉的`__init__`之外，我们还实现了五个特殊的方法。请注意，它们都不是在类中直接调用的，也不是在 doctests 所演示的类的典型用法中直接调用的。如前所述，Python 解释器是大多数特殊方法的唯一频繁调用者。

示例 1-2 实现了两个运算符:`+`和`*`，展示了`__add__`和`__mul__`的基本用法。在这两种情况下，这些方法都创建并返回一个新的`Vector`实例，并且不修改任何一个操作数——仅仅是读取`self`或`other`。这是中缀操作符的预期行为:创建新对象而不触及它们的操作数。在第 16 章中，我会有更多的话要说。

###### 警告

如实现的那样，示例 1-2 允许一个`Vector`乘以一个数，但不允许一个数乘以一个`Vector`，这违背了标量乘法的可交换性。我们将用第 16 章中的特殊方法`__rmul__`来解决这个问题。

在下面的章节中，我们将讨论`Vector`中的其他特殊方法。

## 字符串表示

`__repr__`特殊方法由`repr`内置调用，以获取被检查对象的字符串表示。如果没有自定义的`__repr__`，Python 的控制台会显示一个`Vector`实例`<Vector object at 0x10e100070>`。

交互控制台和调试器对表达式求值的结果调用`repr`，经典格式中的 `%r`占位符使用`%`运算符，新的[格式字符串语法](https://fpy.li/1-4)中的 `!r` 转换字段使用 *f 字符串*`str.format`方法。

注意，我们的`__repr__`中的 *f 字符串*使用`!r`来获取要显示的属性的标准表示。这是一个很好的实践，因为它显示了`Vector(1, 2)`和`Vector('1', '2')`之间的关键区别——后者在这个例子的上下文中不起作用，因为构造函数的参数应该是数字，而不是`str`。

由`__repr__`返回的字符串应该是明确的，如果可能的话，应该与重新创建所表示的对象所需的源代码相匹配。这就是为什么我们的`Vector`表示看起来像是在调用类的构造函数(例如，`Vector(3, 4)`)。

在 对比中，`__str__`由`str()`内置调用，由`print`函数隐式使用。它应该返回一个适合向最终用户显示的字符串。

有时由`__repr__`返回的相同字符串是用户友好的，你不需要编码`__str__`，因为从`object`类继承的实现调用`__repr__`作为后备。示例 5-2 是本书中带有自定义`__str__`的几个示例之一。

###### 小费

有使用`toString`方法的语言经验的程序员倾向于实现`__str__`而不是`__repr__`。如果在 Python 中只实现了这些特殊方法中的一个，选择`__repr__`。

[“Python 中的`__str__`和`__repr__`有什么区别？”](https://fpy.li/1-5)是 Pythonistas Alex Martelli 和 Martijn Pieters 的优秀贡献的堆栈溢出问题。

## 自定义类型的布尔值

虽然 Python 有一个`bool`类型，但它接受布尔上下文中的任何对象，比如控制`if`或`while`语句的表达式，或者作为`and`、`or`和`not`的操作数。为了确定一个值`x`是*真值*还是*假值*，Python 应用了`bool(x)`，它返回`True`或`False`。

默认情况下，用户定义类的实例被认为是真的，除非实现了`__bool__`或`__len__`。基本上，`bool(x)`调用`x.__bool__()`并使用结果。如果`__bool__`没有实现，Python 尝试调用`x.__len__()`，如果返回 0，`bool`返回`False`。否则`bool`返回`True`。

我们对`__bool__`的实现在概念上很简单:如果向量的大小为零，它返回`False`，否则返回`True`。我们使用`bool(abs(self))`将数量转换为布尔值，因为`__bool__`应该返回一个布尔值。在`__bool__`方法之外，很少需要显式调用`bool()`，因为任何对象都可以在布尔上下文中使用。

请注意特殊方法`__bool__`如何允许您的对象遵循 Python 标准库文档*的[章节](https://fpy.li/1-6)中定义的真值测试规则。*

###### 注意

`Vector.__bool__`的一个更快的实现是:

```
    def __bool__(self):
        return bool(self.x or self.y)
```

这更难阅读，但避免了通过`abs`、`__abs__`、正方形和平方根的旅程。需要到`bool`的显式转换，因为`__bool__`必须返回一个布尔值，而`or`按原样返回任一操作数:`x or y`如果为真，则计算结果为`x`，否则结果为`y`，无论它是什么。

## 收集 API

图 1-2 文档 语言中必不可少的集合类型的接口。图中所有的类都是 ABCs— *抽象基类*。ABCs 和`collections.abc`模块包含在第 13 章中。这一简短部分的目标是给出 Python 最重要的集合接口的全景，展示它们是如何通过特殊方法构建的。

![UML class diagram with all superclasses and some subclasses of `abc.Collection`](Images/flpy_0102.png)

###### 图 1-2。具有基本集合类型的 UML 类图。*斜体*中的方法名是抽象的，所以必须由`list`和`dict`等具体子类实现。其余的方法有具体的实现，因此子类可以继承它们。

每个顶级 ABC 都有一个特殊的方法。`Collection`ABC(Python 3.6 中的新功能)统一了三个基本接口，每个集合都应该实现:

*   `Iterable`到到支持`for`、[解包](https://fpy.li/1-7)，以及其他形式的迭代

*   `Sized`到T34 支持`len`内置功能

*   `Container`到到支持`in`操作符

Python 不需要具体的类来继承这些 ABC。任何实现`__len__`的类都满足`Sized`接口。

`Collection`的三个非常重要的专业是:

*   `Sequence`，形式化`list`和`str`等内置的接口

*   `Mapping`，由`dict`、`collections.defaultdict`等实现。

*   `Set`、`set`和`frozenset`内置类型的接口

只有`Sequence`是`Reversible`，因为序列支持其内容的任意排序，而映射和集合不支持。

###### 注意

从 Python 3.7 开始，`dict`类型被正式“排序”，但这仅仅意味着键插入顺序被保留。你不能随心所欲地重新排列`dict`中的键。

`Set` ABC 中的所有特殊方法都实现了中缀运算符。例如， `a & b` 计算集合`a`和`b`的交集，在`__and__`特殊方法中实现。

接下来的两章将详细介绍标准库序列、映射和集合。

现在让我们考虑一下 Python 数据模型中定义的特殊方法的主要类别。

# 特殊方法概述

*的 [【数据模型】章节](https://fpy.li/dtmodel)Python 语言参考*列出了 80 多个特殊方法名称。其中一半以上实现了算术、位和比较运算符。作为可用内容的概述，请参见下表。

表 1-1 显示了特殊的方法名，不包括那些用于实现中缀运算符或核心数学函数的方法名，如`abs`。这些方法中的大部分将在整本书中涵盖，包括最近增加的:异步特殊方法，如`__anext__`(在 Python 3.5 中增加)，以及类定制钩子`__init_subclass__`(来自 Python 3.6)。

Table 1-1\. Special method names (operators excluded)

| 种类 | 方法名称 |
| --- | --- |
| 字符串/字节表示 | `__repr__ __str__ __format__ __bytes__ __fspath__` |
| 转换为数字 | `__bool__ __complex__ __int__ __float__ __hash__ __index__` |
| 模拟集合 | `__len__ __getitem__ __setitem__ __delitem__` `__contains__` |
| 循环 | `__iter__ __aiter__ __next__ __anext__ __reversed__` |
| 可调用或协同执行 | `__call__ __await__` |
| 上下文管理 | `__enter__ __exit__ __aexit__ __aenter__` |
| 实例创建和销毁 | `__new__ __init__ __del__` |
| 属性管理 | `__getattr__ __getattribute__ __setattr__ __delattr__ __dir__` |
| 属性描述符 | `__get__ __set__ __delete__ __set_name__` |
| 抽象基类 | `__instancecheck__ __subclasscheck__` |
| 类元编程 | `__prepare__ __init_subclass__ __class_getitem__ __mro_entries__` |

表 1-2 中列出的特殊方法支持中缀和数值运算符。这里最近的名字是`__matmul__`、`__rmatmul__`和`__imatmul__`，它们是在 Python 3.5 中添加的，以支持将`@`用作矩阵乘法的中缀运算符，我们将在第 16 章中看到。

Table 1-2\. Special method names and symbols for operators

| 操作员类别 | 标志 | 方法名称 |
| --- | --- | --- |
| 一元数字 | `- + abs()` | `__neg__ __pos__ __abs__` |
| 丰富的比较 | `< <= == != > >=` | `__lt__ __le__ __eq__ __ne__ __gt__ __ge__` |
| 算术 | `+ - * / // % @ divmod() round() ** pow()` | `__add__ __sub__ __mul__` `__truediv__` `__floordiv__ __mod__`:T25 `__matmul__` `__divmod__ __round__ __pow__` |
| 反向算术 | (交换操作数的算术运算符) | `__radd__ __rsub__ __rmul__ __rtruediv__ __rfloordiv__ __rmod__ __rmatmul__ __rdivmod__ __rpow__` |
| 扩充赋值算法 | `+= -= *= /= //= %= @= **=` | `__iadd__ __isub__ __imul__ __itruediv__ __ifloordiv__ __imod__ __imatmul__ __ipow__` |
| 按位 | `& &#124; ^ << >> ~` | `__and__ __or__ __xor__ __lshift__ __rshift__ __invert__` |
| 反向逐位 | (交换操作数的按位运算符) | `__rand__ __ror__ __rxor__ __rlshift__ __rrshift__` |
| 扩充赋值按位 | `&= &#124;= ^= <<= >>=` | `__iand__ __ior__ __ixor__ __ilshift__ __irshift__` |

###### 注意

当第一个操作数上对应的特殊方法无法使用时，Python 会在第二个操作数上调用反向运算符特殊方法。扩充赋值是将中缀运算符与变量赋值结合起来的快捷方式，例如`a += b`。

第 16 章详细解释了反向运算符和增广赋值。

# 为什么 len 不是一种方法

我 在 2013 年向核心开发者 Raymond Hettinger 问过这个问题，他回答的关键是《Python 的禅》](https://fpy.li/1-8)中的一句话:“实用性战胜了纯粹性。”在[“如何使用特殊方法”中，我描述了当`x`是一个内置类型的实例时`len(x)`如何运行得非常快。没有为 CPython 的内置对象调用任何方法:长度只是从 C struct 的一个字段中读取。获取集合中的项目数是一个常见的操作，必须有效地用于基本的和不同的类型，如`str`、`list`、`memoryview`等等。

换句话说，`len`没有被作为方法调用，因为它作为 Python 数据模型的一部分得到了特殊处理，就像`abs`一样。但是多亏了特殊的方法`__len__`，你也可以让`len`和你自己的定制对象一起工作。这是对高效内置对象的需求和语言一致性之间的合理折衷。同样出自《Python 的禅》:“特例不会特殊到违反规则。”

###### 注意

如果您认为`abs`和`len`是一元操作符，您可能更倾向于原谅它们的功能外观，而不是面向对象语言中的方法调用语法。事实上，ABC 语言 Python 的直接祖先，开创了 Python 的许多特性——有一个相当于`len`的`#`操作符(你可以写`#s`)。当作为中缀操作符使用时，编写为`x#s`，它计算`s`中`x`的出现次数，在 Python 中你得到的是`s.count(x)`，对于任何序列`s`。

# 章节摘要

通过实现特殊的方法，你的对象可以像内置类型一样工作，实现社区认为的 Pythonic 式的表达性编码风格。

Python 对象的一个基本要求是提供可用的字符串表示，一个用于调试和日志记录，另一个用于呈现给最终用户。这就是数据模型中存在特殊方法`__repr__`和`__str__`的原因。

如`FrenchDeck`示例所示，仿真序列是特殊方法最常见的用途之一。例如，数据库库经常返回包装在类似序列的集合中的查询结果。充分利用现有的序列类型是第 2 章的主题。当我们创建一个`Vector`类的多维扩展时，实现你自己的序列将在第 12 章中讨论。

由于操作符重载，Python 提供了丰富的数字类型选择，从内置的到`decimal.Decimal`和`fractions.Fraction`，都支持中缀算术操作符。 *NumPy* 数据科学库支持带有矩阵和张量的中缀运算符。通过增强 `Vector` 示例，实现操作符——包括反向操作符和增强赋值——将在第 16 章中显示。

本书涵盖了 Python 数据模型的大多数其他特殊方法的使用和实现。

# 进一步阅读

[【数据模型】章节](https://fpy.li/dtmodel)*Python 语言参考*是本章和本书大部分内容的权威来源。

[*巨蟒一言以蔽之*，第 3 版。](https://fpy.li/pynut3)Alex Martelli、Anna Ravenscroft 和 Steve Holden (O'Reilly)对数据模型进行了出色的介绍。除了 CPython 的实际 C 源代码之外，他们对属性访问机制的描述是我见过的最权威的。Martelli 也是 Stack Overflow 的一个多产贡献者，发布了超过 6200 个回答。在[堆栈溢出](https://fpy.li/1-9)查看他的用户配置文件。

David Beazley 有两本书在 Python 3 的上下文中详细介绍了数据模型: [*Python 基本参考*](https://dabeaz.com/per.html) ，第 4 版。(Addison-Wesley)，以及 [*Python Cookbook* ，第 3 版。与布莱恩·k·琼斯合著。](https://fpy.li/pycook3)

[](https://mitpress.mit.edu/books/art-metaobject-protocol)*元对象协议的艺术(麻省理工学院出版社)Gregor Kiczales、Jim des Rivieres 和 Daniel G. Bobrow 解释了元对象协议的概念，Python 数据模型就是其中的一个例子。*

 **[1][《Jython 的故事》](https://fpy.li/1-1)，由萨缪尔·佩德罗尼(Samuele Pedroni)和诺埃尔·拉平(Noel Rappin，O'Reilly)为《Jython 精要[](https://fpy.li/1-2)*写的前言。*

*[2]C struct 是一种带有命名字段的记录类型。***