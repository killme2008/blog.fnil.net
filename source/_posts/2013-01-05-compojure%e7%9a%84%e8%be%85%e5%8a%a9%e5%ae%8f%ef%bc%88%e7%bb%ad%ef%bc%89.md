---
title: Compojure的辅助宏（续）
author: dennis_zhuang
layout: post
permalink: /index.php/archives/76
isc_post_images:
  - 'a:1:{s:0:"";a:2:{s:3:"src";b:0;s:9:"thumbnail";b:1;}}'
views:
  - 193
sfw_pwd:
  - SJbOmzb7JzyR
categories:
  - Clojure
  - 'Trick &amp; Tips'
tags:
  - clojure
  - compojure
  - trick
  - web
---
<div id="post-entry-excerpt-76" class="entry-part">
  <p>
    <a href="http://blog.fnil.net/index.php/archives/27">前一篇文章</a>提到Compojure的几个辅助宏defhandler,def-signin-handler和check-params，定义起handler function变得方便了不少，但是每次用check-params写参数检查之类的前置条件还是比较繁琐，例如：
  </p>
  
  <pre class="brush: clojure; notranslate">(defhandler signin [username password]
  (check-params
    (empty? username) "Username is required."
    (empty? password) "Password is required."
    :else
    (do-validation...)))
</pre>
  
  <p>
    每次都要这么写check-params仍然是比较郁闷的事情，其实我们完全可以将这些前置的检查条件作为handler的元信息附加进去，我们都知道Clojure的函数支持契约式编程，通过指定<code>:pre</code>和<code>:post</code>就可以设定函数的前置和后置条件，一个简单例子：
  </p>
  
  <pre class="brush: clojure; notranslate">(defn square [x]
     {:pre [(number? x)]
      :post [(pos? x)]}
     (* x x))
</pre>
  
  <p>
    :pre是前置条件，要求x必须为数字，:post为后置条件，要求返回结果为正数（其实这个条件是错误的，为啥？），测试看看：
  </p>
  
  <pre class="brush: clojure; notranslate">user=&gt; (square "a")
AssertionError Assert failed: (number? x)  user/square (NO_SOURCE_FILE:6)
user=&gt; (square 0)
AssertionError Assert failed: (pos? x)  user/square (NO_SOURCE_FILE:12)
user=&gt; (square 3)
9
</pre>
  
  <p>
    可以看到前置条件和后置条件都起作用了。内置于语言的契约式编程，让Clojure程序可以做到更健壮。
  </p>
  
  <p>
    回到原来的话题，同样，我们也希望defhandler的这些前置条件也可以类似函数的“契约”那样配置，如果不满足前置条件就返回失败，前置条件都满足才执行handler的body部分，这就需要对defhandler宏做一个改造，将check-params内置进来。具体的改造过程不说，我还是直接给结果，这个宏整体的结构是跟defn这个核心宏是一样的,首先我们将原来的defhandler宏改名为defhandler0，新定义一个宏称为defhandler：
  </p>
  
  <pre class="brush: clojure; notranslate">(defmacro defhandler0
    [name args & body]
    `(defn ~name [req#]
        (let [{:keys ~args :or {~'req req#}} (:params req#)]
        ~@body)))   
(defn has-pre? [fdecl]
    (and (map? (first fdecl)) (:pre (first fdecl))))
(def defhandler (fn defhandler [&form &env name & fdecl]
                  (let [m (if (string? (first fdecl))
                            {:doc (first fdecl)}
                            {})
                        m (if (map? (first fdecl))
                            (conj m (first fdecl))
                            m)
                        m (conj (if (meta name) (meta name) {}) m)
                        fdecl (if (map? (first fdecl))
                                (next fdecl)
                                fdecl)
                        args (if (vector? (first fdecl))
                               (first fdecl)
                               [])
                        fdecl (if (vector? (first fdecl))
                                (next fdecl)
                                fdecl)
                        pre (when (has-pre? fdecl)
                              (:pre (first fdecl)))
                        fdecl (if (has-pre? fdecl)
                                (next fdecl)
                                fdecl)]
                    (if pre
                      (list `defhandler0 (with-meta name m) args (concat (cons `check-params
                                                                          pre)
                                                                    (cons :else fdecl)))
                      (list* `defhandler0 (with-meta name m) args fdecl)))))
(.setMacro (var defhandler))
</pre>
  
  <p>
    macroexpand下，可以看出来原理很简单，就是检测有没有:pre条件，有的话将这些条件放到check-params里，将body放到check-params的:else部分。
  </p>
  
  <p>
    那么，现在前面的signin handler简化为：
  </p>
  
  <pre class="brush: clojure; notranslate">(defhandler signin [username password]
  {:pre [ (empty? username) "Username is required."
          (empty? password) "Password is required."]}
  (do-validation...))
</pre>
  
  <p>
    将参数检查这些前置条件都转移到:pre里面，核心的body只处理正常的逻辑，整体看起来清晰顺眼。defhandler不仅可以定义:pre，其实也可以关联metadata和doc:
  </p>
  
  <pre class="brush: clojure; notranslate">(defhandler signin "Signin handler" [username password] ...)
(defhandler signin {:doc "Signin handler" :added "0.1"} [username password] ...)
</pre>
  
  <p>
    也就是跟defn功能类似了。眼尖的朋友可能会注意到，defhandler宏其实有个隐患，如果参数vector后面是一个map，并且包含:pre就会被认为是前置条件，那么handler正常的返回结果就不能同时是个map并且包含:pre，这是个小小的限制。
  </p>
  
  <p>
    前一篇文章提到的def-signin-handler的改造也与此类似，两者的代码重复怎么消除，就请读者自行尝试吧。
  </p>
</div>

<div id="post-footer-76" class="post-footer clear">
</div>