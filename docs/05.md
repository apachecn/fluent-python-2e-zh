<link href="Styles/Style00.css" rel="stylesheet" type="text/css"> <link href="Styles/Style01.css" rel="stylesheet" type="text/css"> 

# 第四章。 Unicode 文本与字节

> 人类使用文本。计算机用字节说话。
> 
> Esther Nam 和 Travis Fischer，“Python 中的字符编码和 Unicode”[1]

Python 3 引入了人类文本字符串和原始字节序列之间的明显区别。字节序列到 Unicode 文本的隐式转换已经成为过去。本章讨论 Unicode 字符串、二进制序列以及用于在它们之间转换的编码。

根据您使用 Python 的工作类型，您可能认为理解 Unicode 并不重要。这不太可能，但无论如何都无法逃脱`str`与`byte`的鸿沟。另外，您会发现专门化的二进制序列类型提供了“通用”Python 2 `str`类型所没有的特性。

在本章中，我们将探讨以下主题:

*   字符、码位和字节表示

*   二进制序列的独特特征:`bytes`、`bytearray`和`memoryview`

*   完整 Unicode 和旧式字符集的编码

*   避免和处理编码错误

*   处理文本文件的最佳实践

*   默认编码陷阱和标准 I/O 问题

*   带规范化的安全 Unicode 文本比较

*   用于规范化、大小写折叠和强力删除音调符号的实用函数

*   使用`locale`和 *pyuca* 库对 Unicode 文本进行正确排序

*   Unicode 数据库中的字符元数据

*   处理`str`和`bytes`的双模式 API

# 本章的新内容

Python 3 中对 Unicode 的支持已经非常全面和稳定，所以最值得注意的是“按名称查找字符”，描述了一个用于搜索 Unicode 数据库的实用程序——一个从命令行查找带圆圈的数字和微笑的猫的好方法。

值得一提的一个小变化是 Windows 上的 Unicode 支持，从 Python 3.6 开始，它变得更好更简单，我们将在“小心编码默认值”中看到。

让我们从字符、代码点和字节这些不太新但很基本的概念开始。

###### 注意

对于 第二版，我扩展了关于`struct`模块的部分，并在线发布在[“用 struct 解析二进制记录”](https://fpy.li/4-3)的[*fluentpython.com*](http://fluentpython.com)配套网站上。

在那里你还会发现[“构建多字符表情符号”](https://fpy.li/4-4)，描述了如何通过组合 Unicode 字符来制作国旗、彩虹旗、不同肤色的人以及多样化的家庭图标。

# 性格问题

“字符串”的概念很简单:字符串是一系列字符。问题出在“性格”的定义上。

在 2021 年，我们对“字符”的最佳定义是 Unicode 字符。因此，我们从 Python 3 `str`中得到的项目是 Unicode 字符，就像 Python 2 中的`unicode`对象的项目一样——而不是我们从 Python 2 `str`中得到的原始字节。

Unicode 标准明确地将字符的标识与特定的字节表示分开:

*   一个字符的身份——它的 *码位*——是一个从 0 到 1，114，111(以 10 为基数)的数字，在 Unicode 标准中显示为 4 到 6 个十六进制数字加一个“U+”前缀，从 U+0000 到 U+10FFFF。例如，字母 A 的码位是 U+0041，欧元符号是 U+20AC，音乐符号 G 谱号被分配给码位 U+1D11E。在 Unicode 13 . 0 . 0(Python 3 . 10 . 0 B4 中使用的标准)中，大约 13%的有效代码点分配有字符。

*   代表一个字符的实际字节取决于正在使用的 *编码*。编码是一种将代码点转换为字节序列的算法，反之亦然。字母 A (U+0041)的码位在 UTF-8 编码中被编码为单字节`\x41`，或者在 UTF-16LE 编码中被编码为字节`\x41\x00`。作为另一个例子，UTF-8 需要三个字节—`\xe2\x82\xac`—来编码欧元符号(U+20AC)，但是在 UTF-16LE 中，相同的码位被编码为两个字节:`\xac\x20`。

从码位转换成字节是*编码*；从字节转换成码位就是 *解码*。参见示例 4-1 。

##### 示例 4-1：编码和解码

```
>>>s='café'>>>len(s)①4>>>b=s.encode('utf8')②>>>bb'caf\xc3\xa9'③>>>len(b)④5>>>b.decode('utf8')⑤'café'
```

① `str` `'café'`有四个 Unicode 字符。

② 使用 UTF-8 编码对`str`到`bytes`进行编码。

③ `bytes`文字有一个`b`前缀。

④ `bytes` `b`有五个字节(在 UTF-8 中“é”的码位编码为两个字节)。

⑤ 使用 UTF-8 编码对`bytes`至`str`进行解码。

###### 小费

如果你需要一个记忆助手来帮助区分`.decode()`和`.encode()`，说服自己字节序列可以是神秘的机器核心转储，而 Unicode `str`对象是“人类”文本。所以，我们*解码* `bytes`到`str`得到人类可读的文本，我们*编码* `str`到`bytes`进行存储或传输是有意义的。

虽然 Python 3 `str`几乎是 Python 2 `unicode`类型的新名称，但 Python 3 `bytes`并不是简单地将旧的`str`改名，还有与之密切相关的`bytearray`类型。因此，在讨论编码/解码问题之前，有必要先了解一下二进制序列类型。

# 字节要素

新的二进制序列类型在许多方面与 Python 2 `str`不同。首先要知道的是二进制序列有两种基本的内置类型:Python 3 中引入的不可变类型`bytes`和 Python 2.6 中添加的可变类型`bytearray`。[2]Python 文档有时使用通用术语“字节串”来指代`bytes`和`bytearray`。我避免使用这个令人困惑的术语。

`bytes`或`bytearray`中的每一项都是从 0 到 255 的整数，而不是像 Python 2 `str`中的单字符字符串。但是，二进制序列的一个片段总是产生相同类型的二进制序列，包括长度为 1 的片段。参见示例 4-2 。

##### 示例 4-2：一个五字节序列，如`bytes`和`bytearray`

```
>>> cafe=bytes('café',encoding='utf_8')①>>> cafeb'caf\xc3\xa9' >>> cafe[0]②99 >>> cafe[:1]③b'c' >>> cafe_arr=bytearray(cafe)>>> cafe_arr④bytearray(b'caf\xc3\xa9') >>> cafe_arr[-1:]⑤bytearray(b'\xa9')
```

① 给定一个编码，`bytes`可以从`str`构建。

② `range(256)`中的每一项都是整数。

③ `bytes`的切片也是`bytes`——甚至是单个字节的切片。

④ `bytearray`没有文字语法:它们显示为`bytearray()`，带有一个`bytes`文字作为参数。

⑤ 一片`bytearray`也是一片`bytearray`。

###### 警告

事实上，`my_bytes[0]`检索一个`int`但是`my_bytes[:1]`返回一个长度为 1 的`bytes`序列只是令人惊讶，因为我们习惯了 Python 的`str`类型，其中`s[0] == s[:1]`。对于 Python 中的所有其他序列类型，1 项不同于长度为 1 的切片。

虽然二进制序列实际上是整数序列，但它们的文字符号反映了 ASCII 文本经常嵌入其中的事实。因此，根据每个字节值，使用四种不同的显示:

*   对于十进制代码为 32 到 126 的字节—从空格到`~`(波浪号)—使用 ASCII 字符本身。

*   对于对应于制表符、换行符、回车符和`\`的字节，使用转义序列`\t`、`\n`、`\r`和`\\`。

*   如果字符串分隔符`'`和`"`都出现在字节序列中，则整个序列由`'`分隔，其中的任何`'`都被转义为`\'`。 [3]

*   对于其他字节值，使用十六进制转义序列(例如，`\x00`是空字节)。

这就是为什么在示例 4-2 中你会看到`b'caf\xc3\xa9'`:前三个字节`b'caf'`在可打印的 ASCII 范围内，后两个不在。

`bytes`和`bytearray`都支持除格式化方法(`format`、`format_map`)和依赖 Unicode 数据的方法之外的所有`str`方法，包括`casefold`、`isdecimal`、`isidentifier`、`isnumeric`、`isprintable`和`encode`。这意味着您可以使用熟悉的字符串方法，如`endswith`、`replace`、`strip`、`translate`、`upper`以及许多其他带有二进制序列的方法——只使用`bytes`而不使用`str`参数。此外，`re`模块中的正则表达式函数也适用于二进制序列，如果正则表达式是从二进制序列而不是`str`编译的。从 Python 3.5 开始，`%`操作符再次处理二进制序列。 [4]

二进制序列有一个`str`没有的类方法，叫做`fromhex`，它通过解析由空格分隔的成对十六进制数字来构建二进制序列:

```
>>> bytes.fromhex('31 4B CE A9')
b'1K\xce\xa9'
```

构建`bytes`或`bytearray`实例的另一种方法是调用它们的构造函数:

*   一个`str`和一个`encoding`关键字参数

*   提供值从 0 到 255 的项的 iterable

*   实现缓冲协议(例如，`bytes`、`bytearray`、`memoryview`、`array.array`)的对象，将字节从源对象复制到新创建的二进制序列

###### 警告

直到 Python 3.5，还可以用单个整数调用`bytes`或`bytearray`来创建用空字节初始化的那个大小的二进制序列。Python 3.5 中不赞成使用此签名，Python 3.6 中移除了此签名。参见[PEP 467——二进制序列的微小 API 改进](https://fpy.li/pep467)。

从类似缓冲区的对象构建二进制序列是一个低级操作，可能涉及类型转换。参见示例 4-3 中的演示。

##### 示例 4-3：从数组的原始数据中初始化字节

```
>>> importarray>>> numbers=array.array('h',[-2,-1,0,1,2])①>>> octets=bytes(numbers)②>>> octetsb'\xfe\xff\xff\xff\x00\x00\x01\x00\x02\x00' ③
```

① Typecode `'h'`创建一个短整数`array`(16 位)。

② `octets`保存组成`numbers`的字节的副本。

③ 这是代表 5 个短整数的 10 个字节。

从任何类似缓冲区的源创建一个`bytes`或`bytearray`对象将总是复制字节。相反，`memoryview`对象让你在二进制数据结构之间共享内存，正如我们在“内存视图”中看到的。

在对 Python 中的二进制序列类型进行了基本探索之后，让我们看看它们是如何与字符串相互转换的。

# 基本编码器/解码器

Python 发行版捆绑了 100 多个 *编解码器*(编码器/解码器)，用于文本到字节的转换，反之亦然。每个编解码器都有一个名称，如`'utf_8'`，通常还有别名，如`'utf8'`、`'utf-8'`和`'U8'`，您可以在`open()`、`str.encode()`、`bytes.decode()`等函数中将其用作`encoding`参数。示例 4-4 显示了编码为三个不同字节序列的相同文本。

##### 示例 4-4：用三种编解码器编码的字符串“厄尔尼诺”产生非常不同的字节序列

```
>>> for codec in ['latin_1', 'utf_8', 'utf_16']:
...     print(codec, 'El Niño'.encode(codec), sep='\t')
...
latin_1 b'El Ni\xf1o'
utf_8   b'El Ni\xc3\xb1o'
utf_16  b'\xff\xfeE\x00l\x00 \x00N\x00i\x00\xf1\x00o\x00'
```

图 4-1 展示了从字母“A”到 G 谱号音乐符号等字符生成字节的各种编解码器。请注意，最后三种编码是可变长度的多字节编码。

![Encodings demonstration table](Images/flpy_0401.png)

###### 图 4-1。12 个字符，它们的码位，以及它们在 7 种不同编码中的字节表示(十六进制)(星号表示该字符不能在该编码中表示)。

图 4-1 中所有的星号表明一些编码，像 ASCII 甚至多字节 GB2312，不能代表每一个 Unicode 字符。然而，UTF 编码是为处理每个 Unicode 码位而设计的。

图 4-1 中所示的编码被选为代表性样本:

`latin1` a.k.a. `iso8859_1`

重要是因为它是其他编码的基础，比如`cp1252`和 Unicode 本身(注意`latin1`字节值是如何出现在`cp1252`字节甚至代码点中的)。

`cp1252`

微软创建的一个有用的`latin1`超集，增加了像花引号和€(欧元)这样有用的符号；一些 Windows 应用程序称之为“ANSI”，但它从来都不是真正的 ANSI 标准。

`cp437`

IBM PC 的原始字符集，带有方框图字符。与后来出现的`latin1`格格不入。

`gb2312`

编码 mainland China 使用的简体中文表意文字的传统标准；亚洲语言中广泛使用的多字节编码之一。

`utf-8`

web 上最常见的 8 位编码，到目前为止，截至 2021 年 7 月，[《W³Techs:网站字符编码的使用统计》](https://fpy.li/4-5)声称，97%的网站使用 UTF-8，高于我在 2014 年 9 月本书第一版中写下这段话时的 81.4%。

`utf-16le`

UTF 16 位编码方案的一种形式；所有 UTF-16 编码都通过称为“代理对”的转义序列支持 U+FFFF 之外的代码点

###### 警告

UTF-16 早在 1996 年就取代了最初的 16 位 Unicode 1.0 编码——UCS-2。UCS-2 仍然在许多系统中使用，尽管从上个世纪开始就被弃用，因为它只支持 U+FFFF 的代码点。截至 2021 年，超过 57%的分配码点高于 U+FFFF，包括所有重要的表情符号。

至此，对常见编码的概述已经完成，接下来我们将处理编码和解码操作中的问题。

# 理解编码/解码问题

虽然 有一个通用的`UnicodeError`异常，但是 Python 报告的错误通常更具体:要么是一个`UnicodeEncodeError`(将`str`转换为二进制序列时)，要么是一个`UnicodeDecodeError`(将二进制序列读入`str`时)。当源代码编码意外时，加载 Python 模块也可能引发`SyntaxError`。我们将在下一节中展示如何处理所有这些错误。

###### 小费

当出现 Unicode 错误时，首先要注意的是异常的确切类型。是一个`UnicodeEncodeError`、一个`UnicodeDecodeError`还是其他一些提到编码问题的错误(例如`SyntaxError`)。要解决问题，你得先了解它。

## 应对 unicode 编码错误

大多数非 UTF 编解码器只处理一小部分 Unicode 字符。当将文本转换为字节时，如果在目标编码中没有定义字符，将引发`UnicodeEncodeError`，除非通过向编码方法或函数传递一个`errors`参数来提供特殊处理。错误处理程序的行为如示例 4-5 所示。

##### 示例 4-5：字节编码:成功和错误处理

```
>>> city='São Paulo'>>> city.encode('utf_8')①b'S\xc3\xa3o Paulo' >>> city.encode('utf_16')b'\xff\xfeS\x00\xe3\x00o\x00 \x00P\x00a\x00u\x00l\x00o\x00' >>> city.encode('iso8859_1')②b'S\xe3o Paulo' >>> city.encode('cp437')③Traceback (most recent call last):
 File "<stdin>", line 1, in <module> File "/.../lib/python3.4/encodings/cp437.py", line 12, in encodereturncodecs.charmap_encode(input,errors,encoding_map)UnicodeEncodeError: 'charmap' codec can't encode character '\xe3' inposition 1: character maps to <undefined> >>> city.encode('cp437',errors='ignore')④b'So Paulo' >>> city.encode('cp437',errors='replace')⑤b'S?o Paulo' >>> city.encode('cp437',errors='xmlcharrefreplace')⑥b'S&#227;o Paulo'
```

① UTF 编码处理任何`str`。

② `iso8859_1`也适用于`'São Paulo'`字符串。

③ `cp437`无法对`'ã'`(带颚化符的“a”)进行编码。默认的错误处理程序`'strict'`引发`UnicodeEncodeError`。

④ `error='ignore'`处理程序跳过无法编码的字符；这通常是一个非常糟糕的想法，会导致无声的数据丢失。

⑤ 编码时，`error='replace'`用`'?'`替代无法编码的字符；数据也会丢失，但用户会得到一个线索，表明出了问题。

⑥ `'xmlcharrefreplace'`用 XML 实体替换无法编码的字符。如果你不能使用 UTF，并且你不能承受数据丢失，这是唯一的选择。

###### 注意

`codecs`错误处理是可扩展的。您可以通过向`codecs.register_error`函数传递一个名称和一个错误处理函数来为`errors`参数注册额外的字符串。参见[中的`codecs.register_error`文档](https://fpy.li/4-6)。

ASCII 是我所知道的所有编码的一个公共子集，因此如果文本完全由 ASCII 字符组成，编码应该总是有效的。Python 3.7 新增了一个布尔方法 [`str.isascii()`](https://fpy.li/4-7) 来检查你的 Unicode 文本是否是 100%纯 ASCII。如果是，您应该能够在不引发`UnicodeEncodeError`的情况下以任何编码将其编码为字节。

## 应对 UnicodeDecodeError 错误

不是每个字节都保存一个有效的 ASCII 字符，也不是每个字节序列都是有效的 UTF-8 或 UTF-16；因此，当您在将二进制序列转换为文本时采用这些编码之一时，如果发现意外的字节，您将得到一个`UnicodeDecodeError`。

另一方面，许多传统的 8 位编码，如`'cp1252'`、`'iso8859_1'`和`'koi8_r'`，能够解码任何字节流，包括随机噪声，而不会报告错误。因此，如果您的程序假定了错误的 8 位编码，它将会默默地解码垃圾。

###### 小费

Garbled characters are known as gremlins or mojibake (文字化け—Japanese for “transformed text”).

示例 4-6 说明了使用错误的编解码器可能会产生小精灵或 `UnicodeDecodeError` 。

##### 示例 4-6：解码从`str`到字节:成功和错误处理

```
>>> octets=b'Montr\xe9al'①>>> octets.decode('cp1252')②'Montréal' >>> octets.decode('iso8859_7')③'Montrιal' >>> octets.decode('koi8_r')④'MontrИal' >>> octets.decode('utf_8')⑤Traceback (most recent call last):
 File "<stdin>", line 1, in <module>UnicodeDecodeError: 'utf-8' codec can't decode byte 0xe9 in position 5:invalid continuation byte >>> octets.decode('utf_8',errors='replace')⑥'Montr�al'
```

① 单词“Montréal”编码为`latin1`；`'\xe9'`是表示“é”的字节。

② 用 Windows 1252 解码可以，因为它是`latin1`的超集。

③ ISO-8859-7 是为希腊文设计的，所以`'\xe9'`字节被曲解，并且没有发出错误。

④ KOI8-R 代表俄罗斯。现在`'\xe9'`代表西里尔字母“и”。

⑤ `'utf_8'`编解码器检测到`octets`不是有效的 UTF-8，并引发`UnicodeDecodeError`。

⑥ 使用`'replace'`错误处理，`\xe9`被替换为“↑”(代码点 U+FFFD )，官方 Unicode `REPLACEMENT CHARACTER`用来表示未知字符。

## 加载带有意外编码的模块时出现语法错误

UTF-8 是 Python 3 的默认源代码编码，正如 ASCII 是 Python 2 的默认编码一样。如果你装一个*。py* 模块包含非 UTF-8 数据且没有编码声明，您会得到如下消息:

```
SyntaxError: Non-UTF-8 code starting with '\xe1' in file ola.py on line
  1, but no encoding declared; see https://python.org/dev/peps/pep-0263/
  for details
```

因为 UTF-8 广泛部署在 GNU/Linux 和 macOS 系统中，一个可能的场景是打开一个*。用`cp1252`在 Windows 上创建的 py* 文件。注意，这个错误甚至在 Python for Windows 中也会发生，因为 Python 3 源代码的默认编码是跨所有平台的 UTF-8。

要解决这个问题，在文件顶部添加一个神奇的`coding`注释，如示例 4-7 所示。

##### 示例 4-7：OLA . py:“你好，世界！”葡萄牙语

```
# coding: cp1252

print('Olá, Mundo!')
```

###### 小费

既然 Python 3 源代码不再局限于 ASCII，而是默认为优秀的 UTF-8 编码，那么对像`'cp1252'`这样的传统编码的源代码的最好“修复”就是将它们转换成 UTF-8，而不要为`coding`注释费心。如果你的编辑器不支持 UTF 8，是时候换一个了。

假设你有一个文本文件，可以是源代码或者诗歌，但是你不知道它的编码。如何检测实际的编码？下一节的答案。

## 如何发现字节序列的编码

如何找到一个字节序列的编码？简而言之:不能。必须告诉你。

一些通信协议和文件格式，如 HTTP 和 XML，包含明确告诉我们内容如何编码的头。您可以肯定一些字节流不是 ASCII，因为它们包含超过 127 的字节值，并且 UTF-8 和 UTF-16 的构建方式也限制了可能的字节序列。

然而，考虑到人类语言也有它们的规则和限制，一旦你假设一个字节流是人类*纯文本*，使用试探法和统计学就有可能嗅出它的编码。例如，如果`b'\x00'`字节是常见的，那么它很可能是 16 位或 32 位编码，而不是 8 位方案，因为纯文本中的空字符是 bug。当字节序列`b'\x20\x00'`经常出现时，它更有可能是 UTF-16LE 编码中的空格字符(U+0020 ),而不是模糊的 U+2000 `EN QUAD`字符——不管那是什么。

这就是软件包[“Chardet—通用字符编码检测器”](https://fpy.li/4-8)猜测 30 多种支持的编码之一的工作方式。Chardet 是一个 Python 库，你可以在你的程序中使用，但是也包括一个命令行工具`chardetect`。下面是它在本章源文件中报告的内容:

```
$ chardetect 04-text-byte.asciidoc
04-text-byte.asciidoc: utf-8 with confidence 0.99
```

尽管编码文本的二进制序列通常不带有显式的编码提示，但 UTF 格式可能会在文本内容前添加一个字节顺序标记。接下来将对此进行解释。

## BOM:一个有用的小精灵

在 示例 4-4 中，你可能已经注意到在 UTF-16 编码序列的开头有几个额外的字节。他们又来了:

```
>>> u16 = 'El Niño'.encode('utf_16')
>>> u16
b'\xff\xfeE\x00l\x00 \x00N\x00i\x00\xf1\x00o\x00'
```

字节为`b'\xff\xfe'`。那是一个*BOM*——字节顺序标记——表示执行编码的英特尔 CPU 的“小端”字节顺序。

在 little-endian 机器上，对于每个码位，最不重要的字节排在前面:字母`'E'`，码位 U+0045(十进制 69)，在字节偏移量 2 和 3 中编码为`69`和`0`:

```
>>> list(u16)
[255, 254, 69, 0, 108, 0, 32, 0, 78, 0, 105, 0, 241, 0, 111, 0]
```

在大端 CPU 上，编码会相反；`'E'`将被编码为`0`和`69`。

为了避免混淆，UTF-16 编码在要编码的文本前面加上特殊的不可见字符`ZERO WIDTH NO-BREAK SPACE` (U+FEFF)。在小端系统中，编码为`b'\xff\xfe'`(十进制 255，254)。因为按照设计，Unicode 中没有 U+FFFE 字符，所以字节序列`b'\xff\xfe'`必须表示小端编码中的`ZERO WIDTH NO-BREAK SPACE`，所以编解码器知道使用哪个字节顺序。

有一种 UTF-16 的变体——UTF-16LE——是显式的小端字节序，还有一种显式的大端字节序——UTF-16BE。如果您使用它们，BOM 不会生成:

```
>>> u16le = 'El Niño'.encode('utf_16le')
>>> list(u16le)
[69, 0, 108, 0, 32, 0, 78, 0, 105, 0, 241, 0, 111, 0]
>>> u16be = 'El Niño'.encode('utf_16be')
>>> list(u16be)
[0, 69, 0, 108, 0, 32, 0, 78, 0, 105, 0, 241, 0, 111]
```

如果存在的话，BOM 应该被 UTF-16 编解码器过滤掉，这样你就只能得到没有前导`ZERO WIDTH NO-BREAK SPACE`的文件的实际文本内容。Unicode 标准规定，如果一个文件是 UTF-16 并且没有 BOM，它应该被假定为 UTF-16BE(大端字节序)。然而，英特尔 x86 架构是小端的，所以在野外有大量没有 BOM 的小端 UTF-16。

整个字节序问题只影响使用多于一个字节的字的编码，比如 UTF-16 和 UTF-32。UTF-8 的一大优势是，不管机器的字节顺序如何，它都会产生相同的字节序列，因此不需要 BOM。尽管如此，一些 Windows 应用程序(特别是记事本)还是会将 BOM 添加到 UTF-8 文件中 Excel 依赖 BOM 来检测 UTF-8 文件，否则它会假定内容是用 Windows 代码页编码的。这种带有 BOM 的 UTF-8 编码在 Python 的编解码器注册表中被称为 UTF-8-SIG。UTF-8-SIG 中编码的字符 U+FEFF 是三字节序列`b'\xef\xbb\xbf'`。因此，如果一个文件以这三个字节开头，它很可能是一个带有 BOM 的 UTF-8 文件。

# 凯勒关于 UTF-8-SIG 的提示

技术评论者之一 Caleb Hattingh 建议在阅读 UTF 8 文件时总是使用 UTF 8-SIG 编解码器。这是无害的，因为 UTF-8-SIG 正确地读取有或没有 BOM 的文件，并且不返回 BOM 本身。在写作时，我推荐使用 UTF-8 来实现一般的互操作性。例如，如果 Python 脚本以注释:`#!/usr/bin/env python3`开头，就可以在 Unix 系统中执行。文件的前两个字节必须是`b'#!'`才能工作，但是 BOM 打破了这个惯例。如果你有将数据导出到需要 BOM 的应用程序的特定要求，使用 UTF-8-SIG，但要注意 Python 的[编解码器文档](https://fpy.li/4-9)说:“在 UTF-8 中，不鼓励使用 BOM，通常应该避免。”

我们现在转到 Python 3 中的文本文件处理。

# 处理文本文件

处理文本 I/O 的 最佳实践是“Unicode 三明治”(图 4-2 )。 [5] 这意味着`bytes`应该在输入时尽早解码为`str`(例如，当打开文件进行读取时)。三明治的“填充”是程序的业务逻辑，文本处理只在`str`对象上完成。你不应该在其他处理过程中编码或解码。在输出时，`str`尽可能晚地被编码为`bytes`。大多数 web 框架都是这样工作的，我们在使用它们的时候很少接触`bytes`。例如，在 Django 中，您的视图应该输出 Unicode`str`；Django 自己负责编码对`bytes`的响应，默认使用 UTF-8。

Python 3 更容易遵循 Unicode 三明治的建议，因为`open()`内置在读取时进行必要的解码，当以文本模式写入文件时进行编码，所以你从`my_file.read()`获得并传递给`my_file.write(text)`的都是`str`对象。

因此，使用文本文件显然很简单。但是如果你依赖默认编码，你会被咬。

![Unicode sandwich diagram](Images/flpy_0402.png)

###### 图 4-2。Unicode 三明治:当前文本处理的最佳实践。

考虑示例 4-8 中的控制台会话。你能发现这个漏洞吗？

##### 示例 4-8：平台编码问题(如果您在您的机器上尝试，您可能会看到问题，也可能看不到)

```
>>> open('cafe.txt', 'w', encoding='utf_8').write('café')
4
>>> open('cafe.txt').read()
'cafÃ©'
```

bug:我在写文件时指定了 UTF-8 编码，但在读取文件时没有这样做，所以 Python 假设 Windows 默认文件编码——代码页 1252——并且文件中的结尾字节被解码为字符`'Ã©'`而不是`'é'`。

我在 Python 3.8.1，64 位，Windows 10 (build 18363)上运行了示例 4-8 。同样的语句在最近的 GNU/Linux 或 macOS 上运行得非常好，因为它们的默认编码是 UTF-8，给人一种一切正常的错觉。如果在打开文件进行写入时省略了 encoding 参数，那么将使用本地默认编码，我们将使用相同的编码正确地读取文件。但是，这个脚本会根据平台，甚至根据同一平台中的区域设置，生成具有不同字节内容的文件，从而产生兼容性问题。

###### 小费

必须在多台机器上或多种场合下运行的代码不应该依赖于编码默认值。当打开文本文件时，总是传递一个显式的`encoding=`参数，因为缺省值可能从一台机器到下一台机器，或者从一天到下一天发生变化。

在示例 4-8 中一个奇怪的细节是，第一条语句中的`write`函数报告写了四个字符，但是在下一行中读了五个字符。示例 4-9 是示例 4-8 的的延伸版本，解释了那个和其他细节。

##### 示例 4-9：对运行在 Windows 上的示例 4-8 的进一步检查揭示了错误以及如何修复它

```
>>> fp=open('cafe.txt','w',encoding='utf_8')>>> fp①<_io.TextIOWrapper name='cafe.txt' mode='w' encoding='utf_8'> >>> fp.write('café')②4 >>> fp.close()>>> importos>>> os.stat('cafe.txt').st_size③5 >>> fp2=open('cafe.txt')>>> fp2④<_io.TextIOWrapper name='cafe.txt' mode='r' encoding='cp1252'> >>> fp2.encoding⑤'cp1252' >>> fp2.read()⑥'cafÃ©' >>> fp3=open('cafe.txt',encoding='utf_8')⑦>>> fp3<_io.TextIOWrapper name='cafe.txt' mode='r' encoding='utf_8'> >>> fp3.read()⑧'café' >>> fp4=open('cafe.txt','rb')⑨>>> fp4⑩<_io.BufferedReader name='cafe.txt'> >>> fp4.read()⑪b'caf\xc3\xa9'
```

① 默认情况下，`open`使用文本模式并返回一个带有特定编码的`TextIOWrapper`对象。

② `TextIOWrapper`上的`write`方法返回写入的 Unicode 字符数。

③ `os.stat`表示文件有 5 个字节；UTF-8 将`'é'`编码为 2 个字节，0xc3 和 0xa9。

④ 打开一个没有显式编码的文本文件会返回一个`TextIOWrapper`，其编码被设置为本地的默认值。

⑤ 一个`TextIOWrapper`对象有一个可以检查的编码属性:在本例中是`cp1252`。

⑥ 在 Windows `cp1252`编码中，字节 0xc3 是一个“γ”(带颚化符的 A)，0xa9 是版权标志。

⑦ 用正确的编码打开同一文件。

⑧ 预期的结果:同样的四个 Unicode 字符用于`'café'`。

⑨ `'rb'`标志以二进制模式打开文件进行读取。

⑩ 返回的对象是`BufferedReader`而不是`TextIOWrapper`。

⑪ 如预期的那样，读取返回字节。

###### 小费

不要以二进制模式打开文本文件，除非你需要分析文件内容来确定编码——即使这样，你也应该使用 Chardet 而不是重新发明轮子(参见[“如何发现一个字节序列的编码”](Images/#discover_encoding))。普通代码应该只使用二进制模式打开二进制文件，就像光栅图像一样。

示例 4-9 中的问题与打开文本文件时依赖默认设置有关。这种默认值有几个来源，如下一节所示。

## 当心编码默认值

几个设置影响 Python 中 I/O 的编码默认值。参见示例 4-10 中的 *default_encodings.py* 脚本。

##### 示例 4-10：探索编码默认值

```
import locale
import sys

expressions = """
 locale.getpreferredencoding()
 type(my_file)
 my_file.encoding
 sys.stdout.isatty()
 sys.stdout.encoding
 sys.stdin.isatty()
 sys.stdin.encoding
 sys.stderr.isatty()
 sys.stderr.encoding
 sys.getdefaultencoding()
 sys.getfilesystemencoding()
 """

my_file = open('dummy', 'w')

for expression in expressions.split():
    value = eval(expression)
    print(f'{expression:>30} -> {value!r}')
```

GNU/Linux (Ubuntu 14.04 到 19.10)和 macOS (10.9 到 10.14)上的示例 4-10 的输出是相同的，表明`UTF-8`在这些系统中到处都被使用:

```
$ python3 default_encodings.py
 locale.getpreferredencoding() -> 'UTF-8'
                 type(my_file) -> <class '_io.TextIOWrapper'>
              my_file.encoding -> 'UTF-8'
           sys.stdout.isatty() -> True
           sys.stdout.encoding -> 'utf-8'
            sys.stdin.isatty() -> True
            sys.stdin.encoding -> 'utf-8'
           sys.stderr.isatty() -> True
           sys.stderr.encoding -> 'utf-8'
      sys.getdefaultencoding() -> 'utf-8'
   sys.getfilesystemencoding() -> 'utf-8'
```

然而在 Windows 上，输出是示例 4-11 。

##### 示例 4-11：Windows 10 PowerShell 上的默认编码(在 cmd.exe 上输出相同)

```
>chcp①Activecodepage:437>pythondefault_encodings.py②locale.getpreferredencoding()->'cp1252'③type(my_file)-><class'_io.TextIOWrapper'>my_file.encoding->'cp1252'④sys.stdout.isatty()->True⑤sys.stdout.encoding->'utf-8'⑥sys.stdin.isatty()->Truesys.stdin.encoding->'utf-8'sys.stderr.isatty()->Truesys.stderr.encoding->'utf-8'sys.getdefaultencoding()->'utf-8'sys.getfilesystemencoding()->'utf-8'
```

① `chcp`显示了控制台的活动代码页:`437`。

② 运行 *default_encodings.py* ，输出到控制台。

③ `locale.getpreferredencoding()`是最重要的设定。

④ 文本文件默认使用`locale.getpreferredencoding()`。

⑤ 输出要去控制台，所以`sys.stdout.isatty()`就是`True`。

⑥ 现在，`sys.stdout.encoding`和`chcp`报告的控制台代码页不一样！

自从我写了这本书的第一版，Windows 本身和 Python for Windows 中的 Unicode 支持变得更好了。示例 4-11 曾经在 Windows 7 上报告 Python 3.4 中的四种不同编码。`stdout`、`stdin`和`stderr`的编码曾经与`chcp`命令报告的活动代码页相同，但现在它们都是`utf-8`感谢PEP 528——将 Windows 控制台编码更改为 Python 3.6 中实现的 UTF-8](https://fpy.li/pep528) ，以及*cmd.exe*中 PowerShell 中的 Unicode 支持(从 2018 年 10 月的 Windows 1809 开始)。 ^([6) 奇怪的是`chcp`和`sys.stdout.encoding`在`stdout`写入控制台时说不同的话，但是很棒的是，现在我们可以在 Windows 上打印 Unicode 字符串而没有编码错误——除非用户将输出重定向到一个文件，我们很快就会看到这一点。这并不意味着所有你最喜欢的表情符号都会出现在控制台上:这也取决于控制台使用的字体。

另一个变化是[PEP 529——将 Windows 文件系统编码改为 UTF-8](https://fpy.li/pep529) ，也是在 Python 3.6 中实现的，它将文件系统编码(用于表示目录和文件的名称)从微软专有的 MBCS 改为 UTF-8。

然而，如果示例 4-10 的输出被重定向到一个文件，如下所示:

```
Z:\>python default_encodings.py > encodings.log
```

然后，`sys.stdout.isatty()`的值变为`False`，并且`sys.stdout.encoding`由该机器中的`locale.getpreferredencoding()`、`'cp1252'`设定，但`sys.stdin.encoding`和`sys.stderr.encoding`仍为`utf-8`。

###### 小费

在 示例 4-12 中，我对 Unicode 文字使用了`'\N{}'`转义，在这里我们在`\N{}`中写入了字符的正式名称。它相当冗长，但是明确而安全:如果名字不存在，Python 会抛出`SyntaxError`——这比写一个可能是错误的十六进制数要好得多，但是你很久以后才会发现。你可能想写一个注释来解释字符代码，所以`\N{}`的冗长很容易被接受。

这意味着像示例 4-12 这样的脚本在打印到控制台时可以工作，但是当输出被重定向到文件时可能会中断。

##### 示例 4-12：标准输出检查

```
import sys
from unicodedata import name

print(sys.version)
print()
print('sys.stdout.isatty():', sys.stdout.isatty())
print('sys.stdout.encoding:', sys.stdout.encoding)
print()

test_chars = [
    '\N{HORIZONTAL ELLIPSIS}',       # exists in cp1252, not in cp437
    '\N{INFINITY}',                  # exists in cp437, not in cp1252
    '\N{CIRCLED NUMBER FORTY TWO}',  # not in cp437 or in cp1252
]

for char in test_chars:
    print(f'Trying to output {name(char)}:')
    print(char)
```

示例 4-12 显示`sys.stdout.isatty()`的结果、`sys.​stdout.encoding`的值以及这三个字符:

*   `'…'``HORIZONTAL ELLIPSIS`—存在于 CP 1252 中，不存在于 CP 437 中。

*   `'∞'``INFINITY`—存在于 CP 437 中，但不存在于 CP 1252 中。

*   `'㊷'``CIRCLED NUMBER FORTY TWO`—在 CP 1252 或 CP 437 中不存在。

当我在 PowerShell 或*cmd.exe*上运行 *stdout_check.py* 时，它会像图 4-3 中捕获的那样工作。

![Screen capture of `stdout_check.py` on PowerShell](Images/flpy_0403.png)

###### 图 4-3。在 PowerShell 上运行 *stdout_check.py* 。

尽管`chcp`报告活动代码为 437，但`sys.stdout.encoding`是 UTF-8，因此`HORIZONTAL ELLIPSIS`和`INFINITY`都正确输出。`CIRCLED NUMBER FORTY TWO`被矩形代替，但没有出现错误。假设它被识别为有效字符，但是控制台字体没有显示它的字形。

然而，当我将 *stdout_check.py* 的输出重定向到一个文件时，我得到了图 4-4 。

![Screen capture of `stdout_check.py` on PowerShell, redirecting output](Images/flpy_0404.png)

###### 图 4-4。在 PowerShell 上运行 *stdout_check.py* ，重定向输出。

图 4-4 演示的第一个问题是`UnicodeEncodeError`提及字符`'\u221e'`，因为`sys.stdout.encoding`是`'cp1252'`——一个没有`INFINITY`字符的代码页。

用`type`命令读取*out . txt*——或者像 VS Code 或 Sublime Text 这样的 Windows 编辑器——显示我得到的不是水平省略号，而是`'à'` ( `LATIN SMALL LETTER A WITH GRAVE`)。事实证明，CP 1252 中的字节值 0x85 表示`'…'`，但在 CP 437 中，相同的字节值表示`'à'`。因此，似乎活动代码页确实很重要，不是以一种明智或有用的方式，而是作为糟糕的 Unicode 体验的部分解释。

###### 注意

我使用了一台为美国市场配置的笔记本电脑，运行 Windows 10 OEM 来运行这些实验。针对其他国家/地区本地化的 Windows 版本可能会有不同的编码配置。例如，在巴西，Windows 控制台默认使用代码页 850，而不是 437。

为了总结这个令人恼火的默认编码问题，让我们最后看一下示例 4-11 中的不同编码:

*   如果在打开文件时省略了`encoding`参数，则缺省值由`locale.getpreferredencoding()`给出(示例 4-11 中的`'cp1252'`)。

*   在 Python 3.6 之前，`sys.stdout|stdin|stderr`的编码曾经由 [`PYTHONIOENCODING`](https://fpy.li/4-12) 环境变量设置——现在该变量被忽略，除非 [`PYTHONLEGACYWINDOWSSTDIO`](https://fpy.li/4-13) 被设置为非空字符串。否则，标准 I/O 的编码是交互式 I/O 的 UTF-8，或者如果输出/输入被重定向到/来自文件，则由`locale.getpreferredencoding()`定义。

*   `sys.getdefaultencoding()`由 Python 内部使用，用于二进制数据与`str`之间的隐式转换。不支持更改此设置。

*   `sys.getfilesystemencoding()`用于编码/解码文件名(非文件内容)。当`open()`为文件名获取一个`str`参数时使用它；如果文件名是作为一个`bytes`参数给出的，它将被原封不动地传递给 OS API。

###### 注意

在 GNU/Linux 和 macOS 上，默认情况下所有这些编码都设置为 UTF-8，而且已经有好几年了，所以 I/O 处理所有的 Unicode 字符。在 Windows 上，不仅在同一个系统中使用不同的编码，而且它们通常是像`'cp850'`或`'cp1252'`这样只支持 ASCII 的代码页，有 127 个额外的字符，这些字符从一种编码到另一种编码是不相同的。因此，Windows 用户更有可能面临编码错误，除非他们格外小心。

总而言之，最重要的编码设置是由`locale.getpreferredencoding()`返回的:这是打开文本文件和`sys.stdout/stdin/stderr`重定向到文件时的默认设置。然而,[文档](https://fpy.li/4-14)的内容如下(部分):

> `locale.getpreferredencoding(do_setlocale=True)`
> 
> 返回用于文本数据的编码，根据用户喜好。用户偏好在不同的系统上有不同的表达方式，在某些系统上可能无法通过编程实现，因此该函数只返回一个猜测值。[……]

因此，关于编码默认值的最佳建议是:不要依赖它们。

如果您遵循 Unicode 三明治的建议，并且总是明确您的程序中的编码，您将避免许多痛苦。不幸的是，即使你把你的`bytes`正确地转换成`str`，Unicode 也是痛苦的。接下来的两个部分涵盖了在 ASCII-land 中很简单，但在 planet Unicode 中变得相当复杂的主题:文本规范化(例如，将文本转换成统一的表示，以便进行比较)和排序。

# 规范化 Unicode 以进行可靠的比较

字符串比较 很复杂，因为 Unicode 有组合字符:附加在前面字符上的音调符号和其他标记，在打印时显示为一个。

例如，单词“café”可能有两种组合方式，使用四个或五个代码点，但结果看起来完全相同:

```
>>> s1 = 'café'
>>> s2 = 'cafe\N{COMBINING ACUTE ACCENT}'
>>> s1, s2
('café', 'café')
>>> len(s1), len(s2)
(4, 5)
>>> s1 == s2
False
```

将`COMBINING ACUTE ACCENT` (U+0301)放在“e”之后会呈现“é”。在 Unicode 标准中，像`'é'`和`'e\u0301'`这样的序列被称为“规范等价物”，应用程序应该将它们视为相同。但是 Python 看到了两个不同的代码点序列，并认为它们不相等。

解决方法是`unicodedata.normalize()`。该函数的第一个参数是四个字符串之一:`'NFC'`、`'NFD'`、`'NFKC'`和`'NFKD'`。先说前两个。

规范化形式 C (NFC) 组成码点以产生最短的等价字符串，而 NFD 分解，将组成的字符扩展为基本字符和单独的组合字符。这两种规范化都使比较按预期进行，如下例所示:

```
>>> from unicodedata import normalize
>>> s1 = 'café'
>>> s2 = 'cafe\N{COMBINING ACUTE ACCENT}'
>>> len(s1), len(s2)
(4, 5)
>>> len(normalize('NFC', s1)), len(normalize('NFC', s2))
(4, 4)
>>> len(normalize('NFD', s1)), len(normalize('NFD', s2))
(5, 5)
>>> normalize('NFC', s1) == normalize('NFC', s2)
True
>>> normalize('NFD', s1) == normalize('NFD', s2)
True
```

键盘驱动程序通常生成合成字符，因此用户键入的文本将默认为 NFC。然而，为了安全起见，在保存之前用`normalize('NFC', user_text)`规范化字符串可能是好的。NFC 也是 W3C 在[“万维网字符模型:字符串匹配和搜索”](https://fpy.li/4-15)中推荐的规范化形式。

一些单个字符被 NFC 标准化为另一个单个字符。电阻的欧姆(ω)单位的符号被归一化为希腊大写字母 omega。它们在视觉上是相同的，但它们比较起来是不相等的，因此有必要进行标准化以避免意外:

```
>>> from unicodedata import normalize, name
>>> ohm = '\u2126'
>>> name(ohm)
'OHM SIGN'
>>> ohm_c = normalize('NFC', ohm)
>>> name(ohm_c)
'GREEK CAPITAL LETTER OMEGA'
>>> ohm == ohm_c
False
>>> normalize('NFC', ohm) == normalize('NFC', ohm_c)
True
```

另外两种规范化形式是 NFKC 和 NFKD，其中字母 K 代表“兼容性”这些是规范化的更强形式，影响所谓的“兼容性字符”尽管 Unicode 的一个目标是每个字符都有一个“规范的”代码点，但是为了与现有标准兼容，有些字符会出现不止一次。例如，`MICRO SIGN`、`µ` ( `U+00B5`)被添加到 Unicode 中，以支持到`latin1`的往返转换，其中包括它，即使同一个字符是带有码位`U+03BC` ( `GREEK SMALL LETTER MU`)的希腊字母表的一部分。所以，微型符号被认为是“兼容性字符”

在 NFKC 和 NFKD 形式中，每个兼容性字符都被一个或多个被认为是“首选”表示的字符的“兼容性分解”所取代，即使有一些格式丢失——理想情况下，格式应该是外部标记的责任，而不是 Unicode 的一部分。举个例子，二分之一分数`'½'` ( `U+00BD`)的相容性分解是三个字符`'1/2'`的序列，微符号`'µ'` ( `U+00B5`)的相容性分解是小写的 mu `'μ'` ( `U+03BC`)。 [7]

以下是 NFKC 在实践中的工作方式:

```
>>> from unicodedata import normalize, name
>>> half = '\N{VULGAR FRACTION ONE HALF}'
>>> print(half)
½
>>> normalize('NFKC', half)
'1⁄2'
>>> for char in normalize('NFKC', half):
...     print(char, name(char), sep='\t')
...
1	DIGIT ONE
⁄	FRACTION SLASH
2	DIGIT TWO
>>> four_squared = '4²'
>>> normalize('NFKC', four_squared)
'42'
>>> micro = 'µ'
>>> micro_kc = normalize('NFKC', micro)
>>> micro, micro_kc
('µ', 'μ')
>>> ord(micro), ord(micro_kc)
(181, 956)
>>> name(micro), name(micro_kc)
('MICRO SIGN', 'GREEK SMALL LETTER MU')
```

虽然`'1⁄2'`是`'½'`的合理替代，而且微符号确实是小写的希腊文 mu，但是把`'4²'`转换成`'42'`就改变了意思。应用程序可以将`'4²'`存储为`'4<sup>2</sup>'`，但是`normalize`函数对格式化一无所知。因此，NFKC 或 NFKD 可能会丢失或扭曲信息，但它们可以为搜索和索引产生方便的中间表示。

不幸的是，使用 Unicode 时，一切总是比看起来复杂。对于`VULGAR FRACTION ONE HALF`，NFKC 归一化产生了由`FRACTION SLASH`连接的 1 和 2，而不是`SOLIDUS`，也称为“斜杠”ASCII 码十进制 47 的熟悉字符。因此，搜索三字符 ASCII 序列`'1/2'`不会找到规范化的 Unicode 序列。

###### 警告

NFKC 和 NFKD 规范化会导致数据丢失，应该只应用于特殊情况，如搜索和索引，而不是用于文本的永久存储。

当准备搜索或索引文本时，另一个操作是有用的:案例折叠，我们的下一个主题。

## 案例折叠

Case folding 本质上是将所有文本转换成小写，并进行一些额外的转换。它由`str.casefold()`方法支持。

对于任何只包含`latin1`字符的字符串`s`，`s.casefold()`产生与`s.lower()`相同的结果，只有两个例外——微型符号`'µ'`被更改为希腊文小写 mu(在大多数字体中看起来是一样的),德语 Eszett 或“sharp s”()变成了“ss”:

```
>>> micro = 'µ'
>>> name(micro)
'MICRO SIGN'
>>> micro_cf = micro.casefold()
>>> name(micro_cf)
'GREEK SMALL LETTER MU'
>>> micro, micro_cf
('µ', 'μ')
>>> eszett = 'ß'
>>> name(eszett)
'LATIN SMALL LETTER SHARP S'
>>> eszett_cf = eszett.casefold()
>>> eszett, eszett_cf
('ß', 'ss')
```

有将近 300 个代码点，`str.casefold()`和`str.lower()`返回不同的结果。

与 Unicode 相关的任何事物一样，大小写折叠是一个困难的问题，有许多语言上的特殊情况，但是 Python 核心团队努力提供了一个有望为大多数用户工作的解决方案。

在接下来的几节中，我们将把我们的规范化知识用于开发效用函数。

## 标准化文本匹配的效用函数

正如我们已经看到的，NFC 和 NFD 使用起来是安全的，并且允许在 Unicode 字符串之间进行合理的比较。NFC 是大多数应用程序的最佳规范化形式。`str.casefold()`是不区分大小写的比较方式。

如果你用多种语言处理文本，像示例 4-13 中的`nfc_equal`和`fold_equal`这样的函数对你的工具箱是很有用的补充。

##### 示例 4-13： normeq.py:规范化的 Unicode 字符串比较

```
"""
Utility functions for normalized Unicode string comparison.

Using Normal Form C, case sensitive:

 >>> s1 = 'café'
 >>> s2 = 'cafe\u0301'
 >>> s1 == s2
 False
 >>> nfc_equal(s1, s2)
 True
 >>> nfc_equal('A', 'a')
 False

Using Normal Form C with case folding:

 >>> s3 = 'Straße'
 >>> s4 = 'strasse'
 >>> s3 == s4
 False
 >>> nfc_equal(s3, s4)
 False
 >>> fold_equal(s3, s4)
 True
 >>> fold_equal(s1, s2)
 True
 >>> fold_equal('A', 'a')
 True

"""

from unicodedata import normalize

def nfc_equal(str1, str2):
    return normalize('NFC', str1) == normalize('NFC', str2)

def fold_equal(str1, str2):
    return (normalize('NFC', str1).casefold() ==
            normalize('NFC', str2).casefold())
```

除了 Unicode 标准化和大小写折叠——它们都是 Unicode 标准的一部分——有时应用更深层次的转换是有意义的，比如将`'café'`改为`'cafe'`。我们将在下一节中看到时间和方式。

## 极端的“正常化”:去掉音调符号

谷歌搜索秘笈涉及许多技巧，但其中一个显然是忽略了音调符号(例如，重音符号、西数字等)。)，至少在某些语境下。删除音调符号不是规范化的正确形式，因为它通常会改变单词的含义，并且在搜索时可能会产生误报。但它有助于应对生活中的一些事实:人们有时会懒惰或不知道音调符号的正确用法，拼写规则会随着时间的推移而变化，这意味着口音在生活语言中时隐时现。

除了搜索之外，去掉音调符号还有助于提高 URL 的可读性，至少在基于拉丁语的语言中是这样。看看维基百科关于圣保罗市的文章的网址:

```
https://en.wikipedia.org/wiki/S%C3%A3o_Paulo
```

`%C3%A3`部分是 URL 转义的、单个字母“a”(带颚化符的“a”)的 UTF-8 呈现。以下内容更容易识别，即使它不是正确的拼写:

```
https://en.wikipedia.org/wiki/Sao_Paulo
```

要删除`str`中的所有音调符号，可以使用类似示例 4-14 的函数。

##### 示例 4-14： simplify.py:删除所有合并标记的函数

```
importunicodedataimportstringdefshave_marks(txt):"""Remove all diacritic marks"""norm_txt=unicodedata.normalize('NFD',txt)①shaved=''.join(cforcinnorm_txtifnotunicodedata.combining(c))②returnunicodedata.normalize('NFC',shaved)③
```

① 将所有字符分解成基本字符和组合符号。

② 过滤掉所有结合标记。

③ 重组所有字符。

示例 4-15 展示了`shave_marks`的几种用法。

##### 示例 4-15：使用示例 4-14 中`shave_marks`的两个例子

```
>>> order='“Herr Voß: • ½ cup of Œtker™ caffè latte • bowl of açaí.”'>>> shave_marks(order)'“Herr Voß: • ½ cup of Œtker™ caffe latte • bowl of acai.”' ①>>> Greek='Ζέφυρος, Zéfiro'>>> shave_marks(Greek)'Ζεφυρος, Zefiro' ②
```

① 只有字母“è”、“和”被替换。

② “έ”和“é”都被替换了。

示例 4-14 中的函数`shave_marks`工作正常，但可能有些过火。通常删除音调符号的原因是为了将拉丁文文本转换为纯 ASCII，但`shave_marks`也会改变非拉丁文字符——如希腊字母——这些字符永远不会因为失去重音符号而变成 ASCII。因此，只有当基本字符是拉丁字母表中的一个字母时，分析每个基本字符并删除附加标记才有意义。这就是示例 4-16 的作用。

##### 示例 4-16：从拉丁字符中删除组合符号的功能(省略导入语句，因为这是示例 4-14 中 simplify.py 模块的一部分)

```
defshave_marks_latin(txt):"""Remove all diacritic marks from Latin base characters"""norm_txt=unicodedata.normalize('NFD',txt)①latin_base=Falsepreserve=[]forcinnorm_txt:ifunicodedata.combining(c)andlatin_base:②continue# ignore diacritic on Latin base charpreserve.append(c)③# if it isn't a combining char, it's a new base charifnotunicodedata.combining(c):④latin_base=cinstring.ascii_lettersshaved=''.join(preserve)returnunicodedata.normalize('NFC',shaved)⑤
```

① 将所有字符分解成基本字符和组合符号。

② 当基本字符是拉丁文时，跳过组合符号。

③ 否则，保持当前角色。

④ 检测新的基本字符，并确定它是否是拉丁语。

⑤ 重组所有字符。

一个更激进的步骤是替换西方文本中常见的符号(例如，弯引号、长破折号、项目符号等)。)转化为`ASCII`当量。这就是示例 4-17 中函数`asciize`的作用。

##### 示例 4-17：将一些西方印刷符号转换成 ASCII 码(这个片段也是来自示例 4-14 的 simplify.py 的一部分)

```
single_map=str.maketrans("""‚ƒ„ˆ‹‘’“”•–—˜›""",①"""'f"^<''""---~>""")multi_map=str.maketrans({②'€':'EUR','…':'...','Æ':'AE','æ':'ae','Œ':'OE','œ':'oe','™':'(TM)','‰':'<per mille>','†':'**','‡':'***',})multi_map.update(single_map)③defdewinize(txt):"""Replace Win1252 symbols with ASCII chars or sequences"""returntxt.translate(multi_map)④defasciize(txt):no_marks=shave_marks_latin(dewinize(txt))⑤no_marks=no_marks.replace('ß','ss')⑥returnunicodedata.normalize('NFKC',no_marks)⑦
```

① 为字符到字符的替换建立映射表。

② 构建字符到字符串替换的映射表。

③ 合并映射表。

④ `dewinize`不影响`ASCII`或`latin1`文本，只影响`cp1252`中`latin1`的微软附加物。

⑤ 使用`dewinize`并去除变音符号。

⑥ 将 Eszett 替换为“ss”(我们在这里没有使用 case fold，因为我们希望保留案例)。

⑦ 应用 NFKC 规范化将字符与其兼容码位组合在一起。

示例 4-18 为使用中的`asciize`。

##### 示例 4-18：使用中`asciize`的两个示例示例 4-17

```
>>> order='“Herr Voß: • ½ cup of Œtker™ caffè latte • bowl of açaí.”'>>> dewinize(order)'"Herr Voß: - ½ cup of OEtker(TM) caffè latte - bowl of açaí."' ①>>> asciize(order)'"Herr Voss: - 1⁄2 cup of OEtker(TM) caffe latte - bowl of acai."' ②
```

① `dewinize`替换花引号、项目符号和(商标符号)。

② `asciize`应用`dewinize`，删除音调符号，替换`'ß'`。

###### 警告

不同的语言有自己的删除音调符号的规则。比如德国人把`'ü'`改成`'ue'`。我们的`asciize`功能没有那么精致，所以它可能适合也可能不适合您的语言。不过，它对葡萄牙语来说还可以接受。

总的来说， *simplify.py* 中的函数超越了标准规范化，对文本进行了深度处理，很有可能改变其含义。只有你能决定是否走这么远，知道目标语言，你的用户，以及转换后的文本将如何使用。

这就结束了我们对 Unicode 文本规范化的讨论。

现在我们来梳理一下 Unicode 排序。T32

# 排序 Unicode 文本

Python 通过逐个比较每个序列中的项目，对任何类型的序列进行排序。对于字符串，这意味着比较代码点。不幸的是，这对任何使用非 ASCII 字符的人来说都会产生无法接受的结果。

考虑对巴西种植的水果进行排序:

```
>>> fruits = ['caju', 'atemoia', 'cajá', 'açaí', 'acerola']
>>> sorted(fruits)
['acerola', 'atemoia', 'açaí', 'caju', 'cajá']
```

不同的地区有不同的排序规则，但是在葡萄牙语和许多使用拉丁字母的语言中，重音和西数字在排序时几乎没有区别。 [8] 所以“cajá”被排序为“caja”，必须在“caju”之前

排序后的`fruits`列表应该是:

```
['açaí', 'acerola', 'atemoia', 'cajá', 'caju']
```

在 Python 中对非 ASCII 文本进行排序的标准方法是使用`locale.strxfrm`函数，根据 [`locale`模块文档](https://fpy.li/4-16)，该函数“将一个字符串转换成一个可用于区域设置比较的字符串”

要启用`locale.strxfrm`，您必须首先为您的应用程序设置一个合适的语言环境，并祈祷操作系统支持它。示例 4-19 中的命令序列可能对你有用。

##### 示例 4-19： *locale_sort.py* :使用`locale.strxfrm`函数作为排序键

```
import locale
my_locale = locale.setlocale(locale.LC_COLLATE, 'pt_BR.UTF-8')
print(my_locale)
fruits = ['caju', 'atemoia', 'cajá', 'açaí', 'acerola']
sorted_fruits = sorted(fruits, key=locale.strxfrm)
print(sorted_fruits)
```

在安装了`pt_BR.UTF-8`语言环境的 GNU/Linux (Ubuntu 19.10)上运行示例 4-19 ，我得到了正确的结果:

```
'pt_BR.UTF-8'
['açaí', 'acerola', 'atemoia', 'cajá', 'caju']
```

所以需要在排序的时候使用 `locale.strxfrm` 作为键之前调用`setlocale(LC_COLLATE, «your_locale»)`。

不过，有一些注意事项:

*   因为区域设置是全局的，所以不推荐在库中调用`setlocale`。您的应用程序或框架应该在进程开始时设置区域设置，并且不应该在之后更改它。

*   该语言环境必须安装在操作系统上，否则`setlocale`会引发`locale.Error: unsupported locale setting`异常。

*   你必须知道如何拼写地区名称。

*   语言环境必须由操作系统的制造商正确实现。我在 Ubuntu 19.10 上成功了，在 macOS 10.14 上就不行了。在 macOS 上，调用`setlocale(LC_COLLATE, 'pt_BR.UTF-8')`返回没有抱怨的字符串`'pt_BR.UTF-8'`。但是`sorted(fruits, key=locale.strxfrm)`产生了与`sorted(fruits)`相同的错误结果。我也尝试了 macOS 上的`fr_FR`、`es_ES`和`de_DE`语言环境，但是`locale.strxfrm`从来没有成功过。 [9]

因此，国际化排序的标准库解决方案是可行的，但似乎只在 GNU/Linux 上得到很好的支持(如果您是专家，可能在 Windows 上也是如此)。即使这样，它也依赖于地区设置，造成部署问题。

幸运的是，有一个更简单的解决方案:在 *PyPI* 上提供的 *pyuca* 库。

## 使用 Unicode 排序算法进行排序

詹姆斯·陶贝尔，多产的 Django 撰稿人，一定感受到了这种痛苦并创造了*py uca*](https://fpy.li/4-17)，一个 Unicode 排序算法(UCA)的纯 Python 实现。[示例 4-20 展示了它的易用性。

##### 示例 4-20：使用`pyuca.Collator.sort_key`方法

```
>>> import pyuca
>>> coll = pyuca.Collator()
>>> fruits = ['caju', 'atemoia', 'cajá', 'açaí', 'acerola']
>>> sorted_fruits = sorted(fruits, key=coll.sort_key)
>>> sorted_fruits
['açaí', 'acerola', 'atemoia', 'cajá', 'caju']
```

这很简单，可以在 GNU/Linux、macOS 和 Windows 上运行，至少在我的小样本中是这样。

`pyuca`不考虑语言环境。如果需要定制排序，可以向`Collator()`构造函数提供定制排序表的路径。开箱即用的是 [*allkeys.txt*](https://fpy.li/4-18) ，与项目捆绑。那只是从*中复制的[默认 Unicode 归类元素表。](https://fpy.li/4-19)*

 *# PyICU: Miro 对 Unicode 排序的推荐

(技术评论员 Miroslav ediv 精通多种语言，是 Unicode 方面的专家。这是他写的关于丘卡的文章。)

*pyuca* 有一种排序算法不考虑个别语言的排序顺序。例如，德语中的δ在 A 和 B 之间，而瑞典语中的δ在 z 之后。看看 [PyICU](https://fpy.li/4-20) 吧，它像 locale 一样工作，而不改变进程的语言环境。如果您想更改土耳其语中 ii̇/ıi 的大小写，这也是必需的。PyICU 包含一个必须编译的扩展，所以在某些系统中可能比 *pyuca* 更难安装，pyuca 只是 Python。

顺便说一下，这个校对表是构成 Unicode 数据库(我们的下一个主题)的许多数据文件之一。*  *# Unicode 数据库

Unicode 标准提供了一个完整的数据库——以几个结构化文本文件的形式——不仅包括将代码点映射到字符名称的表，还包括关于单个字符以及它们如何相关的元数据。例如，Unicode 数据库记录字符是可打印的、字母、十进制数字还是其他数字符号。这就是`str`方法`isalpha`、`isprintable`、`isdecimal`和`isnumeric`的工作方式。`str.casefold`还使用 Unicode 表中的信息。

###### 注意

`unicodedata.category(char)`函数从 Unicode 数据库返回两个字母的类别`char`。更高级的`str`方法更容易使用。例如，如果`label`中的每个字符都属于这些类别之一:`Lm`、`Lt`、`Lu`、`Ll`或`Lo`，则 [`label.isalpha()`](https://fpy.li/4-21) 返回`True`。要了解这些代码的含义，请参见英文维基百科的[“Unicode 字符属性”文章](https://fpy.li/4-23)中的[“一般类别”](https://fpy.li/4-22)。

## 按名称查找字符

`unicodedata`模块具有检索角色元数据的功能，其中`unicodedata.name()`返回角色在标准中的正式名称。图 4-5 展示了该功能。 [10]

![Exploring unicodedata.name in the Python console](Images/flpy_0405.png)

###### 图 4-5。探索 Python 控制台中的`unicodedata.name()`。

你可以使用`name()`功能构建应用程序，让用户通过名字搜索角色。图 4-6 展示了 *cf.py* 命令行脚本，该脚本将一个或多个单词作为参数，并列出官方 Unicode 名称中包含这些单词的字符。 *cf.py* 的完整源代码在示例 4-21 中。

![Using cf.py to find smiling cats.](Images/flpy_0406.png)

###### 图 4-6。使用 *cf.py* 寻找微笑的猫。

###### 警告

表情符号的支持因操作系统和应用程序的不同而大相径庭。近年来，macOS 终端为表情符号提供了最好的支持，其次是现代 GNU/Linux 图形终端。windows*cmd.exe*和 PowerShell 现在支持 Unicode 输出，但是当我在 2020 年 1 月写这一节时，它们仍然不显示表情符号——至少不“开箱即用”科技评论家莱昂纳多·罗塞尔告诉我微软的一款新的开源 [Windows 终端，它可能比旧的微软控制台有更好的 Unicode 支持。我没有时间去尝试。](https://fpy.li/4-24)

在示例 4-21 中，注意`find`函数中的`if`语句，使用`.issubset()`方法快速测试`query`集合中的所有单词是否出现在根据角色名字构建的单词列表中。感谢 Python 丰富的 set API，我们不需要嵌套的`for`循环和另一个`if`来实现这个检查。

##### 示例 4-21： cf.py:字符查找工具

```
#!/usr/bin/env python3importsysimportunicodedataSTART,END=ord(''),sys.maxunicode+1①deffind(*query_words,start=START,end=END):②query={w.upper()forwinquery_words}③forcodeinrange(start,end):char=chr(code)④name=unicodedata.name(char,None)⑤ifnameandquery.issubset(name.split()):⑥print(f'U+{code:04X}\t{char}\t{name}')⑦defmain(words):ifwords:find(*words)else:print('Please provide words to find.')if__name__=='__main__':main(sys.argv[1:])
```

① 为要搜索的代码点范围设置默认值。

② `find`接受`query_words`和可选的仅关键字参数来限制搜索范围，以便于测试。

③ 将`query_words`转换成一组大写字符串。

④ 获取`code`的 Unicode 字符。

⑤ 获取字符的名称，如果代码点未赋值，则为`None`。

⑥ 如果有一个名字，将它分割成一个单词列表，然后检查`query`集合是否是该列表的子集。

⑦ 打印出带有`U+9999`格式的码位、字符及其名称的行。

`unicodedata`模块还有其他有趣的功能。接下来，我们将看到一些与从具有数字意义的字符中获取信息相关的内容。

## 字符的数字含义

`unicodedata`模块包括检查一个 Unicode 字符是否代表一个数字的函数，如果是，它代表人类的数值——与它的码位号相对。示例 4-22 展示了`unicodedata.name()`和`unicodedata.numeric()`的用法，以及`str`的`.isdecimal()`和`.isnumeric()`方法。

##### 示例 4-22：Unicode 数据库数字字符元数据的演示(标注描述输出中的每一列)

```
importunicodedataimportrere_digit=re.compile(r'\d')sample='1\xbc\xb2\u0969\u136b\u216b\u2466\u2480\u3285'forcharinsample:print(f'U+{ord(char):04x}',①char.center(6),②'re_dig'ifre_digit.match(char)else'-',③'isdig'ifchar.isdigit()else'-',④'isnum'ifchar.isnumeric()else'-',⑤f'{unicodedata.numeric(char):5.2f}',⑥unicodedata.name(char),⑦sep='\t')
```

① `U+0000`格式的代码点。

② 字符集中在长度为 6 的`str`中。

③ 如果字符匹配`r'\d'`正则表达式，则显示`re_dig`。

④ 如果`char.isdigit()`是`True`则显示`isdig`。

⑤ 如果`char.isnumeric()`为`True`则显示`isnum`。

⑥ 用宽度 5 和 2 个小数位格式化的数值。

⑦ Unicode 字符名称。

运行示例 4-22 给你图 4-7 ，如果你的终端字体有所有这些字形。

![Numeric characters screenshot](Images/flpy_0407.png)

###### 图 4-7。显示数字字符及其元数据的 macOS 终端；`re_dig`表示字符匹配正则表达式`r'\d'`。

图 4-7 第六列是对角色调用`unicodedata.numeric(char)`的结果。这表明 Unicode 知道表示数字的符号的数值。因此，如果您想创建一个支持泰米尔数字或罗马数字的电子表格应用程序，就去做吧！

图 4-7 显示正则表达式`r'\d'`匹配数字“1”和梵文数字 3，但不匹配其他一些被`isdigit`函数认为是数字的字符。`re`模块对 Unicode 并不了解。PyPI 上新的`regex`模块旨在最终取代`re`，并提供更好的 Unicode 支持。 [11] 我们将在下一节回到`re`模块。

在本章中，我们已经使用了几个`unicodedata`函数，但是还有很多我们没有涉及到。参见 [`unicodedata`模块](https://fpy.li/4-25)的标准库文档。

接下来，我们将快速查看双模 API，这些 API 提供了接受`str`或`bytes`参数的函数，并根据类型进行特殊处理。

# 双模式字符串和字节 API

Python 的标准库 具有接受`str`或`bytes`参数的函数，并且根据类型表现不同。一些例子可以在`re`和`os`和模块中找到。

## 正则表达式中字符串与字节

如果用`bytes`构建正则表达式，那么`\d`、`\w`等模式只匹配 ASCII 字符；相反，如果这些模式被给定为`str`，它们匹配 ASCII 以外的 Unicode 数字或字母。示例 4-23 和图 4-8 比较`str`和`bytes`模式如何匹配字母、ASCII 数字、上标和泰米尔数字。

##### 示例 4-23： ramanujan.py:比较简单的`str`和`bytes`正则表达式的行为

```
importrere_numbers_str=re.compile(r'\d+')①re_words_str=re.compile(r'\w+')re_numbers_bytes=re.compile(rb'\d+')②re_words_bytes=re.compile(rb'\w+')text_str=("Ramanujan saw \u0be7\u0bed\u0be8\u0bef"③" as 1729 = 1³ + 12³ = 9³ + 10³.")④text_bytes=text_str.encode('utf_8')⑤print(f'Text\n {text_str!r}')print('Numbers')print(' str  :',re_numbers_str.findall(text_str))⑥print(' bytes:',re_numbers_bytes.findall(text_bytes))⑦print('Words')print(' str  :',re_words_str.findall(text_str))⑧print(' bytes:',re_words_bytes.findall(text_bytes))⑨
```

① 前两个正则表达式属于`str`类型。

② 后两种是`bytes`型。

③ 要搜索的 Unicode 文本，包含用于`1729`的泰米尔数字(逻辑行一直延续到右括号标记)。

④ 该字符串在编译时与前一个连接在一起(参见[2 . 4 . 2)。*中的字符串文字串联*](https://fpy.li/4-26)(Python 语言参考)。

⑤ 使用`bytes`正则表达式进行搜索需要一个`bytes`字符串。

⑥ `str`模式`r'\d+'`匹配泰米尔语和 ASCII 数字。

⑦ `bytes`模式`rb'\d+'`只匹配数字的 ASCII 字节。

⑧ `str`模式`r'\w+'`匹配字母、上标、泰米尔语和 ASCII 数字。

⑨ `bytes`模式`rb'\w+'`只匹配字母和数字的 ASCII 字节。

![Output of ramanujan.py](Images/flpy_0408.png)

###### 图 4-8。运行示例 4-23 中 ramanujan.py 的截图。

例子 4-23 是一个简单的例子来说明一点:你可以在`str`和`bytes`上使用正则表达式，但是在第二种情况下，ASCII 范围之外的字节被视为非数字和非单词字符。

对于`str`正则表达式，有一个`re.ASCII`标志使得`\w`、`\W`、`\b`、`\B`、`\d`、`\D`、`\s`、`\S`进行纯 ASCII 匹配。详见`re`模块的[文档。](https://fpy.li/4-27)

另一个重要的双模模块是`os`。

## 操作系统函数中的字符串与字节

GNU/Linux 内核不理解 Unicode，所以在现实世界中，你可能会发现由字节序列组成的文件名在任何合理的编码方案中都是无效的，并且不能被解码成`str`。客户端使用各种操作系统的文件服务器特别容易出现这个问题。

为了解决这个问题，所有接受文件名或路径名的`os`模块函数都将参数作为`str`或`bytes`。如果使用`str`参数调用一个这样的函数，该参数将使用`sys.getfilesystemencoding()`命名的编解码器自动转换，操作系统响应将使用相同的编解码器解码。这几乎总是您想要的，符合 Unicode 三明治的最佳实践。

但是如果您必须处理(也许是修复)不能以那种方式处理的文件名，您可以将`bytes`参数传递给`os`函数以获得`bytes`返回值。这个特性允许你处理任何文件或路径名，不管你能找到多少小精灵。参见示例 4-24 。

##### 示例 4-24： `listdir`同`str`和`bytes`的争论和结果

```
>>> os.listdir('.')①['abc.txt', 'digits-of-π.txt'] >>> os.listdir(b'.')②[b'abc.txt', b'digits-of-\xcf\x80.txt']
```

① 第二个文件名是“π的位数”。txt”(带希腊字母 pi)。

② 给定一个`byte`参数，`listdir`返回字节形式的文件名:`b'\xcf\x80'`是希腊字母 pi 的 UTF-8 编码。

为了帮助手动处理文件名或路径名的`str`或`bytes`序列，`os`模块提供了特殊的编码和解码功能`os.fsencode(name_or_path)`和`os.fsdecode(name_or_path)`。从 Python 3.6 开始，这两个函数都接受类型为`str`、`bytes`的参数或实现`os.PathLike`接口的对象。

Unicode 是一个很深的兔子洞。是时候结束我们对`str`和`bytes`的探索了。

# 章节摘要

我们在这一章的开始就摒弃了`1 character == 1 byte`的观点。随着世界采用 Unicode，我们需要将文本字符串的概念与在文件中表示它们的二进制序列分开，Python 3 强制实现了这种分离。

在简要概述了二进制序列数据类型— `bytes`、`bytearray`和、`memoryview`、—之后，我们开始编码和解码，对重要的编解码器进行了抽样，然后介绍了防止或处理由 Python 源文件中错误编码导致的臭名昭著的`UnicodeEncodeError`、`UnicodeDecodeError`和`SyntaxError`的方法。

然后，我们考虑了在没有元数据的情况下编码检测的理论和实践:理论上，这是不可能的，但实际上 Chardet 包很好地完成了许多流行的编码。然后，字节顺序标记作为唯一的编码提示出现在 UTF-16 和 UTF-32 文件中，有时也出现在 UTF-8 文件中。

在下一节中，我们演示了打开文本文件，这是一个简单的任务，除了一个缺陷:当您打开文本文件时，`encoding=`关键字参数不是必需的，但它应该是必需的。如果您未能指定编码，那么由于默认编码的冲突，您最终会得到一个设法生成跨平台不兼容的“纯文本”的程序。然后我们展示了 Python 默认使用的不同编码设置，以及如何检测它们。对于 Windows 用户来说，一个悲哀的现实是，在同一台机器中，这些设置通常有不同的值，并且这些值是互不兼容的；相比之下，GNU/Linux 和 macOS 用户生活在一个更快乐的地方，UTF-8 几乎是所有地方的默认设置。

Unicode 提供了多种表示某些字符的方式，因此规范化是文本匹配的先决条件。除了解释规范化和大小写折叠之外，我们还提供了一些实用函数，您可以根据自己的需要进行调整，包括彻底的转换，比如删除所有重音符号。然后我们看到了如何通过利用标准的`locale`模块——带有一些警告——和一个不依赖于复杂的地区配置的替代方案——外部的 *pyuca* 包来正确地对 Unicode 文本进行排序。

我们利用 Unicode 数据库编写了一个命令行实用程序来按名称搜索字符——由于 Python 的强大功能，只需 28 行代码。我们浏览了其他 Unicode 元数据，并简要概述了双模式 API，其中一些函数可以用`str`或`bytes`参数调用，产生不同的结果。

# 进一步阅读

Ned Batchelder 的 2012 PyCon US talk [“实用的 Unicode，或者说，我如何停止痛苦？”](https://fpy.li/4-28)出类拔萃。Ned 非常专业，他提供了一份完整的谈话记录以及幻灯片和视频。

“Python 中的字符编码和 Unicode:如何(╯ □有尊严的)╯︵┻━┻)([幻灯片](https://fpy.li/4-1)，[视频](https://fpy.li/4-2))是 Esther Nam 和 Travis Fischer 在 PyCon 2014 上的精彩演讲，我在那里找到了本章的精辟格言:“人类使用文本。电脑讲字节。”

Lennart Regebro——本书第一版的技术评审之一——在短文[“解开 Unicode:什么是 Unicode？”中分享了他的“有用的 Unicode 心智模型(UMMU)”](https://fpy.li/4-31)。Unicode 是一个复杂的标准，所以 Lennart 的 UMMU 是一个非常有用的起点。

Python 文档中的官方[“Unicode how to”](https://fpy.li/4-32)从几个不同的角度探讨了这个主题，从一个很好的历史介绍，到语法细节、编解码器、正则表达式、文件名和支持 Unicode 的 I/O(即 Unicode 三明治)的最佳实践，每一节都有大量额外的参考链接。[Mark Pilgrim 的巨著](https://fpy.li/4-33)[*Dive into Python 3*](https://fpy.li/4-34)(a press)的第 4 章“Strings”也非常好地介绍了 Python 3 中的 Unicode 支持。在同一本书中，[第 15 章](https://fpy.li/4-35)描述了 Chardet 库是如何从 Python 2 移植到 Python 3 的，这是一个很有价值的案例研究，因为从旧的`str`到新的`bytes`的切换是大多数迁移问题的原因，这也是设计用于检测编码的库的核心问题。

如果你知道 Python 2，但不熟悉 Python 3，吉多·范·罗苏姆的[“Python 3.0 中的新特性”](https://fpy.li/4-36)有 15 个要点总结了哪些变化，并提供了大量链接。Guido 以直言不讳的陈述开始:“你所认为的关于二进制数据和 Unicode 的一切都变了。”阿明·罗纳切尔的博客文章[“Python 上 Unicode 的更新指南”](https://fpy.li/4-37)很有深度，强调了Python 3 中 Unicode 的一些缺陷(阿明不是 Python 3 的忠实粉丝)。

第 3 版 [*Python 指南*的第 2 章，“字符串和文本”。由 David Beazley 和 Brian K. Jones 编写的](https://fpy.li/pycook3) (O'Reilly)有几个处理 Unicode 规范化、净化文本和对字节序列执行面向文本的操作的方法。第 5 章介绍了文件和 I/O，其中包括“配方 5.17”。将字节写入文本文件”，表明在任何文本文件的基础上，总有一个可以在需要时直接访问的二进制流。在本书的后面,`struct`模块将在“食谱 6.11”中使用。读写结构的二进制数组。

Nick Coghlan 的“Python Notes”博客有两篇帖子与本章非常相关:[“Python 3 和 ASCII 兼容的二进制协议”](https://fpy.li/4-38)和[“用 Python 3 处理文本文件”](https://fpy.li/4-39)。强烈推荐。

Python 支持的编码列表可在`codecs`模块文档的[“标准编码”](https://fpy.li/4-40)中找到。如果您需要以编程方式获取该列表，请查看 CPython 源代码附带的[*/Tools/unicode/list codecs . py*](https://fpy.li/4-41)脚本中是如何完成的。

Jukka K. Korpela (O'Reilly)的《T2》[Unicode Explained](https://fpy.li/4-42)和 Richard Gillam (Addison-Wesley)的《Unicode 揭秘 》这两本书不是专门针对 Python 的，但对我学习 Unicode 概念很有帮助。Victor Stinner 的 [*用 Unicode*](https://fpy.li/4-44) 编程是一本免费的自助出版的书(Creative Commons BY-SA ),它涵盖了一般的 Unicode，以及主要操作系统和一些编程语言(包括 Python)环境中的工具和 API。

W3C 页面[“Case Folding:a Introduction”](https://fpy.li/4-45)和[“Character Model for the World Wide Web:String Matching”](https://fpy.li/4-15)涵盖了标准化概念，前者是一个温和的介绍，后者是一个用标准语言编写的工作组说明，与[“Unicode Standard annual Forms # 15—Unicode Normalization Forms”](https://fpy.li/4-47)的语气相同。来自[](https://fpy.li/4-49)*Unicode.org 的[“常见问题，规范化”](https://fpy.li/4-48)部分更具可读性，马克·戴维斯的[“NFC 常见问题”](https://fpy.li/4-50)也是如此，他是多种 Unicode 算法的作者，也是撰写本文时 Unicode Consortium 的主席。*

 *2016 年，纽约现代艺术博物馆(MoMA)的收藏了最初的表情符号，这是 1999 年栗田茂隆为日本移动运营商 NTT DOCOMO 设计的 176 个表情符号。追溯到更早的历史， *表情百科*](https://fpy.li/4-52) 发表了[“纠正第一个表情集的记录”](https://fpy.li/4-53)，将日本软银称为已知最早的表情集，于 1997 年部署在手机中。软银的 set 是现在 Unicode 中 90 个表情符号的来源，包括 U+1F4A9 ( `PILE OF POO`)。马修·罗森伯格的[*emojitracker.com*](https://fpy.li/4-54)是一个实时仪表盘，显示推特上表情符号的使用次数，并实时更新。当我写这篇文章时，`FACE WITH TEARS OF JOY` (U+1F602) 是 Twitter 上最受欢迎的表情符号，记录了超过 3313667315 次事件。*  *^([1)PyCon 2014 talk 的第 12 张幻灯片“Python 中的字符编码和 Unicode”([幻灯片](https://fpy.li/4-1)、[视频](https://fpy.li/4-2))。

[2] Python 2.6 和 2.7 也有`bytes`，但只是`str`类型的别名。

[3] 花絮:Python 默认使用的作为字符串分隔符的 ASCII“单引号”字符，实际上在 Unicode 标准中被命名为撇号。真正的单引号是不对称的:左为 U+2018，右为 U+2019 。

[4] 在 Python 3.0 到 3.4 中就不行了，给处理二进制数据的开发者造成了很大的痛苦。此反转记录在 [PEP 461 中——将%格式添加到字节和字节数组](https://fpy.li/pep461)。

[5] 我第一次看到“Unicode 三明治”这个术语是在美国 PyCon 2012 上 Ned Batchelder 的精彩[“实用主义 Unicode”演讲](https://fpy.li/4-10)中。

[6] 来源:[“Windows 命令行:Unicode 和 UTF-8 输出文本缓冲区”](https://fpy.li/4-11)。

奇怪的是，微型符号被认为是“兼容字符”，但欧姆符号却不是。最终结果是，NFC 没有触及微符号，而是将欧姆符号更改为大写的ω，而 NFKC 和 NFKD 将欧姆和微都更改为希腊字符。

[8] 音调符号仅在极少数情况下影响排序，因为它们是两个单词之间的唯一区别，在这种情况下，带有音调符号的单词排在普通单词之后。

[9] 再次，我找不到解决方案，但确实找到了其他人报告的相同问题。亚历克斯·马尔泰利(Alex Martelli)是技术评测人员之一，他在装有 macOS 10.9 的麦金塔电脑上使用`setlocale`和`locale.strxfrm`没有任何问题。总之:你的里程可能会有所不同。

这是一张图片——不是代码清单——因为在我写这篇文章的时候，表情符号还没有得到 O'Reilly 的数字出版工具链的很好支持。

[11] 虽然在识别这个特殊样本中的数字方面并不比`re`好。**