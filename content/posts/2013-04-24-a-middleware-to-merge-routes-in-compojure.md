---
title: A middleware to merge routes in compojure
author: dennis_zhuang
layout: post
permalink: /index.php/archives/159
isc_post_images:
  - 
views:
  - 196
sfw_pwd:
  - CZF82JOCk2OP
  - 5m4b1OoTMQOb
categories:
  - Clojure
  - 开发心得
tags:
  - clojure
  - compojure
  - merge
  - routes
---
<div id="post-entry-excerpt-159" class="entry-part">
  <p>
    有时候，我们可能定义很多个不同的route，有的可能有context，有的没有，有的是动态请求，有的是静态请求，那么就有组合route的需求，利用compojure的routing函数即可做到：
  </p>
  
  <pre class="brush: clojure; notranslate"> (use 'compojure.core)
 (defn merge-routes [& handlers]
     (fn [req]
         (apply routing req handlers)))
</pre>
  
  <p>
    使用：
  </p>
  
  <pre class="brush: clojure; notranslate">(defroutes api-routes
    (context "/v1" [] our-routes))
(defroutes static-routes
    (route/resources "/console"))
(def app
    (handler/site (merge-routes api-routes static-routes) ......))
</pre>
</div>

<div id="post-footer-159" class="post-footer clear">
</div>