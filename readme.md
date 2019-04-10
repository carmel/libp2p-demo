# `go-libp2p` 示例教程
在此文件夹中，您可以找到各种示例来帮助您开始使用go-libp2p。每个示例旨在帮助你延伸你对libp2p和p2p网络的了解，其中一些整合了您可以遵循的完整教程。

如果发现任何问题或者如果想贡献并添加新教程，请告诉我们，欢迎提交pr，谢谢！

## 示例教程

- [libp2p 'host'](#libp2p-'host')
- [使用libp2p构建http代理](#使用libp2p构建http代理)
- [echo host](#libp2p之echo客户机/服务器)
- [Multicodecs with protobufs-多路复用器](#使用rpc样式的protobuf与libp2p进行协议多路复用)
- [P2P聊天应用](#P2P聊天应用)
- [整合节点发现的P2P聊天应用](#整合节点发现的P2P聊天应用)
- [mdns实现节点发现的P2P聊天应用](#mdns实现节点发现的P2P聊天应用)

## 故障排除

构建示例时，确保您有一个干净的`$GOPATH`。 如果您已经拉取并构建了其他`libp2p`仓库，那么在构建示例时可能会出现类似于下面的错误。请注意，运行示例或使用`libp2p`而**不需要使用**`gx`包管理器。
```
$:~/go/src/github.com/libp2p/go-libp2p-examples/libp2p-host$ go build host.go 
# command-line-arguments
./host.go:36:18: cannot use priv (type "github.com/libp2p/go-libp2p-crypto".PrivKey) as type "gx/ipfs/QmNiJiXwWE3kRhZrC5ej3kSjWHm337pYfhjLGSCDNKJP2s/go-libp2p-crypto".PrivKey in argument to libp2p.Identity:
        "github.com/libp2p/go-libp2p-crypto".PrivKey does not implement "gx/ipfs/QmNiJiXwWE3kRhZrC5ej3kSjWHm337pYfhjLGSCDNKJP2s/go-libp2p-crypto".PrivKey (wrong type for Equals method)
                have Equals("github.com/libp2p/go-libp2p-crypto".Key) bool
                want Equals("gx/ipfs/QmNiJiXwWE3kRhZrC5ej3kSjWHm337pYfhjLGSCDNKJP2s/go-libp2p-crypto".Key) bool
```

要得到一个干净的`$GOPATH`，可以执行如下操作：
```
> mkdir /tmp/libp2p-examples
> export GOPATH=/tmp/libp2p/examples
```
# libp2p 'host'

对于大多数应用来说，host是您需要开始使用的基本构建块。本指南将介绍如何搭建与使用一个简单的host。

host是一种抽象，可以在群集之上管理服务。它提供了一个干净的界面来连接指定的远程节点上的服务。

你若想以默认配置来创建一个host，可以执行以下操作：

```go
import (
	"context"
	"crypto/rand"
	"fmt"

	libp2p "github.com/libp2p/go-libp2p"
	crypto "github.com/libp2p/go-libp2p-crypto"
)


// The context governs the lifetime of the libp2p node  
// context上下文控制libp2p节点的生存周期
ctx, cancel := context.WithCancel(context.Background())
defer cancel()

// To construct a simple host with all the default settings, just use `New`  
// 构造一个具有所有默认设置的简单host，只需使用“New”方法
h, err := libp2p.New(ctx)
if err != nil {
	panic(err)
}

fmt.Printf("Hello World, my hosts ID is %s\n", h.ID())
```

如果想要更多地控制配置，可以为构造函数指定一些选项。有关构造函数支持的所有配置的完整列表，请参阅：[options.go](https://github.com/libp2p/go-libp2p/blob/master/options.go)

在下面的代码片段中，我们生成自己的ID并指定我们想要监听的地址：
```go
// Set your own keypair  
// 配置自身的密钥对
priv, _, err := crypto.GenerateEd25519Key(rand.Reader)
if err != nil {
	panic(err)
}

h2, err := libp2p.New(ctx,
	// Use your own created keypair  
	// 使用自身创建的密钥对
	libp2p.Identity(priv),

	// Set your own listen address
	// The config takes an array of addresses, specify as many as you want.  
	// 配置自身的监听地址
	// 该配置采用地址数组的形式，想指定多少就可指定多少
	libp2p.ListenAddrStrings("/ip4/0.0.0.0/tcp/9000"),
)
if err != nil {
	panic(err)
}

fmt.Printf("Hello World, my second hosts ID is %s\n", h2.ID())
```

就这样，你有了一个libp2p host，已经准备好开始做一些很棒的p2p网络了！

在以后的指南中，我们将讨论host的使用方法，以不同的方式配置它们（提示：有很多方法可以设置它们），以及使用有趣的方式将这种技术应用于你要构建的各种应用程序上。

要查看这些包含各种配置的代码，请查看[host.go](host.go)。

# 使用libp2p构建http代理

该示例展示如何使用libp2p构建一个简单的HTTP代理服务：

```
                                                                                                    XXX
                                                                                                   XX  XXXXXX
                                                                                                  X         XX
                                                                                        XXXXXXX  XX          XX XXXXXXXXXX
                  +----------------+                +-----------------+              XXX      XXX            XXX        XXX
 HTTP Request     |                |                |                 |             XX                                    XX
+----------------->                | libp2p stream  |                 |  HTTP       X                                      X
                  |  Local peer    <---------------->  Remote peer    <------------->     HTTP SERVER - THE INTERNET      XX
<-----------------+                |                |                 | Req & Resp   XX                                   X
  HTTP Response   |  libp2p host   |                |  libp2p host    |               XXXX XXXX XXXXXXXXXXXXXXXXXXXX   XXXX
                  +----------------+                +-----------------+                                            XXXXX
```

为代理一个HTTP请求，我们创建一个本地节点来监听`localhost:9900`。对该地址执行的HTTP请求通过libp2p流经由隧道传输到远程节点，远程节点接着执行HTTP请求并将响应发送回本地节点，并由本地节点将其中继给用户。

注意，这是一种非常简单的代理方法，没有执行任何header管理，也不支持HTTPS。`proxy.go`代码经过全面讨论，详细说明了每一步中发生的事情。

## 构建

在`go-libp2p-examples`目录运行以下命令：

```
> make deps
> cd http-proxy/
> go build
```

## 用法

先运行“远程”节点，如下所示。它将打印本地节点地址。如果你想在单独的机器上运行，请相应地更换IP：

```sh
> ./http-proxy
Proxy server is ready
libp2p-peer addresses:
/ip4/127.0.0.1/tcp/12000/ipfs/QmddTrQXhA9AkCpXPTkcY7e22NK73TwkUms3a44DhTKJTD
```

然后运行本地节点，指定http请求要转发的远程节点，如下所示：

```
> ./http-proxy -d /ip4/127.0.0.1/tcp/12000/ipfs/QmddTrQXhA9AkCpXPTkcY7e22NK73TwkUms3a44DhTKJTD
Proxy server is ready
libp2p-peer addresses:
/ip4/127.0.0.1/tcp/12001/ipfs/Qmaa2AYTha1UqcFVX97p9R1UP7vbzDLY7bqWsZw1135QvN
proxy listening on  127.0.0.1:9900
```

正如所看到的，指定的代理服务打印出监听地址`127.0.0.1:9900`。你现在可以用这个地址作为一个代理，用`curl`进行测试：

```
> curl -x "127.0.0.1:9900" "http://ipfs.io/ipfs/QmfUX75pGRBRDnjeoMkQzuQczuCup2aYbeLxz5NzeSu9G6"
it works!
```

# libp2p之echo客户机/服务器

这是一个快速展示如何使用go-libp2p堆栈的示例，包括Host/Basichost，Network/Swarm，Streams，Peerstores和Multiaddresses。

此示例可以在侦听模式或拨号模式下启动。

在侦听模式下，它将等待`/ echo / 1.0.0`协议上的传入连接。 每当它收到一个流时，它会在流上写下消息“Hello，world！”`并关闭它。

在拨号模式下，节点将启动，连接到给定地址，打开到目标节点的流，并读取基于协议`/ echo / 1.0.0`的消息。

## 构建

在`go-libp2p-examples`目录运行以下命令：

```
> make deps
> cd echo/
> go build
```

## 用法

```
> ./echo -l 10000
2017/03/15 14:11:32 I am /ip4/127.0.0.1/tcp/10000/ipfs/QmYo41GybvrXk8y8Xnm1P7pfA4YEXCpfnLyzgRPnNbG35e
2017/03/15 14:11:32 Now run "./echo -l 10001 -d /ip4/127.0.0.1/tcp/10000/ipfs/QmYo41GybvrXk8y8Xnm1P7pfA4YEXCpfnLyzgRPnNbG35e" on a different terminal
2017/03/15 14:11:32 listening for connections
```

作为监听方的libp2p主机将打印它的`Multiaddress`，以明确如何被访问到（ip4 + tcp）及其随机生成的ID（`QmYo41Gyb ...`）

现在运行另一个与监听方通信的节点：

```
> ./echo -l 10001 -d /ip4/127.0.0.1/tcp/10000/ipfs/QmYo41GybvrXk8y8Xnm1P7pfA4YEXCpfnLyzgRPnNbG35e
```

新节点向监听方发送消息“Hello，world！”`，然后在流上回显它并关闭它。监听方记录消息，发送方记录响应。

## 实现细节

makeBasicHost()方法创建一个go-libp2p-basichost对象。basichost对象包装了go-libp2 swarm并且应该优先被使用。go-libp2p-swarm网络是一个符合go-libp2p-net网络接口的swarm，负责维护流，连接，在它们上复用不同的协议，处理传入的连接等。

为了创建swarm（和一个`basichost`），这个例子需要：

`ipfs协议ID`，如QmNtX1cvrm2K6mQmMEaMxAuB4rTexhd87vpYVot4sEZzxc。该示例在每次运行时自动生成密钥对，并使用从公钥中提取的ID（公钥的哈希值）。使用`-insecure`时，它会使连接保持未加密状态（否则，它会使用密钥对来加密通信）。
`Multiaddress`，以明确如何被访问到这个节点。可以有好几个（例如，使用不同的协议或位置）。示例：/ip4/127.0.0.1/tcp/1234。
`go-libp2p-peerstore`，用作地址簿，在节点ID与multiaddresses之间进行匹配。当手动打开连接时（使用`Connect()`），peertore会自动装载。或者，我们可以像示例一样手动添加`Addddr()`。

`basichost`，现在可用以使用`NewStream`打开流（两个节点之间的双向通道），并使用它们发送和接收标记有Protocol.ID（字符串）的数据。主机还可以通过`SetStreamHandle()`方法监听指定协议的传入连接。

该示例利用以上所有这些以保证监听方与发送方之间使用协议`/echo/1.0.0`（也可以是其他协议）进行的通信。

# 使用rpc样式的protobuf与libp2p进行协议多路复用

该例展示如何通过protobufs使用libp2p Streams在libp2p主机之间编码和传输信息。进入该例前需要先熟悉echo示例。

## 创建

在`go-libp2p-examples`目录运行以下命令：运行以下命令：

```sh
> make deps
> cd multipro/
> go build
```

## 用法

```sh
> ./multipro
```

## 实现细节

该例创建了支持ping和echo两个协议的两个libp2p主机。

每个协议都包含RPC样式的请求和响应，每个请求和响应都是一个类型化的protobufs消息（和一个go数据对象）。

这与将整个p2p协议定义为具有许多可选字段的一个protobuf消息（在使用各种不同的基于protobuf的p2p-lib协议中可观察到-诸如dht）不同。

该例展示了如何将异步接收的响应与其请求进行匹配。当处理响应需要访问请求数据时很有用。

其思想是在每个消息的基础上使用libp2p协议多路复用。

### 特征
1.两种基于类似RPC的请求-响应模式实现的协议-Ping和Echo  
2.脚手架，用于快速实现新的应用级版本化RPC类协议  
3.调用方可对传入消息数据进行完全认证（可能不是消息的发送方节点）  
4.在protobufs中使用p2p格式，其中包含所有协议消息共享的字段  
5.处理响应时对请求数据的完全访问权限

# P2P聊天应用

该程序演示了一个简单的p2p聊天应用程序。它可以在两个节点之间工作  
1.两者都有一个私有IP地址（处于同一网络）  
2.其中至少有一个拥有公共IP地址。

假设'A'和'B'在不同的网络上，主机'A'可能有也可能没有公共IP地址，但至少主机'B'有一个。

用法：在主机'B'上运行`./chat-sp <SOURCE_PORT>`，其中<SOURCE_PORT>可以是任何端口号。然后在节点'A'上运行`./chat -d <MULTIADDR_B>`[`<MULTIADDR_B>`是主机'B'的`multiaddress`，可以从主机'B'控制台获得]。

## 构建

在`go-libp2p-examples`目录运行以下命令：
```
> make deps
> cd chat
> go build
```

## 用法

操作节点'B'：

```
> ./chat -sp 3001
Run ./chat -d /ip4/127.0.0.1/tcp/3001/ipfs/QmdXGaeGiVA745XorV1jr11RHxB9z4fqykm6xCUPX1aTJo

2018/02/27 01:21:32 Got a new stream!
> hi (received messages in green colour)
> hello (sent messages in white colour)
> no
```

操作节点'A'.  
将127.0.0.1用公共IP<PUBLIC_IP>替代，如果节点'B'有的话.

```
> ./chat -d /ip4/127.0.0.1/tcp/3001/ipfs/QmdXGaeGiVA745XorV1jr11RHxB9z4fqykm6xCUPX1aTJo
Run ./chat -d /ip4/127.0.0.1/tcp/3001/ipfs/QmdXGaeGiVA745XorV1jr11RHxB9z4fqykm6xCUPX1aTJo

下面是节点的multiaddress:
/ip4/0.0.0.0/tcp/0/ipfs/QmWVx9NwsgaVWMRHNCpesq1WQAw2T3JurjGDNeVNWifPS7
> hi
> hello
```
 
**注意：** 默认情况下会启用调试模式，调试模式将始终在每次执行时生成相同的节点ID（在每个节点上）。运行可执行文件时，使用`--debug false`标志禁用调试。

**注意：** 如果您正在寻找具有节点发现的实现，[chat-with-rendezvous]（../ chat-with-rendezvous）支持通过`rendezvous point`进行节点发现。
