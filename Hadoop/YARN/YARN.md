## 1. 概述

### 1.1 ResourceManager 基本职责

1. 与 **客户端**交互：客户端通过RPC协议 **ApplicationClientProtocol** 向 **ResourceManager** 提交应用程序、查询应用程序状态和控制应用程序。
2. 与 **ApplicationMaster** 交互: **ResourceManager** 通过RPC协议 **ApplicationMasterProrocol**, 启动和管理 **ApplicationMaster**， 并在运行失败时重新启动它；
3. 与 **NodeManager** 交互，**NodeManager** 通过RPC协议 **ResourceTracker** 向 **ResourceManager** 注册、汇报节点健康状况和Container运行状态，并领取 **ResourceManager** 下达的命令，这些命令包含初始化、清理Container等。

### 1.2 ResourceManager 内部架构

- **用户交互模块**
  - **ClientRMService**: 为普通用户提供服务， 处理来自客户端的各种RPC请求，比如提交应用程序、终止应用程序、获取应用程序运行状态等。
  - **AdminService**: 为管理员用户提供服务，管理员可以通过这个服务管理集群，比如动态更新节点列表、更新ACL列表、更新队列信息等。
  - **WebApp**: 通过Web界面为用户展示集群资源使用情况和应用程序运行状态等信息。
- **NodeManager 管理模块**
    - **NMLivelinessMonitor**: 监听一个NodeManager是否活着，如果一个NodeManager在一定时间内未汇报心跳信息，则认为它死掉了，需要将其从集群中移除。
    - **NodesListManager**: 维护正常节点和异常节点列表，管理exclude和include节点列表。
    - **ResourceTrackService**: 处理来自NodeManager的请求，主要包括注册和心跳两种请求，其中，注册是NodeManager启动时发生的行为，请求信息中包含节点ID、可用资源上限等信息；而心跳是周期性行为，包含各个Container运行状态，运行的Application列表、节点健康状况等信息，作为请求应答，ResourceTrackService 可为NodeManager返回待释放的Contaniner列表、Application列表等信息。
- **ApplicationMaster 管理模块**
    - **AMLivelinessMonitor**: 监控 ApplicationMaster 是否活着，如果一个 ApplicationMaster 在一定时间内未汇报心跳信息，则认为它死掉了,它上面所有正在运行的 Container 将被设置为失败状态，而 ApplicationMaster 本身会被重新分配到另外一个节点上执行。
    - **ApplicationMasterLauncher**: 与某个 NodeManager 通信，要求它为某个应用程序启动 ApplicationMaster。
    - **ApplicationMasterService**(AMS): 处理来自 ApplicationMaster 的请求，主要包含注册和心跳两种请求，其中，注册是 ApplicationMaster 启动时发生的行为，请求信息中包含 ApplicationMaster 启动节点的对外RPC端口号和tracking URL等信息；而心跳是周期性行为，汇报信息包含所需要的资源描述、待释放的Container列表、黑名单列表等，而AWS则为之返回新分配的Container、失败的Container、待抢占的Container列表等信息。
- **Application 管理模块**
    - **ApplicationACLsManager**: 管理应用程序访问权限，包含两部分权限：查看权限和修改权限。
    - **RMAppManager**：管理应用程序的启动和关闭。
    - **ContainerAllocationExpirer**: 当AM收到RM新分配的一个Container后，必须在一定时间内在对应的NodeManager上启动该Container，否则RM将强制回收该Container，而一个已经分配的Container是否该被收回则由 **ContainerAllocationExpirer** 决定和执行。
- **状态机管理模块**
  - **RMApp**: RMApp 维护了一个 Application 的整个运行周期，包括从启动到运行结束整个过程。由于一个 Application 的生命周期可能会启动多个 Application 运行实例（Application Attempt），因此可认为，RMApp 维护的是同一个 Application 启动的所有运行实例的生命周期。
  - **RMAppAttempt**: 一个 Application 可能启动多个运行实例，即一个运行实例失败后，可能再启动一个重新运行，而每次启动成为一次运行尝试或者运行实例，RMAppAttempt 维护了一次运行尝试的整个生命周期。
  - **RMContainer**: RMContainer维护了一个Container运行周期，包括从创建到运行结束的整个过程。
  - **RMNode**: 维护了一个 NodeManager的生命周期。
- **安全管理模块**： RM 自带了非常全面的权限管理机制，主要由 **ClientToAMSecretManager**、**ContainerTokenSecretManager**、**ApplicationTokenSecretManager** 等模块。
- **资源分配模块**： 此模块主要涉及一个组件—— **ResourceScheduler**，即资源调度器，它按照一定的约束条件（比如队列容量限制等）将集群中的资源分配给各个应用程序，当前主要考虑CPU和内存资源。YARN自带了一个批处理资源调度器——FIFO和两个多用户调度器——Fair Scheduler 和 Capacity Scheduler。

### 1.3 ResourceManager 事件和事件处理器

> YARN 所有服务和组件都是通过中央异步调度器组织在一起的，不同组件之间通过事件交互，从而实现了一个异步并行的高效系统。

## 2. 用户交互模块


## 3. ApplicationMaster 管理

> 管理 ApplicationMaster 步骤：

1. 用户向 YARN ResourceManager 提交应用程序， ResourceManager 收到请求后，先向资源调度器申请用以启动 ApplicationMaster 的资源，待申请到资源后，再由 **ApplicationMasterLauncher** 与对应的NodeManager通信，从而启动应用程序的 ApplicationMaster；
2. ApplicationMaster 启动完成后，**ApplicationMasterLauncher** 会通过事件的形式将刚刚启动的 ApplicationMaster 注册到 **AMLivelinessMonitor**，以启动心跳监控。
3. ApplicationMaster 启动后，先向 **ApplicationMasterService** 注册，并将自己所在的 host、port 等信息汇报给它；
4. ApplicationMaster 运行过程中，周期性地向 **ApplicationMasterService** 汇报“心跳”信息；
5. **ApplicationMasterService** 每次收到 ApplicationMaster 的心跳信息后，将通知 **AMLivelinessMonitor** 更新应用程序的最近汇报心跳时间；
6. 当应用程序运行完成后，ApplicationMaster 向 **ApplicationMasterService** 发送请求，注销自己；
7. **ApplicationMasterService** 收到注册请求后，标注应用程序运行状态为完成，同时通知 **AMLivelinessMonitor** 移除对它的心跳监控。

### 3.1 ApplicationMasterLauncher

> ApplicationMasterLauncher 既是一个服务，也是一个事件处理器，它处理 AMLauncherEvent 事件，该类型事件有两种： LAUNCH、CLEANUP。ApplicationMasterLauncher 维护了一个线程池，从而能够尽快处理这两种事件。

- **LAUCHER** 类型事件: 它会与对应的 NodeManager 通信，要求它启动 ApplicationMaster。
- **CLEANUP** 类型事件：它会与对应的 NodeManager 通信，要求它杀死 ApplocationMaster。

### 3.2 AMLivelinessMonitor

### 3.3 ApplicationMasterService

## 4. NodeManager 管理

## 5. Applicaion 管理

## 6. 状态机管理

> 生命周期管理

###  6.1 RMApp 状态机

> RMApp -> RMAppImpl

> 9种基本状态( RMAppState )：

1. NEW: 
2. NEW_SAVING:
3. SUBMITTED:
4. ACCEPTED:
5. RUNNING:
6. FAILED:
7. KILLED:
8. FINISHING:
9. FINISHED:

> 12种基本事件（RMAppEventType）

1. STARTED:
2. RECOVER:
3. KILL:
4. APP_REJECTED:
5. APP_ACCEPTED:
6. APP_SAVED:
7. ATTEMPT_REGISTERED:
8. ATTEMPT_FINISHING:
9. ATTEMPT_FINISHED:
10. NODE_UPDATE:
11. ATTEMPT_FAILED:
12. ATTEMPT_KILLED:

- 以上事件主要来源于 **ClientRMService** 和 **RMAppAttemptImpl** 两类组件

### 6.2 RMAppAttempt 状态机

> RMAppAttempt --> RMAppAttemptImpl

- 由于在一次运行尝试中，最重要的组件是 **ApplicationMaster**， 它的当前状态可代表整个应用程序的当前运行状态，因此 **RMAppAttemptImpl** 本质上是维护的 ApplicationMaster 生命周期。

> 13 种基本状态 （RMAppAttemptState）

1. NEW:
2. SUBMITTED:
3. SCHEDULED:
4. ALLOCATED_SAVING:
5. ALLOCATED:
6. LAUNCHED:
7. RUNNING:
8. FAILED:
9. KILLED:
10. FINISHED:
11. LAUNCHED_UNMANAGED_SAVING:
12. RECOVERED:

> 15 种基本事件 （RMAppAttemptEvent）

1. START:
2. KILL:
3. APP_ACCEPTED:
4. CONTAINER_ALLOCATED:
5. ATTEMPT_SAVED:
6. LAUNCHED:
7. REGISTERED:
8. UNREGISTERED:
9. CONTAINER_FINISHED:
10. EXPIRE:
11. CONTAINER_ACQUIRED:
12. LAUNCH_FAILED:
13. RECOVER:
14. APP_REJECTED:
15. STATUS_UPDATE:


### 6.3 RMContainer 状态机

> RMContainer --> RMContainerImpl

> 9 个基本状态（RMContainerState）

1. NEW:
2. RESERVED:
3. ALLOCATED:
4. ACQUIRED:
5. RUNNING:
6. RELEASED:
7. COMPLETED:
8. EXPIRED:
9. KILLED:

> 8 个基本事件（RMContainerEventType）

1. START:
2. RESERVED:
3. ACQUIRED:
4. LAUNCHED:
5. FINISHED:
6. RELEASED:
7. KILL:
8. EXPIRE:


### 6.4 RMNode 状态机

> RMNode --> RMNodeImpl

> 6 个基本状态（NodeState）

1. NEW:
2. RUNNING:
3. DECOMMSIONED:
4. UNHEALTHY:
5. LOST:
6. REBOOTED:

> 8 个基本事件（RMNodeEventType）

1. STARTED:
2. STATUS_UPDATE:
3. DECONMMISSION:
4. EXPIRE:
5. REBOOTING:
6. CLEANUP_APP:
7. CLEANUP_CONTAINER:
8. RECONNECTED:

## 7. 几个常见行为分析

- 启动 ApplicationMaster
- 申请和分配 Container
- 杀死 Application
- Container 超时
- ApplicationMaster 超时
- NodeManager 超时

## 8. 安全管理

> Hadoop 2.0 中的认证机制采用 Kerberos 和 Token 两种方案，而授权则是通过引入访问控制列表 （Access Control List， ACL）实现的。

### 8.1 术语介绍

- **Kerberos**：是一种基于第三方服务的认证协议，其特点是用户只需输入一次身份验证信息就可以凭借此验证获得的票据访问多个服务。Kerberos 是一种非常安全的认证协议。
- **Token**：是一种基于共享秘钥的双方身份认证机制，它会被加入到当前的UGI（UserGroupInformation）对象中，并以 Credential 对象的形式加入到 JAAS （Java Authentication and Authentization Service， Java认证和授权服务） Subject 中，当在UGI.doAS上下文中执行RPC函数时，Subject 信息将被推送到线程上下文中。
- **Principal**：Hadoop集群中被认证或授权的主体，包括用户、Hadoop服务、Container、Application、Localizer、Shuffle Data 等。


### 8.2 Hadoop 认证机制

> Hadoop 认证机制同时采用了 Kerberos 和 Token 两种技术，Kerberos 用于用户与服务和服务与服务之间的认证，它是一种基于可信的第三方服务的认证机制，在高并发情况下，效率极低。

- 在Hadoop中，Client与NameNode及Client与ResourceManager之间的初次通信均采用 Kerberos 进行身份认证，之后便换用 Delegation Token以较小开销。而 DataNode 与 NameNode 及 NodeManager 与 ResourceManager 之间的认证始终采用 Kerberos 机制。

### 8.3 Hadoop 授权机制

> ACL

## 9. 容错机制