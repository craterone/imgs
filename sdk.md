## SDK架构

SDK主要分三个模块
- 本地流(MediaStream or MediaTrack)
- WebSocket（信令）
- RTCPeerConnection（pub/sub）

### 本地流
用于获得本地摄像头，麦克风等设备，通过`RTCPeerConnection`发布到服务器。本地流包括以下类型：

1. 音视频（audio，video）
2. 屏幕分享（screensharing）
3. 音视频文件（file）
4. 画布（canvas）

各个端支持类型
- Web支持类型:1,2,3,4
- Android支持类型:1,2,3
- iOS支持类型:1


`macOS` 和 `Windows` 屏幕分享


按照最新`WebRTC` 支持的 `Unified Plan`，目前使用`mediatrack`进行传输。`mediasoup`也只支持`Unified Plan`。

**P.S.** 在`Android`和`iOS`需要通过`RTCPeerConnectionFactory` 来创建`mediatrack`。


### WebSocket 
用于和信令服务器连接，进行通讯。

需要注意以下几点：
1. 消息需分2种，通知（`notify`）和 请求（`request`）。`notify`用于通知，不需要`response`确认。`request`需要确认回复`response`，并且每条消息需要一一对应的`id`，并且增加`timeout`机制。
2. `WebRTC`需要`https`支持，需要确保移动端`Websocket`第三方包支持`wss`。


### RTCPeerConnection
流媒体传输通道，用于**发布**和**订阅**音视频。

`RTCPeerConnection` 两种使用方式。
1. 连接服务器只建立 2道 `RTCPeerConnection`，1道用于发布，1道用于订阅。
    - **优点**：节省端口和连接时间；利于`WebRTC transportcc`算法控制带宽。
    - **缺点**：单房间人数过多，会造成下行服务器带宽过载。
2. 每道流都建立1道`RTCPeerConnection`。
    - **优点**：每到连接可以可以选择单独的服务器，不会造成下行服务器过载。
    - **缺点**：多通道会抢流量，造成某些连接出现“饥饿现象”，即有些带宽分多了，有的连接一点儿带宽也都分不到。

`mediasoup` 推荐使用第1种方式。
  
## SDK 设计

- transport : 对应客户端`RTCPeerConnection`和媒体服务器的连接。
- sender : 对应发送端1路`mediaTrack`。
- receiver : 对应接收端1路`mediaTrack`。


**第一阶段**：
1. 通过 `token` 连接到信令服务器。
2. 向信令服务器发送请求，创建 2个 `transport`，分别用于发布和订阅。
3. `transport`创建成功，返回 `ICE` 信息(`username`,`password`,`candidate`) 和 `DTLS fingerprint` 信息。
4. 获得信息后，并不马上连接媒体服务器。


**发布**：
1. 创建一路 `mediaTrack`，通过 `RTCPeerConnection.addTransceiver`添加到连接中。
2. `createOffer` 并 `setLocalDescription`，然后从`offer`的`sdp`中获得本地的 `DTLS fingerprint`。
3. 根据第一阶段获得`ICE` 信息 ， `DTLS fingerprint` 信息 和 `offer sdp` 生成 `answer`。（\*）
4. 通过生成的`answer`调用`setRemoteDescription`，并把第2步中的本地`DTLS fingerprint`发回给信令服务器（仅调用1次）。
5. 从`offer sdp`中获取当前`mediatrack`的信息，本地存为`sender`（sender中包括发送编码，ssrc等信息），发送给信令服务器。(\*)

**订阅**：
1. 收到其他端的**发布**的**通知**，`SDK`回调接口信息，确认是否订阅。
2. 通过调用信令服务器确认订阅，确认订阅后返回`receiver`信息（receiver包括接收的编码，ssrc等信息）。(\*)
3. 根据第一阶段获得`ICE` 信息 ， `DTLS fingerprint` 信息 和 `receiver` 信息  生成 `answer`。（\*）
4. 设置 `answer` 和 生成 `offer`。


`transport` 同时需要实现 `异步队列`，多个`发布`或者`订阅`操作，需要`异步队列`保持运行顺序。


- mediasoup `异步队列`： https://github.com/versatica/awaitqueue
- mediasoup  `C++客户端连接分装` ： https://mediasoup.org/documentation/v3/libmediasoupclient/design/  ， 问题：1. 只有底层接口，2. 没有多线程处理

----

**Token机制**：连接信令服务器应需要验证方式，而且需要支持附带参数。需要一个`token`签发服务器，客户端通过签发服务器获得`token`再去信令服务器验证。信令服务器和签发服务器同时持有`secret`，通过`secret`进行加密和解密，推荐 `tokenlib`。

**tokenlib**
- python ：https://github.com/mozilla-services/tokenlib
- go ：https://github.com/egoag/tokenlib

## 业务 SDK
 
![](https://raw.githubusercontent.com/craterone/imgs/master/%E4%B8%9A%E5%8A%A1sdk.png)
