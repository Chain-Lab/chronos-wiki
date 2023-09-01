# go-libp2p

### 启动本地节点

通过 `libp2p.New()` 函数使用默认的配置启动一个节点

```go
package main

import (
    "fmt"
    "github.com/libp2p/go-libp2p"
)

func main() {
    // start a libp2p node with default settings
    node, err := libp2p.New()
    if err != nil {
        panic(err)
    }

    // print the node's listening addresses
    fmt.Println("Listen addresses:", node.Addrs())

    // shut the node down
    if err := node.Close(); err != nil {
        panic(err)
    }
}
```

### 节点连接

在启动本地节点后，通过 `node.Addrs()` 函数得到节点的地址，如果需要连接到另外一个节点，可以通过该地址信息连接到对端节点

```go
package main

import (
    "context"
    "fmt"
    "os"
    "os/signal"
    "syscall"

    "github.com/libp2p/go-libp2p"
    peerstore "github.com/libp2p/go-libp2p-core/peer"
    "github.com/libp2p/go-libp2p/p2p/protocol/ping"
    multiaddr "github.com/multiformats/go-multiaddr"
)

func main() {
    ...
    addr, err := multiaddr.NewMutiaddr(addr_str)
    
    if err != nil {
        panic(err)
    }
    peer, err := peerstore.AddrInfoFromP2pAddr(addr)
    if err != nil {
        panic(err)
    } 
    if err := node.Connect(context.Background(), *peer); err != nil {
        panic(err)
    }
}
```

### DHT 启动

在 go-libp2p 中，可以使用 mDNS 或 [[Kademlia 协议]] 来发现其他的节点[^1]，**mDNS 在端口 5353 上多播 UDP 消息，但不适用于互联网中，只能用于本地的内网环境。**

节点的发现需要结合 go-libp2p-kad-dht 中的 `ipfsDHT` 和 go-libp2p 中的 `discovery` 来实现 (能不能给 `ipfsDHT` 换一个名字)

```go
// @param ctx: 程序运行上下文
// @param host: 运行 libp2p 的主机
// @param bootstrapPeers: 引导节点数组
// @return disc: 路由发现类
func NewKDHT(ctx context.Context, host host.Host, bootstrapPeers []multiaddr.Multiaddr) (*disc.RoutingDiscovery, error) {
	// dht 的配置项
    var options []dht.Option
	
    // 如果没有引导节点，以服务器模式 ModeServer 启动
	if len(bootstrapPeers) == 0 {
		options = append(options, dht.Mode(dht.ModeServer))
	}
	
    // 生成一个 DHT 实例
	kdht, err := dht.New(ctx, host, options...)
	if err != nil {
		return nil, err
	}
	
    // 启动 DHT 服务
	if err = kdht.Bootstrap(ctx); err != nil {
		return nil, err
	}
	
    
    // 遍历引导节点数组并尝试连接
	for _, peerAddr := range bootstrapPeers {
		peerinfo, _ := peer.AddrInfoFromP2pAddr(peerAddr)

		wg.Add(1)
		go func() {
			defer wg.Done()
			if err := host.Connect(ctx, *peerinfo); err != nil {
				log.Printf("Error while connecting to node %q: %-v", peerinfo, err)
			} else {
				log.Printf("Connection established with bootstrap node: %q", *peerinfo)
			}
		}()
	}
	wg.Wait()

	return disc.NewRoutingDiscovery(kdht), nil
}
```

### 节点发现

在建立了 DHT 连接后，可以使用返回的 disc 查询到其他节点，（**这里发现的节点的数量是否有数量上限？**）

```go
func Discover(ctx context.Context, h host.Host, dht *dht.IpfsDHT, rendezvous string) {
    // 建立 DHT 并且得到 RoutingDiscovery 对象
	var routingDiscovery = discovery.NewRoutingDiscovery(dht)
    
    // 启动一个持续广播的 go-routine，直到上下文取消，每3个小时宣布当前节点的存在
	discovery.Advertise(ctx, routingDiscovery, rendezvous)

	ticker := time.NewTicker(time.Second * 1)
	defer ticker.Stop()

	for {
		select {
		case <-ctx.Done():
			return
		case <-ticker.C:
			// 得到发现的其他节点的列表
			peers, err := discovery.FindPeers(ctx, routingDiscovery, rendezvous)
			if err != nil {
				log.Fatal(err)
			}

			for _, p := range peers {
                // 过滤掉本地节点
				if p.ID == h.ID() {
					continue
				}
                // 建立连接，在编写代码时可以在这里启动信息处理协程
				if h.Network().Connectedness(p.ID) != network.Connected {
                        _, err = h.Network().DialPeer(ctx, p.ID)
					if err != nil {
						continue
					}
				}
			}
		}
	}
}
```

