# **Cloud Components**

## Edge Controller

### Edge Controller概述

Edge Controller时Kubernetes apiserver和edgecore之间的桥梁

### Edge Controller的功能

以下是Edge Controller的功能：

1、Downstream Controller：从Kubernetes apiserver同步add/update/delete事件到edgecore；

2、Upsteam Controller：同步监控(watch)和更新资源(resource)和事件(events)（node,pod和configmap）到k8s apiserver，同时订阅来自edgecore的消息

3、Controller Manager：创建manager接口，这个接口实现了管理ConfigmapManager、LocationCache和podManager的事件

### Downstream Controller

同步add/update/delete事件到edge

1）Downstream controller： 监控(watch)k8s apiserver并且将更新通过cloudHub发送给edgecore；

2）通过cloudHub同步（pod，configmap，secret）add/update/delete事件到edge；

3）创建Respective manager（pod，configmap， secret）通过调用manager接口处理事件；

4）指出configmap和secret应该被送到那个节点

![](图片\DownstreamController.png)

### Upstream Controller

同步监控(watch)和更新资源(resources)和事件(events)的状态

1）Upsteam Controller接收来自edgecore的消息并且将更新同步到kubernetes apiserver；

2）创建停止channel用于分发和停止事件以用于处理pods、configmaps、node和secrets；

3）创建消息channel用来更新Nodestatus、Podstatus、Secret和configmap有关的事件；

4）得到Pod状态信息，比如Ready、Initialized、PodScheduled和Unschedulable的细节；

5）下面是Pod状态有关的信息：

​	**Ready**：PodReady表示pod可以用于服务请求，并且应当按匹配的服务添加到负载均衡池

​	**PodScheduled**：代表此pod的调度过程

​	**Unschedulable**：表示scheduler不能够立即调度pod，可能是由于集群中的资源(resource)不足

​	**Initialized**：容器中的所有初始化容器都已经成功启动

​	**ContainerReady**：表示pod中的所有容器都已经就绪(ready)

6）下面是PodStatus的信息：

​	**PodPhase**：pod的当前状态

​	**Conditions**：pod处于当前状态的原因

​	**HostIP**：pod被分配到节点的ip

​	**PodIP**：pod的ip

​	**QosClass**：基于资源要求分配给pod

![](图片/UpstreamController.png)

### Controller Manager

创建manager接口并且实现ConfigmapManager、LocationCache和podManager

1）Manager定义了一个manager的接口，ConfigManager、Podmanager和Secretmanager实现该接口

2）管理OnAdd、OnUpdate、OnDelete事件，这些事件会从kubernetes apiserver同步到相应的edge node上

3）创建eventManager（pod，configmap，secret），这个eventManager为每个事件会启动一个CommonResourceEventHandler、NewListWatch和newShare Informer以同步(add/update/delete)事件(pod/configmap/secret)到edgecore，当然也是通过cloudHub；

4）下面是由Controller Manager创建的handlers的列表：

​	**CommonResourceEventHandler**：NewcommonResourceEventHandler为Configmap  manger和pod manager创建CommonResourceEventHandler；

​	**NewListWatch**：根据给定的资源namespace的client和字段选择器(field selector)创建一个新的ListWatch

​	**NewSharedInformer**：创建ListWatch的新实例

## CloudHub

### CloudHub概述

CloudHub是cloudcore的一个module，作为Controllers和边缘端的桥梁。支持websocket的链接，也支持QUIC协议。EdgeHub可以选择其中一种协议和cloudHub通信。CloudHub的功能是连接edge和Controllers。

连接到边缘(通过EdgeHub)通过建立在websocket上的HTTP协议。对于内部通信，它直接和Controllers通信。所有被发往CloudHub的请求都是context对象，它们被存储在channelQ中。channelQ中也存放着标有nodeID的事件对象的channels。

CloudHub的主要功能有：

1）得到消息context，创建事件对象的ChannelQ；

2）建立基于websocket的http连接

3）维持websocket连接

4）读取来自边缘侧的消息

5）向边缘侧发送消息

6）将消息发送给Controller

### 得到消息context，创建事件对象的ChannelQ

context对象被存储在channelQ中。为每一个nodeID创建channel，并且消息都被转变为事件对象之后通过channel发送

### 建立基于websocket的http连接

1）通过context对象提供的路径加载TLS证书；

2）启动HTTPS server

3）通过接受conn对象，https连接升级为websocket连接

4）ServeConn函数用于处理所有到来的连接

### 读取来自边缘侧的消息

1）首先设置一个存活期的长短（即超过这个时间没用发送心跳，连接就会断掉）；

2）从连接中读取JSON对象；

3）之后Message Router的各个属性得到设置

4）消息被转变为事件对象以用于云端内部的通信

5）最后事件被转发给Controllers

### 向边缘侧发送消息

1）为已知的nodeID接受所有事件对象；

2）检查该节点是否已经接受过该请求和该节点是否存活；

3）事件对象被转变为消息结构体

4）写的超时间隔被设置好。消息通过websocket被发送到边缘侧

### 将消息发送给Controller

1）每当有来自websocket连接的请求，都会有一个默认的有timestamp，clientID和事件类型消息被发送到controller

2）如果node下线了，那么抛出异常，并且节点失败事件会被发送到controllers

### 用法

CloudHub可以通过如下的三种方法进行配置：

1）只启动websocket server

2）只启动quic server

3）同时启动websocket server和quic server

## Device Controller

### Device Controller概述

Device Controller是KubeEdge的云端组件，它用于进行设备管理。KubeEdge中的设备管理是通过使用Kubernetes的CRD描述设备的metadata/status和device controller以用于在云端之间同步这些设备的更新事件实现的。Device Controller会开启两个单独的goroutine，分别叫做upstream controller和downstream controller。这两个并不是单独的controller，在这里一提只是用作说明。

Device Controller使用device model和device instance来实现设备管理：

​	**Device Model**：device model描述device暴露的属性(properties)和用于访问这些属性的属性访问器(property visitor)。device model就像是一个可以复用的模板，通过这个模板可以创建管理许多设备(device)。

​	**Device Instance**：一个设备实例代表了实际的设备。它就像是device model的实例，并且其参考属性定义在model中。device的spec部分是静态的，device status部分则包含动态变化的数据比如某个属性期望的状态和设备当前的状态。

注意：部分协议的device model和device instance样例可以再$GOPATH/src/github.com/kubeedge/kubeedge/build/crd-samples/devices路径下找到。

![](图片/device-crd-model.png)

### Device Controller的功能

以下是device controller的功能：

​	**DownStream Controller**：通过监听k8s apiserver将设备更新从云同步到边

​	**Upstream Controller**：通过使用device twin组件，将设备的更新从边同步到云

### Upstream Controller

Upsream Controller监听来自边缘节点更新信息，并且讲这些更新通过apiserver在云端进行同步。更新操作的类型根据Upstream Controller可能采取的动作进行分类：

| Update Type             | Action                                             |
| ----------------------- | -------------------------------------------------- |
| Device Twin报告状态更新 | Upsteam Controller根据报告的更新状态在云端进行同步 |

![](图片/device-upstream-controller.png)

#### 同步Device Twin中更新的值到云端

mapper监听了device的更新状态，并且通过MQTT的消息队列发送到event bus上。event bus将device的当前状态发送给device twin，device twin在本地保存一份副本并将其同步到云端。device controller监听来自边缘端的更新操作（通过cloudHub）并且在云端更新设备的状态。

![](图片/device-updates-edge-cloud.png)

### Downstream Controller

Downstream Controller检测k8s apiserver中设备的更新。根据downstream controller可能采取的动作，将更新如下分类：

| Update Type                       | Action                                                       |
| --------------------------------- | ------------------------------------------------------------ |
| New Device Model Created          | NA                                                           |
| New Device Created                | 控制器创建一个新的configmap来存储设备的属性和在device model中定义的和device有关的visitor。configmap被保存在etcd中。在edge controller中存在的configmap同步机制将会将configmap同步到边。运行在容器里的mapper应用将会得到更新后的configmap，并使用得到的属性值和visitor metadata去访问设备。device controller此外还会报告device twin metadata的更新到边缘侧 |
| Device Node Membership Updated    | device controller发送一个membership更新事件到边缘侧          |
| Device Twin Desired State Updated | device controller发送一个twin更新事件到边缘侧                |
| Device Deleted                    | downstream controller发送device twin删除事件到所有的和device有关的device twin。同时它也会删除和device有关的configmap，并且会将删除事件同步到边缘侧。删除事件完成之后mapper应用会立即停止控制device。 |

![](图片/device-downstream-controller.png)

之所以要是用configmap来存储设备的属性和visitor是因为这些metadata只被边缘侧的mapper应用用来连接设备和收集数据。mappers以容器的形式运行就可以获取这些数据。任何在云端对设备属性和visitor的增加、删除更新操作都会被downstream controller监听并且会更新在etcd中的与之有关的configmap。如果一个mapper想知道devices有哪些属性，它可以从设备实例中获得这些model信息。此外，它也可以从device实例中知道连接协议的有关信息。一旦它可以访问device model，它就能得到device支持的属性。为了可以访问这些属性，mapper需要知道对应的visitor的信息。这可以通过propertyVisitor列表获得。最后通过使用visitorConfig，mapper可以读取/写入设备属性。

#### 同步更新的期望Device Twin属性到边缘端

![](图片/device-updates-cloud-edge.png)

Device Controller监听云端中的设备更新并将其转发到节点上。这些更新操作都会被device twin保存。mapper通过MQTT代理得到更新信息并且在基于更新后的信息操作设备。