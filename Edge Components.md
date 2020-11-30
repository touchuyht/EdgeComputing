# Edge Components

## EdgeD

### EdgeD概述

EdgeD是一个边缘节点module，它负责管理pod的生命周期。它帮助用户将容器化的workloads载或者是应用部署到边缘节点上。这些workloads可以执行的操作包括简单的自动记录数据的管理到数据分析或者是机器学习等等。通过在云端使用kubectl命令行接口，用户可以让这些命令启动workloads。

通过CRI容器运行时接口，EdgeD可以支持集中OCI标准的运行时。

有许多模块协同工作以完成EdgeD的职能。

![](图片/edged-overall.png)

### Pod管理

EdgeD负责的pod增加、删除和修改。同时它也会使用pod status manager跟踪pod的健康状态。它的主要作用有如下：

​	接收和处理来自metamanager的pod 增加/删除/修改消息（addition/deletion/modification）

​	为pod的增加和删除使用单独的work queues

​	处理worker的日程以检查work queues对pod操作的完成状况

​	为configmap和secrets使用分开的缓存

​	定期删除orphaned pods（节点OOM以后或者节点异常崩溃的情况下，pod未能被正常的清理而导致的孤儿进程）

![](图片/pod-addition-flow.png)

Fig2：Pod Addition Flow

![](图片/pod-deletion-flow.png)

Fig 3：Pod Deletion Flow

![](图片/pod-update-flow.png)

Fig 4：Pod Updation Flow

所以pod的更新是先删除原有pod再根据新的配置信息创建一个pod吗？

### Pod生命周期事件生成器(Pod Lifecycle Event Generator)

这个module为边缘侧帮助监控pod的状态。通过使用存活探针和就绪探针，它更新为每个pod更新其在pod status manager中的信息。

![](图片/pleg-flow.png)

Fig 5：PLEG at EdgeD

### EdgeD的CRI接口

CRI容器运行时接口——是一个插件接口，通过它能够支持许多的容器运行时，比如Docker，containerd，CRI-O等等，而不需要重新编译EdgeD。

为什么EdgeD要使用CRI？

EdgeD需要通过CRI支持多种容器运行时，以实现：

​	在资源受限的无法运行Docker运行的节点上运行更加轻量化的容器运行时

​	在边缘侧支持多种容器运行时，比如Docker、containered、CRI-O等等

之后将会考虑支持对应的CNI接口和pause容器和IP

![](图片/edged-cri.png)

### Secret管理

在EdgeD中，Secrets被单独处理。对于诸如添加、删除和修改的操作，使用了不同的配置消息和接口。通过使用这些接口，secrets被存储在缓存中。

![](图片/secret-handling.png)

Fig 7：Secret Message Handling at EdgeD

EdgeD使用MetaClient module来充MetaManager中获取secrets。如果EdgeD请求一个新的还未被保存在MetaManager中的secret，请求将会被转发到云端。在发送包含有secret的相应之前，MeraManager将其存储在本地的数据库中。之后的请求都会从本地数据库中获取数据，以减少延迟。下面的流程图展示了一个secret是如何从MetaManager和云端获取到的。它同时也描述了secret是如果存储在MetaManager中的。

![](图片/query-secret-from-edged.png)

Fig 8：Query Secret by EdgeD

### Probe管理

probe management为每个pod创建就绪探针和存储探针。就绪探针帮助监控何时pod进入了运行状态。存活探针帮助监控容器的健康状况，以此判断pod是正常运行还是死掉了。如前所述，PLEG（Pod声明周期事件生成器）使用probe management提供的服务。

### ConfigMap Management

在EdgeD部分，ConfigMaps也被单独处理。对于诸如添加、删除和修改的操作，有不同的配置消息和接口。通过使用这些接口，ConfigMaps的更新被存储在本地存储中。

![](图片/configmap-handling.png)

Fig 9：ConfigMap Message Handling at EdgeD

EdgeD使用MetaClient module来从MetaManager中获取ConfigMaps。如果EdgeD请求的新ConfigMap还未被存储在MetaManager中，这个请求被转发至云端。在发送包含ConfigMap的相应之前，MetaManager将其存储在本地数据库中。之后对于同样的ConfigMap查询将会从本地数据库中取数据，以减少延迟。下面的流程图展示了从MetaManager和云端获取ConfigMap的过程。同时它也说明了ConfigMaps是如何存储在MetaManager中的。

![](图片/query-configmap-from-edged.png)

Fig 10：Query ConfigMaps by EdgeD

### Container GC

容器垃圾回收由edged每分钟执行一次，它通过使用指定的容器垃圾回收策略清除死掉的容器。容器垃圾回收的策略由是三个变量指定，这三个变量可以由用户指定：

​	**MinAge**：指定的是一个容器可以被垃圾回收的最小时间，如果设置为0则没有限制

​	**MaxPerPodContainer**：是一个容器对(UID, container name)中允许的最多死亡pod数，小于0则没有限制

​	**MaxContainers**：是总的最多死亡容器数，小于0则没有限制。大体而言，最先死亡的pod将会第一个被回收

### Image GC

容器镜像回收由EdgeD每5秒钟执行一次，基于容器镜像的回收策略收集磁盘信息。容器镜像回收策略需要考虑两个因素，`HighThresholdPercent`和`LowThresholdPercent`。当磁盘的使用亮超过`HighThresholdPercent`之后将会启动镜像垃圾回收，它会尝试删除未被使用的镜像知道磁盘使用量低于`HighThresholdPercent`。最近最少未被使用的镜像将会被第一个删除。

### Status Manager

Status Manager是一个单独的EdgeD例程，每10秒钟收集一次Pod的状态，并且会通过metaclient接口将其同步到云端。

![](图片/pod-status-manger-flow.png)

Fig 11：Status Manager Flow

### Volume Management

Volume manager作为边缘侧的例程运行，基于每个node上调度的pod收集那些将被attached/mounted/unmounted/detached的volume信息。

在启动pod之前，所有在pod specs字段中被指定的volumes会被attached和mounted，在挂在存储卷期间pod的创建流程会被阻塞。

### MetaClient

MetaClient是一个MetaManager的接口。它从metamanager和云端帮助EdgeD获取ConfigMaps和secret的详细信息。它也会发送sync类型的消息，node状态和pod状态到MetaManager再到云端。

## EventBus

### EventBus概述

Eventbus是一个发送/接受mqtt各个topics消息的接口。

它支持如下的三种模式：

​	**internalMqttMode**

​	**externalMqttMode**

​	**bothMqttMode**

### Topic

eventbus订阅(subscribe)了如下的topics：

```latex
- $hw/events/upload/#                    ‘这一项是说最后一次的更新事件’
- SYS/dis/upload_records				 ‘这一项所有的更新记录’
- SYS/dis/upload_records/+               ‘所有更新记录中的子记录’
- $hw/event/node/+/membership/get        ‘所有节点的事件’
- $hw/event/node/+/membership/get/+      ‘所有节点的事件的子记录’
- $hw/events/device/+/state/update       ‘所有设备状态的更新’
- $hw/events/device/+/state/update/+     ‘所有设备状态的更新子记录’
- $hw/event/device/+/twin/+              ‘所有设备的所有本地存储’
```

topic通配符(wildcards)的意义：

| widlcard | Description                                                  |
| -------- | ------------------------------------------------------------ |
| #        | 必须是topic的最后一个字符，并且要匹配当前的tree和所有的subtree |
| +        | 匹配topictree的恰好一个item                                  |

### 流程图

**1、eventbus转发external client的消息**

![](图片/eventbus-handleMsgFromClient.jpg)

**2、eventbus通过发送响应至external client**

![](图片/eventbus-handleResMsgToClient.jpg)

对于internalmode的流程是一致的，只是internal mode中eventbus是message broker自己。

## MetaManager

### MetaManager概述

MetaManage用于处理EdgeD和EdgeHub之间的消息。它同时也有将metadata存储到轻量化的数据库或者从数据库中取出metadata的作用。

Metamanager基于如下的操作列表接收不同类型的消息：

​	Insert

​	Update

​	Delete

​	Query

​	Response

​	NodeConnection

​	MetaSync

### Insert Operation

当有新的对象被创建时Insert操作消息就通过云端被接受到。比如用户创建了一一个新的应用容器。

![](图片/meta-insert.png)

插入操作的请求通过edgehub接收。它将请求分发给metamanager，由matamanager将其存储到本地数据库。metamanager然后以异步的方式发送消息到EdgeD。EdgeD处理具体的插入请求，比如开始一个容器，之后将响应写入消息。MetaManager审查这条消息，提取出response并将其发给EdgeHub，再由其发送给云端。

### Update Operation

更新操作可以由在云上的对象引起，也可以由边上的对象引起。

更新操作消息的流向和插入操作类似。此外，MetaManager还会检查需要更新的资源是否在本地发生了该百年。只有当本地发生了变化时，这个更新操作才会被本地存储并且消息被发送到EdgeD，响应被发送到云端。

![](图片/meta-update.png)

### Delete Operation

当Pod被从云端删除时，会触发Delete操作。

![](图片/meta-delete.png)

### Query Operation

查询操作可以查询本地的元数据或者是远程的数据，比如云端的configmaps/secrets。EdgeD从MetaManager查询这些metadata，它处理本地或者远程的查询请求并且将response返回边缘侧。一个消息资源可以基于'/'分隔符被划分成三部分(resKey,resType,resId)。

![](图片/meta-query.png)

### Response Operation

在云/边的任何操作都会有response返回。

### NodeConnection Operation

NodeConnection消息通过edgeHub被接收，其中包含云端连接状态的信息。MetaManager跟踪其状态（保存于内存中），并且会在比如有向云端的远程访问请求时使用特定的动作。

### MetaSync Operation

MetaSync操作消息会周期性的由MetaManager发送已同步在边缘节点运行的pod的状态。同步的间隔可以在conf/edge.yaml中进行（默认间隔为60s）。

```yaml
meta:
    sync:
        podstatus:
            interval: 60 #seconds
```

## EdgeHub

### EdgeHub概述

EdgeHub负责和CloudHub的组件进行通信。它可以连接到CloudHub，使用web-socket连接或者QUIC协议。它支持的功能包括同步云端资源的更新、报告边缘侧的主机和设备状态的变化。

它扮演了云边通信的角色。它负责将云端的消息发送给边缘侧的对应module，反之亦然。

EdgeHub的主要功能有如下：

​	保持连接

​	发布客户端的信息

​	路由到云

​	路由到边

### 保持连接

在每个心跳周期，一套保持连接的消息或者心跳消息被发送到CloudHub。

### 发布客户端的信息

发布客户端信息的主要职责是通知其他的groups或者modules有关和云端连接的有关信息。

它会通过发送beehive的消息到所有的groups(metaGroup,twinGroup和busGroup)通知他们和云端的连接是否畅通

### 路由到云

路由到云的主要功能是接收来自其他module的发送到云端(通过beehive框架)的消息，并且将它们通过websocket连接发送到云端。

主要的步骤如下：

​	1、持续从beehive Context接收消息；

​	2、发送消息到CloudHub；

​	3、如果接收到的是一条同步消息那么：

​		3.1 如果response实在syncChannel上被接收到的，那么他会创建一个map[string]chan message的字典，其中message的messageID是键；

​		3.2 每一个周期等待在创建的channel上接收心跳消息，如果在一个周期内没有接受到任何response，则超时

​		3.3 通过使用SendResponse函数，response被从channel发回给module。

![](图片/route-to-cloud.png)

### 路由到边

路由到边的主要职责是接收来自云端的消息（通过websocket连接）并且将它们发送给指定的groups（通过beehive框架）。

在这一过程中的主要步骤：

​	接收来自CloudHub的消息

​	检测接收消息的group是否被找到了

​	检查是否是对SendSync()函数的response

​	如果不是一条response消息则其被发送给指定的group

​	如果是一条response消息则被发送给syncKeep channel

![](图片/route-to-edge.png)

### 用法

EdgeHub可以被配置为通过如下协议和云端通信：

​	通过websocket协议

​	通过QUIC协议

## DeviceTwin

### DeviceTwin概述

DeviceTwin module负责存储设备的状态、处理设备的属性、处理device twin的操作、创建边设备和边节点的membership关系、同步设备的状态到云端并且在云端之间同步device twin的信息。它也提供对应用的查询接口。Device Twin由四个子模块组成(即membership module, communication module, device module和device twin module)来执行device module的职责。

### Device Twin Controller的职能

以下是Device Twin Controller的职能：

​	同步数据自（或到）数据库（sqlite）

​	注册和启动子模块

​	分发消息到子模块

​	健康检查

#### 同步数据自（或到）数据库（sqlite）

对于所有由边节点管理的设备，device twin会执行如下的操作：

​	检查的device是否在device twin的context中（存储在device twin context中的设备列表），如果没有则添加一个互斥体到context中

​	从数据库查询device

​	从数据库查询device属性

​	从数据库查询device twin

​	将device，device属性和device twin数据组合成一个结构体并且将其存储在device twin context中

#### 注册和启动子模块

注册四个device twin的子模块并且将其作为单独的goroutine运行

#### 分发消息到子模块

·1、通过beehive框架持续监听device twin的消息

2、发送接收到的消息到device twin的通信模块

3、根据消息源对消息进行分类，比如消息是否来自eventBus、edgeManager或者EdgeHub，并且填充模块的action module map（ActionModuleMap是一个module的action map）

4、发送消息给指定的device twin module

#### 健康检查

Device Twin Contoller定期（每60s）发送ping消息到每个子模块。每个子模块每接收到一个ping消息就更新自己字典中的时间戳。contoller会检查每个module的时间戳是否超过两分钟，否则就重启这个子模块。

### Module

DeviceTwin由如下的四个模块组成：

​	Membership Module

​	Twin Module

​	Communication Module

​	Device Module

Membership Module

membership module的主要职责是提供新的membership到通过云端添加到边缘的新设备。这个module半丁新添加的设备到边缘节点，并且创建边缘节点和边缘设备的membership。

由这个模块执行的主要功能有：

​	1、初始化action callback map，它是一个map[string]Callback的字典，其中包含可以被调用的回调函数

​	2、接收发送到membership module的消息

​	3、对于每一条读取的action message，都有一个对应的函数会被调用

​	4、通过heartbeat channel接收心跳并且将心跳发送给controller

membership module可以执行如下的action callbacks：

​	dealMembershipGet

​	dealMembershipUpdated

​	dealMembershipDetail

**dealMembershipGet**：dealMembershipGet()从cache返回和特定边缘节点关联的设备的信息

​	eventbus首先从订阅的topic上接收到消息（membership-get topic）

​	这个消息到达device twin controller，这个模块进一步将消息发送给membership module

​	membershipmodule从cache(context)得到和边缘节点绑定的设备并且将消息发送给通信模块。它同时也会处理在前述过程中产生的错误并且将错误发送给通信模块，这时设备细节将不会被发送。

​	通信模块将消息发送给eventbus组件，由它进一步将消息发送给指定的MQTT topic(getmemmbership result topic)

![](图片/membership-get.png)

dealMembershipUpdated：dealMembershipUpdated()更新节点的membership细节。它添加刚刚添加的设备到边缘侧的group并且从边缘group删除刚刚删除的，如果设备发生了更改或者更新，则更新设备的细节。

​	EdgeHub module接收来自云端的membership update message，并且将其进一步发送给device twin controller，再由后者进一步发送给membership module

​	membership module添加刚刚被添加的设备，删除刚刚被删除的设备，摈弃给更新存在数据库和本地缓存中的设备

​	当更新一个设备的细节之后，一条message被发送给device twin的通信模块，再由其发送给eventbus以将消息发送到指定的MQTT topic上

![](图片/membership-update.png)

dealMembershipDetail：dealMembershipDetail()提供边缘节点的membership细节，当删除最近移除的边缘设备的membership之后提供和边缘节点有关的设备细节。

​	eventbus module接收到订阅主题上的消息，消息被进一步发送给device twin controller，再由其进一步发送给membership module

​	membership module添加消息中提到的设备，移除不在本地缓存中的设备

​	当更新了设备细节之后，一条消息被发送给devivce twin的通信模块

![](图片/membership-detail.png)

### Twin Module

twin module的主要职责是处理所有和device twin有关的操作。它可以执行的操作包括device twin更新、取得device twin和同步device twin到云端。

这个模块的主要职责有如下：

1、初始化action callback map（是一个action(string)的字典），callback函数是其值，它可以执行请求

2、接收发送到device twin module的消息

3、对于每一条读取的action message，对应的函数会被调用

4、接受来自heartbeat通道的heartbeat并且将其发送给controller。

twin module可以执行如下的action callbacks：

​	dealTwinUpdate

​	dealTwinGet

​	dealTwinSync

**dealTwinUpdate**：dealTwinUpdate()更新device twin中指定设备的信息

​	更新device twin的消息可以由EdgeHub module接收来自云端的消息或者通过MQTT代理通过eventbus组件接收（mapper会将消息发送到device twin update topic上）

​	然后消息被发送给device twin controller，之后被发送给device twin module

​	device module更新存储在数据库中的twin的值，并将含有更新结果的消息发送给通信模块

​	通信模块将消息通过eventbus发送给MQTT代理

![](图片/devicetwin-update.png)

**dealTwinGet**：dealTwinGet()提供了有关特定设备的device twin消息

​	eventbus组件接收到达订阅的twin get topic熵的消息，并将其发送给device twin controller，后者将其进一步发送给twin module

​	twin module得到和某i一设备有关的device twin信息，并且将其发送给通信模块，它同时也会处理诸如设备找不到或者其他内部问题的错误

​	通信模块将信息发送到eventbus组件，由后者将其发布到特定的主题上

![](图片/devicetwin-get.png)

**dealTwinSync**：dealTwinSync()同步device twin的信息到云端

​	eventbus模块接收订阅的twin cloud sync topic的消息

​	消息被发送给device twin controller，再由其发送给twin module

​	twin module将同步存储再数据库中的twin信息，并将twin同步的结果发送给通信模块

​	通信模块将其进一步发送给EdgeHub组件，由后者通过websocket连接将更新发送到云端

​	这个函数同时也会执行诸如发布更新后的twin细节文档、改变devie twin和通过通信模块发布更新结果，由通信模块将数据发送到EdgeHub，再由其发送到eventbus由其发布到MQTT代理上。

![](图片/sync-to-cloud.png)

### Communication Module

通信模块的主要职责是保证device twin和其他组件的通信。

这个模块的主要功能有如下：

​	1、初始化action callback map，它是一个map[string]callback的字典，值为执行操作的callback函数

​	2、接收发送给通信模块的消息

​	3、对于每一条读取的消息执行对应的函数

​	4、确认消息指定的操作是否完成，当操作未完成时需要重做

​	5、接收heartbeat channel上的heartbeat，并且发送heartbeat到controller

下面是通信模块可以执行的一些action callback：

​	dealSendToCloud

​	dealSendToEdge

​	dealLifeCycle

​	dealConfirm

**dealSendToCloud**：dealSendToCloud()被用来发送数据到CloudHub组件。这个函数首先确保已经和云端建立连接，然后将消息发送给EdgeHub module（通过beehive框架），由后者进一步消息发送给云端（通过websocket连接）

**dealSendToEdge**：dealSendToEdge()被用来向边缘中的其他组件发送消息。这个函数发送接收到的消息到EdgeHub模块（通过beehive框架）。EdgeHub module在接收到消息之后将其发送给指定的接收者。

**dealLifeCycle**：dealLifeCycle()检查和云端的连接是否建立和device twin是否处于连接断开的状态，然后它将状态改变为连接并且将节点的信息发送给EdgeHub。如果CloudHub处于失联状态，它将device twin的状态设置为disconnected。

**dealConfirm**：dealConfirm()被用来确认事件。它将会检查消息的类型是否正确然后将其从确认字典中删除其id。

### Device Module

Device Module的主要职责是执行设备有关的操作，比如更新设备的状态、更新设备的属性。

这个模块执行的主要功能有：

1、初始化action callback map（是一个action(string)的字典），其中callback函数是值，这个函数用于执行请求；

2、接收发送到device module的消息；

3、对于每一条读取的消息执行相应的函数；

4、在heartbeat channel上接收heartbeat消息，并发送心跳到controller

下面是device module执行的action callbacks：

​	dealDeviceUpdated

​	dealDeviceStateUpdate

**dealDeviceUpdated**：dealDeviceUpdated()处理设备属性需要更新情况下的有关操作。它将存储在数据库中的设备属性进行更新，比如增加设备属性、更新属性值、或者删除属性。同时它也会将设备属性更新的结果发布到eventbus组件。

​	device属性更新由云端发起，通过EdgeHub被发送到边缘侧

​	EdgeHub组件将消息发送给device twin controller，再由其进一步发送给device module

​	当device module发送设备属性更新的结果到eventbus（通过device twin的通信组件），device module更新数据库中的device属性。eventbus之后将其发布到特定的主题上。

![](图片/device-update.png)

**dealDeviceStateUpdate**：dealDeviceStateUpdate()处理设备状态需要更新情况下的操作。它更新设备的状态和数据库中设备最后一次上线的状态。它同时也会通过通信模块发送更新状态的结果到云端（通过）和eventbus module，后者会将其更新结果发布到MQTT代理的指定主题上。

​	device状态更新由发布在特定主题上的消息引起，这个主题由eventbus组件订阅

​	devicebus组件发送消息到device twin controller，由其进一步将消息发送到device module

​	device module更新设备的状态和数据库中最后一次设备上线的状态

​	device module然后通过device twin的通信模块发送device状态更新的结果到eventbus组件和EdgeHub组件。eventbus组件发布更新的结果到特定的topic上，而EdgeHub同时设备更新的状态发送到云端。

![](图片/device-state-update.png)

### Tables

DeviceTwin module会再数据库中创建三张表，如下：

​	Device Table

​	Device Attribute Table

​	Device Twin Table

#### Device Table

device表包含有被添加到指定节点的设备数据。其字段如下:

| Column Name | Description          |
| ----------- | -------------------- |
| ID          | 设备ID               |
| Name        | 设备名字             |
| Description | 设备描述             |
| State       | 设备状态             |
| LastOnline  | 设备最后一次在线时间 |

可以执行的操作：

下面是一些可以在这些数据上执行的操作

​	**Save Device**：插入一个设备到设备表中

​	**Delete Device By ID**：通过ID从设备表中删除一个设备

​	**Update Device Field**：更新设备表中的一个字段

​	**Update Device Fields**：更新设备表中的多个字段

​	**Query Device**：从设备表中查询一个设备

​	**Query Device All**：查询设备表中所有的设备

​	**Update Device Multi**：更新设备表中的多个设备的多个列

​	**Add Device Trans**：在一个事务中插入设备、设备属性和device twin。如果任一操作失败，整个操作都会回滚

​	**Delete Device Trans**：在一个事务中删除设备、设备属性和device twin。如果任一操作失败，整个操作都会回滚。

 