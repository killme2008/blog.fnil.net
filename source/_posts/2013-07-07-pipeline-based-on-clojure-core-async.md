---
title: Pipeline based on clojure core.async
author: dennis_zhuang
layout: post
permalink: /index.php/archives/176
isc_post_images:
  - 
views:
  - 195
sfw_pwd:
  - bfimpKNNKQNk
categories:
  - Clojure
  - 并发
---
<div id="post-entry-excerpt-176" class="entry-part">
  <p>
    Show me the code:
  </p>
  
  <pre class="brush: clojure; notranslate">(require ' [ clojure.core.async :as async :refer :all ])
(defn input [ source ]
  (when source
    (&lt;!! (source))))

(defn output [x]
  (go x))

(defn default-handler [x]
  (output x))

(defn default-processor [handler source]
  (when-let [x (input source)]
    (handler x)))

(defn pipeline-element [& opts]
  (let [{:keys [handler processor]
         :or {handler default-handler
              processor default-processor}} opts]
    (fn [ source ]
      (fn []
        (processor handler source)))))

(defmacro | [& fns]
  `(-&gt; ~@fns))
</pre>
  
  <p>
    Then create a pipeline to add line number for lines read from stdin:
  </p>
  
  <pre class="brush: clojure; notranslate">(def producer ((pipeline-element :processor (fn [handler _]
                                                  (handler (read-line)))) nil))
(def line-filter (let [line (atom 1)]
                  (pipeline-element :handler (fn [x]
                                               (let [x (format "%5d %s" @line x)]
                                                 (swap! line inc)
                                                 (output x))))))

(def consumer (pipeline-element :handler (fn [x]
                                               (println x))))

(def x (| producer line-filter consumer))
</pre>
  
  <p>
    Output:
  </p>
  
  <pre class="brush: shell; notranslate">user=&gt; (x)
  #_=&gt; hello world
    1 hello world
nil
user=&gt; (x)
  #_=&gt; again
    2 again
nil
</pre>
  
  <p>
    <a href="https://github.com/clojure/core.async">core.async</a> is really awesome!
  </p>
</div>

<div id="post-footer-176" class="post-footer clear">
</div>