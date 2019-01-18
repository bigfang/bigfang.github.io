---
title: 后台运行PhoenixFramework应用
date: 2017-02-17 21:43:40
tags:
  - elixir
  - phoenix
categories:
  - 折腾
---

如果你不打算用`Exrm`来部署应用的话，就用下面这条命令吧

```shell
MIX_ENV=prod PORT=4001 elixir --detached -S mix phoenix.server
```

附上目前版本的`elixir --help`

```shell
% elixir --help
Usage: elixir [options] [.exs file] [data]

  -v                          Prints version and exits
  -e COMMAND                  Evaluates the given command (*)
  -r FILE                     Requires the given files/patterns (*)
  -S SCRIPT                   Finds and executes the given script in PATH
  -pr FILE                    Requires the given files/patterns in parallel (*)
  -pa PATH                    Prepends the given path to Erlang code path (*)
  -pz PATH                    Appends the given path to Erlang code path (*)

  --app APP                   Starts the given app and its dependencies (*)
  --cookie COOKIE             Sets a cookie for this distributed node
  --detached                  Starts the Erlang VM detached from console
  --erl SWITCHES              Switches to be passed down to Erlang (*)
  --hidden                    Makes a hidden node
  --logger-otp-reports BOOL   Enables or disables OTP reporting
  --logger-sasl-reports BOOL  Enables or disables SASL reporting
  --name NAME                 Makes and assigns a name to the distributed node
  --no-halt                   Does not halt the Erlang VM after execution
  --sname NAME                Makes and assigns a short name to the distributed node
  --werl                      Uses Erlang's Windows shell GUI (Windows only)

** Options marked with (*) can be given more than once
** Options given after the .exs file or -- are passed down to the executed code
** Options can be passed to the Erlang runtime using ELIXIR_ERL_OPTIONS or --erl
```
