---
tags: [MQ]
date: 2017-05-02
picture: /开发/IoT/coap/iot.png
---

# COAP 应用详解

CoAP是一种应用层协议，它运行于UDP协议之上而不是像HTTP那样运行于TCP之上。

COAP 相当于 HTTP Restful 就比如， arm 相当于 x86

## COAP 应用层解释

coap请求代码示例

```go
coapClient, err := coap.Dial("myhourse.local:5683")
req := coap.Message{
		Type:      coap.Confirmable,
		Code:      coap.POST,
		MessageID: 12345,
		Payload:   cbor.MustMarshal(People {
			Name :"张三丰",
		}),
	}
req.SetOption(coap.ContentFormat, coap.AppCBOR)
req.SetPathString("/lock/3/users")
rv, err := coapClient.Send(req)
```

```bash
coap://myhourse.local/lock/3/users
```
`GET` `POST` `PUT` `DELETE`



### 简介

> CoAP协议基于REST 构架，REST 是指表述性状态转换架构，是互联网资源访问协议的一般性设计风格。为了克服HTTP对于受限环境的劣势，CoAP既考虑到数据报长度的最优化，又考虑到提供可靠通信。一方面，CoAP提供URI，REST 式的方法如GET，POST，PUT和DELETE，以及可以独立定义的头选项提供的可扩展性。另一方面，CoAP基于轻量级的UDP协议，并且允许IP多播。而组通信是物联网最重要的需求之一，比如说用于自动化应用中。为了弥补UDP传输的不可靠性，CoAP定义了带有重传机制的事务处理机制。并且提供资源发现机制，并带有资源描述。



CoAP的特点

* CoAP采用了二进制报头，而不是文本报头(text header)。

* CoAP降低了头的可用选项的数量。

* CoAP减少了一些HTTP的方法。

* CoAP可以支持检测装置。

CoAP与Http协议

CoAP协议采用了双层的结构。事务层(Transaction layer)处理节点间的信息交换，同时，也提供对多播和拥塞控制的支持。请求/响应层(Request/Response layer)用以传输对资源进行操作的请求和相应信息。CoAP协议的REST 构架基于该层的通信，REST请求附在一个CON 或者NON消息上，而REST响应附在匹配的ACK消息上。



![coap_http](coap_http.jpg)

> CoAP的双层处理方式，使得CoAP没有采用TCP协议，也可以提供可靠的传输机制。利用默认的定时器和指数增长的重传间隔时间实现 CON消息的重传，直到接收方发出确认消息。另外，CoAP的双层处理方式支持异步通信，这是物联网和M2M应用的关键需求之一。


### 报文详情

```coap
   0                   1                   2                   3
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |Ver| T |  TKL  |      Code     |          Message ID           |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |   Token (if any, TKL bytes) ...
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |   Options (if any) ...
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |1 1 1 1 1 1 1 1|    Payload (if any) ...
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-
```

### Type
```
    Confirmable 	 COAPType = 0 //需要响应，确认
	NonConfirmable   COAPType = 1 
	Acknowledgement  COAPType = 2 //确认
	Reset COAPCode   COAPType = 3
```

####  请求类型 Confirmable NonConfirmable


0 Confirmable：需要被确认的请求，如果CON请求被发送，那么对方必须做出响应。 （用于同步请求，控制指令等）

1 NonConfirmable：不需要被确认的请求，如果NON请求被发送，那么对方不必做出回应。（用于属性上报，日志记录等）

#### 响应类型 Acknowledgement Reset

2 Acknowledgement：应答消息，接受到Confirmable消息的响应。(ACK)

3 Reset 消息代表的是一个消息（需要应答或者不需要应答的消息）被收到了，但是由于缺少某些上下文信息而无法被正常的处理。这种情况通常是由于接收节点重启了，因而缺失了一些必要的信息，导致当前接收到的消息无法被处理。利用reset消息，也是一种低开销的检查端是否存活的方式（也称作CoAP ping，发送一个空的需应答消息）类似于http响应的：Provisional headers are shown


### Code

####  请求方法码（Method Code）

类似于 HTTP的请求 METHOD，用于消息请求

```
    GET    COAPCode = 1
	POST   COAPCode = 2
	PUT    COAPCode = 3
	DELETE COAPCode = 4
```

#### 响应码（Response Code）

用于消息等响应，类似于HTTP的200，400，404, 401 等

```
	// 2.x
	CoapCodeCreated  CoapCode = 65 // 2.01
	CoapCodeDeleted  CoapCode = 66 // 2.02
	CoapCodeValid    CoapCode = 67 // 2.03
	CoapCodeChanged  CoapCode = 68 // 2.04
	CoapCodeContent  CoapCode = 69 // 2.05
	CoapCodeContinue CoapCode = 95 // 2.31

	// 4.x
	CoapCodeBadRequest               CoapCode = 128 // 4.00
	CoapCodeUnauthorized             CoapCode = 129 // 4.01
	CoapCodeBadOption                CoapCode = 130 // 4.02
	CoapCodeForbidden                CoapCode = 131 // 4.03
	CoapCodeNotFound                 CoapCode = 132 // 4.04
	CoapCodeMethodNotAllowed         CoapCode = 133 // 4.05
	CoapCodeNotAcceptable            CoapCode = 134 // 4.06
	CoapCodeRequestEntityIncomplete  CoapCode = 136 // 4.08
	CoapCodeConflict                 CoapCode = 137 // 4.09
	CoapCodePreconditionFailed       CoapCode = 140 // 4.12
	CoapCodeRequestEntityTooLarge    CoapCode = 141 // 4.13
	CoapCodeUnsupportedContentFormat CoapCode = 143 // 4.15

	// 5.x
	CoapCodeInternalServerError  CoapCode = 160 // 5.00
	CoapCodeNotImplemented       CoapCode = 161 // 5.01
	CoapCodeBadGateway           CoapCode = 162 // 5.02
	CoapCodeServiceUnavailable   CoapCode = 163 // 5.03
	CoapCodeGatewayTimeout       CoapCode = 164 // 5.04
	CoapCodeProxyingNotSupported CoapCode = 165 // 5.05
```

### Token

令牌（token） token是用于匹配响应与请求的。token的值有0~8字节（注意，每个信息都携带token，即使其长度为零）。每个请求都携带由客户端生成的token，服务端在响应时必须复制（不能修改）这个token。

token用作client-local标示，用于区分并发请求，也称为“request ID”。


### Options


```
   +-----+---+---+---+---+----------------+--------+--------+----------+
   | No. | C | U | N | R | Name           | Format | Length | Default  |
   +-----+---+---+---+---+----------------+--------+--------+----------+
   |   1 | x |   |   | x | If-Match       | opaque | 0-8    | (none)   |
   |   3 | x | x | - |   | Uri-Host       | string | 1-255  | (see     |
   |     |   |   |   |   |                |        |        | below)   |
   |   4 |   |   |   | x | ETag           | opaque | 1-8    | (none)   |
   |   5 | x |   |   |   | If-None-Match  | empty  | 0      | (none)   |
   |   7 | x | x | - |   | Uri-Port       | uint   | 0-2    | (see     |
   |     |   |   |   |   |                |        |        | below)   |
   |   8 |   |   |   | x | Location-Path  | string | 0-255  | (none)   |
   |  11 | x | x | - | x | Uri-Path       | string | 0-255  | (none)   |
   |  12 |   |   |   |   | Content-Format | uint   | 0-2    | (none)   |
   |  14 |   | x | - |   | Max-Age        | uint   | 0-4    | 60       |
   |  15 | x | x | - | x | Uri-Query      | string | 0-255  | (none)   |
   |  17 | x |   |   |   | Accept         | uint   | 0-2    | (none)   |
   |  20 |   |   |   | x | Location-Query | string | 0-255  | (none)   |
   |  35 | x | x | - |   | Proxy-Uri      | string | 1-1034 | (none)   |
   |  39 | x | x | - |   | Proxy-Scheme   | string | 1-255  | (none)   |
   |  60 |   |   | x |   | Size1          | uint   | 0-4    | (none)   |
   +-----+---+---+---+---+----------------+--------+--------+----------+
```

```
	IfMatch       OptionID = 1
	URIHost       OptionID = 3
	ETag          OptionID = 4
	IfNoneMatch   OptionID = 5
	Observe       OptionID = 6
	URIPort       OptionID = 7
	LocationPath  OptionID = 8
	URIPath       OptionID = 11
	ContentFormat OptionID = 12
	MaxAge        OptionID = 14
	URIQuery      OptionID = 15
	Accept        OptionID = 17
	LocationQuery OptionID = 20
	ProxyURI      OptionID = 35
	ProxyScheme   OptionID = 39
	Size1         OptionID = 60
```

保留定义：

```
       +-------------+---------------------------------------+
          |       Range | Registration Procedures               |
          +-------------+---------------------------------------+
          |       0-255 | IETF Review or IESG Approval          |
          |    256-2047 | Specification Required                |
          |  2048-64999 | Expert Review                         |
          | 65000-65535 | Experimental use (no operational use) |
          +-------------+---------------------------------------+
```

添加Token验证请求， 可用于鉴权 (ietf草稿文档)

```
   +-----+---+---+---+---+----------------------+--------+--------+------------------ ----+
   | No. | C | U | N | R | Name                 | Format | Length | Default               |
   +-----+---+---+---+---+----------------------+--------+--------+-----------------------+
   |  64 |   |   |   |   | Authorization        | opaque | 1-1034 | (none)                |
   |  65 | x |   |   |   | Authorization-Format | uint   | 0-2    | application/cose+cbor |
   +-----+---+---+---+---+--------------- ------+--------+--------+-----------------------+
```

application/cose+cbor 可以理解为二进制的json编码格式，比Json更小，但是大部分情况下， Authorization-Format这个值可以写0，即  text/plain;charset=utf-8 格式。

#### Uri-Host, Uri-Port, Uri-Path, and Uri-Query

例如接口如下：

```bash
GET coap://dc.spotmau.cn:5683/lock/v3/home/2435/users?name=张三丰
```
查询HOME ID 是 2435 的家庭中，姓名为张三丰的用户的详情。

对应的参数为:

Uri-Host: dc.spotmau.cn
Uri-Port: 5683
Uri-Path: lock/v3/users/dn6426523
Uri-Query: name=张三丰

#### If-Match 

字面意思：如果目标数据的ETag和 if-match数值相同，则做某事。用于保护多个客户端在同一资源下进行类似操作时意外覆盖（比如“lost update”问题）。

例如：多个客户端同时请求修改门锁数据时，添加 if-match aaaaa, 则表示，只有门锁数据的etag为aaaaa的时候才修改，否则忽略。

#### ETag 

资源唯一标识符，用户检测变化。md5, time 等混合。（一般服务器端使用）

#### If-None-Match

字面意思：如果目标数据不存在时，则做某事。否则忽略。

#### Observe 

用于消息订阅/发布模式，详见：观察者机制代码实现

#### Location-Path和 Location-Query

Location-Path和Location-Query选项定义由一个绝对路径、一个请求字符串，或者二者一起组成的相对URI。

#### Content-Format

```
	TextPlain     MediaType = 0  // text/plain;charset=utf-8
	AppLinkFormat MediaType = 40 // application/link-format
	AppXML        MediaType = 41 // application/xml
	AppOctets     MediaType = 42 // application/octet-stream
	AppExi        MediaType = 47 // application/exi
	AppJSON       MediaType = 50 // application/json
	AppCBOR       MediaType = 60 // application/json
```

#### Max-Age

Max-Age 定义最大缓存实践， 最多能缓存的时间， 一般为GET到的内容的有效期。

该选项的值是一个整型的秒数，从0到2**32-1（大约136.1年）。如响应中没有定义这个选项，它的默认值是60秒。

####  Accept 

类似于 http的Accept，客户端接受服务端返回它指定的格式。如果服务端无法提供客户端指定的格式，服务端必须返回一个4.06“Not Acceptable”，

#### Proxy-Uri和Proxy-Scheme

和 http的 类似

#### Size1

用于限制请求字节数， 如果一个请求返回413 （Request Entity Too Large）， 则该请求所有响应会包含Size1字段。

## COAP 订阅机制代码实现

观察者模式又称发布订阅模式

代码演示：

Server

```go
func periodicTransmitter(l *net.UDPConn, a *net.UDPAddr, m *coap.Message) {
	subded := time.Now()
	for {
		msg := coap.Message{
			Type:      coap.Acknowledgement,
			Code:      coap.Content,
			MessageID: m.MessageID,
			Payload:   []byte(fmt.Sprintf("Been running for %v", time.Since(subded))),
		}

		msg.SetOption(coap.ContentFormat, coap.TextPlain)
		log.Println(m.Path())
		msg.SetOption(coap.LocationPath, m.Path())

		log.Printf("Transmitting %v", msg)
		err := coap.Transmit(l, a, msg)
		if err != nil {
			log.Printf("Error on transmitter, stopping: %v", err)
			return
		}
		time.Sleep(time.Second)
	}
}

func main() {
	coap.ListenAndServe("udp", ":5683", coap.FuncHandler(func(l *net.UDPConn, a *net.UDPAddr, m *coap.Message) *coap.Message {
			if m.Code == coap.GET && m.Option(coap.Observe) != nil {
				//判断如果是Observe类型，则调用订阅事件
				if value, ok := m.Option(coap.Observe).(uint32); ok && value == 1 && m.Path()[0]== "obs"  {
					go periodicTransmitter(l, a, m)
				}
			}
			return nil
		}))

}
```

Client

```go
req := coap.Message{
		Type:      coap.NonConfirmable,
		Code:      coap.GET,
		MessageID: 12345,
	}
	req.AddOption(coap.Observe, 1)
	req.SetPathString("/obs")

	c, err := coap.Dial("udp", "localhost:5683")
	if err != nil {
		log.Fatalf("Error dialing: %v", err)
	}

	rv, err := c.Send(req)
	if err != nil {
		log.Fatalf("Error sending request: %v", err)
	}
	for err == nil {
		if rv != nil {
			if err != nil {
				log.Fatalf("Error receiving: %v", err)
			}
			log.Printf("Got %s", rv.Payload)
		}
		rv, err = c.Receive()

	}
   log.Printf("Done...\n", err)
```

从代码分析可看出，发布订阅模式其实是对指定path的轮询Receive，与http轮询不同的是，发送一次请求，没有握手机制。可以设置重试次数。



## COAP 多播模式实现

CoAP支持在IP多播组中发送请求，这相当于连续的单播CoAP。使用场景类似于我们的APP消息推送，这种场景只推送消息，不需要接受消息应答。

### 消息层

* 多播请求的特点是目的地址由具体的CoAP端地址变成了IP多播地址，多播请求必须是不需应答消息。

### 请求响应层
如果服务端决定响应一个多播请求，它不应该立即响应。相反它应该会等待一段时间才进行响应。我们把这段时间称为空闲（Leisure）时间。


## COAP 服务器API最佳实践

#### API 文档示例:  设备向服务器端请求

向某个家庭添加用户

```
POST coaps://dc.spotmau.cn:5683/lock/v3/home/:id/users
```

##### 请求参数 

```
Content-Type: application/json
```

|  参数   |   类型   | 是否必须 | 描述            |
| :---: | :----: | :--: | ------------- |
| name  | string |  是   | 人员姓名：长度：2～16  |
| image | string |  否   | 人员头像的base64编码 |
| desc  | string |  否   | 人员描述          |


##### 响应

```
Code: 0 OK
```

```
Code:  != 0
```

| 状态码  | 错误类型                  | 描述         |
| :--: | --------------------- | ---------- |
| 4.00 | invalid_home          | 未找到对应家庭    |
| 4.00 | invalid_phone         | 被授权用户未绑定手机 |
| 5.00 | internal_server_error | 系统处理错误     |
| 4.09 | conflict              | 创建家庭数量超过限制 |


```json
{
  "error": "invalid_home",
  "error_description": "home not found"
}
```

 CoAP-HTTP Proxying

 HTTP-CoAP Proxying

 


#### API 文档示例:  服务器向设备请求

代理模式

添加用户

```
POST coaps://[2001:db8::2:1]:5683/device_id/lock/:id/people
```

```
{
  "name":"张三丰"
}
```
删除用户
```
DELETE coaps://[2001:db8::2:1]:5683/device_id/lock/:id/people/:uid
```

修改用户


```
PUT coaps://[2001:db8::2:1]:5683/device_id/lock/:id/people/:uid
```

```
{
  "name":"张三"
}
```
查询用户
```
GET coaps://[2001:db8::2:1]:5683/device_id/lock/:id/people?name="张三"
```










```
GET coaps://[2001:db8::2:1]:5683/device_id/open
```

##### 请求参数

```
Authorization: {{wang_guan_token}}
```

##### 响应

```
Code: 2.x OK
```

```
Code:  != 2.x
```

| 状态码  | 错误类型                  | 描述                   |
| :--: | --------------------- | -------------------- |
| 4.04 | not_found             | 设备不存在                |
| 4.01 | unauthorized          | 没有添加验证               |
| 4.03 | forbidden             | 操作被拒绝                |
| 5.00 | internal_server_error | 处理错误，会将错误信息返回到结果Body |

## COAPs 加密传输

COAP 采用 DTLS作为加密手段， PSK 是DTLS 定义的密钥交换方案之一，相对于公钥证书方案(如 ECDHA_RSA) 来说，其具备更加轻量化、高效的优点。

```
//服务器端
server.HandlePSK(func(id string) []byte {
		return []byte("secretPSK")
	})
server.ListenAndServeDTLS(":5684")

//客户端
conn, err := coap.DialDTLS("localhost:5684", "clientId", "secretPSK")
	if err != nil {
		panic(err.Error())
	}
```

## 总结

从代码实现可以看出CoAP 是类似于 HTTP Restful API形式， 请求和返回消息格式一致，集成设备订阅模式的物联网解决方案。CoAP可以作为短连和长连处理，支持重连机制的无序消息解决方案。

CoAP相对于MqTT而言，有以下不同：

1. CoAP 允许将设备作为服务器终端，允许在没有长连的情况下向设备发送指令。
2. MqTT 的思想是将设备集中联网，然后采取订阅发送的模式。
3. CoAP 的订阅模式不能获取消息的应答。
4. 模式使用场景：以开门为例：开门，则向设备发送 GET coap://cip:cport/lock_id/unlock 方法， 关门则发送  coap://cip:cport/lock_id/lock 方法。消息推送则采取订阅方式。 
5. CoAP 默认采用PSK算法保证数据传输加密。



## 请求/响应的匹配 

见于目前请求响应模型和当前系统差别很大，就请求/响应的匹配做补充说明。

### 5.3.1, 令牌（token） 

token是用于匹配响应与请求的。token的值有0~8字节（注意，每个信息都携带token，即使其长度为零）。每个请求都携带由客户端生成的token，服务端在响应时必须复制（不能修改）这个token。

token用作client-local标示，用于区分并发请求（参见5.3节），也称为“request ID”。

客户端生成token时需要注意，当前使用的token对给定的源端和目的端应该都是独一无二的。（注意客户端在生成token时，如果要向不同的端（比如源端口号不同）中发送请求，可以使用同样的token）。当只向目的端产生一个token，或者向每个目的端发送的请求都是顺序的，且都是附带响应，token为空也是可行的。有多种策略实现。

如果客户端不使用传输层安全(TLS，见第9章）发送请求，就需要使用复杂的，随机的token来防止欺诈响应（见11.4节），起到保护功能，这也是token允许使用最多8个字节的原因。token中随机组件的实际长度取决于客户端的安全需求和欺诈响应造成的威胁程度。接入到互联网的客户端至少应该使用32位随机码，记住，没有直接连接互联网也不一定能有效防范欺诈。注意，Message ID几乎没有添加保护，因为它通常是顺序分配的，因此可能被猜测到，并通过欺诈响应绕过。客户端想要优化token长度，可能会向进一步检测正在进行的攻击等级（例如计算接收的token不匹配的消息个数）。[RFC4086]讨论对安全的随机性要求。

端接收一个不是它生成的token，必须把这个token当做不透明的，不能假设它的内容和结构。 

### 5.3.2, 请求/响应匹配规则

 确切的匹配响应与请求的规则如下：

响应的源端必须和原始请求的目的端一致。

在附带响应中，CON请求和ACK的“Message ID”必须匹配，响应和原始请求的“token”必须匹配。在单独响应中，只需响应和原始请求的“token”匹配。万一信息携带异常的响应（不是认定的端，端地址、token和客户端的期望不匹配），这个响应必须被拒绝（见4.2和4.3）。

注意：客户端接收到CON响应之后，可能想在回复完ACK马上清除这个消息的状态。如果这个ACK丢失，且服务端重传这个CON消息，客户端可能不会再有任何与该响应相关联的状态，会导致这个重传成为异常消息；客户端可能会发送RST信息，这样它就不会再收到更多的重传消息。这个行为是正常的，并不是一个错误（没有积极优化内存使用状态的客户端会将第二个CON认定为重发。客户端事实上期望从服务器[observe]得到更多消息，就必须在任何情况下保持状态）。

