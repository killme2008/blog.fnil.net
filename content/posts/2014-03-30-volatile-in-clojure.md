---
title: Volatile in clojure
author: dennis_zhuang
layout: post
permalink: /index.php/archives/284
isc_post_images:
  - 
views:
  - 67
sfw_pwd:
  - 3H2uGMYWFVHh
categories:
  - Clojure
---
<div id="post-entry-excerpt-284" class="entry-part">
  <p>
    Java 1.5重新理顺了内存模型，使得volatile关键字的行为更清晰和明确。怎么在Java里使用volatile，可以看看这篇旧文《<a href="http://www.ibm.com/developerworks/cn/java/j-jtp06197.html">Java理论与实践：正确使用Volatile</a>》。
  </p>
  
  <p>
    在Clojure里又怎么声明一个volatile变量呢？答案是volatile-mutable的metadata。一段Java代码：
  </p>
  
  <pre class="brush: java; notranslate">public class Person {
  volatile long age;
}
</pre>
  
  <p>
    等价的Clojure代码是：
  </p>
  
  <pre class="brush: clojure; notranslate">(deftype Person [^:volatile-mutable ^long age])
</pre>
  
  <p>
    也可以写成：
  </p>
  
  <pre class="brush: clojure; notranslate">(deftype Person [^{:volatile-mutable true :tag long} age])
</pre>
  
  <p>
    具体到<a href="https://github.com/clojure/clojure/blob/master/src/jvm/clojure/lang/Compiler.java#L4721-4723">编译器</a>：
  </p>
  
  <pre class="brush: java; notranslate">boolean isVolatile(LocalBinding lb){
return RT.booleanCast(RT.contains(fields, lb.sym)) &&
           RT.booleanCast(RT.get(lb.sym.meta(), Keyword.intern("volatile-mutable")));
}
</pre>
  
  <p>
    如果有volatile-mutable标记，就给access modifier加上AC_VOLATILE。
  </p>
</div>

<div id="post-footer-284" class="post-footer clear">
</div>