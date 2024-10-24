
### 一、STOMP 简介


如果直接使用 WebSocket 会非常累，就像用 Socket 编写 Web 应用。没有高层级的交互协议，就需要我们定义应用间所发消息的语义，还需要确保连接的两端都能遵循这些语义。


如 HTTP 在 TCP 套接字之上添加了请求\-响应模型层一样，STOMP 是在 WebSocket 之上提供了基于帧的线路格式层，用来定义消息的语义。


与 HTTP 请求和响应类似，STOMP 帧由命令、一个或多个头信息以及负载组成。像下面这段，就是发送数据的一个 STOMP 帧：



```
SEND
transaction:tx-0
destination:/app/hello
content-length:20

{"message":"Hello!"}

```

在这个示例中，STOMP 命令是 send，表明会发送一些内容。紧接着是三个头信息：一个表示消息的的事务机制，一个用来表示消息要发送到哪里的目的地，另外一个则包含了负载的大小。然后，紧接着是一个空行，STOMP 帧的最后是负载内容。


### 二、服务端实现


#### 1、启用STOMP功能


STOMP 的消息根据前缀的不同分为三种。如下，以 `/app` 开头的消息都可以路由到带有 @Mapping 注解的方法中；以`/topic` 开头的消息都会发送到 STOMP 代理中，根据你所选择的 STOMP 代理不同，目的地的可选前缀也会有所限制；以 `/user` 开头的消息会将消息重路由到某个用户独有的目的地上。


添加依赖



```
<dependency>
    <groupId>org.noeargroupId>
    <artifactId>solon-net-stompartifactId>
dependency>

```

添加端点监听，并设定 broker 目的地前缀



```
@ServerEndpoint("/demo")
public class DemoStompBroker extends StompBroker {
    public DemoStompBroker() {
        this.setBrokerDestinationPrefixes("/topic/");
    }
}

```

#### 2、处理来自客户端的STOMP消息


服务端处理客户端发来的 STOMP 消息，主要用的是 `@Mapping` 注解（和 MVC 开发一样），也可以增加 `@Message` 方式限有定注解。如下：



```
@Message //如果不加，同时匹配 http 及其它请求
@Mapping("/app/marco")
@To("*:/topic/marco")
public Shout greeting(Shout shout) throws Exception {
    log.debug("接收到消息：" + shout.getMessage());
    Shout s = new Shout();
    s.setMessage("Polo!");
    return s;
}

```

2\.1 `@Mapping` 指定目的地是 `/app/marco`（我们将其约定为应用的目的地前缀）。


2\.2 方法接收一个 Shout 参数，因为 Solon 的执行器根据内容类型自动会将 STOMP 消息的负载转换为 Shout 对象。


2\.3 尤其注意，这个处理器方法有一个返回值，这个返回值并不是返回给客户端的，而是转发给消息代理的，如果客户端想要这个返回值的话，只能从消息代理订阅。`@To` 注解重写了消息代理的目的地，如果不指定`@To`，帧所发往的目的地会与触发处理器方法的目的地相同。


2\.4 如果客户端就是想要服务端直接返回消息呢？听起来不就是HTTP做的事情！即使这样，STOMP 仍然为这种一次性的响应提供了支持，用的还是`@Mapping` 注解，与HTTP不同的是，这种请求\-响应模式是异步的...



```
@Message
@Mapping("/app/getShout")
public Shout getShout(){
   Shout shout = new Shout();
   shout.setMessage("Hello STOMP");
   return shout;
}

```

#### 3、发送消息到客户端


3\.1 在应用的任意地方发送消息


使用 StompEmitter 接口，可以实现自由的向任意目的地发送消息。



```
@Inject
private StompEmitter stompEmitter;

/**
* 通过 http 接口，广播消息
*/
@Http
@Mapping("/broadcastShout")
public void broadcast(Context ctx, Shout shout) {
    String json = ctx.renderAndReturn(shout); //渲染数据
    stompEmitter.sendTo("/topic/shouts", json);
}

```

3\.2 更多发送消息的方式


如果消息只想发送给特定的用户呢？或者发给当前用户？或者所有订阅用户？solon\-net\-stomp 给了两种方式来实现这种功能：


* 一种是 StompEmitter 接口的 sendTo 方法。
* 一种是 基于 @To 注解。




| StompEmitter 接口 | 对应的 `@To` 注解 | 说明 |
| --- | --- | --- |
|  | `@To("target:destination?")` | To 注解表达式（stomp 请求时有效） |
|  |  |  |
| sendToSession | `@To(".:/...")` 或 `@To(".")` | 发给当前客户端订阅者 |
| sendToUser | `@To("user:/...")` 或 `@To("user")` | 发给特定用户订阅者 |
| sendTo | `@To("*:/...")` 或 `@To("*")` | 发给代理，再转发给所有订阅者 |


#### 4、处理消息异常


在处理消息的时候，有可能会出错并抛出异常。因为STOMP消息异步的特点，发送者可能永远也不会知道出现了错误。可以调整端点监听，添加 StompListener 实现。



```
@ServerEndpoint("/demo")
public class DemoStompBroker extends StompBroker implements StompListener{
    public DemoStompBroker(){
        //可选：添加鉴权监听器（此示例，用本类实现监听）
        this.addListener(this);
        this.setBrokerDestinationPrefixes("/topic/");
    }
    
    @Override
    public void onError(StompSession session, Throwable error) {
        //可选：如果出错，反馈给客户端（比如用 "/user/app/errors"）
        getEmitter().sendToSession(session,
                "/user/app/errors",
                new Message(error.getMessage()));
    }
}

```

### 三、客户端


STOMP 可以使用 stomp.js。接口参考： [https://stomp\-js.github.io/api\-docs/latest/classes/Client.html](https://github.com)


#### 1、创建连接并订阅



```
let stomp = new StompJs.Client({
    brokerURL: "ws://127.0.0.1:8080/demo?user=user01",
    onConnect: function (frame) {
        stomp.subscribe("/topic/marco", function (message) {
            let obj = JSON.parse(message.body);
            console.log("订阅的服务端消息：" + obj.message);
        });
        
        stomp.subscribe("/app/getShout", function (message) {
            let obj = JSON.parse(message.body);
            console.log("订阅的服务端应胜消息：" + obj.message);
        });
        
        stomp.subscribe("/user/app/errors", function (message) {
            console.log("订阅的服务端返回的异常消息：" + message.body);
        });
    }
});

```

#### 2、发送消息



```
stomp.publish({
    destination: "/app/marco",
    headers: {"content-type": "text/json"},
    body: JSON.stringify({"message": "Marco!"})
});

```

 本博客参考[西部世界官网](https://tianchuang88.com)。转载请注明出处！
