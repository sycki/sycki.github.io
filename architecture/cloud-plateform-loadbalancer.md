# 云平台 - 负载均衡器
负载均衡是云平台的重要组成部分，本文介绍开源项目[好雨云帮平台](https://github.com/goodrain/rainbond)(以下简称云帮)中负载均衡模块的具体实现，以及它出于什么样的考虑，希望能给有需要的同学带来一些参考和思路。

## 为什么需要负载均衡
首先，云帮平台的内部网络划分是支持多租户的，每个租户有一个私有的IP段，不同租户的网络是相互不可见的，当我们把一个容器化的应用部署到云帮后，云帮平台会为这个容器分配一个内部IP，用于同一租户中的不同应用在集群内部通信，而从集群外部是不能直接访问的。所以我们需要有一个集群入口控制器让用户能方便地访问这些应用。

其次，云帮中部署的每个应用都可以有多个实例，假设我们为一个WEB应用部署了三个实例，然后每个实例分担一部分流量，这时我每就需要在它们前面加一个负载均衡器来分发流量给三个实例。

除了上述的基本功能以外，我们的负载均衡器还必须支持更多的功能，比如：

* 入口控制器能够根据数据包信息（如协议、端口号、主机名等）将请求转发给指定的应用。
* 实时地发现集群中应用的变化（如添加自定义域名、添加证书、添加端口等）并动态更自身的转发规则。
* 要同时支持HTTP、TLS、TCP和UDP协议，因为有时不只WEB应用需要向外提供服条，像RPC、MySQL等也需要对外开放。
* 最后一点也很重要，那就是支持高可用。

综上所述，我们需要一个同时支持L4、L7的负载均衡器集群，还必须能够自动发现集群中的应用变化以更新自己的转发规则。

## 云帮中的负载均衡

### 整体架构
以下是云帮负载均衡模块的架构示意图：

![loadbalancer-architecture](cloud-plateform-loadbalancer/loadbalancer-architecture.png)

1. `web`：表示云帮中的一个应用，并且有三个实例。
2. `api-server`：表示kubeneters的kube-apiserver组件。
3. `entrance`：表示云帮的负载均衡器通用接口，支持多种负载均衡器插件。

### Entrance实现
云帮中的负载均衡是面向应用的，不同的应用可以使用不同的负载均衡，所以我们设计了Entrance组件，它可以集成多种负载均衡插件，OpenResty就是其中之一，这意味着云帮不仅支持OpenResty，还可以方支持其它负载均衡插件。

它的主要工作是从kube-apiserver中通过Restful API监听应用容器的IP、端口，service和endpoint等的资源变化，然后把这些资源抽象为通用的负载均衡资源并缓存在ETCD中，这些通用资源有：

* `Pool`：表示一个负载均衡池，其中包括多个节点，对应上图中的三个WEB实例。
* `Node`：表示Pool中的一个节点，对应上图中的其中一个WEB实例。
* `Domain`：表示一个域名，负载均衡器可以识别一个数据包中的域名信息然后将数据转发给对应的Pool。
* `VirtualService`：表示监听了某个端口的虚拟主机，还指明了端口的协议名称，主要用来处理L4入口控制和负载均衡。
* `Rule`：表示一条转发规则，用来描述域名跟Pool的对应关系，还指明了端口的协议名称与证书信息，主要用来处理L7入口控制和负载均衡。

当有资源发生变化时，Entrance会将通用资源转化为相应插件的资源并调用该插件的Restful API。

从上图中可以看到，有两个Entrance和两个OpenResty实例，它们的关系是：每个Entrance中持有所有OpenResty的地址，当有信息需要更新时，Entrance会将信息更新到所有的OpenResty。那两个Entrance之间怎么协调呢？这里我们利用ETCD本身的特性做了分布式锁，保证只有一个Entrance有权限向OpenResty更新信息，这样就实现了高可用。

### OpenResty件插
OpenResty是一个可以用Lua脚本来处理请求和业条逻辑的WEB应用，并且内置了众多Lua相关的指定和函数供开发者使用，很合适开发Restful API服务器，我们将OpenResty作为Entrance的插件之一原因如下：

1. 它是基于Nginx开发，所以稳定性和性能方面不用太担心。
2. 比较接近我们的目标，OpenResty已经帮我们把Lua模块编译进去，我们可以很方便地用Lua脚本丰富负载均衡器的功能，可以让我们省去一些工作量。
3. 同时支持L7和L4的负载均衡。

我们在OpenResty端嵌入了一个Rest API服务器，这些API是用Lua写的，前面说过OpenResty集成了Lua脚本功能，我们可以直接用Lua来处理请求，下面是Nginx配置文件的其中一部分：
```
# custom api of upstream and server
server {
    listen 10002;

    location ~ /v1/upstreams/([-_0-9a-zA-Z.@]+) {
        set $src_name $1;
        content_by_lua_file lua/upstream.lua;
    }

    location ~ /v1/servers/([-_0-9a-zA-Z.@]+) {
        set $src_name $1;
        content_by_lua_file lua/server.lua;
    }

}
```
当我们调用下面的API时：
```
curl -s localhost:10002/v1/servers/app1 -X POST -d "$json_data"
```
OpenResty会执行相应的Lua脚中，也就是`lua/server.lua`，前面说过，OpenResty内置了很多Lua相关的指命与函数，可以让Lua与Nginx更好地交互，所以我们在脚本中很容器处理接收到的JSON数据，并将其转换为配置Nginx文件，由于Lua代码较多就不贴出来了，可以在本文的引用部分找到该项目地址。

这里有个需要注意的地方，当收到大量修改server和upstream的请求时，OpenResty需要频繁加载配置文件，这样会增加负载且影响性能。实际上OpenResty有很多第三方插件可以使用，有一个叫dyups的插件可以做到动态修改upstream，它的使用方式如下，Lua代码：
```
-- 增加或更新指定upstream
dyups.update("upstream_name", [[server 127.0.0.1:8088;]])

-- 删除指定upstream
dyups.delete("upstream_name")
```
执行成功后就已经生效了，不需要我们执行`nginx -s reload`命令，这会提高一些效率。

对于server的修改暂时还没有相应用插件做到动态修改，所以实际上我们的负载均衡器分两种情况，如果更新了upstream配置会即时生效，而更新server配置则需要加上`nginx -s reload`命令。

## 结语
我们用Entrance加OpenResty实现了一个可插拔且高可用的负载均衡器，整体来说并不复杂，最后希望本文能带给你一些帮助。现在我们已经把我们的OpenResty插件分离出来做为一个子项目并且开源在Github上，你可以下载并单独使用：[Github地址](https://github.com/goodrain/lb-openresty)。

## 引用与参考
1. [Openresty项目](https://github.com/openresty/openresty)
1. [dyups插件](https://github.com/yzprofile/ngx_http_dyups_module)
1. [好雨云帮](https://github.com/goodrain/rainbond)

关于
---

__作者__：sycki

__阅读__：97

__点赞__：0

__创建__：2018-04-30
