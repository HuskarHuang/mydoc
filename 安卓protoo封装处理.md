# 安卓protoo封装处理

由于公司项目要通过websocket与server端进行通讯，并且是通过protoo进行处理，由于protoo在js有完善的封装工具，所以安卓端也需要进行封装，方便后续使用。

### protoo json格式

#### 客户端发送request方法：

```json
{
    "request": true,
    "id": 6114798,
    "method": "method1",
    "data": {
        "key1": "key1value",
        "key2": "key2value",
        "key3": "key3value"
    }
}
```

#### 服务端接收对应的response方法：

- 返回ok


```json
{
    "response": true,
    "id": 6114798,
    "ok": true,
    "data": {
        "key1": "key1value",
        "key2": "key2value",
        "key3": "key3value"
    }
}
```

> 即方法method1的请求id为6114798，服务返回对应的id

- 返回error


```json
{
    "response": true,
    "id": 6114798,
    "ok": false,
    "errorCode": -9,
    "errorReason": "sfu not found"
}
```

同样，服务端也可以给客户端发送request，客户端也要响应对应的response

其中data为客户端与服务端交互的具体数据，根据具体业务需要进行处理即可

同时服务端也可以给客户端推送通知，客户端不需要回复，格式如下：

```json
{
    "notification": true,
    "method": "peer-join",
    "data": {
        "rid": "room1",
        "uid": "64236c21-21e8-4a3d-9f80-c767d1e1d67f",
        "info": {
            "name": "zhou",
            "head": ""
        }
    }
}
```

### protoo js 使用

js工具库的使用方法如下：

#### 客户端发送request

```js
try
{
  const data = await peer.request('method1', { foo: 'bar' });
  console.log('got response data:', data);
}catch (error){
  console.error('request failed:', error);
}
```

> 发送一个同步请求，并且直接返回data

#### 客户端接收request并返回正确或者错误的应答

```js
peer.on('request', (request, accept, reject) => {
  if (something in request) 
      accept({ foo: 'bar' });
  else 
      reject(400, 'Not Here');
});
```

#### 客户端接收notification

```js
peer.on('notification', (notification) => {
  // Do something.
});
```

### protoo安卓使用

最终，安卓端封装一个peer对象，使用如下：

#### 客户端发送request

```java
try {
  String data = mProtoo.syncRequest("method1", data -> jsonPut(data, "foo", "bar"));
  Log.d(TAG, "got response data:" + data);
} catch (ProtooException e) {
  logError("request failed:", e);
}
```

#### 客户端接收request，接收notification

```java
private Protoo.Listener peerListener = new Protoo.Listener() {
      @Override
      public void onRequest(@NonNull Message.Request request, @NonNull Protoo.ServerRequestHandler handler) {
        Logger.d(TAG, "onRequest() " + request.getData().toString());
        if (true) {
           handler.accept(jsonPut(new JSONObject(), "foo", "bar"))
        } else {
           handler.reject(400, "Not Here")
        }
      }

      @Override
      public void onNotification(@NonNull Message.Notification notification) {
          // Do something.
      }
    };
```

### 安卓实现分析

#### syncRequest实现

mProtoo.syncRequest会调用下面

```java
private Observable<String> request(String method, @NonNull JSONObject data) {
  Logger.d(TAG, "request(), method: " + method);
  return Observable.create(
      emitter ->
          request(
              method,
              data,
              new ClientRequestHandler() {
                @Override
                public void resolve(String data) {
                  if (!emitter.isDisposed()) {
                    emitter.onNext(data);
                  }
                }

                @Override
                public void reject(long error, String errorReason) {
                  if (!emitter.isDisposed()) {
                    emitter.onError(new ProtooException(error, errorReason));
                  }
                }
              }));
}
```

> 使用rxJava将requset的异步转同步，最后调用emitter.onNext(data)返回data，调用emitter.onError抛出异常

request实现如下：

```java
public void request(String method, @NonNull JSONObject data, ClientRequestHandler clientRequestHandler) {
  JSONObject request = Message.createRequest(method, data);
  long requestId = request.optLong("id");
  Logger.d(TAG, String.format("request() [method:%s, data:%s]", method, data.toString()));
  String payload = mTransport.sendMessage(request);

  long timeout = (long) (1500 * (15 + (0.1 * payload.length())));
  mSends.put(
      requestId, new ClientRequestHandlerProxy(requestId, method, timeout, clientRequestHandler));
}
```

> 根据用户传的method与data生成一个request的json对象，并且通过websocket sendMessage发送出去，根据消息的长度，设置timeout超时时间，如果超时没有收到响应，会抛出异常，如果收到响应，则会删除mSends的一个值mSends.remove(response.getId())，并且根据response的ok与否回调上面request的resolve与reject

超时处理如下：

```java
mSends.remove(mRequestId);
// TODO (HaiyangWu): error code redefine. use http timeout
if (mClientRequestHandler != null) {
  mClientRequestHandler.reject(408, "request timeout");
}
```

response的处理如下：

```java
private void handleResponse(Message.Response response) {
  ClientRequestHandlerProxy sent = mSends.remove(response.getId());
  if (sent == null) {
    Logger.e(
        TAG, "received response does not match any sent request [id:" + response.getId() + "]");
    return;
  }

  sent.close();
  if (response.isOK()) {
    sent.resolve(response.getData().toString());
  } else {
    sent.reject(response.getErrorCode(), response.getErrorReason());
  }
}
```

> mSends.remove(response.getId())的同时，返回回调对象，并且根据response的ok与否回调上面request的resolve与reject

#### 客户端接收request，接收notification实现

```java
@Override
public void onMessage(Message message) {
  if (mClosed) {
    return;
  }
  Logger.d(TAG, "onMessage()");
  if (message instanceof Message.Request) {
    handleRequest((Message.Request) message);
  } else if (message instanceof Message.Response) {
    handleResponse((Message.Response) message);
  } else if (message instanceof Message.Notification) {
    handleNotification((Message.Notification) message);
  }
}
```

##### 客户端接收request

```java
mListener.onRequest(
    request,
    new ServerRequestHandler() {
      @Override
      public void accept(String data) {
        try {
          JSONObject response;
          if (TextUtils.isEmpty(data)) {
            response = Message.createSuccessResponse(request, new JSONObject());
          } else {
            response = Message.createSuccessResponse(request, new JSONObject(data));
          }
          mTransport.sendMessage(response);
        } catch (Exception e) {
          e.printStackTrace();
        }
      }

      @Override
      public void reject(long code, String errorReason) {
        JSONObject response = Message.createErrorResponse(request, code, errorReason);
        try {
          mTransport.sendMessage(response);
        } catch (Exception e) {
          e.printStackTrace();
        }
      }
    });
```

> 会将request传给上层，并且实现一个ServerRequestHandler，里面有accept与reject的实现，用户拿到这个ServerRequestHandler对象，就可以调用accept与reject方法，其中accept会根据上层应该传的data，封装成response发送给服务端，如果是reject，会由上层传code与errorReason并且发送给服务端

##### 客户端接收notification

```java
mListener.onNotification(notification);
```

> 直接回调上层的onNotification，并且将notification对象传给上层