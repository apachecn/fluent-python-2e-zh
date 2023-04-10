<link href="Styles/Style00.css" rel="stylesheet" type="text/css"> <link href="Styles/Style01.css" rel="stylesheet" type="text/css"> 

# 第十八章。 with、match 和 else 块

> 上下文管理器可能会变得和子程序本身一样重要。我们对它们只是略知皮毛。……]Basic 有一个`with`语句，很多语言中都有`with`语句。但是他们做的事情不一样，他们都做一些很浅的事情，他们把你从重复的点状[属性]查找中拯救出来，他们不做设置和拆卸。不要因为同名就认为是一回事。`with`声明是一件大事。 ^([1)
> 
> 雷蒙德·赫廷格，雄辩的 Python 传道者

这一章是关于控制流的特性，这些特性在其他语言中并不常见，因此在 Python 中容易被忽略或利用不足。它们是:

*   `with`语句和上下文管理器协议

*   与`match/case`的模式匹配

*   `for`、`while`和`try`语句中的`else`子句

在上下文管理器对象的控制下,`with`语句建立一个临时上下文并可靠地拆除它。这可以防止错误并减少样板代码，同时使 API 更安全、更易于使用。除了自动关闭文件，Python 程序员还发现了`with`块的许多用途。

我们在前面的章节中已经看到了模式匹配，但是在这里我们将看到一种语言的语法是如何被表达为序列模式的。这个观察解释了为什么`match/case`是创建易于理解和扩展的语言处理器的有效工具。我们将研究一个完整的解释器，它是 Scheme 语言的一个小但功能强大的子集。同样的想法可以应用于开发一种模板语言或 DSL(领域特定语言),以在一个更大的系统中编码业务规则。

`else`条款没什么大不了的，但是当与`for`、`while`和`try`一起正确使用时，它确实有助于传达意图。

# 本章的新内容

“lis . py 中的模式匹配:案例研究”是一个新的部分。

我更新了“context lib 实用程序”以涵盖自 Python 3.6 以来添加的`contextlib`模块的一些特性，以及 Python 3.10 中引入的新的带括号的上下文管理器语法。

先说强有力的`with`语句。

# 上下文管理器和块

上下文管理器 对象的存在是为了控制一个`with`语句，就像迭代器的存在是为了控制一个`for`语句。

`with`语句旨在简化`try/finally`的一些常见用法，它保证在一段代码后执行一些操作，即使该代码段被`return`、异常或`sys.exit()`调用终止。`finally`子句中的代码通常会释放一个关键资源或恢复某个临时改变的先前状态。

Python 社区正在为上下文管理器寻找新的、创造性的用途。标准库中的一些示例如下:

*   管理`sqlite3`模块中的事务—参见[“将连接用作上下文管理器”](https://fpy.li/18-2)。

*   安全处理锁、条件和信号量—如 [`threading`模块文档](https://fpy.li/18-3)中所述。

*   使用`Decimal`对象为算术运算设置自定义环境—参见 [`decimal.localcontext`文档](https://fpy.li/18-4)。

*   修补测试对象—参见 [`unittest.mock.patch`功能](https://fpy.li/18-5)。

上下文管理器接口由`__enter__`和`__exit__`方法组成。在`with`的顶部，Python 调用上下文管理器对象的`__enter__`方法。当`with`块由于某种原因完成或终止时，Python 调用上下文管理器对象上的`__exit__`。

最常见的例子是确保文件对象被关闭。示例 18-1 是使用`with`关闭文件的详细演示。

##### 示例 18-1：作为上下文管理器的文件对象的演示

```
>>> withopen('mirror.py')asfp:①... src=fp.read(60)②...>>> len(src)60 >>> fp③<_io.TextIOWrapper name='mirror.py' mode='r' encoding='UTF-8'> >>> fp.closed,fp.encoding④(True, 'UTF-8') >>> fp.read(60)⑤Traceback (most recent call last):
 File "<stdin>", line 1, in <module>ValueError: I/O operation on closed file.
```

① `fp`被绑定到打开的文本文件，因为文件的`__enter__`方法返回`self`。

② 从`fp`中读取`60` Unicode 字符。

③ `fp`变量仍然可用— `with`块不像函数那样定义新的作用域。

④ 我们可以读取`fp`对象的属性。

⑤ 但是我们不能从`fp`中读取更多的文本，因为在`with`块的末尾，调用了`TextIOWrapper.__exit__`方法，它关闭了文件。

示例 18-1 中的第一个标注提出了一个微妙但重要的观点:上下文管理器对象是在`with`之后评估表达式的结果，但是绑定到目标变量的值(在`as`子句中)是由上下文管理器对象的`__enter__`方法返回的结果。

恰好`open()`函数返回`TextIOWrapper`的一个实例，它的`__enter__`方法返回`self`。但是在不同的类中，`__enter__`方法也可能返回一些其他对象，而不是上下文管理器实例。

当控制流以任何方式退出`with`块时，在上下文管理器对象上调用`__exit__`方法，而不是在由`__enter__`返回的任何对象上调用。

`with`语句的`as`子句是可选的。在`open`的情况下，我们总是需要它来获得对文件的引用，这样我们就可以对它调用方法。但是一些上下文管理器返回`None`,因为它们没有有用的对象返回给用户。

示例 18-2 展示了一个完全无关紧要的上下文管理器的操作，旨在突出上下文管理器和其`__enter__`方法返回的对象之间的区别。

##### 示例 18-2：测试驱动`LookingGlass`上下文管理器类

```
>>>frommirrorimportLookingGlass>>>withLookingGlass()aswhat:①...print('Alice, Kitty and Snowdrop')②...print(what)...pordwonSdnayttiK,ecilAYKCOWREBBAJ>>>what③'JABBERWOCKY'>>>print('Back to normal.')④Backtonormal.
```

① 上下文管理器是`LookingGlass`的一个实例；Python 在上下文管理器上调用`__enter__`，结果被绑定到`what`。

② 打印一个`str`，然后是目标变量`what`的值。每个`print`的输出将反向输出。

③ 现在`with`块结束了。我们可以看到`__enter__`返回的值，保存在`what`中，就是字符串`'JABBERWOCKY'`。

④ 程序输出不再反转。

示例 18-3 为`LookingGlass`的实施。

##### 示例 18-3：mirror . py:`LookingGlass`上下文管理器类的代码

```
importsysclassLookingGlass:def__enter__(self):①self.original_write=sys.stdout.write②sys.stdout.write=self.reverse_write③return'JABBERWOCKY'④defreverse_write(self,text):⑤self.original_write(text[::-1])def__exit__(self,exc_type,exc_value,traceback):⑥sys.stdout.write=self.original_write⑦ifexc_typeisZeroDivisionError:⑧print('Please DO NOT divide by zero!')returnTrue⑨⑩
```

① Python 调用`__enter__`，除了`self`之外没有任何参数。

② 保持原来的`sys.stdout.write`方法，这样我们以后就可以恢复了。

③ 猴子补丁`sys.stdout.write`，用我们自己的方法替换它。

④ 返回`'JABBERWOCKY'`字符串，这样我们就有东西可以放入目标变量`what`中。

⑤ 我们对`sys.stdout.write`的替换颠倒了`text`的论点，并调用原来的实现。

⑥ 如果一切顺利，Python 用`None, None, None`调用`__exit__`；如果出现异常，三个参数将获得异常数据，如本示例后所述。

⑦ 恢复原方法为`sys.stdout.write`。

⑧ 如果异常不是`None`并且其类型是`ZeroDivisionError`，则打印一条消息…

⑨ …并返回`True`来告诉解释器异常已被处理。

⑩ 如果`__exit__`返回`None`或任何*错误*值，则`with`块中产生的任何异常将被传播。

###### 小费

当真正的应用程序接管标准输出时，他们经常想用另一个类似文件的对象暂时替换`sys.stdout`，然后切换回原来的。 [`contextlib.redirect_stdout`](https://fpy.li/18-6) 上下文管理器正是这样做的:只需将代表`sys.stdout`的类似文件的对象传递给它。

解释器调用没有参数的`__enter__`方法——除了隐式的`self`。传递给`__exit__`的三个参数是:

`exc_type`

异常类(如`ZeroDivisionError`)。

`exc_value`

异常实例。有时，传递给异常构造函数的参数——比如错误消息——可以在`exc_value.args`中找到。

`traceback`

一个`traceback`对象。 [2]

要详细了解上下文管理器如何工作，请参见示例 18-4 ，其中 `LookingGlass` 在`with`块之外使用，因此我们可以手动调用其`__enter__`和`__exit__`方法。

##### 示例 18-4：不带`with`挡块练习`LookingGlass`

```
>>>frommirrorimportLookingGlass>>>manager=LookingGlass()①>>>manager# doctest: +ELLIPSIS<mirror.LookingGlassobjectat0x...>>>>monster=manager.__enter__()②>>>monster=='JABBERWOCKY'③eurT>>>monster'YKCOWREBBAJ'>>>manager# doctest: +ELLIPSIS>...tatcejbossalGgnikooL.rorrim<>>>manager.__exit__(None,None,None)④>>>monster'JABBERWOCKY'
```

① 实例化并检查`manager`实例。

② 调用管理器的`__enter__`方法，并将结果存储在`monster`中。

③ `monster`是字符串`'JABBERWOCKY'`。因为通过`stdout`的所有输出都经过我们在`__enter__`中修补的`write`方法，所以`True`标识符出现颠倒。

④ 调用`manager.__exit__`恢复之前的`stdout.write`。

# Python 3.10 中带括号的上下文管理器

Python 3.10 采用了[一个新的、更强大的解析器](https://fpy.li/pep617)，允许比旧的 [LL(1)解析器](https://fpy.li/18-8)更好的新语法。一个语法增强是允许带括号的上下文管理器，如下所示:

```
with (
    CtxManager1() as example1,
    CtxManager2() as example2,
    CtxManager3() as example3,
):
    ...
```

在 3.10 之前，我们必须把它写成嵌套的`with`块。

标准库包括`contextlib`包，其中有方便的函数、类和装饰器，用于构建、组合和使用上下文管理器。

## 上下文库实用程序

在推出你自己的上下文管理器类之前，先看看 Python 文档中的[`contextlib`——“用于`with`的实用工具——语句上下文”](https://fpy.li/18-9)。也许你要构建的东西已经存在，或者有一个类或一些可调用的东西会让你的工作更容易。

除了在示例 18-3 之后提到的`redirect_stdout`上下文管理器之外，Python 3.5 中还添加了`redirect_stderr`——它的功能与前者相同，但输出指向`stderr`。

`contextlib`包装还包括:

`closing`

从提供`close()`方法但不实现`__enter__/__exit__`接口的对象中构建上下文管理器的函数。

`suppress`

上下文管理器暂时忽略作为参数给出的异常。

`nullcontext`

一个什么都不做的上下文管理器，用来简化可能没有实现合适的上下文管理器的对象的条件逻辑。当`with`块之前的条件代码可能会也可能不会为`with`语句提供上下文管理器时，它充当替身——在 Python 3.7 中添加。

`contextlib`模块提供了比刚才提到的装饰器更广泛适用的类和装饰器:

`@contextmanager`

一个装饰器，让你从一个简单的生成器函数中构建一个上下文管理器，而不是创建一个类和实现接口。参见“使用@ context manager”。

`AbstractContextManager`

形式化上下文管理器接口的 ABC，通过子类化使创建上下文管理器类变得更加容易——在 Python 3.6 中添加。

`ContextDecorator`

一个基类，用于定义基于类的上下文管理器，也可以用作函数装饰器，在托管上下文中运行整个函数。

`ExitStack`

允许您输入可变数量的上下文管理器的上下文管理器。当`with`块结束时，`ExitStack`以后进先出的顺序调用堆栈上下文管理器的`__exit__`方法。当您事先不知道需要在您的`with`块中输入多少个上下文管理器时，使用这个类；例如，当同时打开任意文件列表中的所有文件时。

在 Python 3.7 中，`contextlib`增加了`AbstractAsyncContextManager`、`@asynccontextmanager`、`AsyncExitStack`。它们类似于名称中没有`async`部分的等效实用程序，但设计用于新的`async with`语句，包含在第 21 章的中。

这些实用程序中使用最广泛的是`@contextmanager`装饰器，因此它值得更多的关注。这个装饰器也很有趣，因为它展示了与迭代无关的`yield`语句的用法。

## 使用@contextmanager

`@contextmanager` decorator 是一个优雅而实用的工具，它集合了三个独特的 Python 特性:函数装饰器、生成器和`with`语句。

使用`@contextmanager`减少了创建上下文管理器的样板文件:不用用`__enter__/__exit__`方法编写整个类，只需用单个`yield`实现一个生成器，该生成器将生成您希望`__enter__`方法返回的任何内容。

在一个用`@contextmanager`修饰的生成器中，`yield`将函数体拆分成两部分:解释器调用`__enter__`时，`yield`之前的一切都将在`with`块的开头执行；当`__exit__`在程序块末尾被调用时，`yield`之后的代码将会运行。

示例 18-5 将示例 18-3 中的`LookingGlass`类替换为生成器函数。

##### 示例 18-5： mirror_gen.py:用生成器实现的上下文管理器

```
importcontextlibimportsys@contextlib.contextmanager①deflooking_glass():original_write=sys.stdout.write②defreverse_write(text):③original_write(text[::-1])sys.stdout.write=reverse_write④yield'JABBERWOCKY'⑤sys.stdout.write=original_write⑥
```

① 使用`contextmanager`装饰器。

② 保留原来的`sys.stdout.write`方法。

③ `reverse_write`可以稍后调用`original_write`因为在其闭包中可用。

④ 用`reverse_write`替换`sys.stdout.write`。

⑤ 生成将绑定到`with`语句的`as`子句中的目标变量的值。当`with`的主体执行时，生成器暂停。

⑥ 当控制退出`with`块时，执行在`yield`之后继续；这里恢复了原来的`sys.stdout.write`。

示例 18-6 显示了`looking_glass`功能的运行情况。

##### 示例 18-6：测试驱动`looking_glass`上下文管理器功能

```
>>>frommirror_genimportlooking_glass>>>withlooking_glass()aswhat:①...print('Alice, Kitty and Snowdrop')...print(what)...pordwonSdnayttiK,ecilAYKCOWREBBAJ>>>what'JABBERWOCKY'>>>print('back to normal')backtonormal
```

① 与示例 18-2 唯一不同的是上下文管理器的名称:`looking_glass`而不是`LookingGlass`。

`contextlib.contextmanager`装饰器将函数包装在实现`__enter__`和`__exit__`方法的类中。 [3]

该类的`__enter__`方法:

1.  调用生成器函数来获得一个生成器对象——让我们称之为`gen`。

2.  调用`next(gen)`将其驱动到`yield`关键字。

3.  返回由`next(gen)`产生的值，允许用户将其绑定到`with/as`表单中的变量。

当`with`程序块终止时，`__exit__`方法:

1.  检查异常是否作为`exc_type`被传递；如果是，则调用`gen.throw(exception)`，导致在生成器函数体内的`yield`行中引发异常。

2.  否则，调用`next(gen)`，在`yield`之后继续执行生成器函数体。

示例 18-5 有一个缺陷:如果在`with`块的体中引发了一个异常，Python 解释器会捕捉它，并在`looking_glass`内部的`yield`表达式中再次引发。但是这里没有错误处理，所以`looking_glass`生成器将终止，而不会恢复原来的`sys.stdout.write`方法，使系统处于无效状态。

示例 18-7 增加了对`ZeroDivisionError`异常的特殊处理，使其功能等同于基于类的示例 18-3 。

##### 示例 18-7： mirror_gen_exc.py:基于生成器的上下文管理器实现异常处理——与示例 18-3 相同的外部行为

```
importcontextlibimportsys@contextlib.contextmanagerdeflooking_glass():original_write=sys.stdout.writedefreverse_write(text):original_write(text[::-1])sys.stdout.write=reverse_writemsg=''①try:yield'JABBERWOCKY'exceptZeroDivisionError:②msg='Please DO NOT divide by zero!'finally:sys.stdout.write=original_write③ifmsg:print(msg)④
```

① 为可能的错误消息创建一个变量；这是相对于示例 18-5 的第一个变化。

② 通过设置错误信息来处理`ZeroDivisionError`。

③ 取消`sys.stdout.write`的猴子补丁。

④ 显示错误消息(如果已设置)。

回想一下`__exit__`方法告诉解释器它已经通过返回 truthy 值处理了异常；在这种情况下，解释器抑制异常。另一方面，如果`__exit__`没有显式返回值，则解释器得到通常的`None`，并传播异常。对于`@contextmanager`，默认行为是颠倒的:装饰器提供的`__exit__`方法假设发送到生成器中的任何异常都被处理并且应该被抑制。

###### 小费

在`yield`周围有一个`try/finally`(或`with`区块)是使用`@contextmanager`不可避免的代价，因为你永远不知道你的上下文管理器的用户在`with`区块内将做什么。^(4T28)

`@contextmanager`的一个鲜为人知的特点是，用它装饰的发电机本身也可以用作装饰。 [5] 发生这种情况是因为`@contextmanager`是用`contextlib.ContextDecorator`类实现的。

示例 18-8 显示了用作装饰器的来自示例 18-5 的`looking_glass`上下文管理器。

##### 示例 18-8：上下文管理器`looking_glass`也是一个装饰器

```
>>>@looking_glass()...defverse():...print('The time has come')...>>>verse()①emocsahemitehT>>>print('back to normal')②backtonormal
```

① `looking_glass`在`verse`的主体运行前后工作。

② 这证实了原来的`sys.write`已经恢复。

将示例 18-8 与示例 18-6 进行对比，其中`looking_glass`用作上下文管理器。

标准库之外的`@contextmanager`的一个有趣的现实例子是 Martijn Pieters 的使用上下文管理器](https://fpy.li/18-11)的就地文件重写。[示例 18-9 说明如何使用。

##### 示例 18-9：用于就地重写文件的上下文管理器

```
import csv

with inplace(csvfilename, 'r', newline='') as (infh, outfh):
    reader = csv.reader(infh)
    writer = csv.writer(outfh)

    for row in reader:
        row += ['new', 'columns']
        writer.writerow(row)
```

`inplace`函数是一个上下文管理器，它为您提供同一个文件的两个句柄(在本例中为`infh`和`outfh`)，允许您的代码同时对其进行读取和写入。它比标准库的 [`fileinput.input`函数](https://fpy.li/18-12)更容易使用(顺便说一下，它还提供了上下文管理器)。

如果你想研究 Martijn 的`inplace`源代码(在[文章](https://fpy.li/18-11)中列出)，找到 `yield`关键词:在它处理设置上下文之前的一切，这需要创建一个备份文件，然后打开并产生对可读和可写文件句柄的引用，这些句柄将由`__enter__`调用返回。在`yield`关闭文件之后的`__exit__`处理过程处理并从备份中恢复出出错的文件。

我们对`with`语句和上下文管理器的概述到此结束。让我们在完整示例的背景下转向`match/case`。

# lis.py 中的模式匹配:案例研究

在 “解释器中的模式匹配序列”中，我们看到了从 Peter Norvig 的 *lis.py* 解释器的`evaluate`函数中提取的序列模式的例子，移植到 Python 3.10。在这一节中，我想给出一个关于 *lis.py* 如何工作的更广泛的概述，并探索`evaluate`的所有`case`子句，不仅解释模式，还解释解释器在每个`case`中做什么。

除了展示更多的模式匹配，我写这一节有三个原因:

1.  Norvig 的 *lis.py* 是惯用 Python 代码的一个很好的例子。

2.  方案的简单性是语言设计的大师课。

3.  学习解释器如何工作让我对 Python 和编程语言有了更深的理解——解释的或编译的。

在查看 Python 代码之前，让我们先了解一下 Scheme，这样您就可以理解这个案例研究——以防您以前没有见过 Scheme 或 Lisp。

## 方案语法

在 Scheme 中，表达式和语句之间没有区别，就像我们在 Python 中一样。此外，没有中缀运算符。所有表达式都使用类似于`(+ x 13)`的前缀符号，而不是`x + 13`。相同的前缀符号用于函数调用——例如`(gcd x 13)`——和特殊形式——例如`(define x 13)`,我们在 Python 中将它们写成赋值语句`x = 13`。Scheme 和大多数 Lisp 方言使用的符号被称为 *S 表达式*。 [6]

示例 18-10 显示了方案中的一个简单例子。

##### 示例 18-10：方案中的最大公约数

```
(define (mod m n)
    (- m (* n (quotient m n))))

(define (gcd m n)
    (if (= n 0)
        m
        (gcd n (mod m n))))

(display (gcd 18 45))
```

示例 18-10 展示了三个方案表达式:两个函数定义——`mod`和`gcd`——以及一个对`display`的调用，它将输出`(gcd 18 45)`的结果 9。示例 18-11 是 Python 中相同的代码(比递归 [*欧几里德算法*](https://fpy.li/18-14) 的英文解释短)。

##### 示例 18-11：同示例 18-10 ，用 Python 编写

```
def mod(m, n):
    return m - (m // n * n)

def gcd(m, n):
    if n == 0:
        return m
    else:
        return gcd(n, mod(m, n))

print(gcd(18, 45))
```

在惯用的 Python 中，我会使用`%`操作符而不是重新发明`mod`，并且使用`while`循环而不是递归会更有效。但是我想展示两个函数定义，并使例子尽可能相似，以帮助您阅读 Scheme 代码。

方案没有类似`while`或`for`的迭代控制流程命令。迭代是通过递归完成的。请注意 Scheme 和 Python 示例中没有赋值。广泛使用递归和极少使用赋值是函数式编程的标志。 [7]

现在我们来回顾一下 *lis.py* 的 Python 3.10 版本的代码。带测试的完整源代码在 GitHub 资源库[*fluent python/example-code-2e*](https://fpy.li/code)的[*18-with-match/lispy/py 3.10/*](https://fpy.li/18-15)目录下。

## 进口和类型

示例 18-12 显示lis . py 的第一行。使用`TypeAlias`和`|`类型的 union 操作符需要 Python 3.10。

##### 示例 18-12：lis.py:文件的顶部

```
import math
import operator as op
from collections import ChainMap
from itertools import chain
from typing import Any, TypeAlias, NoReturn

Symbol: TypeAlias = str
Atom: TypeAlias = float | int | Symbol
Expression: TypeAlias = Atom | list
```

定义的类型有:

`Symbol`

只是`str`的别名。在 *lis.py* 中，`Symbol`用于标识符；没有带切片、拆分等操作的字符串数据类型。 [8]

`Atom`

一个简单的语法元素，比如一个数字或者一个`Symbol`——相对于由不同部分组成的复合结构，比如一个列表。

`Expression`

Scheme 程序的构建块是由原子和列表组成的表达式，可能是嵌套的。

## 解析器

Norvig 的解析器是 36 行代码，展示了 Python 在处理 S 表达式的简单递归语法方面的强大功能——没有字符串数据、注释、宏和其他标准模式的特性，这些特性使解析变得更加复杂(示例 18-13 )。

##### 示例 18-13： lis.py:主要的解析函数

```
def parse(program: str) -> Expression:
    "Read a Scheme expression from a string."
    return read_from_tokens(tokenize(program))

def tokenize(s: str) -> list[str]:
    "Convert a string into a list of tokens."
    return s.replace('(', ' ( ').replace(')', ' ) ').split()

def read_from_tokens(tokens: list[str]) -> Expression:
    "Read an expression from a sequence of tokens."
    # more parsing code omitted in book listing
```

该组的主要函数是`parse`，它将一个 S 表达式作为`str`并返回一个`Expression`对象，如示例 18-12 中所定义:一个`Atom`或一个`list`可能包含更多的原子和嵌套列表。

Norvig 在`tokenize`中使用了一个聪明的技巧:他在输入中的每个括号前后添加空格，然后将其拆分，从而产生一个语法标记列表，其中`'('`和`')'`作为单独的标记。这个快捷方式之所以有效，是因为 *lis.py* 的小方案中没有字符串类型，所以每一个`'('`或者`')'`都是一个表达式分隔符。递归解析代码在`read_from_tokens`中，这是一个 14 行的函数，可以在[*fluent python/example-code-2e*](https://fpy.li/18-17)资源库中读取。我将跳过它，因为我想专注于解释器的其他部分。

以下是从[*lispy/py 3.10/examples _ test . py*](https://fpy.li/18-18)中摘录的一些 doctests:

```
>>> from lis import parse
>>> parse('1.5')
1.5
>>> parse('ni!')
'ni!'
>>> parse('(gcd 18 45)')
['gcd', 18, 45]
>>> parse('''
... (define double
...     (lambda (n)
...         (* n 2)))
... ''')
['define', 'double', ['lambda', ['n'], ['*', 'n', 2]]]
```

这个模式子集的解析规则很简单:

1.  看起来像数字的令牌被解析为`float`或`int`。

2.  任何不是`'('`或`')'`的东西都被解析为`Symbol`——一个用作标识符的`str`。这包括像`+`、`set!`和`make-counter`这样的源文本，它们在 Scheme 中是有效的标识符，但在 Python 中不是。

3.  `'('`和`')'`中的表达式被递归解析为包含原子的列表或可能包含原子和更多嵌套列表的嵌套列表。

使用 Python 解释器的术语，`parse`的输出是一个 AST(抽象语法树):模式程序的一个方便的表示，作为形成树状结构的嵌套列表，其中最外面的列表是主干，内部的列表是分支，原子是树叶(图 18-1 )。

![Scheme code, a tree diagram, and Python objects](Images/flpy_1801.png)

###### 图 18-1。表示为源代码(具体语法)、树和 Python 对象序列(抽象语法)的 Scheme `lambda`表达式。

## 环境

`Environment`类扩展了`collections.ChainMap`，添加了一个`change`方法来更新一个链式字典中的值，这个值由`ChainMap`实例保存在一个映射列表中:属性`self.maps`。需要使用`change`方法来支持 Scheme `(set! …)`表单，这将在后面描述；参见示例 18-14 。

##### 示例 18-14：*lis . py*`Environment`类

```
class Environment(ChainMap[Symbol, Any]):
    "A ChainMap that allows changing an item in-place."

    def change(self, key: Symbol, value: Any) -> None:
        "Find where key is defined and change the value there."
        for map in self.maps:
            if key in map:
                map[key] = value  # type: ignore[index]
                return
        raise KeyError(key)
```

注意，`change`方法只更新现有的键。 [9] 试图更改未找到的键会引发`KeyError`。

这个文档测试展示了`Environment`的工作原理:

```
>>> fromlisimportEnvironment>>> inner_env={'a':2}>>> outer_env={'a':0,'b':1}>>> env=Environment(inner_env,outer_env)>>> env['a']①2 >>> env['a']=111②>>> env['c']=222>>> envEnvironment({'a': 111, 'c': 222}, {'a': 0, 'b': 1}) >>> env.change('b',333)③>>> envEnvironment({'a': 111, 'c': 222}, {'a': 0, 'b': 333})
```

① 读取值时，`Environment`的作用与`ChainMap`相同:从左到右在嵌套映射中搜索键。这就是为什么`outer_env`中的`a`值被`inner_env`中的值遮蔽的原因。

② 用`[]`赋值会覆盖或插入新的条目，但总是在第一个映射中，在这个例子中是`inner_env`。

③ `env.change('b', 333)`寻找`'b'`键，并在`outer_env`中为其指定一个新值。

接下来是`standard_env()`函数，它构建并返回一个加载了预定义函数的`Environment`，类似于 Python 的始终可用的`__builtins__`模块(示例 18-15 )。

##### 示例 18-15： lis.py: `standard_env()`构建并返回全局环境

```
def standard_env() -> Environment:
    "An environment with some Scheme standard procedures."
    env = Environment()
    env.update(vars(math))   # sin, cos, sqrt, pi, ...
    env.update({
            '+': op.add,
            '-': op.sub,
            '*': op.mul,
            '/': op.truediv,
            # omitted here: more operator definitions
            'abs': abs,
            'append': lambda *args: list(chain(*args)),
            'apply': lambda proc, args: proc(*args),
            'begin': lambda *x: x[-1],
            'car': lambda x: x[0],
            'cdr': lambda x: x[1:],
            # omitted here: more function definitions
            'number?': lambda x: isinstance(x, (int, float)),
            'procedure?': callable,
            'round': round,
            'symbol?': lambda x: isinstance(x, Symbol),
    })
    return env
```

总而言之，`env`映射加载了:

*   Python 的`math`模块中的所有函数

*   从 Python 的`op`模块中选择的运算符

*   用 Python 的`lambda`构建的简单而强大的函数

*   Python 内置了重命名，比如将`callable`作为`procedure?`，或者直接映射，比如`round`

## REPL

Norvig 的 REPL (read-eval-print-loop)易于理解，但不便于用户使用(参见示例 18-16 )。如果没有给 *lis.py* 提供任何命令行参数，则`repl()`函数由`main()`调用，该函数在模块末尾定义。在`lis.py>`提示下，我们必须输入正确完整的表达式；如果我们忘了加上一个右括号， *lis.py* 就会崩溃。^(T4110)

##### 示例 18-16：REPL 函数

```
def repl(prompt: str = 'lis.py> ') -> NoReturn:
    "A prompt-read-eval-print loop."
    global_env = Environment({}, standard_env())
    while True:
        ast = parse(input(prompt))
        val = evaluate(ast, global_env)
        if val is not None:
            print(lispstr(val))

def lispstr(exp: object) -> str:
    "Convert a Python object back into a Lisp-readable string."
    if isinstance(exp, list):
        return '(' + ' '.join(map(lispstr, exp)) + ')'
    else:
        return str(exp)
```

以下是对这两个函数的简要说明:

`repl(prompt: str = 'lis.py> ') -> NoReturn`

调用`standard_env()`为全局环境提供内置函数，然后进入无限循环，读取并解析每一行输入，在全局环境中进行计算并显示结果—除非是`None`。`global_env`可由`evaluate`修改。例如，当用户定义一个新的全局变量或命名函数时，它被存储在环境的第一个映射中——空的`dict`在`Environment`构造函数调用中的第一行`repl`。

`lispstr(exp: object) -> str`

`parse`的反函数:给定一个表示表达式的 Python 对象，`parse`返回其 Scheme 源代码。例如，给定`['+', 2, 3]`，结果为`'(+ 2 3)'`。

## 评估者

现在我们可以欣赏诺威格的表情评估工具的妙处了——用`match/case`做得更漂亮了一点。示例 18-17 中的`evaluate`功能采用`parse`和`Environment`制造的`Expression`。

`evaluate`的主体是单个的`match`语句，以表达式`exp`为主体。`case`模式以惊人的清晰表达 Scheme 的语法和语义。

##### 示例 18-17： `evaluate`获取表达式并计算其值

```
KEYWORDS = ['quote', 'if', 'lambda', 'define', 'set!']

def evaluate(exp: Expression, env: Environment) -> Any:
    "Evaluate an expression in an environment."
    match exp:
        case int(x) | float(x):
            return x
        case Symbol(var):
            return env[var]
        case ['quote', x]:
            return x
        case ['if', test, consequence, alternative]:
            if evaluate(test, env):
                return evaluate(consequence, env)
            else:
                return evaluate(alternative, env)
        case ['lambda', [*parms], *body] if body:
            return Procedure(parms, body, env)
        case ['define', Symbol(name), value_exp]:
            env[name] = evaluate(value_exp, env)
        case ['define', [Symbol(name), *parms], *body] if body:
            env[name] = Procedure(parms, body, env)
        case ['set!', Symbol(name), value_exp]:
            env.change(name, evaluate(value_exp, env))
        case [func_exp, *args] if func_exp not in KEYWORDS:
            proc = evaluate(func_exp, env)
            values = [evaluate(arg, env) for arg in args]
            return proc(*values)
        case _:
            raise SyntaxError(lispstr(exp))
```

让我们研究一下每个`case`分句及其作用。在某些情况下，我添加了显示 S 表达式的注释，当解析为 Python 列表时，这些注释将匹配该模式。取自 [*示例的 doctests _ test . py*](https://fpy.li/18-21)演示了每个`case`。

### 评估数字

```
    case int(x) | float(x):
        return x
```

Subject:

例如`int`或`float`。

Action:

按原样返回值。

Example:

```
>>> from lis import parse, evaluate, standard_env
>>> evaluate(parse('1.5'), {})
1.5
```

### 评估符号

```
    case Symbol(var):
        return env[var]
```

Subject:

`Symbol`的实例，即用作标识符的`str`。

Action:

在`env`中查找`var`并返回其值。

Examples:

```
>>> evaluate(parse('+'), standard_env())
<built-in function add>
>>> evaluate(parse('ni!'), standard_env())
Traceback (most recent call last):
    ...
KeyError: 'ni!'
```

### (引用…)

`quote`特殊形式将原子和列表视为数据，而不是要计算的表达式。

```
    # (quote (99 bottles of beer))
    case ['quote', x]:
        return x
```

Subject:

列表以符号`'quote'`开始，后跟一个表达式`x`。

Action:

不评估就返回`x`。

Examples:

```
>>> evaluate(parse('(quote no-such-name)'), standard_env())
'no-such-name'
>>> evaluate(parse('(quote (99 bottles of beer))'), standard_env())
[99, 'bottles', 'of', 'beer']
>>> evaluate(parse('(quote (/ 10 0))'), standard_env())
['/', 10, 0]
```

如果没有`quote`，测试中的每个表达式都会产生一个错误:

*   `no-such-name`会在环境中抬头，抬起`KeyError`

*   `(99 bottles of beer)`无法计算，因为数字 99 不是一个 `Symbol` 命名的特殊形式、运算符或函数

*   `(/ 10 0)`会提高`ZeroDivisionError`

### (如果……)

```
    # (if (< x 0) 0 x)
    case ['if', test, consequence, alternative]:
        if evaluate(test, env):
            return evaluate(consequence, env)
        else:
            return evaluate(alternative, env)
```

Subject:

列表以`'if'`开头，后面跟着三个表达式:`test`、`consequence`和`alternative`。

Action:

评估`test`:

*   如果为真，评估`consequence`并返回其值。

*   否则，评估`alternative`并返回其值。

Examples:

```
>>> evaluate(parse('(if (= 3 3) 1 0))'), standard_env())
1
>>> evaluate(parse('(if (= 3 4) 1 0))'), standard_env())
0
```

`consequence`和`alternative`分支必须是单一表达式。如果一个分支中需要多个表达式，你可以用`(begin exp1 exp2…)`将它们组合起来，作为 *lis.py* 中的一个函数提供——参见示例 18-15 。

### (λ…)

Scheme 的`lambda`表单定义了匿名函数。它不受 Python 的`lambda`的限制:任何可以用 Scheme 编写的函数都可以用`(lambda …)`语法编写。

```
    # (lambda (a b) (/ (+ a b) 2))
    case ['lambda' [*parms], *body] if body:
        return Procedure(parms, body, env)
```

Subject:

列表从`'lambda'`开始，然后是:

*   零个或多个参数名的列表。

*   在`body`中收集的一个或多个表达式(守卫确保`body`不为空)。

Action:

创建并返回一个新的`Procedure`实例，带有参数名、作为主体的表达式列表和当前环境。

Example:

```
>>> expr = '(lambda (a b) (* (/ a b) 100))'
>>> f = evaluate(parse(expr), standard_env())
>>> f  # doctest: +ELLIPSIS
<lis.Procedure object at 0x...>
>>> f(15, 20)
75.0
```

`Procedure`类实现了闭包的概念:一个包含参数名、函数体和对定义函数的环境的引用的可调用对象。我们一会儿将研究`Procedure`的代码。

### (定义…)

`define`关键字有两种不同的语法形式。最简单的是:

```
    # (define half (/ 1 2))
    case ['define', Symbol(name), value_exp]:
        env[name] = evaluate(value_exp, env)
```

Subject:

列表以`'define'`开始，后面跟着一个`Symbol`和一个表达式。

Action:

使用`name`作为关键字，对表达式求值并将其值放入`env`。

Example:

```
>>> global_env = standard_env()
>>> evaluate(parse('(define answer (* 7 6))'), global_env)
>>> global_env['answer']
42
```

这个`case`的 doctest 创建了一个`global_env`，这样我们就可以验证`evaluate`是否将`answer`放入了那个`Environment`中。

我们可以使用这个简单的`define`形式来创建变量或者将名字绑定到匿名函数，使用`(lambda …)`作为`value_exp`。

标准模式提供了定义命名函数的快捷方式。那是第二种`define`形式:

```
    # (define (average a b) (/ (+ a b) 2))
    case ['define', [Symbol(name), *parms], *body] if body:
        env[name] = Procedure(parms, body, env)
```

Subject:

列表从`'define'`开始，然后是:

*   一个以`Symbol(name)`开头的列表，后面是零个或多个收集到一个名为`parms`的列表中的条目。

*   在`body`中收集的一个或多个表达式(守卫确保`body`不为空)。

Action:

*   用参数名、作为主体的表达式列表和当前环境创建一个新的`Procedure`实例。

*   使用`name`作为键将`Procedure`放入`env`中。

示例 18-18 中的 doctest 定义了一个名为`%`的函数，该函数计算一个百分比并将其添加到`global_env`中。

##### 示例 18-18：定义一个名为`%`的计算百分比的函数

```
>>> global_env = standard_env()
>>> percent = '(define (% a b) (* (/ a b) 100))'
>>> evaluate(parse(percent), global_env)
>>> global_env['%']  # doctest: +ELLIPSIS
<lis.Procedure object at 0x...>
>>> global_env['%'](170, 200)
85.0
```

在调用了`evaluate`之后，我们检查`%`是否绑定到了一个`Procedure`上，它接受两个数字参数并返回一个百分比。

第二个`define` `case`的模式不强制`parms`中的项目都是`Symbol`实例。在构建`Procedure`之前，我必须检查这一点，但我没有——为了让代码像 Norvig 的代码一样容易理解。

### (设定！…)

`set!`表单改变先前定义的变量的值。 [11]

```
    # (set! n (+ n 1))
    case ['set!', Symbol(name), value_exp]:
        env.change(name, evaluate(value_exp, env))
```

Subject:

列表以`'set!'`开始，后面跟着一个`Symbol`和一个表达式。

Action:

用表达式评估的结果更新`env`中`name`的值。

`Environment.change`方法从局部到全局遍历链接的环境，并用新值更新第一次出现的`name`。如果我们没有实现`'set!'`关键字，我们可以在这个解释器的任何地方使用 Python 的`ChainMap`作为`Environment`类型。

现在我们来看一个函数调用。

### 函数调用

```
    # (gcd (* 2 105) 84)
    case [func_exp, *args] if func_exp not in KEYWORDS:
        proc = evaluate(func_exp, env)
        values = [evaluate(arg, env) for arg in args]
        return proc(*values)
```

Subject:

包含一个或多个项目的列表。

防护罩确保`func_exp`不是示例 18-17 中`['quote', 'if', 'define', 'lambda', 'set!']`的前一项。

该模式匹配任何带有一个或多个表达式的列表，将第一个表达式绑定到`func_exp`并将其余的绑定到`args`作为一个列表，该列表可能是空的。

Action:

*   评估`func_exp`以获得一个函数`proc`。

*   评估`args`中的每一项，建立一个参数值列表。

*   用值作为单独的参数调用`proc`，返回结果。

Example:

```
>>> evaluate(parse('(% (* 12 14) (- 500 100))'), global_env)
42.0
```

这个 doctest 上接示例 18-18 :它假设`global_env`有一个名为`%`的函数。给`%`的参数是算术表达式，以强调参数在函数被调用之前被评估。

这个`case`中的守卫是需要的，因为`[func_exp, *args]`用一个或多个项目匹配任何序列主题。但是，如果`func_exp`是一个关键字，而主语与之前的任何一个格都不匹配，那么就真的是语法错误了。

### 捕捉语法错误

如果主题`exp`与前面的任何一个案例都不匹配，那么捕获全部`case`将引发一个`SyntaxError`:

```
    case _:
        raise SyntaxError(lispstr(exp))
```

下面是一个被报告为`SyntaxError`的畸形`(lambda …)`的例子:

```
>>> evaluate(parse('(lambda is not like this)'), standard_env())
Traceback (most recent call last):
    ...
SyntaxError: (lambda is not like this)
```

如果函数调用的`case`没有拒绝关键字的保护，那么`(lambda is not like this)`表达式将作为函数调用处理，这将引发`KeyError`,因为`'lambda'`不是环境的一部分——就像`lambda`不是 Python 内置函数一样。

## 过程:实现闭包的类

`Procedure`类可以很好地命名为`Closure`，因为这就是它所代表的:一个函数定义和一个环境。函数定义包括组成函数体的参数和表达式的名称。当调用函数来提供*自由变量*的值时，使用环境:出现在函数体中但不是参数、局部变量或全局变量的变量。我们在“闭包”中看到了*闭包*和*自由变量*的概念。

我们学习了如何在 Python 中使用闭包，但是现在我们可以更深入地了解闭包是如何在 *lis.py* 中实现的:

```
classProcedure:"A user-defined Scheme procedure."def__init__(①self,parms:list[Symbol],body:list[Expression],env:Environment):self.parms=parms②self.body=bodyself.env=envdef__call__(self,*args:Expression)->Any:③local_env=dict(zip(self.parms,args))④env=Environment(local_env,self.env)⑤forexpinself.body:⑥result=evaluate(exp,env)returnresult⑦
```

① 当函数由`lambda`或`define`表单定义时调用。

② 保存参数名称、主体表达式和环境供以后使用。

③ 由`case [func_exp, *args]`条款最后一行的`proc(*values)`调用。

④ 构建`local_env`映射`self.parms`为局部变量名，给定`args`为值。

⑤ 构建一个新的组合`env`，先放`local_env`，然后放`self.env`——定义函数时保存的环境。

⑥ 迭代`self.body`中的每个表达式，在组合的`env`中对其求值。

⑦ 返回最后一个表达式的计算结果。

在 [*lis.py*](https://fpy.li/18-24) 中的`evaluate`之后有几个简单的函数:`run`读取一个完整的 Scheme 程序并执行它，`main`根据命令行调用`run`或`repl`——类似于 Python 的做法。我不会描述这些功能，因为它们没有什么新意。我的目标是与您分享 Norvig 的小解释器的美妙之处，让您更深入地了解闭包是如何工作的，并展示`match/case`是 Python 的一个很好的补充。

为了总结这个关于模式匹配的扩展部分，让我们形式化 OR 模式的概念。

## 使用或模式

由`|`分隔的 系列模式是一个 *或-模式*](https://fpy.li/18-25) :如果任何子模式成功，则成功。[“评估数字”中的模式是或模式:

```
    case int(x) | float(x):
        return x
```

OR 模式中的所有子模式必须使用相同的变量。这个限制是必要的，以确保变量可用于保护表达式和`case`主体，而不管匹配的子模式是什么。

###### 警告

在`case`子句的上下文中，`|`操作符有特殊的含义。它不会触发`__or__`特殊方法，该方法在其他上下文中处理类似于`a | b`的表达式，在其他上下文中，它被重载以执行诸如集合并集或整数按位或之类的操作，这取决于操作数。

OR 模式不限于出现在模式的顶层。您也可以在子模式中使用`|`。例如，如果我们希望 *lis.py* 接受希腊字母λ (lambda) [12] 以及关键字`lambda`，我们可以像这样重写模式:

```
    # (λ (a b) (/ (+ a b) 2) )
    case ['lambda' | 'λ', [*parms], *body] if body:
        return Procedure(parms, body, env)
```

现在我们可以转到本章的第三个也是最后一个主题:Python 中可能出现`else`子句的不寻常的地方。

# 做这个，然后做那个:else 阻塞在 if 之外

这个 不是什么秘密，但它是一个未被重视的语言特性:`else`子句不仅可以用在`if`语句中，还可以用在`for`、`while`和`try`语句中。

`for/else`、`while/else`、`try/else`的语义密切相关，但与`if/else`有很大不同。最初，`else`这个词其实阻碍了我对这些功能的理解，但最终我习惯了。

规则如下:

`for`

只有当`for`循环运行完成时`else`程序块才会运行(即，如果`for`被`break`中止，则不会运行)。

`while`

只有当`while`循环因条件变为*错误*而退出时，`else`程序块才会运行(即，如果`while`因`break`而中止，则不会运行)。

`try`

只有在`try`程序块中没有出现异常时，`else`程序块才会运行。[正式文件](https://fpy.li/18-27)也说明:“`else`条款中的例外不按前面的`except`条款处理。”

在所有情况下，如果异常或`return`、`break`或`continue`语句导致控制跳出复合语句的主块，则`else`子句也会被跳过。

###### 注意

我认为除了`if`之外，在所有情况下`else`都是一个很差的关键字选择。它暗示了一个排除的选择，比如，“运行这个循环，否则执行那个”，但是循环中的语义是相反的:“运行这个循环，然后执行那个。”这表明`then`是一个更好的关键字——这在`try`上下文中也是有意义的:“尝试这个，然后做那个。”然而，添加一个新的关键字对语言来说是一个突破性的改变——不是一个容易做出的决定。

将`else`与这些语句一起使用通常会使代码更容易阅读，并省去设置控制标志或编写额外`if`语句的麻烦。

在循环中使用`else`通常遵循以下代码片段的模式:

```
for item in my_list:
    if item.flavor == 'banana':
        break
else:
    raise ValueError('No banana flavor found!')
```

在`try/except`块的情况下，`else`一开始可能显得多余。毕竟，只有当`dangerous_call()`没有引发异常时，下面代码片段中的`after_call()`才会运行，对吗？

```
try:
    dangerous_call()
    after_call()
except OSError:
    log('OSError...')
```

然而，这样做会无缘无故地将`after_call()`放入`try`块中。为了清楚和正确起见，`try`块的主体应该只包含可能产生预期异常的语句。这更好:

```
try:
    dangerous_call()
except OSError:
    log('OSError...')
else:
    after_call()
```

现在很清楚，`try`块是在防范 `dangerous_call()` 中可能出现的错误，而不是在`after_call()`中。同样显而易见的是，只有在`try`块中没有出现异常的情况下，`after_call()`才会执行。

在 Python 中，`try/except`通常用于控制流，而不仅仅用于错误处理。在官方 Python 词汇表[中甚至有一个缩写/口号:](https://fpy.li/18-28)

> EAFP
> 
> 请求原谅比许可容易。这种常见的 Python 编码风格假设存在有效的键或属性，并在假设证明为假时捕捉异常。这种简洁快速的风格的特点是存在许多 try 和 except 语句。这种技术与许多其他语言中常见的 *LBYL* 风格形成对比，例如 C.

词汇表随后定义了 LBYL:

> LBYL
> 
> 三思而后行。这种编码风格在进行调用或查找之前显式测试前置条件。这种风格与 EAFP 的方法形成了鲜明的对比，其特点是出现了许多 if 语句。在多线程环境中，LBYL 方法可能会在“寻找”和“跳跃”之间引入竞争条件例如，如果另一个线程在测试之后、查找之前从映射中移除键，则代码 if key in mapping:return mapping[key]可能会失败。这个问题可以通过锁或使用 EAFP 方法来解决。
> 
> T32

鉴于 EAFP 风格，在`try/except`语句中很好地了解和使用`else`块更有意义。

###### 注意

在讨论`match`语句的时候，有人(包括我)认为它也应该有一个`else`子句。最后决定不需要它，因为`case _:`做同样的工作。 [13]

现在让我们总结一下这一章。

# 章节摘要

这一章从上下文管理器和`with`语句的含义开始，很快超越了自动关闭打开的文件的一般用法。我们实现了一个定制的上下文管理器:带有`__enter__/__exit__`方法的`LookingGlass`类，并看到了如何在`__exit__`方法中处理异常。Raymond Hettinger 在他的 PyCon US 2013 主题演讲中提出的一个关键观点是`with`不仅仅用于资源管理；它是一个工具，用于分解常见的安装和拆卸代码，或者需要在另一个过程之前和之后完成的任何一对操作。 [14]

我们回顾了`contextlib`标准库模块中的函数。其中一个是`@contextmanager` decorator，它使得使用一个简单的生成器和一个`yield`来实现上下文管理器成为可能——这是一个比用至少两个方法编码一个类更精简的解决方案。我们将`LookingGlass`重新实现为`looking_glass`生成器函数，并讨论了如何在使用`@contextmanager`时进行异常处理。

然后我们学习了 Peter Norvig 优雅的 *lis.py* ，这是一个用惯用 Python 编写的 Scheme 解释器，被重构为使用`evaluate`中的`match/case`——任何解释器的核心功能。理解`evaluate`如何工作需要回顾一点 Scheme，一个 S 表达式的解析器，一个简单的 REPL，以及通过`collection.ChainMap`的`Environment`子类的嵌套作用域的构造。最终， *lis.py* 成为了一个探索远不止模式匹配的载体。它展示了解释器的不同部分如何协同工作，阐明了 Python 本身的核心特性:为什么保留关键字是必要的，作用域规则是如何工作的，以及闭包是如何构建和使用的。

# 进一步阅读

[第 8 章，“复合语句”，](https://fpy.li/18-27) *中的《Python 语言参考》*说了关于`if`、`for`、`while`和`try`语句中的`else`子句的几乎所有内容。关于`try/except`的 Python 用法，不管有没有`else`，Raymond Hettinger 对问题[有一个很好的回答:“在 Python 中使用 try-except-else 是一个好习惯吗？”](https://fpy.li/18-31)在 StackOverflow。 [*蟒概括地说*](https://fpy.li/pynut3) ，第 3 版。，其中有一章是关于例外的，对 EAFP 风格进行了精彩的讨论，称赞计算机先驱格蕾丝·赫柏创造了“请求原谅比请求许可更容易”这句话

**Python 标准库*，第 4 章，“内置类型”，有一节专门讨论[“上下文管理器类型”](https://fpy.li/18-32)。`__enter__/__exit__`特殊方法也记录在*Python 语言参考文献*中的[“带语句上下文管理器”](https://fpy.li/18-33)。在 [PEP 343 中引入了上下文管理器，即“with”语句](https://fpy.li/pep343)。*

 *Raymond Hettinger 在他的 [PyCon US 2013 主题演讲](https://fpy.li/18-29)中强调了`with`声明是一个“成功的语言特色”。在同一个会议上，他还在他的演讲[“将代码转换成漂亮的、惯用的 Python”](https://fpy.li/18-35)中展示了上下文管理器的一些有趣的应用。

Jeff Preshing 的博客文章[“带有语句的 Python *示例”*](https://fpy.li/18-36)对于使用带有`pycairo`图形库的上下文管理器的示例很有趣。

`contextlib.ExitStack`类基于 Nikolaus Rath 的一个原始想法，他写了一篇简短的文章解释了为什么它有用:[“关于 Python exit stack 的美”](https://fpy.li/18-37)。在那篇文章中，Rath 提出,`ExitStack`与 Go 中的`defer`语句相似，但更灵活——我认为这是该语言中最好的想法之一。

Beazley 和 Jones 在他们的 *[Python Cookbook，](https://fpy.li/pycook3)* 第三版中为非常不同的目的设计了上下文管理器。“配方 8.3。“使对象支持上下文管理协议”实现了一个`LazyConnection`类，它的实例是上下文管理器，在`with`块中自动打开和关闭网络连接。“配方 9.22。以简单的方式定义上下文管理器”引入了一个用于计时代码的上下文管理器，另一个用于对一个`list`对象进行事务性修改:在`with`块中，制作了一个`list`实例的工作副本，所有的修改都应用于该工作副本。只有当`with`块无异常完成时，工作副本才替换原始列表。简单又巧妙。

Peter Norvig 在帖子(如何编写一个(Lisp)解释器(用 Python))](https://fpy.li/18-38)和[(一个((更好的)Lisp)解释器(用 Python))](https://fpy.li/18-39)中描述了他的小 Scheme 解释器。 *lis.py* 和 *lispy.py* 的代码是[*nor vig/py tudes*](https://fpy.li/18-40)库。我的存储库[*fluent Python/lispy*](https://fpy.li/18-41)包含了 *lis.py* 的 *mylis* forks，更新到 Python 3.10，有了更好的 REPL、命令行集成、示例、更多测试以及学习 Scheme 更多内容的参考资料。学习和实验的最佳方案方言和环境是[拍](https://fpy.li/18-42)。*  *^([1)PyCon US 2013 主题演讲:[《是什么让 Python 牛逼》](https://fpy.li/18-1)；关于`with`的部分从 23:00 开始，到 26:15 结束。

[2]`self`收到的三个参数正是你在一个`try/finally`语句的`finally`块中调用 [`sys.exc_info()`](https://fpy.li/18-7) 得到的。这是有意义的，考虑到`with`语句旨在取代`try/finally`的大部分使用，并且调用`sys.exc_info()`通常是必要的，以确定需要什么清理动作。

[3] 实际班名为`_GeneratorContextManager`。如果你想知道它到底是如何工作的，请阅读 Python 3.10 中的 *Lib/contextlib.py* 中的[源代码](https://fpy.li/18-10)。

这条建议直接引用了本书的科技评论家之一莱昂纳多·罗塞尔的评论。说得好，利奥！

至少我和其他技术评测人员是在 Caleb Hattingh 告诉我们之后才知道的。谢谢凯勒。

人们抱怨 Lisp 中有太多的括号，但是有思想的缩进和一个好的编辑器可以解决这个问题。主要的可读性问题是对函数调用使用相同的`(f …)`符号，以及像`(define …)`、`(if …)`和`(quote …)`这样的特殊形式，它们的行为完全不像函数调用。

^(7](ch18.xhtml#idm46582396220528-marker)) 为了使递归迭代实用高效，Scheme 和其他函数式语言实现了*适当的尾调用*。有关这方面的更多信息，请参见[“肥皂盒”。

[8] 但是 Norvig 的第二个解释器 [*lispy.py*](https://fpy.li/18-16) 支持字符串作为数据类型，以及高级特性，如语法宏、延续和适当的尾部调用。然而， *lispy.py* 几乎是 *lis.py* 的三倍长——也更难理解。

[9]`# type: ignore[index]`注释是因为*排版*问题 [#6042](https://fpy.li/18-19) 而出现的，这个问题在我阅读本章时尚未解决。`ChainMap`被注释为`MutableMapping`，但是`maps`属性中的类型提示说它是`Mapping`的列表，间接使得整个`ChainMap`在 Mypy 看来是不可变的。

[10] 当我研究 Norvig 的 *lis.py* 和 *lispy.py* 时，我开始了一个名为 [*mylis*](https://fpy.li/18-20) 的 fork，它增加了一些特性，包括一个 REPL，它接受部分 S 表达式并提示继续，类似于 Python 的 REPL 知道我们还没有完成并给出第二个提示(`...`)，直到我们输入一个完整的表达式或语句可以被求值。mylis 也优雅地处理了一些错误，但是仍然很容易崩溃。它远不如 Python 的 REPL 健壮。

[11] 赋值是很多编程教程中最先教授的特性之一，但`set!`只出现在最知名的方案书第 220 页， [*计算机程序的结构与解释*，第 2 版。，](https://fpy.li/18-22)作者艾贝尔森等人(麻省理工学院出版社)，又名 SICP 或“巫师之书”用函数式风格编码可以让我们远离命令式和面向对象编程中常见的状态变化。

[12]λ(U+03BB)的官方 Unicode 名称是希腊文小写字母 LAMDA。这不是输入错误:这个字符在 Unicode 数据库中被命名为“lamda ”,而没有“b”。根据英文维基百科文章[“Lambda”](https://fpy.li/18-26)，Unicode Consortium 采用这种拼写是因为“希腊国家机构表达的偏好”。

[13] 看着 python-dev 邮件列表中的讨论，我以为`else`被拒绝的一个原因是对如何在`match`内缩进缺乏共识:`else`应该和`match`缩进一级，还是和`case`缩进一级？

[14] 见[幻灯片 21《Python 很牛逼》](https://fpy.li/18-29)。*