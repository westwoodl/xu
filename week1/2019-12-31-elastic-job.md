# Elastic-Job

星期二 🌤



- 分布式调度协调    分片
- 弹性扩容缩容
- 失效转移
- 错过执行作业重触发
- 作业分片一致性，保证同一分片在分布式环境中仅一个执行实例
- 自诊断并修复分布式不稳定造成的问题
- 支持并行调度
- 支持作业生命周期操作
- 丰富的作业类型
- Spring整合以及命名空间提供
- 运维平台



## 核心理念
### 1. 分布式调度
Elastic-Job-Lite并无作业调度中心节点，而是基于部署作业框架的程序在到达相应时间点时各自触发调度。

注册中心仅用于作业注册和监控信息存储。而主作业节点仅用于处理分片和清理等功能。

### 2. 作业高可用
Elastic-Job-Lite提供最安全的方式执行作业。将分片总数设置为1，并使用多于1台的服务器执行作业，作业将会以1主n从的方式执行。

一旦执行作业的服务器崩溃，等待执行的服务器将会在下次作业启动时替补执行。开启失效转移功能效果更好，可以保证在本次作业执行时崩溃，备机立即启动替补执行。

### 3. 最大限度利用资源
Elastic-Job-Lite也提供最灵活的方式，最大限度的提高执行作业的吞吐量。将分片项设置为大于服务器的数量，最好是大于服务器倍数的数量，作业将会合理的利用分布式资源，动态的分配分片项。

例如：3台服务器，分成10片，则分片项分配结果为服务器A=0,1,2;服务器B=3,4,5;服务器C=6,7,8,9。 如果服务器C崩溃，则分片项分配结果为服务器A=0,1,2,3,4;服务器B=5,6,7,8,9。在不丢失分片项的情况下，最大限度的利用现有资源提高吞吐量。



## 整体架构图

![my-logo.png](https://img-blog.csdnimg.cn/2019021916501732.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxOTI0ODYyMDc3,size_16,color_FFFFFF,t_70 "架构图")


## 实现原理
### 弹性分布式实现

- 第一台服务器上线触发主服务器选举。主服务器一旦下线，则重新触发选举，选举过程中阻塞，只有主服务器选举完成，才会执行其他任务。
- 某作业服务器上线时会自动将服务器信息注册到注册中心，下线时会自动更新服务器状态。
- 主节点选举，服务器上下线，分片总数变更均更新重新分片标记。
- 定时任务触发时，如需重新分片，则通过主服务器分片，分片过程中阻塞，分片结束后才可执行任务。如分片过程中主服务器下线，则先选举主服务器，再分片。
- 通过上一项说明可知，为了维持作业运行时的稳定性，运行过程中只会标记分片状态，不会重新分片。分片仅可能发生在下次任务触发前。
- 每次分片都会按服务器IP排序，保证分片结果不会产生较大波动。
- 实现失效转移功能，在某台服务器执行完毕后主动抓取未分配的分片，并且在某台服务器下线后主动寻找可用的服务器执行任务。



### 注册中心数据结构
注册中心在定义的命名空间下，创建作业名称节点，用于区分不同作业，所以作业一旦创建则不能修改作业名称，如果修改名称将视为新的作业。作业名称节点下又包含4个数据子节点，分别是config, instances, sharding, servers和leader。

#### config节点
作业配置信息，以JSON格式存储

#### instances节点
作业运行实例信息，子节点是当前作业运行实例的主键。作业运行实例主键由作业运行服务器的IP地址和PID构成。作业运行实例主键均为临时节点，当作业实例上线时注册，下线时自动清理。注册中心监控这些节点的变化来协调分布式作业的分片以及高可用。 可在作业运行实例节点写入TRIGGER表示该实例立即执行一次。

#### sharding节点
作业分片信息，子节点是分片项序号，从零开始，至分片总数减一。分片项序号的子节点存储详细信息。每个分片项下的子节点用于控制和记录分片运行状态。节点详细信息说明：


子节点名| 临时节点| 描述
:-|:-:|:-:
instance|否|执行该分片项的作业运行实例主键
running|是|分片项正在运行的状态仅配置monitorExecution时有效
failover|是|如果该分片项被失效转移分配给其他作业服务器，则此节点值记录执行此分片的作业服务器IP
misfire|否|是否开启错过任务重新执行
disabled|否|是否禁用此分片项

#### servers节点
作业服务器信息，子节点是作业服务器的IP地址。可在IP地址节点写入DISABLED表示该服务器禁用。 在新的cloud native架构下，servers节点大幅弱化，仅包含控制服务器是否可以禁用这一功能。为了更加纯粹的实现job核心，servers功能未来可能删除，控制服务器是否禁用的能力应该下放至自动化部署系统。

#### leader节点
作业服务器主节点信息，分为election，sharding和failover三个子节点。分别用于主节点选举，分片和失效转移处理。

leader节点是内部使用的节点，如果对作业框架原理不感兴趣，可不关注此节点。

子节点名|临时节点|描述
:-|:-|:-
election\instance|是|主节点服务器IP地址一旦该节点被删除将会触发重新选举重新选举的过程中一切主节点相关的操作都将阻塞
election\latch|否|主节点选举的分布式锁为curator的分布式锁使用
sharding\necessary|否|是否需要重新分片的标记 如果分片总数变化，或作业服务器节点上下线或启用/禁用，以及主节点选举，会触发设置重分片标记作业在下次执行时使用主节点重新分片，且中间不会被打断作业执行时不会触发分片
sharding\processing|是|主节点在分片时持有的节点如果有此节点，所有的作业执行都将阻塞，直至分片结束主节点分片结束或主节点崩溃会删除此临时节点
failover\items\分片项|否|一旦有作业崩溃，则会向此节点记录当有空闲作业服务器时，会从此节点抓取需失效转移的作业项
failover\items\latch|否|分配失效转移分片项时占用的分布式锁为curator的分布式锁使用

zk相关注册分片信息示例：



### 流程图
#### 作业启动


![jagtu.png](https://img-blog.csdnimg.cn/20190219165201636.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxOTI0ODYyMDc3,size_16,color_FFFFFF,t_70 " 流程图")


#### 作业执行
![](https://img-blog.csdnimg.cn/20190219165306269.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxOTI0ODYyMDc3,size_16,color_FFFFFF,t_70 "")

### 作业分片策略
#### 框架提供的分片策略
##### AverageAllocationJobShardingStrategy
全路径：

com.dangdang.ddframe.job.lite.api.strategy.impl.AverageAllocationJobShardingStrategy

策略说明：

基于平均分配算法的分片策略，也是默认的分片策略。

如果分片不能整除，则不能整除的多余分片将依次追加到序号小的服务器。如：

如果有3台服务器，分成9片，则每台服务器分到的分片是：1=[0,1,2], 2=[3,4,5], 3=[6,7,8]

如果有3台服务器，分成8片，则每台服务器分到的分片是：1=[0,1,6], 2=[2,3,7], 3=[4,5]

如果有3台服务器，分成10片，则每台服务器分到的分片是：1=[0,1,2,9], 2=[3,4,5], 3=[6,7,8]

###### OdevitySortByNameJobShardingStrategy
全路径：

com.dangdang.ddframe.job.lite.api.strategy.impl.OdevitySortByNameJobShardingStrategy

策略说明：

根据作业名的哈希值奇偶数决定IP升降序算法的分片策略。

作业名的哈希值为奇数则IP升序。

作业名的哈希值为偶数则IP降序。

用于不同的作业平均分配负载至不同的服务器。

AverageAllocationJobShardingStrategy的缺点是，一旦分片数小于作业服务器数，作业将永远分配至IP地址靠前的服务器，导致IP地址靠后的服务器空闲。而OdevitySortByNameJobShardingStrategy则可以根据作业名称重新分配服务器负载。如：

如果有3台服务器，分成2片，作业名称的哈希值为奇数，则每台服务器分到的分片是：1=[0], 2=[1], 3=[]

如果有3台服务器，分成2片，作业名称的哈希值为偶数，则每台服务器分到的分片是：3=[0], 2=[1], 1=[]

###### RotateServerByNameJobShardingStrategy
全路径：

com.dangdang.ddframe.job.lite.api.strategy.impl.RotateServerByNameJobShardingStrategy

策略说明：

根据作业名的哈希值对服务器列表进行轮转的分片策略。

#### 自定义分片策略
实现JobShardingStrategy接口并实现sharding方法，接口方法参数为作业服务器IP列表和分片策略选项，分片策略选项包括作业名称，分片总数以及分片序列号和个性化参数对照表，可以根据需求定制化自己的分片策略。

欢迎将分片策略以插件的形式贡献至com.dangdang.ddframe.job.lite.api.strategy包。

#### 配置分片策略
与配置通常的作业属性相同，在spring命名空间或者JobConfiguration中配置jobShardingStrategyClass属性，属性值是作业分片策略类的全路径。