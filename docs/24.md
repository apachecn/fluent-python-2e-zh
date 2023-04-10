<link href="Styles/Style00.css" rel="stylesheet" type="text/css"> <link href="Styles/Style01.css" rel="stylesheet" type="text/css"> 

# 第二十章。同时执行人

> 抨击线程的人通常是系统程序员，他们头脑中有典型的应用程序程序员一生中从未遇到过的用例。[...]在应用程序程序员可能遇到的 99%的用例中，产生一堆独立线程并在队列中收集结果的简单模式是人们需要知道的一切。
> 
> Michele Simionato，Python 深度思考者 [1]

这一章关注的是`concurrent.futures.Executor`类，它们封装了 Michele Simionato 描述的“生成一堆独立线程并将结果收集到一个队列中”的模式。并发执行器使得这种模式使用起来几乎微不足道，不仅对于线程，对于进程也是如此——对于计算密集型任务非常有用。

在这里我还引入了 *futures* 的概念——表示操作异步执行的对象，类似于 JavaScript 承诺。这个原始的想法不仅是`concurrent.futures`的基础，也是`asyncio`包的基础，这是第 21 章的主题。

# 本章的新内容

我将这一章从“与未来的并发性”重命名为“并发执行器”，因为执行器是这里涉及的最重要的高级特性。期货是低级对象，重点在“期货在哪里？”，但在其余章节中大多看不见。

所有的 HTTP 客户端示例现在都使用新的 [*HTTPX*](https://fpy.li/httpx) 库，它提供了同步和异步 API。

由于在 Python 3.7 的 `http.server`](https://fpy.li/20-2) 包中添加了多线程服务器，现在[“带进度显示和错误处理的下载”中的实验设置更简单了。之前标准库只有单线程的`BaseHttpServer`，对于并发客户端的实验毫无益处，所以在本书第一版中我不得不求助于外部工具。

“使用 concurrent.futures 启动流程”现在演示了执行器如何简化我们在“多核素数检查器代码”中看到的代码。

最后，我把大部分理论移到了新的第 19 章，“Python 中的并发模型”。

# 并发 Web 下载

并发性 对于高效的网络 I/O 来说是必不可少的:应用程序应该做些别的事情直到响应回来，而不是无所事事地等待远程机器。 [2]

为了用代码演示，我编写了三个简单的程序从网上下载 20 个国家的国旗图片。第一个是 *flags.py* ，它按顺序运行:只有当前一个图像被下载并保存到本地时，它才会请求下一个图像。另外两个脚本进行并发下载:它们几乎同时请求几个图像，并在它们到达时保存它们。 *flags_threadpool.py* 脚本使用`concurrent.futures`包，而 *flags_asyncio.py* 使用`asyncio`。

示例 20-1 显示了运行三个脚本的结果，每个脚本运行三次。我还在 YouTube 上发布了一个 73s 视频，这样你就可以在 macOS Finder 窗口显示保存的标志时观看它们的运行。脚本正在从 CDN 后面的 fluentpython.com 下载图像，所以你可能会在第一次运行时看到较慢的结果。](https://fpy.li/20-3)[示例 20-1 中的结果是多次运行后得到的，所以 CDN 缓存是热的。

##### 示例 20-1：脚本 flags.py、flags_threadpool.py 和 flags_asyncio.py 的三次典型运行

```
$ python3 flags.py
BD BR CD CN DE EG ET FR ID IN IR JP MX NG PH PK RU TR US VN ① 20 flags downloaded in 7.26s ② $ python3 flags.py
BD BR CD CN DE EG ET FR ID IN IR JP MX NG PH PK RU TR US VN
20 flags downloaded in 7.20s
$ python3 flags.py
BD BR CD CN DE EG ET FR ID IN IR JP MX NG PH PK RU TR US VN
20 flags downloaded in 7.09s
$ python3 flags_threadpool.py
DE BD CN JP ID EG NG BR RU CD IR MX US PH FR PK VN IN ET TR
20 flags downloaded in 1.37s ③ $ python3 flags_threadpool.py
EG BR FR IN BD JP DE RU PK PH CD MX ID US NG TR CN VN ET IR
20 flags downloaded in 1.60s
$ python3 flags_threadpool.py
BD DE EG CN ID RU IN VN ET MX FR CD NG US JP TR PK BR IR PH
20 flags downloaded in 1.22s
$ python3 flags_asyncio.py ④ BD BR IN ID TR DE CN US IR PK PH FR RU NG VN ET MX EG JP CD
20 flags downloaded in 1.36s
$ python3 flags_asyncio.py
RU CN BR IN FR BD TR EG VN IR PH CD ET ID NG DE JP PK MX US
20 flags downloaded in 1.27s
$ python3 flags_asyncio.py
RU IN ID DE BR VN PK MX US IR ET EG NG BD FR CN JP PH CD TR ⑤ 20 flags downloaded in 1.42s
```

① 每次运行的输出以下载时的国旗国家代码开始，以一条说明运行时间的消息结束。

② 下载 20 张图片平均需要 7.18 秒。

③ *flags_threadpool.py* 的平均值为 1.40 秒

④ 对于 *flags_asyncio.py* ，平均时间为 1.35 秒。

⑤ 注意国家代码的顺序:对于并发脚本，每次下载的顺序都不同。

并发脚本之间的性能差异并不显著，但它们都比顺序脚本快五倍以上——而且这只是针对下载 20 个文件(每个文件几千字节)的小任务。如果将任务扩展到数百次下载，并发脚本可能会比顺序代码快 20 倍甚至更多。

###### 警告

在对公共 web 服务器测试并发 HTTP 客户端时，您可能会无意中发起拒绝服务(DoS)攻击，或者被怀疑这样做。在示例 20-1 的情况下，这样做是可以的，因为那些脚本被硬编码为只发出 20 个请求。在本章的后面，我们将使用 Python 的`http.server`包来运行测试。

现在让我们研究一下示例 20-1 中测试的两个脚本的实现: *flags.py* 和 *flags_threadpool.py* 。我将把第三个脚本 *flags_asyncio.py* 留给第 21 章中的，但是我想一起演示这三个脚本来说明两点:

1.  无论您使用哪种并发结构——线程或协程——如果您正确编码，您都会发现在网络 I/O 操作中，与顺序代码相比，吞吐量大大提高。

2.  对于可以控制发出多少请求的 HTTP 客户端，线程和协程之间的性能没有显著差异。 [3]

继续编码。

## 顺序下载脚本

示例 20-2 包含对 *flags.py* 的实现，我们在示例 20-1 中运行的第一个脚本。它不是很有趣，但是我们将重用它的大部分代码和设置来实现并发脚本，所以它值得一些关注。

###### 注意

为了清楚起见，在示例 20-2 中没有错误处理。我们将在后面处理异常，但这里我想把重点放在代码的基本结构上，以便更容易将这个脚本与并发脚本进行对比。

##### 示例 20-2： flags.py:顺序下载脚本；一些函数将被其他脚本重用

```
importtimefrompathlibimportPathfromtypingimportCallableimporthttpx①POP20_CC=('CN IN US ID BR PK NG BD RU JP ''MX PH VN ET EG DE IR TR CD FR').split()②BASE_URL='https://www.fluentpython.com/data/flags'③DEST_DIR=Path('downloaded')④defsave_flag(img:bytes,filename:str)->None:⑤(DEST_DIR/filename).write_bytes(img)defget_flag(cc:str)->bytes:⑥url=f'{BASE_URL}/{cc}/{cc}.gif'.lower()resp=httpx.get(url,timeout=6.1,⑦follow_redirects=True)⑧resp.raise_for_status()⑨returnresp.contentdefdownload_many(cc_list:list[str])->int:⑩forccinsorted(cc_list):⑪image=get_flag(cc)save_flag(image,f'{cc}.gif')print(cc,end='',flush=True)⑫returnlen(cc_list)defmain(downloader:Callable[[list[str]],int])->None:⑬DEST_DIR.mkdir(exist_ok=True)⑭t0=time.perf_counter()⑮count=downloader(POP20_CC)elapsed=time.perf_counter()-t0print(f'\n{count} downloads in {elapsed:.2f}s')if__name__=='__main__':main(download_many)⑯
```

① 导入`httpx`库。它不是标准库的一部分，所以按照惯例，导入是在标准库模块和空行之后进行的。

② 按人口递减顺序排列的 20 个人口最多的国家的 ISO 3166 国家代码列表。

③ 带有标志图像的目录。[4]

④ 保存图像的本地目录。

⑤ 将`img`字节保存到`DEST_DIR`中的`filename`。

⑥ 给定一个国家代码，构建 URL 并下载图像，返回响应的二进制内容。

⑦ 给网络操作添加一个合理的超时是一个很好的做法，可以避免无缘无故的阻塞几分钟。

⑧ 默认情况下， *HTTPX* 不跟随重定向。 [5]

⑨ 这个脚本中没有错误处理，但是如果 HTTP 状态不在 2XX 范围内，这个方法会引发一个异常——强烈建议避免无声的失败。

⑩ `download_many`是与并发实现进行比较的关键函数。

⑪ 按照字母顺序遍历国家代码列表，这样可以很容易地看到输出中保留了顺序；返回下载的国家代码的数量。

⑫ 在同一行中一次显示一个国家代码，这样我们可以看到每次下载的进度。`end=' '`参数替换了通常在每行末尾用空格字符打印的换行符，因此所有国家代码都在同一行中渐进显示。需要`flush=True`参数，因为默认情况下，Python 输出是行缓冲的，这意味着 Python 只在换行符后显示打印字符。

⑬ `main`必须调用将进行下载的函数；这样，我们可以在`threadpool`和`ascyncio`示例中使用`main`作为其他`download_many`实现的库函数。

⑭ 如果需要，创建`DEST_DIR`；如果目录存在，不要引发错误。

⑮ 运行`downloader`功能后，记录并报告经过的时间。

⑯ 用`download_many`函数调用`main`。

###### 小费

[*HTTPX*](https://fpy.li/httpx) 库的灵感来自于 python[*requests*](https://fpy.li/20-5)包，但是建立在更现代的基础上。至关重要的是， *HTTPX* 提供了同步和异步 API，因此我们可以在本章和下一章的所有 HTTP 客户端示例中使用它。Python 的标准库提供了`urllib.request`模块，但是它的 API 仅仅是同步的，并且不是用户友好的。

*flags.py* 真的没什么新意。它作为比较其他脚本的基线，我将它作为一个库来使用，以避免在实现它们时出现冗余代码。现在让我们看看使用`concurrent.futures`的重新实现。

## 用 concurrent.futures 下载

`concurrent.futures`包的主要特性是`ThreadPoolExecutor`和`ProcessPoolExecutor`类，它们分别实现了一个 API 来提交调用，以便在不同的线程或进程中执行。这些类透明地管理工作线程或进程池，以及分发作业和收集结果的队列。但是这个接口是非常高级的，对于一个简单的用例，比如我们的 flag 下载，我们不需要知道任何这些细节。

示例 20-3 展示了使用`ThreadPoolExecutor.map`方法实现并发下载的最简单方法。

##### 示例 20-3： flags_threadpool.py:使用`futures.ThreadPoolExecutor`的线程下载脚本

```
fromconcurrentimportfuturesfromflagsimportsave_flag,get_flag,main①defdownload_one(cc:str):②image=get_flag(cc)save_flag(image,f'{cc}.gif')print(cc,end='',flush=True)returnccdefdownload_many(cc_list:list[str])->int:withfutures.ThreadPoolExecutor()asexecutor:③res=executor.map(download_one,sorted(cc_list))④returnlen(list(res))⑤if__name__=='__main__':main(download_many)⑥
```

① 重用`flags`模块中的一些功能(示例 20-2 )。

② 函数下载单个图像；这是每个工人将要执行的。

③ 将`ThreadPoolExecutor`实例化为上下文管理器；`executor​.__exit__`方法将调用`executor.shutdown(wait=True)`，它将阻塞，直到所有线程都完成。

④ `map`方法类似于`map`内置，除了`download_one`函数会被多线程并发调用；它返回一个生成器，您可以迭代该生成器来检索每个函数调用返回的值——在这种情况下，每个对`download_one`的调用都将返回一个国家代码。

⑤ 返回获得的结果数。如果任何一个线程调用引发了异常，那么当`list`构造函数中的隐式`next()`调用试图从`executor.map`返回的迭代器中检索相应的返回值时，就会引发异常。

⑥ 从`flags`模块调用`main`函数，传递`download_many`的并发版本。

注意来自示例 20-3 的`download_one`函数实质上是来自示例 20-2 的`download_many`函数中的`for`循环体。在编写并发代码时，这是一个常见的重构:将一个顺序的`for`循环的主体变成一个要并发调用的函数。

###### 小费

示例 20-3 非常简短，因为我能够重用顺序 *flags.py* 脚本中的大多数函数。`concurrent.futures`最好的特性之一是使在遗留顺序代码上添加并发执行变得简单。

`ThreadPoolExecutor`构造函数有几个未显示的参数，但第一个也是最重要的参数是`max_workers`，它设置了要执行的最大工作线程数。当`max_workers`为`None`(默认值)时，`ThreadPool​Executor`使用以下表达式决定其值—从 Python 3.8 开始:

```
max_workers = min(32, os.cpu_count() + 4)
```

基本原理在 [`ThreadPoolExecutor`文档](https://fpy.li/20-6)中解释:

> 该默认值为 I/O 绑定任务保留至少 5 个工作线程。它最多利用 32 个 CPU 内核来执行 CPU 受限任务，从而释放 GIL。它避免了在众核机器上隐式使用大量资源。
> 
> `ThreadPoolExecutor`现在在启动`max_workers`工作线程之前重用空闲的工作线程。

总结一下:为`max_workers`计算的缺省值是合理的，并且`ThreadPoolExecutor`避免了不必要的启动新工人。理解`max_workers`背后的逻辑可以帮助你自己决定何时以及如何设置它。

这个库叫做 *concurrency.futures* ，但是在示例 20-3 中没有看到 futures，所以你可能想知道它们在哪里。下一节解释。

## 未来在哪里？

期货 是`concurrent.futures`和`asyncio`的核心组件，但是作为这些库的用户，我们有时看不到它们。示例 20-3 在幕后依赖期货，但是我写的代码并没有直接接触到它们。这一部分是对未来的概述，通过一个例子来展示它们的作用。

从 Python 3.4 开始，标准库中有两个名为`Future`的类:`concurrent.futures.Future`和`asyncio.Future`。它们服务于相同的目的:任何一个`Future`类的实例都代表一个可能已经完成也可能没有完成的延迟计算。这有点类似于 Twisted 中的`Deferred`类，Tornado 中的`Future`类，以及现代 JavaScript 中的`Promise`。

Futures 封装了挂起的操作，以便我们可以将它们放入队列，检查它们是否完成，并在它们变得可用时检索结果(或异常)。

关于未来，需要知道的一件重要的事情是，你和我都不应该创建它们:它们应该被并发框架专门实例化，无论是`concurrent.futures`还是`asyncio`。原因如下:`Future`代表最终会运行的东西，因此它必须被安排运行，这就是框架的工作。特别是，`concurrent.futures.Future`实例仅作为提交一个可调用的实例的结果来创建，以便与`concurrent.futures.Executor`子类一起执行。例如，`Executor.submit()`方法获取一个可调用函数，安排它运行，并返回一个`Future`。

应用程序代码不应该改变未来的状态:当它代表的计算完成时，并发框架会改变未来的状态，而我们无法控制这种情况何时发生。

两种类型的`Future`都有一个`.done()`方法，该方法是非阻塞的，并返回一个 Boolean，告诉你由那个 future 包装的可调用函数是否已经执行。然而，客户机代码通常要求得到通知，而不是反复询问未来是否完成。这就是为什么两个`Future`类都有一个`.add_done_callback()`方法:你给它一个 callable，当未来完成时，这个 callable 将以未来作为单一参数被调用。请注意，回调 callable 将在将来运行包装函数的同一个工作线程或进程中运行。

还有一个`.result()`方法，当未来完成时，它在两个类中的作用是相同的:它返回可调用函数的结果，或者重新引发可调用函数执行时可能抛出的任何异常。然而，当未来没有完成时，`result`方法的行为在两种风格的`Future`之间有很大的不同。在`concurrency.futures.Future`实例中，调用`f.result()`将阻塞调用者的线程，直到结果准备好。可以传递一个可选的`timeout`参数，如果未来没有在指定时间内完成，`result`方法会引发`TimeoutError`。`asyncio.Future.result`方法不支持超时，而`await`是在`asyncio`中获得未来结果的首选方式——但是`await`不能用于`concurrency.futures.Future`实例。

两个库中的几个函数返回期货；其他人在实现中以对用户透明的方式使用它们。后者的一个例子是我们在示例 20-3 中看到的`Executor.map`:它返回一个迭代器，其中`__next__`调用每个期货的`result`方法，所以我们得到的是期货的结果，而不是期货本身。

为了更实际地了解期货，我们可以重写示例 20-3 来使用 [`concurrent.futures.as_completed`](https://fpy.li/20-7) 函数，该函数接受期货的一个 iterable 并返回一个迭代器，该迭代器生成期货。

使用`futures.as_completed`只需要改变`download_many`功能。更高层次的`executor.map`调用被两个`for`循环代替:一个创建和调度期货，另一个检索它们的结果。在此过程中，我们将添加一些`print`调用来显示完成前后的每个未来。示例 20-4 显示了新`download_many`功能的代码。`download_many`的代码从 5 行增加到了 17 行，但是现在我们要检查神秘的未来。其余功能同示例 20-3 。

##### 示例 20-4：flags _ thread pool _ futures . py:将`executor.map`替换为`download_many`函数中的`executor.submit`和`futures.as_completed`

```
defdownload_many(cc_list:list[str])->int:cc_list=cc_list[:5]①withfutures.ThreadPoolExecutor(max_workers=3)asexecutor:②to_do:list[futures.Future]=[]forccinsorted(cc_list):③future=executor.submit(download_one,cc)④to_do.append(future)⑤print(f'Scheduled for {cc}: {future}')⑥forcount,futureinenumerate(futures.as_completed(to_do),1):⑦res:str=future.result()⑧print(f'{future} result: {res!r}')⑨returncount
```

① 在本演示中，仅使用人口最多的前五个国家。

② 将`max_workers`设置为`3`，这样我们可以在输出中看到未决的期货。

③ 按字母顺序遍历国家代码，以明确结果将无序到达。

④ `executor.submit`调度要执行的 callable，并返回一个代表该挂起操作的`future`。

⑤ 存储每个`future`，以便我们稍后可以使用`as_completed`检索它们。

⑥ 显示带有国家代码和相应的`future`的信息。

⑦ `as_completed`完成时产生期货。

⑧ 得到这个`future`的结果。

⑨ 显示`future`及其结果。

注意在这个例子中`future.result()`调用永远不会阻塞，因为`future`正从`as_completed`出来。示例 20-5 为示例 20-4 中一次运行的输出。

##### 示例 20-5：flags _ thread pool _ futures . py 的输出

```
$ python3 flags_threadpool_futures.py
Scheduled for BR: <Future at 0x100791518 state=running> ① Scheduled for CN: <Future at 0x100791710 state=running>
Scheduled for ID: <Future at 0x100791a90 state=running>
Scheduled for IN: <Future at 0x101807080 state=pending> ② Scheduled for US: <Future at 0x101807128 state=pending>
CN <Future at 0x100791710 state=finished returned str> result: 'CN' ③ BR ID <Future at 0x100791518 state=finished returned str> result: 'BR' ④ <Future at 0x100791a90 state=finished returned str> result: 'ID'
IN <Future at 0x101807080 state=finished returned str> result: 'IN'
US <Future at 0x101807128 state=finished returned str> result: 'US'

5 downloads in 0.70s
```

① 未来是按字母顺序排列的；未来的`repr()`表示它的状态:前三个是`running`，因为有三个工作线程。

② 最后两个期货是`pending`，等待工人线程。

③ 这里的第一个`CN`是`download_one`在一个工作线程中的输出；该行的其余部分是`download_many`的输出。

④ 这里主线程中`download_many`之前的两个线程输出代码可以显示第一个线程的结果。

###### 小费

我推荐尝试一下*flags _ thread pool _ futures . py*。如果你运行几次，你会发现结果的顺序有所不同。将`max_workers`增加到`5`会增加结果顺序的变化。将其减少到`1`将使这个脚本按顺序运行，结果的顺序将始终是`submit`调用的顺序。

我们看到了使用`concurrent.futures`的下载脚本的两个变体:一个在示例 20-3 中使用`ThreadPoolExecutor.map`，另一个在示例 20-4 中使用`futures.as_completed`。如果你对 *flags_asyncio.py* 的代码很好奇，可以偷看一下第 21 章中的示例 21-3 ，在那里有解释。

现在让我们简单地看一下使用`concurrent.futures`解决 CPU 密集型任务的 GIL 的一种简单方法。T34

# 使用并发期货启动流程

[`concurrent.futures`文档页面](https://fpy.li/20-8)的副标题为“启动并行任务”这个包支持多核机器上的并行计算，因为它支持使用`ProcessPool​Executor`类在多个 Python 进程之间分配工作。

`ProcessPoolExecutor`和`ThreadPoolExecutor`都实现了 [`Executor`](https://fpy.li/20-9) 接口，所以使用`concurrent.futures`很容易从基于线程的解决方案切换到基于流程的解决方案。

对于标志下载示例或任何 I/O 绑定的作业，使用`ProcessPoolExecutor`没有任何优势。这很容易验证；只需更改示例 20-3 中的这些行:

```
def download_many(cc_list: list[str]) -> int:
    with futures.ThreadPoolExecutor() as executor:
```

对此:

```
def download_many(cc_list: list[str]) -> int:
    with futures.ProcessPoolExecutor() as executor:
```

`ProcessPoolExecutor`的构造函数也有一个`max_workers`参数，默认为`None`。在这种情况下，执行程序将工人的数量限制为由`os.cpu_count()`返回的数量。

进程比线程使用更多的内存和更长的启动时间，所以`ProcessPoolExecutor` 的真正值是在 CPU 密集型作业中。让我们回到【一个自制的进程池】的素性测试例，用 `concurrent.futures`重写。

## 多核素数检测器 Redux

在“多核质数检查器代码”](ch19.xhtml#code_for_multicore_prime_sec)中，我们学习了 *procs.py* ，这是一个使用`multiprocessing`检查一些大数的质数的脚本。在[示例 20-6 中，我们使用`ProcessPool​Executor`解决了 *proc_pool.py* 程序中的相同问题。从第一次导入到最后的`main()`调用， *procs.py* 有 43 行非空代码， *proc_pool.py* 缩短了 31—28%。

##### 示例 20-6：proc _ pool . py:*procs . py*用`ProcessPoolExecutor`重写

```
importsysfromconcurrentimportfutures①fromtimeimportperf_counterfromtypingimportNamedTuplefromprimesimportis_prime,NUMBERSclassPrimeResult(NamedTuple):②n:intflag:boolelapsed:floatdefcheck(n:int)->PrimeResult:t0=perf_counter()res=is_prime(n)returnPrimeResult(n,res,perf_counter()-t0)defmain()->None:iflen(sys.argv)<2:workers=None③else:workers=int(sys.argv[1])executor=futures.ProcessPoolExecutor(workers)④actual_workers=executor._max_workers# type: ignore ⑤print(f'Checking {len(NUMBERS)} numbers with {actual_workers} processes:')t0=perf_counter()numbers=sorted(NUMBERS,reverse=True)⑥withexecutor:⑦forn,prime,elapsedinexecutor.map(check,numbers):⑧label='P'ifprimeelse''print(f'{n:16}  {label} {elapsed:9.6f}s')time=perf_counter()-t0print(f'Total time: {time:.2f}s')if__name__=='__main__':main()
```

① 无需导入`multiprocessing`、`SimpleQueue`等。；`concurrent.futures`隐藏所有这些。

② `PrimeResult`元组和`check`函数与我们在 *procs.py* 中看到的相同，但是我们不再需要队列和`worker`函数。

③ 如果没有给出命令行参数，我们不会自己决定使用多少工人，而是将`workers`设置为`None`并让`ProcessPoolExecutor`决定。

④ 在这里，我在➐的`with`块之前构建了`ProcessPoolExecutor`，这样我就可以显示下一行中的实际工人数量。

⑤ `_max_workers`是`ProcessPoolExecutor`的一个未记录的实例属性。当变量`workers`为`None`时，我决定用它来显示工人的数量。当我访问 Mypy 时，它会正确地报错，所以我添加了`type: ignore`注释来使其静音。

⑥ 将待检查的数字按降序排列。这将揭示 *proc_pool.py* 与 *procs.py* 的行为差异。见本例后解释。

⑦ 使用`executor`作为上下文管理器。

⑧ `executor.map`调用按照与`numbers`参数相同的顺序返回由`check`返回的`PrimeResult`实例。

如果你运行示例 20-6 ，你会看到结果以严格的降序出现，如示例 20-7 所示。相比之下， *procs.py* 的输出顺序(如“基于过程的解决方案”所示)受到检查每个数字是否为素数的难度的严重影响。例如， *procs.py* 显示了接近顶部的 77777777777777 的结果，因为它的除数很低，为 7，所以`is_prime`很快确定它不是一个素数。

相比之下，777777536340681 是 88191709 ² ，因此`is_prime`将需要更长时间来确定它是一个合数，甚至需要更长时间来确定 7777777777777753 是质数——因此这两个数字都出现在 *procs.py* 输出的末尾附近。

运行 *proc_pool.py* ，您不仅会观察到结果的降序，还会观察到程序在显示 99999999999999 的结果后似乎被卡住了。

##### 示例 20-7：proc _ pool . py 的输出

```
$ ./proc_pool.py
Checking 20 numbers with 12 processes:
9999999999999999     0.000024s ① 9999999999999917  P  9.500677s ② 7777777777777777     0.000022s ③ 7777777777777753  P  8.976933s
7777777536340681     8.896149s
6666667141414921     8.537621s
6666666666666719  P  8.548641s
6666666666666666     0.000002s
5555555555555555     0.000017s
5555555555555503  P  8.214086s
5555553133149889     8.067247s
4444444488888889     7.546234s
4444444444444444     0.000002s
4444444444444423  P  7.622370s
3333335652092209     6.724649s
3333333333333333     0.000018s
3333333333333301  P  6.655039s
 299593572317531  P  2.072723s
 142702110479723  P  1.461840s
               2  P  0.000001s
Total time: 9.65s
```

① 这条线出现得很快。

② 这一行需要 9.5s 以上才能显现出来。

③ 所有剩余的行几乎立即出现。

这就是为什么 *proc_pool.py* 会这样:

*   如前所述，`executor.map(check, numbers)`返回结果的顺序与`numbers`给出的顺序相同。

*   默认情况下， *proc_pool.py* 使用与 CPU 数量一样多的工作线程——这就是当`max_workers`为`None`时 `ProcessPoolExecutor` 所做的事情。这台笔记本电脑有 12 个进程。

*   因为我们是降序提交`numbers`，第一个是 9999999999999999；以 9 为除数，返回很快。

*   第二个数字是 9999999999999917，是样本中最大的素数。这将比所有其他检查花费更长的时间。

*   与此同时，剩下的 11 个进程将检查其他数字，这些数字要么是素数，要么是具有大因子的合成数，要么是具有非常小因子的合成数。

*   当负责 99999999999999917 的工人最终确定这是一个质数时，所有其他进程都完成了它们最后的作业，因此结果会立即出现。

###### 注意

尽管 *proc_pool.py* 的进程不如 *procs.py* 的进程可见，但对于相同数量的工作线程和 CPU 内核，总体执行时间实际上与图 19-2 所示的相同。

理解并发程序的行为并不简单，所以这里有第二个实验可以帮助你可视化`Executor.map`的操作。

# 使用 Executor.map 进行实验

让我们 研究一下`Executor.map`，现在使用一个`ThreadPoolExecutor`，有三个工人运行五个输出时间戳消息的调用程序。代码在示例 20-8 中，输出在示例 20-9 中。

##### 示例 20-8： demo_executor_map.py:简单演示`ThreadPoolExecutor`的 map 方法

```
fromtimeimportsleep,strftimefromconcurrentimportfuturesdefdisplay(*args):①print(strftime('[%H:%M:%S]'),end='')print(*args)defloiter(n):②msg='{}loiter({}): doing nothing for {}s...'display(msg.format('\t'*n,n,n))sleep(n)msg='{}loiter({}): done.'display(msg.format('\t'*n,n))returnn*10③defmain():display('Script starting.')executor=futures.ThreadPoolExecutor(max_workers=3)④results=executor.map(loiter,range(5))⑤display('results:',results)⑥display('Waiting for individual results:')fori,resultinenumerate(results):⑦display(f'result {i}: {result}')if__name__=='__main__':main()
```

① 这个函数简单地打印它得到的任何参数，前面加上一个格式为`[HH:MM:SS]`的时间戳。

② `loiter`除了在启动时显示一条消息，休眠`n`秒，然后在结束时显示一条消息外，什么也不做；制表符用于根据`n`的值缩进消息。

③ `loiter`返回`n * 10`，这样我们可以看到如何收集结果。

④ 创建一个有三个线程的`ThreadPoolExecutor`。

⑤ 向`executor`提交五个任务。因为只有三个线程，所以只有三个任务会立即开始:调用`loiter(0)`、`loiter(1)`和`loiter(2)`；这是一个非阻塞调用。

⑥ 立即显示调用`executor.map`的`results`:它是一个发电机，如示例 20-9 中的输出所示。

⑦ `for`循环中的`enumerate`调用将隐式调用`next(results)`，而`next(results)`又将调用代表第一个调用`loiter(0)`的【内部】`_f`期货上的`_f.result()`。`result`方法将阻塞，直到未来完成，因此该循环中的每次迭代都必须等待下一个结果准备好。

我鼓励你运行示例 20-8 ，并看到显示正在逐步更新。当你这样做的时候，试着使用`ThreadPool​Executor`的`max_workers`参数和为`executor.map`调用产生参数的`range`函数——或者用手工选择的值列表来替换它，以创建不同的延迟。

示例 20-9 为示例 20-8 中的试运行。

##### 示例 20-9：示例 20-8 中 demo_executor_map.py 的运行示例

```
$ python3 demo_executor_map.py
[15:56:50] Script starting. ① [15:56:50] loiter(0): doing nothing for 0s... ② [15:56:50] loiter(0): done.
[15:56:50]      loiter(1): doing nothing for 1s... ③ [15:56:50]              loiter(2): doing nothing for 2s...
[15:56:50] results: <generator object result_iterator at 0x106517168> ④ [15:56:50]                      loiter(3): doing nothing for 3s... ⑤ [15:56:50] Waiting for individual results:
[15:56:50] result 0: 0 ⑥ [15:56:51]      loiter(1): done. ⑦ [15:56:51]                              loiter(4): doing nothing for 4s...
[15:56:51] result 1: 10 ⑧ [15:56:52]              loiter(2): done. ⑨ [15:56:52] result 2: 20
[15:56:53]                      loiter(3): done.
[15:56:53] result 3: 30
[15:56:55]                              loiter(4): done. ⑩ [15:56:55] result 4: 40
```

① 这次跑步开始于 15:56:50。

② 第一个线程执行`loiter(0)`，所以会休眠 0s，甚至在第二个线程有机会启动之前就返回，但是 YMMV。 [6]

③ `loiter(1)`和`loiter(2)`立即启动(因为线程池有三个工作线程，所以可以并发运行三个函数)。

④ 这说明`executor.map`返回的`results`是发电机；到目前为止，无论任务数量和`max_workers`设置如何，都不会阻塞。

⑤ 因为`loiter(0)`已经完成，第一个工作者现在可以为`loiter(3)`启动第四个线程。

⑥ 这是执行可能阻塞的地方，取决于给`loiter`调用的参数:`results`生成器的`__next__`方法必须等到第一个 future 完成。在这种情况下，它不会阻塞，因为对`loiter(0)`的调用在这个循环开始之前已经完成。请注意，到目前为止，所有事情都发生在同一秒钟内:15:56:50。

⑦ `loiter(1)`一秒钟后完成，15:56:51。线程被释放以启动 `loiter(4)`。

⑧ `loiter(1)`的结果显示:`10`。现在`for`循环将阻塞等待`loiter(2)`的结果。

⑨ 模式重复:`loiter(2)`完成，其结果显示；同`loiter(3)`。

⑩ 在`loiter(4)`完成之前有 2s 的延迟，因为它在 15:56:51 开始，4s 内什么也没做。

`Executor.map`函数很容易使用，但通常最好是在结果准备好的时候就获取，而不考虑提交的顺序。为此，我们需要一个`Executor.submit`方法和`futures.as_completed`函数的组合，正如我们在示例 20-4 中看到的。我们将在“使用 futures . as _ completed”中回到这个技巧。

###### 小费

`executor.submit`和`futures.as_completed`的组合比`executor.map`更灵活，因为你可以`submit` 不同的调用和参数，而`executor.map`被设计成在不同的参数上运行相同的调用。此外，传递给`futures.as_completed`的期货集合可能来自不止一个执行人——也许有些是由的`ThreadPoolExecutor` 实例创建的，而其他的则来自 `ProcessPoolExecutor`。

在下一节中，我们将恢复带有新需求的标志下载示例，这将迫使我们迭代`futures.as_completed`的结果，而不是使用`executor.map`。

# 带有进度显示和错误处理的下载

正如 所提到的，“并发 Web 下载”中的脚本没有错误处理，以便于阅读和对比三种方法的结构:顺序、线程和异步。

为了测试对各种错误条件的处理，我创建了`flags2`示例:

flags2_common.py

该模块包含所有`flags2`示例使用的通用函数和设置，包括一个`main`函数，它负责命令行解析、计时和报告结果。那是真正的支持代码，和本章的主题没有直接关系，所以我这里就不列出源代码了，不过你可以在[*fluent python/example-code-2e*](https://fpy.li/code)repository:[*20-executors/get flags/flags 2 _ common . py*](https://fpy.li/20-10)中读到。

flags2_sequential.py

一个顺序 HTTP 客户端，具有正确的错误处理和进度条显示。它的`download_one`功能也被`flags2_threadpool.py`使用。

flags2_threadpool.py

基于`futures.ThreadPoolExecutor`的并发 HTTP 客户端，演示错误处理和进度条的集成。

flags2_asyncio.py

功能与前一示例相同，但使用`asyncio`和`httpx`实现。这将在第 21 章的“增强 asyncio 下载器”、中涉及。

# 测试并发客户端时要小心

当在公共 web 服务器上测试并发 HTTP 客户端时，您可能每秒生成许多请求，这就是拒绝服务(DoS)攻击的方式。当攻击公共服务器时，要小心控制你的客户端。为了进行测试，设置一个本地 HTTP 服务器。参见“设置测试服务器”获取说明。

`flags2`示例最明显的特征是它们有一个动画的文本模式进度条，用 *tqdm* 包](https://fpy.li/20-11)实现。我在 YouTube 上贴了一个 [108s 的视频，展示进度条，对比三个`flags2`脚本的速度。在视频中，我从连续的下载开始，但是我在 32 秒后中断了它，因为点击 676 个 URL 并获得 194 个标志需要超过 5 分钟。然后，我运行线程化和`asyncio`脚本各三次，每次它们都在 6 秒或更短时间内完成任务(也就是说，快了 60 多倍)。](https://fpy.li/20-12)[图 20-1 显示了两个截图:运行*flags 2 _ thread pool . py*时和运行后。

![flags2_threadpool.py running with progress bar](Images/flpy_2001.png)

###### 图 20-1。左上:用 tqdm 生成的实时进度条运行 flags 2 _ thread pool . py；右下角:脚本完成后的同一个终端窗口。

最简单的 *tqdm* 例子出现在动画*中。gif* 在项目的 [*README.md*](https://fpy.li/20-13) 。如果您在安装了 *tqdm* 包之后在 Python 控制台中键入以下代码，您将会看到一个动画进度条，其中的注释是:

```
>>> import time
>>> from tqdm import tqdm
>>> for i in tqdm(range(1000)):
...     time.sleep(.01)
...
>>> # -> progress bar will appear here <-
```

除了简洁的效果之外，`tqdm`函数在概念上也很有趣:它消耗任何 iterable 并产生一个迭代器，当它被消耗时，显示进度条并估计完成所有迭代的剩余时间。为了计算这个估计值，`tqdm`需要获得一个具有`len`的 iterable，或者另外接收一个具有预期项数的`total=`参数。将`tqdm`与我们的`flags2`示例集成在一起提供了一个机会，通过强迫我们使用 [`futures.as_completed`](https://fpy.li/20-7) 和 [`asyncio.as_completed`](https://fpy.li/20-15) 函数，让`tqdm`可以显示每个未来完成时的进度。

`flags2`示例的另一个特性是命令行界面。所有三个脚本都接受相同的选项，您可以通过运行任何带有`-h`选项的脚本来查看它们。示例 20-10 显示了帮助文本。

##### 示例 20-10：flags 2 系列中脚本的帮助屏幕

```
$ python3 flags2_threadpool.py -h
usage: flags2_threadpool.py [-h] [-a] [-e] [-l N] [-m CONCURRENT] [-s LABEL]
                            [-v]
                            [CC [CC ...]]

Download flags for country codes. Default: top 20 countries by population.

positional arguments:
  CC                    country code or 1st letter (eg. B for BA...BZ)

optional arguments:
  -h, --help            show this help message and exit
  -a, --all             get all available flags (AD to ZW)
  -e, --every           get flags for every possible code (AA...ZZ)
  -l N, --limit N       limit to N first codes
  -m CONCURRENT, --max_req CONCURRENT
                        maximum concurrent requests (default=30)
  -s LABEL, --server LABEL
                        Server to hit; one of DELAY, ERROR, LOCAL, REMOTE
                        (default=LOCAL)
  -v, --verbose         output detailed progress info
```

所有参数都是可选的。但是`-s/--server`对于测试来说是必不可少的:它让您选择在测试中使用哪个 HTTP 服务器和端口。传递这些不区分大小写的标签之一，以确定脚本将在何处查找标志:

`LOCAL`

使用`http://localhost:8000/flags`；这是默认值。您应该配置一个本地 HTTP 服务器在端口 8000 上应答。有关说明，请参见以下注释。

`REMOTE`

使用`http://fluentpython.com/data/flags`；这是一个由我拥有的公共网站，托管在一个共享服务器上。请不要用太多的并发请求来打击它。*fluentpython.com*域由 [Cloudflare](https://fpy.li/20-16) CDN(内容交付网络)处理，所以你可能会注意到第一次下载速度较慢，但当 CDN 缓存预热时，下载速度会变快。

`DELAY`

使用`http://localhost:8001/flags`；延迟 HTTP 响应的服务器应该监听端口 8001。我写 *slow_server.py* 是为了更容易实验。你会在 [*Fluent Python* 代码库](https://fpy.li/code)的 *20-futures/getflags/* 目录下找到。有关说明，请参见以下注释。

`ERROR`

使用`http://localhost:8002/flags`；返回一些 HTTP 错误的服务器应该监听端口 8002。接下来是说明。

# 设置测试服务器

如果 你没有本地 HTTP 服务器进行测试，我在[*fluent Python/example-code-2e*](https://fpy.li/code)资源库中的[*20-executors/get flags/readme . adoc*](https://fpy.li/20-17)中只使用 Python ≥ 3.9(无外部库)编写了设置说明。简而言之， *README.adoc* 描述了如何使用:

`python3 -m http.server`

端口 8000 上的`LOCAL`服务器

`python3 slow_server.py`

端口 8001 上的`DELAY`服务器，在每次响应前增加随机延迟 .5s 到 5s

`python3 slow_server.py 8002 --error-rate .25`

端口 8002 上的`ERROR`服务器，除了随机延迟之外，还有 25%的机会返回[“418 我是茶壶”](https://fpy.li/20-18)错误响应

默认情况下，每个*标记 2*。py* 脚本将使用默认的并发连接数从`LOCAL`服务器(`http://localhost:8000/flags`)获取 20 个人口最多的国家的国旗，默认的并发连接数因脚本而异。示例 20-11 显示了使用所有默认设置运行 *flags2_sequential.py* 脚本的示例。要运行它，你需要一个本地服务器，正如“测试并发客户端时要小心”中所解释的。

##### 示例 20-11：使用所有默认值运行 flags 2 _ sequential . py:`LOCAL site`，前 20 个标志，1 个并发连接

```
$ python3 flags2_sequential.py
LOCAL site: http://localhost:8000/flags
Searching for 20 flags: from BD to VN
1 concurrent connection will be used.
--------------------
20 flags downloaded.
Elapsed time: 0.10s
```

您可以通过多种方式选择要下载的标志。示例 20-12 显示了如何下载国家代码以字母 A、B 或 c 开头的所有旗帜。

##### 示例 20-12：运行 flags2_threadpool.py 从`DELAY`服务器获取所有带有国家代码前缀 A、B 或 C 的标志

```
$ python3 flags2_threadpool.py -s DELAY a b c
DELAY site: http://localhost:8001/flags
Searching for 78 flags: from AA to CZ
30 concurrent connections will be used.
--------------------
43 flags downloaded.
35 not found.
Elapsed time: 1.72s
```

无论如何选择国家代码，可通过`-l/--limit`选项限制要获取的标志数量。示例 20-13 演示了如何准确运行 100 个请求，结合`-a`选项，用`-l 100`获取所有标志。

##### 示例 20-13：运行 flags2_asyncio.py，使用 100 个并发请求(`-m 100`)从`ERROR`服务器获取 100 个标志(`-al 100`)

```
$ python3 flags2_asyncio.py -s ERROR -al 100 -m 100
ERROR site: http://localhost:8002/flags
Searching for 100 flags: from AD to LK
100 concurrent connections will be used.
--------------------
73 flags downloaded.
27 errors.
Elapsed time: 0.64s
```

这是`flags2`示例的用户界面。让我们看看它们是如何实现的。

## flags2 示例中的错误处理

这三个例子中处理 HTTP 错误的共同策略是由负责下载单个文件的函数(`download_one`)处理 404 错误(未发现)。任何其他异常传播到由`download_many`函数或`supervisor`协程处理——在`asyncio`示例中。

同样，我们将从研究顺序代码开始，顺序代码更容易理解——并且主要由线程池脚本重用。示例 20-14 显示了在 *flags2_sequential.py* 和 *flags2_threadpool.py* 脚本中执行实际下载的函数。

##### 示例 20-14： flags2_sequential.py:负责下载的基本函数；两者都在 flags2_threadpool.py 中重用

```
fromcollectionsimportCounterfromhttpimportHTTPStatusimporthttpximporttqdm# type: ignore ①fromflags2_commonimportmain,save_flag,DownloadStatus②DEFAULT_CONCUR_REQ=1MAX_CONCUR_REQ=1defget_flag(base_url:str,cc:str)->bytes:url=f'{base_url}/{cc}/{cc}.gif'.lower()resp=httpx.get(url,timeout=3.1,follow_redirects=True)resp.raise_for_status()③returnresp.contentdefdownload_one(cc:str,base_url:str,verbose:bool=False)->DownloadStatus:try:image=get_flag(base_url,cc)excepthttpx.HTTPStatusErrorasexc:④res=exc.responseifres.status_code==HTTPStatus.NOT_FOUND:status=DownloadStatus.NOT_FOUND⑤msg=f'not found: {res.url}'else:raise⑥else:save_flag(image,f'{cc}.gif')status=DownloadStatus.OKmsg='OK'ifverbose:⑦print(cc,msg)returnstatus
```

① 导入`tqdm`进度条显示库，并告诉 Mypy 跳过检查。 [7]

② 从`flags2_common`模块导入几个函数和一个`Enum`。

③ 如果 HTTP 状态代码不在`range(200, 300)`中，则引发`HTTPStetusError`。

④ `download_one`捕捉`HTTPStatusError`专门处理 HTTP 代码 404…

⑤ …通过将其本地`status`设置为`DownloadStatus.NOT_FOUND`；`DownloadStatus`是从 *flags2_common.py* 导入的`Enum`。

⑥ 任何其他的`HTTPStatusError`异常都会被重新引发并传播给调用者。

⑦ 如果设置了`-v/--verbose`命令行选项，则显示国家代码和状态信息；这是您在详细模式下看到的进度。

示例 20-15 列出了`download_many`功能的顺序版本。这段代码很简单，但是与即将出现的并发版本进行对比是值得研究的。关注它如何报告进度、处理错误和记录下载。

##### 示例 20-15：flags 2 _ sequential . py:`download_many`的顺序实现

```
defdownload_many(cc_list:list[str],base_url:str,verbose:bool,_unused_concur_req:int)->Counter[DownloadStatus]:counter:Counter[DownloadStatus]=Counter()①cc_iter=sorted(cc_list)②ifnotverbose:cc_iter=tqdm.tqdm(cc_iter)③forccincc_iter:try:status=download_one(cc,base_url,verbose)④excepthttpx.HTTPStatusErrorasexc:⑤error_msg='HTTP error {resp.status_code} - {resp.reason_phrase}'error_msg=error_msg.format(resp=exc.response)excepthttpx.RequestErrorasexc:⑥error_msg=f'{exc} {type(exc)}'.strip()exceptKeyboardInterrupt:⑦breakelse:⑧error_msg=''iferror_msg:status=DownloadStatus.ERROR⑨counter[status]+=1⑩ifverboseanderror_msg:⑪print(f'{cc} error: {error_msg}')returncounter⑫
```

① 这个`Counter`将记录不同的下载结果:`DownloadStatus.OK`、`DownloadStatus.NOT_FOUND`或`DownloadStatus.ERROR`。

② `cc_iter`保存作为参数接收的国家代码列表，按字母顺序排序。

③ 如果不是在详细模式下运行，`cc_iter`将被传递给`tqdm`，后者返回一个迭代器，生成`cc_iter`中的项目，同时还显示进度条。

④ 连续呼叫`download_one`。

⑤ 由`get_flag`引发且不由`download_one`处理的 HTTP 状态码异常在此处理。

⑥ 其他与网络相关的异常在这里处理。任何其他异常都将中止脚本，因为调用`download_many`的`flags2_common.main`函数没有`try/except`。

⑦ 如果用户点击 Ctrl-C，退出循环。

⑧ 如果没有异常逃脱`download_one`，清除错误信息。

⑨ 如果有错误，相应地设置本地`status`。

⑩ 增加该`status`的计数器。

⑪ 在详细模式下，显示当前国家代码的错误消息(如果有)。

⑫ 返回`counter`以便`main`可以在最终报告中显示数字。

我们现在将研究重构的线程池示例， *flags2_threadpool.py* 。

## 使用 futures.as_completed

在中，为了集成 *tqdm* 进度条并处理每个请求的错误，的*flags 2 _ thread pool . py*脚本将`futures.ThreadPoolExecutor`与我们已经看到的 `futures.as_completed` 函数一起使用。示例 20-16 是 *flags2_threadpool.py* 的完整列表。仅执行`download_many`功能；其他函数重用自 *flags2_common.py* 和 *flags2_sequential.py* 。

##### 示例 20-16： flags2_threadpool.py:完整列表

```
fromcollectionsimportCounterfromconcurrent.futuresimportThreadPoolExecutor,as_completedimporthttpximporttqdm# type: ignorefromflags2_commonimportmain,DownloadStatusfromflags2_sequentialimportdownload_one①DEFAULT_CONCUR_REQ=30②MAX_CONCUR_REQ=1000③defdownload_many(cc_list:list[str],base_url:str,verbose:bool,concur_req:int)->Counter[DownloadStatus]:counter:Counter[DownloadStatus]=Counter()withThreadPoolExecutor(max_workers=concur_req)asexecutor:④to_do_map={}⑤forccinsorted(cc_list):⑥future=executor.submit(download_one,cc,base_url,verbose)⑦to_do_map[future]=cc⑧done_iter=as_completed(to_do_map)⑨ifnotverbose:done_iter=tqdm.tqdm(done_iter,total=len(cc_list))⑩forfutureindone_iter:⑪try:status=future.result()⑫excepthttpx.HTTPStatusErrorasexc:⑬error_msg='HTTP error {resp.status_code} - {resp.reason_phrase}'error_msg=error_msg.format(resp=exc.response)excepthttpx.RequestErrorasexc:error_msg=f'{exc} {type(exc)}'.strip()exceptKeyboardInterrupt:breakelse:error_msg=''iferror_msg:status=DownloadStatus.ERRORcounter[status]+=1ifverboseanderror_msg:cc=to_do_map[future]⑭print(f'{cc} error: {error_msg}')returncounterif__name__=='__main__':main(download_many,DEFAULT_CONCUR_REQ,MAX_CONCUR_REQ)
```

① 重复使用`flags2_sequential`中的`download_one`(示例 20-14 )。

② 如果没有给出`-m/--max_req`命令行选项，这将是并发请求的最大数量，实现为线程池的大小；如果要下载的标志数量较少，实际数量可能会更少。

③ `MAX_CONCUR_REQ`限制并发请求的最大数量，而不考虑要下载的标志数量或`-m/--max_req`命令行选项。这是一种安全预防措施，以避免启动太多线程，造成大量内存开销。

④ 创建`executor`，将`max_workers`设置为`concur_req`，由`main`函数计算为:`MAX_CONCUR_REQ`、`cc_list`的长度或`-m/--max_req`命令行选项的值中的较小值。这避免了创建不必要的线程。

⑤ 这个`dict`将把每个`Future`实例——代表一次下载——与各自的错误报告国家代码进行映射。

⑥ 按字母顺序遍历国家代码列表。结果的顺序更多地取决于 HTTP 响应的时间，但是如果线程池的大小(由`concur_req`给出)比`len(cc_list)`小得多，您可能会注意到下载是按字母顺序分组的。

⑦ 对`executor.submit`的每次调用都调度一个可调用程序的执行，并返回一个`Future`实例。第一个参数是 callable，其余的是它将接收的参数。

⑧ 将`future`和国家代码存储在`dict`中。

⑨ `futures.as_completed`返回一个迭代器，该迭代器在每个任务完成时产生未来。

⑩ 如果不是在 verbose 模式下，用`tqdm`函数包装`as_completed`的结果，显示进度条；因为`done_iter`没有`len`，所以我们必须告诉`tqdm`预期的项数是多少作为`total=`的自变量，这样`tqdm`就可以估算剩余的工作。

⑪ 当未来完成时，对其进行迭代。

⑫ 对 future 调用`result`方法或者返回 callable 返回的值，或者引发 callable 执行时捕获的任何异常。该方法可能会阻止等待解决方案，但在本例中不会，因为`as_completed`只返回已完成的期货。

⑬ 处理潜在的异常情况；该功能的其余部分与示例 20-15 中的顺序`download_many`相同，除了下一个标注。

⑭ 为了提供错误信息的上下文，使用当前的`future`作为关键字从`to_do_map`中检索国家代码。这在顺序版本中是不必要的，因为我们在迭代国家代码列表，所以我们知道当前的`cc`；我们在这里迭代未来。

###### 小费

示例 20-16 使用了一个对`futures.as_completed`非常有用的习语:建立一个`dict`来将每个未来映射到当未来完成时可能有用的其他数据。这里的`to_do_map`将每个未来映射到分配给它的国家代码。这使得对期货结果进行后续处理变得容易，尽管事实上它们是无序生产的。

Python 线程非常适合 I/O 密集型应用程序，并且`concurrent.futures`包使得它对于某些用例的使用相对简单。使用`ProcessPoolExecutor`，您还可以在多个内核上解决 CPU 密集型问题——如果计算是[“令人尴尬的并行”](https://fpy.li/20-19)。我们对`concurrent.futures`的基本介绍到此结束。

# 章节摘要

我们通过比较两个并发 HTTP 客户端和一个顺序客户端开始了这一章，展示了并发解决方案比顺序脚本有显著的性能提升。

在研究了基于`concurrent.futures`的第一个例子后，我们仔细观察了未来对象，或者是`concurrent.futures.Future`或者是`asyncio​.Future`的实例，强调了这些类的共同点(它们的区别将在第 21 章中强调)。我们看到了如何通过调用`Executor.submit`来创建未来，并使用`concurrent.futures.as_completed`来迭代完成的未来。

然后我们讨论了用`concurrent.futures.ProcessPoolExecutor`类使用多个进程，绕过 GIL 并使用多个 CPU 内核来简化我们在第 19 章中第一次看到的多核素数检查器。

在接下来的部分中，我们看到了`concurrent.futures.ThreadPoolExecutor`如何通过一个教导性的例子工作，启动几秒钟内什么都不做的任务，除了用时间戳显示它们的状态。

接下来，我们回到旗帜下载的例子。用进度条和适当的错误处理来增强它们，促进了对`future.as_completed`生成器函数的进一步探索，展示了一个通用模式:在`dict`中存储未来，以便在提交时将进一步的信息链接到它们，这样当未来出现在`as_completed`迭代器中时，我们可以使用这些信息。

# 进一步阅读

Brian Quinlan 贡献了这个 `concurrent.futures`包，他在一个名为[“未来就在眼前！”](https://fpy.li/20-20)在 2010 年澳大利亚国际电脑展上。昆兰的演讲没有幻灯片；他通过在 Python 控制台中直接键入代码来展示这个库的功能。作为一个激励性的例子，演示中有一个简短的视频，XKCD 漫画家/程序员兰道尔·门罗对谷歌地图进行了一次无意的 DoS 攻击，以建立一个他所在城市周围的驾驶时间彩色地图。对该库的正式介绍是[PEP 3148-`futures`——异步执行计算](https://fpy.li/pep3148)。在 PEP 中，昆兰写道`concurrent.futures`库“深受 Java `java.util.concurrent`包的影响。”

有关`concurrent.futures`的其他资源，请参见第 19 章。在“线程和进程并发”中所有涉及 Python 的`threading`和`multiprocessing`的参考资料也涵盖了`concurrent.futures`。

[1] 来自 Michele Simionato 的帖子[“Python 中的线程、进程和并发性:一些想法”](https://fpy.li/20-1)，总结为“消除围绕多核(非)革命的宣传，以及一些(希望是)关于线程和其他形式的并发性的明智评论。”

[2] 特别是如果您的云提供商按秒租用机器，不管 CPU 有多忙。

[4] 这些图片最初来自美国政府出版物 [CIA World Factbook](https://fpy.li/20-4) 。我把它们复制到我的网站上，以避免对 cia.gov 发起 DOS 攻击的风险。

[5] 设置`follow_redirects=True`在这个例子中是不需要的，但是我想强调一下 *HTTPX* 和*请求*之间的重要区别。此外，在这个例子中设置`follow_redirects=True`给了我将来在其他地方托管图像文件的灵活性。我认为`follow_redirects​=False`的 *HTTPX* 默认设置是明智的，因为意外的重定向会掩盖不必要的请求并使错误诊断复杂化。

[6] 您的里程数可能会有所不同:对于线程，您永远不知道几乎同时发生的事件的确切顺序；有可能在另一台机器上，你看到`loiter(1)`在`loiter(0)`结束前开始，特别是因为`sleep`总是释放 GIL，所以即使你睡了 0 秒，Python 也可能切换到另一个线程。

[7] 截至 2021 年 9 月，`tdqm`当前版本中没有类型提示。没关系。世界不会因此而毁灭。感谢 Guido 的可选输入！

[8] 第 9 张幻灯片，来自[“协程和并发的奇妙课程”](https://fpy.li/20-21)教程，在 PyCon 2009 上展示。