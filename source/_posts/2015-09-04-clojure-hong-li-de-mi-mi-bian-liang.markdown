---
layout: post
title: "Clojure 宏里的秘密变量"
date: 2015-09-04 23:04:47 +0800
comments: true
categories: clojure
---

原来在读 clojure.core 源码的时候，就发现宏有用到两个神奇的变量 `&form` 和  `&env`，比如 `defn` 宏：

```clojure
(def 
 ……
 defn (fn defn [&form &env name & fdecl]
        ;; Note: Cannot delegate this check to def because of the call to (with-meta name ..)
        (if (instance? clojure.lang.Symbol name)
          nil
          (throw (IllegalArgumentException. "Fi
……    
(. (var defn) (setMacro))      
```         

这里有很关键的一行代码： `(. (var defn) (setMacro)) ` 我们后面会谈到。

`defn` 之所以需要明确声明 `&form` 和  `&env`（顺序还必须 `&from` 在前）两个函数参数，是因为他没有使用我们通常用到的 `defmacro` 的方式，当然 `defmacro` 本质上也是一个宏。`defmacro` 会隐式地加入这两个参数，不信我们看下：

```clojure
user=> (macroexpand `(defmacro nothing [a] `~a))
(do 
   (clojure.core/defn user/nothing 
       ([&form &env user/a] user/a)) 
       (. (var user/nothing) (setMacro)) (var user/nothing))
```

看到了吧，本质上 `defmacro` 做的事情就是使用`defn` 定义一个函数，并且比普通函数增加了两个“隐藏”参数，然后将这个函数的 var 设置为宏，通过 `setMacro` 方法。所以，普通函数和宏的区别就这两点：

* 宏多了开头的两个隐藏参数：`&form` 和  `&env`
* 宏对应的 var 调用了 `setMacro`。

当编译器遇到 list 里的第一个参数的 var 是一个宏的时候，他就会去展开表达式，替换 list 。本质上你就是通过 `setMacro` 告诉编译器，我这个 var 是一个宏，你要先做 macroexpand，然后再继续求值。

因此，其实，我们也可以这样定义宏，比如最常见的 `when` 宏：

```clojure
(defn my-when [&form &env test & body]
  `(if ~test (do ~@body)))
```

如果没有 `setMacro`，那么求值的顺序将不同，先求值参数，再执行函数体，并且 my-when 至少要接收三个参数（`&form` 和  `&env` 被当成普通参数了）:

```clojure
user=> (my-when false (println 2))
2
ArityException Wrong number of args (2) passed to: user/my-when  clojure.lang.AFn.throwArity (AFn.java:429)
```

加上 `setMacro`:

```clojure
(.setMacro (var my-when))

user=> (my-when false (println 2))
nil
user=> (my-when true (println 2) 4 5)
2
5
```

回到题目， `&from` 和 `&env` 代表了什么？

`&from` 是用来记录这个宏在**被调用时候**的 from ，而 `&env` 记录这个宏在**被调用时候**的的 local binding（或者说“局部变量”，更精确的是局部绑定）。

看下《Mastering clojure macros》这本书给的例子：

```clojure
(defmacro info-about-caller [arg]
         (pprint {:form &form :env &env})
         `(println "called macro, arg is" ~arg))
```

简单地打印两个隐藏参数和宏调用参数：

```sh
user=> (info-about-caller 1)
{:form (info-about-caller 1), :env nil}
called macro, arg is 1
nil
user=> (info-about-caller (+ 2 3))
{:form (info-about-caller (+ 2 3)), :env nil}
called macro, arg is 5
nil
```

正确地打印了宏被调用时候的 form 是什么样，但是 env 都是 nil，加上 let 看看：

```sh
user=> (let [foo "bar" baz "quux"] (info-about-caller 1))
{:form (info-about-caller 1),
 :env
 {baz #<LocalBinding clojure.lang.Compiler$LocalBinding@3d2f7354>,
  foo #<LocalBinding clojure.lang.Compiler$LocalBinding@4745aa90>}}
called macro, arg is 1
nil
```

可以看到，let 形成的局部绑定被打印出来了。

两个隐藏参数都是 [clojure 编译器帮你收集并传入的](https://github.com/clojure/clojure/blob/master/src/jvm/clojure/lang/Compiler.java#L6778-L6783)，通常你不会去操作这两个参数。如果你读过 clojure.core 的代码，也会看到官方库其实也几乎没有用到这两个隐藏参数，唯一几个地方用到是获取 `&form` 的元信息，传递原始 form 的信息给展开后的新 form，元信息里最重要的就是代码的行列，当宏调用出错的时候，方便调试，一个例子：

```clojure
 (defmacro inspect-called-form [& argument]
         {:form (list 'quote &form)})
```

调用试试：

```sh
user=> ^{:doc "this is a doc metadata for the form"} (inspect-called-form 1 2 3)
{:form (inspect-called-form 1 2 3)}
user=> (meta (:form *1))
{:doc "this is a doc metadata for the form", :line 23, :column 1}
```

通过 `&form` 你可以随时获取调用当时的元信息。

而 `&env` 可以让你“偷窥”调用当时的局部绑定情况：

```sh
user=> (defmacro inspect-caller-locals []
         (->> (keys &env)
              (map (fn [k] [`'~k k]))
              (into {})))
#'user/inspect-caller-locals
user=> (let [foo "bar" baz "quux"] (inspect-caller-locals))
{baz "quux", foo "bar"}
```










