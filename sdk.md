## SDK架构

SDK主要分三个模块
- 本地流(MediaStream or MediaTrack)
- WebSocket（信令）
- PeerConnection（pub/sub）

### 本地流
用于获得本地摄像头，麦克风等设备，通过`Peerconnection`发布到服务器。本地流包括以下类型：

1. 音视频（audio，video）
2. 屏幕分享（screensharing）
3. 音视频文件（file）
4. 画布（canvas）

各个端支持类型
- Web支持类型:1,2,3,4
- Android支持类型:1,2,3
- iOS支持类型:1

按照最新`WebRTC` 支持的 `Unified Plan`，目前使用`mediatrack`进行传输。`mediasoup`也只支持`Unified Plan`。

**P.S.** 在`Android`和`iOS`需要通过`RTCPeerConnectionFactory` 来创建`mediatrack`。


### WebSocket 
用于和信令服务器连接，进行通讯。

需要注意以下几点：
1. 消息需分2种，`notify`和`request`。`notify`用于通知，不需要`response`确认。`request`需要确认回复`response`，并且每条消息需要一一对应的`id`，并且增加`timeout`机制。
2. `WebRTC`需要`https`支持，需要确保移动端`Websocket`第三方包支持`wss`。


### Peerconnection
流媒体传输通道，用于**发布**和**订阅**音视频。

`Peerconnection` 两种使用方式。
1. 连接服务器只建立 2道 `Peerconnection`，1道用于发布，1道用于订阅。
    - **优点**：节省端口和连接时间；利于`WebRTC transportcc`算法控制带宽。
    - **缺点**：单房间人数过多，会造成下行服务器带宽过载。
2. 每道流都建立1道`Peerconnection`。
    - **优点**：每到连接可以可以选择单独的服务器，不会造成下行服务器过载。
    - **缺点**：多通道会抢流量，造成某些连接出现“饥饿现象”，即有些带宽分多了，有的连接一点儿带宽也都分不到。

`mediasoup` 推荐使用第1种方式。
  
## SDK 设计

因为`WebRTC`底层采用工厂模式，所以为了与之相对应，`SDK`也采用了工厂模式，需要一个主入口。

- Web SDK 接口参考：https://github.com/0-u-0/dugon-web-sdk
- iOS SDK 接口参考：https://github.com/0-u-0/dugon-ios-sdk 

以下为封装的具体模块（以下为dugon为例）：
1. 主入口：`dugon`，通过`dugon`创建 `session`和`media`。
2. `Session`：一个`session`对应一个房间，连接`session`需要一个`token`用于验证，`token`里同时包含其他附加信息。同时，`session`包含和服信令服务器的主要连接，并且主要通过`session`的接口**发布**和**订阅**音视频流。
3. `mediasource`：包含上述`本地流`的所有类型的封装。
4. `transport`：上述`Peerconnection`的封装
5. `sender`：对应1道**发布**音频流或者视频流，用户可以用过`sender`的回调来监听流的状态。通过`session.publish`发布并创建`sender`，同时应该支持**编码**等参数配置。
6. `receiver`：对应1道**订阅**音频流或者视频流。


**Token机制**：连接信令服务器应需要验证方式，而且需要支持附带参数。需要一个`token`签发服务器，客户端通过签发服务器获得`token`再去信令服务器验证。信令服务器和签发服务器同时持有`secret`，通过`secret`进行加密和解密，推荐 `tokenlib`。

**tokenlib**
- python ：https://github.com/mozilla-services/tokenlib
- go ：https://github.com/egoag/tokenlib


 

