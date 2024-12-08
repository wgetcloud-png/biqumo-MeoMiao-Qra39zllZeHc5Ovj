
## 一、 **生命周期钩子**


Kubernetes 中的 **生命周期钩子（Lifecycle Hooks）** 是在容器生命周期的特定阶段执行操作的机制。通过钩子，可以在容器启动后（PostStart）或停止前（PreStop）执行一些初始化或清理工作。


#### **钩子的作用**


1. **PostStart（启动后）**
	* 在容器启动后立即触发执行。
	* 用于完成启动后的初始化操作，例如加载配置、启动辅助进程。
2. **PreStop（停止前）**
	* 在容器收到终止信号（如 `SIGTERM`）时触发执行。
	* 用于执行停止前的清理工作，例如保存状态、关闭连接、释放资源。


#### **钩子的实现方式**


钩子支持以下两种执行方式：


* **`exec`：** 直接在容器内部运行指定命令。
* **`httpGet`：** 通过 HTTP GET 请求调用一个端点。


钩子定义的基本结构：



```
lifecycle:
  postStart:
    exec:
      command: ["sh", "-c", "echo 'Container started'"]
  preStop:
    exec:
      command: ["sh", "-c", "echo 'Container stopping'; sleep 5"]

```

就比如下面的例子：



```
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-demo
spec:
  containers:
    - name: nginx
      image: nginx:1.21.1
      lifecycle:
        postStart:
          exec:
            command:
              - "/bin/sh"
              - "-c"
              - |
                echo "PostStart hook triggered! Initializing..."; 
                sleep 3
        preStop:
          exec:
            command:
              - "/bin/sh"
              - "-c"
              - |
                echo "PreStop hook triggered! Cleaning up..."; 
                sleep 5
      ports:
        - containerPort: 80
      resources:
        requests:
          memory: "64Mi"
          cpu: "250m"

```

#### **工作原理与解释**


1. **PostStart：**
	* 容器启动后，`PostStart` 钩子立即执行。
	* 在示例中，`PostStart` 钩子通过 `echo` 输出一条日志并等待 3 秒。
2. **PreStop：**
	* 当容器收到终止信号（如 `kubectl delete pod` 或 `kubectl scale`）时，`PreStop` 钩子立即执行。
	* 在示例中，`PreStop` 钩子输出一条日志并等待 5 秒，模拟资源清理。


#### **注意事项**


1. **执行时间限制：**
	* 钩子执行时间受 Pod 的 `terminationGracePeriodSeconds` 控制，默认宽限时间为 30 秒。如果 `PreStop` 未在宽限时间内完成，容器将被强制终止。
2. **运行环境：**
	* 钩子在容器的文件系统和环境变量中运行，因此可以直接访问容器内部的资源。
3. **错误处理：**
	* 如果钩子失败，容器不会因此失败，但会记录错误日志。
4. **钩子与主进程的关系：**
	* 钩子运行与容器主进程无直接依赖关系。PostStart 并不阻塞主进程启动，而 PreStop 是在终止信号发送后触发。


通过合理利用 Kubernetes 钩子，可以在容器生命周期的不同阶段完成初始化和清理操作，从而提升应用的可靠性和自动化水平。


## 二、探针


Kubernetes 中的**探针（Probes）**用于检测容器的健康状态和服务状态。通过探针，Kubernetes 可以决定容器是否能够接受流量或者是否需要重启，从而提高应用的可用性和可靠性。


#### **探针的类型**


1. **就绪探针（Readiness Probe）**
	* 用于判断容器是否已经准备好接受请求。
	* 如果探针检查失败，Kubernetes 会将该 Pod 从服务的负载均衡中移除。
2. **存活探针（Liveness Probe）**
	* 用于判断容器是否处于健康状态。
	* 如果探针检查失败，Kubernetes 会重新启动容器。
3. **启动探针（Startup Probe）**
	* 用于判断容器的启动是否完成。
	* 启动探针成功后，Kubernetes 不会再调用它，而是转而使用其他探针（如存活探针）。


#### **1\. HTTP 请求探针（httpGet）**


通过向容器的特定 HTTP 端点发送请求检测健康状态：



```
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10

```

#### **2\. 命令探针（exec）**


运行容器内部的命令，并根据退出码判断健康状态。


* 返回值为 `0` 表示成功，非 `0` 表示失败：



```
livenessProbe:
  exec:
    command:
      - cat
      - /tmp/healthy
  initialDelaySeconds: 5
  periodSeconds: 10

```

#### **3\. TCP 探针（tcpSocket）**


检查容器指定端口的连通性：



```
livenessProbe:
  tcpSocket:
    port: 3306
  initialDelaySeconds: 5
  periodSeconds: 10

```



| **特性** | **探针（Probes）** | **钩子（Hooks）** |
| --- | --- | --- |
| **作用** | 持续检查容器的健康状态和服务状态 | 在容器生命周期的特定时间点执行一次性操作 |
| **触发时机** | 周期性运行，贯穿容器生命周期 | 容器启动后或终止前的一次性操作 |
| **结果影响** | 可能触发容器重启或移除服务负载均衡 | 不直接影响容器状态，只执行指定逻辑 |
| **常见用途** | 健康检查、故障恢复、负载均衡 | 初始化操作（`postStart`）或清理任务（`preStop`） |


## 三、K8S常用控制器


​ 在 Kubernetes 中，使用 **控制器** 而不是直接使用 **Pod** 是因为控制器提供了更强的自动化管理和可靠性。控制器（如 **ReplicaSet**、**Deployment** 等）可以自动管理 Pod 的副本，确保应用始终有正确数量的 Pod 运行。当 Pod 失败或被删除时，控制器会自动替代，保证服务的高可用性。而直接使用 Pod 无法做到这一点，必须手动监控和管理。


​ 控制器支持 **滚动更新** 和 **回滚**，使得应用更新时可以逐个更新 Pod，避免中断服务并能在出现问题时快速回滚。控制器还支持 **自动扩缩容**，根据负载自动调整 Pod 数量，这对于应对动态流量非常重要。


控制器提供了 **自愈能力**，当 Pod 失败时，控制器会自动重启或替换，减少人工干预。同时，控制器能够集成与其他 Kubernetes 资源（如服务、存储和网络策略）无缝工作，实现更高效的资源管理。控制器的 **声明式管理** 可以简化配置和操作，避免手动干预 Pod 的创建和管理，使得 Kubernetes 集群的管理更加灵活和自动化。因此，使用控制器管理 Pod 能提供更高的可靠性、自动化和灵活性。


在 Kubernetes 中，**ReplicaSet**、**Deployment**、**StatefulSet**、**DaemonSet**、**Job** 和 **CronJob** 是常见的工作负载资源类型，它们用于管理不同场景下的 Pod 生命周期。




| 资源类型 | 是否有状态 | 适用场景 | 特点 |
| --- | --- | --- | --- |
| ReplicaSet | 无状态 | 简单副本管理 | 确保 Pod 副本数量一致，但功能较基础 |
| Deployment | 无状态 | 动态扩缩容、版本管理 | 支持滚动更新、回滚等高级功能 |
| StatefulSet | 有状态 | 有状态服务（数据库、消息队列） | 保持 Pod 的固定身份和持久存储 |
| DaemonSet | 无状态 | 节点级任务（监控、日志收集） | 确保每节点运行一个 Pod |
| Job | 无状态 | 一次性任务（数据迁移、备份） | 任务完成后 Pod 停止运行 |
| CronJob | 无状态 | 定时任务（备份、清理） | 按时间表定期创建 Job |


注：下面的代码案例会引入上文提到的钩子和探针。


#### 1\.ReplicaSet 控制器


ReplicaSet（下面简称RS） ，用于确保指定数量的 Pod 副本始终在集群中运行，可以提供简单的无状态服务，用作 Deployment 的底层组件。一般情况下，RS控制器能做的事情，Deployment控制器都能做，而且Deployment比RS功能更多，更高级。我的建议是可以拿RS来写yaml文件提升自己对控制器的理解。


#### **特点：**


* 维护一组 Pod 的副本，确保这些 Pod 的数量始终与定义的副本数一致。
* 如果某个 Pod 意外终止，ReplicaSet 会自动创建新的 Pod 来补足。


#### **主要字段：**


* `replicas`：期望的副本数量。
* `selector`：选择器，用于匹配需要管理的 Pod 标签。
* `template`：Pod 模板，定义 Pod 的属性。


下面提供一个用RS部署nginx的yaml文件案例：



```
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: example-replicaset
  labels:
    app: rs-nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      initContainers:
        - name: init-nginx
          image: nginx:1.21.1
          imagePullPolicy: IfNotPresent
          command:
            - "/bin/bash"
            - "-c"
            - "echo 'container init already!!'; sleep 10"
      containers:
        - name: my-container
          image: nginx:1.21.1
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
          command:
            - "/bin/bash"
            - "-c"
            - "sleep 300"
          lifecycle: 					# 生命周期钩子
            postStart:
              exec:
                command:
                  - "/bin/sh"
                  - "-c"
                  - "echo 'Container has started';"
            preStop:
              exec:
                command:
                  - "/bin/sh"
                  - "-c"
                  - "echo 'Container is stopping'; sleep 5;"
          livenessProbe:                ##### 存活探针
            httpGet:
              path: /healthz            # 探针检查的路径
              port: 80                  # 探针检查的端口
            initialDelaySeconds: 5      # 首次检查延迟时间
            periodSeconds: 10           # 检查间隔时间
            timeoutSeconds: 1           # 每次检查的超时时间
            failureThreshold: 3         # 连续失败的次数后重启容器
          readinessProbe:               ##### 就绪探针
            httpGet:
              path: /ready              # 探针检查的路径
              port: 80                  # 探针检查的端口
            initialDelaySeconds: 3      # 首次检查延迟时间
            periodSeconds: 5            # 检查间隔时间
            timeoutSeconds: 1           # 每次检查的超时时间
            failureThreshold: 2         # 连续失败的次数后移出服务流量

```

​ 这个YAML 配置定义了一个 Kubernetes ReplicaSet，管理 3 个 Nginx 容器副本，确保服务的高可用性。它包含一个初始化容器，在主容器启动前执行初始化任务。主容器暴露端口 80，并配置了生命周期钩子：`postStart` 在容器启动后执行，`preStop` 在容器停止前执行。还设置了存活探针和就绪探针，分别用于监控容器健康和是否准备好接受流量，探针配置包括检查路径、延迟时间、检查间隔及失败阈值，确保容器在故障时自动重启或移除流量。


#### 2\.Deployment 控制器


Deployment 是用于声明式管理应用部署的高级控制器，底层依赖于 ReplicaSet。对比与RS控制器，Deployment控制器增加了更新策略以及扩缩容等内容。


**特点：**


* 支持滚动更新和回滚，确保服务的高可用性。
* 自动创建和管理 ReplicaSet。
* 能够轻松扩缩容和版本升级。


**主要字段：**


* `strategy`：更新策略（如滚动更新）。
* `replicas`：Pod 副本数。
* `revisionHistoryLimit`：保留的历史版本数量。


下面提供一个用Deployment部署nginx的yaml文件案例：



```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      initContainers:
        - name: init-nginx
          image: nginx:1.21.1
          imagePullPolicy: IfNotPresent
          command:
            - "/bin/bash"
            - "-c"
            - "echo 'container init already!!'; sleep 10"
      containers:
        - name: my-container
          image: nginx:1.21.1
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
          command:
            - "/bin/bash"
            - "-c"
            - "sleep 300"
          lifecycle:                   # 生命周期钩子
            postStart:
              exec:
                command:
                  - "/bin/sh"
                  - "-c"
                  - "echo 'Container has started';"
            preStop:
              exec:
                command:
                  - "/bin/sh"
                  - "-c"
                  - "echo 'Container is stopping'; sleep 5;"
          livenessProbe:                ##### 存活探针
            httpGet:
              path: /healthz            # 探针检查的路径
              port: 80                  # 探针检查的端口
            initialDelaySeconds: 5      # 首次检查延迟时间
            periodSeconds: 10           # 检查间隔时间
            timeoutSeconds: 1           # 每次检查的超时时间
            failureThreshold: 3         # 连续失败的次数后重启容器
          readinessProbe:               ##### 就绪探针
            httpGet:
              path: /ready              # 探针检查的路径
              port: 80                  # 探针检查的端口
            initialDelaySeconds: 3      # 首次检查延迟时间
            periodSeconds: 5            # 检查间隔时间
            timeoutSeconds: 1           # 每次检查的超时时间
            failureThreshold: 2         # 连续失败的次数后移出服务流量

```

#### 3\.StatefulSet控制器


StatefulSet 专为有状态应用设计，提供稳定的网络标识、持久化存储和有序部署/删除。与前两个我们介绍的控制器不同的是，Statefulset控制器更适合用于部署需要数据长期存储的Pod或者固定网络标识的资源。


**特点：**


* 每个 Pod 都有一个固定的名字和序号（如 `my-app-0`, `my-app-1`）。
* 支持稳定的存储（与 PersistentVolume 结合使用）。
* 提供有序的 Pod 启动、更新和终止。


**适用场景：**


* 数据库（如 MySQL、MongoDB）。
* 消息队列（如 Kafka）。
* 有状态应用（需要持久存储或固定网络标识）。


下面提供一个用Deployment部署nginx的yaml文件案例（通过上面两个控制器，我们已经了解了探针和钩子的原理和使用方法，接下来展示的yaml案例将不再加入探针和钩子的内容，方便我们更能直观的观察各个控制器的区别！）：



```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: example-statefulset
spec:
  serviceName: "example"
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-container
        image: nginx
        volumeMounts:
        - name: data
          mountPath: /data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi

```

#### 4\.DaemonSet控制器


DaemonSet 确保每个节点（或符合条件的节点）上都运行一个 Pod。这个控制器非常适用于监控或者是网络插件的部署，但是需要注意在Master主节点上，默认会有一个NoSchedule的污点，需要在yaml添加相对应的容忍度。


可以通过 \*\*kubectl describe node  \*\*进行查看，如下所示：



```
[root@master ~]# kubectl describe node master
Name:               master
Roles:              control-plane
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=master
                    kubernetes.io/os=linux
                    node-role.kubernetes.io/control-plane=
                    node.kubernetes.io/exclude-from-external-load-balancers=
Annotations:        kubeadm.alpha.kubernetes.io/cri-socket: unix:///run/containerd/containerd.sock
                    node.alpha.kubernetes.io/ttl: 0
                    projectcalico.org/IPv4Address: 192.168.116.131/24
                    projectcalico.org/IPv4IPIPTunnelAddr: 10.251.205.128
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Fri, 04 Oct 2024 21:33:23 +0800
Taints:             node-role.kubernetes.io/control-plane:NoSchedule
Unschedulable:      false
Lease:
  HolderIdentity:  master
  AcquireTime:     
  RenewTime:       Sat, 07 Dec 2024 20:47:33 +0800
Conditions:
  Type                 Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----                 ------  -----------------                 ------------------                ------                       -------
  NetworkUnavailable   False   Tue, 12 Nov 2024 17:22:18 +0800   Tue, 12 Nov 2024 17:22:18 +0800   CalicoIsUp                   Calico is running on this node
  MemoryPressure       False   Tue, 12 Nov 2024 23:29:12 +0800   Sat, 05 Oct 2024 00:28:44 +0800   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure         False   Tue, 12 Nov 2024 23:29:12 +0800   Sat, 05 Oct 2024 00:28:44 +0800   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure          False   Tue, 12 Nov 2024 23:29:12 +0800   Sat, 05 Oct 2024 00:28:44 +0800   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready                True    Tue, 12 Nov 2024 23:29:12 +0800   Sat, 05 Oct 2024 14:45:43 +0800   KubeletReady                 kubelet is posting ready status
Addresses:
  InternalIP:  192.168.116.131
  Hostname:    master
Capacity:
  cpu:                4
  ephemeral-storage:  40940Mi
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             3793428Ki
  pods:               110
Allocatable:
  cpu:                4
  ephemeral-storage:  38635831233
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             3691028Ki
  pods:               110
System Info:
  Machine ID:                 062407b38be842d2ba33e2ad841f7bfb
  System UUID:                bd814d56-4024-2228-d6f1-0b011fa4eb3e
  Boot ID:                    6946bb88-fd9a-47f6-8d8d-e9a544e7a50f
  Kernel Version:             4.18.0-425.3.1.el8.x86_64
  OS Image:                   Rocky Linux 8.7 (Green Obsidian)
  Operating System:           linux
  Architecture:               amd64
  Container Runtime Version:  containerd://1.6.32
  Kubelet Version:            v1.28.2
  Kube-Proxy Version:         v1.28.2
PodCIDR:                      10.240.0.0/24
PodCIDRs:                     10.240.0.0/24

```

可以清晰的看到在Taint部分，污点级别是NoSchedule：



> \[!NOTE]
> 
> 
> Taints: node\-role.kubernetes.io/control\-plane:NoSchedule


#### **特点：**


* 每个节点上仅运行一个 Pod。
* 新节点加入集群时，DaemonSet 会自动为其启动 Pod。


#### **适用场景：**


* 日志收集器（如 Fluentd）。
* 节点监控代理（如 Prometheus Node Exporter）。
* 网络组件（如 Calico、Weave Net）。



```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: example-daemonset
spec:
  selector:
    matchLabels:
      app: my-daemon
  template:
    metadata:
      labels:
        app: my-daemon
    spec:
      containers:
      - name: my-container
        image: nginx

```

#### 5\.Job控制器


Job 是专门设计来管理需要在一定时间内完成的单次任务的，如批处理作业、数据库迁移等。


**特点：**


* 一旦任务完成，Pod 将停止运行。
* 可以控制任务的并发数量和重试策略。


**适用场景：**


* 数据处理任务。
* 数据迁移或备份任务。
* 批处理作业。


**主要字段：**


* `completions`：任务完成所需成功 Pod 数量。
* `parallelism`：允许同时运行的 Pod 数量。



```
apiVersion: batch/v1
kind: Job
metadata:
  name: job-example
spec:
  parallelism: 1  # 并发执行的 Pod 数量
  completions: 1  # 任务需要完成的 Pod 数量
  template:
    spec:
      containers:
      - name: busybox
        image: busybox
        command: ["sh", "-c", "echo 'Task Completed'; sleep 5"]
      restartPolicy: OnFailure  # Pod 失败时重新启动

```

#### 6\.CronJob控制器


**CronJob** 是 Kubernetes 中 **Job** 的扩展，用于定期执行任务。它的作用类似于 Linux 系统中的 `cron`，可以指定任务的执行时间表，自动定期触发并执行 Job。


**特点：**


* 支持基于时间表（cron 表达式）的定时任务。
* 自动创建和管理 Job。


**适用场景：**


* 定期备份任务。
* 周期性报告生成。
* 定时清理任务。


**主要字段：**


* `schedule`：cron 表达式定义执行时间。
* `jobTemplate`：Job 模板，定义具体任务。



```
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cronjob-example
spec:
  schedule: "*/5 * * * *"  # Cron 表达式，表示每 5 分钟执行一次
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: busybox
            image: busybox
            command: ["sh", "-c", "echo 'Cron job executed'; sleep 5"]
          restartPolicy: OnFailure  # Pod 失败时重新启动

```

 本博客参考[slower加速器](https://jisuanqi.org)。转载请注明出处！
