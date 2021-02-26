# android webrtc wifi切4G的解决方案

## 测试问题描述：在单聊场景下的音视频通话中，从wifi切换到4g百分百会出现异常，而从4g切换到wifi正常

- 分析原因：主要是由于wifi与4g同时开的时候，进入视频会议，走的是wifi的通道，当wifi关掉的时候，这时会切换到4g通道，这时websocket会回调onclose,这时要自己进行重连，即出现非正常关闭时，要进行关闭重连操作

```java
        @Override
        public void onClose(int code, String reason, boolean remote) {
            //NORMAL = 1000;
            //ABNORMAL_CLOSE = 1006;
            //NEVER_CONNECTED = -1
            if(code != CloseFrame.NORMAL) {
                tryTime--;
                if(tryTime > 0) {
                    executor.execute(() -> {
                        boolean reconnectSuccess;
                        try {
                            int i = 5;
                            while (!NetworkUtil.isNetworkIsAvailable() && i > 0) {
                                i--;
                                SystemClock.sleep(200);
                            }
                            SystemClock.sleep(200);
                            reconnectSuccess = ws.reconnectBlocking();
                            Log.e(TAG, "tryTime = " + tryTime);
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                            reconnectSuccess = false;
                        }
                        Log.i(TAG, "to reconnect result = " + reconnectSuccess);

                        if(reconnectSuccess) {
                            events.onReconnectRoom();
                        }
                    });
                    return;
                }
            }

            Log.i(TAG, "onClose state = " + state);
            synchronized (closeEventLock) {
                closeEvent = true;
                closeEventLock.notify();
            }
            handler.post(() -> {
                if (state != WebSocketConnectionState.CLOSED) {
                    state = WebSocketConnectionState.CLOSED;
                    events.onWebSocketClose();
                }
            });
        }
```

- 以为问题解决，实际这时看到对方的视频卡住不动了，经过定位分析，由于自己发布流到服务端和从服务端订阅别人的流，走的都是p2p，通过分析流量消耗，知道自己发出去的流，对方是能正常看到的，但却看不到对方的流了，而且自己这时也没有下行流量，分析可知，订阅的时候，这时本地没有流量消耗，但是此时重新打开wifi却又能正常看到对方了，通过分析得知，发布流到服务器的时候，这时服务器接收的ip跟端口没有变化，但是订阅的时候，本地是由wifi切到4g，这时服务器推送到本地的ip跟端口肯定发生变化，所以导致没有收到流，但服务器却不知ip端口发生变化了，经过验证，需要将订阅的流关闭，然后再创建一个，这时候就能正常了

- 调用webrtc的peerConnection的dispose，将对应的流通话关闭，而后再重新打开一个

```java
          peerConnection.dispose();
          peerConnection = null;
```

- 创建一个新的连接通道

```java
PeerConnection peerConnection = factory.createPeerConnection(rtcConfig, pcObserver);
```

- 然后再重新调用janus的leave命令，取消订阅流，再调用attach命令，进行重新订阅，本以为问题解决，但是发现执行leave命令的时候，后台能够正常返回ack的同步命令，但是却收不到异步命令，后台服务通话抓包发现，确实已经发送了异步命令了，通过对所抓包的进一步的分析，发现客户端用4g对应的ip去请求，这时服务端能够正常的将同步消息返回到4g对应的ip上，但是却将异步消息发送到wifi对应的ip上，导致客户端无法正常收到异步消息，因为此时wifi已经关闭了

- 后续通过更新后台janus服务，便可解决问题