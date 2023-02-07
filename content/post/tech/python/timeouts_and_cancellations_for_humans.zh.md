---
author: "Author Nathaniel J. Smith, translated by Dennis"
title: "Timeouts and Cancellations for Humans"
date: 2023-02-07T15:39:59+08:00
description: "作者介绍了目前的一些超时和取消的方案，提出了自己的取消域的方案，并介绍了一下自己的Trio库。"
draft: true
tags: ["python", "async programming", "timeouts", "I/O", "asyncio"]
categories: ["tech"]
postid: "9772a5f52a6a2f06d7c1ddd40f877d2cb51ca113"
---
# 说明：

- 原文出处[Timeouts and cancellation for humans — njs blog (vorpus.org)](https://vorpus.org/blog/timeouts-and-cancellation-for-humans/)。
- 译者以已经获得了原作者的允许以翻译该文章。
- 本文原作者为[Nathaniel J. Smith](https://github.com/njsmith)，[Trio](https://github.com/python-trio/trio)库的作者，[NumPy](https://github.com/numpy), [PyPa](https://github.com/pypa)相关项目的代码贡献者。
- 术语翻译说明：翻译内容可能混用，函数参数名会保留英文，常见词(如bug)不翻译。
    - primitive: 原语、底层函数
    - cancel scope: 取消域
    - cancel token: 取消令牌
    - timeout: 超时
    - deadline: 大家很习惯且考虑到翻译成截止时间比较啰嗦就不做翻译了。
    - shield: 没有想好如何翻译，就没有做翻译。
    - nursery: Trio中的一个启动子任务的系统，翻译成托儿所有点傻，就未做翻译。
- 其他翻译：见[翻译说明](#翻译说明)
- 本文评论使用的是disqus

# 引言

你的代码可能完美无缺，但不幸的是外面的世界并不这么靠谱。有些时候，其他人的代码可能会崩溃或者卡住。网络会掉线，打印机会报错。你的代码也需要为这些情况做好准备。每当你从网络读取数据、尝试获取跨进程锁、或者发送一条HTTP请求，你至少要考虑到以下三种可能：

- 请求可能成功
- 请求可能失败
- 请求可能挂起。春去秋来，没有成功，没有失败，没有任何响应。

前面两种情况非常直观。最后一种情况你需要使用超时来处理。几乎每一处你与其他进程、系统通信的地方都需要设置超时，如果你没有设置超时，那就是一个潜在的bug。

说实话，如果你也像大多数开发者一样，那么你的代码可能会有很多因为缺失超时导致的bug，我的代码也是这样的。因为正确进行I/O操作非常常见且重要，你可能觉得任何一种编程环境都能为所有操作提供简单而健壮的设置超时的方法。但奇怪的是，事实并不如此。大多数带超时的API都非常单调且容易出错，以至于要求开发人员把这些弄对是不切实际的。所以别难过，你的代码有这些超时相关的bug并不是你的错，而是那些I/O库的错！

我目前正在写一个[I/O库](https://trio.readthedocs.io/en/stable/)。和其他任何I/O库都不一样，这个I/O库的所有卖点是它痴迷于简单易用。我想让你能够在使用Trio时，能够简单而又正确地对任意I/O操作应用超时。但是，用户友好的超时API是一件很棘手的事。所以我会在这篇博客中深入讨论各种可能的设计，特别是那些给我提供灵感的设计。接下来我会介绍我想到的设计，以及为什么我会认为该设计是古老的”状态艺术”的一个切实优化。最后，我将会讨论为什么Trio库的思路为适应面更广，同时我将会给出一个不错的同步Python的原型实现。

那么超时处理有什么难的呢？

# 目录:
- [简单的超时机制不支持抽象](#简单的超时机制不支持抽象)
- [绝对的deadline是可组合的(但是用起来很麻烦)](#绝对的deadline是可组合的但是用起来很麻烦)
- [取消令牌(Cancel tokens)](#取消令牌cancel-tokens)
  - [取消令牌封装了取消状态](#取消令牌封装了取消状态)
  - [取消令牌属于水平触发，可以根据你程序的需要来确定范围](#取消令牌属于水平触发可以根据你程序的需要来确定范围)
  - [实际上因为人类的懒惰取消令牌并不可靠](#实际上因为人类的懒惰取消令牌并不可靠)
- [取消域: Trio关于超时和取消的人性化方案](#取消域-trio关于超时和取消的人性化方案)
  - [取消域(Cancel scope)是如何工作的](#取消域cancel-scope是如何工作的)
  - [我们需要在哪些地方检查取消？](#我们需要在哪些地方检查取消)
  - [一个逃逸口(escape hatch)](#一个逃逸口escape-hatch)
  - [取消域与并发](#取消域与并发)
  - [小结](#小结)
- [还有哪些地方可以从取消域中受益？](#还有哪些地方可以从取消域中受益)
  - [同步、单线程Python](#同步单线程python)
  - [asyncio](#asyncio)
  - [其他语言](#其他语言)
- [现在去修复你的超时bug！](#现在去修复你的超时bug)
- [注释](#注释)
- [翻译说明](#翻译说明)

# 简单的超时机制不支持抽象

最简单的处理超时的方式无疑是给每个可能阻塞的函数加上timeout参数。在Python标准库中，你可以找到像threading.Lock.acquire这样的函数：

```python
lock = threading.Lock()

# 等待最多10s以获取锁
lock.acquire(timeout=10)
```

如果使用socket网络模块，也是类似的用法，只是timeout会被设置在socket对象的属性上而不是在每次调用时传递。

```python
sock = socket.socket()

# 设置一次超时
sock.settimeout(10)
# 等待最多10s以建立和远程主机的连接
sock.connect(...)
# 等待最多10s以接收远程主机发送的数据
sock.recv(...)
```

这种方式比每次显式的传递timeout参数要更加方便一些(后面我们将会讨论这种方式的问题)，但重要的是要知道这只是换汤不换药罢了。在语义上和我们看到的threading.Lock是一样的：每次函数调用都会有自己的10秒超时。

这会有什么问题呢？目前一切看起来都足够直观。如果我们总是写一些直接操作这些底层API的代码，那么可能也足够高效。但是，编程是需要抽象的。假设我们想要从[S3](https://en.wikipedia.org/wiki/Amazon_S3)拉取一个文件，我们可能会使用boto3的[S3.Client.get_object](https://botocore.amazonaws.com/v1/documentation/api/latest/reference/services/s3.html#S3.Client.get_object)方法。S3.Client.get_object会做一些什么操作呢？它通过调用[requests](https://requests.readthedocs.io/en/latest/)库的方法发送一系列的HTTP请求到S3服务器。每次调用requests库的方法内部又会调用一系列的socket模块的方法来进行实际的网络通信[[1]](#注释)。

从用户的角度看，从远程服务拉取文件使用了3个不同的API：

```python
s3client.get_object(...)
requests.get("https://...")
sock.recv(...)
```

好了，上面3个方法调用处于不同的抽象级别，用户无需关心这些细节是抽象掉这些实现细节的出发点。所以，如果我们计划在所有地方使用timeout参数，我们需要这些方法都接收一个timeout参数：

```python
s3client.get_object(..., timeout=10)
requests.get("https://...", timeout=10)
sock.recv(..., timeout=10)
```

现在问题出现了：如果我们真的这么做，那么实际实现这些函数会很蛋疼。为什么呢？让我们看个简单的例子。处理HTTP响应的时候，当我们看到Content-Length头的时候，我们需要从网络读取很多字节的数据以获取实际的响应体。所以，requests的某个地方应该有如下这么一个循环：

```python
def read_body(sock, content_length):
    body = bytearray()
    while len(body) < content_length:
        max_to_receive = content_length - len(body)
        body += sock.recv(max_to_receive)
    assert len(body) == content_length
    return body
```

我们将对这段循环代码做一些改动以支持超时。我们希望这样说“我愿意最多等待10s的时间以读取整个响应体”。但是我们不能无脑地将timeout参数传递给recv方法。因为想象一下这种情况，我们总共有10秒钟的时间来完成所有操作。第一次调用recv用了6秒，供第二次recv调用的超时时间只有4秒。由于使用了timeout参数的方式，每当需要调用不同抽象层的代码时，我们都需要写一些恼人的垃圾代码来重新计算超时时间：

```python
def read_body(sock, content_length, timeout):
    read_body_deadline = timeout + time.monotonic()
    body = bytearray()
    while len(body) < content_length:
        max_to_receive = content_length - len(body)
        recv_timeout = read_body_deadline - time.monotonic()
        body += sock.recv(max_to_receive, timeout=recv_timeout)
    assert len(body) == content_length
    return body
```

(这部分代码其实已经算精简了，毕竟我们假设sock.recv方法会接收一个timeout参数。如果你想真的实现这部分代码，你需要在调用每一个socket方法前调用settimeout方法，然后可能还需要使用添加一些try/finally块来回滚或者处理影响你程序其他部分的风险点。)

实际上，没有人会这么干。我所知道的所有能接受timeout参数的更高一层的Python库都是只是原封不动的把timeout参数传给下一层。这样也就打破了抽象原则。举个例子，有两个今天你可能用过的Python API，而且他们看起来接受差不多的timeout参数:

```python
import threading
lock = threading.Lock()
lock.acquire(timeout=10)

import requests
requests.get("https://...", timeout=10)
```

但事实是这两个timeout参数的意思完全不一样。前者的意思是“尝试获取锁，10秒以后放弃”。后者的意思是“尝试访问指定的URL，但是在任意底层socket操作超过10秒后放弃”。你使用requests库的原因可能是因为你不想关心底层socket，但是抱歉，你无论如何还是需要关心。实际上，现在不可能保证requests.get方法会在一定时间内返回：如果某个恶意服务器或者行为不端的服务器每10秒只发送1字节的数据，那么我们上面的requests调用会一次次地重置这个超时时间，永远不会返回。

我这里不是故意挑requests的刺，Python API到处都是这样的问题。我使用requests作为例子是因为Kenneth Reitz以痴迷于让API尽可能直观和符合直觉而负有盛名，这是仅有的他没能做到这一点的地方。我想在requests的API中，这是[仅有的一处在文档中使用了提示框来警告你该参数是反直觉的](https://requests.readthedocs.io/en/latest/user/quickstart/#timeouts)地方。所以连Kenneth Reitz都不能正确处理这一切，我认为我们可以总结出“仅仅添加给API添加timeout参数”并不能让API符合人们的预期。

# 绝对的deadline是可组合的(但是用起来很麻烦)

如果timeout参数没用，我们有什么替代品呢？现在有一条部分人提倡的观点。考虑到上面read_body的例子，我们把传入的相对超时时间(从开始调用该函数的瞬间起10秒)转换成一个绝对时间(时钟时间为12:01:34.851)，然后再在每一次调用socket操作前再转换回相对时间。如果我们编写的API都使用deadline概念而不是超时，那么代码可以更加精简。因为你可以将deadline传递给底层抽象层，所以会让代码库的作者的工作简单一些。

```python
def read_body(sock, content_length, deadline):
    body = bytearray()
    while len(body) < content_length:
        max_to_receive = content_length - len(body)
        body += sock.recv(max_to_receive, deadline=deadline)
    assert len(body) == content_length
    return body

 # 总共等待10秒来下载整个响应体
 deadline = time.monotonic() + 10
 read_body(sock, content_length, deadline)
```

(一套很出名的像这样工作的API是[Go的socket API](https://golang.org/pkg/net/#Conn))

但是这种方式有个缺陷：该方法成功的将这些麻烦移出了底层库，但是将麻烦推给了使用API的用户。在使用超时机制的最顶层，库的使用者可能会想要这么描述”10秒之后放弃”，如果你只接受deadline参数，那么使用者得在所有地方都手动转换一遍。你也可以让每个函数同时接受timeout和deadline参数，不过这样你就需要在每个函数里加上一些模板代码对他们进行归一化，如果同时指定了这两个参数就报错等等。Deadline机制相对于原始的超时机制是个进步，但是似乎还是缺失了一些抽象。

# 取消令牌(Cancel tokens)

## 取消令牌封装了取消状态

这部分就是缺失的抽象：为了替代下面这种同时支持两个不同的参数的形式：

```python
# 用户这样调用
requests.get(..., timeout=...)
# 库代码会这样调用
requests.get(..., deadline=...)
# 我们这样来实现
def get(..., deadline=None, timeout=None):
    deadline = normalize_deadline(deadline, timeout)
    ...
```

我们可以把超时过期信息封装在一个有方便的构造函数的对象里去：

```python
class Deadline:
    def __init__(self, deadline):
        self.deadline = deadline

def after(timeout):
    return Deadline(time.monotonic() + timeout)

# 总共等待10秒钟以获取该URL
requests.get("https://...", deadline=after(10))
```

对于用户来说这样看起来就很棒、很自然了，但是因为内部使用了绝对超时时间，所以对于库的实现来说也很简单。

一旦我们走到这一步了，我们可以进一步抽象。毕竟，我们可能不止因为超时而想取消一个阻塞操作。“10秒钟以后就放弃”是“在<某个条件成立>后放弃”的一个特例。如果你使用requests库来实现一个浏览器，你想要表述”开始获取这个URL，不过在’停止’按钮被点击以后放弃”。代码库大多数情况下把Deadline对象当做黑盒传递给底层代码，并相信最终某些底层原语会将其进行正确解释。所以，不仅仅考虑把该对象作为deadline的封装，我们可以考虑把该对象作为任意的”现在需要放弃”的检查的封装。考虑到其更加抽象的事实，我们可以把它叫做CanelToken(取消令牌，后面遇到cancel token这种分开写的情况会翻译为取消令牌)而不是Deadline：

```python
# 抱歉，这个库是假想的
from cancel_tokens import cancel_after, cancel_on_callback

# 返回一个10秒以后进入取消状态的黑盒CancelToken对象
cancel_token = cancel_after(10)
# 这个请求会在10秒后放弃
requests.get("https://...", cancel_token=cancel_token)

# 返回一个在回调函数被调用后进入取消状态的黑盒CancelToken对象
cancel_callback, cancel_token = cancel_on_callback()
# 让'stop'按钮被点击时调用回调函数
stop_button.on_press = cancel_callback
# 这个请求会在'stop'按钮被点击后放弃
requests.get("https://...", cancel_token=cancel_token)
```

把取消条件升级到第一等(first-class)的对象让我们的超时API更加易用，同时也可以让其更加的强大。现在我们不仅仅可以处理超时，还可以处理任意的取消条件，编写并发代码时这也是一个很常见的需求。(举个例子，它可以允许我们进行这样的表述“同时发起两个重复的请求，当其中一个完成后立即取消另一个请求”。)这是一个伟大的点子。据我所知，这个想法最初来自于Joe Duffy在C#中运行的[cancellation tokens](https://devblogs.microsoft.com/pfxteam/net-4-cancellation-framework/)，另外Go语言中的Context对象也是基于同样的想法。这些老哥们真的是太聪明了！事实上，取消令牌还可以解决一些在传统取消系统中出现的一些问题。

## 取消令牌属于水平触发，可以根据你程序的需要来确定范围

在我们超时和取消API的小教程中，我们是从超时开始考虑的。如果你从取消开始考虑，那么还会有另外一种在很多系统中都会见到的常见模式：一个通过唤醒以及抛出异常来允许你取消一个线程(任务、或者你框架里的其他线程替代品)的方法。例子包括asyncio的[Task.cancel](https://docs.python.org/3/library/asyncio-task.html#asyncio.Task.cancel)，Curio的[Task.cancel](https://curio.readthedocs.io/en/latest/reference.html#Task.cancel)，pthread的cancellation，Java的[Thread.interrupt](https://docs.oracle.com/javase/8/docs/api/java/lang/Thread.html#interrupt--)，C#的[Thread.interrupt](https://msdn.microsoft.com/en-us/library/system.threading.thread.interrupt(v=vs.110).aspx)等等。这种情况，我把这叫做“线程中断”的方式来进行取消操作。

在线程中断方式中，取消是一个针对实体的即时事件：一次调用→一个线程/任务中出现一个异常。这样会有两个问题。

一是范围问题非常明显：如果你有一个函数你想正常调用，但是你可能需要取消这次调用，那么你就必须要创建一个新的线程/任务/其他替代品。

```python
http_thread = spawn_new_thread(requests.get, "https://...")
# 当'stop'按钮被点击后会调用http_thread.interrupt方法
stop_button.on_click = http_thread.interrupt
try:
    http_response = http_thread.wait_for_result()
except Interrupted:
    ...
```

这里并不是因为并发而使用线程。这仅仅是为了让你限制取消的范围而使用的一种撇足的方式。

另外，如果你有一堆复杂工作需要取消。比如，某些内部启动了大量工作线程的情况？在前面的例子中，如果requests.get创建了一些额外的后台线程，当我们取消了第一个线程，他们可能会被挂起。正确处理这个问题需要一些复杂而精细的簿记(代码)。

取消令牌就可以解决这些问题：取消令牌可以取消”任何传入了令牌的东西”。这些东西可以是一个函数、一个复杂的多层次线程池、或者任何他们两者之间的东西。

线程中断机制的另外一个问题更加微妙：把取消当做事件对待。从另一方面来说，取消令牌将取消封装成了一个状态：它们初始为未取消状态，最终转换为取消状态。

这挺微妙，但是这让取消令牌更不容易出错。一种会这样想的原因是[边缘触发和水平触发的区别](https://lwn.net/Articles/25137/)：线程中断API为取消提供了边缘触发的通知，相对的取消令牌提供了水平触发。边缘触发API使用起来是出了名的棘手。你可以在Python的[threading.Event](https://docs.python.org/3/library/threading.html#threading.Event)看到这么一个例子：虽然threading.Event被叫做”事件”，实际上其内部有一个布尔状态，使用取消令牌进行取消就像设置这么一个事件。

这样讲其实很抽象，我们让它更加具体一点。考虑一个确保连接正常关闭的使用try/finally的常见模式。有这么一个例子，一个函数创建一个WebSocket连接，发送消息，然后关闭该连接，不管send_message是否会抛出异常[[2]](#注释)：

```python
def send_websocket_messages(url, messages):
    open_websocket_connection(url)
    try:
        for message in messages:
            ws.send_message(message)
    finally:
        ws.close()
```

现在我们假设开始运行这个函数，但是在某个时间点，另一端断开了网络连接，然后我们的send_message调用就一直处于挂起状态。最终，我们不想等了，然后就要取消这个操作。

使用线程中断式的边缘触发API的话，这会导致send_message调用立即产生一个异常，接下来我们的连接清理代码会自动运行。目前为止还很好。但是这里关于websockt协议有一个有趣的事实：它有[一条“close”消息](https://tools.ietf.org/html/rfc6455#section-5.5.1)需要你在关闭连接前发送出去。总的来说这是一件好事，这样可以让关闭操作更加彻底。所以，当我们调用ws.close时，它会尝试发送这条消息。但是，这个例子中，我们尝试关闭连接是因为我们已经不认为另一方会接收任何新的消息。那么现在ws.close调用也会一直挂起。

如果我们使用取消令牌，这个问题将不会发生：

```python
def send_websocket_messages(url, messages, cancel_token):
    open_websocket_connection(url, cancel_token=cancel_token)
    try:
        for message in messages:
            ws.send_message(message, cancel_token=cancel_token)
    finally:
        ws.close(cancel_token=cancel_token)
```

一旦取消令牌被触发，将来所有基于该令牌的操作都会被取消，所以ws.close调用不会被卡住。这是一个更不容易出错的范式。

有趣的是，如此多的旧接口都会这样出错。如果你按着这篇博客中所做的一路走下来，开始思考把超时应用到由多个阻塞调用组成的复杂操作中。那么显然，如果第一个调用用光了所有的时间，那么后面的任何调用都应该立即失败。超时自然是水平触发的。接下来当我们把超时泛化到任意方式的取消，其仍然有效。但是，如果你仅仅为原语操作考虑超时，那么永远达不到这样的效果。又如果你直接使用通用的取消API(译注: 这里指Task.cancel这种API)来实现超时(如twisted和asyncio)，那么就很容易错失水平触发取消机制的优势。

## 实际上因为人类的懒惰取消令牌并不可靠

所以取消令牌有如此好的语义，也确实比超时和deadline更好，但是取消令牌有一些使用上的问题：为了编写支持取消的函数，你需要接受模板式的参数并将其传递给你所调用的子程序。然后还需要记住，一个正确且健壮的程序需要在每一个进行I/O操作的函数、栈中的每一帧都支持取消。一旦你偷懒、不管或者忘记把取消令牌传递给调用的子程序，那么你就会搞出一个潜在的bug。

人类讨厌这样的模板。我的意思是，不包括你，我相信你是个会确保每一个函数都正确支持取消并且flossess every day(这句真的不知道怎么翻译了-_-!)的勤奋程序员。但是可能你的一些同事并不勤奋？或者你可能需要依赖一些其他人写的库？你有多相信第三方库会正确运作？随着调用栈的增长，每个人的每部分代码都正确运作的概率会趋近于零。

我能举几个真实的例子来支持前面的论点吗？Emmm，在C#和Go这两个最出名的提供并且一直提倡使用取消令牌的语言中，其底层网络原语仍然没有提供取消令牌的支持[[3]](#注释)。这就像…底层操作会因为某些你无法控制的原因挂起，你需要准备好超时和取消。但是…我猜他们只是还没有开始实现这项功能。或者，他们的socket层只支持那种老式的在socket对象上设置[timeout](https://learn.microsoft.com/en-us/dotnet/api/system.net.sockets.socket.receivetimeout?redirectedfrom=MSDN&view=net-7.0#System_Net_Sockets_Socket_ReceiveTimeout)和[deadline](https://pkg.go.dev/net#IPConn.SetDeadline)的机制。如果你想使用取消令牌(或者说context，Go的说法)，那么你得自己弄清楚如何桥接这两种系统。

Go标准库确实给出了一个该怎样做的例子：他们建立网络连接(和Python的socket.connect差不多)的函数确实支持接受一个取消令牌。实现这个函数用了[40行代码](https://github.com/golang/go/blob/bf0f69220255941196c684f235727fd6dc747b5c/src/net/fd_unix.go#L99-L141)和一个后台任务。第一版代码还有个[花了一年时间才在生产环境中找出来的竞争条件](https://github.com/golang/go/issues/16523)。

我不是想开玩笑。这玩意儿很难。但C#和Go是由专业的全职开发者组成的团队在维护和财富50强公司支持的巨型项目。如果他们都不能搞定，谁能？不是我，我只是一个尝试重新实现Python I/O的凡人。我不能承受把事情搞的如此复杂。

# 取消域: Trio关于超时和取消的人性化方案

还记得在本篇博客的开始，我们提到Python的socket相关方法没有使用独立的timeout参数，但是取而代之的是允许你在socket对象上设置超时时间，然后隐式的把这个超时时间传递每个你调用的方法吗？在前面一章，我们提到C#和Go也做了差不多一样的事？我想他们可能在寻找一些新的办法。也许我们应该接受，当你有些需要每次调用方法都要传递的数据，这应该是计算机需要干的事，而不是让易出错的人类来做这项工作——不过是以一种支持复杂抽象的通用方式来做，而不仅仅只是处理一下socket。

## 取消域(Cancel scope)是如何工作的

下面是在Trio中如何给一个HTTP请求添加10秒超时的例子：

```python
# 原语API
with trio.open_cancel_scope() as cancel_scope:
    cancel_scope.deadline = trio.current_time() + 10
    await request.get("https://...")
```

当然正常情况下你应该用一个更方便的[包装函数](https://trio.readthedocs.io/en/latest/reference-core.html#trio.move_on_after)，像这样：

```python
# 等效但更加地道的方式:
with trio.move_on_after(10):
    await requests.get("https://...")
```

但由于这篇文章是关于底层设计的，我们将会集中精力在原语(设计)上。(致谢：使用with块来实现超时这个想法是我第一次看Dave Beazley的Curio库时发现的，虽然我改了很多。我把相关细节放在了脚注[[4]](#注释)中。)

你应该将with open_cancel_scope()视为创建了一个取消令牌，但是实际上它并没有对外暴露任何CancelToken对象。相反的，取消令牌被压入了一个不可见的内部栈中，并自动应用到with语句块中调用的所有阻塞操作中。这样requests就不需要透传任何东西，当它最终向网络发送或者从网络接收数据时，那些原语调用会自动应用deadline。

cancel_scope对象可以让我们控制取消状态：你可以修改deadline、通过调用cancel_scope.cancel()进行显式取消[等其他操作](https://trio.readthedocs.io/en/latest/reference-core.html#trio.The%20cancel%20scope%20interface)。如果你会C#，那么它其实和[CancellationTokenSource](https://learn.microsoft.com/en-us/dotnet/api/system.threading.cancellationtokensource?redirectedfrom=MSDN&view=net-7.0)相似。一个实用的骚操作是它能在更基础的扩展语义的取消域上实现人们熟悉的[超时后抛](https://github.com/python-trio/trio/blob/07d144e701ae8ad46d393f6ca1d1294ea8fc2012/trio/_timeouts.py#L96-L118)错误的API。

当一个操作被取消了，它会产生一个Cancelled异常，这个异常被用来让栈回到正确的with open_cancel_scope块上。取消域可以嵌套，Cancelled异常知道哪一个域触发了他们，然后会不断向上传播直到到达对应的with语句块。(因此，你应该始终让Trio的运行时来管理cancelled异常的触发和捕获，以便它可以正常的追踪这些关系。)

支持嵌套非常重要，因为一些操作可能希望把内部使用超时作为实现细节。举个例子，当你让Trio创建一个到具有多个IP地址的主机的TCP连接，Trio会使用“[Happy Eyeballs](https://en.wikipedia.org/wiki/Happy_Eyeballs)”算法以[交错开始的方式来并行地尝试创建多个连接](https://trio.readthedocs.io/en/latest/reference-io.html#trio.open_tcp_stream)。这样就需要一个[内部超时](https://github.com/python-trio/trio/blob/d063d672de15edc231b14c0a9bc3673e5275a9dc/trio/_highlevel_open_tcp_stream.py#L260-L265)来决定什么时候初始化下一次连接尝试。但是用户并不需要关心这个细节！如果你想表述“尝试连接example.com:443，10秒以后放弃”，那么代码应该如下：

```python
with trio.move_on_after(10):
    tcp_stream = await trio.open_tcp_stream("example.com", 443)
```

这样就一切正常了。多亏了取消域嵌套规则，事实证明open_tcp_stream无需额外代码即可正确处理此问题。

## 我们需要在哪些地方检查取消？

面向取消编写正确的代码会比较棘手。如果突然出现一个Cancelled异常，用户正好也没有捕获它，也许当时用户的代码正在处理一些微妙的数据结构，这个异常可能会破坏内部状态并且产生难以追踪的bug。另一方面，如果你没能及时地注意到操作被取消，那么超时和取消系统也不会有太大用。所以，对于任意一个系统来说，一个重要的挑战是列出一些“金科玉律”，这些规范需要经常被检查但是又不能太频繁，然后通过某种方式把这些“金科玉律”告知用户以确保他们的代码万无一失。

在使用Trio的场景中，这非常简单。因为一些其他的原因，我们已经使用Python的async/await语法来注释(annotate)阻塞函数。这主要的作用是让你阅读函数的代码即可立刻确定哪些地方会阻塞等待事件发生。例子：

```python
async def user_defined_function():
    print("Hello!")
    await trio.sleep(1)
    print("Goodbyte!")
```

这里我们可以看出trio.sleep会阻塞，因为它前面有一个特殊的await关键字。你不能不使用该关键字来调用trio.sleep(或者其他任何Trio内置的阻塞原语)，因为这些函数被标注成了异步函数。如果你想使用await关键字，那么你也必须将调用函数标注成异步的，这是Python强制要求的，这也意味着所有调用该用户自定义函数(user_defined_function)的函数也需要使用await关键字。这是说的通的，因为如果一个用户自定义函数调用一个阻塞函数，那么这个函数也会成为一个阻塞函数。在很多其他系统中，一个函数会不会阻塞，你只能通过检查所有潜在的被调用函数、被它们调用的函数等来进行确认。async/await让其成为全局运行时属性，并让它们一眼就可以在源码中看出来。

Trio的取消域基于(piggy-back on)该系统：我们声明，不管何时你看到一个await关键字，这就是你可能需要处理Cancelled异常的地方。要么因为它直接调用了Trio中直接检查取消的原语，要么因为它调用了一个间接调用这些原语的函数，从而可能会看到Cancelled异常冒出来。这样做会有几个优点。向用户解释起来非常简单。它涵盖了你绝对需要超时/取消支持以避免无限挂起的所有函数——只有阻塞函数(functions that block)才会永远阻塞。这意味着任何定期执行I/O的函数也会定期的进行取消检查，所以大多数时候你不需要担心这一点(尽管对于偶尔运行的长时间纯计算，你可能想要通过调用await trio.sleep(0)来添加一些显式的取消检查，无论如何你必须要这样做才能让调度器工作！)。阻塞函数往往有[多种故障模式](https://docs.python.org/3/library/exceptions.html#os-exceptions)，所以在很多场景中，处理Cancelled异常所需的清理代码也会在处理其他异常时用到，比如说处理表现异常的网络对点。Trio的协作式多任务系统也使用await来标记调度器可能需要切换到另一个任务的地方，所以你必须要小心不要在横跨await时让数据结构处于不一致的状态。取消和async/await就像花生酱和巧克力一样形影不离。

## 一个逃逸口(escape hatch)

虽然默认在所有阻塞原语调用时做取消检查很好，但是有非常少的场景，你会想要禁用它并明确的控制取消。这些场景太罕见了，以至于我没法举出一个简单的例子(但是在Trio源码中有一些神奇的例子，你可以自己grep找出来)。为了提供这种逃逸口, 你可以设置一个取消域以从外部的取消中“保护(shield)”其内容。就像这样：

```python
with trio.move_on_after(10):
    with trio.open_cancel_scope() as inner_scope:
        inner_scope.shield = True
        # 睡眠20秒, 忽略整体的10秒超时
        await trio.sleep(20)
```

为了支持组合，shielding对于取消域栈来说非常敏感：它只阻止外部的取消域不被应用，不会对内部的取消域生效。在我们上面的例子中，我们的shield不会对trio.sleep中可能用到的任何取消域有任何影响，这些取消域仍然会正常工作。这样很好，因为无论如何trio.sleep内部怎么做是其自身的实现细节。而且事实上，trio.sleep确实在内部[使用了一个取消域](https://github.com/python-trio/trio/blob/07d144e701ae8ad46d393f6ca1d1294ea8fc2012/trio/_timeouts.py#L65-L66)!
[[5]](#注释)

shield是取消域的属性而不是专门的”shield域“的一个原因是这样实现这种嵌套会比较方便，因为我们可以重用取消域线程的栈结构。另外一个原因是，任何一个你禁用外部超时的地方，你都需要想想你要怎么做以确保程序不会永远挂起，而且有个取消域在那里可以很容易的应用一个新的在当前代码控制下的超时：

```python
# 演示一下屏蔽scope可以用在避免在禁用外部超时后引起的挂起
with trio.move_on_after(10) as outer_scope:
    with trio.move_on_after(15) as inner_scope:
        inner_scope.shield = True
        # 当shielding scope超时后，会在15秒后返回
        await trio.sleep(1000000)
```

现在如果你是一个Trio用户那么请忘记你看了这部分内容。如果你认为你需要用到shielding那么你几乎肯定应该重新考虑你在尝试做什么。但是如果你是一个在添加取消域支持的I/O运行时实现者，那么这是一个重要的功能。

## 取消域与并发

最后，还有一个Trio的功能需要在这里被提及。本文到现在为止，我还没有讨论太多并发的事。超时和取消在很大程度上是独立的，前面讨论的东西都可以直接应用到单线程同步代码中。但是我们做了一些看似不重要的假设：如果你在with块中调用了一个函数，那么 (a) 函数实际将会在with块的内部运行，以及 (b) 函数抛出的任何异常都会传递回with块以让其捕获这些异常。不幸的是很多多线程和并发库违反了这一点，特别是某些任务被创建或者调度的场景：

```python
# 这看起来完全没问题(innocent enough)
with move_on_after(10):
    do_the_thing()

# 但是并不是这样的
def do_the_thing():
    # 使用一些像大多数系统使用的封装(made-up)API
    start_task_in_background(some_worker_that_will_actually_do_the_thing)
```

如果我们只是单独看with块，这看起来会完全没问题。但是当我们看到do_the_thing是如何实现的，我们意识到在后台任务完成前我们很可能已经退出了with块，所以这里会出现歧义：是否需要将超时应用到后台任务？如果应用到了后台任务，那么我们该怎么处理Cancelled异常呢？在大多数系统中，未处理的后台线程/任务的异常会被简单地忽略掉。

然而，因为Trio其独特的并发方式，并不会出现这些问题。Trio的[nursery system](https://trio.readthedocs.io/en/latest/reference-core.html#tasks-let-you-do-multiple-things-at-once)意味着子任务总是会被集成进调用栈，最终会形成一颗调用树。具体来说，Trio没有全局的start_task_in_background原语，强制使用nursery这种方式。相反的，如果你想创建一个子任务，你必须要先创建一个”nursery”块(以让[孩子(任务)生存](https://www.dictionary.com/browse/nursery)，明白吗?)，那么子任务的生命周期就和创建nursery的with块绑定在了一起：

```python
with move_on_after(10):
    await do_the_thing()

async def do_the_thing():
    async with trio.open_nursery() as nursery:
        nursery.start_soon(some_worker_that_will_actually_do_the_thing)
        # 现在直到子任务结束前，'async with'块不会完成。而且如果子任务出现了
        # 一个未处理的异常，那么该异常也会在父任务中重新产生。这样这个例子就会
        # 显得很傻，“后台任务”表现起来就像一个函数调用，这就是(nursery sytem的)要点。
```

这套系统有很多优势，但是这里(和本文)相关联的一点是其保留了取消域所依赖的关键假设。任何一个给定的nursery要么在取消域内，要么在取消域外，我们可以通过检查with open_cancel_scope块是否包含async with open_nursery来进行判断。接下来这点就非常明确了，如果nursery在一个取消域中，那么取消域就应该应用到nursery中的所有子任务。这意味着如果我们对一个函数应用了超时，那么它不能通过创建子任务从该超时中”逃逸(escape)”，超时也会被应用到子任务中去。(例外就是你把外部的nursery对象传给函数，那么该函数可以在该nursery中创建子任务，这样就可以从超时中逃逸。但是这对于调用者来说显而易见，因为他们必须要提供一个nursery对象，这关键是要弄清楚发生了什么，并不是为了让它们不能创建子任务。)

## 小结

回到我们最初的例子：我在做一些把requests迁移到Trio上运行的初步工作([你可以帮忙！](https://github.com/python-trio/hip/issues/1))，目前看起来Trio版本不仅比传统同步版本能更好的处理超时，而且它能零代码做到这一点，所有你想要检查取消的地方都是会由Trio自动处理的地方，所有你需要特别关注的处理异常的地方都是requests因为其他原因来处理其他异常的地方。

天下没有白吃的午餐，处理取消仍然是出现bug的地方，而且需要在写代码的时候多加注意。但是Trio的取消域比其他我找到的系统都更加容易使用，而且也更加可靠。希望我们可以让超时bug成为例外而不是常例(rule)。

# 还有哪些地方可以从取消域中受益？

如果你使用的是Trio的话，那太好了。这(文中的想法)是只能在Trio的环境中使用，还是更普遍？ 需要进行什么样的调整才能在其他环境中使用它？

如果你想要实现取消域的话，那么你需要：

- 某种隐式上下文本地存储来追踪取消域栈。如果你使用线程，那么thread-local存储可以用。如果你使用的是更加奇特的东西，那么你需要找到你使用的系统的(存储)等价物。(举个例子，在Go语言中，你需要goroutine-local存储，众所周知它[不存在](https://stackoverflow.com/questions/31932945/does-go-have-something-like-threadlocal-from-java)。)这可能有点棘手，比如Python中的例子，我们需要[PEP-568](https://peps.python.org/pep-0568/)来消除[取消域和生成器之间](https://github.com/python-trio/trio/issues/264)的一些有问题的交互。
- 一种能够划定取消域边界的方法。Python的with块就能很好的工作。其他方法包括专有的语法，或者将取消域限制在单独的函数调用中，如with_timeout(10, some_fn, arg1, arg2)(不过这样会强行进行一些尴尬的代码分解(awkward factorings), 并且你需要想出一些方法来公开取消域对象)。
- 一条在超时/取消发生后将栈回退到合适的取消域的策略。异常很好用，只要你有办法在取消域的边界捕获到它们，这也是另一个Python with块能很好用的原因。但是如果你使用的语言使用了诸如错误码而不是异常，那么我相信你可以从中构建出一些栈回退的协议。
- 一则怎样将取消域集成进你的并发API(如果有的话)的story。当然理想的是像Trio的nursery系统一样(nursery系统还有很多其他的优势，但这需要另一篇博客来写了)。但即使没有它，你可以考虑在取消域中创建的子任务都继承该取消域，而不用考虑子任务何时结束。(除非他们考虑使用shielding功能之类的东西来退出取消域。)
- 一些确定哪些操作是可取消的约定，然后将这些规则告知用户。如前面所提到的，async/await非常适合，但是如果你没有使用async/await，那么其他一些约定也是可以的。具有丰富静态类型系统的语言也可能以某种方式利用他们。最坏的情况也就是你小心地用文档标注每一个函数。
- 取消域和你所有关心的阻塞I/O原语的集成。如果你是从头开始构建系统，那么这点非常简单。异步系统在这里有一项优势，因为把所有东西集成进事件循环(event loop)已经强制你以某种统一的方式重新实现你所有的I/O原语，这同时给了你一个很好的机会来添加统一的取消处理。

## 同步、单线程Python

我们最初的例子用到了requests，requests是一个常规的同步库。前面几乎所有内容都同样适用于同步或者并发代码。所以我认为探索在经典同步Python中运用这些想法很有趣。可能我们可以修复requests，这样requests就不需要因为其timeout参数而道歉了。

我们需要接受以下一些限制：

- 取消域不会无处不在, 代码库必须确保它们只使用了”启用取消域(scope-enabled)”的阻塞操作。可能从长远来看，我们可以想象取消域成为标准库的一部分，并且能够集成进标准原语中，但是即使是这样，也仍然会有一些三方扩展库会使用自己的I/O操作而不是使用标准库的I/O操作。不过另一方面，像requests这样的库可以小心的只使用启用取消域(scope-enabled)的库，然后标注自己本身也是启用域的。(这可能是Trio这种异步库在超时和取消方面最大的优势了：作为异步使用(being async)并没有什么不同，但是异步库必须要重新实现所有的基础I/O原语以将其集成进I/O循环中(I/O loop)。如果你无论如何都要重新实现所有操作，那么就很容易让支持取消保持一致。)
- 同步单线程Python中没有类似await这样的标识符来标记操作可取消。这意味着用户必须更加小心，并且确认每个函数的说明，但是这仍然比现在让timeout参数正常工作所花费的功夫要少。
- Python的底层同步原语整体上只支持基于超时的取消，不能支持任意事件，所以我们可能不能提供一个cancel_scope.cancel()操作。但是这个限制并不是大问题，因为如果你有一个同步单线程程序，这唯一的线程已经在某些阻塞操作中挂起，这样哪个线程还可以调用cancel()函数呢？

总结一下：它不可能和Trio提供的(环境)一样好，但是它仍然会非常有用，而且肯定会比我们现在已有的方式好用。

如果你对此感兴趣，[你可以看一下我实现的POC代码](https://github.com/njsmith/deadline-scopes)。

## asyncio

这篇博文最初的动机之一是和Yury讨论我们是否可以将Trio的改动运用到asyncio中去。透过前面的分析来看asyncio，有几件事让我们眼前一亮：

- 取消域模型是一种隐式状态的任意范围的取消令牌，该模型和asyncio当前的面向任务的、边缘触发的取消(然后Future层也有稍微不同的取消模型)有一些不匹配(impedence mismatch)，所以我们需要一些如何将他们融合在一起的story。又或者可以将任务迁移到带状态的取消模型？
- 如果没有nursery系统的话，就没有可靠的方式对取消进行跨任务传播，而且还有很多不同的在不同抽象层次的有些像创建子任务的操作(比如loop.call_soon)。你可以建立一条所有子任务总是继承他们创建者的取消域的规则，但是我不确定这是否是一个好想法，这需要在考虑一下。
- 如果没有一种将异常传播回调用栈的通用机制的话，就没有可靠的方式把Cancelled异常传递回最初的域，通常asyncio只是简单地打印并丢弃掉来自任务的未处理的异常。这样或许挺好的？

不幸的是，asyncio处在一个比较尴尬的位置上，因为它建立在一个基于前10年在Python中使用异步I/O经验的架构之上。在那架构被固定以后，又往Python中添加了一些让这些经验失效的语法。但是仍有可能在一些妥协下接受一些本文中的想法。

## 其他语言

如果你在使用其他语言，我很想听听取消域的想法是如何运用的，如果有的话。比如，对于不使用异常的语言，或者缺少Python with块提供的用户扩展语法的语言，它肯定需要做一些调整。

# 现在去修复你的超时bug！

或者你想继续了解更多关于Trio的信息，我们有[一篇人们可能会喜欢的指引文档](https://trio.readthedocs.io/en/stable/)。

# 注释

你可以[在Trio论坛讨论这篇博文](https://trio.discourse.group/t/discussion-timeouts-and-cancellation-for-humans/26)

1. 事实上我在这里掩盖了好几层抽象：其实它更像是这样 boto3→botocore→requests→urllib3→http.client→socket
2. 实际上我们可能会在这里使用with表达式，像这样：
    
    ```python
    def send_websocket_messages(url, messages):
        with open_websocket_connection(url) as ws:
            for message in messages:
                ws.send_message(message)
    ```
    
    这样会让问题更难以发现，因为现在让我们程序挂起的讨厌的ws.close调用完全不可见了。
    
3. C#的顶层异步网络函数实际上会接受取消令牌参数然后忽略他们，这简直难以置信。
4. Curio的超时继承自一种线程中断风格的取消模型(类似于Java/C#中的Thread.interrupt)，所以超时过期是边缘触发的，[在我向Dave吐槽前](https://github.com/dabeaz/curio/issues/82)根本不支持嵌套，而且只会应用到当前任务，不会应用到任何它可能创建的子任务。Trio的取消域基本上是Cruio的超时块 + C#的取消令牌 + 一个更直接的嵌套模型 + 屏蔽 + 使子任务遵守栈规范的基于nursery的并发。继续阅读以了解这些东西都是什么:-)
5. 可能有趣的上下文：在其他系统中，常常会有一些像”call_later”这种原语，这会在指定时间调度执行一些代码。(这是一种注册任意代码以在某个特定事件触发后运行的“回调模型”的一种特例。)在这种系统中，你可能会希望取消域deadlines被设计成像call_later(deadline, scope.cancel())这样。但Trio的核心设计原则之一就是反对所有的回调范式，理由是这是一种伪装的产生后台任务的方式，而我们认为没有伪装后台任务的并发已经足够难了。所以在Trio中，实现call_later这样的功能的方式是创建一个子任务，然后让其休眠到指定的时间。事件通知总是通过唤醒一个任务来完成，而不是创建一个新的任务。这意味着取消域deadlines是Tiro计时的核心原语！其他所有像sleep这种时间操作都是基于取消域deadlines上实现的。

# 翻译说明

- innocent: 原意为无辜、无知，但是感觉用起来有点奇怪，就翻译成了没问题。
- impedance mismatch: 原文中为impedence mismatch，可能是个typo。原意为阻抗不匹配，翻译时只保留了不匹配。



