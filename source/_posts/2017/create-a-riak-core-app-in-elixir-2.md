---
title: 【译】在Elixir中创建riak_core应用（2）
date: 2017-02-23 23:11:49
tags:
  - elixir
  - erlang
  - riak_core
  - 分布式
categories:
  - 翻译
---

越翻越觉得汗，水平太臭，一边翻译一边就能发现自己的错误，真怕误人子弟，建议对照英文原文看。


### 原文：[Create a riak_core application in Elixir (Part 2)](https://medium.com/@GPad/create-a-riak-core-application-in-elixir-part-2-88bdec73f368)


## 在Elixir中创建riak_core应用（2）

首先，感谢所有读过我[之前一篇](https://medium.com/@GPad/create-a-riak-core-application-in-elixir-part-1-41354c1f26c3)文章的人，
在这篇文章中，我们将继续使用前一篇文章中的代码，并加一些新的功能，你可以在[这里](https://github.com/gpad/no_slides)找到最终版本。

我们将添加的第一个功能是经典的Ping。

![If you played the original then you are old as I am :-)](https://cdn-images-1.medium.com/max/1600/0*p1k9GZI8FK0vUMyt.jpg)

开始之前，快速回顾一下，我们已经有了一个简单的“空”应用，它可以编译并用下面的命令在三个不同的环境下运行：

```bash
# this is node 1
MIX_ENV=gpad_1 iex --name gpad_1@127.0.0.1 -S mix run

# this is node 2
MIX_ENV=gpad_2 iex --name gpad_2@127.0.0.1 -S mix run

# this is node 3
MIX_ENV=gpad_3 iex --name gpad_3@127.0.0.1 -S mix run
```

我们要做的是增加一个简单的可以在控制台中使用的`ping`功能，用来展示集群如何工作。


### 节点，虚拟节点和组织他们的环

riak_core的主要概念之一是与[环](http://docs.basho.com/riak/kv/2.2.0/learn/concepts/clusters/#the-ring)有关的
虚拟节点的概念

![riak-ring](imgs/riak-ring.png)

这个想法很简单，你应该取得你在应用中管理的所有可能的值（这被成为键空间）并进行分区。
例如riak_core使用经典的hash函数从二进制串中取得一个值，代码类似这样：

```bash
iex(1)> key = "gpad"
"gpad"
iex(2)> idx = :erlang.term_to_binary(key) |> :crypto.sha
<<211, 108, 199, 240, 242, 57, 27, 91, 139, 82, 154, 145, 27, 215, 191, 24, 107, 77, 162, 202>>
iex(3)> << v::unsigned-big-integer-size(160) >> = idx
<<211, 108, 199, 240, 242, 57, 27, 91, 139, 82, 154, 145, 27, 215, 191, 24, 107, 77, 162, 202>>
iex(4)> v
1207022950459909619005619617169028319269647917770
iex(5)>
```

如你所见，我们将一个单词作为作为键并转为二进制串，然后使用`:crypt.sha`来hash这个它，然后，我们使用了
[二进制模式匹配](https://hexdocs.pm/elixir/Kernel.SpecialForms.html#%3C%3C%3E%3E/1-binary-bitstring-matching)
显示了这个值的，它的范围在0到2^160之间

riak_core默认把键空间（2^160）分割为64个分区（partition），每个分区负责管理（*认领*）环上的一段（segment）。
这样的单个分区被成为虚拟节点（*vnode*），虚拟节点分布在真实的节点上，这样，当我们添加或删除节点时，可以做到尽可能少的数据传输。
（更多的细节将在下一篇文章中介绍）

我们身处BEAM的世界中，所以很自然就会想到将每个*vnode*映射到一个Erlang的进程。Riak_core非常不错的提供了一个[behaviour](http://erlang.org/doc/design_principles/des_princ.html#id69904)来帮助我们实现vnode进程。在Elixir中使用行为（behaviour）
也非常简单。我建议在`lib/no_slides`中创建一个名为`NoSlides.VNode`的模块，然后像下面这样声明`riak_core_vnode`行为：

```elixir
defmodule NoSlides.VNode do
  @behaviour :riak_core_vnode
end
```

如果这是你尝试编译一下工程，你会得到这样一些告警信息**warning: undefined behaviour function …**。这是因为我们还没有实现这些行为。
如果你熟悉面向对象编程，你可以把行为想像成接口，模块应该实现一系列的函数，并在一个确定的上下文中执行，下面是我们看到的告警：

```bash
warning: undefined behaviour function delete/1
warning: undefined behaviour function encode_handoff_item/2
warning: undefined behaviour function handle_command/3
warning: undefined behaviour function handle_coverage/4
warning: undefined behaviour function handle_exit/3
warning: undefined behaviour function handle_handoff_command/3
warning: undefined behaviour function handle_handoff_data/2
warning: undefined behaviour function handoff_cancelled/1
warning: undefined behaviour function handoff_finished/2
warning: undefined behaviour function handoff_starting/2
warning: undefined behaviour function init/1
warning: undefined behaviour function is_empty/1
warning: undefined behaviour function terminate/2
```

应该在模块中实现下面这些函数，这些函数会在出发特定事件时被调用：

* **init/1** -- 当虚拟节点创建时被调用，它和[GenServer](https://hexdocs.pm/elixir/GenServer.html#c:init/1)
的*init*非常相似。它接收一个数组作为参数，数组里是vnode负责管理的分区的值。该函数返回进程的状态（state）。在其他的回调函数中，
进程状态总是作为最后一个传入的参数。
* **terminate/2** -- 当虚拟节点终止时被调用，它接收reason（原子类型）和state作为参数。
* **handle_exit/3** -- 当vnode[关联](http://erlang.org/doc/reference_manual/processes.html#id87759)的进程死亡时，
该函数被调用。它接收死亡进程的pid，reason和state。你可以返回`{:noreply, state}`来继续（运行程序）。
* **delete/1** -- 当需要删除与vnode关联的数据时调用，和其他函数一样，它接收state作为参数并返回`{:ok, new_state}`。
* **handle_command/3** -- 当vnode执行命令时被调用，这个函数被用来实现我们自己的命令。
* **handle_coverage/4** -- 当我们创建[coverage命令](https://marianoguerra.github.io/little-riak-core-book/listing-keys-from-a-bucket.html)时调用。我们将在下一篇文章中处理这种类型的命令。
* 剩下的函数都和[handoff过程](https://github.com/basho/riak_core/wiki/Handoffs)（handoffs procedure）有关，我们将在下一篇中讨论


当实现最后一个函数`start_vnode/1`后，系统就可以正常工作了，最终，你的模块看起来是这样：

```elixir

defmodule NoSlides.VNode do
  @behaviour :riak_core_vnode

  def start_vnode(partition) do
    :riak_core_vnode_master.get_vnode_pid(partition, __MODULE__)
  end

  def init([partition]) do
    {:ok, partition}
  end

  def handle_command(request, sender, state) do
    # we work here!!!
  end

  def handoff_starting(_dest, state) do
    {true, state}
  end

  def handoff_cancelled(state) do
    {:ok, state}
  end

  def handoff_finished(_dest, state) do
    {:ok, state}
  end

  def handle_handoff_command(_fold_req, _sender, state) do
    {:noreply, state}
  end

  def is_empty(state) do
    {true, state}
  end

  def terminate(_reason, _state) do
    :ok
  end

  def delete(state) do
    {:ok, state}
  end

  def handle_handoff_data(_bin_data, state) do
    {:reply, :ok, state}
  end

  def encode_handoff_item(_k, _v) do
  end

  def handle_coverage(_req, _key_spaces, _sender, state) do
    {:stop, :not_implemented, state}
  end

  def handle_exit(_pid, _reason, state) do
    {:noreply, state}
  end

end
```

函数`handle_command/3`里才包含了真正的工作，使用模式匹配（就像GenServer一样），我们像这样来实现handle_command

```elixir
def handle_command({:ping, v}, _sender, state) do
  {:reply, {:pong, v + 1}, state}
end
```

现在，我们已经实现了ping命令。那么如何从控制台执行？我们需要引入一个我喜欢称之为**服务**的概念。
服务是一个模块，它包装了一些操作，可以与riak_core暴露出来的命令交互。服务应该在riak_core中注册，
这样riak_core才会知道是什么节点暴露了服务。代码如下：

```elixir
defmodule NoSlides.Service do

  def ping(v\\1) do
    idx = :riak_core_util.chash_key({"noslides", "ping#{v}"})
    pref_list = :riak_core_apl.get_primary_apl(idx, 1, NoSlides.Service)

    [{index_node, _type}] = pref_list

    :riak_core_vnode_master.sync_command(index_node, {:ping, v}, NoSlides.VNode_master)
  end

end
```

在第四行，我们计算了要存储的值的id。这个id是*键空间*上的一个值。利用这个**idx**，我们可以用**get_primary_apl**（第5行）向riak_core请求一个**首选项列表**（preference list）。首选项列表是一个集合，它包含了哪些节点处理哪些分区的信息。当调用*get_primary_apl*时，
我们需要列表的长度1（第二个参数）和一个实现了**NoSlides.Service**的节点（第三个参数）。在这个例子中，我们只请求了一个元素，
因为我们只希望在一个节点上执行命令，下一篇文章中，我们会讨论*冗余*（redundancy）。从首选项列表中，我们提取**index_node**，
用于标识执行命令的真实的或是虚拟的节点。这个节点对具有**idx**标识的数据拥有所有权。

在第9行，我们用**index_node**作为参数调用函数`:riak_core_vnode_master.sync_command`。该函数是同步的，
也就是说直到vnode模块完成工作后它才会返回。如果查看`:riak_core_vnode_master`的代码，你会发现`:riak_core_vnode_master.command`，
这个函数采用的则是异步的方式。

你可能还发现了`sync_spawn_command`，它类似于`sync_command`，检查代码的话，你会发现这样的注释

```erlang
%% Send a synchronous spawned command to an individual Index/Node
%% combination.
%% Will not return until the vnode has returned, but the
%%% vnode_master will
%% continue to handle requests.
sync_spawn_command({Index,Node}, Msg, VMaster) ->
```

事实上不是这样的，这些可能是老的注释或者是老的实现，最后要说的是，riak_core约定，
vnode的名称带有`_master`后缀（**NoSlides.VNode_master**）

现在，我们已经实现了**Service**和**VNode**，还需要把所有的这些集中在一起，为此，我们要从头开始...

### 启动riak_core程序
在OTP应用中，我们需要从一个实现了[Application](https://hexdocs.pm/elixir/Application.html)行为的模块开始。
如果你用mix创建了一个空的工程，那么你可能已经有了一个名为`NoSlides`的模块并引入了`use Application`，扔掉它并替换成这样：
```elixir
defmodule NoSlides do
  use Application
  require Logger

  def start(_type, _args) do
    case NoSlides.Supervisor.start_link do
      {:ok, pid} ->
        :ok = :riak_core.register(vnode_module: NoSlides.VNode)
        :ok = :riak_core_node_watcher.service_up(NoSlides.Service, self())
        {:ok, pid}
      {:error, reason} ->
        Logger.error("Unable to start NoSlides supervisor because: #{inspect reason}")
    end
  end

end
```

第6行，我们启动了一个supervisor，稍后再来实现它。如果一切顺利，我们在第8行注册实现了vnode的模块，
在第9行注册实现了服务的模块。

supervisor应该在`lib/no_slides/supervisor.ex`中实现，内容像是这样：

```elixir
defmodule NoSlides.Supervisor do
  use Supervisor

  def start_link do
    # riak_core appends _sup to the application name.
    Supervisor.start_link(__MODULE__, [], [name: :no_slides_sup])
  end

  def init(_args) do
    children = [
      worker(:riak_core_vnode_master, [NoSlides.VNode], id: NoSlides.VNode_master_worker)
    ]
    supervise(children, strategy: :one_for_one, max_restarts: 5, max_seconds: 10)
  end

end
```

这是一个经典的supervisor，但我们要注意一些细节，supervisor的id应该以`_sup`为结束（第6行），
而worker的id应该使用`_master_worker`后缀（第11行）

之后，我们可以使用下面的命令启动节点：

```bash
MIX_ENV=gpad_1 iex --name gpad_1@127.0.0.1 -S mix run
```

在IEx中，可以运行ping服务：
```bash
Interactive Elixir (1.3.4) - press Ctrl+C to exit (type h() ENTER for help)
iex(gpad_1@127.0.0.1)1> NoSlides.Service.ping
{:pong, 2}
iex(gpad_1@127.0.0.1)2> NoSlides.Service.ping(42)
{:pong, 43}
iex(gpad_1@127.0.0.1)3>
```

现在，我们可以加入更多的节点来运行*分布式的ping*，第一步，我们需要在不同的控制台启动更多的节点：
```bash
# this is node 1
MIX_ENV=gpad_1 iex --name gpad_1@127.0.0.1 -S mix run

# this is node 2
MIX_ENV=gpad_2 iex --name gpad_2@127.0.0.1 -S mix run

# this is node 3
MIX_ENV=gpad_3 iex --name gpad_3@127.0.0.1 -S mix run
```

现在可以将所有的节点连接起来了，在第二个节点的控制台键入：
```bash
iex(gpad_2@127.0.0.1)1> :riak_core.join('gpad_1@127.0.0.1')
```

同样的，在节点3执行相同的操作
```bash
iex(gpad_3@127.0.0.1)1> :riak_core.join('gpad_1@127.0.0.1')
```

如果检查节点1的控制台，那么你会看到这样的日志：
```bash
12:21:53.168 [info] 'gpad_2@127.0.0.1' joined cluster with status 'valid'
12:22:39.155 [info] 'gpad_3@127.0.0.1' joined cluster with status 'joining'
12:22:39.191 [info] 'gpad_3@127.0.0.1' changed from 'joining' to 'valid'
```

现在，你可以使用下面的命令来请求riak_core打印**环**的状态：

```bash
{:ok, ring} = :riak_core_ring_manager.get_my_ring
:riak_core_ring.pretty_print(ring, [:legend])
```

你会看到这样的输出：

```bash
============================= Nodes =============================
Node a: 22 ( 34.4%) gpad_1@127.0.0.1
Node b: 21 ( 32.8%) gpad_2@127.0.0.1
Node c: 21 ( 32.8%) gpad_3@127.0.0.1
============================= Ring =============================
abcc|abcc|abcc|abcc|abcc|abcc|abcc|abcc|abcc|abcc|abca|abba|abba|abba|abba|abba|
```

正如你所看到的，环被分成64个分区，节点1拥有22个VNode，剩下两个节点各拥有21个，你同样可以在**observer**中看到：

![observer](imgs/observer.png)

现在我们可以在VNode的实现中加入日志，这样我们就可以看到是哪个节点响应了ping（记得在模块顶部加入`require Logger`）

```elixir
def handle_command({:ping, v}, _sender, state) do
  Logger.debug("Receive ping with value: #{v}")
  {:reply, {:pong, v + 1}, state}
end
```

在控制台，执行命令：
```bash
iex(gpad_1@127.0.0.1)1> NoSlides.Service.ping
{:pong, 2}
```

节点2上，就可以看到如下日志：
```bash
12:43:00.822 [debug] Receive ping with value: 1
```

我们已经有了一个非常简单的分布式ping，如果改变传递给ping的值，能够看到响应ping的不同的节点。
例如，如果使用42，则节点3会作出响应。

现在，我们有了ping，这样我们就可以创建一个基于内存的键值存储。


### 内存中的KV

现在我们已经了解如何连接服务和vnode，我们可以在服务模块上暴露`get`和`put`函数，来轻松创建一个内存键值存储，

```elixir
defmodule NoSlides.Service do

 # ...

 def put(k, v) do
    idx = :riak_core_util.chash_key({"noslides", k})
    pref_list = :riak_core_apl.get_primary_apl(idx, 1, NoSlides.Service)

    [{index_node, _type}] = pref_list

    :riak_core_vnode_master.command(index_node, {:put, {k, v}}, NoSlides.VNode_master)
  end

  def get(k) do
    idx = :riak_core_util.chash_key({"noslides", k})
    pref_list = :riak_core_apl.get_primary_apl(idx, 1, NoSlides.Service)

    [{index_node, _type}] = pref_list

    :riak_core_vnode_master.sync_command(index_node, {:get, k}, NoSlides.VNode_master)
  end

end
```

同样，在VNode中也要添加`get`和`put`的实现：
```elixir
defmodule NoSlides.VNode do

  def init([partition]) do
    {:ok, %{partition: partition, data: %{}}}
  end

  # ...

  def handle_command({:put, {k, v}}, sender, state) do
    Logger.debug("[put]: k: #{inspect k} v: #{inspect v}")
    new_state = Map.update(state, :data, %{}, fn data -> Map.put(data, k, v) end)
    {:noreply, new_state}
  end

  def handle_command({:get, k}, sender, state) do
    Logger.debug("[get]: k: #{inspect k}")
    {:reply, Map.get(state.data, k, nil), state}
  end

end
```

现在，你可以从不同的节点上进行获取和添加的操作了。

重启所有节点，但不需要重新执行加入节点的操作，在节点1的控制台，执行下面的命令：
```bash
iex(gpad_1@127.0.0.1)1> NoSlides.Service.put(:k, 42)
:ok
iex(gpad_1@127.0.0.1)2> NoSlides.Service.get(:k)
42
```

检查节点2上的日志：
```bash
iex(gpad_2@127.0.0.1)1>
19:31:30.634 [debug] [put]: k: :k v: 42
19:31:39.242 [debug] [get]: k: :k
```

同样，在节点3也可以取值
```bash
iex(gpad_3@127.0.0.1)1> NoSlides.Service.get(:k)
42
```

我们拥有了一个内存中的键值存储，你可以添加很多不同类型数据作为值：
```bash
iex(gpad_1@127.0.0.1)9> NoSlides.Service.put("gpad", %{ blogs: ["riak_core I", "riak_core II"] })
:ok
iex(gpad_1@127.0.0.1)10> NoSlides.Service.get("gpad")
%{blogs: ["riak_core I", "riak_core II"]}
```

同样也可以是键：
```bash
iex(gpad_1@127.0.0.1)11> NoSlides.Service.put(%{a: 1, b: 2}, "gpad")
:ok
iex(gpad_1@127.0.0.1)12> NoSlides.Service.get(%{a: 1, b: 2})
"gpad"
```

这是一个简单的内存键值存储的开始，还有一个开放的问题：
* 如果一个节点使用`:riak_core.leave`离开集群，会发生什么？
* 如果某个节点崩溃，会发生什么？
* 我们如何获取所有键的列表？

我会在下一篇中尝试回答这些问题，如果你有任何问题或发现了什么错误，请不要犹豫，在下面留言吧。

*作为译者，我的英文水平也很挫，如果读者发现任何问题，也请留言吧*
