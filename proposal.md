# 项目信息

## 项目名称: Sentinel Go Envoy WASM 与 eBPF 扩展
## 方案描述: 

此处项目主要分为 2 大块的需求:
1. 利用 Envoy WASM 扩展机制，结合 Sentinel Go 版本，基于 proxy-wasm-go-sdk 实现 Envoy WASM Sentinel 流控插件，可以支持现有 Sentinel Go 的所有能力，支持通过 Sentinel CRD 标准方式配置规则，并确保 Sentinel 自身性能损耗不会很大;
2. 结合 eBPF 实现 Sentinel 拦截插件，探索在底层进行端口、IP、流量维度的流量控制能 力，并对 Sentinel Go 进行一定的性能优化与轻量化来满足底层控制的需要。


### 需求一： Envoy wasm extension plugin

### 项目背景

由于 sentinel token server for Envoy Global Rate Limiting Service 模型的限制，场景和性能有局限，加上云原生近几年的高速发展，sentinel wasm extension 在社区中的呼声越来越高。利用 Envoy wasm extension，借助 sentinel go 原生的实现，可以实现全面的流量治理能力与标准覆盖。

### 涉及工具

- tinygo 0.24.0 linux/amd64 (using go version go1.18.4 and LLVM version 14.0.0)
- func-e 1.1.3
- kubernetes/kubectl
- Istio
- Envoy

### 架构设计

本次实现的 sentinel wasm extension 是基于 sentinel-go 的版本。所以要用到 proxy-wasm-go-sdk。这个 repo 对 Proxy-Wasm ABI 的一封封装，提供了一系列 Go 语言的 API。使用这个 SDK，可以生成与 Proxy-Wasm 规范兼容的 Wasm 二进制文件。

在讨论 wasm 架构设计之前，要写谈谈 Wasm 虚拟机 (Wasm VM) 。Wasm VM 是加载插件的载体。在 Envoy 中，在每个线程中创建 VM 并且 VM 之间相互隔离。因此，我们创建的程序将被复制到 Envoy 创建的线程中，并加载到每个 VM 上。 Proxy-Wasm 规范中允许在单个 VM 中拥有多个插件。换句话说，一个 VM 可以被多个插件共同使用。

考虑到 sentinel 未来不仅要支持 http filter，有可能还要支持 grpc 的 filter，所以这次 wasm 插件的设计思路是基于 L4 tcp 来实现的。阅读 wasm 的文档，我们可以知道，tcp 层可以做的操作有：tcp 数据帧相关操作、建立连接，断开链接，上行数据处理，下行数据处理。

> 值得提的是，还有一种特性是，Wasm Service 是一种在单例 VM 中运行的插件（即 Envoy 主线程中仅存在一个实例）。它主要用于 filter 并行执行一些额外的工作，例如聚合指标、日志等。有时，这样的单例 VM 本身也称为 Wasm 服务。sentinel 如果有聚合指标的操作，可以借助 Wasm Service 来完成。
> 

![](./pic/wasm_vm.png)

从 vm 加载时机来看，线程先加载 vm，然后 vm 创建 context，然后加载 plugin context。sentinel 插件被 vm 加载，在这个时候，可以加载咱们指定的一些 rule CRD 规范 yaml 文件，进行 sentinel 初始化操作。sentinel plugin 初始操作完成以后，便开始监听 tcp connection 了。

![](./pic/plugin.png)

针对每一个 tcp connection，插件会拦击每一个上行数据包和下行数据包，这个时候存在 5 个 hook 可以执行。在 OnNewConnection 的时候初始化 entry，开启 go 的统计协程。在 OnStreamDone 的时候调用 exit 函数，并输出 BlockError 错误信息。

如果中间出现限流，熔断等操作，在 OnUpstreamData 中拿到 BlockError，并返回这个错误信息给 client。每个 connection 都会触发 entry，如果是正常退出，则会触发 exit，如果是被限流，则返回 BlockError。

### 额外的一些思考

既然每个线程创建了 vm，每个 vm 上有创建了 plugin。那么每个种类的 filter 不一定都要创建在一个 vm 上，可以按照种类不同，创建在多个 vm 上。因为有可能 vm 可承受的 tcp connection 存在一个上限。分散到多个 vm 上，这样可以支持更大的吞吐量。上述只是我的一个猜想，具体行为还需要在生产环境中测试才能确定是否要分 vm。如果分 vm，跨 vm 之间的数据同步和通信是可以实现的。


### 具体代码实现

代码见 [main.go](https://github.com/halfrost/sentinel-go-envoy-proxy-wasm/blob/sentinel-go/main.go)

### 需求二： eBPF Program


### 项目背景

考虑到近几年 eBPF 在云原生上的大力发展，cilium 等开源项目也在大力探索 eBPF，sentinel 社区也期待在 eBPF 上进行探索。期望利用 eBPF 在内核态的能力，进行端口、IP、流量维度的流量控制能力。重点关注以下几点：

- eBPF 里面可以拿到哪些信息，做哪些维度的流量治理
- 不解包的情况下能拿到哪些信息；部分解包（低成本）的情况下能拿到哪些信息
- 如果需要解包的话，成本如何



### 涉及工具

- cilium/ebpf
- BCC
- bpftrace
- Ubuntu 22.04.1 LTS (Jammy Jellyfish)


### 架构设计

先分析 Linux 接收网络包的执行过程，可以分析出以下的处理顺序。

![](./pic/ebpf1.png)

从接收网络包开始，对端将网络包发到本机的网卡，然后经过网络驱动，流程控制以后，进入到网络协议栈，这里还有一层 NetFilter。TCP 和 UDP 包分别被内核处理。内核处理完以后会交给上层 socket，最后经过 system call 交给指定的应用。在这个处理路径中，我发现可以在 2 个地方编写 eBPF 程序，一个是 位于网络驱动层的 XDP，另外一个是进入到了内核中的 TC 程序。

选择 XDP 的原因是，这时候网络包还没有进入内核中，在网络驱动中直接处理，可以节省内核处理网络包的时间，节省系统资源，节约 cpu 的时间片。在这里可以直接处理 qps 数等基本计数类观测型参数。选择 TC 的原因是，这时网络包进入了内核，不需要我们手写解包的复杂代码，内核的网络协议栈帮我们处理好了解包步骤。所以在这里进行一些流量控制比较方便。

确定了 eBPF 的 2 个 hook 观测点以后，接下来考虑如何把程序注入到内核中。通过查阅文档，可以知道如下的 eBPF 加载过程：

![](./pic/ebpf3.png)


先用 eBPF 工具开发出源代码，然后使用 BCC 工具，将代码编译成 BPF bytecode，并注入到内核态中。通过内核态的验证以后，eBPF 代码就会开始运行。假设我们需要观察 TCP 链接总数，那么内核态的程序需要把 TCP 链接总数聚合，存入到 map 中。用户态从这个 map 中读取 tcp 链接总数。这个处理流程对应下图中的 1，2，3 async read，这些处理流程。

![](./pic/ebpf2.png)


### 具体代码实现

代码见 [main.go](https://github.com/halfrost/sentinel-go-envoy-proxy-wasm/tree/sentinel-go/ebpf/ebpf-kernel)



## 项目开发时间计划:
1. 详细设计文档 7/1 – 7/20
2. 代码 7/20-9/10
3. 开发 Envoy WASM Extension 7/20 – 8/20
4. 开发 eBPF 程序 8/20-9/20
4. 完整的测试用例和测试代码 9/20-9/30






# 项目进度 

### 需求一： Envoy wasm extension plugin

### 待解决的问题

#### 1. tinygo 存在限制

鉴于 tinygo 的限制，不能完美支持 cgo。这意味着我不能使用 etcd 和 prometheus 相关功能。因为它们俩会依赖 xxhash。xxhash 需要 cgo 编译。于是我注释掉 etcd 和 prometheus 以及它们关联的依赖。注掉以后，再编译确实可以通过。但是 sentinel 会出 bug。调试以后发现，在 exporter/metric/prometheus/exporter.go:20:2 这一行中，如果没有 prometheus client 依赖包，metric 是 0，这会导致 sentinel 底层统计计数一部分为 0，这会直接导致 sentinel 永远不会触发流控逻辑。我这两天一直在折腾这部分，还是没调试出来。如果有 sentinel-golang 开发同学，我很想问问他，如果移除 prometheus，有什么办法能 workaround。这部分我暂时想不到其他方法可以绕开。

在 wasm plugin 中，常见的统计计数都很容易实现，但是涉及到 prometheus 中一些底层的 metric，目前会受到 tinygo 的限制，导致编译错误。我在 Stack Overflow，github 上也寻找过解决方案，看了一些比较复杂的 wasm plugin，tinygo 的版本还没有人写出来和 prometheus 有关的功能。关于 cgo 的问题，已经提 issue 给 tinygo 团队了。他们目前正在修复中，issue 在这里：[https://github.com/tinygo-org/tinygo/issues/3044](https://github.com/tinygo-org/tinygo/issues/3044)

#### 2. reflect 的问题

除去 cgo 的问题，还有一些关于 reflect 的问题。可能你会好奇，哪里会用到 reflect？其实在读取 yaml 文件的时候就会用到。在 yaml 文件中用 map 传递一些值给 plugin，比如 resource name，crd path。这些是包在一个 json 中。tinygo 解析的时候，并不知道里面有哪些字段，它会以 interface{} 类型去解析。这就会用到 reflect 了。在现在的实现代码中写了读取 crd yaml 的代码了。但是实际运行会崩溃。错误和这个 issue 是一样的：[https://github.com/tinygo-org/tinygo/issues/2660](https://github.com/tinygo-org/tinygo/issues/2660)。这个问题有 workaround 的方法，即传值不要传递一个可变结构的 map。这样 tinygo 解析的时候不会用 interface 去序列化，也不会触发这个 reflect 的 bug。但是这个 bug 真的很常见。如果传一个 json 数据，就 panic 了。json 是一个可变结构的 map。

### 当前进度

基于上述的一些原因。我还没有完整的把 sentinel wasm 跑起来。因为目前涉及到 prometheus，暂时还没想到 workaround 方法。我目前测试的 case 是，把我写好的 wasm 放到 envoy 上运行。然后在这台机器上 curl 对应的接口。触发 wasm tcp filter，进而可以测试 tcp filter 的逻辑。我觉得重头还是测试 sentinel 在 wasm plugin 上的逻辑。


### 需求二： eBPF Program


### 待解决的问题

用户态如何利用 map 中的数据，并提供给 sentinel 使用。考虑获取 qps 的场景，例如：

```go
func (s *Slot) OnEntryPassed(ctx *base.EntryContext) {
	s.recordPassFor(ctx.StatNode, ctx.Input.BatchCount)
	if ctx.Resource.FlowType() == base.Inbound {
		s.recordPassFor(InboundNode(), ctx.Input.BatchCount)
	}

	handledCounter.Add(float64(ctx.Input.BatchCount), ctx.Resource.Name(), ResultPass, "")
}
```

在通过 inbound 流量的时候，高并发情况下记录链接总数这块逻辑都可以用 eBPF 内核态程序处理。

```go
func (m *SlidingWindowMetric) GetQPS(event base.MetricEvent) float64 {
	return m.getQPSWithTime(util.CurrentTimeMillis(), event)
}

func (m *SlidingWindowMetric) getSatisfiedBuckets(now uint64) []*BucketWrap {
	start, end := m.getBucketStartRange(now)
	// Extracts the buckets of which the startTime is between [start, end]
	// which means the time view of the buckets is [firstStart, endStart+bucketLength)
	satisfiedBuckets := m.real.ValuesConditional(now, func(ws uint64) bool {
		return ws >= start && ws <= end
	})
	return satisfiedBuckets
}
```

在计算 qps 的时候，使用了链接总数，这时不使用自己维护的链接总数，在用户态中读取内核态 map 中的个数获取链接总数。上述代码和内部实现的 sliding window 类强绑定。这个类并不是只用来统计 qps，它经过封装以后，现在是统计所有用户关心的 metric。如果我只想实现 qps 从内核态 map 中读取，个别函数需要针对 qps 单独改造。这块改造还有点问题。


### 当前进度

实现了 eBPF 在 XDP 上检测 IPv4 网络链接，并讲总数写入到内核的 map 中。但是用户态如何使用这个数据，还有点问题。用户态监听这个 map 的数据的时机是否要依赖内核态程序启动完成，还有待考虑。如果将内核态的代码和用户态的代码封装成在一个模块中，是最完美的做法。这样启动先后关系就不需要额外考虑了。


### 未来的一些规划


关于 eBPF feature 的长远规划，在我阅读了 cilium 的源码之后，给了我很多启示，相比于人家的工程型的代码，我的代码显得有点简陋，主要有 2 点：

1. 内核代码和用户态代码。我现在把 2 个代码分开了，是 2 个独立的程序。我这样做的目的是，内核态代码可以单独部署，用户态代码跟随者 sentinel 代码一起运行在用户态。内核态代码出 bug 了，可以热更新，单独发布更新。我看了几个开源的大型的 ebpf 项目，大家的做法是写成一个程序，但是 ebpf 是其中单独的一个模块。我以 cilium 举例，它内部有 10 几个 ebpf，对应 5 个不同的内核 hook 挂载点。这 10 段 ebpf 代码是通过 cilium daemon 统一生成字节码的，然后由这个 daemon 注册到 5 个系统挂载点中。我很赞同这种做法。一个项目就一个程序，多个模块。这才是成熟的工程型的项目。我重构的第一步也会考虑做成这样。

2. 内核态的程序可以减少依赖，不依赖 cilium/ebpf 模块，通过 system call 的方式把数据存储到 map 中。

![](./pic/ebpf4.png)

3. 现在我的代码没有取消 ebpf 挂载的逻辑。没有这些代码，我的 feature 没法上线吧？需要增加一个开关，可以让用户控制打开和关闭 ebpf 功能，热插拔 ebpf 这个功能。也可以加上降级的策略。出问题以后可以降级到原来的统计逻辑上。之前统计的指标是从 Prometheus 上拿的数据。

当完成了 1 的功能以后，有一个模块可以动态分发内核态代码，以后新增新类型 ebpf 程序就都是这个模块内的小功能了。可以动态分发新的 ebpf 代码。


# 心得与后续工作安排

首先非常感谢主办方举办这次编程之夏的活动。让我接触到了 sentinel 这么优秀的项目。我也认识的 2 位非常专业的导师。在我迷惑的时候给与我帮助。谢谢大家。

关于后续工作安排，即使活动结束了，我依旧会继续开发，将上述遇到的问题解决，我很喜欢 sentinel 社区，也很看好 sentinel 在 Cloud Native 中 service mesh 的未来发展。我会继续为 sentinel 贡献代码，希望以上 2 个功能能尽快做到完美，最终能上线到生产环境。




