---
layout: post
title: "Clojure里使用curator做Leader节点选举"
date: 2014-05-22 23:40:09 +0800
comments: true
categories: clojure,zookeeper
---

[Curator](http://curator.apache.org/) 框架刚出来的时候，我就用它帮 [Storm](http://storm.incubator.apache.org/) 重构了 zookeeper 模块。使用 [zookeeper](http://zookeeper.apache.org/)，如果用 java 语言，curator 框架是最佳选择。

最近在做一个节点选举的功能，在几个节点之间选举一个 leader 来跑一个独占服务。原来的方案是直接利用 hostname 匹配，跟配置的 hostname 一致的固定某台机器来执行。Failover 靠人肉和自动化脚本。为了做让 failover 自动化，自动选举节点是更好的方案。理所当然，我尝试在 clojure 里使用 curator 框架。
curator 提供了 <a href="http://curator.apache.org/curator-recipes/leader-election.html">Leader Election</a>功能，我要做的只是封装这个Java API，在clojure里更好地使用。

首先，肯定是继承[LeaderSelectorListenerAdapter](http://curator.apache.org/apidocs/org/apache/curator/framework/recipes/leader/LeaderSelectorListenerAdapter.html) 来实现 [LeaderSelectorListener](https://curator.apache.org/apidocs/org/apache/curator/framework/recipes/leader/LeaderSelectorListener.html) ，监听本节点是否成功获取 leadership，当本节点成功被选举的时候，<code>LeaderSelectorListener</code> 的<code>takeLeadership</code>方法将调用，你应该阻塞这个方法，直到：

<ul>
<li>你想释放自己的 leadership，比如节点重启了，或者独占服务异常了，想重新发起选举。</li>
<li>Zookeeper session 状态异常，比如 <code>SUSPENDED</code> 或者 <code>LOST</code> ，需要解除阻塞，重新发起选举。</li>
</ul>

继承<code>LeaderSelectorListenerAdapter</code>我们用 [proxy](http://clojuredocs.org/clojure_core/1.2.0/clojure.core/proxy) 函数，阻塞呢？Clojure提供了[promise](http://clojuredocs.org/clojure_core/clojure.core/promise)，当 promise 没有值的时候， deref 调用会阻塞， promise 本质上是一个[CountDownLatch](http://wiki.fnil.net/index.php?title=Clojure%E5%B9%B6%E5%8F%91#future.E3.80.81promise.E5.92.8C.E7.BA.BF.E7.A8.8B)。我们就利用它来阻塞 <code>takeLeadership</code> 方法，封装下这个过程：

``` clojure
;;保存curator框架客户端的atom
(defonce ^:private curator-framework (atom nil))
;;保存LeaderSelector列表的atom
(defonce ^:private leader-selectors (atom []))

(defn elect-leader
   "参与leader选举，如果被选举为leader，调用aquire函数，释放leadership的时候调用release函数。
   path表示参与选举节点共同使用的zookeeper上的路径。"
  [path aquire release]
  (if (nil? @curator-framework)
    (throw (IllegalStateException. "Please call start at first."))
    (let [p (promise)
          listener (proxy [LeaderSelectorListenerAdapter] []
                                             (takeLeadership [fk]
                                               (aquire fk)
                                               (try
                                                 @p
                                                 (catch InterruptedException _
                                                   (release fk)))))
          selector (LeaderSelector. @curator-framework path listener)]
      (swap! leader-selectors conj [p selector])
      (doto selector
        ;;u/hostname是一个工具方法，用于获取本机hostname，方便调试和分析
        (.setId (u/hostname))
        (.autoRequeue)
        (.start)))))
```


关键性的代码就是`takeLeadership`方法的实现，传入的`aquire`方法最好是非阻塞或者可中断（释放 leadership 的时候，curator 会尝试中断 `takeLeadership` 方法，解除阻塞）。为了保持独占，我们对`p`这个 promise 做阻塞性的`deref`调用，如果阻塞被中断 catch 到`InterruptedException`，我们就调用传入的`release`方法，你可以在`release`里做一些释放 leadership 后的资源清理的工作。

最后，奉上一些创建和销毁 curator client 的辅助代码：

```clojure
(defn ^CuratorFramework mk-client
  "Create curator framework client."
  []
  (let [builder (CuratorFrameworkFactory/builder)]
    (doto builder
      (.connectString (env :zk-connect "localhost:2181"))
      (.connectionTimeoutMs (config/int-var :zk-connect-timeout 10000))
      (.sessionTimeoutMs (config/int-var :zk-session-timeout 10000))
      (.retryPolicy (ExponentialBackoffRetry. (config/int-var :zk-retry-interval 1000) (config/int-var :zk-retry-times 5))))
    (let [fk (.build builder)]
      (.start fk)
      fk)))

(defn start
  "Initialize curator framework client."
  []
  (reset! curator-framework (mk-client)))


(defn stop
  "Stop all elections,release leadership if they have.
  Then shutdown the curator framework client."
  []
  (when-let [promise-selectors (seq @leader-selectors)]
    (doseq [[p s] promise-selectors]
      (deliver p true)
      (.close s)))
  (reset! leader-selectors [])
  (when @curator-framework
    (.close @curator-framework))
  (reset! curator-framework nil))

```

关键性的`stop`方法，使用`deliver`给 promise 传递一个`true`值来解除阻塞，从而释放了`leadership`，让其他选举发起新一轮选举。

