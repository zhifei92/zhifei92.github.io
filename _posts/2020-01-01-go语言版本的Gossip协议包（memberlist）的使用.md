# 说明
由于工作的契机，最近学习了下Gossip，以及go语言的实现版本HashiCorp/memberlist。网上有个最基本的memberlist使用的example，在下边的链接中，感兴趣可以按照文档运行下感受感受。本文主要讲解memberlist [v0.1.5](https://github.com/hashicorp/memberlist/releases/tag/v0.1.5) 的使用细节。
# Gossip
Gossip是最终一致性协议，有很好的容错性、健壮性。目前Prometheus的告警组件alertmanager、redis、s3、区块链等项目都有使用Gossip。本文不介绍Gossip原理，大家自行谷歌。
# example
简单的几步即可搭建gossip集群
```
1、new个配置文件
c := memberlist.DefaultLocalConfig()
2、创建gossip网络
m, err := memberlist.Create(c)
3、将节点加入到集群
m.Join(parts)
4、实现delegate接口
5、将我们需要同步的数据加到广播队列
broadcasts.QueueBroadcast(userdate)
```
感谢已经有网友为我们实现了一个example（[https://github.com/asim/memberlist](https://github.com/asim/memberlist)
）。
# 配置文件
1. 我们与memberlist交互就是这个Config 配置文件，里面包括基本配置BindAddr 、BindPort ，优化配置GossipInterval、 GossipNodes，代理接口Delegate、EventDelegate。memberlist给了我们三个默认配置，我们不做任何修改，直接创建默认配置即可以简单的跑启一个gossip集群。局域网络的配置DefaultLANConfig，外网DefaultWANConfig，本地DefaultLocalConfig，后两者都是在DefaultLANConfig
基础上修改，区别是根据网速，调整了gossip interval、tcptimeout超时时间等。
2. 接下来主要介绍下Config的各个配置项，以便大家在使用memberlist时，能够知道哪个参数应该修改，哪个代理需要实现。
```
type Config struct {
	// The name of this node. This must be unique in the cluster.
	Name string

	// Transport is a hook for providing custom code to communicate with
	// other nodes. If this is left nil, then memberlist will by default
	// make a NetTransport using BindAddr and BindPort from this structure.
	Transport Transport

	// Configuration related to what address to bind to and ports to
	// listen on. The port is used for both UDP and TCP gossip. It is
	// assumed other nodes are running on this port, but they do not need
	// to.
	BindAddr string
	BindPort int

	// Configuration related to what address to advertise to other
	// cluster members. Used for nat traversal.
	AdvertiseAddr string
	AdvertisePort int

	// ProtocolVersion is the configured protocol version that we
	// will _speak_. This must be between ProtocolVersionMin and
	// ProtocolVersionMax.
	ProtocolVersion uint8

	// TCPTimeout is the timeout for establishing a stream connection with
	// a remote node for a full state sync, and for stream read and write
	// operations. This is a legacy name for backwards compatibility, but
	// should really be called StreamTimeout now that we have generalized
	// the transport.
	TCPTimeout time.Duration

	/* IndirectChecks is the number of nodes that will be asked to perform
	 an indirect probe of a node in the case a direct probe fails. Memberlist
	 waits for an ack from any single indirect node, so increasing this
	 number will increase the likelihood that an indirect probe will succeed
	 at the expense of bandwidth. */
	IndirectChecks int

	// RetransmitMult is the multiplier for the number of retransmissions
	// that are attempted for messages broadcasted over gossip. The actual
	// count of retransmissions is calculated using the formula:
	//
	//   Retransmits = RetransmitMult * log(N+1)
	//
	// This allows the retransmits to scale properly with cluster size. The
	// higher the multiplier, the more likely a failed broadcast is to converge
	// at the expense of increased bandwidth.
	RetransmitMult int

	// SuspicionMult is the multiplier for determining the time an
	// inaccessible node is considered suspect before declaring it dead.
	// The actual timeout is calculated using the formula:
	//
	//   SuspicionTimeout = SuspicionMult * log(N+1) * ProbeInterval
	//
	// This allows the timeout to scale properly with expected propagation
	// delay with a larger cluster size. The higher the multiplier, the longer
	// an inaccessible node is considered part of the cluster before declaring
	// it dead, giving that suspect node more time to refute if it is indeed
	// still alive.
	SuspicionMult int

	// SuspicionMaxTimeoutMult is the multiplier applied to the
	// SuspicionTimeout used as an upper bound on detection time. This max
	// timeout is calculated using the formula:
	//
	// SuspicionMaxTimeout = SuspicionMaxTimeoutMult * SuspicionTimeout
	//
	// If everything is working properly, confirmations from other nodes will
	// accelerate suspicion timers in a manner which will cause the timeout
	// to reach the base SuspicionTimeout before that elapses, so this value
	// will typically only come into play if a node is experiencing issues
	// communicating with other nodes. It should be set to a something fairly
	// large so that a node having problems will have a lot of chances to
	// recover before falsely declaring other nodes as failed, but short
	// enough for a legitimately isolated node to still make progress marking
	// nodes failed in a reasonable amount of time.
	SuspicionMaxTimeoutMult int

	// PushPullInterval is the interval between complete state syncs.
	// Complete state syncs are done with a single node over TCP and are
	// quite expensive relative to standard gossiped messages. Setting this
	// to zero will disable state push/pull syncs completely.
	//
	// Setting this interval lower (more frequent) will increase convergence
	// speeds across larger clusters at the expense of increased bandwidth
	// usage.
	PushPullInterval time.Duration

	// ProbeInterval and ProbeTimeout are used to configure probing
	// behavior for memberlist.
	//
	// ProbeInterval is the interval between random node probes. Setting
	// this lower (more frequent) will cause the memberlist cluster to detect
	// failed nodes more quickly at the expense of increased bandwidth usage.
	//
	// ProbeTimeout is the timeout to wait for an ack from a probed node
	// before assuming it is unhealthy. This should be set to 99-percentile
	// of RTT (round-trip time) on your network.
	ProbeInterval time.Duration
	ProbeTimeout  time.Duration

	// DisableTcpPings will turn off the fallback TCP pings that are attempted
	// if the direct UDP ping fails. These get pipelined along with the
	// indirect UDP pings.
	DisableTcpPings bool

	// AwarenessMaxMultiplier will increase the probe interval if the node
	// becomes aware that it might be degraded and not meeting the soft real
	// time requirements to reliably probe other nodes.
	AwarenessMaxMultiplier int

	// GossipInterval and GossipNodes are used to configure the gossip
	// behavior of memberlist.
	//
	// GossipInterval is the interval between sending messages that need
	// to be gossiped that haven't been able to piggyback on probing messages.
	// If this is set to zero, non-piggyback gossip is disabled. By lowering
	// this value (more frequent) gossip messages are propagated across
	// the cluster more quickly at the expense of increased bandwidth.
	//
	// GossipNodes is the number of random nodes to send gossip messages to
	// per GossipInterval. Increasing this number causes the gossip messages
	// to propagate across the cluster more quickly at the expense of
	// increased bandwidth.
	//
	// GossipToTheDeadTime is the interval after which a node has died that
	// we will still try to gossip to it. This gives it a chance to refute.
	GossipInterval      time.Duration
	GossipNodes         int
	GossipToTheDeadTime time.Duration

	// GossipVerifyIncoming controls whether to enforce encryption for incoming
	// gossip. It is used for upshifting from unencrypted to encrypted gossip on
	// a running cluster.
	GossipVerifyIncoming bool

	// GossipVerifyOutgoing controls whether to enforce encryption for outgoing
	// gossip. It is used for upshifting from unencrypted to encrypted gossip on
	// a running cluster.
	GossipVerifyOutgoing bool

	// EnableCompression is used to control message compression. This can
	// be used to reduce bandwidth usage at the cost of slightly more CPU
	// utilization. This is only available starting at protocol version 1.
	EnableCompression bool

	// SecretKey is used to initialize the primary encryption key in a keyring.
	// The primary encryption key is the only key used to encrypt messages and
	// the first key used while attempting to decrypt messages. Providing a
	// value for this primary key will enable message-level encryption and
	// verification, and automatically install the key onto the keyring.
	// The value should be either 16, 24, or 32 bytes to select AES-128,
	// AES-192, or AES-256.
	SecretKey []byte

	// The keyring holds all of the encryption keys used internally. It is
	// automatically initialized using the SecretKey and SecretKeys values.
	Keyring *Keyring

	// Delegate and Events are delegates for receiving and providing
	// data to memberlist via callback mechanisms. For Delegate, see
	// the Delegate interface. For Events, see the EventDelegate interface.
	//
	// The DelegateProtocolMin/Max are used to guarantee protocol-compatibility
	// for any custom messages that the delegate might do (broadcasts,
	// local/remote state, etc.). If you don't set these, then the protocol
	// versions will just be zero, and version compliance won't be done.
	Delegate                Delegate
	DelegateProtocolVersion uint8
	DelegateProtocolMin     uint8
	DelegateProtocolMax     uint8
	Events                  EventDelegate
	Conflict                ConflictDelegate
	Merge                   MergeDelegate
	Ping                    PingDelegate
	Alive                   AliveDelegate

	// DNSConfigPath points to the system's DNS config file, usually located
	// at /etc/resolv.conf. It can be overridden via config for easier testing.
	DNSConfigPath string

	// LogOutput is the writer where logs should be sent. If this is not
	// set, logging will go to stderr by default. You cannot specify both LogOutput
	// and Logger at the same time.
	LogOutput io.Writer

	// Logger is a custom logger which you provide. If Logger is set, it will use
	// this for the internal logger. If Logger is not set, it will fall back to the
	// behavior for using LogOutput. You cannot specify both LogOutput and Logger
	// at the same time.
	Logger *log.Logger

	// Size of Memberlist's internal channel which handles UDP messages. The
	// size of this determines the size of the queue which Memberlist will keep
	// while UDP messages are handled.
	HandoffQueueDepth int

	// Maximum number of bytes that memberlist will put in a packet (this
	// will be for UDP packets by default with a NetTransport). A safe value
	// for this is typically 1400 bytes (which is the default). However,
	// depending on your network's MTU (Maximum Transmission Unit) you may
	// be able to increase this to get more content into each gossip packet.
	// This is a legacy name for backward compatibility but should really be
	// called PacketBufferSize now that we have generalized the transport.
	UDPBufferSize int

	// DeadNodeReclaimTime controls the time before a dead node's name can be
	// reclaimed by one with a different address or port. By default, this is 0,
	// meaning nodes cannot be reclaimed this way.
	DeadNodeReclaimTime time.Duration
}
```

##### 配置项解释
- Name：节点名字，在集群中必须唯一（查询节点的唯一标识），可以在hostname后加个uuid等。
- Transport：节点间通信的基础服务，包括tcp、udp。不实现这个接口，默认使用memberlist提供的NetTransport。这个基本不用配置，**只有官方提供的功能满足不了你的时候，把源码理解透了，在考虑自定义这个接口**。
- BindAddr、BindPort：这两个很简单，就是gossip peer绑定的地址，新节点可以通过集群中任意一个节点的
BindAddr、BindPort加入到集群中。默认"0.0.0.0"、‘7946’, BindPort如果配置成‘0’，memberlist会动态绑定一个端口。
- AdvertiseAddr、AdvertisePort：这对ip、port是集群其他节点与自己通讯用的，也就是外部可以访问到的ip和端口。上边的BindAddr、BindPort（eg，"0.0.0.0"、‘7946’）是本地监听的ip和端口，但实际本机的出口地址是192.168.1.100:7946，那么AdvertiseAddr应该是192.168.1.100，AdvertisePort是7946，否则其他节点不知道你的真实ip。可能带来误解这个又要根据AdvertiseAddr、AdvertisePort在本地启一个什么服务，其实不是。默认是空，memberlist会解析到节点绑定的ip和port，nat转换后的ip和port。
- ProtocolVersion：协议版本，现在有五个版本，差别不大，例如v3加了tcp  ping ，v4支持间接ping。默认使用v2版本。
- TCPTimeout：建立tcp链接的超时时间，根据网络情况配置即可。
- IndirectChecks：v4协议支持间接探测。当节点启动之后，每个一定的时间间隔，会选取一个节点对其发送一个PING（UDP）消息，当PING消息失败后，会随机选取IndirectChecks个节点发起间接的PING。
- RetransmitMult：广播队列里的消息发送失败超过一定次数后，消息就会被丢弃。RetransmitMult就是用算重传次数的，Retransmits = RetransmitMult * log(N+1)。
- SuspicionMaxTimeoutMult、SuspicionMult：探测某个节点超时后，会将改节点标记为“suspect”节点。节点被标记为“suspect”后，本地启动一个定时器，发出一个“suspect”广播，在一段时间被如果收到其他节点发送过来的“suspect”消息，就将本地的“suspect”确认数加1，当确认数达到要求之后并且该节依旧不是alive状态，会将该节点标记dead。这个时间计算方式如下：
1.SuspicionTimeout = SuspicionMult * log(N+1) * ProbeInterval
2.SuspicionMaxTimeout = SuspicionMaxTimeoutMult * SuspicionTimeout
这两个值一般也不要我们配置，使用默认即可。
- PushPullInterval：每隔PushPullInterval时间间隔，随机选取一个节点，跟它建立tcp连接，然后将本节点的全部状态通过tcp传到对方，对方也把他的状态响应回来，进行状态同步。根据网络情况进行调整即可。
- ProbeInterval、ProbeTimeout：探针间隔和，探测超时时间。根据网络情况进行调整即可。
- DisableTcpPings：关闭tcp ping。这个参数一般也不用管它。
- AwarenessMaxMultiplier：在节点认为自己不能可靠的探测其他节点时，会根据这个参数增加探测间隔。一般使用默认配置即可。
- GossipInterval：检查广播队列是否有数据需要发送给其他节点的时间
- GossipNodes：每次给几个节点扩散数据
- GossipToTheDeadTime：在这个时间内仍然会尝试给Dead状态的节点发送数据。使用默认配置即可
- GossipVerifyIncoming、GossipVerifyOutgoing，SecretKey、Keyring：加密算法和秘钥（SecretKey、Keyring）确定是否对网络上的数据进行加密。GossipVerifyIncoming、GossipVerifyOutgoing为true的作用是如果加密失败则报错，为false表示加密失败则明文传输。
- EnableCompression ：是否数据压缩，默认是true，可以减少贷款。根据需要配置
- DNSConfigPath：指向系统的DNS配置文件，linux就是“/etc/resolv.conf”
- LogOutput、Logger：可以定义memerlist的日志输出方式
- HandoffQueueDepth：广播队列的大小，默认1024，基本足够，不需要改动。
- DeadNodeReclaimTime：如果配置了，加入节点处于dead状态，DeadNodeReclaimTime时间范围内允许相同节点name相同，但ip、port不同的状态对其更新。默认是0，如果发生上述现象则发生冲突。不理解没关系，不配置就可以了。
##### 代理
- Delegate：这个接口一定要实现的，同步业务数据靠的就是这个接口。
1、NotifyMsg([]byte)：每当用户有新数据加到广播队列时，回调此方法通知其他节点同步状态
2、LocalState(join bool) []byte、MergeRemoteState(buf []byte, join bool)：每隔PushPullInterval 周期，本地memberlist回调LocalState方法，把本地全部数据发送到其他节点；其他节点memberlist回调MergeRemoteState，接受数据进行同步。
大家注意下上面两者的区别，一个数新增数据的广播，另一个是通过tcp全量数据同步（加快节点同步状态；加强一致性保障）。

- DelegateProtocolVersion、DelegateProtocolMin、DelegateProtocolMax：业务级别的Delegate版本控制，基本也用不到。什么时候感觉自己需要个这东西，回头在看见就行。
- Events：Events代理，节点加入集群、离开、更新时通知业务层，保存节点状态可以用来链接重连之类的工作。
- Ping：ping代理用来通知业务层，一次ping的ttl，如果需要打点监控的话，可以实现下，否则不需要实现。
- Alive：收到Alive通知后通知业务层，如果需要打点监控的话，可以实现下，否则不需要实现。
- Merge：这个代理感觉没什么用，慢慢发现吧。
# 最后
哪里有问题，还请大家多多指正
# 参考
[https://www.consul.io/docs/internals/gossip.html](https://www.consul.io/docs/internals/gossip.html)
[https://en.wikipedia.org/wiki/Gossip_protocol](https://en.wikipedia.org/wiki/Gossip_protocol)
[https://github.com/asim/memberlist](https://github.com/asim/memberlist)
[https://github.com/hashicorp/memberlist](https://github.com/hashicorp/memberlist)
[https://zhuanlan.zhihu.com/p/41228196](https://zhuanlan.zhihu.com/p/41228196)