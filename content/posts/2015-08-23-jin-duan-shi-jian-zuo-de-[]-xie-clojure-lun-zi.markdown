---
layout: post
title: "近段时间做的一些 clojure 轮子"
date: 2015-08-23 19:35:07 +0800
comments: true
categories: 开源 clojure
---

LeanCloud 可能（应该是）国内最大规模的 clojure 应用，无论是存储、推送还是聊天都是构建在 clojure 之上。单纯 API 服务，每天的规模都是亿次规模的动态调用请求。使用一门小众语言的后果是，你需要造很多别的语言里已经有的轮子。好在 clojure 可以直接用 java 类库，很多轮子你只是包装下 java 类库即可。

我们已经造了很多的 clojure 轮子。下面说说我最近造的一些 clojure 轮子。

## Hystrix 相关

首先是跟 [Netflix/Hystrix](https://github.com/Netflix/Hystrix) 相关的。Hystrix 的设计理念真是相见恨晚，早在淘宝的时候就听过大名，真正开始使用和了解才从今年开始。从他的设计文档来看，我过去很多的土法轮子人家都总结成 Pattern，并且设计了美妙的 API，例如 [Request Collapsing](https://github.com/Netflix/Hystrix/wiki/How-it-Works#RequestCollapsing) 和 [Request Caching](https://github.com/Netflix/Hystrix/wiki/How-it-Works#RequestCaching) 都是很朴素的想法，我在 xmemcached 实现里就做了 get 请求和 set 请求合并等技巧来提升性能；对外部调用利用线程池和信号量做隔离，原来在 Notify 的实现上也充分使用了这些技术。但是没有它总结的这么好，并且提供了丰富的配置项。一个侧面反映了我的抽象能力上的欠缺，或者说思考的还不够深入。

回到正题，我开始在我们的 API 服务里使用 hystrix 隔离和控制各种外部调用，使用了 [hystrix-clj](https://github.com/Netflix/Hystrix/tree/master/hystrix-contrib/hystrix-clj)，这个轮子是官方提供的， API 封装的非常漂亮，你只需要将 `defn` 替换成 `defcommand` 就可以将一个普通的 clojure 函数用 hystrix 封装起来，并且利用 metadata 来配置 hystrix，充分体现了 clojure 的能力。不过这个库原来在处理参数重载的函数的时候有 Bug，我提了个 [PR](https://github.com/Netflix/Hystrix/commit/bebb4cb48bd2f045aa119fee8a1ec5d9988b1334) 解决了下，已经大量应用在我们的服务上了。

Hystrix 提供了一个 dashboard 用来实时展现各种服务的 QPS（单机和集群）、平均耗时、错误统计等，官方推荐用 [hystrix-event-stream-clj](https://github.com/josephwilk/hystrix-event-stream-clj) ，在你的 service 里提供一个 `/hystrix.stream` 给 dashbaord 或者 [turbine](https://github.com/Netflix/Turbine) 收集数据并展现。不过这个库对于 jetty 的支持不好，request 对象按照他的方式集成会引入不必要的 java 对象，无法正确地被序列化和反序列化。因此，我提供了另一种方式—— [ring-jetty-hystrix-adapter](https://github.com/killme2008/ring-jetty-hystrix-adapter)，基本跟 ring-jetty-adpater 的使用方式一样：

```clojure
(require '[ring-jetty-hystrix-adapter.core :as jetty])

(jetty/run-jetty-with-hystrix {:port 3000
                               :max-threads 10
                               :hystrix-servlet-path "/hystrix.stream"
                               :join? false})
```

只是多了个 `hystrix-servlet-path` 参数，指定提供的 event stream 的请求路径是什么，默认是 `"/hystrix.stream"`。

接下来是配置，Netflix 提供的轮子都是成套的，比如配置它就有 [archaius](https://github.com/Netflix/archaius)，这又是一个类似过去在淘宝做过的 diamond 的东西，不过他不提供服务端，专心做好客户端的事情。我现在就拿 taobao diamond server + netflix archaius 当做我们的分布式配置方案。 Diamond server 的设计是非常朴素的，也非常可靠，利用域名+多机静态化配置文件的方式，将风险降到最低。

在 clojure 里使用 archaius，当然可以用他的 java 客户端，不过我们过去都在用 [environ](https://github.com/weavejester/environ) 做配置，为了将迁移成本降到最低，很直接的想法就是按照 environ 的方式来封装 archaius，这就有了 [clj-archaius](https://github.com/leancloud/clj-archaius)，使用方式跟 environ 没有什么区别，同时提供了动态注册配置监听器的方法：

```clojure
(require '[clj-archaius.core :refer :all])
(int-env :a)
(int-env :not-exists 100)
(on-int-env :a (fn [] (println "The new :a is " (int-env :a))))
```

## 流控

原来我们 API 的流控算法简单的基于 memcached 计数器，总所周知，这样的思路无法很好地应对瞬时高峰等情况，也无法做到更精确的控制。

常见的流控算法是 [Token Bucket](https://en.wikipedia.org/wiki/Token_bucket)，考察了几种实现后，决定按照这篇[博客](https://engineering.classdojo.com/blog/2015/02/06/rolling-rate-limiter/)提供的思路来实现，它主要使用 redis 的 zset 和 multi 操作来实现 token bucket 算法，解决了其他基于 redis 算法实现可能存在的不精确和性能问题。不过他的实现使用了 zrange 命令，这在大并发下会耗费很大的流量在跟 redis 交互上，我根据它的思路做了改造，其实只要获取最小时间戳、最大时间戳以及当前请求数就可以了，大大减少了网络流量，从测试来看，比之原来的实现 QPS 翻了一倍。最终的产物就是 [clj-rate-limiter](https://github.com/killme2008/clj-rate-limiter)，专供 clojure 的流控类库，有内存和 redis 存储两个版本。具体使用请参考文档，恕不重复了。

## Lighthouse

[lighthouse](https://github.com/killme2008/lighthouse) 是用来做 zookeeper 一些常见操作的类库，例如节点选举、服务发现和负载均衡等，封装了 curator 类库，只是更方便 clojure 使用而已。

比如选举：

```clojure
(require '[lighthouse.leader :refer :all])

(start-election cli "/leader_election"
  (fn [cli path id]
    (println id "got leadership."))
  (fn [cli path id]
    (println id "released leadership."))
  :id "node-1")
```

`start-election` 接收两个函数，分别在被选举为主节点和释放的时候回调，返回的是一个 clojure promise，你可以 `deliver` true 或者 false 来释放 leadership：

```clojure
(def p (start-election ......))

;;释放 leadership,但是仍然参与选举
(deliver p false)
;;释放 leadership，并不再参与选举
(deliver p true)
```

节点的负载均衡(例如 RPC 请求)也非常简单，定义一个 balancer ，直接调用即可，具体参见文档。

## defun

要说我去年做的最好玩的轮子应该是这个类库 [defun](https://github.com/killme2008/defun)，一个赋予 defn 宏以模式匹配威力的小类库，他结合了 defn 和 [core.match](https://github.com/clojure/core.match)，现在你可以在 clojure 里定义类似 Erlang 或者 Elixir 的函数，基于参数的模式匹配：

```clojure
 (use '[defun :only [defun]])

 (defun accum
      ([0 ret] ret)
      ([n ret] (recur (dec n) (+ n ret)))
      ([n] (recur n 0)))

     (accum 100)
     ;;5050
```

更多精彩例子请参见它的 readme 吧。我主要用它和 [Instaparse](https://github.com/Engelberg/instaparse) 结合做了 CQL 语法解释器，用来将 SQL 翻译成 mongodb 查询。

