---
layout: post
title: "Clojure Web 编程之安全篇"
date: 2014-06-07 22:53:55 +0800
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

* 避免调用任何形式的`eval`函数，包括`eval`,`read-string`,`read`等函数。如果要读取客户端clojure数据，请使用`clojure.edn/read-string`替代`clojure.core/read-string`，因为 edn 有明确的[格式要求](https://github.com/edn-format/edn)。

### 其他注入

跟 XSS 攻击有关，要对任何用户输入并且需要渲染的内容做检测、清除或者转义危险标签等。

CRLF注入，通过提交带有`CR`和`LF`换行标记的字符串，达到XSS攻击的目的，通常是允许用户输出HTTP应答头的时候才会发生。因此，**禁止任何形式的允许用户自定义HTTP头，如果非要这样，请注意替换`\r`和`\n`字符**


## XSS攻击

[XSS 攻击] 本质上是通过提交恶意脚本来达到攻击目的。因此，

### 对输入做处理

* 使用一个服务端 validator 框架，Clojure 里可以选择[bouncer](https://github.com/leonardoborges/bouncer)或者[validateur](https://github.com/michaelklishin/validateur)
* 对输入做 escape，每个模板库都有提供这样的方式，不如`hiccup.util/escape-html`等。

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

* 启用 [HttpOnly]()，这可以防止大多数 XSS 攻击，因为没办法简单地获取 cookie 值了。
* 不要设置 Domain 属性，防止其他子域名安全漏洞影响到主站。
* 尽量不要设置 Expire 和 Max-Age 属性，防止浏览器本地磁盘保存 cookie。
* 使用简单的 cookie 名称，比如`id`，防止泄露业务逻辑。
* 如果使用 HTTPS，设置 Secure 属性为true来加密传输 cookie 。

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

Cookie 的加密可以采用一些对称机密算法，但是密钥应该随机产生并且每个用户、每次登录都应该不同，还需要设置一个合理的有效期。更好的策略是 cookie 值本身不带有业务信息（防止直接被解密），而是随机产生的标示符，在服务端根据这个标示符获取有效的用户信息做认证和授权。



## CSRF攻击

## 点击劫持

## 跨域AJAX调用

## 认证和授权

## Redirect和Forward

## 文件上传

## 加密

## 参考资料



