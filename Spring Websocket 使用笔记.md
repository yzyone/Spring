# Spring Websocket 使用笔记 #

## 前言 ##

现在主流的web容器基本均已支持websocket，但各容器的websocket接口都不尽相同。为了统一websocket实现，便于今后在不同web容器间的移植，这里使用spring websocket框架集成。

## spring依赖 ##

添加spring所需依赖


```
   <spring.framework.version>4.2.1.RELEASE</spring.framework.version>
   <dependency><!-- spring-core -->
        <groupId>org.springframework</groupId>
        <artifactId>spring-core</artifactId>
        <version>${spring.framework.version}</version>
    </dependency>
    <dependency><!-- spring-beans -->
        <groupId>org.springframework</groupId>
        <artifactId>spring-beans</artifactId>
        <version>${spring.framework.version}</version>
    </dependency>
    <dependency><!-- spring-context -->
        <groupId>org.springframework</groupId>
        <artifactId>spring-context</artifactId>
        <version>${spring.framework.version}</version>
    </dependency>
    <dependency><!-- spring-context-support -->
        <groupId>org.springframework</groupId>
        <artifactId>spring-context-support</artifactId>
        <version>${spring.framework.version}</version>
    </dependency>
    <dependency><!-- spring-aop -->
        <groupId>org.springframework</groupId>
        <artifactId>spring-aop</artifactId>
        <version>${spring.framework.version}</version>
    </dependency>
    <dependency><!-- spring-expression -->
        <groupId>org.springframework</groupId>
        <artifactId>spring-expression</artifactId>
        <version>${spring.framework.version}</version>
    </dependency>
    <dependency><!-- spring-aspects -->
        <groupId>org.springframework</groupId>
        <artifactId>spring-aspects</artifactId>
        <version>${spring.framework.version}</version>
    </dependency>
    <dependency><!-- spring-web -->
        <groupId>org.springframework</groupId>
        <artifactId>spring-web</artifactId>
        <version>${spring.framework.version}</version>
    </dependency>
    <dependency><!-- spring-websocket -->
        <groupId>org.springframework</groupId>
        <artifactId>spring-websocket</artifactId>
        <version>${spring.framework.version}</version>
    </dependency>
```


## websocket依赖 ##



对于jetty，添加如下依赖



```
    <dependency>
        <groupId>org.eclipse.jetty.websocket</groupId>
        <artifactId>websocket-server</artifactId>
        <version>9.3.3.v20150827</version>
        <scope>provided</scope>
    </dependency>
```

对于tomcat，添加如下依赖

```
    <dependency>
        <groupId>org.apache.tomcat.embed</groupId>
        <artifactId>tomcat-embed-websocket</artifactId>
        <version>8.0.23</version>
        <scope>provided</scope>
    </dependency>
```

对于上述jetty或者tomcat依赖，需要注意依赖版本与容器版本的兼容性，另一方面需要注意将依赖的scope设置为provided

## 添加Handler ##

添加HandshakeInterceptor

```
/**
 * <pre>
 * 豆瓣电台websocket握手接口实现
 * </pre>
 *
 * @author  ManerFan 2015年9月7日
 */
@Component
public class DoubanFMHandshakeInterceptor extends HttpSessionHandshakeInterceptor {

    @Override
    public boolean beforeHandshake(ServerHttpRequest request, ServerHttpResponse response,
            WebSocketHandler wsHandler, Map<String, Object> attributes) throws Exception {
        // 握手前的处理逻辑
        return super.beforeHandshake(request, response, wsHandler, attributes);
    }

    @Override
    public void afterHandshake(ServerHttpRequest request, ServerHttpResponse response,
            WebSocketHandler wsHandler, Exception ex) {
        // 握手后的处理逻辑
        super.afterHandshake(request, response, wsHandler, ex);
    }

}
```

spring提供了多种HandshakeInterceptor实现，可以根据实际需要在其基础上实现自己的逻辑。当然，也可以直接使用spring的默认HandshakeInterceptor

添加WebSocketHandler

```
/**
 * <pre>
 * 豆瓣FM websocket消息处理
 * </pre>
 *
 * @author ManerFan 2015年9月7日
 */
@Component
public class DoubanFMWebSocketHandler extends TextWebSocketHandler {

    /**
     * logger
     */
    private static final Logger LOGGER = CommonLogger.getLogger();

    /**
     * 记录连接上的所有session
     */
    private Map<String, WebSocketSession> sessionMap = Collections
            .synchronizedMap(new HashMap<String, WebSocketSession>());

    @Override
    public void afterConnectionEstablished(WebSocketSession session) throws Exception {
        // 建立连接之后
        LOGGER.info("[{}:{}] has bean connected", session.getUri(), session.getId());
        sessionMap.put(session.getId(), session);
    }

    @Override
    protected void handleTextMessage(WebSocketSession session, TextMessage message)
            throws Exception {
        // 收到消息之后
        String payload = message.getPayload();
        Signalling signalling = JacksonUtils.instance().readValue(payload, TYPE_REFERENCE);
        if (null == signalling) {
            LOGGER.warn("Receive an Empty Payload [?][{}] from [{}:{}].", payload, session.getUri(),
                    session.getId());
            return;
        }

        switch (signalling.getSignal()) {
            case C_START: /* 播放某一频道 */
                doStart(session, signalling.getMessage());
                break;
            case C_STOP: /* 停止收听 */
                doStop();
                break;
            case C_SKIP: /* 跳过该首 */
                doSkip(signalling.getMessage());
                break;
            case C_RATE: /* 喜欢 */
                doRate(signalling.getMessage());
                break;
            case C_UNRATE: /* 取消喜欢 */
                doUnRate(signalling.getMessage());
                break;
            case C_BYE: /* 不再收听 */
                doBye(signalling.getMessage());
                break;
            default:
                break;
        }
    }

    @Override
    public void afterConnectionClosed(WebSocketSession session, CloseStatus status)
            throws Exception {
        // 连接断开之后
        LOGGER.info("[{}:{}] has bean closed", session.getUri(), session.getId());
        sessionMap.remove(session.getId());

        if (sessionMap.isEmpty()) {
            /* 没有客户端在线了 */
            doStop();
        }
    }
}
```

spring同样提供了TextWebSocketHandler BinaryWebSocketHandler等多种WebSocketHandler，可根据实际需要进行选择

## 初始化websocket ##

websocket的初始化，可以使用配置方式，也可以使用注解方式，关于spring websocket的使用及配置，详见websocket-doc，这里记录注解方式初始化方法

```
/**
 * <pre>
 * WebSocket初始化配置
 * 
 * <a href="http://docs.spring.io/spring/docs/current/spring-framework-reference/html/websocket.html">DOC for Spring Websocket</a>
 * </pre>
 *
 * @author  ManerFan 2015年9月7日
 */
@Configuration
@EnableWebMvc
@EnableWebSocket
public class WebSocketConfig extends WebMvcConfigurerAdapter implements WebSocketConfigurer {

    /**
     *  <beans xmlns="http://www.springframework.org/schema/beans"  
     *      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"  
     *      xmlns:websocket="http://www.springframework.org/schema/websocket"  
     *      xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd  
     *      http://www.springframework.org/schema/websocket http://www.springframework.org/schema/websocket/spring-websocket.xsd">
     * 
     *      <bean id="handler" class="com.manerfan.webapp.controller.douban.fm.websockt.FMWebSocketHandler"/>  
     *
     *      <websocket:handlers>  
     *         <websocket:mapping path="/websocket/doubanfm" handler="handler"/>  
     *         <websocket:handshake-interceptors>  
     *             <bean class="com.manerfan.webapp.controller.douban.fm.websock.FMHandshakeInterceptor"/>  
     *         </websocket:handshake-interceptors> 
     *         <!-- <websocket:sockjs/> --> 
     *      </websocket:handlers>
     *  </beans>
     */

    /** Douban FM */
    @Autowired
    private DoubanFMWebSocketHandler doubanFmWebSocketHandler;
    @Autowired
    private DoubanFMHandshakeInterceptor doubanFMHandshakeInterceptor;

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        /** Douban FM */
        registry.addHandler(doubanFmWebSocketHandler, "/websocket/doubanfm").addInterceptors(
                doubanFMHandshakeInterceptor).setAllowedOrigins("*")/*不加AllowedOrigins，有可能会全拒绝*/;
        /*registry.addHandler(doubanFmWebSocketHandler, "/sockjs/doubanfm")
                .addInterceptors(doubanFMHandshakeInterceptor).withSockJS();*/ /*用于支持SockJS*/
    }

    /**
     * Each underlying WebSocket engine exposes configuration properties 
     * that control runtime characteristics such as the size of message buffer sizes, 
     * idle timeout, and others.
     */

    /**
     * For Tomcat, WildFly, and GlassFish add a ServletServerContainerFactoryBean
     * to your WebSocket Java config:
     */
    /*@Bean
    public ServletServerContainerFactoryBean createWebSocketContainer() {
        ServletServerContainerFactoryBean container = new ServletServerContainerFactoryBean();
        container.setMaxTextMessageBufferSize(8192);
        container.setMaxBinaryMessageBufferSize(8192);
        return container;
    }*/

    /**
     * For Jetty, you’ll need to supply a pre-configured Jetty WebSocketServerFactory
     * and plug that into Spring’s DefaultHandshakeHandler through your WebSocket Java config:
     */
    @Bean
    public DefaultHandshakeHandler handshakeHandler() {

        WebSocketPolicy policy = new WebSocketPolicy(WebSocketBehavior.SERVER);
        policy.setInputBufferSize(8192); /* 设置消息缓冲大小 */
        policy.setIdleTimeout(600000); /* 10分钟read不到数据的话，则断开该客户端 */

        return new DefaultHandshakeHandler(
                new JettyRequestUpgradeStrategy(new WebSocketServerFactory(policy)));
    }

}
```

setAllowedOrigins用于约束允许建立连接的客户端
withSockJS用于提供SockJS支持

至此，websocket后台框架便搭建完成
javascript可使用如下方式与后台建立连接进行通信

```
/* 建立连接 */
var ws = new WebSocket("ws://localhost/rasp/websocket/doubanfm");
ws.onopen = function() { /* 连接成功时 */
    ws.send(JSON.stringify({signal:'C_START', message:'179'}));
};
ws.onmessage = function (evt){ /* 收到消息时 */
    var received_msg = evt.data;
    console.info("Message is received... " + received_msg);
};
ws.onclose = function() { /* 连接断开时 */
    console.warn("Connection is closed...");
};
ws.onerror = function(e) { /* 出现错误时 */
    console.error(e);
}
```

## 开发时遇到的异常及解决办法 ##

    No suitable default RequestUpgradeStrategy found

```
严重: Context initialization failed
org.springframework.beans.factory.BeanCreationException: Error creating bean with name ‘webSocketHandlerMapping’ defined in class path resource [org/springframework/web/socket/config/annotation/DelegatingWebSocketConfiguration.class]: Bean instantiation via factory method failed; nested exception is org.springframework.beans.BeanInstantiationException: Failed to instantiate [org.springframework.web.servlet.HandlerMapping]: Factory method ‘webSocketHandlerMapping’ threw exception; nested exception is java.lang.IllegalStateException: No suitable default RequestUpgradeStrategy found
at org.springframework.beans.factory.support.ConstructorResolver.instantiateUsingFactoryMethod(ConstructorResolver.java:599)
at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.instantiateUsingFactoryMethod(AbstractAutowireCapableBeanFactory.java:1111)
at …
Caused by: org.springframework.beans.BeanInstantiationException: Failed to instantiate [org.springframework.web.servlet.HandlerMapping]: Factory method ‘webSocketHandlerMapping’ threw exception; nested exception is java.lang.IllegalStateException: No suitable default RequestUpgradeStrategy found
at org.springframework.beans.factory.support.SimpleInstantiationStrategy.instantiate(SimpleInstantiationStrategy.java:189)
at …
Caused by: java.lang.IllegalStateException: No suitable default RequestUpgradeStrategy found
at org.springframework.web.socket.server.support.DefaultHandshakeHandler.initRequestUpgradeStrategy(DefaultHandshakeHandler.java:127)
at …
```

出现上述异常，是由于缺少RequestUpgradeStrategy
对于jetty，需要添加如下依赖

```
    <dependency>
        <groupId>org.eclipse.jetty.websocket</groupId>
        <artifactId>websocket-server</artifactId>
        <version>9.3.3.v20150827</version>
        <scope>provided</scope>
    </dependency>
```

对于tomcat，需要添加如下依赖

```
    <dependency>
        <groupId>org.apache.tomcat.embed</groupId>
        <artifactId>tomcat-embed-websocket</artifactId>
        <version>8.0.23</version>
        <scope>provided</scope>
    </dependency>
```

```
java.lang.ClassCastException: org.eclipse.jetty.server.HttpConnection cannot be cast to org.eclipse.jetty.server.HttpConnection
```

页面报如下异常

    WebSocket connection to ‘ws://localhost/rasp/websocket/doubanfm’ failed: Error during WebSocket handshake: Unexpected response code: 500

后台报如下异常

```
WARN:oejs.ServletHandler:qtp1237514926-20:
org.springframework.web.util.NestedServletException: Request processing failed; nested exception is org.springframework.web.socket.server.HandshakeFailureException: Uncaught failure for request http://localhost:80/rasp/websocket/doubanfm; nested exception is java.lang.ClassCastException: org.eclipse.jetty.server.HttpConnection cannot be cast to org.eclipse.jetty.server.HttpConnection
at org.springframework.web.servlet.FrameworkServlet.processRequest(FrameworkServlet.java:978)
at …
Caused by:
org.springframework.web.socket.server.HandshakeFailureException: Uncaught failure for request http://localhost:80/rasp/websocket/doubanfm; nested exception is java.lang.ClassCastException: org.eclipse.jetty.server.HttpConnection cannot be cast to org.eclipse.jetty.server.HttpConnection
at org.springframework.web.socket.server.support.WebSocketHttpRequestHandler.handleRequest(WebSocketHttpRequestHandler.java:135)
at …
Caused by:
java.lang.ClassCastException: org.eclipse.jetty.server.HttpConnection cannot be cast to org.eclipse.jetty.server.HttpConnection
at org.eclipse.jetty.websocket.server.WebSocketServerFactory.acceptWebSocket(WebSocketServerFactory.java:182)
at …
```

出现此情况，一般是由于工程中引用的javax.servlet与web容器相关jar包冲突导致
将javax.servlet相关依赖scope置为provided即可

```
    <dependency>
        <groupId>javax.servlet</groupId>
        <artifactId>servlet-api</artifactId>
        <scope>provided</scope><!-- 只在编译的时候 -->
    </dependency>
```

	Unexpected response code: 403

页面报此异常，一般是由于websocket拒绝了此链接请求
在websocket初始化时，将setAllowedOrigins加上即可

————————————————

版权声明：本文为CSDN博主「ManerFan」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。

原文链接：https://blog.csdn.net/manerfan/article/details/48526681

