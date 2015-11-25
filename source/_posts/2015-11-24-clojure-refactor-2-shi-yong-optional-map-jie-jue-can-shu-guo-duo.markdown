---
layout: post
title: "Refactor Clojure (2) -- 使用 optional map 解决参数过多问题"
date: 2015-11-24 09:30:06 +0800
comments: true
categories: clojure refactor 重构
---

这个系列准备提到的方式可能对大多数 clojure 老手来说都不新鲜，我的目的主要是自己归纳总结，并且同时重构自己的代码，其次是针对解决方式，引出一些讨论。因此每篇博客都分成三部分：

* 问题，说明这次要解决什么问题。
* 解决，提出初步的解决方案和最终本文要讨论的解决方案。
* 讨论，针对这个解决方案做进一步的扩展讨论。

权且抛砖引玉，欢迎有兴趣的朋友参与讨论。

## 问题

通常，我们开始写一个函数来完成某个功能，可能一开始只需要一两个参数就够了，接下来用户提出更多的定制化需求，不可避免，你可能需要传递更多参数进来。当一个方法的参数膨胀到 5 个以上，哪怕 clojure 有内置的 metadata 文档系统，这个函数也不可避免的难用起来。如果这是一个内部方法，也许还能承受，但是如果开放给用户，它的易用性就很成问题。

除了参数过多之外，很可能大多数参数用户是不需要的，内部应该提供一些默认值给这些参数，而不是每次都让用户传递。

以一个问题为例子，我们有这么一个查询接口，一开始他是这样的，你需要传递查询的表名和 where 条件：

```clj
(defn query-objects [table where])
```

接下来，发现用户需要分页功能，我们增加了 skip 和 limit:

```clj
(defn query-objects [table where skip limit])
```

过了没几分钟，用户提出希望能指定返回的对象的字段列表，ok，我们继续增加一个 keys 参数：

```clj
(defn query-objects [table where query-keys skip limit])
```

又过了几天，用户提出需要能排序和关联查询，好吧，我们需要加入排序字段和 include 字段：

```clj
(defn query-objects [table where query-keys skip limit order include])
```

这个过程如果再持续下去，这个方法的参数将无可避免地膨胀，它的问题包括：

* 用户需要知道参数名，也需要知道参数的顺序。
* 另外一个就是刚才提到的默认参数，可能大多数用户只是想查询第一个页，每页 100 条，那么 skip 和 limit 默认就是可以是 0 和 100等。这样的用户目前也不得不传递很多参数进来。
* 万一哪天你想修改这两个默认值，你需要去修改所有调用这个方法的地方，这完全违背了开闭原则（是的，哪怕是 FP，我们也希望遵循开闭原则。）

## 解决 

一个简单的解决办法可能是使用重载参数，请注意** clojure 仅支持参数个数的重载，不支持参数类型重载**：

```clj
(defn query-objects 
 ([table where query-keys]
    (query-objects table where query-keys 0 100))
  ([table where query-keys skip limit]
    (query-objects table where query-keys skip limit))
  ([table where query-keys skip limit order include]
  ......))
```

这样，老用户仍然可以使用 `(query-objects table where)` 来简单查询，其他用户也可以选择对应的参数版本来调用，skip 和 limit 我们也给了一个默认值来调用，一切看起来挺 ok。

不过，如果一个用户希望是查询表的时候排序某个字段，然后不想设定 skip 和 limit，他仍然不得不调用最多参数的版本：

```
(query-objects "TestTable" {:a 1} 0 100 "createdAt" nil)
```

问题仍然没有解决，用户仍然很痛苦。

在《重构》这本书里，遇到这种情况给出的解决方案就是将参数包装成对象—— parameter object，如果参数很多， parameter object 还可以使用 builder 模式来构造。

在 clojure 里，我们会用 map 来包装这些参数，利用强大的 destructring，我们还能为参数提供默认值：

```clj
(defn query-objects [table where & {:keys [query-keys skip limit order include]}]
  ......)
```

这里的  `&` 表示后面的参数是可选的参数，`{:keys [query-keys skip limit order include]}` 表示我们预计可选参数是偶数个，可以组合成一个 map，并且 key 都是 keyword 类型（Keyword 就是类似 Ruby 的 symbol，符号），我们要取出其中的 `query-keys skip limit order include` 这些参数。

这样一来，用户可以选择自己传入什么参数：

```clj
(query-objects "TestTable" {:a 1})
(query-objects "TestTable" {:a 1} :skip 0 :limit 10)
(query-objects "TestTable" {:a 1} :skip 0 :limit 10 :query-keys ["key1" "key2"])
```

不过这里我们没有为 skip 和 limit 设置默认值，这可以通过 `:or` 来提供：

```clj
(defn query-objects [table where & {:keys [query-keys skip limit order include]
                                    :or {skip 0 limit 100}}]
  ......)
```

通过 `:or {skip 0 limit 10}` 我们为这两个参数提供了默认值。

destructring 是在编译器做的展开，同时支持在 `let` 和 `loop` 上，因为 defn、 fn 等展开后内部用到了 let，因此可以直接对参数使用 destructring。关于 map destructring 更多请参考 [Map binding destructuring](http://clojure.org/special_forms#Special%20Forms--Binding%20Forms%20(Destructuring)-Map%20binding%20destructuring)。

如果我们还想对整个参数 map 做一个引用 var，可以使用 `:as`:

```clj
(defn query-objects [table where & {:keys [query-keys skip limit order include]
                                    :or {skip 0 limit 100}
                                    :as options}]
  ......)
```

在函数体内， options 将指代整个参数 map 对象。但是，请注意 **options 将不包括默认值，如果参数没有提供，它在 options 里也不会出现，这可能是一个陷阱。**

```clj
(defn query-objects [table where & {:keys [query-keys skip limit order include]
                                    :or {skip 0 limit 100} 
                                    :as options}] 
  [skip limit options])

(query-objects  "TestTable" {:a 1}) 
#=> [0 10 nil] 
```

## 讨论

可以看到 optional map 比较好的解决了参数过多的问题，通过为参数提供更有意义的命名(keyword)的方式，也避免用户需要明确记忆参数顺序的问题。

但是这种方式也存在缺陷，用户很容易写错参数名称，比如用户不小心这样调用：

```clj
(query-objects "TestTable" {:a 1} :sklp 100 :limit 10)
```

不小心将 `skip` 写成了 `sklp`，导致实际 `skip` 限定根本没有生效。这种问题，我们可以利用 clojure 内置的契约编程范式来解决，为函数加入 `:pre` 约束：

```clj
(defn query-objects [table where & {:keys [query-keys skip limit order include]]
  {:pre [(every? #{:query-keys :skip :limit :order :include} (keys options))]}
  ......)
```

我们使用 `:pre` 元信息加入强制约束谓词 `(every? #{:query-keys :skip :limit :order :include} (keys options))`，要求 options 出现的参数名称只能在 `:query-keys :skip :limit :order :include` 范围内，如果传入其他参数，将抛出断言异常：

```clj
(query-objects  "TestTable" {:a 1} :sklp 3)
#AssertionError Assert failed: (every? #{:query-keys :limit :include :order :skip} (keys options))  user/query-objects
```
不过这里参数名称重复了两次，一次在参数列表，一次在断言，理论上我们可以通过某种方式来消除，这是另一篇博客要讲的了。



