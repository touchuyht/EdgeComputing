# Kubeedge有关论文阅读

某个领域读上100篇就能成为专家。

## Edge computing platforms for Internet of Things

作者对于Kubedge的评价：Aside from the basic infrastructure for cloud-to-edge communication and managing containers, KubeEdge does not provide other tools required for building IoT solutions, such as device management, data storage, device and sensor communications, edge intelligence, or security tools. Therefore KubeEdge is not a complete IoT platform, but one that can be incorporated into a selected architecture in addition to other elements, such as a public cloud platform.

### IoT中使用到的网络协议：

**IP协议栈**

**WiFi 802.11**

**蜂窝网络3G,4G,5G**

然而上述三者的功耗相对较高，大部分电池供电的设备可能无法运行这些协议。使用低功耗的本地通信协议接入到一个性能功耗更大设备（智能网关）是一种选择。

**TCP UDP HTTP** 

**HTTPS**:  adds a security layer under the HTTP transport, using the TLS protocol to encrypt and authenticate communications.

**QUIC**: QUIC取代了HTTPS的三层冗余结构（TCP, HTTP, TLS），提供更低的连接延迟、与IP无关的连接、单连接多种流数据、更低的往返事件以提升在较差连接质量的网络中的性能。

**Message Queue Telemetry Transport (MQTT)** 1 is a lightweight messaging protocol based on the publish/subscribe messaging pattern. In publish-subscribe messaging, senders send messages by publishing them on a topic, while senders receive them by subscribing to the topic; this allows for communication without having the senders or receivers know about the existence of the others and leads to greater scalability and flexibility than delivering messages using specific destination addresses. A MQTT messaging system is organized around a message broker that acts as a server to which clients connect to publish messages or subscribe to topics. The topics are arbitrary in the sense that they are not preconfigured on the server. MQTT runs over the TCP/IP protocol stack by default but may also be implemented over other protocols that provide reliable bidirectional connections, such as HTTP.

### 从云计算到边缘计算

Cloud computing refers to the idea of providing computing as a service over a network from massive centralized cloud data centers: utility computing. This model has become successful due to its inherent efficiency, but it is being challenged by IoT applications with **special latency, bandwidth, reliability, power consumption, and privacy** requirements.

 A public cloud is a service where cloud computing is provided to the general public; in contrast, a private cloud is a cloud-scale operation where the service is internal to an organization. The service may be provided on multiple different levels: in Infrastructure as a Service (IaaS), as basic compute, storage, and network capacity; in Platform as a Service (PaaS), as a platform on which applications may be developed and deployed on; and in Software as a Service, as software applications.

 Providing virtual machines as a service is an example of IaaS called Virtual Private Server (VPS).

Cloud computing produces highly scalable computing resources efficiently and therefore is in a good position to provide the capacity required by IoT , but it has inherent limitations.While the amount of processing capacity and generated data has increased, the growth is outpacing the increase in network capacity required to transfer the data to the cloud.

**edge**: refers to any location outside a centralized data center, closer to where data is sourced or consumed.

**Edge computing has several potential benefits:**

 • network delay is reduced, improving response times,

 • reduced network bandwidth requirements and usage,

 • improved availability through less reliance on a cloud connection,

 • improved control over data for security and privacy, 

• reduced energy consumption through offloading computation from devices and less network communication, and

 • reduced cloud processing costs.

**雾计算是个什么东西？**

Fog computing is a concept closely related to edge computing . In fog computing, the cloud is extended to the edge of the network by deploying the same virtualization tools used in the cloud on infrastructure placed outside datacenters.

**Another edge computing paradigm is Mobile Edge Computing (MEC)**, where edge applications are run on mobile base stations with integrated computing capacity, turning mobile network operators into cloud service providers [60]. Mobile base stations are widespread and located close to users, which makes them promising for providing edge capacity.

**Network Function Virtualization (NFV )** decouples services provided by network infrastructure from hardware by using virtualization [61]. It brings the elasticity of the cloud to network infrastructure: for example, services that have previously been provided by multiple network appliances can be consolidated into a single piece of hardware using 17 NFV, reducing capital costs. Additionally, it enables flexible deployment of novel services, such as services that bring video streaming services closer to users for improved quality. Having infrastructure capable of running virtualized workloads creates an opportunity to implement MEC using the infrastructure [60].

**边缘计算的一些应用：**

CDN、自动驾驶软件、视频分析、AR增强现实。

边缘计算在物联网中应用当今面临的挑战：

**可编程性**

**成本因素和商业模式**

**资源管理：**边缘计算具有去中心化、多样化和潜在的多租户特性。要使得资源既能最大化利用又能在多用户之间公平分配是个很大的挑战。

**隐私和安全**

Encryption is commonly used to prevent eavesdropping while in transit or when stored on disk, but when data is processed, it is generally necessary to access it. A possible solution to the problem is homomorphic encryption, which enables computation to be done on encrypted data so that the original data is not revealed to the computing system [68]. In homomorphic encryption, data is encrypted in a way that the encrypted data has the same mathematical properties as the cleartext. After performing mathematical operations on the encrypted data, the results can be decrypted, giving the same results as if the operations were performed on the original data.

Another security problem is the challenge of verifying that computation done by an edge server is correct and not tampered with by an attacker or a malicious operator. Verifiable computing is a technology that addresses this by having the computation include mathematical proof that the result is correct [69]. Combining verifiable computing with homomorphic encryption would enable outsourcing computations securely to untrusted parties.

#### **当前主流的边缘计算解决方案**

#### **AWS IoT for the edge**

![image-20201127195753653](C:\Users\yhtyh\AppData\Roaming\Typora\typora-user-images\image-20201127195753653.png)

**Device SDK** The Device SDK is a collection of libraries, sample code and associated documentation for developers to connect their devices to AWS IoT services .

**Device Gateway** The Device Gateway service is the connection point for devices communicating with AWS IoT services using IPv4 or IPv6.

**Message Broker** The Message Broker service transmits messages between IoT devices, AWS services, and applications7 . The service operates using a publish-subscribe model: messages are sent by publishing them on a topic and received by subscribing to it. Because of the publish-subscribe model, the devices and services can communicate through the system without knowing who is sending data or receiving it, enabling scalable bidirectional communication with low latency. 

**Device Shadow** The Device Shadow service persistently stores the state of a device, enabling the state to be retrieved and manipulated even when the device is temporarily offline .

**Rules Engine** The Rules Engine receives messages and delivers them to devices or AWS services according to predefined rules.

AWS IoT Greengrass Connectors are pre-built software modules that interact with infrastructure, devices, and clouds, such as the AWS cloud or third-party services . With connectors, logic and integration can be deployed on the edge without having to directly deal with protocols, credentials, or APIs. Connectors exist for connecting with various AWS and third-party services, running Docker containers, doing ML inference on the core device, and communicating with hardware serial ports and Raspberry Pi GPIO pins .

#### **Azure IoT Hub**

![image-20201127202939859](C:\Users\yhtyh\AppData\Roaming\Typora\typora-user-images\image-20201127202939859.png)

Device endpoints support MQTT and AMQP messaging protocols along with HTTPS, while the service endpoints support only AMQP, except for the direct method request endpoint, which uses HTTPS. 

**Device provisioning** Device Provisioning Service (DPS) is a helper service to the IoT Hub that simplifies the process of onboarding new IoT devices.

**Resource provider** Resource provider is a helper service to the IoT Hub that enables creating, deleting, and updating properties of IoT Hubs.

**Identity registry** Information of the identities of all devices permitted to connect to an IoT Hub is contained in an identity registry. 

**Device twin** Device twin is a replica of information of a device’s properties, such as its configuration, stored for each device connected to an IoT Hub.

**Direct methods** Direct methods are requests directed at a device that either execute or fail immediately.

**Jobs** Timed tasks called jobs may be set on an IoT Hub to execute direct methods on devices or update device twin properties. 

**Service SDK** Service SDKs support building backend applications that use the IoT Hub.

**Message routing** Message routing allows for routing of device telemetry messages and events from devices to endpoints, and enables filtering data arriving from devices before delivery.

**Event Grid integration** IoT Hub integrates with the Event Grid service to deliver data generated by IoT devices to the Event Grid .

**Access control and security** An access control policy specified in the IoT Hub controls the level of access a service or a device has when connecting to an IoT Hub through an endpoint . Security tokens with a limited lifetime generated by the Device or Service SDKs are used to authenticate devices and services; devices may also authenticate using secure certificates or custom authentication methods. All communications with endpoints are secured using the TLS protocol.

#### EdgeX Foundry

![image-20201127204819620](C:\Users\yhtyh\AppData\Roaming\Typora\typora-user-images\image-20201127204819620.png)

The microservices EdgeX Foundry consists of are organized in four service layers and two system layers. In EdgeX Foundry parlance, north side stands for the cloud and the network communicating with it, while south side stands for things and the devices, sensors, and actuators that interact with things . Communication between microservices is possible in north, south, and lateral directions.

**Core services layer** The core services layer consists of the services most fundamental to edge operation, separating the north and south side layers . These include core data service, which stores collected data; command service, which handles actuation requests; metadata service, which manages and stores metadata of the devices and sensors connected to the system; and configuration and registry service, which provides the system with information of the deployed microservices and their configuration.

**Device services layer** and device services SDK The device services layer consists of microservices that directly interact with IoT devices, such as end devices, sensors, and actuators .

**Security services layer** The security services layer contain microservices that provide services for the secure operation of an IoT system . These include secret store service, which securely stores secrets, such as tokens, passwords and certificates, required for other microservices for their operation; and API gateway, which acts as a single point of entry for external clients to the REST APIs exposed by EdgeX Foundry microservices, preventing unauthorized access.

**Management services layer** The management services layer provides functionality for managing microservices in the system, consisting of the system management agent (SMA), which provides functionality for controlling and monitoring microservices in the system . 

Aside from the basic infrastructure for cloud-to-edge communication and managing containers, KubeEdge does not provide other tools required for building IoT solutions, such as device management, data storage, device and sensor communications, edge intelligence, or security tools. Therefore KubeEdge is not a complete IoT platform, but one that can be incorporated into a selected architecture in addition to other elements, such as a public cloud platform.

#### **KubeEdge**

The architecture and components of KubeEdge are depicted in 3.4. KubeEdge consists of two parts: CloudCore that runs in the cloud and integrates with Kubernetes, facilitating communication with edge devices and managing edge device metadata and synchronization; and EdgeCore, a lightweight agent run on edge devices, managing containerized applications and bridging communications with devices and applications .

![image-20201127210601915](C:\Users\yhtyh\AppData\Roaming\Typora\typora-user-images\image-20201127210601915.png)

**CloudCore components**

**EdgeController** EdgeController is a bridge between Kubernetes running in the cloud and KubeEdge . It communicates with a Kubernetes system in the cloud by using standard APIs to connect with a Kubernetes API server, which is the central communication point of a Kubernetes installation. EdgeController relays control and monitoring messages between the Kubernetes cluster and edge devices.

**DeviceController** DeviceController manages device metadata, such as device status and other properties, by relaying metadata updates between Kubernetes and the edge . It communicates with a Kubernetes system in the cloud similarly with the EdgeController, by using standard APIs. Reported properties sent by the edge devices are delivered to Kubernetes, and desired properties received from Kubernetes are relayed to the edge.

**CloudHub** CloudHub is the cloud-side connection point for edge devices . It listens to and maintains connections from edge devices on a predefined address, and relays messages between CloudCore and the edge. CloudHub supports edge connections using HTTP over WebSocket protocol with TLS encryption or the QUIC protocol. It also sends connection status messages to the DeviceController when connections are established or lost.

**EdgeCore components**

**EdgeHub** EdgeHub is the edge-side connector for edge devices . It connects to the CloudHub using a predefined address and HTTP over WebSocket with TLS encryption or QUIC protocols and relays messages between other components and the cloud. It also sends periodic keepalive messages to maintain the cloud connection and reports cloud connection status to the other modules.

**MetaManager** MetaManager processes messages related to the management and monitoring of containerized workloads to access this data even when a cloud connection is not available . It receives updates, such as creation or deletion of workloads or status updates, stores them in a SQLite database on the local device, and relays them to the cloud or EdgeD when changes are detected. The stored state is periodically synchronized with container management. MetaManager also serves requests from other components that query for the stored metadata.

**DeviceTwin** DeviceTwin stores and processes updates to device metadata similarly to the MetaManager, to enable disconnected operations . It stores device metadata in an SQLite database on the local device, receives updates and queries to the data, and periodically synchronizes the data with the cloud.

**ServiceBus** ServiceBus is an HTTP client for the EdgeCore to enable communicating with application components, such as microservices exposing a REST API, running on the edge .

**EdgeD** EdgeD is responsible for managing containerized applications running on the edge device . After receiving management commands from MetaManager, it starts, deletes, or modifies local containerized workloads using Docker management tools. It also monitors the workloads and reports status information to MetaManager and periodically frees up disk space by removing dead containers and unused container images.

**EventBus** EventBus is a bridge between KubeEdge and MQTT publish/subscribe messaging . It functions as a client to a MQTT broker, which is typically running locally on the edge device, and relays messages between the broker and KubeEdge. Through EventBus, IoT devices supporting MQTT protocol can communicate with the KubeEdge deployment.

评价：KubeEdge试图建立一个开放的标准的边缘计算平台，很有前景。提供基本的元数据存储服务，完全没有设备管理能力；除了使用TLS加密，没有其他的安全措施（比如设备认证、访问控制、数据安全、API网关）；基于Kubernetes简单扩展即可将云原生应用搬到边缘，但是要完全支持IoT边缘应用需要许多的其他元素。

Despite its promise and plentitude of research efforts, edge computing has yet to find a killer application to break through as the cloud has. Particular challenges in edge computing include making it easy for developers to create edge computing applications; creating pricing and business models that make it profitable to produce and use edge computing systems; managing distributed and heterogeneous edge resources; and security and privacy issues.

参考论文：

**容器技术有关：**

[48] Wes Felter, Alexandre Ferreira, Ram Rajamony, and Juan Rubio. “An updated performance comparison of virtual machines and Linux containers”. In: 2015 IEEE International Symposium on Performance Analysis of Systems and Software, ISPASS 2015, Philadelphia, PA, USA, March 29-31, 2015. IEEE Computer Society, 2015, pp. 171–172. url: https://doi.org/10.1109/ISPASS.2015.7095802. 

[49] Anil Madhavapeddy, Richard Mortier, Charalampos Rotsos, David J. Scott, Balraj Singh, Thomas Gazagnaire, Steven Smith, Steven Hand, and Jon Crowcroft. “Unikernels: library operating systems for the cloud”. In: Architectural Support for Programming Languages and Operating Systems, ASPLOS ’13, Houston, TX, USA - March 16 - 20, 2013. Ed. by Vivek Sarkar and Rastislav Bodik. ACM, 2013, pp. 461– 472. url: https://doi.org/10.1145/2451116.2451167.

**为什么云计算满足了IoT的高计算能力和弹性却不能满足IoT的需要（为什么边缘计算会兴起）？**

[46] Weisong Shi, Jie Cao, Quan Zhang, Youhuizi Li, and Lanyu Xu. “Edge Computing: Vision and Challenges”. In: IEEE Internet of Things Journal 3.5 (2016), pp. 637– 646. url: https://doi.org/10.1109/JIOT.2016.2579198.

[56]Cisco Systems. Cisco Global Cloud Index: Forecast and Methodology, 2016–2021. Nov. 2018. url: https://www.cisco.com/c/en/us/solutions/collateral/ service-provider/global-cloud-index-gci/white-paper-c11-738085.html.

[4] Pedro Garc´ıa L´opez, Alberto Montresor, Dick H. J. Epema, Anwitaman Datta, Teruo Higashino, Adriana Iamnitchi, Marinho P. Barcellos, Pascal Felber, and Etienne Rivi´ere. “Edge-centric Computing: Vision and Challenges”. In: Computer Communication Review 45.5 (2015), pp. 37–42. url: https://doi.org/10.1145/2831347. 2831354.

[57] Wazir Zada Khan, Ejaz Ahmed, Saqib Hakak, Ibrar Yaqoob, and Arif Ahmed. “Edge Computing: A survey”. In: Future Gener. Comput. Syst. 97 (2019), pp. 219–235. url: https://doi.org/10.1016/j.future.2019.02.050.

**QUIC**

[39] Adam Langley, Alistair Riddoch, Alyssa Wilk, Antonio Vicente, Charles Krasic, Dan Zhang, Fan Yang, Fedor Kouranov, Ian Swett, Janardhan R. Iyengar, Jeff Bailey, Jeremy Dorfman, Jim Roskind, Joanna Kulik, Patrik Westin, Raman Tenneti, Robbie Shade, Ryan Hamilton, Victor Vasiliev, Wan-Teh Chang, and Zhongyi Shi. “The QUIC Transport Protocol: Design and Internet-Scale Deployment”. In: Proceedings of the Conference of the ACM Special Interest Group on Data Communication, SIGCOMM 2017, Los Angeles, CA, USA, August 21-25, 2017. ACM, 2017, pp. 183–196. url: https://doi.org/10.1145/3098822.3098842.

**雾计算**

[58] Flavio Bonomi, Rodolfo A. Milito, Jiang Zhu, and Sateesh Addepalli. “Fog computing and its role in the Internet of Things”. In: Proceedings of the first edition of the MCC workshop on Mobile cloud computing, MCCSIGCOMM 2012, Helsinki, Finland, August 17, 2012. Ed. by Mario Gerla and Dijiang Huang. ACM, 2012, pp. 13– 16. url: https://doi.org/10.1145/2342509.2342513.

**NFV 与 MEC**：

[60] Rodrigo Roman, Javier L´opez, and Masahiro Mambo. “Mobile Edge Computing, Fog et al.: A survey and analysis of security, threats and challenges”. In: Future Generation Comp. Syst. 78 (2018), pp. 680–698. url: https://doi.org/10.1016/ j.future.2016.11.009.

[61] Bo Han, Vijay Gopalakrishnan, Lusheng Ji, and Seungjoon Lee. “Network Function Virtualization: Challenges and opportunities for innovations”. In: IEEE Communications Magazine 53.2 (2015), pp. 90–97. url: https://doi.org/10.1109/MCOM. 2015.7045396.

**使用加密解决边缘计算所面临的安全问题：**

[68] Marten van Dijk, Craig Gentry, Shai Halevi, and Vinod Vaikuntanathan. “Fully Homomorphic Encryption over the Integers”. In: Advances in Cryptology - EUROCRYPT 2010, 29th Annual International Conference on the Theory and Applications of Cryptographic Techniques, Monaco / French Riviera, May 30 - June 3, 2010. Proceedings. Ed. by Henri Gilbert. Vol. 6110. Lecture Notes in Computer Science. Springer, 2010, pp. 24–43. url: https://doi.org/10.1007/978-3-642-13190- 5%5C_2.

**验证计算：**

[69] Rosario Gennaro, Craig Gentry, and Bryan Parno. “Non-interactive Verifiable Computing: Outsourcing Computation to Untrusted Workers”. In: Advances in Cryptology - CRYPTO 2010, 30th Annual Cryptology Conference, Santa Barbara, CA, USA, August 15-19, 2010. Proceedings. Ed. by Tal Rabin. Vol. 6223. Lecture Notes in Computer Science. Springer, 2010, pp. 465–482. url: https://doi.org/10.1007/978- 3-642-14623-7%5C_25.

**AWS IoT的安全性**

[79] Mahmoud Ammar, Giovanni Russello, and Bruno Crispo. “Internet of Things: A survey on the security of IoT frameworks”. In: J. Inf. Sec. Appl. 38 (2018), pp. 8–27. url: https://doi.org/10.1016/j.jisa.2017.11.002.

**各种边缘计算框架性能的比较**

[87] Halim Fathoni, Chao-Tung Yang, Chih-Hung Chang, and Chin-Yin Huang. “Performance Comparison of Lightweight Kubernetes in Edge Devices”. In: Pervasive Systems, Algorithms and Networks - 16th International Symposium, I-SPAN 2019, Naples, Italy, September 16-20, 2019, Proceedings. Ed. by Christian Esposito, Jiman Hong, and Kim-Kwang Raymond Choo. Vol. 1080. Communications in Computer and Information Science. Springer, 2019, pp. 304–309. url: https://doi.org/10. 1007/978-3-030-30143-9%5C_25.