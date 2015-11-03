---
layout: post
title: "Erlang 和 Elixir shell 历史记录设置"
date: 2015-11-03 07:36:40 +0800
comments: true
categories: erlang elixir
---

Erlang 的 erl 和 Elixir 的 iex 都只有当前 session 的历史记录，可以通过 ctrl + r 或者上下方向键来返回历史记录，并执行。但是当 session 一旦退出，重新启动一个 shell session，前一个历史记录就没有了，这个就非常麻烦。

题外：在 clojure 里， lein repl 帮你处理了这个事情，它将历史命令保存在 `~/.lein_history` 文件，在不同 session 之间可以随时调取历史记录。如果使用内置的 clojure REPL，也可以使用 [rlwrap](http://www.blogjava.net/killme2008/archive/2012/02/14/369976.html) 来包装，提供历史记录功能。

### erlang-history

不过庆幸的是有一个开源项目帮你解决了这个问题—— [erlang-history](https://github.com/ferd/erlang-history)，它的解决方式比较重量级，通过给 Kernel 打补丁的方式（线上环境肯定不推荐），保存历史记录到 erlang dets。安装非常简单：

```sh
git clone git@github.com:ferd/erlang-history.git
cd erlang-history
make install
```

可能会提示你需要 sudo 权限，因为它要替换 Erlang 默认的 kernel.beam。

安装后，默认的 erl 和 iex 命令就拥有历史记录功能了。不过可能你想修改下一些默认配置。erlang-history 提供的选项包括：

```
hist - true | false :是否启用，默认 true
hist_file - string(): 历史记录文件的 dets 文件名，字符串，默认是 ~/.erlang-history.$NODENAME
hist_size - 1..n    : 历史记录大小，数字，默认 500
hist_drop - ["some", "string", ...]: 不想保存的命令列表。
```

### 配置 erl

虽然可以通过命令行传入配置参数，不过更推荐的方式还是写一个配置文件，例如放到 `~/erl_hist.config`:

```erlang
[{kernel,[
  {hist_size, 1000},
  {hist_drop, ["q().", ":init.stop()", "init:stop()."]}
]}].
```

我就配置了两个，最大历史记录 1000 条，忽略 3 个命令，两个是 erlang 的退出命令，一个是 Elixir 的 `:init.stop()` 。虽然一般都不会调用命令来退出，而是连续两个 ctrl + c。

接下来在 `~/.bash_profile` 里为 erl 修改个别名：

```sh
alias erl="erl -config $HOME/erl_hist.config"
```

让 erl 默认读取配置文件，erlang-history 的配置就生效了。

### 配置 iex

配置 iex 只是为 iex 加个别名而已，通过 `--erl` 传入参数：

```sh
alias iex="iex --erl '+P 4000000 +K true -config $HOME/erl_hist.config'"
```

我的配置夹藏了一点私货 `+P 4000000 +K true`，增大最大进程数限制和启用 kernel-poll，方便平常测试学习。




