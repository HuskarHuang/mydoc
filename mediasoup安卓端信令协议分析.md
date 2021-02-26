[TOC]

由于公司要用到mediasoup作为sfu，故对mediasoup安卓客户端协议与信令进行分析，如下：

以下为三个人、陆续订阅两个人的信令协议：

## getRouterRtpCapabilities

up:      {"request":true,"id":4198726,"method":"getRouterRtpCapabilities","data":{}}

down: {"response":true,"id":4198726,"ok":true,"data":{"codecs":[{"kind":"audio","mimeType":"audio/opus","clockRate":48000,"channels":2,"rtcpFeedback":[{"type":"transport-cc","parameter":""}],"parameters":{},"preferredPayloadType":100},{"kind":"video","mimeType":"video/VP8","clockRate":90000,"rtcpFeedback":[{"type":"nack","parameter":""},{"type":"nack","parameter":"pli"},{"type":"ccm","parameter":"fir"},{"type":"goog-remb","parameter":""},{"type":"transport-cc","parameter":""}],"parameters":{"x-google-start-bitrate":1000},"preferredPayloadType":101},{"kind":"video","mimeType":"video/rtx","preferredPayloadType":102,"clockRate":90000,"parameters":{"apt":101},"rtcpFeedback":[]},{"kind":"video","mimeType":"video/VP9","clockRate":90000,"rtcpFeedback":[{"type":"nack","parameter":""},{"type":"nack","parameter":"pli"},{"type":"ccm","parameter":"fir"},{"type":"goog-remb","parameter":""},{"type":"transport-cc","parameter":""}],"parameters":{"profile-id":2,"x-google-start-bitrate":1000},"preferredPayloadType":103},{"kind":"video","mimeType":"video/rtx","preferredPayloadType":104,"clockRate":90000,"parameters":{"apt":103},"rtcpFeedback":[]},{"kind":"video","mimeType":"video/H264","clockRate":90000,"parameters":{"packetization-mode":1,"level-asymmetry-allowed":1,"profile-level-id":"4d0032","x-google-start-bitrate":1000},"rtcpFeedback":[{"type":"nack","parameter":""},{"type":"nack","parameter":"pli"},{"type":"ccm","parameter":"fir"},{"type":"goog-remb","parameter":""},{"type":"transport-cc","parameter":""}],"preferredPayloadType":105},{"kind":"video","mimeType":"video/rtx","preferredPayloadType":106,"clockRate":90000,"parameters":{"apt":105},"rtcpFeedback":[]},{"kind":"video","mimeType":"video/H264","clockRate":90000,"parameters":{"packetization-mode":1,"level-asymmetry-allowed":1,"profile-level-id":"42e01f","x-google-start-bitrate":1000},"rtcpFeedback":[{"type":"nack","parameter":""},{"type":"nack","parameter":"pli"},{"type":"ccm","parameter":"fir"},{"type":"goog-remb","parameter":""},{"type":"transport-cc","parameter":""}],"preferredPayloadType":107},{"kind":"video","mimeType":"video/rtx","preferredPayloadType":108,"clockRate":90000,"parameters":{"apt":107},"rtcpFeedback":[]}],"headerExtensions":[{"kind":"audio","uri":"urn:ietf:params:rtp-hdrext:sdes:mid","preferredId":1,"preferredEncrypt":false,"direction":"sendrecv"},{"kind":"video","uri":"urn:ietf:params:rtp-hdrext:sdes:mid","preferredId":1,"preferredEncrypt":false,"direction":"sendrecv"},{"kind":"video","uri":"urn:ietf:params:rtp-hdrext:sdes:rtp-stream-id","preferredId":2,"preferredEncrypt":false,"direction":"recvonly"},{"kind":"video","uri":"urn:ietf:params:rtp-hdrext:sdes:repaired-rtp-stream-id","preferredId":3,"preferredEncrypt":false,"direction":"recvonly"},{"kind":"audio","uri":"http://www.webrtc.org/experiments/rtp-hdrext/abs-send-time","preferredId":4,"preferredEncrypt":false,"direction":"sendrecv"},{"kind":"video","uri":"http://www.webrtc.org/experiments/rtp-hdrext/abs-send-time","preferredId":4,"preferredEncrypt":false,"direction":"sendrecv"},{"kind":"audio","uri":"http://www.ietf.org/id/draft-holmer-rmcat-transport-wide-cc-extensions-01","preferredId":5,"preferredEncrypt":false,"direction":"recvonly"},{"kind":"video","uri":"http://www.ietf.org/id/draft-holmer-rmcat-transport-wide-cc-extensions-01","preferredId":5,"preferredEncrypt":false,"direction":"sendrecv"},{"kind":"video","uri":"http://tools.ietf.org/html/draft-ietf-avtext-framemarking-07","preferredId":6,"preferredEncrypt":false,"direction":"sendrecv"},{"kind":"video","uri":"urn:ietf:params:rtp-hdrext:framemarking","preferredId":7,"preferredEncrypt":false,"direction":"sendrecv"},{"kind":"audio","uri":"urn:ietf:params:rtp-hdrext:ssrc-audio-level","preferredId":10,"preferredEncrypt":false,"direction":"sendrecv"},{"kind":"video","uri":"urn:3gpp:video-orientation","preferredId":11,"preferredEncrypt":false,"direction":"sendrecv"},{"kind":"video","uri":"urn:ietf:params:rtp-hdrext:toffset","preferredId":12,"preferredEncrypt":false,"direction":"sendrecv"}]}}

首先发送getRouterRtpCapabilities方法，服务返回服务的Capabilities，客户端拿到这个json数据后，缓存到本地，代码如下:

通过本地sdp,转换成本地的RtpCapabilities：

```c++
auto nativeRtpCapabilities = Handler::GetNativeRtpCapabilities(peerConnectionOptions);
```
转换成本地的RtpCapabilities如下：
```c++
// May throw.
auto offer                 = pc->CreateOffer(options);
auto sdpObject             = sdptransform::parse(offer);
auto nativeRtpCapabilities = Sdp::Utils::extractRtpCapabilities(sdpObject);
```

最终将通过对比本地的RtpCapabilities与远程服务Capabilities，本地生成一个extendedRtpCapabilities

```c++
// Get extended RTP capabilities.
this->extendedRtpCapabilities =
  ortc::getExtendedRtpCapabilities(nativeRtpCapabilities, routerRtpCapabilities);
```

## createWebRtcTransport  "producing": true "id": 5a59bce2-f102-4efb-999f-47063f0aa542

up:      {"request":true,"id":6114798,"method":"createWebRtcTransport","data":{"forceTcp":false,"producing":true,"consuming":false,"sctpCapabilities":{"numStreams":{"OS":1024,"MIS":1024}}}}

down: {"response":true,"id":6114798,"ok":true,"data":{"id":"5a59bce2-f102-4efb-999f-47063f0aa542","iceParameters":{"iceLite":true,"password":"dv4q8212r8rrjugbqs5jlnmfwdc4kr1c","usernameFragment":"8mz1n5s1ujayqd72"},"iceCandidates":[{"foundation":"udpcandidate","ip":"192.168.1.132","port":47426,"priority":1076302079,"protocol":"udp","type":"host"}],"dtlsParameters":{"fingerprints":[{"algorithm":"sha-1","value":"01:D5:17:F4:BF:69:9C:14:5D:6B:2D:14:50:7A:89:55:32:83:AA:2A"},{"algorithm":"sha-224","value":"37:4E:27:BE:DC:93:17:0B:01:F8:72:AC:FC:B3:32:33:C6:D4:52:E7:81:FC:16:36:A6:F1:2C:75"},{"algorithm":"sha-256","value":"E2:24:3F:9B:E7:22:24:97:F3:C3:D7:C5:47:44:30:33:B5:97:4C:27:BF:D7:F9:7A:63:77:D3:BC:06:35:B9:B5"},{"algorithm":"sha-384","value":"BE:B0:B8:D6:72:0C:64:45:C8:A2:1F:27:93:9F:CA:4B:48:74:04:90:44:85:4D:13:FC:B1:45:47:FA:F9:CA:CD:95:8B:7E:6F:DA:56:8F:34:7F:9A:95:16:F8:F8:17:EB"},{"algorithm":"sha-512","value":"B8:D5:50:E3:CB:28:62:56:8D:EE:DB:56:22:DE:48:64:A8:BA:4B:0E:EE:3A:80:23:05:C2:23:F0:E5:6F:3C:4E:9D:34:3B:D9:34:E5:C1:00:5C:DB:59:3B:65:33:32:78:CB:20:FA:22:0B:4A:FF:0A:51:1D:3B:53:8B:A3:E5:03"}],"role":"auto"},"sctpParameters":{"MIS":1024,"OS":1024,"isDataChannel":true,"maxMessageSize":262144,"port":5000}}}

获取到服务的iceParameters、iceCandidates、dtlsParameters、id后客户端相应创建一个SendTransport对象，在SendTransport将这些参数传递给SendHandler对象缓存

```c++
this->sendHandler.reset(new SendHandler(
  this,
  iceParameters,
  iceCandidates,
  dtlsParameters,
  sctpParameters,
  peerConnectionOptions,
  sendingRtpParametersByKind,
  sendingRemoteRtpParametersByKind));

Transport::SetHandler(this->sendHandler.get());
```

并且在父类Handler会根据参数创建remoteSdp对象，用于保存服务器传下来的参数

```c++
this->pc.reset(new PeerConnection(this, peerConnectionOptions));

this->remoteSdp.reset(
  new Sdp::RemoteSdp(iceParameters, iceCandidates, dtlsParameters, sctpParameters));
```

这样，上行流通道对象就创建了

## createWebRtcTransport  "producing": false   "id": fa7ab4fa-a7b1-46c9-a0b4-8940cc048fb0

up:      {"request":true,"id":9165309,"method":"createWebRtcTransport","data":{"forceTcp":false,"producing":false,"consuming":true,"sctpCapabilities":{"numStreams":{"OS":1024,"MIS":1024}}}}

down: {"response":true,"id":9165309,"ok":true,"data":{"id":"fa7ab4fa-a7b1-46c9-a0b4-8940cc048fb0","iceParameters":{"iceLite":true,"password":"uvdr94tqvumyxvulzwucu9mfwnfnysn6","usernameFragment":"pusmcbk96d15wmxw"},"iceCandidates":[{"foundation":"udpcandidate","ip":"192.168.1.132","port":41336,"priority":1076302079,"protocol":"udp","type":"host"}],"dtlsParameters":{"fingerprints":[{"algorithm":"sha-1","value":"01:D5:17:F4:BF:69:9C:14:5D:6B:2D:14:50:7A:89:55:32:83:AA:2A"},{"algorithm":"sha-224","value":"37:4E:27:BE:DC:93:17:0B:01:F8:72:AC:FC:B3:32:33:C6:D4:52:E7:81:FC:16:36:A6:F1:2C:75"},{"algorithm":"sha-256","value":"E2:24:3F:9B:E7:22:24:97:F3:C3:D7:C5:47:44:30:33:B5:97:4C:27:BF:D7:F9:7A:63:77:D3:BC:06:35:B9:B5"},{"algorithm":"sha-384","value":"BE:B0:B8:D6:72:0C:64:45:C8:A2:1F:27:93:9F:CA:4B:48:74:04:90:44:85:4D:13:FC:B1:45:47:FA:F9:CA:CD:95:8B:7E:6F:DA:56:8F:34:7F:9A:95:16:F8:F8:17:EB"},{"algorithm":"sha-512","value":"B8:D5:50:E3:CB:28:62:56:8D:EE:DB:56:22:DE:48:64:A8:BA:4B:0E:EE:3A:80:23:05:C2:23:F0:E5:6F:3C:4E:9D:34:3B:D9:34:E5:C1:00:5C:DB:59:3B:65:33:32:78:CB:20:FA:22:0B:4A:FF:0A:51:1D:3B:53:8B:A3:E5:03"}],"role":"auto"},"sctpParameters":{"MIS":1024,"OS":1024,"isDataChannel":true,"maxMessageSize":262144,"port":5000}}}

获取到服务的iceParameters、iceCandidates、dtlsParameters、id后客户端相应创建一个RecvTransport对象，在RecvTransport将这些参数传递给SendHandler对象缓存

```c++
this->recvHandler.reset(new RecvHandler(
  this, iceParameters, iceCandidates, dtlsParameters, sctpParameters, peerConnectionOptions));

Transport::SetHandler(this->recvHandler.get());
```

并且在父类Handler会根据参数创建remoteSdp对象，用于保存服务器传下来的参数

```c++
this->pc.reset(new PeerConnection(this, peerConnectionOptions));

this->remoteSdp.reset(
  new Sdp::RemoteSdp(iceParameters, iceCandidates, dtlsParameters, sctpParameters));
```

这样，下行流通道对象就创建了

## join

up:      {"request":true,"id":4296214,"method":"join","data":{"displayName":"Charjabug","device":{"flag":"chrome","name":"Chrome","version":"87.0.4280.141"},"rtpCapabilities":{"codecs":[{"mimeType":"audio/opus","kind":"audio","preferredPayloadType":100,"clockRate":48000,"channels":2,"parameters":{"minptime":10,"useinbandfec":1},"rtcpFeedback":[{"type":"transport-cc","parameter":""}]},{"mimeType":"video/VP8","kind":"video","preferredPayloadType":101,"clockRate":90000,"parameters":{},"rtcpFeedback":[{"type":"goog-remb","parameter":""},{"type":"transport-cc","parameter":""},{"type":"ccm","parameter":"fir"},{"type":"nack","parameter":""},{"type":"nack","parameter":"pli"}]},{"mimeType":"video/rtx","kind":"video","preferredPayloadType":102,"clockRate":90000,"parameters":{"apt":101},"rtcpFeedback":[]},{"mimeType":"video/VP9","kind":"video","preferredPayloadType":103,"clockRate":90000,"parameters":{"profile-id":2},"rtcpFeedback":[{"type":"goog-remb","parameter":""},{"type":"transport-cc","parameter":""},{"type":"ccm","parameter":"fir"},{"type":"nack","parameter":""},{"type":"nack","parameter":"pli"}]},{"mimeType":"video/rtx","kind":"video","preferredPayloadType":104,"clockRate":90000,"parameters":{"apt":103},"rtcpFeedback":[]},{"mimeType":"video/H264","kind":"video","preferredPayloadType":105,"clockRate":90000,"parameters":{"level-asymmetry-allowed":1,"packetization-mode":1,"profile-level-id":"4d001f"},"rtcpFeedback":[{"type":"goog-remb","parameter":""},{"type":"transport-cc","parameter":""},{"type":"ccm","parameter":"fir"},{"type":"nack","parameter":""},{"type":"nack","parameter":"pli"}]},{"mimeType":"video/rtx","kind":"video","preferredPayloadType":106,"clockRate":90000,"parameters":{"apt":105},"rtcpFeedback":[]},{"mimeType":"video/H264","kind":"video","preferredPayloadType":107,"clockRate":90000,"parameters":{"level-asymmetry-allowed":1,"packetization-mode":1,"profile-level-id":"42e01f"},"rtcpFeedback":[{"type":"goog-remb","parameter":""},{"type":"transport-cc","parameter":""},{"type":"ccm","parameter":"fir"},{"type":"nack","parameter":""},{"type":"nack","parameter":"pli"}]},{"mimeType":"video/rtx","kind":"video","preferredPayloadType":108,"clockRate":90000,"parameters":{"apt":107},"rtcpFeedback":[]}],"headerExtensions":[{"kind":"audio","uri":"urn:ietf:params:rtp-hdrext:sdes:mid","preferredId":1,"preferredEncrypt":false,"direction":"sendrecv"},{"kind":"video","uri":"urn:ietf:params:rtp-hdrext:sdes:mid","preferredId":1,"preferredEncrypt":false,"direction":"sendrecv"},{"kind":"audio","uri":"http://www.webrtc.org/experiments/rtp-hdrext/abs-send-time","preferredId":4,"preferredEncrypt":false,"direction":"sendrecv"},{"kind":"video","uri":"http://www.webrtc.org/experiments/rtp-hdrext/abs-send-time","preferredId":4,"preferredEncrypt":false,"direction":"sendrecv"},{"kind":"video","uri":"http://www.ietf.org/id/draft-holmer-rmcat-transport-wide-cc-extensions-01","preferredId":5,"preferredEncrypt":false,"direction":"sendrecv"},{"kind":"audio","uri":"urn:ietf:params:rtp-hdrext:ssrc-audio-level","preferredId":10,"preferredEncrypt":false,"direction":"sendrecv"},{"kind":"video","uri":"urn:3gpp:video-orientation","preferredId":11,"preferredEncrypt":false,"direction":"sendrecv"},{"kind":"video","uri":"urn:ietf:params:rtp-hdrext:toffset","preferredId":12,"preferredEncrypt":false,"direction":"sendrecv"}]},"sctpCapabilities":{"numStreams":{"OS":1024,"MIS":1024}}}}

down: {"response":true,"id":4296214,"ok":true,"data":{"peers":[]}}

## connectWebRtcTransport "transportId":  5a59bce2-f102-4efb-999f-47063f0aa542  "role": server

up:      {"request":true,"id":2334562,"method":"connectWebRtcTransport","data":{"transportId":"5a59bce2-f102-4efb-999f-47063f0aa542","dtlsParameters":{"role":"server","fingerprints":[{"algorithm":"sha-256","value":"FA:D3:32:2E:E0:F4:C7:D5:BA:10:EE:14:F2:34:87:AD:20:3C:E7:27:A6:4C:0B:48:2C:3E:00:AC:5D:E7:F5:A8"}]}}}

down: {"response":true,"id":2334562,"ok":true,"data":{}}

上行，当本地调用produce创建音频流或者视频流时，如下：

```java
Producer kindProducer = mSendTransport.produce(listener, track, encodings, codecOptions);
```

会调用到C++层

```c++
auto sendData = this->sendHandler->Send(track, &normalizedEncodings, codecOptions);
```
Send方法会SetLocalDescription和SetRemoteDescription，这样完成跟sfu的udp连接
```c++
this->pc->CreateOffer(options);
this->pc->SetLocalDescription(PeerConnection::SdpType::OFFER, offer);
this->pc->SetRemoteDescription(PeerConnection::SdpType::ANSWER, answer);
```

会调用如下：

```c++
// Transport is not ready.
if (!this->transportReady)
   this->SetupTransport("server", localSdpObject);
```

> this->transportReady默认为false，SetupTransport只会调用一次，最终connectWebRtcTransport会只调用一次

会回调java层，即OnConnect也只会调用一次

```java
this->privateListener->OnConnect(dtlsParameters);
```

此时在java层拿到dtlsParameters，然后传给connectWebRtcTransport方法，将参数传给服务，最后调用

```c++
producerId =
  this->listener->OnProduce(this, track->kind(), sendData.rtpParameters, appData).get();
```

这样会回调java层的OnProduce方法，此时可以调用produce方法，将参数传给服务端

## connectWebRtcTransport  "transportId":  fa7ab4fa-a7b1-46c9-a0b4-8940cc048fb0 "role": client

up:      {"request":true,"id":4946951,"method":"connectWebRtcTransport","data":{"transportId":"fa7ab4fa-a7b1-46c9-a0b4-8940cc048fb0","dtlsParameters":{"role":"client","fingerprints":[{"algorithm":"sha-256","value":"B4:A2:E8:E9:99:83:DF:94:99:47:F2:BF:A9:D7:FC:7B:AF:40:A5:12:4D:D1:F5:39:53:8A:25:A9:53:53:7A:0F"}]}}}

down: {"response":true,"id":4946951,"ok":true,"data":{}}

下行，即在第一次收到newConsumer方法后，会有一次回调onConnect(Transport transport, String dtlsParameters)方法，此时将本地的dtlsParameters传给服务器

## produce "transportId": 5a59bce2-f102-4efb-999f-47063f0aa542  "kind": audio  "id": ab123c0c-c074-4f9c-a77e-18fefe081bba

up:      {"request":true,"id":5266424,"method":"produce","data":{"transportId":"5a59bce2-f102-4efb-999f-47063f0aa542","kind":"audio","rtpParameters":{"codecs":[{"mimeType":"audio/opus","payloadType":111,"clockRate":48000,"channels":2,"parameters":{"minptime":10,"useinbandfec":1,"sprop-stereo":1,"usedtx":1},"rtcpFeedback":[{"type":"transport-cc","parameter":""}]}],"headerExtensions":[{"uri":"urn:ietf:params:rtp-hdrext:sdes:mid","id":4,"encrypt":false,"parameters":{}},{"uri":"http://www.webrtc.org/experiments/rtp-hdrext/abs-send-time","id":2,"encrypt":false,"parameters":{}},{"uri":"http://www.ietf.org/id/draft-holmer-rmcat-transport-wide-cc-extensions-01","id":3,"encrypt":false,"parameters":{}},{"uri":"urn:ietf:params:rtp-hdrext:ssrc-audio-level","id":1,"encrypt":false,"parameters":{}}],"encodings":[{"ssrc":3781898444,"dtx":false}],"rtcp":{"cname":"tC2cANk3Ev+BV3Ar","reducedSize":true},"mid":"0"},"appData":{}}}

down: {"response":true,"id":5266424,"ok":true,"data":{"id":"ab123c0c-c074-4f9c-a77e-18fefe081bba"}}

音频produce 当java层回调OnProduce并获取对方参数时，调用produce方法给服务器，服务器返回produceId，这样就根据id，创建produce对象，并将对象存入数组

```c++
auto* producer = new Producer(
  this,
  producerListener,
  producerId,
  sendData.localId,
  sendData.rtpSender,
  track,
  sendData.rtpParameters,
  appData);

this->producers[producer->GetId()] = producer;
```



## produce "transportId": 5a59bce2-f102-4efb-999f-47063f0aa542  "kind": video  "id": 8184c2d1-814b-4182-bb9d-c9e14e351804

up:      {"request":true,"id":439972,"method":"produce","data":{"transportId":"5a59bce2-f102-4efb-999f-47063f0aa542","kind":"video","rtpParameters":{"codecs":[{"mimeType":"video/VP8","payloadType":96,"clockRate":90000,"parameters":{},"rtcpFeedback":[{"type":"goog-remb","parameter":""},{"type":"transport-cc","parameter":""},{"type":"ccm","parameter":"fir"},{"type":"nack","parameter":""},{"type":"nack","parameter":"pli"}]},{"mimeType":"video/rtx","payloadType":97,"clockRate":90000,"parameters":{"apt":96},"rtcpFeedback":[]}],"headerExtensions":[{"uri":"urn:ietf:params:rtp-hdrext:sdes:mid","id":4,"encrypt":false,"parameters":{}},{"uri":"urn:ietf:params:rtp-hdrext:sdes:rtp-stream-id","id":5,"encrypt":false,"parameters":{}},{"uri":"urn:ietf:params:rtp-hdrext:sdes:repaired-rtp-stream-id","id":6,"encrypt":false,"parameters":{}},{"uri":"http://www.webrtc.org/experiments/rtp-hdrext/abs-send-time","id":2,"encrypt":false,"parameters":{}},{"uri":"http://www.ietf.org/id/draft-holmer-rmcat-transport-wide-cc-extensions-01","id":3,"encrypt":false,"parameters":{}},{"uri":"urn:3gpp:video-orientation","id":13,"encrypt":false,"parameters":{}},{"uri":"urn:ietf:params:rtp-hdrext:toffset","id":14,"encrypt":false,"parameters":{}}],"encodings":[{"active":true,"scaleResolutionDownBy":4,"rid":"r0","scalabilityMode":"S1T3","dtx":false},{"active":true,"scaleResolutionDownBy":2,"rid":"r1","scalabilityMode":"S1T3","dtx":false},{"active":true,"scaleResolutionDownBy":1,"rid":"r2","scalabilityMode":"S1T3","dtx":false}],"rtcp":{"cname":"","reducedSize":true},"mid":"2"},"appData":{}}}

down: {"response":true,"id":439972,"ok":true,"data":{"id":"8184c2d1-814b-4182-bb9d-c9e14e351804"}}

视频produce 当java层回调OnProduce并获取对方参数时，调用produce方法给服务器，服务器返回produceId，这样就根据id，创建produce对象，并将对象存入数组



## newPeer "id": ljdzho0b

down: {"notification":true,"method":"newPeer","data":{"id":"ljdzho0b","displayName":"Steenee","device":{"flag":"chrome","name":"Chrome","version":"87.0.4280.141"}}}

后台通知本地加入的用户信息，包括id、名称等

## newConsumer "peerId": ljdzho0b "producerId": 711a4beb-2330-4b04-bdf5-a7c9bdb42976 ,"id": ab70219f-1b61-49cc-899f-2b091e8fe7b0 ,"kind": audio

down: {"request":true,"id":1343647,"method":"newConsumer","data":{"peerId":"ljdzho0b","producerId":"711a4beb-2330-4b04-bdf5-a7c9bdb42976","id":"ab70219f-1b61-49cc-899f-2b091e8fe7b0","kind":"audio","rtpParameters":{"codecs":[{"mimeType":"audio/opus","payloadType":100,"clockRate":48000,"channels":2,"parameters":{"minptime":10,"useinbandfec":1,"sprop-stereo":1,"usedtx":1},"rtcpFeedback":[]}],"headerExtensions":[{"uri":"urn:ietf:params:rtp-hdrext:sdes:mid","id":1,"encrypt":false,"parameters":{}},{"uri":"http://www.webrtc.org/experiments/rtp-hdrext/abs-send-time","id":4,"encrypt":false,"parameters":{}},{"uri":"urn:ietf:params:rtp-hdrext:ssrc-audio-level","id":10,"encrypt":false,"parameters":{}}],"encodings":[{"ssrc":599222400}],"rtcp":{"cname":"ZWirKcP/V3id5BuT","reducedSize":true,"mux":true},"mid":"0"},"type":"simple","appData":{"peerId":"ljdzho0b"},"producerPaused":false}}

up:      {"response":true,"id":1343647,"ok":true,"data":{}}

当收到服务发来的newConsumer方法时，说明这时候有新的人加入房间，并且根据这些参数，创建一个消费者对象，并且缓存到RecvTransport对象的consumers数组中，如下

```c++
// May throw.
auto recvData = this->recvHandler->Receive(id, kind, rtpParameters);

auto* consumer = new Consumer(
  this,
  consumerListener,
  id,
  recvData.localId,
  producerId,
  recvData.rtpReceiver,
  recvData.track,
  *rtpParameters,
  appData);

this->consumers[consumer->GetId()] = consumer;
```

在Receive方法中会完成sdp操作，即完成跟sfu的udp连接

```c++
this->pc->SetRemoteDescription(PeerConnection::SdpType::OFFER, offer);
auto answer         = this->pc->CreateAnswer(options);
this->pc->SetLocalDescription(PeerConnection::SdpType::ANSWER, answer);
```

并且最后会调用

```c++
if (!this->transportReady)
   this->SetupTransport("client", localSdpObject);
```

会调用如下：

```c++
this->privateListener->OnConnect(dtlsParameters);
```

即会只调用一次OnConnect方法，这个方法回调我们可以调用connectWebRtcTransport

## newConsumer  "peerId": ljdzho0b ,"producerId": 08f9a3a4-4c5d-43b9-b0c0-2dd651c6fdab , "id": f7eecedd-3bb3-4c26-8da6-d2ed64eb34f1 ,"kind": video

down: {"request":true,"id":8941600,"method":"newConsumer","data":{"peerId":"ljdzho0b","producerId":"08f9a3a4-4c5d-43b9-b0c0-2dd651c6fdab","id":"f7eecedd-3bb3-4c26-8da6-d2ed64eb34f1","kind":"video","rtpParameters":{"codecs":[{"mimeType":"video/VP8","payloadType":101,"clockRate":90000,"parameters":{},"rtcpFeedback":[{"type":"transport-cc","parameter":""},{"type":"ccm","parameter":"fir"},{"type":"nack","parameter":""},{"type":"nack","parameter":"pli"}]},{"mimeType":"video/rtx","payloadType":102,"clockRate":90000,"parameters":{"apt":101},"rtcpFeedback":[]}],"headerExtensions":[{"uri":"urn:ietf:params:rtp-hdrext:sdes:mid","id":1,"encrypt":false,"parameters":{}},{"uri":"http://www.webrtc.org/experiments/rtp-hdrext/abs-send-time","id":4,"encrypt":false,"parameters":{}},{"uri":"http://www.ietf.org/id/draft-holmer-rmcat-transport-wide-cc-extensions-01","id":5,"encrypt":false,"parameters":{}},{"uri":"urn:3gpp:video-orientation","id":11,"encrypt":false,"parameters":{}},{"uri":"urn:ietf:params:rtp-hdrext:toffset","id":12,"encrypt":false,"parameters":{}}],"encodings":[{"ssrc":935534581,"rtx":{"ssrc":246460922},"scalabilityMode":"S3T3"}],"rtcp":{"cname":"ZWirKcP/V3id5BuT","reducedSize":true,"mux":true},"mid":"1"},"type":"simulcast","appData":{"peerId":"ljdzho0b"},"producerPaused":false}}

up:      {"response":true,"id":8941600,"ok":true,"data":{}}

同上，是视频，即可拿到videoTrack，并且在SurfaceViewRenderer中显示了



## newPeer "id": obwogklw

down: {"notification":true,"method":"newPeer","data":{"id":"obwogklw","displayName":"Munna","device":{"flag":"chrome","name":"Chrome","version":"87.0.4280.141"}}}

down: {"notification":true,"method":"activeSpeaker","data":{"peerId":"d3qry1lj","volume":-51}}

同上

## newConsumer "peerId": obwogklw ,"producerId": 72cfae14-0369-4beb-9ecf-7aba0edabb34 ,"id": 01c42133-83db-4885-a094-e2bcf2f1d25c ,"kind": audio

down: {"request":true,"id":5069692,"method":"newConsumer","data":{"peerId":"obwogklw","producerId":"72cfae14-0369-4beb-9ecf-7aba0edabb34","id":"01c42133-83db-4885-a094-e2bcf2f1d25c","kind":"audio","rtpParameters":{"codecs":[{"mimeType":"audio/opus","payloadType":100,"clockRate":48000,"channels":2,"parameters":{"minptime":10,"useinbandfec":1,"sprop-stereo":1,"usedtx":1},"rtcpFeedback":[]}],"headerExtensions":[{"uri":"urn:ietf:params:rtp-hdrext:sdes:mid","id":1,"encrypt":false,"parameters":{}},{"uri":"http://www.webrtc.org/experiments/rtp-hdrext/abs-send-time","id":4,"encrypt":false,"parameters":{}},{"uri":"urn:ietf:params:rtp-hdrext:ssrc-audio-level","id":10,"encrypt":false,"parameters":{}}],"encodings":[{"ssrc":214825094}],"rtcp":{"cname":"+2Bw98Gmhb0hxeLZ","reducedSize":true,"mux":true},"mid":"2"},"type":"simple","appData":{"peerId":"obwogklw"},"producerPaused":false}}

up:      {"response":true,"id":5069692,"ok":true,"data":{}}

同上



## newConsumer "peerId": obwogklw ,"producerId": 3b63efa1-3f84-4baf-abca-0baec611ecb2 ,"id": 9a93c8b7-9e52-45b1-a6ce-d1a930b2549a , "kind": video

down: {"request":true,"id":1616965,"method":"newConsumer","data":{"peerId":"obwogklw","producerId":"3b63efa1-3f84-4baf-abca-0baec611ecb2","id":"9a93c8b7-9e52-45b1-a6ce-d1a930b2549a","kind":"video","rtpParameters":{"codecs":[{"mimeType":"video/VP8","payloadType":101,"clockRate":90000,"parameters":{},"rtcpFeedback":[{"type":"transport-cc","parameter":""},{"type":"ccm","parameter":"fir"},{"type":"nack","parameter":""},{"type":"nack","parameter":"pli"}]},{"mimeType":"video/rtx","payloadType":102,"clockRate":90000,"parameters":{"apt":101},"rtcpFeedback":[]}],"headerExtensions":[{"uri":"urn:ietf:params:rtp-hdrext:sdes:mid","id":1,"encrypt":false,"parameters":{}},{"uri":"http://www.webrtc.org/experiments/rtp-hdrext/abs-send-time","id":4,"encrypt":false,"parameters":{}},{"uri":"http://www.ietf.org/id/draft-holmer-rmcat-transport-wide-cc-extensions-01","id":5,"encrypt":false,"parameters":{}},{"uri":"urn:3gpp:video-orientation","id":11,"encrypt":false,"parameters":{}},{"uri":"urn:ietf:params:rtp-hdrext:toffset","id":12,"encrypt":false,"parameters":{}}],"encodings":[{"ssrc":590915094,"rtx":{"ssrc":313135943},"scalabilityMode":"S3T3"}],"rtcp":{"cname":"+2Bw98Gmhb0hxeLZ","reducedSize":true,"mux":true},"mid":"3"},"type":"simulcast","appData":{"peerId":"obwogklw"},"producerPaused":false}}

up:      {"response":true,"id":1616965,"ok":true,"data":{}}

同上



----

## 流程

createWebRtcTransport  调用一次、获取到服务的iceParameters、iceCandidates、dtlsParameters 推流

createWebRtcTransport  调用一次、获取到服务的iceParameters、iceCandidates、dtlsParameters 拉流

produce ->   (只调用一次)onConnect ->   (只调用一次)connectWebRtcTransport 传本地dtlsParameters ->  每次 sdp 设置  -> onProduce -> 通过服务获取produceId ->c++创建Produce对象

newConsumer -> consume  -> (只调用一次)OnConnect ->  connectWebRtcTransport 传本地dtlsParameters -> 每次sdp 设置