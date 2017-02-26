---
title: 【译】在Elixir中创建riak_core应用（3）
date: 2017-02-26 21:44:56
tags:
  - elixir
  - erlang
  - riak_core
  - 分布式
categories:
  - 翻译
---

因为网上关于elixir使用riak_core的文章很少，正好elixirstatus又推送了这几篇，于是就抱着正好学英语心态翻译了一下，
对于我这种英文水平，弄这个确实费了不少功夫，不过倒是培养了对洋文内容的耐心。看到英文不再那么心浮气躁了。
以至于今天早晨竟然躺在床上看完了两篇挺长的讲phoenix的英文文章。

也许是我水平的原因吧，这个系列的第三篇理解起来完全没有床上的那篇顺利，我甚至产生了一种作者没有在认真写的错觉。
所以决定以后就不翻译这种公共博客平台上的文章了，感觉总体上质量不如那些自建博客的。


### 原文：[Create a riak_core application in Elixir (Part 3)](https://medium.com/@GPad/create-a-riak-core-application-in-elixir-part-3-8bac36632be0)

这篇有的地方简直不知所云，尤其是讲`handle_handoff_command`的那一段，说了第一点，然后就找不着第二点了。
导致我越来越摸不着头脑。想了想还是把这篇烂翻译放出来吧，好在代码比较容易看懂，有点背景知识连蒙带猜还是能知道大概的。


## 在Elixir中创建riak_core应用（3）

嗨，欢迎回来！这已经是本系列的第三篇文章了。在[上一篇](https://medium.com/@GPad/create-a-riak-core-application-in-elixir-part-2-88bdec73f368)
文章中，我提出了这样几个问题：

* 如果一个节点使用`:riak_core.leave`离开集群，会发生什么？
* 如果某个节点崩溃，会发生什么？
* 如何获取所有的键？

本文中，我们将尝试回答第一个问题，*如果一个节点退出集群，会发生什么？*

![Classic riak_core cluster](https://cdn-images-1.medium.com/max/1600/0*9bU3ClL1A46_SjCe.)

或许你在阅读本文之前，已经做了实验，那么你会发现一些数据丢失了。同样，添加节点也会导致这个问题，为什么？

### 移交（Handoff）

还记得数据在集群中是如何组织的么？固定数量的虚拟节点（vnode）平均的分布在大量的物理节点之上，
每个vnode管理着键空间上的一个分区（partition）。当只有一个物理节点时，所有的虚拟节点都分布在该节点上，
当添加一个（物理）节点后，一部分虚拟节点就会“移动”到这第二个节点上来。现实中，这个过程有点儿复杂，
因为虚拟节点并不会自己移动，第一个节点启动后，它会启动所有的虚拟节点，然后第二个节点加入，
这时，两个节点会同时运行所有的虚拟节点。Riak_core会执行这样一个过程：杀死每个物理节点上不必要的虚拟节点，
同时把数据从一个物理节点移动到另一个物理节点。这个过程被称为移交（handoff）。可以在Riak_core的[wiki](https://github.com/basho/riak_core/wiki/Handoffs)查看更详细内容。

如果你还记得，在[vnode的行为](https://github.com/basho/riak_core/blob/develop/src/riak_core_vnode.erl#L95)（behaviour）中
有一些专门用来管理移交的函数，riak_core就是通过调用这些函数来管理本文开头的那种情况的，让我们来填上*正确的*代码，开始吧！

![fill-code](https://cdn-images-1.medium.com/max/1600/0*nZYEA6HjAS7uHN4r.jpg)

#### handoff_starting(dest, state)
该函数是riak_core调用的第一个函数，表示移交开始。（目标）vnode应该返回`{true, state}`来继续移交过程并管理vnode的状态。
如果有任何原因导致vnode无法管理移交过程，那么应该返回false并推迟移交。

#### handoff_cancelled(state)
如果有什么原因导致移交过程停止，该函数将会被调用。

#### is_empty(state)
riak_core调用该函数来评估vnode是否含有数据。如果vnode返回`{:true, state}`即表示不存在数据，
那么就会立即调用`delete`函数（来执行删除），否则*真正的*移交过程会从`handle_handoff_command`开始

#### handle_handoff_command(request, sender, state)
该函数在移交中被调用主要用来管理两种情况，第一种是将数据从源物理节点传输到目标节点，
这时调用该函数使用的`request`参数是一个类型为`riak_core_fol_req_v2`的记录（record）。
为了能够操作这个记录，我们需要从Erlang的include文件中导入该record的定义。
为此，我们需要使用[Record](https://hexdocs.pm/elixir/Record.html)模块，代码如下：

```elixir
defmodule NoSlides.VNode do
  require Logger
  @behaviour :riak_core_vnode

  # ...

  require Record
  Record.defrecord :fold_req_v2, :riak_core_fold_req_v2, Record.extract(:riak_core_fold_req_v2, from_lib: "riak_core/include/riak_core_vnode.hrl")

  def handle_handoff_command(fold_req_v2() = fold_req, _sender, state) do
    Logger.debug ">>>>> Handoff V2 <<<<<<"
    foldfun = fold_req_v2(fold_req, :foldfun)
    acc0 = fold_req_v2(fold_req, :acc0)
    acc_final = state.data |> Enum.reduce(acc0, fn {k, v}, acc ->
      foldfun.(k, v, acc)
    end)
    {:reply, acc_final, state}
  end

  def handle_handoff_command(request, sender, state) do
    Logger.debug ">>> Handoff generic request <<<"
    handle_command(request, sender, state)
  end

  # ...

end
```

在第8行，从riak_core的头文件中导入了record的定义。然后，我们使用已定义的record作为第10行中进行模式匹配的函数的目标。
这个函数的作用只是在移交过程中转移数据。这个record包含两个重要的部分，`foldfun`和`acc0`，`foldfun`应当包含一个被每条移动的数据调用的函数。
如果你知道[reduce](https://en.wikipedia.org/wiki/Fold_%28higher-order_function%29)（或是fold）的概念，这个过程应该很熟悉。
循环集合（collection）中的数据，从一个预先定义的值开始 将数据累积在一个变量中。我们的例子中，初始值是从接收到的record中提取出来的`acc0`。
14行到16行，我们从`state.data`中获取vnode的所有数据，然后交给`Enum.reduce/3`处理。这里依然使用`acc0`作为初始值。最后，
我们定义了一个匿名函数接收键值对和累加值，并把键值对和累加值传递给`foldfun`，将这个运算的结果作为这个函数的最终返回值。

在这个操作中，如果vnode接收到一个普通命令，比如`get`或者`put`，riak_core将路由这些请求，
把他们发往`handle_handoff_command`函数而不是使用`handle_command`。在我们这里，通过模式匹配识别出这种情况，然后直接调用`handle_command`，在这个过程中，riak_core接收数据并传给`foldfun`，
为了将数据传输到目标物理节点，另一个函数会被用来把数据转换为二进制形式，。这个函数是encode_handoff_item。

#### encode_handoff_item(k, v)
这个函数很简单，要求对数据进行编码以便传输，其他节点得到数据后再进行解码，我们可以简单的填上：
```elixir
:erlang.term_to_binary({k, v})
```

#### handle_handoff_data(bin_data, state)
在另一个节点上接受到的数据，用如下的Erlang的函数来解码数据：
```elixir
{k, v} = :erlang.binary_to_term(bin_data)
```

#### handoff_finished(dest, state)
当该函数执行，表示移交过程已经结束。如果一切顺利，应当返回`{:ok, state}`。它在`delete`函数和`terminate`函数之前执行的。

#### delete(state)
该函数在vnode被终止之前被调用，我们应当删除所有数据，在我们的例子中，因为没有存储任何东西，所以返回的state应当采用下面的形式：
```elixir
{:ok, Map.put(state, :data, %{})}
```

#### terminate(reason, state)
这是整个移交过程的最后一个函数，可以输出日志来记录终止的原因，在我们的例子中，应当为`:normal`


### 总结
正如你看到的那样，在发送端节点上调用了很多的函数，而在接收端节点上，只有`handle_handoff_data`被调用。
要记住，`handle_handoff_command`更为重要一些，因为它接收封装在fold_req_v2记录中的数据，但也要记得它也能接收普通的命令。

另一个有趣的事情就是函数`handoff_started`，vnode可以选择导出这个函数，这样它才会被调用，只有在这种情况下，当移交开始，vnode才能够停止移交并返回`{:error, reason}`。

### 参考：
我们的所有实现都放在了这个[Repo](https://github.com/gpad/no_slides)，另外还有一些basho发表的关于如何管理移交的有趣的文章：

* http://basho.com/posts/technical/understanding-riak_core-handoff/
* http://basho.com/posts/technical/understanding-riak_core-visitfun/
* http://basho.com/posts/technical/understanding-riak_core-building-handoff/

在下一篇中，我将讨论“如何获取所有的键”，我们会在最后一篇来讨论怎样管理崩溃节点。

如果你又问题或者不清楚的地方，请在下面留言。下次再见！
