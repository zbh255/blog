# Golang 实现一个简单的Sock5 Server (一)

#### 参考资料和参考文章

> rfc1928原始文档：https://www.ietf.org/rfc/rfc1928.txt
>
> 协议细则的中文描述：https://blog.csdn.net/sjailjq/article/details/81637196
>
> 代理流程：https://www.dyxmq.cn/network/socks5.html

> 本文章中的代码都可以在这里找到:[Github](https://github.com/zbh255/ss5-simple)，当然版本库可能会随着变更与本文的代码不相同，你也可以使用我提供的`api`实现自己的`Socks Server`和处理流程，之后会更新第二篇文章：《实现一个支持用户验证的socks5 server》，那么接下来就是正文。

---

> 从`socks`客户端的视角看待他和`socks`服务器的交互流程

![](https://blog-xiao-hui-1257821917.file.myqcloud.com/%E6%96%87%E7%AB%A0%E7%B4%A0%E6%9D%90/20211224/SequenceDiagram1.png)

> 本文偏重实现，具体的工作细节和协议细节不再赘述，不了解协议细节的读者可以看看参考资料中的文章

首先我们需要描述所有消息类型

`net/socks_const.go`

```go
const (
	SOCKS5_VERSION byte = 0x05 // 版本号5
	SOCKS4_VERSION byte = 0x04 // 版本号4
)

const (
	SOCKS5_METHOD_NOAUTH              byte = 0x00 // 无验证
	SOCKS5_METHOD_CSSAPI              byte = 0x01 // GSSAPI
	SOCKS5_METHOD_USERNAMEANDPASSWORD byte = 0x02 // 用户名和密码验证
	SOCKS5_METHOD_NOACCEPTABLEMETHODS byte = 0xff
)

const (
	SOCKS5_CMD_CONNECT byte = iota // 客户端发起的Connect 连接方式
	SOCKS5_CMD_BIND // 客户端发起的Bind连接方式
	SOCKS5_CMD_UDP_ASSSOCIATE // 客户端发起的UdpAsssociate连接方式
)

// 客户端请求中的地址类型
const (
	SOCKS5_ATYP_IPV4   byte = 0x01 // ipv4
	SOCKS5_ATYP_DOMAIN byte = 0x03 // ipv6
	SOCKS5_ATYP_IPV6   byte = 0x04 // 域名
)

// 服务端响应消息的类型
const (
	SOCKS5_REPLY_SUCCESS byte = iota // ok
	SOCKS5_REPLY_FAILURE // Socks5服务器故障
	SOCKS5_REPLY_NOT_RULESET // 不允许的规则
	SOCKS5_REPLY_NETWORK_UNREACHABLE // 网络不可达
	SOCKS5_REPLY_HOST_UNREACHABLE // 主机不可达
	SOCKS5_REPLY_CONNECTION_REFUSED // 连接被拒绝
  SOCKS5_REPLY_TTL_EXPIRED // ttl耗尽(超时)
	SOCKS5_REPLY_CMD_NOTSUPPORT // 不支持的命令
	SOCKS5_REPLY_ADDRESSTYPE_NOTSUPPORT // 不支持的地址类型
	SOCKS5_REPLY_UNASSIGNED // 未定义
)
```

`net/socks_header.go`

接下来就是对请求和响应数据的格式做一个抽象

```go
// 客户端的握手请求，对应图中的2阶段
type Socks5HandshakeRequest struct {
	Version  byte // 版本
	NMethods byte // 方法表的长度
	Methods  [256]byte // 客户端方法表
}

// 服务器的响应，对应图中的3阶段
type Socks5HandshakeResponse struct {
	Version byte // 版本
	Method  byte // 服务器选定的方法
}

// 客户端初次建立连接的请求，对应图中的4阶段
type Socks5MessageRequest struct {
	Version  byte
	Command  byte
	Reserved byte
	AddrType byte
	Adders   []byte
	Port     uint16
}

// 服务器的响应，对应图中的5阶段
type Socks5MessageResponse struct {
	Version  byte
	Reply    byte
	Reserved byte
	AddrType byte
	Adders   []byte
	Port     uint16
}
```

抽象好了请求和响应的格式之后就可以编写编码解码的工具函数了

以下的一些细节需要注意

> - 地址类型为`domain`的时候,`DST.Addr`第一个字节存储的是域名的长度，也就是说最大支持的域名长度为`255`
> - `Port`是以网络字节序传输的也就是常说的大端序，解析的时候需要特殊处理，`encoding/binary`中封装了一些处理大端序的方法

`net/socks5_header.go`

```go
func DecodeSocks5HandshakeRequest(reader io.Reader) (*Socks5HandshakeRequest, error) {
	sock := new(Socks5HandshakeRequest)
	buf := make([]byte, 2)
	_, err := reader.Read(buf)
	if err != nil {
		return nil, err
	}
	sock.Version = buf[0]
	sock.NMethods = buf[1]
	buf = make([]byte, sock.NMethods)
	_, err = reader.Read(buf)
	if err != nil {
		return nil, err
	}
	copy(sock.Methods[:sock.NMethods], buf)
	return sock, nil
}

func EncodeSocks5HandshakeResponse(response *Socks5HandshakeResponse) []byte {
	buf := make([]byte, 2)
	buf[0] = response.Version
	buf[1] = response.Method
	return buf
}

func DecodeSocks5MessageRequest(reader io.Reader) (*Socks5MessageRequest, error) {
	buf := make([]byte, 4)
	_, err := reader.Read(buf)
	if err != nil {
		return nil, err
	}
	sock := new(Socks5MessageRequest)
	sock.Version = buf[0]
	sock.Command = buf[1]
	sock.Reserved = 0x00
	sock.AddrType = buf[3]

	switch sock.AddrType {
	case SOCKS5_ATYP_IPV4:
		buf = make([]byte, 4)
	case SOCKS5_ATYP_IPV6:
		buf = make([]byte, 16)
	case SOCKS5_ATYP_DOMAIN:
		buffer := make([]byte, 1)
		_, err := reader.Read(buffer)
		if err != nil {
			return nil, err
		}
		domainLength := buffer[0]
		buf = make([]byte, domainLength)
	default:
		return nil, errors.New(fmt.Sprintf("socks5 address type not supported : %d", sock.AddrType))
	}
	_, err = reader.Read(buf)
	if err != nil {
		return nil, err
	}
	sock.Adders = buf
	// read port
	buf = make([]byte, 2)
	_, err = reader.Read(buf)
	if err != nil {
		return nil, err
	}
	sock.Port = binary.BigEndian.Uint16(buf)
	return sock, nil
}

func EncodeSocks5MessageResponse(response *Socks5MessageResponse) []byte {
	buf := make([]byte, 0, 16)
	buf = append(buf, response.Version)
	buf = append(buf, response.Reply)
	buf = append(buf, response.Reserved)
	buf = append(buf, response.AddrType)
	// domain
	if response.AddrType == SOCKS5_ATYP_DOMAIN {
		buf = append(buf, byte(len(response.Adders)))
	}
	buf = append(buf, response.Adders...)
	buf = append(buf, make([]byte, 2)...)
	binary.BigEndian.PutUint16(buf[len(buf)-2:], response.Port)
	return buf
}
```

接下来我们就可以实现简单的`socks5`服务器了，以下的代码描述的是一个简单的支持`Comment`命令的服务器，以下代码实现将`html`文件返回给`socks`客户端

以下的代码可以在仓库的`example`文件夹中找到

`hello.html`

```go
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>My Socks5 Server response</title>
</head>
<body>
hello world
</body>
</html>
```

`example/noAbstract.go`

```go
package main

import (
	"errors"
	snet "github.com/zbh255/ss5-simple/net"
	"io/ioutil"
	"log"
	"net"
)

func NoAbstractServer() {
	listener,err := net.Listen("tcp","127.0.0.1:1080")
	if err != nil {
		panic(err)
	}
	defer listener.Close()
	for {
		conn,err := listener.Accept()
		if err != nil {
			log.Printf("[Error] : %s\n", err.Error())
		}
		go func() {
			err := handlerConnection(conn)
			if err != nil {
				log.Printf("[Error] : %s\n", err.Error())
			}
		}()
	}
}

func handlerConnection(conn net.Conn) error {
	// custom defined response message
	defer conn.Close()
	rep, err := ioutil.ReadFile("./hello.html")
	if err != nil {
		panic(err)
	}
	// from conn read handshake request
	hRequest, err := snet.DecodeSocks5HandshakeRequest(conn)
	if err != nil {
		return err
	}
	// create handshake response
	hResponse := new(snet.Socks5HandshakeResponse)
	hResponse.Version = hRequest.Version
	hResponse.Method = snet.SOCKS5_METHOD_NOAUTH
	hResponseBytes := snet.EncodeSocks5HandshakeResponse(hResponse)
	n,err := conn.Write(hResponseBytes)
	if err != nil {
		return err
	}
	if n != len(hResponseBytes) {
		return errors.New("write bytes no equal")
	}
	// read client message request
	mRequest,err := snet.DecodeSocks5MessageRequest(conn)
	if err != nil {
		return err
	}
	// create response message
	mResponse := new(snet.Socks5MessageResponse)
	mResponse.Version = mRequest.Version
	mResponse.Reply = snet.SOCKS5_REPLY_SUCCESS
	mResponse.Reserved = mRequest.Reserved
	mResponse.AddrType = mRequest.AddrType
	mResponse.Adders = mRequest.Adders
	mResponse.Port = mRequest.Port
	mResponseBytes := snet.EncodeSocks5MessageResponse(mResponse)
	n,err = conn.Write(mResponseBytes)
	if err != nil {
		return err
	}
	if n != len(mResponseBytes) {
		return errors.New("write bytes no equal")
	}
	// server ok
	// read bytes message
	buffer := make([]byte,4096)
	_, err = conn.Read(buffer)
	if err != nil {
		return err
	}
	_, err = conn.Write(rep)
	return err
}
```

> 验证是否返回正确的数据

```bash
export all_proxy=socks5://127.0.0.1:1080
curl google.com
```

`OutPut`

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>My Socks5 Server response</title>
</head>
<body>
hello world
</body>
</html>
```

![](https://blog-xiao-hui-1257821917.file.myqcloud.com/%E6%96%87%E7%AB%A0%E7%B4%A0%E6%9D%90/20211224/%E6%88%AA%E5%B1%8F2021-12-24%20%E4%B8%8B%E5%8D%889.29.49.png)

> 自此，已经实现了一个简单的无验证的`Socks5`服务器，代码很少，主要是根据协议的流程图来实现，还有一些小坑要注意。