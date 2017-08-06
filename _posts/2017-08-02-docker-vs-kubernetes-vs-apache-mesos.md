---
layout: post
title: "重新认识 Docker、Kubernetes 和 Apache Mesos"
categories: [Technology]
tags: [Docker, Kubernetes, Mesos]
permalink: /docker-vs-kubernetes-vs-apache-mesos.html
date: July 31, 2017
authror: Amr Abdelrazik
src: https://mesosphere.com/blog/docker-vs-kubernetes-vs-apache-mesos/
---

有无数的文章、讨论和社交网络上的交流在比较 Docker、Kubernetes 和 Mesos。
如果只听取部分信息，你会以为这三个开源项目正在为容器世界的霸权而决战。
你也可能相信选择其一几乎是一个宗教选择；真正的信徒维护他们的信仰，烧死胆敢考虑其它替代品的异端者。

这都是呓语。

虽然三种技术都让利用容器进行部署、管理和扩展应用程序成为可能，其实它们各自解决不同的问题，植根于非常不同的环境背景。
事实上，这三个广泛采用的工具链里没有一个是完全相同的。

与其比较这几项快速发展的技术的重叠特性，不如让我们重新审视每个项目的原始使命、架构、以及他们如何互相补充与交互。


### 从 Docker 说起…

今天的 Docker 公司，起始于一家平台即服务（Platform-as-a-Service，PaaS）初创公司，叫做 dotCloud。
dotCloud 团队发现，为大量应用程序和大量客户去管理软件依赖和二进制文件需要许多工作量。
因此他们将 Linux cgroups 和命名空间（Namespace）的功能合并为一个独立的易用的包，以便应用程序能在任何基础设施上无差别的运行。
这个包就是 Docker 镜像，它提供以下功能：

- **将应用程序和库封装进一个包**（Docker 镜像），应用程序可以一致地部署在多个环境；
- **提供类似 Git 的语义**，如 “docker push”, “docker commit”，让程序开发人员能轻松的采用新技术，并纳入其现有工作流；
- **将 Docker 镜像定义为不可变层**，支持不可变基础设施。发布的变动保存在单独的只读层，让重用镜像和追踪变更变得容易。层还能通过只传输变更而不是整个镜像，节省磁盘空间和网络带宽；
- **用不可变镜像启动 Docker 容器实例**，附带一个可写层来临时存储运行时更改，让应用程序快速部署和扩展变得容易。

Docker 越来越受欢迎，开发人员由在笔记本上运行容器，开始转为在生产环境运行容器。
这需要额外工具在多机之间协调容器，即容器编排。
有趣的是，运行于 Apache Mesos（下文详述）之上的 [Marathon](https://mesosphere.github.io/marathon/) 是支持 Docker 镜像（2014年6月）的首批容器编排工具之一。
那一年，Docker 的创始人兼 CTO Solomon Hykes 称赞 Mesos 为“[生产集群的黄金标准](https://www.google.com/url?q=https://www.youtube.com/watch?v=sGWQ8WiGN8Y&amp;feature=youtu.be&amp;t=35m10s&amp;sa=D&amp;ust=1500923856666000&amp;usg=AFQjCNFLtW96ZWnOUGFPX_XUuVOPdWrd_w)”。
不久，除运行于 Mesos 之上的 Marathon，还出现了许多容器编排技术：[Nomad](https://www.nomadproject.io/)， [Kubernetes](http://kubernetes.io/)，毫不意外，还有 Docker Swarm（[现在是 Docker Engine 的一部分](https://blog.docker.com/2016/06/docker-1-12-built-in-orchestration/)）。

随着 Docker 对开源文件格式进行商业化，该公司还开始引入工具来补充核心的 Docker 文件格式和运行引擎，包括：

- Docker hub 用于 Docker 镜像公共存储；
- Docker registry 用于内部存储；
- Docker cloud，用于构建和运行容器对托管服务；
- Docker datacenter 作为商业产品包含众多 Docker 技术。

![Docker](https://mesosphere.com/wp-content/uploads/2017/07/docker-host.png) <br />
_来源: www.docker.com._

Docker 将软件及其依赖封装进一个软件包中，这一有洞察力的方式改变了软件行业规则；正如 mp3 帮助重塑了音乐行业一样。
Docker 文件格式成为行业标准，主要的容器技术厂商（包括 Docker、Google、Pivotal、Mesosphere 等）成立了 [Cloud Native Computing Foundation (CNCF)](https://www.cncf.io/) 和 [Open Container Initiative (OCI)](https://www.opencontainers.org/)。
今天，CNCF 和 OCI 旨在确保容器技术之间的互操作性和标准接口，并确保使用任何工具构建的 Docker 容器可以运行在任何运行环境和基础架构上。


### 步入 Kubernetes

Google 一早就意识到 Docker 镜像文件的的潜力，并探索在 Google 的云平台上像服务一样交付容器编排。
Google 虽然在容器上有深厚的经验（他们在 Linux 中引入了 cgroups）, 但是在他们的基础架构上，内部容器和像 Borg 这样的分布式计算工具是紧密绑定在一起的。
因此，Google 没有使用任何已经存在系统的代码，而是重新设计了 Kubernetes 对 Docker 容器进行编排。
带着下面的目标和考虑，在 2015 年 2 月 kubernetes 发布了。

- **辅助应用开放者**利用强大的 Docker 容器编排工具，不再需要和底层基础架构进行交互。
- **提供标准的部署接口**和元语，以获得一致的应用部署的体验和跨云的 API。
- **建立一个模块化的 API 核心**，允许厂商以 Kubernetes 技术为核心进行系统集成。

2016 年 3 月，Google 向 CNCF [捐赠了 Kubernetes](https://www.linuxfoundation.org/news-media/announcements/2016/03/cloud-native-computing-foundation-accepts-kubernetes-first-hosted-0)，直到今天仍然保持着在这个项目的贡献者中首位的位置（其后是红帽，CoreOS 等）。

![Kubernetes](https://mesosphere.com/wp-content/uploads/2017/07/kubernetes-architecture.png) <br />
_来源: wikipedia_

应用开发者对 Kubernetes 产生了很大的兴趣，因为 Kubernetes 降低了他们对基础架构和运维团队的依赖。
厂商也喜欢 Kubernetes, 因为它提供了一种简单的方式去拥抱容器，并且为运行自有的 Kubernetes 部署的运营挑战提供了商业化的解决方案（这是很有价值的事情）。
另外值得注意的是，Kubernetes 在 CNCF 的框架下开源，相对的是 Docker Swarm，虽然它也是开源的，但是实际上是被 Docker 公司牢牢控制。

Kubernetes 的核心能力是给应用开发者提供强大的工具，编排无状态的 Docker 容器。
虽然已经有了多个提议，希望扩大这个项目的范围（比如数据分析和有状态的数据服务），这些提议仍然处于非常早期的阶段，能不能成功我们还需要保持观望。


### Apache Mesos

Apache Mesos是加州大学伯克利分校为了研发新一代集群管理器而发起的项目，它借鉴了 [Google Borg](https://research.google.com/pubs/pub43438.html) 系统和 [Facebook’s Tupperware](https://www.youtube.com/watch?v=C_WuUgTqgOc) 系统中大规模、分布式计算架构的相关经验，并把它应用到了mesos上。
当时Borg和Tupperware还是一个和基础设施紧密绑定，架构庞大，并且技术闭源的系统，而Mesos引入了一种开源的模块化架构，这种架构和底层的基础设施是完全独立开的。
为了使基础架构能够动态灵活扩展，[Twitter](https://youtu.be/F1-UEIG7u5g), [Apple(Siri)](http://www.businessinsider.com/apple-siri-uses-apache-mesos-2015-8), [Yelp](https://engineeringblog.yelp.com/2015/11/introducing-paasta-an-open-platform-as-a-service.html), [Uber](http://highscalability.com/blog/2016/9/28/how-uber-manages-a-million-writes-per-second-using-mesos-and.html), [Netflix](http://highscalability.com/blog/2016/9/28/how-uber-manages-a-million-writes-per-second-using-mesos-and.html) 还有很多高科技公司很快就把mesos应用到微服务，大数据计算，实时分析等业务上。

Mesos作为一个集群资源管理器，它解决了以下一系列的难题和挑战：

- **为了简化资源分配**，mesos将整个数据中心资源抽象为一个资源池，同时为私有云和公有云提供了一致的应用和操作体验；
- **为了提高资源利用率，减少成本**，mesos在同一基础架构上协调分配不同类型的工作任务，如数据分析，无状态微服务，分布式数据服务和传统应用程序;
- 使应用服务的相关任务（如部署，自我修复，扩展和升级）自动化，并且保证自动化流程高可用；
- 提供了有效的扩展性来运行新的应用程序而无需修改群集管理器和运行在它上面的的已有应用程序；
- 灵活的的扩展应用程序和底层基础架构，可以从几个节点扩展到数十个，甚至上万个节点。

Mesos具有一种独特的能力，它可以独自管理不同类型的工作任务，包括传统应用程序，如Java，无状态的Docker微服务，批处理作业，实时分析服务和有状态的分布式数据服务。
Mesos之所以能支持多种类型的工作任务，与它的两级架构设计密切相关，这种架构可以使应用在有感知的情况下完成资源调度。
这种调度方法是通过将应用程序特定的操作逻辑封装在“Mesos框架”（类似于操作中的运行手册）中来实现的。
然后，资源管理器Mesos Master在保持隔离的同时会把框架基础架构的这部分资源共享出来。
这种方法允许每个工作任务都有自己的专门调度器，这个调度器清楚其如何部署，扩展和升级等特定操作需求。
应用程序的调度器也是独立开发，管理和更新的，这样的设计使得Mesos具有高度的可扩展性，支持新的工作任务，或许将来会增加更多的功能特性。

![Mesos two-level scheduler](https://mesosphere.com/wp-content/uploads/2017/07/mesos-two-level-scheduler.png)

以团队内部如何管理服务升级为例：
无状态的应用可以从[“blue/green”](https://martinfowler.com/bliki/BlueGreenDeployment.html)部署方法中获益；当旧的应用程序还运行着的时候，一个新版本完全运行起来时，流量会在旧程序销毁时完全切换到新版本的应用中。
但升级数据型工作任务时（如HDFS， Cassandra），每次要先拿掉一个节点，保留一份本地数据卷以避免数据丢失，按照顺序逐个升级下线的节点，同时要在每个节点升级前后执行指定的检测命令。
这些升级步骤都是和应用程序或服务相关联的，甚至和特定版本相关。
这使得使用常规容器编排管理这些数据服务变得异常困难。

Mesos的这种保持每个工作任务以它自己的方式来管理资源调度的特性，使得许多公司将Mesos作为一个统一的平台，将微服务和数据服务同时运行在上面。
有一个类似的数据密集型架构可以参考，[“SMACK stack”](https://mesosphere.com/blog/2017/06/21/smack-stack-new-lamp-stack/),即Spark,Mesos,Akka,Cassandra,Kafka 技术栈。


### 几点声明

请注意，我们在描述 Apache Mesos 时并未提及任何有关容器编排的内容。
那么为什么人们总是自动把容器编排和 Mesos 关联起来呢？
容器编排是可以在mesos模块化结构上运行任务的一个例子，其功能已经由运行在Mesos之上的专门编排“框架”实现，该“框架”名为 Marathon。Marathon 最初在 cgroup 容器中管理应用程序包（例如JAR包、TAR包及ZIP文件），也是第一个支持 docker 容器的容器编排工具（2014年实现该功能）。

所以当人们用 Docker 和 Kubernetes 同 Mesos 比较时，实际上比较的是 Kubernets、Docker Swarm 和运行在 Mesos 上的 Marathon。

为什么这点很重要？
因为坦率的讲，Mesos并不关心谁在它上面运行。
Mesos可以弹性为 JAVA 应用服务器、Docker 容器编排、Jenkins CI 作业、Apache Spark分析、Apache Kafka流等共享基础设施提供集群服务。
Mesos 上甚至可以运行 Kubernetes 或其他容器编排工具，尽管一体化集成尚未实现。

![Mesos Workloads](https://mesosphere.com/wp-content/uploads/2017/07/mesos-workloads.png) <br />
_来源：Apache Mesos Survey 2016_

另一个值得考虑 Mesos 的因素（也是吸引大量企业架构师的原因）是其运行关键性任务的成熟度。
Mesos已经在大量生产环境（成千上万台服务器）中运行超过7年，这就是为什么说相对于市场上的其他容器技术，Mesos已经应用于大量生产环境，拥有可靠的规模。


### 这些都意味着什么？

综上所述，所有这三种技术都与 Docker 容器有关，并允许你使用容器编排实现应用程序可移植和扩展。
那么它们之间如何选择呢？
归结为：为工作任务选择合适的工具（甚至可能是不同的工作选择不同的工具）。
如果你是程序开发人员，正在探寻现代化的方式来构建和打包应用程序，或加速微服务计划，Docker 容器格式和开发工具是最好的途径。

如果你是一个开发／运维团队，希望构建专用于 Docker 容器编排的系统，并愿意着手将你的解决方案与底层基础设施（或依赖 Google Container Engine、Azure Container Service 之类的公共云基础设施）进行整合，Kubernetes 是可考虑的不错的技术选择。

如果你希望建立一个可靠的平台，允许多项关键工作任务如 Docker 容器、传统应用程序（例如 Java）和分布式数据服务（例如 Spark、Kafka、Cassandra、Elastic），并希望所有这些可以跨云服务商和／或数据中心移植，那么 Mesos（或 Mesosphere DC/OS）适合你。

无论你作何选择，都将拥有一套可以更有效利用服务器资源、简化应用程序移植、并提高开发人员敏捷性的工具。
绝对没错。
