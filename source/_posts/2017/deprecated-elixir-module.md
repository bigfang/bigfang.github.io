---
title: Elixir中的废弃模块
date: 2017-03-08 20:29:07
tags:
  - elixir
categories:
  - 折腾
---

最近一段时间为了学习Elixir，不停的在翻阅中英文的书籍、文章和博客。确实学到不少，不过也有些东西确实让我迷惑了一阵子，比如《Elixir程序设计》一书的第8章，着实吓了我一跳，「字典：散列表、散列字典、关键字列表、集合与结构体」，这么多的数据结构，看起来都差不多，一讲到关键部分，书里说的就不甚明了，隔靴搔痒。更诡异的是，还有一些教程，压根就没提到散列字典，还有的文章，甚至出现了`MapSet`这种东西。

怎么回事呢。

作为一门年轻的语言，Elixir在不停的进化，随着版本升级，语言特性也在不断的更新。所以，包括《Elixir程序设计》在内的很多译作与文章,已经落后于语言的发展了。这再一次的说明了，关注[`CHANGELOG`](https://github.com/elixir-lang/elixir/releases)是多么的重要...

目前Elixir的版本是1.4.2，整理一下目前我关注到的一些被废弃的模块

* `Behaviour`：无疑是一个重量级的废弃，使用更优雅的模块属性`@callback`和`@macrocallback`来代替之前的写法，这个新特性自`1.4.0`开始。

* `Dict`：在被废弃之前，`Dict`似乎是实现了某种`Behaviour`，既可以操作`keyword list`，也可以操作`Map`，这个模块确实让人摸不着头脑，所以它在`1.3.0`开始被抛弃了。

* `HashDict`：文档上说用`Map`来代替它，为啥会这样我也不是很懂，不过，[这个](https://gist.github.com/BinaryMuse/bb9f2cbf692e6cfa4841)性能测试也许能说明点什么

* `Set`与`HashSet`：从`1.2.0`开始，就被“软性废弃”（使用它不会报warnning）了，`Set`模块在`1.4.0`时被正式放弃。文档里建议使用`MapSet`来代替它们。

总结一下，目前Elixir中的几种常用数据结构和操作它们的模块如下：

| 数据结构      | 操作模块                                          | 被废弃的模块       |
|---------------|---------------------------------------------------|--------------------|
| Tuple         | [Tuple](https://hexdocs.pm/elixir/Tuple.html)     |                    |
| Keyword list  | [Keyword](https://hexdocs.pm/elixir/Keyword.html) | `Dict`             |
| (Linked) List | [List](https://hexdocs.pm/elixir/List.html)       |                    |
| Map           | [Map](https://hexdocs.pm/elixir/Map.html)         | `Dict`, `HashDict` |
| Set           | [MapSet](https://hexdocs.pm/elixir/MapSet.html)   | `Set`, `HashSet`   |


除了废弃模块，Elixir还加入了一些新的特性，我目前知道的用起来比较爽的有：

* `with...else`语句，这种写法可以大大减少代码量，下面是官方文档中在`with`语句中使用`Guard`的例子，
  ```elixir
  iex> users = %{"melany" => "guest", "bob" => :admin}
  iex> with {:ok, role} when not is_binary(role) <- Map.fetch(users, "bob"),
  ...>      do: {:ok, to_string(role)}
  {:ok, "admin"}
  ```

* `ExUnit.Case`里增加了`describe/2`宏，可以理解成把test case分组，使单元测试的代码可以更便于阅读。下面同样是官方文档中的例子：
  ```elixir
  defmodule UserManagementTest do
    use ExUnit.Case, async: true

    describe "when user is logged in and is an admin" do
      setup [:log_user_in, :set_type_to_admin]

      test ...
    end

    describe "when user is logged in and is a manager" do
      setup [:log_user_in, :set_type_to_manager]

      test ...
    end

    defp log_user_in(context) do
      # ...
    end
  end
  ```
  被`describe`包裹的代码块可以单独运行测试，像这样：
  ```bash
  mix test --only describe:"when user is logged in and is an admin"
  ```
