---
title: 【译】Erlang/Elixir语法速成
date: 2017-01-01 14:26:43
tags:
  - 翻译
  - erlang
  - elixir
categories:
  - 翻译
---


### 原文：[Erlang/Elixir Syntax: A Crash Course](http://elixir-lang.org/crash-course.html)


# Erlang/Elixir语法速成
本文是针对Erlang开发人员的Elixir语法简介，同时也适用于试图了解Erlang的Elixir开发者。
本文简要介绍了Elixir/Erlang语法，互操作能力，在线文档和示例代码等最基础的知识。

1. [运行代码](#1-运行代码)
    1. [Erlang](#11-erlang)
    2. [Elixir](#12-elixir)
2. [显著差异](#2-显著差异)
    1. [操作符名称](#21-操作符名称)
    2. [分隔符](#22-分隔符)
    3. [变量名](#23-变量名)
    4. [函数调用](#24-函数调用)
3. [数据类型](#3-数据类型)
    1. [原子](#31-原子)
    2. [元组](#32-元组)
    3. [列表与二进制串](#33-列表与二进制串)
    4. [键值列表](#-34键值列表)
    5. [映射(Map)](#35-映射(Map))
    6. [正则表达式](#36-正则表达式)
4. [模块](#4-模块)
5. [函数语法](#5-函数语法)
    1. [模式匹配](#51-模式匹配)
    2. [函数识别](#52-函数识别)
    3. [默认值](#53-默认值)
    4. [匿名函数](#54-匿名函数)
    5. [作为一等公民的函数](#55-作为一等公民的函数)
    6. [Elixir中的局部应用与函数捕捉](#56-Elixir中的局部应用与函数捕捉)
6. [流程控制](#6-流程控制)
    1. [Case](#61-case)
    2. [If](#62-if)
    3. [发送和接收消息](#63-发送和接收消息)
7. [将Elixir添加到已有的Erlang程序中](#7-将Elixir添加到已有的Erlang程序中)
    1. [使用Rebar集成](#71-使用Rebar集成)
    2. [手动集成](#72-手动集成)
8. [扩展阅读](#8-扩展阅读)


## 1 运行代码
### 1.1 Erlang
最快速运行代码的方法是启动Erlang shell - `erl`。本文中大部分代码可以直接粘贴到shell中，
Erlang中的命名函数必须包含在模块中，而且必须要在模块编译后才能使用。下面是一个模块的例子：

```erlang
% module_name.erl
-module(module_name).  % you may use some other name
-compile(export_all).

hello() ->
  io:format("~s~n", ["Hello world!"]).
```

编辑文件并保存，在同一目录下运行`erl`，并执行`编译`命令

```erlang
Eshell V5.9  (abort with ^G)
1> c(module_name).
ok
1> module_name:hello().
Hello world!
ok
```

shell运行的同时也可以编辑文件。 但不要忘记执行`c(module_name)`来加载最新的更改。
要注意，文件名必须与在`-module()`中声明的文件名保持一致，扩展名为`.erl`。

### 1.2 Elixir
与Erlang类似，Elixir有一个名为`iex`的交互式shell。编译Elixir代码可以使用`elixirc`（类似于Erlang的`erlc`）。
Elixir还提供一个名为`elixir`的可执行文件来运行Elixir代码。上面的模块用Elixir来写就是这样：

```elixir
# module_name.ex
defmodule ModuleName do
  def hello do
    IO.puts "Hello World"
  end
end
```

然后，在`iex`中编译：

```elixir
Interactive Elixir
iex> c("module_name.ex")
[ModuleName]
iex> ModuleName.hello
Hello world!
:ok
```

要注意的是，在Elixir中，不要求模块必须保存在文件中，Elixir的模块可以直接在shell中定义：

```elixir
defmodule MyModule do
  def hello do
    IO.puts "Another Hello"
  end
end
```

## 2 显著差异
这一节讨论了两种语言之间的一些语法差异。

### 2.1 操作符名称
部分操作符采用了不同的书写方式

| ERLANG  | ELIXIR | 含义             |
|---------|--------|------------------|
| and     | 不可用 | 逻辑‘与’，         |
| andalso | and    | 逻辑‘与’的简短写法 |
| or      | 不可用 | 逻辑‘或’，         |
| orelse  | or     | 逻辑‘与’的简短写法 |
| =:=     | ===    | 全等             |
| =/=     | !==    | 不全等            |
| /=      | !=     | 不等于            |
| =<      | <=     | 小于等于          |

### 2.2 分隔符
Erlang表达式使用点号`.`作为结束，逗号`,`用来分割同一上下文中的多个表达式（例如在函数定义中）。
在Elixir中，表达式由换行符或分号`;`分隔。

**Erlang**
```erlang
X = 2, Y = 3.
X + Y.
```

**Elixir**
```elixir
x = 2; y = 3
x + y
```

### 2.3 变量名
Erlang中的变量只能被绑定一次，Erlang shell提供一个特殊的命令`f`，用于删除某个变量或所有变量的绑定。
Elixir允许变量被多次赋值，如果希望匹配变量之前的值，应当使用`^`。

**Erlang**
```erlang
Eshell V5.9  (abort with ^G)
1> X = 10.
10
2> X = X + 1.
** exception error: no match of right hand side value 11
3> X1 = X + 1.
11
4> f(X).
ok
5> X = X1 * X1.
121
6> f().
ok
7> X.
* 1: variable 'X' is unbound
8> X1.
* 1: variable 'X1' is unbound
```

**Elixir**
```elixir
iex> a = 1
1
iex> a = 2
2
iex> ^a = 3
** (MatchError) no match of right hand side value: 3
```

### 2.4 函数调用
Elixir允许在函数调用中省略括号，而Erlang不允许。

| ERLANG           | ELIXIR        |
|------------------|---------------|
| some_function(). | some_function |
| sum(A, B)        | sum a, b      |

调用模块中的函数的语法不同，Erlang中，下面的代码

```erlang
 lists : last ([ 1 , 2 ]).
```

表示从`list`模块中调用函数`last`，而在Elixir中，使用点号`.`代替冒号`:`

```elixir
List.last([1, 2])
```

**注意** 在Elixir中，由于Erlang的模块用原子表示，因此用如下方法调用Erlang的函数：

```elixir
:lists.sort [3, 2, 1]
```

所有Erlang的内置函数都包含在`:erlang`模块中

## 3 数据类型
Erlang和Elixir的数据类型大部分都相同，但依然有一些差异。

### 3.1 原子
Erlang中，`原子`是以小写字母开头的任意标志符，例如`ok`、`tuple`、`donut`.
以大写字母开头的标识符则会被视为变量名。而在Elixir中，前者被用于变量名，而后者则被视为原子的别名。
Elixir中的原子始终以冒号`:`作为首字符。

**Erlang**
```erlang
im_an_atom.
me_too.

Im_a_var.
X = 10.
```

**Elixir**
```
:im_an_atom
:me_too

im_a_var
x = 10

Module  # 称为原子别名; 展开后是 :'Elixir.Module'
```

非小写字母开头的标识符也可以作为原子。 不过两种语言的语法有所不同：

**Erlang**
```erlang
is_atom(ok).                %=> true
is_atom('0_ok').            %=> true
is_atom('Multiple words').  %=> true
is_atom('').                %=> true
```

**Elixir**
```elixir
is_atom :ok                 #=> true
is_atom :'ok'               #=> true
is_atom Ok                  #=> true
is_atom :"Multiple words"   #=> true
is_atom :""                 #=> true
```

### 3.2 元组
两种语言元组的语法是相同的，不过API有所不同，Elixir尝试使用下面的方法规范化Erlang库：

1. 函数的`subject`总是作为第一个参数。
2. 所有对数据结构的操作均基于零进行。

也就是说，Elixir不会导入默认的`element`和`setelement`函数，而是提供`elem`和`put_elem`作为替代：

**Erlang**
```erlang
element(1, {a, b, c}).       %=> a
setelement(1, {a, b, c}, d). %=> {d, b, c}
```

**Elixir**
```elixir
elem({:a, :b, :c}, 0)         #=> :a
put_elem({:a, :b, :c}, 0, :d) #=> {:d, :b, :c}
```

### 3.3 列表与二进制串
Elixir具有访问二进制串的快捷语法：

**Erlang**
```erlang
is_list('Hello').        %=> false
is_list("Hello").        %=> true
is_binary(<<"Hello">>).  %=> true
```

**Elixir**
```elixir
is_list 'Hello'          #=> true
is_binary "Hello"        #=> true
is_binary <<"Hello">>    #=> true
<<"Hello">> === "Hello"  #=> true
```

Elixir中，**字符串**意味着一个UTF-8编码的二进制串，`String`模块可用于处理字符串。
同时Elixir也希望程序源码采用UTF-8编码。而在Erlang中，**字符串**表示字符的列表，
`:string`模块用于处理它们，但并没有采用UTF-8编码。

Elixir还支持多行字符串（也被称为*heredocs*）：
```elixir
is_binary """
This is a binary
spanning several
lines.
"""
#=> true
```

### 3.4 键值列表（Keyword list）
Elixir中，如果列表是由具有两个元素的元组组成，并且每个元组中的第一个元素是原子，则称这样的列表为键值列表：

**Erlang**
```erlang
Proplist = [{another_key, 20}, {key, 10}].
proplists:get_value(another_key, Proplist).
%=> 20
```

**Elixir**
```elixir
kw = [another_key: 20, key: 10]
kw[:another_key]
#=> 20
```

### 3.5 映射（Map）
Erlang R17中引入了映射，一种无序的键-值数据结构。键和值可以是任意的数据类型，
映射的创建、更新和模式匹配如下所示：

**Erlang**
```erlang
Map = #{key => 0}.
Updated = Map#{key := 1}.
#{key := Value} = Updated.
Value =:= 1.
%=> true
```

**Elixir**
```elixir
map = %{:key => 0}
map = %{map | :key => 1}
%{:key => value} = map
value === 1
#=> true
```

当键为原子时，Elixir可以使用`key: 0`来定义映射，使用`.key`来访问值：

```elixir
map = %{key: 0}
map = %{map | key: 1}
map.key === 1
```

### 3.6 正则表达式
Elixir支持正则表达式语法，允许在编译（elixir源码）时编译正则表达式，
而不是等到运行时再进行编译。而且对于特殊的字符，也无需进行多次转义：

**Erlang**
```erlang
{ ok, Pattern } = re:compile("abc\\s").
re:run("abc ", Pattern).
%=> { match, ["abc "] }
```

**Elixir**
```elixir
Regex.run ~r/abc\s/, "abc "
#=> ["abc "]
```

支持在*heredocs*中书写正则，这样便于理解复杂正则

**Elixir**
```elixir
Regex.regex? ~r"""
This is a regex
spanning several
lines.
"""
#=> true
```

## 4 模块
每个Erlang模块都保存在与其同名的文件中，具有以下结构：

```erlang
-module(hello_module).
-export([some_fun/0, some_fun/1]).

% A "Hello world" function
some_fun() ->
  io:format('~s~n', ['Hello world!']).

% This one works only with lists
some_fun(List) when is_list(List) ->
  io:format('~s~n', List).

% Non-exported functions are private
priv() ->
  secret_info.
```

在这里，我们创建了一个名为`hello_module`的模块。模块中定义了三个函数，
顶部的`export`指令导出了前两个函数，让它们能够被其他模块调用。`export`指令里包含了需要导出函数的列表，
其中每个函数都写作`<函数名>/<元数>`的形式。在这里，元数是函数参数的个数。

和上述Erlang代码作用相同的Elixir代码：

```elixir
defmodule HelloModule do
  # A "Hello world" function
  def some_fun do
    IO.puts "Hello world!"
  end

  # This one works only with lists
  def some_fun(list) when is_list(list) do
    IO.inspect list
  end

  # A private function
  defp priv do
    :secret_info
  end
end
```

在Elixir中，一个文件中可以包含多个模块，并且还允许嵌套定义模块：

```elixir
defmodule HelloModule do
  defmodule Utils do
    def util do
      IO.puts "Utilize"
    end

    defp priv do
      :cant_touch_this
    end
  end

  def dummy do
    :ok
  end
end

defmodule ByeModule do
end

HelloModule.dummy
#=> :ok

HelloModule.Utils.util
#=> "Utilize"

HelloModule.Utils.priv
#=> ** (UndefinedFunctionError) undefined function: HelloModule.Utils.priv/0
```

## 5 函数语法
「Learn You Some Erlang」书中的[这一章][syntax-func]详细讲解了Erlang的模式匹配和函数语法。
而本文只简要介绍主要内容并展示部分示例代码。

[syntax-func]: http://learnyousomeerlang.com/syntax-in-functions

### 5.1 模式匹配
Elixir中的模式匹配基于于Erlang实现，两者通常非常类似：

**Erlang**
```erlang
loop_through([H | T]) ->
  io:format('~p~n', [H]),
  loop_through(T);

loop_through([]) ->
  ok.
```

**Elixir**
```elixir
def loop_through([h | t]) do
  IO.inspect h
  loop_through t
end

def loop_through([]) do
  :ok
end
```

当多次定义名称相同的函数时，每个这样的定义称为**子句** 。
在Erlang中，子句总是按顺序写在一起并使用分号`;`分隔 。 最后一个子句用点号`.`结束。

Elixir不需要通过符号来分隔子句，不过要求子句必须按顺序写在一起。

### 5.2 函数识别
在Erlang和Elixir中，仅凭函数名是无法区分一个函数的。必须通过函数名和*arity*（元/参数）。
下面两个例子中，我们定义了四个不同的函数（所有名字都为`sum`，但它们具有不同的元）：

**Erlang**
```erlang
sum() -> 0.
sum(A) -> A.
sum(A, B) -> A + B.
sum(A, B, C) -> A + B + C.
```

**Elixir**
```elixir
def sum, do: 0
def sum(a), do: a
def sum(a, b), do: a + b
def sum(a, b, c), do: a + b + c
```

Guard表达式提供了一种简明的方法来定义在不同条件下接受有限个数参数的函数。

**Erlang**
```erlang
sum(A, B) when is_integer(A), is_integer(B) ->
  A + B;

sum(A, B) when is_list(A), is_list(B) ->
  A ++ B;

sum(A, B) when is_binary(A), is_binary(B) ->
  <<A/binary,  B/binary>>.

sum(1, 2).
%=> 3

sum([1], [2]).
%=> [1, 2]

sum("a", "b").
%=> "ab"
```

**Elixir**
```elixir
def sum(a, b) when is_integer(a) and is_integer(b) do
  a + b
end

def sum(a, b) when is_list(a) and is_list(b) do
  a ++ b
end

def sum(a, b) when is_binary(a) and is_binary(b) do
  a <> b
end

sum 1, 2
#=> 3

sum [1], [2]
#=> [1, 2]

sum "a", "b"
#=> "ab"
```

### 5.3 默认值
Elixir允许参数具有默认值，而Erlang不允许。
```elixir
def mul_by(x, n \\ 2) do
  x * n
end

mul_by 4, 3 #=> 12
mul_by 4    #=> 8
```

### 5.4 匿名函数
定义匿名函数：

```erlang
Sum = fun(A, B) -> A + B end.
Sum(4, 3).
%=> 7

Square = fun(X) -> X * X end.
lists:map(Square, [1, 2, 3, 4]).
%=> [1, 4, 9, 16]
```

```elixir
sum = fn(a, b) -> a + b end
sum.(4, 3)
#=> 7

square = fn(x) -> x * x end
Enum.map [1, 2, 3, 4], square
#=> [1, 4, 9, 16]
```

定义匿名函数时也可以使用模式匹配。

```erlang
F = fun(Tuple = {a, b}) ->
        io:format("All your ~p are belong to us~n", [Tuple]);
        ([]) ->
        "Empty"
    end.

F([]).
%=> "Empty"

F({a, b}).
%=> "All your {a, b} are belong to us"
```

```elixir
f = fn
      {:a, :b} = tuple ->
        IO.puts "All your #{inspect tuple} are belong to us"
      [] ->
        "Empty"
    end

f.([])
#=> "Empty"

f.({:a, :b})
#=> "All your {:a, :b} are belong to us"
```

### 5.5 作为一等公民（first-class）的函数
匿名函数是*first-class values*，因此它们可以当作参数传递给其他函数，也可以被当作返回值。
对于命名函数，可以使用如下语法实现上述功能。

```erlang
-module(math).
-export([square/1]).

square(X) -> X * X.

lists:map(fun math:square/1, [1, 2, 3]).
%=> [1, 4, 9]
```

```elixir
defmodule Math do
  def square(x) do
    x * x
  end
end

Enum.map [1, 2, 3], &Math.square/1
#=> [1, 4, 9]
```

### 5.6 Elixir中的局部应用与函数捕捉
Elixir可以利用函数的局部应用（partial application），以简洁的方式定义匿名函数：
```elixir
Enum.map [1, 2, 3, 4], &(&1 * 2)
#=> [2, 4, 6, 8]

List.foldl [1, 2, 3, 4], 0, &(&1 + &2)
#=> 10
```

函数捕捉同样使用`&`操作符，它使得命名函数可以作为参数传递。

```elixir
defmodule Math do
  def square(x) do
    x * x
  end
end

Enum.map [1, 2, 3], &Math.square/1
#=> [1, 4, 9]
```
上面的代码相当于Erlang的`fun math:square/1` 。

## 6. 流程控制
`if`和`case`结构在Erlang和Elixir中实际上是表达式，不过依然可以像命令式语言的语句那样，用于流程控制

### 6.1 Case
`case`结构是完全基于模式匹配的流程控制。

**Erlang**
```erlang
case {X, Y} of
  {a, b} -> ok;
  {b, c} -> good;
  Else -> Else
end
```

**Elixir**
```elixir
case {x, y} do
  {:a, :b} -> :ok
  {:b, :c} -> :good
  other -> other
end
```

### 6.2 If

**Erlang**
```erlang
Test_fun = fun (X) ->
  if X > 10 ->
       greater_than_ten;
     X < 10, X > 0 ->
       less_than_ten_positive;
     X < 0; X =:= 0 ->
       zero_or_negative;
     true ->
       exactly_ten
  end
end.

Test_fun(11).
%=> greater_than_ten

Test_fun(-2).
%=> zero_or_negative

Test_fun(10).
%=> exactly_ten
```

**Elixir**
```elixir
test_fun = fn(x) ->
  cond do
    x > 10 ->
      :greater_than_ten
    x < 10 and x > 0 ->
      :less_than_ten_positive
    x < 0 or x === 0 ->
      :zero_or_negative
    true ->
      :exactly_ten
  end
end

test_fun.(44)
#=> :greater_than_ten

test_fun.(0)
#=> :zero_or_negative

test_fun.(10)
#=> :exactly_ten
```

Elixir的`cond`和Erlang的`if`有两个重要的区别：

  * `cond`允许左侧为任意表达式，而Erlang只允许Guard子句；
  * `cond`使用Elixir中的真值概念（除了`nil`和`false`皆为真值），而Erlang的`if`则严格的期望一个布尔值；

Elixir同样提供了一个类似于命令式语言的`if`函数，用于检查一个子句是true还是false：

```elixir
if x > 10 do
  :greater_than_ten
else
  :not_greater_than_ten
end
```

### 6.3 发送和接收消息
发送和接收消息的语法仅略有不同：

```erlang
Pid = self().

Pid ! {hello}.

receive
  {hello} -> ok;
  Other -> Other
after
  10 -> timeout
end.
```

```elixir
pid = Kernel.self

send pid, {:hello}

receive do
  {:hello} -> :ok
  other -> other
after
  10 -> :timeout
end
```

## 7 将Elixir添加到已有的Erlang程序中
Elixir会被编译成BEAM字节码（通过Erlang抽象格式）。这意味着Elixir和Erlang的代码可以互相调用而不需要添加其他任何绑定。
Erlang代码中使用Elixir模块须要以`Elixir.`作为前缀，然后将Elixir的调用附在其后。
例如，这里演示了在Erlang中如何使用Elixir的`String`模块：

```erlang
-module(bstring).
-export([downcase/1]).

downcase(Bin) ->
  'Elixir.String':downcase(Bin).
```

### 7.1 使用Rebar集成
如果使用rebar，应当把Elixir的git仓库引入并作为依赖添加：
```
https://github.com/elixir-lang/elixir.git
```

Elixir的结构与Erlang的OTP类似，被分为不同的应用放在`lib`目录下，可以在[Elixir源码仓库][repo]中看到这种结构。
由于rebar无法识别这种结构，因此需要在`rebar.config`中明确的配置所需要的Elixir应用，例如：
```elixir
{lib_dirs, [
  "deps/elixir/lib"
]}.
```

这样就能直接从Erlang调用Elixir代码了，如果需要编写Elixir代码，还应安装[自动编译Elixir的rebar插件][plugin]。

[repo]: https://github.com/elixir-lang/elixir
[plugin]: https://github.com/yrashk/rebar_elixir_plugin

### 7.2 手动集成
如果不使用rebar，在已有Erlang软件中使用Elixir的最简单的方式是按照[入门指南][guide]中的方法安装Elixir，然后将`lib`添目录加到`ERL_LIBS`中。

## 8 扩展阅读
Erlang的官方文档网站有不错的编程[示例集][examples]，把它们重新用Elixir实现一遍是不错的练习方法。
[Erlang cookbook][cookbook]也提供了更多有用的代码示例。

还可以进一步阅读Elixir的[入门指南][guide]和[在线文档][doc]。


[examples]: http://www.erlang.org/doc/programming_examples/users_guide.html
[cookbook]: http://schemecookbook.org/Erlang/TOC
[guide]: http://elixir-lang.org/getting-started/introduction.html
[doc]: http://elixir-lang.org/docs.html
