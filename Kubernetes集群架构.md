# Kubernetes集群架构

​           Kubernetes属于典型的Server-Client形式的二层架构，在程序级别，Master主要由API Server（kube-apiserver）、Controller-Manager（kube-controller-manager）和Scheduler（kube-scheduler）这3个组件，以及一个用于集群状态存储的etcd存储服务组成，它们构成整个集群的控制平面；而每个Node节点则主要包含kubelet、kube-proxy及容器运行时（Docker是最为常用的实现）3个组件，它们承载运行各类应用容器。各组件如图1-8中的粗体部分组件所示。

<img src="http://pic.ccdxw.top/img/NeatReader-1678184767482.png" style="zoom:150%;" />

## 1. Master组件

​          Master组件是集群的“脑力”输出者，它维护有Kubernetes的所有对象记录，负责持续管理对象状态并响应集群中各种资源对象的管理操作，以及确保各资源对象的实际状态与所需状态相匹配。控制平面的各组件支持以单副本形式运行于单一主机，也能够将每个组件以多副本方式同时运行于多个主机上，提高服务可用级别。控制平面各组件及其主要功能如下。

### 1.1 API Server

​          API Server是Kubernetes控制平面的前端，支持不同类型应用的生命周期编排，包括部署、缩放和滚动更新等。它还是整个集群的网关接口，由kube-apiserver守护程序运行为服务，通过HTTP/HTTPS协议将RESTful API公开给用户，是发往集群的所有REST操作命令的接入点，用于接收、校验以及响应所有的REST请求，并将结果状态持久存储于集群状态存储系统（etcd）中。

### 1.2 集群状态存储 etcd

​              Kubernetes集群的所有状态信息都需要持久存储于存储系统etcd中。etcd是由CoreOS基于Raft协议开发的分布式键值存储，可用于服务发现、共享配置以及一致性保障（如数据库主节点选择、分布式锁等）。显然，在生产环境中应该以etcd集群的方式运行以确保其服务可用性，并需要制定周密的备份策略以确保数据安全可靠。

​           etcd还为其存储的数据提供了监听（watch）机制，用于监视和推送变更。API Server是Kubernetes集群中唯一能够与etcd通信的组件，它封装了这种监听机制，并借此同其他各组件高效协同。

### 1.3 控制器管理器 kube-controller-manager

​          控制器负责实现用户通过API Server提交的终态声明，它通过一系列操作步骤驱动API对象的当前状态逼近或等同于期望状态。Kubernetes提供了驱动Node、Pod、Server、Endpoint、ServiceAccount和Token等数十种类型API对象的控制器。从逻辑上讲，每个控制器都是一个单独的进程，但是为了降低复杂性，它们被统一编译进单个二进制程序文件kube-controller-manager（即控制器管理器），并以单个进程运行。

### 1.4 调度器 kube-scheduler

​         Kubernetes系统上的调度是指为API Server接收到的每一个Pod创建请求，并在集群中为其匹配出一个最佳工作节点。kube-scheduler是默认调度器程序，它在匹配工作节点时的考量因素包括硬件、软件与策略约束，亲和力与反亲和力规范以及数据的局部性等特征。

## 2. node组件 （work node）

​          Node组件是集群的“体力”输出者，因而一个集群通常会有多个Node以提供足够的承载力来运行容器化应用和其他工作负载。每个Node会定期向Master报告自身的状态变动，并接受Master的管理。

### 2.1 kubelet

​          kubelet是Kubernetes中最重要的组件之一，是运行于每个Node之上的“节点代理”服务，负责接收并执行Master发来的指令，以及管理当前Node上Pod对象的容器等任务。它支持从API Server以配置清单形式接收Pod资源定义，或者从指定的本地目录中加载静态Pod配置清单，并通过容器运行时创建、启动和监视容器。

​         kubelet会持续监视当前节点上各Pod的健康状态，包括基于用户自定义的探针进行存活状态探测，并在任何Pod出现问题时将其重建为新实例。它还内置了一个HTTP服务器，监听TCP协议的10248和10250端口：10248端口通过/healthz响应对kubelet程序自身的健康状态进行检测；10250端口用于暴露kubelet API，以验证、接收并响应API Server的通信请求。

### 2.2 容器运行时环境

​         Pod是一组容器组成的集合并包含这些容器的管理机制，它并未额外定义进程的边界或其他更多抽象，因此真正负责运行容器的依然是底层的容器运行时。kubelet通过CRI（容器运行时接口）可支持多种类型的OCI容器运行时，例如docker、containerd、CRI-O、runC、fraki和Kata Containers等。

### 2.3 kube-proxy

​          kube-proxy也是需要运行于集群中每个节点之上的服务进程，它把API Server上的Service资源对象转换为当前节点上的iptables或（与）ipvs规则，这些规则能够将那些发往该Service对象ClusterIP的流量分发至它后端的Pod端点之上。kube-proxy是Kubernetes的核心网络组件，它本质上更像是Pod的代理及负载均衡器，负责确保集群中Node、Service和Pod对象之间的有效通信。

## 3.核心附件（add-ons）

​         附件（add-ons）用于扩展Kubernetes的基本功能，它们通常运行于Kubernetes集群自身之上，可根据重要程度将其划分为必要和可选两个类别。网络插件是必要附件，管理员需要从众多解决方案中根据需要及项目特性选择，常用的有Flannel、Calico、Canal、Cilium和Weave Net等。KubeDNS通常也是必要附件之一，而Web UI（Dashboard）、容器资源监控系统、集群日志系统和Ingress Controller等是常用附件。

- CoreDNS：Kubernetes使用定制的DNS应用程序实现名称解析和服务发现功能，它自1.11版本起默认使用CoreDNS——一种灵活、可扩展的DNS服务器；之前的版本中用到的是kube-dns项目，SkyDNS则是更早一代的解决方案。

  skydns-->kube-dns-->CoreDns

- Dashboard：基于Web的用户接口，用于可视化Kubernetes集群。Dashboard可用于获取集群中资源对象的详细信息，例如集群中的Node、Namespace、Volume、ClusterRole和Job等，也可以创建或者修改这些资源对象。

- 容器资源监控系统：监控系统是分布式应用的重要基础设施，Kubernetes常用的指标监控附件有Metrics-Server、kube-state-metrics和Prometheus等。

- 集群日志系统：日志系统是构建可观测分布式应用的另一个关键性基础设施，用于向监控系统的历史事件补充更详细的信息，帮助管理员发现和定位问题；Kubernetes常用的集中式日志系统是由ElasticSearch、Fluentd和Kibana（称之为EFK）组合提供的整体解决方案。

- Ingress Controller：Ingress资源是Kubernetes将集群外部HTTP/HTTPS流量引入到集群内部专用的资源类型，它仅用于控制流量的规则和配置的集合，其自身并不能进行“流量穿透”，要通过Ingress控制器发挥作用；目前，此类的常用项目有Nginx、Traefik、Envoy、Gloo、kong及HAProxy等。

在这些附件中，CoreDNS、监控系统、日志系统和Ingress控制器基础支撑类服务是可由集群管理的基础设施，而Dashboard则是提高用户效率和体验的可视化工具，类似的项目还有polaris和octant等。