# P2P 网络构建

对于 go-libp2p 的不同模块，可以参考 [长安链P2P网络技术介绍（2）：初识LibP2P](https://cloud.tencent.com/developer/article/1988253)[^1]。当然，这是一个很笼统的介绍，而这篇博文也会比较简单（毕竟它所拥有的功能太多了），只会记录所使用到的一些功能：连接建立、流处理、节点发现、消息广播及配置、资源限制；同时，官方的库中也有一些样例可以参考：[examples - go-libp2p](https://github.com/libp2p/go-libp2p/tree/master/examples)，这篇博文的完整代码可以在 [libp2p-example](https://github.com/Decision2016/libp2p-example) 查看

## 连接建立

先从最简单的 echo 功能开始：两个节点 A 和 B 在建立连接后，节点 A 向节点 B 发送消息 msg，节点 B 也返回消息 msg，可以通过控制台发送消息和打印接收到的消息，这也是 example 中的第一个例子 [echo - go-libp2p](https://github.com/libp2p/go-libp2p/tree/master/examples/echo)

### 节点启动

在 go-libp2p 中可以使用 `func New(opts ...Option) (host.Host, error)` 启动一个节点，它会返回一个 `Host` 对象，通过这个对象可以查询到本地节点的一系列连接信息，也是实现其他功能所需要用到的对象[^2]。在这里先只考虑使用它来获取这个节点的监听地址。

```go
package main

import (
	"github.com/libp2p/go-libp2p"
	"github.com/sirupsen/logrus"
)

func main() {
	host, err := libp2p.New()

	if err != nil {
		logrus.WithError(err).Errorln("Create libp2p host failed.")
		return
	}

	logrus.Infoln("Create new libp2p host, listen addresses: ", host.Addrs())
	if err := host.Close(); err != nil {
		logrus.WithError(err).Errorln("Close host errored.")
	}
}
```

这段代码展示的是一个节点的创建，并且在创建后打印出节点的监听信息

### 节点连接

在连接到另外一个节点之前需要知道节点的连接信息，通常连接信息以 multiaddr[^3] 的形式出现，它的格式通常是这样：

```
/ip4/127.0.0.1/tcp/31514/p2p/12D3KooWSc8cftoRYqPdKUg3CSfdHngvbMuc8b4xEBBkF66RFGV8

/[ip 协议类型/ip 地址/传输协议类型/端口/节点 id
```

可以在指定监听端口后直接打印节点信息，并且让其他节点连接，例如添加以下的日志输出语句，并且通过传参的方式传入 `port`

```go
logrus.Infof("Node address: /ip4/127.0.0.1/tcp/%v/p2p/%s", port, host.ID().String())
```

在拥有节点的信息后，就可以通过 `Connect(ctx context.Context, pi peer.AddrInfo) error` 连接到对端节点，对应的代码如下

```go
ctx := context.Background()
addrStr := "/ip4/127.0.0.1/tcp/31514/p2p/..."

maddr, err := multiaddr.NewMultiaddr(addrStr)
if err != nil {
    logrus.WithError(err).Errorln("Convert address from string failed.")
    return
}

addr, err := peer.AddrInfoFromP2pAddr(maddr)
if err != nil {
    logrus.WithError(err).Errorln("Get address info from multiple address failed.")
    return
}
err = host.Connect(ctx, *addr)
if err != nil {
    logrus.WithError(err).Errorln("Connect to new node failed.")
    return
}
```

结合建立节点的代码，引入传参后可以实现一个连接到对端节点的程序，在传入连接信息 `peer` 时可以连接到对应的节点

```go
func main() {
	flag.Parse()

	if help {
		flag.Usage()
		return
	}

	localMultiAddr, err := multiaddr.NewMultiaddr(
		fmt.Sprintf("/ip4/0.0.0.0/tcp/%d", port),
	)
	host, err := libp2p.New(
		libp2p.ListenAddrs(localMultiAddr),
	)

	if err != nil {
		logrus.WithError(err).Errorln("Create libp2p host failed.")
		return
	}

	logrus.Infoln("Create new libp2p host, listen addresses: ", host.Addrs())
	logrus.Infof("Node address: /ip4/127.0.0.1/tcp/%v/p2p/%s", port, host.ID().String())

	if peerAddr != "" {
		ctx := context.Background()

		maddr, err := multiaddr.NewMultiaddr(peerAddr)
		if err != nil {
			logrus.WithError(err).Errorln("Convert address from string failed.")
			return
		}

		addr, err := peer.AddrInfoFromP2pAddr(maddr)
		if err != nil {
			logrus.WithError(err).Errorln("Get address info from multiple address failed.")
			return
		}
		err = host.Connect(ctx, *addr)
		if err != nil {
			logrus.WithError(err).Errorln("Connect to new node failed.")
			return
		}

		logrus.Infof("Connect to node %s success.", peerAddr)
	}

	c := make(chan os.Signal)
	select {
	case sign := <-c:
		logrus.Infof("Got %s signal. Aborting...", sign)

		if err := host.Close(); err != nil {
			logrus.WithError(err).Errorln("Close host errored.")
		}
	}
}
```

> **传参** 利用 golang 的 flag 包可以接收、解析传入参数
>
> ```golang
> var (
> 	peerAddr string
> 	port     int
> 	help     bool
> )
> 
> func init() {
> 	flag.StringVar(&peerAddr, "peer", "", "Peer node address")
> 	flag.IntVar(&port, "port", 8123, "Node listen port")
> 	flag.BoolVar(&help, "h", false, "Command help")
> 
> 	flag.Usage = usage
> }
> 
> func usage() {
> 	fmt.Fprintf(os.Stderr, `Usage: main [--port port] [--peer address]
> 
> Options:
> `)
> 	flag.PrintDefaults()
> }
> ```

### 数据流

在完成节点的连接后，还需要实现消息的传输才能完成 echo 程序的功能。在 go-libp2p 中具有创建数据流的功能

* 发起流的建立者需要通过 `NewStream(ctx context.Context, p peer.ID, pids ...protocol.ID) (network.Stream, error)` 发起流的建立
* 对端需要通过 `SetStreamHandler(pid protocol.ID, handler network.StreamHandler)` 设置处理流的函数，并且由这个 `handler` 来进行流的处理

####  流处理函数

在这个例子中，接受流的创建的那一方在接受到消息后在控制台通过日志打印，并且将数据重新写入流返回给发送方，其函数如下

```go
func handleStream(s network.Stream) {
	for {
		buf := bufio.NewReader(s)
		str, err := buf.ReadString('\n')
		if err != nil {
			logrus.WithError(err).Errorln("Receive failed, stream routine exit.")
			break
		}

		logrus.Infof("Read from stream: %s", str[:len(str)-1])
		_, err = s.Write([]byte(str))
		if err != nil {
			logrus.WithError(err).Errorln("Write to stream failed, routine exit.")
		}
	}
}
```

同时，在 `main`  函数中使用 `host.SetStreamHandler("/echo/1.0.0", handleStream)` 来设置流的处理函数

在这里 `/echo/1.0.0` 是指定的协议 id，可以根据不同的需要来进行设定

####  发送方函数

`NewStream` 函数在成功创建流之后会返回一个 `Stream` 对象，后续可以通过这个对象来收发数据，基于此可以编写一个 `sendMessage` 的函数作为控制台消息写入协程

```golang
func sendMessage(s network.Stream) {
	var msg string
	buf := bufio.NewReader(s)
	for {
		_, err := fmt.Scan(&msg)
		if err != nil {
			logrus.WithError(err).Errorln("Read data from console failed.")
			break
		}
		msg += "\n"

		_, err = s.Write([]byte(msg))
		if err != nil {
			logrus.WithError(err).Errorln("Write message to stream failed.")
			break
		}

		str, err := buf.ReadString('\n')
		logrus.Infof("Read from stream: %s", str[:len(str)-1])
	}
}
```

并且，在 `main` 函数中连接到对端节点后添加

```golang
		stream, err := host.NewStream(ctx, addr.ID, "/echo/1.0.0")
		if err != nil {
			logrus.WithError(err).Errorln("Create stream failed.")
			return
		}
		go sendMessage(stream)
```

对应于上一段代码，此时的程序在省略掉部分代码后如下所示

```go
func main() {
	...
    
	logrus.Infoln("Create new libp2p host, listen addresses: ", host.Addrs())
	logrus.Infof("Node address: /ip4/127.0.0.1/tcp/%v/p2p/%s", port, host.ID().String())

	if peerAddr != "" {
        ...
		logrus.Infof("Connect to node %s success.", peerAddr)

		stream, err := host.NewStream(ctx, addr.ID, "/echo/1.0.0")
		if err != nil {
			logrus.WithError(err).Errorln("Create stream failed.")
			return
		}
		go sendMessage(stream)
	}
    ...
}

func sendMessage(s network.Stream) {...}

func handleStream(s network.Stream) {...}
```

至此，实现了一个基于 go-libp2p 完成的节点间通信程序。而需要建立一个 P2P 网络所需要的则是连接到一个节点后能够发现其他节点并且进行通信，以及指定节点 id 来进行通信。所以，下一节的内容则是利用 kademlia 库来进行节点的发现和连接。

## 节点发现

在 go-libp2p 中实现了多种节点发现协议：Rendezvous、mDNS 和 DHT[^5]，在这里，使用 DHT 来进行节点的发现，对应 go-libp2p 的子模块库是 go-libp2p-kad-dht。对于 Kademlia 协议，可以参考 [Kademlia 协议及广播方法](https://decision01.com/2022/09/27/Kademlia%E5%8D%8F%E8%AE%AE%E5%8F%8A%E5%B9%BF%E6%92%AD%E6%96%B9%E6%B3%95/) (没写完，可以看看参考文章)此外， go-libp2p-kad-dht 的使用还可以在本文之外参考 [Building an echo application with libp2p](https://ldej.nl/post/building-an-echo-application-with-libp2p/)[^4]

有了节点连接建立的基础后，节点的发现和连接就较为简单了（代码量不大，麻烦的是查阅资料）

### KDHT 实例创建

使用 go-libp2p-kad-dht 需要创建一个 kdht 实例，在完成 host 的创建后可以传入 `host` 来建立实例，并且调用 `Bootstrap` 来启动路由的维护，以下是创建 kdht 实例的代码，传入的参数 `bootstrapPeers` 是启动节点数组，分两种情况考虑：

* 空数组：没有启动节点，自身作为一个启动节点
* 非空数组：启动节点列表，依次遍历所有的节点地址并且尝试连接，在建立连接后可以发现其他节点

```go
func NewKDHT(ctx context.Context, host host.Host, bootstrapPeers []multiaddr.Multiaddr) (*dht.IpfsDHT, error) {
	var options []dht.Option
	// Start with server mode
	options = append(options, dht.Mode(dht.ModeServer))

	// Create new kad-dht instance
	kdht, err := dht.New(ctx, host, options...)
	if err != nil {
		logrus.WithError(err).Errorln("Create new kad-dht instance failed.")
		return nil, err
	}

	// Bootstrap the kad-dht instance
	if err = kdht.Bootstrap(ctx); err != nil {
		logrus.WithError(err).Errorln("Bootstrap kad-dht failed.")
		return nil, err
	}
	
	for _, peerAddr := range bootstrapPeers {
		info, _ := peer.AddrInfoFromP2pAddr(peerAddr)
		if err := host.Connect(ctx, *info); err != nil {
			logrus.WithError(err).Errorf("Errored while connecting to node #%s", *info)
			continue
		} else {
			logrus.Infoln("Connection established with bootstrap node: %q", *info)
		}
	}

	return kdht, nil
}
```

### 路由表查询

在完成 kad-dht 实例的创建后，可以通过它来进行路由表的查询以发现其他节点

这个过程的初始化需要以下对象：

* context：上下文，以便同步关闭协程
* host：创建的节点实例，用于发起流的创建
* dht：创建的 kad-dht 实例，用于进行路由表中其他节点的发现
* rendezvous：节点发现”密钥“，被称为”集合点”，节点们通过这个值来查找到对等的节点

```go
func Discover(ctx context.Context, h host.Host, dht *dht.IpfsDHT, rendezvous string) {
    // Create new discovery object
	var routingDiscovery = routing.NewRoutingDiscovery(dht)
	dutil.Advertise(ctx, routingDiscovery, rendezvous)
	
    // Refresh per 500ms
	ticker := time.NewTicker(500 * time.Millisecond)
	defer ticker.Stop()

	for {
		select {
		case <-ctx.Done():
			return
		case <-ticker.C:
            // Refresh routing table
			dht.RefreshRoutingTable()
            // Find other peers
			peers, err := routingDiscovery.FindPeers(ctx, rendezvous)

			if err != nil {
				logrus.WithField("error", err).Debugln("Find peers failed.")
				continue
			}
			for p := range peers {
                // Check whether is self host
				if p.ID == h.ID() {
					continue
				}
				
                // Check connected
				if h.Network().Connectedness(p.ID) != network.Connected {
					_, err := h.Network().DialPeer(ctx, p.ID)
					if err != nil {
						logrus.WithFields(logrus.Fields{
							"peerID": p.ID,
							"error":  err,
						}).Debugln("Connect to node failed.")
						continue
					}
				}
                
				stream, err := h.NewStream(ctx, p.ID, "/echo/1.0.0")
				if err != nil {
					logrus.WithError(err).Errorln("Create new stream failed.")
				}
				// ... some function with stream
			}
		}
	}
}
```

在较老的版本中，rendezvous 是通过 `discovery.Advertise(ctx, routingDiscovery, rendezvous)` 来“宣布”的，但是在目前的版本下貌似不能生效，需要使用 `dutil.Advertise(ctx, routingDiscovery, rendezvous)`

而最为关键的也是 ticker 的处理函数，`dht.RefreshRoutingTable()` 刷新路由表，然后使用 `FindPeers` 函数在“集合点”发现其他节点；它会返回一个 具有一系列 Peer 的 channel，遍历这个 channel 可以获取到对端节点的信息

并且在得到节点信息后需要根据节点 id 是否为自身，否则会和本地的节点建立流，以及查询是否以及建立了连接，以便在建立数据流之前建立连接。在完成流的建立后，可以基于这个流来进行节点间的通信。要实现一个可以指定节点 id 发送消息的应用，可以添加其他逻辑代码后实现。

## 消息广播

消息广播对应的库是 [go-libp2p-pubsub](https://github.com/libp2p/go-libp2p-pubsub)，目前所实现的广播路由方式有：1、Gossip 协议[^6]；2、 Flood 洪泛协议[^7]；3、 随机路由；同时，go-libp2p 的文档中也写了发布/订阅模式的相关原理[^8]，但是没有涉及到具体的使用；

消息的广播和订阅涉及到四个函数:

* `NewGossipSub`：创建一个 Gossip 广播对象，go-libp2p 底层会创建流，gossip 所使用的的是其底层建立的流，且创建的对象是一个单例
* `Join`：加入一个 topic，并返回一个 `Topic` 对象，调用其 `Subscribe` 函数进行订阅
* `Subscribe`：订阅 topic，返回 `Subscription` 对象，用于后续接收其他节点广播的消息
* `Publish`：广播消息到某个 topic，向订阅了该 topic 的所有节点发送消息

**注：go-libp2p-pubsub 默认的消息限制为 1M，所以大于 1M 的消息无法广播到其他节点会被直接丢弃**

前三个函数在具体使用时如下所示

```go
goosip, err := pubsub.NewGossipSub(ctx, h)
if err != nil {
    logrus.WithError(err).Errorln("Create new gossip subscribe failed.")
    return
}

topic, err := goosip.Join("/gossip/1.0.0")
if err != nil {
    logrus.WithError(err).Errorln("Join topic /gossip/1.0.0 failed.")
    return
}

sub, err := topic.Subscribe()
if err != nil {
    logrus.WithError(err).Errorln("Subscribe topic failed.")
    return
}
```

并且，sub 对象需要一个协程来专门进行处理，协程可以从中取出消息并且进行处理，这个函数通常实现为下面的样子，持续取出消息并且判断是否是自身发出的广播消息，如果是则丢弃，否则进行处理

```go
func handleSub(ctx context.Context, sub *pubsub.Subscription, h host.Host) {
	for {
		msg, err := sub.Next(ctx)
		if err != nil {
			logrus.WithError(err).Errorln("Get data from subscription failed.")
			continue
		}

		if msg.ReceivedFrom == h.ID() {
			continue
		}

		// ... do something with msg.Data
	}
}
```

最后，还需要使用 `Publish` 进行消息的广播，这也较为简单，可以编写一个函数从 channel 中读出消息并且广播到网络中，例如

```go
func broadcastMessage(msgChan chan string, topic *pubsub.Topic) {
	for {
		select {
		case msg := <-msgChan:
			err := topic.Publish(context.Background(), []byte(msg))
			if err != nil {
				logrus.WithError(err).Errorln("Publish message failed.")
				continue
			}
		}
	}
}
```

go-libp2p-pubsub 在发送时会将它打包到一个定义好的 proto 结构体中，序列化后发送，所以在 `handleSub` 中还需要通过 `msg.Data` 获取字节数据

## 节点配置

最后这里放一些使用时会遇到的配置问题，go-libp2p 中会有许多的自定义配置，但是在文档中都没有一一列举，只能自行查阅代码

###  网络资源限制

资源管理器可以用来限制节点对流的创建、连接的创建所消耗的网络资源，这里需要一个函数来创建一个资源管理器

```go
func buildResourceManager() *network.ResourceManager {
	scalingLimits := rcmgr.DefaultLimits
	libp2p.SetDefaultServiceLimits(&scalingLimits)
	scaledDefaultLimits := scalingLimits.AutoScale()

	cfg := rcmgr.PartialLimitConfig{
		System: rcmgr.ResourceLimits{
			//Conns: 20,
			Streams: 50,
		},
	}

	limits := cfg.Build(scaledDefaultLimits)
	limiter := rcmgr.NewFixedLimiter(limits)
	rm, err := rcmgr.NewResourceManager(limiter)

	if err != nil {
		log.Errorln("Build resource manager failed.")
		return nil
	}
	return &rm
}
```

上面的代码里限制了流的最大数量为50，在节点启动前可以创建这样一个资源管理器，并且引入到节点配置中：

```go
host, err := libp2p.New(
    libp2p.ResourceManager(*rm),
)
```

更多的配置需要参阅：[The libp2p Network Resource Manager](https://pkg.go.dev/github.com/srene/go-libp2p/p2p/host/resource-manager)

###  广播调整

正如前面所说，go-libp2p-pubsub 中限制了最大广播数据大小为 1M，在某些场景下是不够的，所以需要进行调整

这需要在创建 Gossip 对象的时候引入配置，例如

```go
const pubsubMaxSize = 1 << 22 // 4 MB

gossip, err = pubsub.NewGossipSub(ctx, h,
    pubsub.WithMaxMessageSize(pubsubMaxSize),
)
```

通过这样的代码可以将广播调整为 4M 内进行广播

## 参考文章

[^1]: [长安链P2P网络技术介绍（2）：初识LibP2P](https://cloud.tencent.com/developer/article/1988253)
[^2]: [从 go-libp2p 开始](https://www.cnblogs.com/superblockchain/p/14103338.html)
[^3]: [Addressing in libp2p](https://github.com/libp2p/specs/blob/master/addressing/README.md)
[^4]: [Building an echo application with libp2p](https://ldej.nl/post/building-an-echo-application-with-libp2p/)
[^5]: [What is Discovery & Routing ](https://docs.libp2p.io/concepts/discovery-routing/overview/)
[^6]: [一致性算法-Gossip协议详解](https://cloud.tencent.com/developer/article/1662426)
[^7]: [洪泛路由](https://zh.wikipedia.org/wiki/%E6%B4%AA%E6%B3%9B%E8%B7%AF%E7%94%B1)
[^8]: [What is Publish/Subscribe](https://docs.libp2p.io/concepts/pubsub/overview/)

