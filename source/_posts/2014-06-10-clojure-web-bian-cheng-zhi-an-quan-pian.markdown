---
layout: post
title: "Clojure Web 编程之安全篇"
date: 2014-06-10 22:53:55 +0800
comments: true
categories: Clojure,web编程
---

最近关注这方面稍微多了点，大概总结下。

## 基本原则

浏览器的安全机制：

* [同源策略](https://developer.mozilla.org/zh-CN/docs/Web/Security/Same-origin_policy):host、port、protocol、sub domain都能影响。
* 沙箱模型，比如 Chrome 的[多进程模型](http://www.ituring.com.cn/article/40164)。
* 恶意网址拦截，现代浏览器基本上都有提供，Google也提供了[开发API](https://developers.google.com/safe-browsing/)查询黑名单。

Clojure的Web开发本质上是基于 Java 的 Servlet 模型，因此也同样遵循 Java 的安全编程模型。这里主要描述 Clojure 里的常见防御策略，具体的漏洞请参考《白帽子讲Web安全》等书籍。

<!-- more -->

## 注入

包括 SQL 注入和其他类型的注入，比如XML、JavaScript甚至HTTP头。

### SQL注入

**任何情况下都应该避免拼接SQL语句，而应该使用参数化的SQL语句**

如果使用[clojure.java.jdbc]，使用占位符`?`替代参数:

``` clojure
(require '[clojure.java.jdbc :as j])
(j/query mysql-db
  ["select * from fruit where appearance = ?" "rosy"]
  :row-fn :cost)
```

如果使用[korma](http://sqlkorma.com/docs)，只要避免使用`exec-raw`执行拼接SQL语句，默认都是参数化SQL语句：

``` clojure
(select users
  (fields :id :first (raw "users.last"))
  (where {:first [like "%_test5"]}))

;;Or when all else fails, you can simply use exec raw
(exec-raw ["SELECT * FROM users WHERE age > ?" [5]] :results) 
```

### 代码注入

* 避免调用任何形式的`eval`函数，包括`eval`,`read-string`,`read`等函数。如果要读取客户端 clojure 结构的数据，请使用`clojure.edn/read-string`替代`clojure.core/read-string`，因为 edn 有明确的[格式要求](https://github.com/edn-format/edn)。

### 其他注入

跟 XSS 攻击有关，要对任何用户输入并且需要渲染的内容做检测、清除或者转义危险标签等。

CRLF注入，通过提交带有`CR`和`LF`换行标记的字符串，达到XSS攻击的目的，通常是允许用户输出HTTP应答头的时候才会发生。因此，**禁止任何形式的允许用户自定义HTTP头，如果非要这样，请注意替换`\r`和`\n`字符**


## XSS攻击

[XSS 攻击](https://www.owasp.org/index.php/Cross-site_Scripting_(XSS) 本质上是利用用户输入漏洞，通过提交恶意脚本来达到攻击目的。因此，

### 对输入做处理

* 使用一个服务端 validator 框架，Clojure 里可以选择[bouncer](https://github.com/leonardoborges/bouncer)或者[validateur](https://github.com/michaelklishin/validateur)
* 对输入做 escape，每个模板库都有提供这样的方式，比如`hiccup.util/escape-html`等。

### 对输出做处理

* 目前大多数 Clojure 模板引擎都会对变量输出做 escape 处理。
* 确实需要富文本输出的，使用 [OWASP](https://www.owasp.org/)的 [antisamy](https://code.google.com/p/owaspantisamy/) 库是最佳选择，它提供了多种可配置的策略来提供不同的安全级别需求。在Clojure里使用也不麻烦，一个简单封装：

``` clojure
(import '(org.owasp.validator.html Policy AntiSamy CleanResults))
(require '[clojure.java.io :as io])

(defonce ^Policy antisamy-policy (Policy/getInstance (io/input-stream (io/resource "antisamy-ebay-1.4.4.xml"))))
(defn sanitize-html
  [html]
  (let [^AntiSamy as (AntiSamy.)
        ^CleanResults cr (.scan as html antisamy-policy AntiSamy/SAX)]
    (.getCleanHTML cr)))
```

### Cookie处理

Cookie必须做到几点：

* 启用 [HttpOnly](https://www.owasp.org/index.php/HttpOnly)，这可以防止大多数 XSS 攻击，因为没办法简单地获取 cookie 值了。
* 不要设置 Domain 属性，防止网站下的其他子域名的安全漏洞影响到主站。
* 尽量不要设置 Expire 和 Max-Age 属性，防止浏览器本地磁盘保存 cookie。但是我们为了避免用户在一定时间重复登录，还是会设置的，不要设置太长的时间最好。
* 使用简单的 cookie 名称，比如`id`，防止泄露业务逻辑。
* 如果使用 HTTPS，设置 `Secure` 属性为`true`来加密传输 cookie 。

在 ring 里，使用`ring.middleware.cookies/wrap-cookies`中间件就可以启用 cookie，cookie就是一个普通的 map 结构，推荐的设置如下：

```
{
  :id {
         :http-only true
         :secure true
         :value "加密后的 cookie 值"
         :path "/"
      }
}
```

Cookie 的加密可以采用一些对称机密算法，但是密钥应该随机产生并且每个用户、每次登录都应该不同，还需要设置一个合理的有效期。更好的策略是 cookie 值本身不带有业务信息（防止直接被解密），而是随机产生的标示符，在服务端根据这个标示符从数据库或者缓存里获取有效的用户信息做认证和授权。



## CSRF攻击

[CSRF攻击](https://www.owasp.org/index.php/Cross-Site_Request_Forgery_(CSRF)（跨站请求攻击）也是常见的安全漏洞。

在 Clojure 里我们需要注意这么几点：

* 避免`ANY`路由，尽量使用明确的语义的`route`，查询走`GET`，创建走`POST`,更新走`PUT`，删除走`DELETE`等。
* 防止CSRF攻击的常见手段：验证码、Refer头检测、token检测。最靠谱的还是Token检测，每次提交都在 Form 带上一个服务端随机产生的 `csrf_token`，在服务端检测提交的 token 是否有效，如果有效才允许做创建、更新或者删除等变更性的操作。当然这个检测依赖的前提是网站没有 XSS 漏洞。如果 cookie 都暴露给攻击者了，那防御也没有意义。
* Clojure 里可以使用 [ring-anti-forgery](https://github.com/ring-clojure/ring-anti-forgery) 这个 middleware 来做自动化检测。它提供了一些辅助方法来产生和检测 token。
* 对于 Ajax 请求来说，Angular 这个 JavaScript 前端框架内置了一套 [csrf 防御机制](http://angularjs-best-practices.blogspot.jp/2013/07/angularjs-and-xsrfcsrf-cross-site.html)，只要你在 cookie 里种上`X-CSRFToken`值，那么他会在每次 AJAX 的 POST、PUT等请求上从 cookie 里获取并设置`X-XSRF-TOKEN`的 HTTP 头，你只需要在服务端检测这个 header 值是否跟 cookie 中的值是否一致就ok，我们可以写个 middleware:

``` clojure
(defn wrap-xsrf-token [handler]
  (fn [req]
    (if (#{:post :put :delete} (:request-method req))
      (if-let [xsrf (-> req :headers (get "x-xsrf-token"))]
        (if (= xsrf (-> req :cookies (get "XSRF-TOKEN") :value))
          (handler req)
          (fail 403 "Invalid access, invalid xsrf token."))
        (fail 403 "Invalid access, missing xsrf token."))
      (handler req))))
```

其中 `fail` 是一个返回错误应答状态码和信息的函数。当没有找到`X-XSRF-TOKEN`头，或者它的值跟 cookie 里的值不匹配的时候，我们都返回 403 状态码和错误信息。

## 点击劫持

[点击劫持](https://www.owasp.org/index.php/Clickjacking)本质上是利用隐藏的 iframe 框架实现的。关于他的客户端防御可以参考 owasp 的 [cheatsheet](https://www.owasp.org/index.php/Clickjacking_Defense_Cheat_Sheet)。

针对服务端来讲，我们可以为服务端渲染的模板添加`X-Frame-Options`的[应答 HTTP 头](https://developer.mozilla.org/en-US/docs/Web/HTTP/X-Frame-Options)，它可以是三个值：

* DENY，完全禁止框架
* SAMEORIGIN 允许当前同域的 frame 加载。
* ALLOW-FROM uri  允许指定 uri 的 frame 加载。

根据你的实际需要来设置，通常推荐设置为`SAMEORIGIN`:

```
    {:status 200
     :headers {"Content-Type" "text/html;charset=utf-8"
               "X-Frame-Options" "SAMEORIGIN"
               "Cache-Control" "no-cache;no-store"}
     :body "html...."
    }
```

在 nginx 里也可以为任何应答添加这个头：

```
add_header X-Frame-Options SAMEORIGIN;
```


## 跨域 AJAX 调用

HTML5 提出了 [CROS](https://developer.mozilla.org/en-US/docs/Web/HTTP/Access_control_CORS)机制来支持跨域 AJAX 请求。它将 HTTP 跨域请求分为两类：

* Simple 请求： 没有设置自定义HTTP头，并且`Content-Type`类型限制在`application/x-www-form-urlencoded, multipart/form-data`或者`text/plain`的范围内。这类请求可以直接发起，目标服务端通过`origin`头认证来决定是否接收请求，如果接受处理并应答，在应答里设置`Access-Control-Allow-Origin`头为通配符`*`或者请求里的`origin`值。
* Preflighted 请求： 需要自定义HTTP头，或者`Content-Type`类型在`application/x-www-form-urlencoded, multipart/form-data`或者`text/plain`的范围之外，比如提交 JSON 或者 XML 数据，那么需要预先发起一次`OPTIONS`请求，查询服务端是否接收这次请求。服务端根据 method,path,origin等信息来决定是否接收这次请求，如果接受，响应 options 应答里要设置`Access-Control-Allow-Origin,Access-Control-Allow-Methods,Access-Control-Allow-Headers,Access-Control-Max-Age`这几个头信息，然后客户端再发起真正的请求，具体见 Mozilla 的 [CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/Access_control_CORS) 文档吧。

跨域是很方便，特别是对于提供 Open API 的服务来说，但是也需要注意几点：

* 不要响应所有路径的`options`应答，慎重选择应答的路径，`(OPTIONS "/*" ...)`是最差的安全实践。只应该允许对外开放的 API 的 options 请求。
* Access-Control-Allow-Origin 的值尽量不要设置为`*`，而应该根据 options 请求的附带的信息来决定是否授权本次访问，如果授权访问，那么也应该设置成请求里的`origin`头的值，而非通配符。
* 限定`Access-Control-Allow-Methods`的范围在`GET,POST,OPTIONS`内,`PUT`和`DELETE`要不要开放需要仔细考量。
* 不要设置`Access-Control-Allow-Credentials`头，如果允许，那么跨域 AJAX 请求将被允许带上 cookie 和 Http Basic 认证信息，这通常是不合理的。


对于请求来源的`origin`判断，可以使用一些第三方服务来判断来源网站是否合法，比如 [金山网址云安全开放 API](http://code.ijinshan.com/api/devmore4.html#md1) 或者 [Google 开放 API](https://developers.google.com/safe-browsing/)。



## 认证和授权

Clojure 类似于 Spring Security 的类库就是 [friend](https://github.com/cemerick/friend)，基于角色的 ACL 权限模型。我没有研究仔细学习过。

如果没有那么复杂的权限模型，采用 [ring session](https://github.com/mmcgrana/ring/wiki/Sessions) 机制就足够实现一个简单的认证和授权模型。 Session 的存储可以使用 cookie。如果 session 存储的数据比较多（比较差的实践），在没有集群服务情况下，内存或者磁盘存储也足够了，如果是集群模型，可以考虑使用 memcached 或者 redis，过去写过一个 [ring-session-memcached](https://github.com/killme2008/ring-session-memcached) 就是基于 memcached 做存储的 ring session 中间件。

这篇[旧文](http://blog.fnil.net/index.php/archives/27/)里提到的`def-signin-handler`宏就是通过判断 cookie 的合法性来决定请求是否经过授权。

### Clojure 里实现 OAuth2 服务端

综合考察下来 [clauth](https://github.com/pelle/clauth) 应该是最成熟的方案。提供了三种授权类型：

* [Authorization Code Grant](http://tools.ietf.org/html/draft-ietf-oauth-v2-25#section-4.1)
* [Client Credential Grant](http://tools.ietf.org/html/draft-ietf-oauth-v2-25#section-4.4)
* [Resource Owner Password Credential Grant](http://tools.ietf.org/html/draft-ietf-oauth-v2-25#section-4.3)

并且存储也完全可以自定义，适配自己的业务数据存储，提供了简单的页面实现，可自主定制。整体来讲，完成度和成熟度比较高。

### Clojure 里实现 OAuth2 客户端

推荐采用 [clj-oauth2](https://github.com/DerGuteMoritz/clj-oauth2) 库。

在实现 OAuth2 认证客户端的时候，需要注意：

* 必须通过 HTTPS 加密传输，无论是 OAuth2 提供方，还是自己的 callback 回调请求。
* 必须使用`state`参数，并且传入一个随机产生值，对 callback 返回的 state 中的值做检测是否相等。这个随机值应该有过期时间（通常在3 ~ 5分钟内），并且每次请求都应该不同。这是为了防范 [OAuth2 CSRF攻击](http://stackoverflow.com/questions/11071482/oauth2-0-server-stack-how-to-use-state-to-prevent-csrf-for-draft2-0-v20)。当然，前提是 OAuth2 Provider 必须能正确返回 state 参数，国内不少网站提供的 OAuth2 协议就没有很好地支持 `state` 参数。


## Redirect和Forward

** 简单来讲，应该避免任何依赖用户输入的 redirect 和 forward 请求。 **

不应该使用用户传入的 URL 做跳转之类的 3xx 请求，这很容易被滥用。

## 文件上传

注意几点：

* 文件类型采用白名单机制，只有符合指定 MimeType 的文件才允许上传。简单的办法是通过检测文件后缀，更安全的做法是检测文件内容，比如使用 [Java 的 ImageIO 来检测图片](http://stackoverflow.com/questions/4169713/how-to-check-a-uploaded-file-whether-it-is-a-image-or-other-file。)
* 指定上传目录，默认 [ring.middleware.multipart-params](http://mmcgrana.github.io/ring/ring.middleware.multipart-params.html)会存储临时文件在系统的临时路径，通常是`/tmp`目录，你也可以通过`(wrap-multipart-params handler :store (fn [file] ...))`来自行决定存储文件到哪里。
* 对上传目录做安全限制，不允许执行权限。`chmod -R a-x /upload-dir`。
* 定期删除上传目录内的过期文件。
* 限制上传文件大小，可以在 Nginx 里配置（比如设置最大请求体为 10m）:

```
client_max_body_size 10m;
```

## 加密

### 随机数

Clojure的 `rand` 函数调用的是`Math.random`方法，实际调用的是`java.util.Random`类。

而在 Java 里产生更安全的随机数的推荐做法是使用 [java.security.SecureRandom](http://docs.oracle.com/javase/7/docs/api/java/security/SecureRandom.html)，这里就自荐下我刚放出去的 [secure-rand库](https://github.com/killme2008/secure-rand)。

它内部使用 ThreadLocal 缓存的 `SecureRandom` 做随机整数、字符串、byte数组生成，具体使用请看项目描述吧。

本来有一个开源的 [crypto-random](https://github.com/weavejester/crypto-random)，不过它每次调用都重新创建 SecureRandom，并且没有提供 clojure.core 里的`rand`、`rand-int`和`rand-nth`的替代方法。

### 加密算法

* 尽量使用 HMAC 算法替换 MD5 做加密签名或者数据完整性检查，参考 [Understanding MD5 Length Extension Attack](http://hi.baidu.com/aullik5/item/a4906012f04552fc9c778afa)。
* 使用 java.security 提供的加密算法，而不是自己实现。
* 加密用的 salt 应该随机产生，每次不同，并且使用安全的随机数发生器。

更多请参考 《白帽子讲web安全》第11章。



## 参考资料

* 书籍 《白帽子讲web安全》 《Java 加密与解密的艺术》
* [The Open Web Application Security Projec](https://www.owasp.org/index.php/Main_Page)
* [Clojure Web Security](http://www.lispcast.com/clojure-web-security)



