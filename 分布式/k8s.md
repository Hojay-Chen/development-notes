# 一、Kubernetes 篇

## 1. 为什么需要 Kubernetes？

要理解 Kubernetes（简称 K8s），得先看看应用程序的部署方式是怎么一步步演进过来的。

**物理机时代**：最早的时候，应用直接部署在物理服务器上。一台机器跑一个应用，资源浪费严重，而且应用之间互相影响——一个应用吃满 CPU，其他应用全卡住。

**虚拟机时代**：于是有了虚拟机（VM）。一台物理机上虚拟出多台虚拟机，每台虚拟机跑一个应用，彼此隔离。但虚拟机的问题在于**太重**——每个虚拟机都要跑一整套操作系统，启动慢、占资源，光是为了隔离就付出了巨大代价。

**容器时代**：Docker 的出现改变了一切。容器把应用和它的依赖打包在一起，**共享宿主机的操作系统内核**，不需要每个应用都跑一个完整 OS。容器启动只要秒级，资源占用极小，"一次构建，到处运行"。这就像从"每家自建独立别墅（虚拟机）"变成了"住公寓（容器），共用地基和水电（内核），但各家独立门户"。

![部署方式演进](https://raw.githubusercontent.com/Hojay-Chen/development-note-resource/main/images/k8s_evolution.png)

容器解决了"怎么打包和运行应用"的问题，但很快新的痛点出现了：**容器多了怎么管？**

当你只有几个容器时，手动 `docker run` 就够了。但当你有几百上千个容器时，一系列问题接踵而至：

- **容器挂了怎么办？** 某个容器崩溃了，谁来把它重新拉起来？
- **流量大了怎么扩？** 突然来了一波流量，怎么自动增加容器数量扛住压力？
- **容器在哪台机器上？** 哪台机器还有空闲资源？容器该调度到哪台机器？
- **容器 IP 一直变怎么办？** 容器重启后 IP 就变了，调用方怎么找到它？
- **怎么平滑升级？** 新版本上线，怎么做到不停机、出问题能回滚？

> **核心矛盾**：容器解决了"单机上的应用打包与隔离"，但没有解决"大规模容器的集群管理"。我们需要一个系统来统一调度、编排、管理成百上千的容器——这就是**容器编排系统（Container Orchestration）**要解决的问题，而 Kubernetes 就是其中最主流的那个。

---

## 2. Kubernetes 是什么？

Kubernetes（K8s，因为 K 和 s 之间有 8 个字母）是 Google 基于内部使用了十年的 Borg 系统开源出来的**容器编排平台**。通俗地讲，如果把每个容器比作一艘船，那 K8s 就是**舰队的总指挥**——你只需要告诉它"我要 3 艘船运这批货"，剩下的事（派几艘、派到哪、船沉了补一艘、换班轮替）它全自动搞定。

K8s 最核心的设计哲学是**声明式 API（Declarative API）**，这与传统的命令式操作有本质区别：

- **命令式（Imperative）**：你告诉系统"**做什么**"。比如"启动一个容器""再启动一个""把那个停掉"。你每一步都要手动下达指令，系统只是被动执行。
- **声明式（Declarative）**：你告诉系统"**我要的状态是什么**"。比如"我要 3 个 Nginx 容器在运行"。至于怎么达到这个状态——启动几个、停掉几个、调度到哪台机器——由 K8s 自己去算并维持。

![声明式 vs 命令式](https://raw.githubusercontent.com/Hojay-Chen/development-note-resource/main/images/k8s_declared_vs_imperative.png)

声明式的好处在于**自愈**和**幂等**。假设你声明了"3 个 Nginx"，如果某个容器崩溃了，K8s 发现当前只有 2 个在运行，与期望状态不符，就会自动再拉起一个补上。你不需要写一堆"如果挂了就重启"的脚本，K8s 的控制循环会持续不断地"比对期望状态 vs 实际状态"，发现偏差就纠正——这套机制叫做 **调谐（Reconcile）**。

> **一句话理解 K8s**：你声明"期望状态"，K8s 持续调谐"实际状态"去逼近期望状态。容器挂了自动补，流量来了自动扩，升级出问题自动回滚——全都围绕"维持期望状态"这个核心。

---

## 3. 核心架构总览

理解了 K8s 的定位，接下来看它的整体架构。一个 K8s 集群由两部分组成：**控制面（Control Plane）**和**工作节点（Worker Node）**。

打个比方：控制面是公司的**管理层**——负责决策、调度、记录；工作节点是**生产线上的工人**——负责真正干活（运行容器）。管理层自己不跑业务容器，只负责指挥；工人节点听从管理层的指令，实际承载应用。

![K8s 集群架构总览](https://raw.githubusercontent.com/Hojay-Chen/development-note-resource/main/images/k8s_architecture.png)

**控制面（Control Plane）**是集群的大脑，包含四个核心组件：

- **kube-apiserver**：集群的唯一入口。所有操作（创建、查询、删除资源）都要经过它，相当于公司的"前台"，谁想找管理层都得先过它。
- **etcd**：集群的"数据库"，以键值对形式存储整个集群的所有状态数据。它是**唯一的有状态组件**，集群的"记忆"全在这里。
- **kube-scheduler**：调度器。当有新的 Pod 需要创建时，它负责决定这个 Pod 该放到哪个工作节点上——就像人事部把新员工分配到合适的工位。
- **kube-controller-manager**：控制器管理器。里面跑着各种控制器，它们是"调谐"机制的具体执行者，持续监听集群状态并纠正偏差。

**工作节点（Worker Node）**是干活的机器，包含三个核心组件：

- **kubelet**：节点上的"代理人"，听命于控制面。它负责管理本节点上的 Pod 生命周期——控制面说"在这台机器上启动这个 Pod"，kubelet 就去实际操作容器运行时（如 Docker / containerd）来拉起容器。
- **kube-proxy**：负责节点上的网络代理和负载均衡，让 Service 的流量能正确转发到对应的 Pod。
- **容器运行时（Container Runtime）**：真正运行容器的软件，比如 containerd、Docker。kubelet 不直接操作容器，而是通过它来管理。

> **架构要点**：控制面负责"想"（决策、调度、记录状态），工作节点负责"做"（运行容器）。所有组件通过 kube-apiserver 通信，所有状态存在 etcd 里。这套"大脑 + 四肢"的分离，正是 K8s 能管理超大规模集群的基础。

---

## 4. 核心组件详解

第 3 节的架构图展示了各组件的位置，但它们具体怎么协作？我们通过**一个 Pod 从创建到运行的完整过程**，把各组件串起来理解。

假设你执行了 `kubectl apply -f nginx.yaml`，想创建一个 Nginx Pod。接下来发生的事情：

**① 请求进入 apiserver**：`kubectl` 命令行工具把 YAML 里的期望状态通过 HTTP 请求发给 **kube-apiserver**。apiserver 会做认证、鉴权、校验，确认这个请求合法且格式正确。

**② 状态写入 etcd**：apiserver 把这个 Pod 的定义（期望有一个 Nginx Pod）写入 **etcd**。此时 etcd 里记录着"有一个 Pod 叫 nginx，期望状态是 Running，但还没分配节点"。

**③ Controller 发现并处理**：**kube-controller-manager** 里的控制器持续监听 etcd 的变化（通过 apiserver）。它发现有一个新 Pod 还没被调度，但注意——控制器本身不负责调度，它负责的是"维持期望状态"。

**④ Scheduler 决定去哪**：**kube-scheduler** 同样在监听。它发现有未调度的 Pod，就开始"看简历选工位"——评估所有工作节点的资源（CPU、内存够不够）、亲和性、约束条件，最终为这个 Pod 选定一个目标节点，并把结果写回 etcd："nginx Pod 分配到 node-2"。

**⑤ Kubelet 拉起容器**：目标节点 **node-2** 上的 **kubelet** 一直在监听"有没有分配给我的 Pod"。它一发现 nginx Pod 被调度到了自己这里，就立刻调用**容器运行时**（containerd）拉取 Nginx 镜像、启动容器。容器跑起来后，kubelet 把"Pod 已 Running"的状态上报回 apiserver，最终存入 etcd。

![组件协作流程](https://raw.githubusercontent.com/Hojay-Chen/development-note-resource/main/images/k8s_component_flow.png)

**⑥ kube-proxy 配置网络**：如果这个 Pod 需要被访问，**kube-proxy** 会负责配置 iptables / IPVS 规则，让后续的网络流量能正确路由到这个 Pod。

> **协作要点**：所有组件都不直接互相调用，而是**都通过 apiserver 读写 etcd 里的状态来间接协作**。这种"共享状态、各自调谐"的设计，叫做**List-Watch 机制**——每个组件 Watch（监听）自己关心的状态变化，各司其职。这就是 K8s 的"事件驱动 + 控制循环"精髓。

---

## 5. 核心概念：Pod

在 K8s 里，你不会直接创建或管理容器，而是创建 **Pod**。**Pod 是 K8s 中最小的可部署和调度单元**，而不是容器。

为什么不直接管容器，非要套一层 Pod？这要从容器的隔离粒度说起。有时候，两个容器需要**紧密协作**——比如一个主业务容器和一个负责收集日志的 sidecar 容器，它们需要共享网络、共享存储、能通过 localhost 互相通信。如果直接管两个独立容器，要实现这种"亲密关系"很麻烦。Pod 的设计就是：**把需要紧密协作的多个容器打包成一个整体**，它们共享同一个网络命名空间（同一个 IP、同一组端口）和存储卷，像住在同一个房间里的室友。

![Pod 与容器的关系](https://raw.githubusercontent.com/Hojay-Chen/development-note-resource/main/images/k8s_pod.png)

理解 Pod 的几个关键点：

- **一个 Pod 可以包含一个或多个容器**，但最常见的还是一个 Pod 一个容器。多容器 Pod 是为 sidecar（边车）等紧密协作场景设计的。
- **同一个 Pod 内的容器共享网络**：它们可以用 `localhost` 互相通信，但要注意端口不能冲突。
- **Pod 是临时的**：Pod 会因为节点故障、升级、缩容等原因被销毁重建，重建后的 Pod 会拿到新的 IP。**不要指望 Pod 的 IP 是稳定的**，这正是后面 Service 存在的原因。
- **Pod 是调度的最小单位**：K8s 不会把一个容器单独调度，而是把整个 Pod 作为一个整体调度到某个节点上。

> **为什么是 Pod 而不是容器**：Pod 在容器之上加了一层"亲密容器组"的抽象，让需要共享网络和存储的容器天然协作。同时，K8s 不直接操作容器，而是通过 Pod 间接管理——这样未来即使容器运行时从 Docker 换成 containerd，K8s 的上层模型也不用变。

---

## 6. 核心概念：Service 与网络

上一节说到，Pod 是临时的，IP 会变。这就带来一个大问题：**如果 Pod 的 IP 随时在变，调用方怎么稳定地找到它？**

想象一下，你的前端 Pod 要调用后端 Pod，但后端 Pod 今天 IP 是 `10.0.0.5`，重启后变成了 `10.0.0.8`，前端难道要每次手动改配置吗？更别说后端可能有 3 个副本 Pod，前端到底该调哪个？

**Service** 就是解决这个问题的。Service 为一组 Pod 提供一个**稳定的访问入口（固定的虚拟 IP 和 DNS 名）**，无论背后的 Pod 怎么生灭变化，Service 的地址始终不变。

那 Service 怎么知道要转发给哪些 Pod？靠 **Label Selector（标签选择器）**。每个 Pod 可以被打上标签（Label，键值对），比如 `app=nginx`。Service 通过声明 "我要选中所有 `app=nginx` 的 Pod"，就自动和这些 Pod 建立了关联。当有 Pod 被创建/销毁时，只要标签匹配，Service 的转发列表会自动更新。

![Service 转发到 Pod](https://raw.githubusercontent.com/Hojay-Chen/development-note-resource/main/images/k8s_service.png)

Service 的工作流程：外部或集群内的请求打到 Service 的虚拟 IP（ClusterIP）→ kube-proxy 在节点上维护的转发规则把流量**负载均衡**地分发到背后匹配的各个 Pod。Pod 挂了，Service 自动把流量切到还活着的 Pod；新 Pod 起来了，自动加入转发列表。

> **Service 的核心价值**：用稳定的虚拟 IP + DNS 屏蔽了 Pod IP 的易变性，用 Label Selector 实现了 Pod 与服务的松耦合。调用方永远只需要认 Service 的地址，不用关心背后有几个 Pod、IP 是什么。

---

## 7. 核心概念：Deployment 与副本管理

前面讲了 Pod 是运行单元、Service 是访问入口，但你总不能手动一个个创建 Pod 吧？万一要跑 3 个副本呢？万一某个 Pod 挂了呢？**Deployment** 就是帮你管理 Pod 副本的"管理者"。

Deployment 是一种**控制器**，你通过它来声明"我要 3 个 Nginx Pod 副本"，它就负责维持这个状态——这正是第 2 节讲的声明式 + 调谐思想的体现。但 Deployment 不是直接管 Pod 的，中间还有一层 **ReplicaSet**。

层级关系是这样的：**Deployment 管理 ReplicaSet，ReplicaSet 管理 Pod**。

- **ReplicaSet** 的职责很单一：确保任意时刻都有指定数量的 Pod 副本在运行。你声明 3 个，它就盯着，少了就补，多了就删——这是"自愈"和"扩缩容"的基础。
- **Deployment** 在 ReplicaSet 之上，主要负责**滚动更新和回滚**。为什么 Deployment 不直接管 Pod 而要套一层 ReplicaSet？因为**版本管理**：每次你更新镜像版本，Deployment 会创建一个**新的 ReplicaSet**，逐步把旧 ReplicaSet 的 Pod 替换成新 ReplicaSet 的 Pod。这样旧 ReplicaSet 保留着（只是副本数归零），万一新版本有问题，可以直接回滚到旧 ReplicaSet。

![Deployment 层级与滚动更新](https://raw.githubusercontent.com/Hojay-Chen/development-note-resource/main/images/k8s_deployment.png)

**滚动更新**的过程：假设从 v1 升级到 v2，Deployment 不会一次性把所有 v1 Pod 杀掉（那会导致服务中断），而是先创建一个 v2 Pod，再杀一个 v1 Pod，逐步替换，保证升级过程中始终有足够数量的 Pod 在服务。如果升级到一半发现 v2 有问题，一键回滚，Deployment 就把 v2 ReplicaSet 缩容、v1 ReplicaSet 扩容回去。

> **Deployment 的核心价值**：你只声明"期望几个副本、用什么镜像"，Deployment + ReplicaSet 自动维持副本数（自愈、扩缩容），并支持平滑的滚动更新与一键回滚。你不需要手动操作任何一个 Pod。

---

## 8. 一个完整的部署流程

前面几节分别讲了架构、组件、Pod、Service、Deployment，现在把它们串成一条完整的主线，看看从你写下 YAML 到服务真正可用，到底发生了什么。

**第一步：写声明文件**。你编写一个 Deployment YAML，声明"我要 3 个 Nginx Pod，镜像版本 1.21"，再写一个 Service YAML，声明"用 Label Selector 选中这些 Pod，对外暴露 80 端口"。

**第二步：提交给 apiserver**。执行 `kubectl apply -f`，请求到达 kube-apiserver，经过认证鉴权后，Deployment 和 Service 的定义被写入 etcd。

**第三步：Controller 创建 ReplicaSet → Pod**。Deployment 控制器监听到新的 Deployment，创建一个 ReplicaSet；ReplicaSet 发现期望 3 个副本但当前 0 个，于是创建 3 个 Pod 资源对象写入 etcd（此时 Pod 还没分配节点）。

**第四步：Scheduler 调度**。kube-scheduler 监听到 3 个未调度的 Pod，分别根据资源、亲和性等规则，把它们分配到合适的工作节点，结果写回 etcd。

**第五步：Kubelet 拉起容器**。各目标节点上的 kubelet 监听到"有 Pod 分配给我了"，调用容器运行时拉取镜像、启动容器。Pod 进入 Running 状态，状态上报回 etcd。

**第六步：kube-proxy 配置网络**。Service 控制器和各节点的 kube-proxy 配合，建立从 Service 虚拟 IP 到这 3 个 Pod 的转发规则。

**第七步：服务可用**。现在，其他服务或外部用户通过 Service 的稳定地址访问，流量被负载均衡到 3 个健康的 Nginx Pod。如果某个 Pod 挂了，ReplicaSet 自动补一个；如果某天流量增大，改一下副本数，Deployment 自动扩容——全程无需手动干预。

![完整部署流程](https://raw.githubusercontent.com/Hojay-Chen/development-note-resource/main/images/k8s_deploy_flow.png)

> **全流程回顾**：YAML 声明 → apiserver 校验写入 etcd → Controller 创建 Pod → Scheduler 调度到节点 → Kubelet 拉起容器 → kube-proxy 配置网络 → Service 对外提供稳定访问。每一步都是组件 Watch 到状态变化后的自动响应，没有一步需要人工操作容器。这就是 K8s"声明式 + 控制循环"的完整威力。

---

## 9. 总结

### 9.1 K8s 解决的核心问题

回到第 1 节提出的痛点，K8s 是如何逐个解决的：

- **容器挂了怎么办** → ReplicaSet 控制器持续调谐，Pod 挂了自动补一个。
- **流量大了怎么扩** → 改声明里的副本数，Deployment 自动扩容；配合 HPA 还能根据 CPU 自动伸缩。
- **容器在哪台机器上** → Scheduler 自动根据资源情况调度，不用人工分配。
- **容器 IP 一直变** → Service 提供稳定的虚拟 IP 和 DNS，屏蔽 Pod IP 的变化。
- **怎么平滑升级** → Deployment 的滚动更新，逐步替换，出问题一键回滚。

### 9.2 核心概念速查表

| 概念 | 定位 | 一句话理解 |
|------|------|-----------|
| **Pod** | 最小调度单元 | 一组紧密协作的容器，共享网络和存储，是 K8s 不直接管容器的原因 |
| **Service** | 网络抽象 | 给一组易变的 Pod 提供稳定的访问入口，靠 Label Selector 关联 |
| **Deployment** | 副本管理器 | 声明期望副本数，管理 ReplicaSet，负责自愈、扩缩容、滚动更新、回滚 |
| **ReplicaSet** | 副本维持器 | 确保 Pod 副本数始终符合期望，是 Deployment 的执行层 |
| **kube-apiserver** | 集群入口 | 所有操作的唯一入口，负责认证、鉴权、校验 |
| **etcd** | 状态存储 | 集群的"数据库"，以键值对存储所有状态数据 |
| **kube-scheduler** | 调度器 | 决定 Pod 该放到哪个工作节点 |
| **kubelet** | 节点代理 | 工作节点上的代理人，管理本节点 Pod 生命周期 |
| **kube-proxy** | 网络代理 | 维护 Service 到 Pod 的转发规则，实现负载均衡 |

### 9.3 学习建议

1. **先建立全局观再钻细节**：先理解"声明式 + 控制循环"的核心思想，和"控制面 + 工作节点"的架构，再去看各个组件和资源的细节，不会迷失。
2. **动手搭一个集群**：用 minikube 或 kind 在本地起一个单节点集群，亲手 `kubectl apply` 一个 Deployment + Service，观察 Pod 的创建、扩缩容、滚动更新，比看十篇文章都管用。
3. **以"一个请求的生命周期"为主线**：像第 8 节那样，把 YAML → apiserver → etcd → Controller → Scheduler → kubelet → Service 这条主线想通，K8s 的各组件就串起来了。
