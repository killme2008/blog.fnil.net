---
title: Compojure的辅助宏
author: dennis_zhuang
layout: post
permalink: /index.php/archives/27
views:
  - 400
isc_post_images:
  - 'a:1:{s:0:"";a:2:{s:3:"src";b:0;s:9:"thumbnail";b:1;}}'
sfw_pwd:
  - h8YJQUPhLkB2
categories:
  - Clojure
  - 'Trick &amp; Tips'
tags:
  - clojure
  - compojure
  - macro
---
<div id="post-entry-excerpt-27" class="entry-part">
  <p>
    <a href="https://github.com/weavejester/compojure">Compojure</a>是clojure世界的web MVC框架事实上的标准，它使用很简单，核心的概念就一个routes，比如一个hello world级别的例子：
  </p>
  
  <pre class="brush: clojure; notranslate">(ns hello-world
    (:use compojure.core)
    (:require [compojure.route :as route]))

(defroutes app
    (GET "/" [] "&lt;h1&gt;Hello World&lt;/h1&gt;")
    (route/not-found "&lt;h1&gt;Page not found&lt;/h1&gt;"))
</pre>
  
  <p>
    它支持RESTFul风格的route设计，例如：
  </p>
  
  <pre class="brush: clojure; notranslate">(GET "/users/:uid" [uid] (get-user uid))
(POST "/users" [name email] (add-user name email))
</pre>
  
  <p>
    如果route和<code>add-user</code>,<code>get-user</code>都是同一个namespace的，这一切都还ok，只不过你需要写3次参数列表，两次在route定义的地方，一次在这些函数定义的地方，或者你也可以将这些函数直接写这routes定义的地方，但是这样看起来就不大爽利，更大的问题是不利于模块的清晰划分。当handler function比较多的时候，拆分namespace是很自然的选择，那么你的代码可能是这样：
  </p>
  
  <pre class="brush: clojure; notranslate">(ns user)
(defn get-user [uid] ...)
(defn add-user [name email] ...)

(ns handler)
(GET "/users/:uid" [uid] (user/get-user uid))
(POST "/users" [name email] (user/add-user name email))
</pre>
  
  <p>
    仍然需要在定义handler function和route的地方将参数列表写上三遍。或者你可以只写一次，这时候handler function默认接受一个参数http request，然后自己解析参数，也许是这样：
  </p>
  
  <pre class="brush: clojure; notranslate">(ns user)
(defn get-user [req]
    (let [uid (-&gt; req :params :uid)]
    ...)
(defn add-user [req]
    (let [name (-&gt; req :params :name)
          email (-&gt; req :params :email)]
     ...))

(ns handler)
(GET "/users/:uid"  [] user/get-user)
(POST "/users" [] user/add-user)
</pre>
  
  <p>
    这样一来，route定义的地方得到了简化，但是handler function却变的复杂起来，需要解析参数，也许可以利用destructring来自动解构函数，但是仍然不够直观。
  </p>
  
  <p>
    接下来，我们就让一个宏来帮我们解决问题吧，希望既能少写代码，又能跟过去一样直观：
  </p>
  
  <pre class="brush: clojure; notranslate">(defmacro defhandler
    [name args & body]
    `(defn ~name [req#]
        (let [{:keys ~args :or {~'req req#}} (:params req#)]
        ~@body)))
</pre>
  
  <p>
    这个宏很简单，调用defhandler最终会调用defn生成一个handler function，只不过我们利用destructring帮我们多做了点工作：将args的参数列表自动跟<code>(:params req)</code>匹配起来，同时，如果<code>args</code>里有<code>req</code>这个名称的参数，我们将它关联到实际的request对象，这时候定义<code>get-user</code>,<code>add-user</code>变的和以前一样直观：
  </p>
  
  <pre class="brush: clojure; notranslate">(ns user)
(defhandler get-user [uid] ...)
(defhandler add-user [name email] ...)    

(ns handler)
(GET "/users/:uid"  [] user/get-user)
(POST "/users" [] user/add-user)
</pre>
  
  <p>
    注意到，我们只是用defhandler替换了defn，其他都没有改变，route定义保持简单和直观。
  </p>
  
  <p>
    更进一步，有的handler function需要登录才能使用，我们可以定义一个def-signin-handler，要求request必须有cookie（这个宏的原始版本属于我的同事孙宁）：
  </p>
  
  <pre class="brush: clojure; notranslate">(defn fail [msg] ...render error message)
(defmacro def-signin-handler [name args & body]
    `(defn ~name [req#]
        ;;这里只判断cookie是否为nil，实际应用还需做合法性校验，防止伪造
        (if (not (nil? (-&gt; req# :cookies (get "my_cookie"))))
            (let [{:keys ~args :or {~'req req#}} (:params req#)]
                ~@body)
            (fail "User doesn't sign in."))))
</pre>
  
  <p>
    接下来你可以利用def-signin-handler定义需要登录的handler function，它会自动判断用户是否登录（通过cookie），然后决定是否继续执行。
  </p>
  
  <p>
    最后，奉上一个基于cond改造的参数校验宏：
  </p>
  
  <pre class="brush: clojure; notranslate">(ns my-ns.util)
(defn fail [msg] ...render error message)
(defmacro check-params
  [& clauses]
  (when clauses
    (list 'if (first clauses)
          (if (next clauses)
            (if (next (next clauses))
              (fail (second clauses))
              (second clauses))
            (throw (IllegalArgumentException.
                    "check-params requires an even number of forms")))
          (cons 'my-ns.util/check-params (next (next clauses))))))
</pre>
  
  <p>
    使用例子：
  </p>
  
  <pre class="brush: clojure; notranslate">(defhandler signup [username password]
   (check-params
     (empty? username) "User name is required."
     (empty? password) "Password is required."
     :else
     (do ...check user name password and return result.)))
</pre>
</div>

<div id="post-footer-27" class="post-footer clear">
</div>