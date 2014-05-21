---
title: Clojure 1.5学习综述
author: dennis_zhuang
layout: post
permalink: /index.php/archives/267
isc_post_images:
  - 
sfw_pwd:
  - vNyDNK43pZMw
views:
  - 164
categories:
  - Clojure
---
<div id="post-entry-excerpt-267" class="entry-part">
  <p>
    这博客来的有点晚，clojure 1.5都发布一年多了。不过我最近才将系统的clojure版本从1.4升级到1.5.1，因此需要重新看一遍clojure 1.5带来的新东西。
  </p>
  
  <h2>
    Reducer库
  </h2>
  
  <p>
    要说clojure 1.5最大的变化应该说是引入了<a href="http://clojure.com/blog/2012/05/08/reducers-a-library-and-model-for-collection-processing.html">reducer库</a>。单纯从这个库提供的函数如map,reduce,filter,remove,flattern等函数来说，跟clojure.core提供的这些函数的功能基本上是一致的，除了返回值略有不同。
  </p>
  
  <p>
    为什么要引入reducer库？或者说引入reducer库带来的好处是什么？我想主要是性能。我们都知道clojure的数据结构，如vector、list、hash-map等都是immutable——每次更新都将产生一个新的数据结构，而不是修改其中的某个节点。为了减少不可变带来的开销，clojure将这些集合类都实现为持久数据结构（persist），每次更新都将复用大部分“老”的数据；并且引入了LazySeq，将map,filter等高阶函数开销分摊到每个迭代步骤中，而不是重复的生成中间数据结构；每个迭代步骤还是批量的，一次一个chunk（32个元素）；通过这些手段来减少不可变数据结构的开销。
  </p>
  
  <p>
    但是呢，Lazy和Chunk都引入了额外的对象创建开销，并且很多时候我们最终都要realized整个数据结构（比如反馈查询结果给用户，查询结果可能是个LazySeq），lazy的额外开销是不必要的；其次，map,reduce,remove等这些高阶函数都基于Sequence的抽象之上，基于Sequence提供的first和next操作来遍历迭代数据结构，这两个操作对于不同的数据结构来说未必是最佳的迭代方式，并且需要将其他结构都转为sequence，也是一个额外的开销。例如对于字符串，最好的迭代其实是使用String的charAt来迭代字符，但是map等函数都会调用seq函数将字符串转成Sequence，多了一层包装，然后统一以first/next的方式来迭代处理。
  </p>
  
  <p>
    综上所述，core库提供的这些高阶函数，仍然是以“流”的方式在转换数据结构。而reducers则不是。它转换的是想要作用的“函数”，因此它是完全函数式的，而非迭代式。比如(map f coll)，clojure.core转换的是coll集合成另一个集合，将f函数作用在coll上得到一个新集合。而reducer的map函数则是转换f函数，将f转换为支持reduce的方式来迭代数据结构，满足下列条件：
  </p>
  
  <ul>
    <li>
      (f) 返回一个identity value。
    </li>
    <li>
      (f ret [k] v)作用在reduce值和每个item上。
    </li>
  </ul>
  
  <p>
    其他filter,remove也是这样，转换的都是f函数，而非集合。最终这些高阶函数生成一个所谓reducible，可以最终调用reduce或者into函数来得到“真正”的结果。关于这个实现，推荐看这篇<a href="http://clojure.com/blog/2012/05/15/anatomy-of-reducer.html">博客</a>和<a href="https://github.com/clojure/clojure/blob/master/src/clj/clojure/core/reducers.clj">源码</a>。
  </p>
  
  <p>
    其次，reducer库还让每个数据结构自己决定处理方式，比如刚才提到的，String的最佳方式就是直接使用chartAt，java数组的最佳方式就是利用一系列array函数来原生操作数组等等，这些都放在<a href="https://github.com/clojure/clojure/blob/master/src/clj/clojure/core/protocols.clj">protocol源码 里</a>。现在core库里的reduce和into也是调用这些优化过的方式。
  </p>
  
  <p>
    最后，reduce也是以eager方式，而非lazy的方式来生成集合，这也避免了lazy的开销。
  </p>
  
  <p>
    Reducer库还基于fork/join框架提供了fold函数来并发处理数据集合，默认切分的子集合大小是512，fold接收reduce和combine函数，其中combine函数需要满足：
  </p>
  
  <ul>
    <li>
      (combinef)返回一个初始值（identity value）
    </li>
    <li>
      combinef需要是可结合的，满足算术结合律，例如加法运算。
    </li>
  </ul>
  
  <p>
    fold的运行过程是并发的，但是结果将保持有序。
  </p>
  
  <p>
    关于reducer库的例子，我这里就不举了，有兴趣看看官方博客和<a href="http://adambard.com/blog/clojure-reducers-for-mortals/">这篇博客</a>就清楚了。
  </p>
  
  <h2>
    新的thread宏
  </h2>
  
  <p>
    clojure 1.5引入了几个新的thread宏，cond->和cond->>，as->以及some->和some->>等。这些也自己doc看看文档和例子就好，没有太多好提的，都是为了减少重复代码，提高代码的可读性而引入的。我对clojure标准库引入这些新宏持保留态度，核心库还是维持在较小的规模上更合适，不能因为哪个方便就随便加入。clojure 1.6貌似没有再加入一些新的thread宏。
  </p>
  
  <h2>
    gen-class和protocol的相关改进
  </h2>
  
  <ul>
    <li>
      gen-class增加指令exposes-methods导出protocted并final的方法，让通过gen-class生成的子类可以访问。
    </li>
    <li>
      gen-class的constructors可以添加annotation元信息。
    </li>
    <li>
      允许定义标记protocol，类似java里的mark interface，没有任何方法，纯粹一个接口
    </li>
  </ul>
  
  <p>
    看例子：
  </p>
  
  <pre class="brush: clojure; notranslate">user=&gt; (defprotocol M)
M
user=&gt; (deftype T [a] M)
user.T
user=&gt; (satisfies? M (T. 1))
true
</pre>
  
  <h2>
    hash-set和hash-map接收重复参数
  </h2>
  
  <p>
    在clojure 1.5之前，下面这个调用将失败：
  </p>
  
  <pre class="brush: clojure; notranslate">user=&gt; (hash-map :a 1 :b 2 :a 3)
IllegalArgumentException Duplicate key: :a
</pre>
  
  <p>
    hash-set也是类似，报错信息告诉你重复的key，在clojure 1.5里这样可以了：
  </p>
  
  <pre class="brush: clojure; notranslate">user=&gt; (hash-map :a 1 :b 2 :a 3)
{:a 3, :b 2}    
</pre>
  
  <p>
    但是字面量的仍然不行：
  </p>
  
  <pre class="brush: clojure; notranslate">user=&gt; #{1 1 2}
IllegalArgumentException Duplicate key: 1    
</pre>
  
  <p>
    哪怕你是通过表达式生成：
  </p>
  
  <pre class="brush: clojure; notranslate">user=&gt; (def x 2)
#'user/x
user=&gt; (def y 4)
#'user/y
user=&gt; #{(inc x) (dec y)}
IllegalArgumentException Duplicate key: 3    
</pre>
  
  <h2>
    edn库
  </h2>
  
  <p>
    新的clojure.edn库，读取解析<a href="https://github.com/edn-format/edn">edn格式</a>的数据，edn格式用在了Rich公司的主打产品Datomic数据库等。
  </p>
  
  <h2>
    性能改进
  </h2>
  
  <p>
    值得一提的一个是Multimethod使用读写锁来保护类型-方法派发表，原来是一个synchronized锁保护起来。读写锁对于multimethod表这种读远远多于写（一般不会再动态添加method）的场景，能大大地提升性能。
  </p>
  
  <h2>
    更多信息
  </h2>
  
  <p>
    请看完整的<a href="https://github.com/clojure/clojure/blob/master/changes.md">changelog</a>。
  </p>
  
  <p>
    clojure 1.6处于alpha状态，有兴趣也可以看看它的changelog。
  </p>
  
  <h2>
    当然，还有core.async
  </h2>
  
  <p>
    <a href="https://github.com/clojure/core.async">async库</a>应该说是使用clojure 1.5的一大理由，不过这是另一篇博客了……
  </p>
</div>

<div id="post-footer-267" class="post-footer clear">
</div>