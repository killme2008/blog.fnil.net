---
title: Hello,Docker
author: dennis_zhuang
layout: post
permalink: /index.php/archives/238
isc_post_images:
  - 
views:
  - 127
sfw_pwd:
  - oCIRMKtzt31c
categories:
  - Uncategorized
  - 虚拟化
---
<div id="post-entry-excerpt-238" class="entry-part">
  <p>
    <a href="http://docker.io/">Docker</a> is an open-source project to easily create lightweight, portable, self-sufficient containers from any application.
  </p>
  
  <h1>
    安装
  </h1>
  
  <p>
    （略），见<a href="http://docs.docker.io/en/latest/installation/ubuntulinux/">官方文档</a>
  </p>
  
  <p>
    ubuntu安装很容易，不过要注意内核版本，至少3.8以上。安装成功后，使用lxc-checkconfig检查下LXC配置是否正常。
  </p>
  
  <h1>
    避免sudo docker
  </h1>
  
  <p>
    docker命令跟默认docker daemon创建的unix socket通讯都需要sudo权限，这可以通过创建docker组来解决。
  </p>
  
  <pre class="brush: shell; notranslate">sudo groupadd docker
#将当前用户加入docker组
sudo gpasswd -a ${USER} docker
#重启docker服务
sudo service docker restart
</pre>
  
  <p>
    这样一来，docker命令的执行就不再需要sudo权限。如果没有生效，退出重新登陆即可。
  </p>
  
  <h1>
    hello world
  </h1>
  
  <p>
    跑个ubuntu的image试试看：
  </p>
  
  <pre class="brush: shell; notranslate">docker pull ubuntu
docker run ubuntu /bin/echo hello world
</pre>
  
  <p>
    这将启动一个container加载ubuntu image并执行echo程序，打印hello world，然后退出。&#8221;docker ps&#8221;可查看container列表。
  </p>
  
  <h2>
    lxc-start: No cgroup mounted on the system
  </h2>
  
  <p>
    这个错误是因为cgroup没有mount，建议mount到/sys/fs/cgroup目录。首先编辑/etc/fstab文件，加入下面这行配置：
  </p>
  
  <pre class="brush: shell; notranslate">none        /sys/fs/cgroup        cgroup        defaults    0    0
</pre>
  
  <p>
    接下来创建目录，并mount：
  </p>
  
  <pre class="brush: shell; notranslate">sudo mkdir -p /sys/fs/cgroup
sudo mount /sys/fs/cgroup
</pre>
  
  <h1>
    创建一个运行Node.js应用的ubuntu image
  </h1>
  
  <p>
    官方给的例子是创建centos上运行node.js应用，具体看<a href="http://docs.docker.io/en/latest/examples/nodejs_web_app/">这里</a>。
  </p>
  
  <p>
    下面尝试用ubuntu创建下。
  </p>
  
  <h2>
    Node.js应用
  </h2>
  
  <p>
    创建一个src目录，存放node.js应用源码和配置
  </p>
  
  <pre class="brush: shell ; notranslate">mkdir src 
touch src/package.json 
touch src/index.js
</pre>
  
  <p>
    其中package.json内容:
  </p>
  
  <pre class="brush: javascript; notranslate"> {
  "name": "docker-ubuntu-hello",
  "private": true,
  "version": "0.0.1",
  "description": "Node.js Hello World app on ubuntu using docker",
  "author": "Daniel Gasienica &lt;daniel@gasienica.ch&gt;, Dennis&lt;killme2008@gmail.com&gt;",
  "dependencies": {
    "express": "3.2.4"
     }
 }
</pre>
  
  <p>
    index.js就是使用express框架渲染首页hello world:
  </p>
  
  <pre class="brush: javascript; notranslate">var express = require('express');    
// Constants
var PORT = 8080;   
// App
var app = express();
app.get('/', function (req, res) {
    res.send('Hello World\n');
});    
app.listen(PORT)
console.log('Running on http://localhost:' + PORT);
</pre>
  
  <h1>
    创建Dockerfile
  </h1>
  
  <p>
    Dockerfile文件配置image众多参数，例如parent image是什么，执行哪些安装命令，拷贝应用文件，导出TCP端口等等。与src目录统计创建文件名为Dockerfile，内容如下：
  </p>
  
  <pre class="brush: shell; notranslate">FROM ubuntu:12.10
RUN apt-get update
RUN apt-get install -y python-software-properties python g++ make software-properties-common
RUN add-apt-repository ppa:chris-lea/node.js
RUN apt-get update
RUN apt-get install -y nodejs
ADD ./src /src
RUN cd /src; npm install
EXPOSE  8080
CMD ["node", "/src/index.js"]
</pre>
  
  <p>
    我们采用官方提供的ubuntu 12.10的image做为os版本，接下来通过RUN指令安装node.js，然后用ADD指令将src目录拷贝到image的/src目录下，并在/src目录下执行npm install安装node.js依赖包，导出应用监听的8080端口到container外，最终通过CMD指令启动node.js应用。
  </p>
  
  <p>
    关于Dockerfile指令参考<a href="http://docs.docker.io/en/latest/use/builder/">这里</a>。
  </p>
  
  <h2>
    构建Image
  </h2>
  
  <p>
    执行下列指令：
  </p>
  
  <pre class="brush: shell; notranslate">docker build -t dennis/node-js .
</pre>
  
  <p>
    通过-t选项为这个image打上一个tag，在构建完成后，通过docker images列出所有的image的时候，可以看到你刚创建的image: dennis/node-js
  </p>
  
  <h2>
    运行container并测试
  </h2>
  
  <p>
    有了image，就可以从这个image启动一个container来运行node.js应用：
  </p>
  
  <pre class="brush: shell; notranslate">docker run -p 47516:8080 -d dennis/node-js
</pre>
  
  <p>
    我们将container内node.js应用监听的8080端口通过-p选项桥接到宿主机器(host machine)的47516端口，也就是说你可以通过47516端口能问到容器内的node.js应用。尝试下：
  </p>
  
  <pre class="brush: shell; notranslate">curl -X GET http://localhost:47516/
</pre>
  
  <p>
    输出hello world。
  </p>
  
  <p>
    本例子的源码已经放到<a href="https://github.com/killme2008/Docker-NodeJS-Ubuntu">Github</a>。
  </p>
</div>

<div id="post-footer-238" class="post-footer clear">
</div>