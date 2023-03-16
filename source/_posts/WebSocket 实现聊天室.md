---
title: WebSocket 实现聊天室
tag: 项目
---

## WebSockect 实现聊天室

**说明：最近二手交易课设有一个需求是实现 购买者和商品发布者有一个聊天对话的功能，类似于咸鱼的聊天对话功能吧。想到的就是 WebSocket 协议来实现，问了一个前端小伙伴，他一般使用 socketio(一个 websocket 框架)，我看了下也有 java 版的，但思考了下决定还是使用原生 websocket 来写前后端。**

#### 什么是 websocket？

这里放一个知乎的高赞回答，看完很清楚。[WebSocket 是什么原理？为什么可以实现持久连接？ - Ovear 的回答 - 知乎](https://www.zhihu.com/question/20215561/answer/40316953)

首先我们先说说大家都很了解的 Http 协议，在 B/S 开发中，我们常用这种协议来进行各种响应和处理。

他的特点就是一个 Request 和 一个 Response 而且是无状态的（想要保持状态需要间接通过 session 和 cookie）。虽然在一些不那么复杂的需求下，这样的机制已经足够了，但是一些复杂的应用场景如需要一直监听某个数据的变化就显得力不从心了。当然我们也可以使用 ajax 来轮询，但这样其实是非常低效率的，你把自己想成服务器，假设有个人（客户端）一直在你耳边叨叨（数据更新了没。。。）我想你也一定要疯掉了吧。

<div align=center><img src="https://yilin-1307688338.cos.ap-nanjing.myqcloud.com/blog/inkun.jpg" alt="img" width="200"></div>

其次 Http 协议的另一个特点，浏览器只能主动发送请求接收信息，不能被动接收服务器信息。这一点确实蛋疼，使得一旦数据有了变化我需要自己去请求，但是我又怎么知道数据什么时候更新了呢？

然鹅，websocket 的出现就可以巧妙的解决这些问题。

websocket 协由握手和数据传输构成

握手基于 HTTP 协议，然后客户端和服务端实现长连接，所以说 websocket 和 http 是有交集的。

![关系图](https://yilin-1307688338.cos.ap-nanjing.myqcloud.com/blog/20230316170428.png)

那么数据如何传输呢？只需要在服务端设立转发的服务，那么数据就可以实现从 A 客户端到 B 客户端的发送，拿聊天举例，正是这种长连接机制以及允许客户端主动接收服务端消息的机制使得聊天消息能够看上去好像在两个客户端建立了连接。其实就是服务器做了一次转发。

#### Java 怎么写服务端的 WebSocket（SpringBoot）

非常类似 Servlet，这里我们需要写**ServerEndPoint**

这里我们主要重写三个方法

1. onOpen(建立连接时自动调用)
2. onMessage(接收消息时自动调用)
3. onClose(关闭连接时自动调用)

当然还有 onError 等方法

```Java
//这个 注解类似http的map ，比如说这样你的 websocket url 就是 ws：localhost/chat
@ServerEndpoint(value = "/chat",configurator = GetHttpSessionConfigurator.class)

public class ChatEndPoint {


    /**
     *  建立连接被调
     */

    @OnOpen
    public void onOpen(Session session, EndpointConfig config){



    }

    /**
     * 接收数据被调用
     */

    @OnMessage
    public void onMessage(String message,Session session){




    }

    /**
     * 关闭连接调用
     */
    @OnClose
    public void onClose(Session session){



    }



}

```

这里需要注意，上面 session 指的是 websocket 的 session，不是 http 的 session，也是用来标识每一个长连接的对象，看到这里聪明的小伙伴应该能想到实现消息转发可以用 session 来标识每一个用户。

所以我们想要实现聊天消息转发可以使用 map 来存储 websocket 的 session。这里我存储 EndPoint 实例类似，因为我们可以使用 endpoint 实例来获取 session 对象，记住每一个用户进行一次 websocket 长连接，就会创建一个 endpoint 对象。

```Java
 /**
     * 用来存储每个客户端对象对应的ChatEndpoint对象 key 是uid
     */
    private static Map<String,ChatEndPoint> users = new ConcurrentHashMap<>();
```

说到这里有小伙伴想问了，项目的一些数据存储在 HttpSession 中既然 websocket 是基于 http 的，那么我能不能取出 Httpsession 在 Endpoint 里使用啊？答案是可以的，只需要在 springboot 配置中在注入对象前 ServerEndpointConfig 放入这个 httpsession 就可以了

配置：

```Java
@Configuration

public class WebSocketConfig {
    @Bean
    public ServerEndpointExporter serverEndpointExporter(){
        return new ServerEndpointExporter();
    }
}



public class GetHttpSessionConfigurator extends ServerEndpointConfig.Configurator {
    @Override
    public void modifyHandshake(ServerEndpointConfig sec, HandshakeRequest request, HandshakeResponse response) {
        HttpSession httpSession = (HttpSession)request.getHttpSession();

        //将httpsession存到配置对象

        sec.getUserProperties().put(HttpSession.class.getName(),httpSession);
    }
}

@ServerEndpoint(value = "/chat",configurator = GetHttpSessionConfigurator.class)

```

获取：

```Java
 @OnOpen
    public void onOpen(Session session, EndpointConfig config){

        this.session = session;

        //获取Httpssion

       HttpSession httpSession = (HttpSession) 	config.getUserProperties().get(HttpSession.class.getName());
       }
```

#### 前端怎么写 WebSocket

前端其实也类似，写起来更简单，只需要 new 一个 websocket 对象就能够实现连接。

```Typescript
let ws: any = null;

export default {
	//连接
  connect() {
    ws = new WebSocket('ws://localhost:8081/api/chat');
  },
	//获取实例
  getWs() {
    return ws;
  },
  //关闭连接，删除实例

  removeWs() {
    ws.close();
  },
};

```

```Typescript
 	//调用方法
 	ws.connect();

    ws.getWs().onopen = function () {

    };

    ws.getWs().onmessage = function (evt) {
    }
    ws.getWs().onclose = function () {

    };
```

#### 聊天怎么实现

思路

1.  每一个客户端与服务端建立连接就将 EndPoint 实例存入 userHashMap（这里使用静态的）中。
2.  客户端断开连接，就将此用户从 userHashMap 去除，所以 userHashMap 始终存储在线用户
3.  客户端发消息，消息内容需要有发消息人，收消息人，内容，时间，封装成一个对象。
4.  服务端 onmessage 接收到就检查用户里 userHashMap 有没有此人（有表示在线），有就找到 session 直接转发给他，没有需要暂时存储到 chatsHashMap，存消息列表 。
5.  那么我们还要修改 1 步骤，这里连接上就要检查 chatsHashMap 有没有自己的消息，有就转发给自己并去除 chatsHashMap 的消息，这样一来就实现了离线和在线用户的聊天功能。

后端全部代码：

```java

@Component
@ServerEndpoint(value = "/chat",configurator = GetHttpSessionConfigurator.class)

public class ChatEndPoint {


    /**
     * 用来存储每个客户端对象对应的ChatEndpoint对象 key 是uid
     */
    private static Map<String,ChatEndPoint> users = new ConcurrentHashMap<>();

    /**
     *  用来存储每个客户端对象对应的聊天记录 key 是uid value 是json
     */

    private static Map<String, ArrayList<String>> chats = new ConcurrentHashMap<>();

    /**
     * websocket session
     */

    private  Session session;


    /**
     * httpsession
     */
    private HttpSession httpSession;



    /**
     *  建立连接被调
     */

    @OnOpen
    public void onOpen(Session session, EndpointConfig config){

        this.session = session;

        //获取Httpssion

       HttpSession httpSession = (HttpSession) config.getUserProperties().get(HttpSession.class.getName());

       this.httpSession = httpSession;


       User user = (User)httpSession.getAttribute(USER_LOGIN_STATE);



        // 将当前对象存储在 容器中 key为uid

        String uid = String.valueOf(user.getUid());

       users.put(uid,this);

       // 判断并建立 暂存 聊天记录的数据结构

       if(!chats.containsKey(uid)) {

           ArrayList<String> arr = new ArrayList<>();
           System.out.println(uid+"调用了一次");
           chats.put(uid,arr);

       }
        String message = MessageUtils.getMessage(true, null,null,null, getUsers());

        System.out.println(message);
        //连接就一条广播
        broadcastAllUsers(MessageUtils.getMessage(true, null, null,null,"当前在线用户人数："+users.size()+"人"));
        broadcastAllUsers(message);

        //获取该用户暂存离线消息并推送

        ArrayList<String> chatCache = chats.get(uid);


        for(String chat:chatCache){
//            ChatEndPoint chatEndPoint = users.get(uid);
//            System.out.println(chat);
            try {
                this.session.getBasicRemote().sendText(chat);
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        chats.remove(uid);


    }

    /**
     * 获取容器里的用户
     * @return
     */
    private Set<String> getUsers(){

        return ChatEndPoint.users.keySet();

    }

    /**
     * 推送所有客户端
     */

    private void broadcastAllUsers(String message){

        Set<String> usersSet = users.keySet();
            try {

                for (String user : usersSet) {
                    ChatEndPoint chatEndPoint = users.get(user);
                    chatEndPoint.session.getBasicRemote().sendText(message);
                }} catch(IOException e){
                    e.printStackTrace();
                }


            }




    /**
     * 接收数据被调用
     */

    @OnMessage
    public void onMessage(String message,Session session){


        ObjectMapper mapper = new ObjectMapper();
        try {
            Message mess = mapper.readValue(message, Message.class);

//            System.out.println(mess);
// 消息的接收者
            String toUid = mess.getToUid();

            ChatEndPoint chatEndPoint = users.get(toUid);

            User user = (User)httpSession.getAttribute(USER_LOGIN_STATE);



            if(chatEndPoint!=null){

                // 接收用户在线直接转发



                if(user==null){

                    throw new BusinessException(ErrorCode.LOGIN_ERROR);
                }

                String message1 = MessageUtils.getMessage(false, user.getUserName(), user.getUid(),user.getPhone(), mess);

                chatEndPoint.session.getBasicRemote().sendText(message1);



            }else{

                //接收用户不在线 先暂时存储消息

                ArrayList<String> messages = chats.get(toUid);

                if(messages == null){
                    messages = new ArrayList<>();
                    chats.put(toUid,messages);

                }

                //存的就是json
                messages.add(MessageUtils.getMessage(false, user.getUserName(), user.getUid(),user.getPhone(), mess));



            }



        } catch (Exception e) {
            e.printStackTrace();
        }


    }

    /**
     * 关闭连接调用
     */
    @OnClose
    public void onClose(Session session){


        User user = (User)httpSession.getAttribute(USER_LOGIN_STATE);

        users.remove(user.getUid().toString());

        System.out.println(user.getUserName()+"下线了，当前在线人数："+users.size()+"人");



    }

    /**
     * 获取当前时间戳，秒
     */
    private String getUnix(){

        long time = System.currentTimeMillis();


        time = time / 1000;

        return String.valueOf(time);

    }


}

```

前端接收怎么存储呢？我暂时只想到存储在 sessionStorage 或者 localStorage 中，但这部分数据存储需要考虑去重和数据对应每个用户，不要疏忽了让别的用户看到了不属于自己的对话内容，那就出大问题了。整体思路挺简单的就是要细心。

#### 实现展示

![image-20221125171042690](https://yilin-1307688338.cos.ap-nanjing.myqcloud.com/blog/image-20221125171042690.png)

![image-20221125171211620](https://yilin-1307688338.cos.ap-nanjing.myqcloud.com/blog/image-20221125171211620.png)

![image-20221125171244569](https://yilin-1307688338.cos.ap-nanjing.myqcloud.com/blog/image-20221125171244569.png)

组件采用的是 react-jwchat 感觉挺好看的，各项功能都正常，就是有时候会 websocket 连接了就断开了，猜测是没写 error 处理吧。

#### 写在最后

最后看我的二手交易系统的前后端源码（websocket 源码也在里面）

[react-jwchat 聊天组件](https://gitee.com/wx_504ae56474/react-jwchat)

[二手交易系统前端](https://gitee.com/yilinyo/lkd-javee-trade-frontend)

[二手交易系统后端](https://gitee.com/yilinyo/lkd-javaee-trade-backend)
