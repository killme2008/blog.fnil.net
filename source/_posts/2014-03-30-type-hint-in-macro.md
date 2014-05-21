---
title: Type hint in macro
author: dennis_zhuang
layout: post
permalink: /index.php/archives/282
isc_post_images:
  - 
views:
  - 39
sfw_pwd:
  - pyvXm22hdqyg
  - pyvXm22hdqyg
categories:
  - Clojure
---
<div id="post-entry-excerpt-282" class="entry-part">
  <p>
    Type hint（类型提示）是Clojure性能优化的关键手段之一，关于type hint可以看看我过去写的这篇博客《<a href="http://www.blogjava.net/killme2008/archive/2012/07/10/382738.html">Clojure笔记：用好Type Hint</a>》。
  </p>
  
  <p>
    不过旧文里没有提到怎么在宏里面使用type hint，我们试试：
  </p>
  
  <pre class="brush: clojure; notranslate">user=&gt; (set! *warn-on-reflection* true)
true
user=&gt; (defmacro str-len
         [s]
         `(.length ^String ~s))
#'user/str-len
user=&gt; (str-len "test")
4
user=&gt; (def a "test")
#'user/a
user=&gt; (str-len a)
Reflection warning, NO_SOURCE_PATH:11:1 - reference to field length can't be resolved.
4
</pre>
  
  <p>
    我们打开了反射警告，定义了一个宏用来获取参数字符串的长度，里面用了Type Hint标记s的类型为String。然后获取字符串&#8221;test&#8221;的长度。当直接传入&#8221;test&#8221;的没有反射警告，返回长度4。当绑定了&#8221;test&#8221;到a，然后获取a长度的时候，反射警告产生了<code>reference to field length can't be resolved</code>。
  </p>
  
  <p>
    为什么字面量(literal)的字符串&#8221;test&#8221;就不会产生反射警告，而var a却会呢？道理很简单，macro会在编译的时候展开，因此<code>(str-len "test")</code>等价于<code>(.length "test")</code>，这个表达式无需type hint就可以得到答案4。事实上如果你给字面量加上type hint，clojure会报错：
  </p>
  
  <pre class="brush: clojure; notranslate">user=&gt; (.length ^String "test")
IllegalArgumentException Metadata can only be applied to IMetas....
</pre>
  
  <p>
    最关键的是，str-len里的type hint其实被“忽略”了，所以当定义a的时候去调用str-len，反射仍然产生。
  </p>
  
  <p>
    正确在宏里使用type hint的方法是使用metadata，所谓type hint宏本质上是给var的metadata加上tag值，因此我们可以用with-meta函数来给符号s加上type hint:
  </p>
  
  <pre class="brush: clojure; notranslate">user=&gt; (defmacro str-len
     [s]
     `(.length ~(with-meta s {:tag `String})))
user=&gt; (str-len a)
4
</pre>
  
  <p>
    Cool! It works。但是：
  </p>
  
  <pre class="brush: clojure; notranslate">user=&gt; (str-len "test")
ClassCastException java.lang.String cannot be cast to clojure.lang.IObj
</pre>
  
  <p>
    这是因为with-meta只能作用在clojure的对象上，而Java的String是没有实现IObj接口的，因此无法打上tag。有没有两全其美的办法呢？答案是有的，我们知道，任何问题都可以通过抽象一层来解决：
  </p>
  
  <pre class="brush: clojure; notranslate">(defmacro str-len
  [s]
  `(let [^String x# ~s]
         (.length x#)))
</pre>
  
  <p>
    引入中间symbol来打上tag。
  </p>
</div>

<div id="post-footer-282" class="post-footer clear">
</div>