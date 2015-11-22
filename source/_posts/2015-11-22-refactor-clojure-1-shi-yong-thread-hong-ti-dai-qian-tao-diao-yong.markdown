---
layout: post
title: "Refactor Clojure (1) -- 使用 thread 宏替代嵌套调用"
date: 2015-11-22 15:41:04 +0800
comments: true
categories: clojure 重构 refactor
---

我一直想写这么一个系列文章，讲讲 clojure 怎么重构代码。想了很久，没有行动，那么就从今天开始吧。这个系列的目的是介绍一些 clojure 重构的常用手法，有 clojure 特有，也有一些跟《重构：改善既有代码的设计》类似的方法，权且抛砖引玉。我会尽量以真实的项目代码为例子来介绍，Let's begin。

## 问题 

Clojure 是函数式编程，而函数式编程一个特点就是我一直喜欢强调的数据流抽象，在实际编程中我们经常做的一个事情是将一个数据使用 map,filter,reduce 等高阶函数做转换，增加一点信息，减少一点信息，过滤一些信息，累积一些信息等等，来得到我们最终想要的数据。例如我最近有这么一个任务，从一个 map 里收集所有的 key，包括内部的嵌套 map，例如这么一个 map:

```clj
(def x {:a 1 :b { :b1 "hello" :b2 2} :c { :c1 {:c21 "world" :c22 5}}})
```


我要写一个函数 all-keys，想得到的结果是 

```clj
user=> (all-keys  x)
[:c :b :a :c1 :c22 :c21 :b2 :b1]
```

顺序我不关心，但是要求能找出所有的 key，包括嵌套。

## 初步解决

解决这个问题不难，我们本质上是要遍历一个树形结构，找出所有 map，然后使用 `keys` 函数获取他们的关键字列表，然后加入一个结果集合。使用 [clojure cheatsheet](http://clojure.org/cheatsheet) 我们根据关键字 tree 找到函数 `tree-seq`:

```clj
user=> (doc tree-seq)
-------------------------
clojure.core/tree-seq
([branch? children root])
  Returns a lazy sequence of the nodes in a tree, via a depth-first walk.
   branch? must be a fn of one arg that returns true if passed a node
   that can have children (but may not).  children must be a fn of one
   arg that returns a sequence of the children. Will only be called on
   nodes for which branch? returns true. Root is the root node of the
  tree.
nil
```

他会按照深度优先遍历的顺序访问树的子节点，你需要提供 `branch?` 谓词函数来判断节点是不是分支，如果是，tree-seq 会调用 `children` 函数来访问子节点， root 就是开始的根节点了，我们这里就是 `x`。

试试，我们的分支都是 map，谓词判断是 `map?`，子节点是 map 的所有值可以用 `vals` 函数得到：

```
user=> (tree-seq map? vals x)
({:c {:c1 {:c22 5, :c21 "world"}}, :b {:b2 2, :b1 "hello"}, :a 1} 
 {:c1 {:c22 5, :c21 "world"}} {:c22 5, :c21 "world"} 5 "world" 
 {:b2 2, :b1 "hello"} 2 "hello" 1)
```

很棒，他访问了所有子节点，返回了节点链表。下一步，我们要找出所有节点是 map 类型的：

```clj
user=> (filter map? (tree-seq map? vals x))
({:c {:c1 {:c22 5, :c21 "world"}}, :b {:b2 2, :b1 "hello"}, :a 1}
 {:c1 {:c22 5, :c21 "world"}} 
 {:c22 5, :c21 "world"} 
 {:b2 2, :b1 "hello"})
```

使用 `(filter map? coll)` 过滤出了所有 map 类型，接下来就是遍历这个链表，取出每个 map 的关键字：

```clj
user=> (map keys (filter map? (tree-seq map? vals x)))
((:c :b :a) (:c1) (:c22 :c21) (:b2 :b1))
```

很赞，使用 `(map keys	 coll)` 我们获取了所有 map 的关键字列表，但是结果是一个链表组成的链表，我们希望能『扁平化』这个链表，该是 `flatten` 出场了：

```clj
user=> (flatten (map keys (filter map? (tree-seq map? vals x))))
(:c :b :a :c1 :c22 :c21 :b2 :b1)
```

Great! 貌似我们已经到达目的地了。我们整理下代码，写一个 all-keys 函数来封装这段逻辑：

```clj
(defn all-keys [coll] 
  (flatten 
    (map keys 
      (filter map? 
        (tree-seq map? vals coll)))))
```

## 重构

很好，我们完成了需求，不过这段代码调用了 4 个高阶函数，层层嵌套。阅读代码的人需要从最里层的 tree-seq 开始，一层一层往外看才能理解他是干什么，有没有办法更好？ 当然可以，我们有 clojure 提供的 thread 宏： `->` 和 `->>`，它就是用来处理这种多层嵌套 form 的场景，简单例子

```clj
(-> 1 (+ 2) (* 4)) # => 12
```

本质上展开为:

```clj
user=> (macroexpand-1  '(-> 1 (+ 2) (* 4)))
(* (+ 1 2) 4)
```

他会将第一个参数插入第二个 form 的第二个位置，然后将这个结果再插入第三个 form 的第二个位置，以此类推形成一个嵌套的 form 结构。而 `->>` 则总是将参数插入 form 的最后一个位置。观察下我们的 all-keys，会发现数据的变换都发生在函数调用的最后一个位置，很明显，我们应该用 `->>`。

因此，我们可以改下 all-keys:

```clj
(defn all-keys [coll] 
  (->> coll
    (tree-seq map? vals)
    (filter map?)
    (map keys)
    (flatten)))
```

使用了 `->>` 之后，整个数据的流转变得很清晰，从上往下一层一层变换。

更进一步， map 和 flatten 其实可以用 mapcat 替换：

```clj
(defn all-keys [coll] 
  (->> coll
    (tree-seq map? vals)
    (filter map?)
    (mapcat keys)))
```

如果我们想对结果排序，最后加一个 `sort`:

```clj
(defn all-keys [coll] 
  (->> coll
    (tree-seq map? vals)
    (filter map?)
    (mapcat keys)
    (sort)))
```

## 讨论

我们这里使用到的都是标准库的高阶函数，他们的参数顺序都是精心组织的，集合放到最后，函数放在中间位置，这样就可以使用 thread 宏，这也提醒我们自己编写函数的时候，也应该尽量遵循这样的原则：

* 将集合参数放到最后。
* 将要变换的数据参数（返回的是这个数据的『变换』）尽量放到第二个位置或者最后的位置。
* 函数应该尽量做到数据的输入和输出，而非单纯产生副作用。

但是你使用的外部程序可能不满足这些原则，这种情况下，你需要引入一个中间函数来包装，例如假设 all-keys 最后我们还想调用一个缓存函数来缓存（我知道有 `memoize`，这里只是举例）这个结果 `(cache keys)`，但是很可惜 cache 函数返回 nil，如果直接写成：

```clj
(defn all-keys [coll] 
  (->> coll
    (tree-seq map? vals)
    (filter map?)
    (mapcat keys)
    (sort)
    (cache)))
```

那么将永远返回 nil，这不是我们想要的，遇到这种情况，我们可以加一层包装：

```clj
defn wrap-cache [x] 
  (cache x) 
  x)
  
(defn all-keys [coll]
  (->> coll
    (tree-seq map? vals)
    (filter map?)
    (mapcat keys)
    (sort)
    (wrap-cache)))  
```

这样，我们保证了结果的正确性，并且保留了 thread 宏的使用。

在 Java 里我们解决这个问题的通常手段应该是调用链的方式，不过 Java8 已经支持 lambda 表达式，想必也有类似的简化代码嵌套的方案。


