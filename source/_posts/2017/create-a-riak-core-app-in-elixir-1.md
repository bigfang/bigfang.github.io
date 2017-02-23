---
title: 【译】在Elixir中创建riak_core应用（1）
date: 2017-02-21 23:02:34
tags:
  - elixir
  - erlang
  - riak_core
  - 分布式
categories:
  - 翻译
---

最近不知道咋了，有些高产啊，目前网上似乎没人翻译这个，争取周末前把三篇翻完


### 原文：[Create a riak_core application in Elixir (Part 1)](https://medium.com/@GPad/create-a-riak-core-application-in-elixir-part-1-41354c1f26c3)


## 在Elixir中创建riak_core应用（I）

12月3号（2016年），我在[NoSlidesConf][conf]做了一次关于riak_core的演讲,
为了用Elixir展示[riak_core][riak_core]一些特性，我写了[这个][example]简单的应用。

[conf]: http://www.noslidesconf.net/#schedule
[example]: https://github.com/gpad/no_slides

在那以后，我决定写一篇关于在Elixir中使用riak_core的教程，在Elixir中调用Erlang的库确实非常简单，
比如这个app就是个很好的例子，网上有很多用Erlang（使用riak_core）的例子，
但遗憾的是，依然有很多“炼金术士”们并不熟悉Erlang的语法:-(

本文中，我将描述如何创建一个空的Elixir应用，它只能作为多个节点在同一台机器上启动而没有其他功能，
要等到下一篇文章，您才能看到实用的功能:-).


### 如何开始
[Riak_core][riak_core]是由[basho][basho]开发，从[riakKV/riakTS][riak]中剥离出来的一个独立的库。
basho的这个[Repo][riak_core]对于Elixir应用来说并不容易使用，但是你可以检出代码作为“main source”。
当然作为一名“炼金师”，你肯定非常熟悉[https://hex.pm/](https://hex.pm/)（如果不是，那你该努力了:-)）
在hex.pm里搜索riak_core，你会发现[riak_core_ng][riak_core_ng]。
这是从原库中[fork](https://github.com/project-fifo/riak_core)出来的，
由[Heinz N. Gies][HNG]维护的版本。

开始之前，先介绍一下我的Erlang和Elixir的版本。本系列教程中，我会一直使用Elixir 1.3.4和Erlang 18.3。
在测试时，我没能用Erlang 19成功运行riak_core。如果你查看basho的[Repo][riak_core]和
riak_core_ng的[Repo][riak_core_ng]，你会发现其中包含了移植到Erlang 19的工作，不过至今还未完成。
我建议使用[kerl][kerl]或者[asdf][asdf]来管理Erlang的版本。如果你想用[observer][observer]（你肯定想的:-)）。
我建议在编译Erlang OTP的时候开启wx。如果你使用OSX，请看[这里](http://featurebranch.com/howto-getting-wx-to-work-with-erlang-r16b02-on-os-x/)，
Linux用户请看[这里](http://stackoverflow.com/questions/32934641/how-to-get-erlang-to-show-ui-components-debugger-and-observer-on-linux)）。
其他系统用户请尝试google。

那么我们开始吧，附上`--sup`参数，创建一个新的mix应用

```bashe
$ mix new no_slides --sup
```

用这种方法，我们创建了一个空的app，首先要做的是把riak_core加入依赖项中。在`mix。exs`文件中加入如下内容：
```elixir
def application do
  [applications: [:riak_core, :logger],
   mod: {NoSlides, []}]
end
# ...
defp deps do
  [
    {:riak_core, "~> 2.2.8", hex: :riak_core_ng}
  ]
end
```

记得要把`:riak_core`放在放在第一位，否则可能会发生一些奇怪的事情:-D.

然后，就像平时一样，获取依赖

```bash
$ mix deps.get
```

现在可以用下面的命令运行你的第一个riak_core app了：

```bash
$ iex -S mix run
```

你看到了好多的error，首先你应该用参数`--name <node_name>`运行node，我的是这样：

```bash
$ iex --name gpad@127.0.0.1 -S mix run
```

这是使用[全名](http://erlang.org/doc/reference_manual/distributed.html)启动程序的方式，
之后，你会得到另一个错误，需要为你的代码配置schema和一些其他的设置。schema存在在`deps/riak_core/priv`目录中，
使用下面的命令，将其拷贝到你的`priv`目录下

```bash
$ cp -r deps/riak_core/priv .
```

最后，要告诉riak_core如何管理节点的必需的信息，我建议在`config/config.exs`文件中加入如下内容：

```elixir
config :riak_core,
  ring_state_dir: 'ring_data_dir',
  handoff_port: 8099,
  handoff_ip: '127.0.0.1',
  schema_dirs: ['priv']
config :sasl,
  errlog_type: :error
```

配置项`riak_core`是riak_core作为单节点启动的必要配置（多节点时我们会做一些小修改），
我们告诉riak_core在哪里保存[环](https://github.com/basho/riak_core/wiki#ring)上的数据（集群配置）。
管理[handoff](https://github.com/basho/riak_core/wiki/Handoffs)的IP/端口，以及在哪里找到schema。
最后的配置也同样重要，我们告诉[sasl](http://erlang.org/doc/man/sasl_app.html)应用只打印错误日志。

至此，你终于可以启动一个单节点的riak_core应用了：

```bash
$ iex --name gpad@127.0.0.1 -S mix run
```

**万岁！！！**


### 多节点环境

我们当然希望用riak_core创建一个分布式的应用，所以我们的应用应当运行在不同的节点上。在开发时，在一台机器上面运行多个节点会更加简单易行。
为了做到这一点，有必要让每个节点用不同的配置来运行，我们可以用mix的[environments](http://elixir-lang.org/getting-started/mix-otp/introduction-to-mix.html#environments)达到这一目的。
我们为每个环境（environment）添加不同的配置文件，通过这些不同的环境来区分各个节点。
在本例中，我把环境命名为`gpad_1`、`gpad_2`和`gpad_3`。我建议在`config`目录中添加三个文件`gpad_1.exs`、`gpad_2.exs`和`gpad_3.exs`。
修改`config.exs`，在文件末尾添加以下内容

```elixir
import_config “#{Mix.env}.exs”
```

在每个新文件中，你需要设置riak_core的配置，使其不会与在同一台机器上运行的其他节点冲突，因此我们有：

gpad_1.exs
```elixir
use Mix.Config
config :riak_core,
  node: 'gpad_1@127.0.0.1',
  web_port: 8198,
  handoff_port: 8199,
  ring_state_dir: 'ring_data_dir_1',
  platform_data_dir: 'data_1'
```

gpad_2.exs
```elixir
use Mix.Config
config :riak_core,
  node: 'gpad_2@127.0.0.1',
  web_port: 8298,
  handoff_port: 8299,
  ring_state_dir: 'ring_data_dir_2',
  platform_data_dir: 'data_2'
```

gpad_3.exs
```elixir
use Mix.Config
config :riak_core,
  node: 'gpad_3@127.0.0.1',
  web_port: 8398,
  handoff_port: 8399,
  ring_state_dir: 'ring_data_dir_3',
  platform_data_dir: 'data_3'
```

这样，我们就有了三个不同的环境，你可以在三个不同的终端下启动它们，在开始之前，请记得要使用Erlang 18，
因为我们需要在不同的环境下编译这个应用，因此要在不同的终端下运行下面的每行命令：
```bash
# this is node 1
MIX_ENV=gpad_1 iex --name gpad_1@127.0.0.1 -S mix run

# this is node 2
MIX_ENV=gpad_2 iex --name gpad_2@127.0.0.1 -S mix run

# this is node 3
MIX_ENV=gpad_3 iex --name gpad_3@127.0.0.1 -S mix run
```

如果你希望像之前一样，仍然使用命令`iex --name gpad@127.0.0.1 -S mix run`来运行一个单节点，
你需要添加一个名为`dev。exs`的文件，它只有一行：
```elixir
use Mix.Config
```

现在，你应该能够在同一台机器上运行三个不同的节点了。

**做的不错！！！**


下一篇中，我们会尝试添加一个简单的ping功能并将三个节点连接在一起。

在结束前，我需要感谢：
* [basho][basho]，他们创造了这个[库][riak_core]和[文档](http://basho.com/search/?q=riak_core)。
* [Heinz N. Gies][HNG]，他创造了可以在Elixir中使用的[riak_core_ng][riak_core_ng]。
* [Ben Tyler](https://github.com/kanatohodets)，他在[ElixirConf.EU](http://www.elixirconf.eu/elixirconf2016/ben-tyler)上发表了精彩的演讲。
* [Mariano Guerra](https://twitter.com/warianoguerra)，他写了一本介绍riak_core的精彩的[书](https://marianoguerra.github.io/little-riak-core-book/)。
* [Ryan Zezeski](https://twitter.com/rzezeski)，他的关于riak_core博客。
* 正在阅读这篇文章并尝试riak_core的你，谢谢。

敬请期待，下次再见！！！



[HNG]: https://twitter.com/heinz_gies
[riak_core]: https://github.com/basho/riak_core
[riak_core_ng]: https://hex.pm/packages/riak_core_ng
[riak]: https://github.com/basho/riak
[basho]: http://basho.com
[observer]: http://erlang.org/doc/apps/observer/observer_ug.html

