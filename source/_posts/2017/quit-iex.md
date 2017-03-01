---
title: IEx的一些Tips
date: 2017-03-01 17:26:56
tags:
  - elixir
  - erlang
  - tips
categories:
  - 折腾
---

### IEx里的帮助
敲个`h`就出来啦，还是彩色的，其实启动iex的时候稍微注意下文字就能发现...
```bash
(master)⚡ % iex
Erlang/OTP 19 [erts-8.2.2] [source] [64-bit] [smp:4:4] [async-threads:10] [hipe] [kernel-poll:false] [dtrace]

Interactive Elixir (1.4.2) - press Ctrl+C to exit (type h() ENTER for help)
iex(1)>
```

### 退出IEx的快捷键
因为`iex`继承自erlang的`erl`，所以退出的方法也颇为诡异，`Ctrl-D`都无法直接退出的

* 连续按两次 `Ctrl-C`。最暴力，但估计也是用的最多的方法了
* `Ctrl-C`之后根据提示，选择`a`。看起来比上面的方式文明不少
* `Ctrl-G`之后，然后输入`q`。`erl`中，`Ctrl-G`之后将进入`作业控制模式（JCL）`，
进入这种模式后，可以键入`h`或者`?`来显示帮助。
* `Ctrl-\`。这种方式等同于执行`:erlang:halt`，看起来最接近`Ctrl-D`，按键次数也是最少，
还是[erlang](http://erlang.org/faq/getting_started.html#idp31948112)“官方推荐”快捷键


### 使用IEx.pry进行调试
其实函数式语言，一般都不需要太复杂的调试（吧），因为验证了输入输出就完事了（呀）。
不过作为一个长得像`ruby`的语言，Elixir的`IEx`提供了一个简化版的[pry](http://pryrepl.org/)。

例如下面的代码，注意第1行，第6行：
```elixir
require IEx

defmodule Adder do
  def add(a, b) do
    c = a + b
    IEx.pry
  end
end
```

当在`iex`里调用`Adder.add/2`时，会出现下面这样的提示
```elixir
iex(1)> Adder.add(1, 2)
Request to pry #PID<0.85.0> at ooxx.exs:6

      def add(a, b) do
        c = a + b
        IEx.pry
      end
    end

Allow? [Yn]
```

输入`y`，`Adder.add/2`所在的进程便会被“撬开”，iex的shell将会被重置，
断点之前的变量和词法作用域都可以被访问到，可以在iex里进行一些计算，就像下面这样
```elixir
pry(1)> Enum.map([a, b, c], &IO.inspect(&1))
1
2
3
[1, 2, 3]
```

`IEx.pry/1`在调用者进程中运行的，在它执行期间会阻塞住调用者，可以使用`respawn`释放调用者，
这个操作会启动一个新的`IEx`进程：
```elixir
pry(2)> respawn()

Interactive Elixir (1.4.2) - press Ctrl+C to exit (type h() ENTER for help)
:ok
iex(1)> self()
#PID<0.89.0>
```

可以看到，之前的pid是`#PID<0.85.0>`，现在pid变成了`#PID<0.89.0>`。