---
title: 当LazySeq遇上closure
author: dennis_zhuang
layout: post
permalink: /index.php/archives/252
isc_post_images:
  - 'a:1:{i:258;a:1:{s:3:"src";s:67:"http://blog.fnil.net/wp-content/uploads/2014/01/Snip20140119_11.png";}}'
views:
  - 120
sfw_pwd:
  - jNRt1qhZUa0M
categories:
  - Clojure
---
<div id="post-entry-excerpt-252" class="entry-part">
  <p>
    LazySeq在Clojure里很关键，它解决了持久数据结构在多次操作中的性能问题，避免了多趟(pass)扫描数据结构，将性能的开销“分担”到一个一个的“实现”的步骤里去（当然，这一步可能比较大，一个chunk是32个元素）。
  </p>
  
  <p>
    正因为有LazySeq，
  </p>
  
  <pre class="brush: clojure; notranslate">(-&gt;&gt; (get-titles)
     (map compute-checksum)
     (filter verify-ok?)
     count)
</pre>
  
  <p>
    也只是需要扫描一趟。
  </p>
  
  <p>
    但是，只要LazySeq的头元素(head)仍然被持有（专业点属于可能叫reachable），它就没办法被回收，并且是所有已经reliazed的元素都将持续保存在内存里面。 如果是个大的数据结构，那将占用相当多的内存，影响性能。
  </p>
  
  <p>
    当LazySeq遇上Closure的时候，悲剧可能就发生了。看一个例子：
  </p>
  
  <pre class="brush: clojure; notranslate">user=&gt; (defn f [g] (g))
#'user/f 
user=&gt; (defn t1  (f (fn [] (dorun (map identity c)))))
#'user/t1
user=&gt; (t1 (range 1e8))
OutOfMemoryError Java heap space  clojure.lang.ChunkBuffer.&lt;init&gt; (ChunkBuffer.java:20)
</pre>
  
  <p>
    f函数只是执行传入的函数，t1方法是构造一个匿名函数并传给f执行（为什么这么做？这只是例子……），匿名函数中调用dorun强制map返回的LazySeq“实现”下，但是dorun不会持有LazySeq的head，因此理论上不应该会有内存溢出，比如我们直接这样跑是不会有内存溢出的：
  </p>
  
  <pre class="brush: clojure; notranslate">user=&gt; (dorun (map identity (range 1e8)))
nil
</pre>
  
  <p>
    但是一放到匿名函数里，并让f来执行就出现内存溢出了。为什么呢？
  </p>
  
  <p>
    &#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8211; 我是分割线 &#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;-
  </p>
  
  <p>
    问题在于匿名函数使用了非参数(没有在匿名函数的参数列表[]里出现)的外部参数c，这形成了一个closure，更专业点，我们叫closed-over closure。在SICP里，只有引用了“自由变量”的函数才可以称为closure，不过通常我们习惯性地将所有匿名函数也称为closure。这里的“自由变量”就是t1的参数c，它被匿名函数closed over了。我们接地气一些，就叫“保存”了这个变量c，并且这个匿名函数可以在多次调用中使用这个c。
  </p>
  
  <p>
    因此原因很简单，LazySeq被dorun“实现”了之后，虽然dorun不持有head，但是匿名函数持有这个LazySeq“实现”后的集合，并且匿名函数无法在f调用完成之前被释放，导致内存被占满并溢出。
  </p>
  
  <p>
    &#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8211; 我是分割线 &#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;-
  </p>
  
  <p>
    解决办法，古怪的once和fn*出场：
  </p>
  
  <pre class="brush: clojure; notranslate">user=&gt; (defn t2  (f (^:once fn* [] (dorun (map identity c)))))
#'user/t2
</pre>
  
  <p>
    这里用fn*而不是fn只是因为fn不可以传入metadata，而^:once就是一个metadata，告诉clojure的编译器说，这个匿名函数的Closure只会被调用一次，不要让Closure继续保存“自由变量”（或者clojure里叫LocalBinding）。让我们测试下（请确保在OOM之后重启过REPL，前面的REPL已经填满了垃圾元素）：
  </p>
  
  <pre class="brush: clojure; notranslate">user=&gt; (t2 (range 1e8))
nil   
</pre>
  
  <p>
    Cool! It works.问题顺利解决。虽然这里once的用法很怪异。不过这个用法在clojure.core里很普遍，比如future宏：
  </p>
  
  <pre class="brush: clojure; notranslate">(defmacro future
  [& body] `(future-call (^{:once true} fn* [] ~@body)))    
</pre>
  
  <p>
    这个问题的更多讨论请参考<a href="http://clj-me.cgrand.net/2013/09/11/macros-closures-and-unexpected-object-retention/">这篇博客</a>和这个<a href="https://groups.google.com/forum/#!msg/clojure/iw7Rwp7wmjo/VSw40hboo1YJ">论坛帖子</a>。上面的例子就来自那里。
  </p>
  
  <p>
    &#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8211; 我是分割线 &#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;-
  </p>
  
  <p>
    没有结束，我们再稍微深入一些，看看Clojure是怎么特殊处理once元信息的。首先，请遵循我过去写的<a href="http://www.blogjava.net/killme2008/archive/2010/07/11/325775.html">《Clojure Hacking Guide》</a>来打印下t1和t2的字节码，我将打印出来的字节码存入文件并opendiff了一下，基本上两者生成的字节码是一致的，除了一个地方：
  </p>
  
  <p>
    <a href="http://blog.fnil.net/wp-content/uploads/2014/01/Snip20140119_11.png"><img src="http://blog.fnil.net/wp-content/uploads/2014/01/Snip20140119_11.png" alt="Snip20140119_1" width="1263" height="168" class="alignnone size-full wp-image-258" /></a>
  </p>
  
  <p>
    左边是t1，右边是t2。
  </p>
  
  <p>
    t2的匿名函数做了一个很关键的事情，就是通过PUTFIELD将自己内部的c的值设置为null（就是ACONST_NULL指令压入栈的null值），就这样“抛弃”了本该持有的c集合。
  </p>
  
  <p>
    具体到<a href="https://github.com/clojure/clojure/blob/master/src/jvm/clojure/lang/Compiler.java#L4827-4832">Clojure编译器在Github的源码)</a>，关键代码是这样：
  </p>
  
  <pre class="brush: java; notranslate">if(onceOnly && clear && lb.canBeCleared)
{
gen.loadThis();
gen.visitInsn(Opcodes.ACONST_NULL);
gen.putField(objtype, lb.name, OBJECT_TYPE);
} 
</pre>
  
  <p>
    真相大白，水落石出，淫者见淫，智者见智……
  </p>
</div>

<div id="post-footer-252" class="post-footer clear">
</div>