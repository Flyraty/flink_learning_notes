### Flink 基础架构
Flink 作为分布式的数据处理引擎，需要高效的资源管理分配方式用以支撑内部流式作业的运行。它集成了常见的资源管理系统，比如 Yarn，Mesos，k8s，也支持以 standalone 的方式运行。本节用来概述 Flink 的架构，介绍 Flink 是如何与客户端应用程序交互的，遇到错误失败时是如何恢复程序的。

### Flink 集群剖析
Flink 运行时主要包括两种进程，JobManager 和 TaskManager，TaskManager 可以启动多个。
<div align=center><img src="https://ci.apache.org/projects/flink/flink-docs-release-1.13/fig/processes.svg" width="70%" height="70%"></div>

客户端并不在程序运行时存在，我们只是通过他向 JM 提交一份 job graph。在这之后，如果是 *detached mode* 的话，客户端会退出，*attached mode* 的话，客户端会继续和 JM 保持连接获取程序的进度信息。客户端可以作为用户程序的一部分用来触发任务执行，也可以是 `./bin/flink run ...` 这个种方式。

JM 和 TM 以不同的方式启动，如果是 standalone 模式，则直接在机器上启动。如果是基于 Yarn，则是直接交由对应的资源管理器进行启动。TM 需要与 JM 保持连接，通知 JM 自己可用，然后由 JM 注册分配 task 到可用的 TM 上。

### JobManager 

*JobManager* 负责协调管理 Flink 任务的执行，比如：何时开始调度任务执行、对已经完成或者失败的任务作出反应、协调 checkpoint 的生成、管理任务的 failover 等等。它主要由以下三个组件组成

- **ResourceManager**

	*ResourceManager* 负责 Flink 集群资源的分配 - 管理 Flink 最小的资源调度单位 **task slots**。Flink 针对不同的资源管理器提供了对应的 ResourceManager 的实现。在 standalone 模式下，Flink 不能自动创建新的 TaskManager。

- **Dispatcher**  
	
	*Dispatcher* 提供了一个 REST 接口用来提交 Flink 作业，每次提交都会生成一个 JobMaster。另外，其还运行着 Flink Web UI。

- **JobMaster**
	
	*JobMaster* 负责管理单个 [Job Graph](https://ci.apache.org/projects/flink/flink-docs-release-1.13/docs/concepts/glossary/#logical-graph) 的运行。 Flink 集群中可以同时运行多个 job，但是每个 job 都有自己单独的 JobMaster。

通常情况下只需要启动一个 JobManager 就 ok，如果要做高可用的话，可以查看 [High Availability (HA)](https://ci.apache.org/projects/flink/flink-docs-release-1.13/docs/deployment/ha/overview/)

### TaskManagers

*TaskManager* 用来实际运行 task，并缓存和交换数据（比如 shuffle 过程中的落盘）。程序运行必须存在一个 TaskManager，TaskManager 中资源调度的最小单位是 slot，它决定了该 TaskManager 上最多可以并行多少 task。需要注意多个 operator 可以运行在一个 subtask 上。

### Tasks and Operator Chains
Flink 支持链接多个 operator 到一个 subtask（也可以说是一个 slot）中运行。每个 task 单独一个线程执行，多个 operator 链接到一起可以提高程序处理的性能，减少各个线程之间的数据交换及各种 buffer 开销。可以选择是否开启 operator chain。
下图展示了 operator chains，source 和 map 链接到了一起，最终的执行是有 5 个 subtask。

<div align=center><img src="https://ci.apache.org/projects/flink/flink-docs-release-1.13/fig/tasks_chains.svg" width="70%" height="70%"></div>

### Task Slots and Resources
每个 TaskManager 都是一个 JVM 进程，内部以多线程的方式执行一个或者多个 subtask。该如何控制 TaskManager 上执行的任务数呢？可以通过 task slot 来设置，其最小为 1。

每个 *task slot* 都代表 TaskManager 资源的一个子集。比如，一个 TM 有三个 slot，则每个 slot 会占用 TM 总资源的 1/3。通过 slot 分配资源意味着不同作业之间 subtask 的运行时互不影响，拥有独立的驻留内存等资源。需要注意这里并不是 CPU 隔离的，slot 只是占有当前 TM 进程的内存资源。

通过指定一定数量的 task slot，用户可以自己决定 subtask 的运行隔离。每个 TM 有一个 slot 代表所有的 subtask 都在不同的 JVM 进程之中运行。每个 TM 拥有多个 slot 代表其中的 subtask 在一个 JVM 进程中。同一进程中的 task 共享 TCP 连接和心跳信息，也可以共享一些数据集，从而减少每个任务的开销。
<div align=center><img src="https://ci.apache.org/projects/flink/flink-docs-release-1.13/fig/tasks_slots.svg" width="70%" height="70%"></div>

默认情况下，Flink 允许不同 task 下的 subtask 共享一个 slot，只要他们属于同一个 Job。这样做的结果可能会造成一个 slot 处理了整个 Job 的 task。设计成这样主要有两点好处  
- Flink 值需要提供跟最大并行度一样的 slot 即可，而不需要计算每个 task 的并行度。
- 刚好的资源利用率，如果没有共享 slot，资源使用比较少的 source/map task 就会独占整个 slot，直至其运行完成，其中剩余的资源都无法被使用。

### Flink 程序的运行
Flink 程序通过 main 方法入口生成一个或者多个作业提交到集群。这些 job 可以在本地 JVM 进程（LocalEnvironment）或者拥有多个机器的远程集群上运行（RemoteEnvironment）。对于每一个程序，ExecutionEnvironment 提供了控制作业执行的方法（比如设置并行度，开启 checkpoint 等）。
Flink 程序可以用 session mode，per job 的方式运行，下面介绍这些运行方式的生命周期及资源管理

#### Flink Session Cluster
- **集群生命周期**：在 session 模式下，Flink 客户端通过连接一个预先启动，长期运行的集群来提交作业。即使提交的作业已经运行完成，集群（包括 JM）仍然会运行，直到整个 session 被手动停止。Flink Session Cluster 的生命周期与上面运行的作业无关。

- **资源隔离**：在作业提交后，RM 负责分配 task slots，在作业运行完成后就会释放响应资源。因为所有作业共享整个集群，所以存在资源竞争 - 比如提交作业的网络带宽。这种共享的一个限制是一旦 TM 崩溃，在其上面运行的 subtask 都会失败。同样的，一旦 JM 挂掉，整个集群的作业都会失败。

- **其他注意事项**：session 模式下，减小了作业的启停开销，主要是 TM 的启动和资源的申请，这对于启动时间大于运行时间的作业是友好的，场景大概在小任务比较多的情况下使用比较好。 

#### Flink Job Cluster
- **集群生命周期**：在 per-job 模式下，会为每个提交的任务单独启动一个集群供作业单独使用。客户端首先从集群管理器申请资源启动 JM，然后向 Dispatcher 提交作业，然后根据作业的资源信息，启动 TM 进程。一旦该作业完成，整个集群也会被销毁。

- **资源隔离**：JM 产生错误只会影响当前的作业。

- **其他注意事项**：因为 RM 需要申请外部资源并等待管理组件启动 TM，per-job 模式适合运行时间较长，要求高可用性的大型作业。生产环境中通常是这种模式

> K8s 不支持 per-job 模式。


#### Flink Application Cluster 
- **集群生命周期**：该模式用来专门运行 Flink 作业，提交作业也在集群上提交而不是客户端。作业提交是一步过程，不需要先启动 JM 在提交。取而代之的是已 jar 包的形式，*ApplicationClusterEntryPoint* 负责执行 main 方法将作业先转换成 job graph。Flink Application Cluster 的生命周期与 Flink 程序本身相关。

- **资源隔离**：在 Flink Application Cluster 中，RM 和 Dispatcher 作用于单个 Flink 程序，提供了比 session mode 下更好的资源隔离，比 per-job 模式下更少的启动时间。

> per-job 模式可以看做运行在客户端上的 Flink Application Cluster 。


### 思考  

**1.从 job 提交来看 Yarn，Flink，Spark？**


**2.Flink 常见资源配置参数？**


**3.咋一看，Flink Application Cluster 是作业提交运行最佳的方式，是这样吗？**