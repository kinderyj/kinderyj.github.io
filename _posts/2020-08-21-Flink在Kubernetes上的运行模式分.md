# 1. 前言

Apache Flink是一个分布式流处理引擎，它提供了丰富且易用的API来处理有状态的流处理应用，并且在支持容错的前提下，高效、大规模的运行此类应用。通过支持事件时间(event-time)、计算状态(state)以及恰好一次(exactly-once)的容错保证，Flink迅速被很多公司采纳，成为了新一代的流计算处理引擎。2020年2月11日，社区发布了Flink 1.10.0版本, 该版本对性能和稳定性做了很大的提升，同时引入了native Kubernetes的特性。对于Flink的下一个稳定版本，社区在2020年4月底冻结新特性的合入，预计在2020年5-6月会推出Flink1.11，该版本重点关注新特性的合入（如FLIP-105，FLIP-115，FLIP-27等）与内核运行时的功能增强，以扩展Flink的使用场景和应对更复杂的应用逻辑。。

1.1. Flink为什么选择Kubernetes

Kubernetes项目源自Google内部Borg项目，基于Borg多年来的优秀实践和其超前的设计理念，并凭借众多豪门、大厂的背书，时至今日，Kubernetes已经成长为容器管理领域的事实标准。在大数据及相关领域，包括Spark，Hive，Airflow，Kafka等众多知名产品正在迁往Kubernetes，Apache Flink也是其中一员。Flink选择Kubernetes作为其底层资源管理平台，原因包括两个方面：

1）Flink特性：流式服务一般是常驻进程，经常用于电信网质量监控、商业数据即席分析、实时风控和实时推荐等对稳定性要求比较高的场景；

2）Kubernetes优势：为在线业务提供了更好的发布、管理机制，并保证其稳定运行，同时Kubernetes具有很好的生态优势，能很方便的和各种运维工具集成，如prometheus监控，主流的日志采集工具等；同时K8S在资源弹性方面提供了很好的扩缩容机制，很大程度上提高了资源利用率。

## 1.2. Flink on Kubernetes的发展历史

在Flink的早期发行版1.2中，已经引入了Flink Session集群模式，用户得以将Flink集群部署在Kubernetes集群之上。随着Flink的逐渐普及，越来越多的Flink任务被提交在用户的集群中，用户发现在session模式下，任务之间会互相影响，隔离性比较差，因此在Flink 1.6版本中，推出了Per Job模式，单个任务独占一个Flink集群，很大的程度上提高了任务的稳定性。在满足了稳定性之后，用户觉得这两种模式，没有做到资源按需创建，往往需要凭用户经验来事先指定Flink集群的规格，在这样的背景之下，native session模式应用而生，在Flink 1.10版本进入Beta阶段，我们增加了native per job模式，在资源按需申请的基础上，提高了应用之间的隔离性。

本文根据Flink在Kubernetes集群上的运行模式的趋势，依次分析了这些模式的特点，并在最后介绍了flink operator方案及其优势。

# 2. Flink运行模式

本文首先分析了Apache Flink 1.10在kubernetes集群上已经GA（生产可用）的两种部署模式，然后分析了处于Beta版本的native session部署模式和即将在Flink1.11发布的native per-job部署模式，最后根据这些部署模式的利弊，介绍了当前比较native kubernetes的部署方式，flink-operator。我们正在使用的Flink版本已经很好的支持了native session和native per-job两种模式，在flink-operator中，我们也对这两种模式也做了支持。接下来将按照以下顺序分析了Flink的运行模式，读者可以结合自身的业务场景，考量适合的Flink运行模式。

Flink session模式
Flink per-job模式
Flink native session模式
Flink native per-job模式
这四种部署模式的优缺点对比，可以用如下表格来概括，更多的内容，请参考接下来的详细描述。此外，Flink社区正在研发Flink Application模式，限于篇幅，本文不展开分析，感兴趣可以参与Flink Application Mode讨论。

![](http://huangxuan.me/img/blog-desktop.jpg)

![img](/img/in-post/(forcify.jpg)


## 2.1. Session Cluster模式

### 2.1.1. 原理简介

Session模式下，Flink集群处于长期运行状态，当集群的Master组件接收到客户端提交的任务后，对任务进行分析并处理。用户将Flink集群的资源描述文件提交到Kubernetes之后，Flink集群的FlinkMaster和TaskManager会被创建出来，如下图所示，TaskManager启动后会向ResourceManager模块注册，这时Flink Session集群已经准备就绪。当用户通过Flink Clint端提交了Job任务时，Dispatcher收到该任务请求，将请求转发给JobManager，由JobManager将任务分配给具体的TaskManager。



添加描述

### 2.1.2. 特点分析

这种类型的Flink集群，FlinkMaster和TaskManager是以Kubernetes deployment的形式长期运行在Kubernetes集群中。在提交作业之前，必须先创建好Flink session集群。多个任务可以同时运行在同一个集群内，任务之间共享K8sResourceManager、Dispatcher，但是JobManager是单独的。这种方式比较适合运行短时作业、即席查询、任务提交频繁、或者对任务启动时长比较敏感的场景。

优点：作业提交的时候，FlinkMaster和TaskManager已经准备好了，当资源充足时，作业能够立即被分配到TaskManager执行，无需等待FlinkMaster、TaskManager、Service等资源的创建；
缺点：1) 需要在提交Job任务之前先创建Flink集群，需要提前指定TaskManager的数量，但是在提交任务前，是难以精准把握具体资源需求的，指定的多了，会有大量TaskManager处于闲置状态，资源利用率就比较低，指定的少了，则会有任务分配不到资源，只能等集群中其他作业执行完成后，释放了资源，下一个作业才会被正常执行。 2) 隔离性比较差，多个Job任务之间存在资源竞争，互相影响；如果一个Job异常导致TaskManager crash了，那么所有运行在这个TaskManager上的Job任务都会被重启；进而，更坏的情况是，多个Jobs任务的重启，大量并发的访问文件系统，会导致其他服务的不可用；最后一点是，在Rest interface上是可以看到同一个session集群里其他人的Job任务。

## 2.2.  Per Job Cluster模式
顾名思义，这种方式会专门为每个job任务创建一个单独的flink集群，当资源描述文件被提交到kubernetes集群，kubernetes会依次创建FlinkMaster Deployment、TaskManager Deployment并运行任务，任务完成后，这些Deployment会被自动清理。



添加描述

### 2.2.1. 特点分析

优点：隔离性比较好，任务之间资源不冲突，一个任务单独使用一个flink集群；相对于Flink session 集群而且，资源随用随建，任务执行完成后立刻销毁资源，资源利用率会高一些；
缺点：需要提前指定TaskManager的数量，如果TaskManager指定的少了会导致作业运行失败，指定的多了仍会降低资源利用率；资源是实时创建的，用户的作业在被运行前，需要先等待以下过程，
                      1）Kubernetes scheduler为FlinkMaster、TaskManager申请资源并调度到宿主机上进行创建；

                      2）Kubernetes kubelet拉取FlinkMaster、TaskManager镜像，并创建出FlinkMaster、TaskManager容器；

                      3）TaskManager启动后，向Flink ResourceManager注册。

这种模式比较适合对启动时间不敏感、且长时间运行的作业。不适合对任务启动时间比较敏感的场景。

## 2.3.  Native Session Cluster模式

### 2.3.1. 原理分析


添加描述

1）Flink提供了kubernetes模式的入口脚本kubernetes-session.sh，当用户执行了该脚本之后，Flink客户端会生成kubernets资源描述文件，包括FlinkMaster service，FlinkMaster deloyment，configmap，service，

并设置了owner reference，在Flink 1.10版本中，是将FlinkMaster service作为其他资源的owner，也就意味着在删除flink集群的时候，只需要删除FlinkMaster service，其他资源则会被以及联的方式自动删除；

2）kubernetes收到来自Flink的资源描述请求后，开始创建FlinkMaster service，FlinkMaster deloyment，以及configmap资源，从图中可以看到，伴随着FlinkMaster的创建，Dispatch和K8sResMngr组件也

同时被创建了，这里的K8sResMngr就是native方式的核心组件，正是这个组件去和kubernetes API server进行通信，申请TaskManager资源；当前，用户已经可以向flink集群提交任务请求了；

3）用户通过flink client向flink集群提交任务，flink client会生成Job graph，然后和jar包一起上传；当任务提交成功后，JobSubmitHandler收到了请求并提交给Dispatcher，并生成JobMaster，JobMaster用于向KubernetesResourceManager申请task资源；

4）KubernetesResourceManager会为taskmanager生成一个新的配置文件，包含了service的地址，这样当Flink Master 异常重建后，能保证taskmanager通过service仍然能连接到新的Flink Master；

5）TaskManager创建成功后注册到slotManager，这时slotManager向TaskManager申请slots，TaskManager提供自己的空闲slots，任务被部署并运行；

### 2.3.2. 特点分析

之前我们提到的两种部署模式，在kubernetes上运行Flink任务是需要事先指定好TaskManager的数量，但是大部分情况你，用户在任务启动前是无法准确的预知该任务所需的TaskManager数量和规格。

指定的多了会资源浪费，指定的少了会导致任务的执行失败。最根本的原因，就是没有native的使用kubernetes资源，这里的native，可以理解为Flink直接与kuberneter通信来申请资源。

这种类型的集群，也是在提交任务之前就创建好了，不过只包含了FlinkMaster及其Entrypoint（service），当任务提交的时候，Flink client会根据任务计算出并行度，进而确定出所需TaskManager的数量，然后Flink内核会直接向Kubernetes API server申请taskmanager，达到资源动态创建的目的。

优点：相对于前两种集群而言，taskManager的资源是实时的、按需进行的创建，对资源的利用率更高，所需资源更精准。
缺点：taskManager是实时创建的，用户的作业真正运行前，与Per Job集群一样，仍需要先等待taskManager的创建，因此对任务启动时间比较敏感的用户，需要进行一定的权衡。

## 2.4. Native Per Job模式
在当前的Apache Flink1.10版本里，Flink native per-job特性尚未发布，预计在后续的Flink1.11版本中提供，我们可以提前一览native per job的特性。

### 2.4.1. 原理分析


添加描述

当任务被提交后，同样由flink来向kubernetes申请资源，其过程与之前提到的native session模式相似，不同之处在于，

1）Flink Master是随着任务的提交而动态创建的；

2）用户可以将Flink、作业Jar包和classpath依赖打包到自己的镜像里；

3）作业运行图由Flink Master生成，所以无需通过RestClient上传Jar包（图2步骤3）。

### 2.4.2. 特点分析：

native per-job cluster也是任务提交的时候才创建flink集群，不同的是，无需用户指定TaskManager资源的数量，因为同样借助了native的特性，flink直接与kubernetes进行通信并按需申请资源。

优点：资源按需申请，适合一次性任务，任务执行后立即释放资源，保证了资源的利用率；
缺点：资源是在任务提交后开始创建，同样意味着对于提交任务后对延时比较敏感的场景，需要一定的权衡; 

# 3. Flink-operator

### 3.1.1. 简介

分析以上四种部署模式，我们发现，对于Flink集群的使用，往往需要用户自行维护部署脚本，向kubernetes提交各种所需的底层资源描述文件（Flink Master，TaskManager，配置文件，Service）。在session cluster下，如果集群不再使用，还需要用户自行删除这些的资源，因为这类集群的资源使用了Kubernetes的垃圾回收机制owner reference，在删除flink集群的时候，需要通过删除资源的Owner来进行及联删除，这对于不熟悉Kubernetes的Flink用户来说，就显得不是很友好了。而通过Flink-operator，我们可以把Flink集群描述成yaml文件，这样，借助Kubernetes的声明式特性和协调控制器，我们可以直接管理Flink集群及其作业，而无需关注底层资源如Deployment，Service，ConfigMap的创建及维护。当前Flink官方还未给出flink-operator方案，不过GoogleCloudPlatform提供了一种基于kubebuilder构建的flink-operator方案。接下来，将介绍flink-operator的安装方式和对Flink集群的管理示例。

### 3.1.2. Flink-operator原理及优势

当Fink operator部署至Kubernetes集群后，FlinkCluster资源和Flink Controller被创建。其中FlinkCluster用于描述Flink集群，如JobManager规格、TaskManager和TaskSlot数量等；Flink Controller实时处理针对FlinkCluster资源的CRUD操作，用户可以像管理内置Kubernetes资源一样管理Flink集群。例如，用户通过Yaml文件描述期望的Flink集群并向Kubernetes提交，Flink controller分析用户的yaml，得到FlinkCluster CR，然后调用API server创建底层资源，如Jobmanager service，JobManager deployment，TaskManager deployment



添加描述

通过使用Flink Operator，有如下优势：

管理Flink集群更加便捷
flink-operator更便于我们管理flink集群，我们不需要针对不同的Flink集群维护kubenretes底层各种资源的部署脚本，唯一需要的，就是FlinkCluster的一个自定义资源的描述文件。创建一个flink session集群，只需要一条kubectl apply命令即可，下图是Flink Session集群的yaml文件，用户只需要在该文件中声明期望的Flink集群配置，flink-operator会自动完成Flink集群的创建和维护工作。如果创建Per Job集群，也只需要在该Yaml中声明Job的属性，如Job名称，Jar包路径即可。通过flink-operator，上文提到的四种Flink运行模式，分别对应一个Yaml文件即可，非常方便。

apiVersion: flinkoperator.k8s.io/v1beta1kind: FlinkClustermetadata:  name: flinksessioncluster-samplespec:  image:    name: flink:1.10.0    pullPolicy: IfNotPresent  jobManager:    accessScope: Cluster    ports:      ui: 8081    resources:      limits:        memory: "1024Mi"        cpu: "200m"  taskManager:    replicas: 1    resources:      limits:        memory: "2024Mi"        cpu: "200m"    volumes:      - name: cache-volume        emptyDir: {}    volumeMounts:      - mountPath: /cache        name: cache-volume  envVars:    - name: FOO      value: bar  flinkProperties:    taskmanager.numberOfTaskSlots: "1"

声明式
通过执行脚本命令式的创建Flink集群各个底层资源，需要用户保证资源是否依次创建成功，往往伴随着辅助的检查脚本。借助flink operator的控制器模式，用户只需声明所期望的Flink集群的状态，剩下的工作全部由flink operator来保证。在flink集群运行的过程中，如果

出现资源异常，如JobManager意外停止甚至被删除，flink operator都会重建这些资源，自动的修复flink集群。

自定义保存点
用户可以指定autoSavePointSeconds和保存路径，flink operator会自动为用户定期保存快照。

自动恢复
流式任务往往是长期运行的，甚至2-3年不停止都是常见的。在任务执行的过程中，可能会有各种个样的原因导致任务失败。用户可以指定任务重启策略，当指定为FromSavePointOnFailure，flink operator自动从最近的保存点重新执行任务。

sidecar containers
sidecar容器也是kubernetes提供的一种设计模式，用户可以在TaskManager Pod里运行sidecar容器，为Job提供辅助的自定义服务或者代理服务。

Ingress集成
用户可以定义Ingress资源，flink operator将会自动创建ingress资源。云厂商托管的Kubernetes集群一般都有Ingress控制器，否则需要用户自行实现Ingress controller。

Prometheus集成
通过在Flink集群的yaml文件里指定metric exporter和metric port，可以与kubernetes集群中的Prometheus进行集成。

# 4. 最后

通过本文，我们了解了 Flink在Kubernetes上运行的不同模式，其中native模式在资源按需申请方面比较突出，借助kubernetes operator，我们可以将Flink集群当成Kubernetes原生的资源一样进行CRUD操作。限于篇幅，本文主要分析了Flink在Kubernetes上的运行模式的区别，后续将会有更多的文章来对Flink在Kubernetes上的最佳实践进行描述，敬请期待。

参考文档

Kubernetes native integration

https://docs.google.com/document/d/1-jNzqGF6NfZuwVaFICoFQ5HFFXzF5NVIagUZByFMfBY/edit#heading=h.thxqqaj3vxmz

Flink operator使用文档

https://github.com/tkestack/flink-on-k8s-operator/tree/nativePerJob
