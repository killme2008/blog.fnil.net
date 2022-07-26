---
layout: post
title: "最近做的一些 clojure 开源项目"
date: 2014-07-12 20:05:23 +0800
comments: true
categories: [clojure,开源] 
---

最近总结下最近的一些开发经验，形成几个 clojure 的开源项目：

## clj.qrcode：二维码生成

* [https://github.com/killme2008/clj.qrgen](https://github.com/killme2008/clj.qrgen)
* 示例：

```clojure
(use 'clj.qrgen)
(as-file (from "hello world"))
(as-bytes (from "hello world"))
(from (vcard "John Doe"
             :email "john.doe@example.org"
             :address "John Doe Street 1, 5678 Doestown"
             :title "Mister"
             :company "John Doe Inc."
             :phonenumber "1234"
             :website "www.example.org"))
```

## secure-rand：安全随机数生成器

* [https://github.com/killme2008/secure-rand](https://github.com/killme2008/secure-rand)
* 想要安全的随机数，还是要使用 `java.security.SecureRandom` 类
* 示例：

```clojure
(ns test
  (:refer-clojure :exclude [rand rand-int rand-nth])
  (:use [secure-rand.core :only [rand rand-int rand-nth]]))

(rand)
(rand 10)
(rand-int 100)
(rand-nth (range 10))
(secure-rand.core/base64 32)
```

## clj.qiniu：七牛云存储 SDK

* [https://github.com/killme2008/clj.qiniu](https://github.com/killme2008/clj.qiniu)
* 封装了官方Java SDK，增加了 bucket 统计和管理的 API:
* 示例

```clojure
(require '[clj.qiniu :as qiniu])
(qiniu/set-config! :access-key "your qiniu access key" :secret-key "your qiniu secret key")
(qiniu/upload-bucket bucket key file)

```

希望能给一些使用 clojure 开发的朋友带来帮助。


