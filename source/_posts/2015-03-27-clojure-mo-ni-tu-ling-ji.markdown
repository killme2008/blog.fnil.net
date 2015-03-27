---
layout: post
title: "Clojure 模拟图灵机"
date: 2015-03-27 17:20:33 +0800
comments: true
categories: 计算理论 clojure
---

最近在读[《计算的本质：深入剖析程序和计算机 》](http://book.douban.com/subject/26148763/)，一本关于计算理论的小册子，使用 Ruby 语言介绍计算理论，第一步分从状态机开始直接，DFA/NFA、自动下推机直到图灵机，并且每个小章节都给出了代码例子。第二部分开始介绍 lambda 、丘奇数、停机问题等函数式编程的基础知识，挺好玩的一个阅读过程。

作者提供的 Ruby 代码在[这里](https://github.com/tomstuart/computationbook)。

我试着用 Clojure 重新实现了里面的图灵机模拟器的例子，

```clojure
(ns cljcomputionbook.tm
  (:require [clojure.string :as cs]))

;;磁带
(defrecord Tape [left middle right blank]
  Object
  (toString [tape]
    (pr-str tape)))

(defmethod print-method Tape [tape writer]
  (.write writer (format "#<Tape %s(%s)%s>"
                         (cs/join (:left tape))
                         (:middle tape)
                         (cs/join (:right tape)))))

(defn write [{:keys [left right blank]} ch]
  (Tape. left ch right blank))

(defmulti move-head (fn [tape direction] direction))

(defmethod move-head :left [{:keys [left middle right blank]} _]
  (Tape.
   (butlast left)
   (or (last left)
       blank)
   (concat [middle] right)
   blank))

(defmethod move-head :right [{:keys [left middle right blank]} _]
  (Tape.
   (concat left [middle])
   (or (first right)
       blank)
   (next right)
   blank))

;;配置格子
(defrecord TMConfiguration [state tape])

(defprotocol Rule
  (applies-rule? [this conf])
  (follow-rule [this conf]))

(defn- next-tape [tape write_character direction]
  (->
   tape
   (write write_character)
   (move-head direction)))

;;规则
(defrecord TMRule [state character next_state write_character direction]
  Rule
  (applies-rule? [this conf]
    (when (and
           (= state (:state conf))
           (= character (-> conf :tape :middle)))
      this))
  (follow-rule [this conf]
    (TMConfiguration.
     next_state
     (next-tape (:tape conf) write_character direction))))

(defprotocol Rulebook
  (next-configuration [this conf])
  (rule-for [this conf])
  (applies-to? [this conf]))

(defrecord DTMRulebook [rules]
  Rulebook
  (next-configuration [this conf]
    (follow-rule
     (rule-for this conf)
     conf))
  (rule-for [this conf]
    (some
     #(applies-rule? % conf)
     rules))
  (applies-to? [this conf]
    ((comp not nil?)
     (rule-for this conf))))

;; 图灵机模拟器
(defrecord DTM [current_configuration accept_states rulebook debug])

(defn accepting? [{:keys [accept_states current_configuration]}]
  (boolean
   (some (partial = (:state current_configuration))
         accept_states)))

(defn stuck? [{:keys [rulebook current_configuration] :as tm}]
  (and
   (not (accepting? tm))
   (not
    (applies-to? rulebook
                 current_configuration))))

(defn- debug-tm [{:keys [current_configuration debug] :as tm}]
  (when debug
    (println "DEBUG: "
             (merge
              (select-keys current_configuration [:state :tape])
              {:accepting? (accepting? tm)
               :stuck? (stuck? tm)}))))

;;单步执行
(defn step [{:keys [current_configuration accept_states rulebook debug]
             :as tm}]
  (debug-tm tm)
  (DTM.
   (next-configuration rulebook current_configuration)
   accept_states
   rulebook
   debug))

;;模拟运行，直到 accept 或者 stuck
(defn run [tm]
  (if (or (accepting? tm)
          (stuck? tm))
    (do
      (when (:debug tm)
        (debug-tm tm))
      tm)
    (recur
     (step tm))))
```

然后编写一个递增二进制数字的规则，就可以模拟运行了：

```clojure

;;定义递增规则
(def rulebook
  (DTMRulebook.
   [(TMRule. 1 0 2 1 :right)
    (TMRule. 1 1 1 0 :left)
    (TMRule. 1 '_ 2 1 :right)
    (TMRule. 2 0 2 0 :right)
    (TMRule. 2 1 2 1 :right)
    (TMRule. 2 '_ 3 '_ :left)]))

;;初始磁带，初始值为二进制 0b1011
(def tape (Tape. [1 0 1] 1 [] '_))

;;运行模拟器
(let [dtm (DTM. (TMConfiguration. 1 tape)
                [3]
                rulebook
                true)
      ran-dtm (run dtm)]
  ;;是否到达接受状态
  (println (accepting? ran-dtm)))
```

Debug 输出：

```sh
DEBUG:  {:stuck? false, :accepting? false, :tape #<Tape 101(1)>, :state 1}
DEBUG:  {:stuck? false, :accepting? false, :tape #<Tape 10(1)0>, :state 1}
DEBUG:  {:stuck? false, :accepting? false, :tape #<Tape 1(0)00>, :state 1}
DEBUG:  {:stuck? false, :accepting? false, :tape #<Tape 11(0)0>, :state 2}
DEBUG:  {:stuck? false, :accepting? false, :tape #<Tape 110(0)>, :state 2}
DEBUG:  {:stuck? false, :accepting? false, :tape #<Tape 1100(_)>, :state 2}
DEBUG:  {:stuck? false, :accepting? true, :tape #<Tape 110(0)_>, :state 3}
true
```

二进制数据从 `1011` 递增为 `1100` 了。

