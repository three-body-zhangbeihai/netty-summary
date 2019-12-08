# netty开发http服务案例

**HttpNettyServer**: 

```java
package com.whh.netty.http;

import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.Channel;
import io.netty.channel.ChannelFuture;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.channel.socket.nio.NioSocketChannel;

/**
 * @program: nettyPro
 * @description:
 * @author: wenyan
 * @create: 2019-12-08 14:42
 **/


public class HttpNettyServer {
    public static void main(String[] args) throws InterruptedException {
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap serverBootstrap = new ServerBootstrap();
            serverBootstrap.group(bossGroup, workerGroup)
                    .channel(NioServerSocketChannel.class)
                    .childHandler(new HttpServerInitializer());
            ChannelFuture channelFuture = serverBootstrap.bind(6668).sync();
            channelFuture.channel().closeFuture().sync();

        }finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }

    }
}
```

<br>

**HttpServerInitializer**: 

```java
package com.whh.netty.http;

import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelPipeline;
import io.netty.channel.socket.SocketChannel;
import io.netty.handler.codec.http.HttpServerCodec;

/**
 * @program: nettyPro
 * @description:
 * @author: wenyan
 * @create: 2019-12-08 15:07
 **/


public class HttpServerInitializer extends ChannelInitializer<SocketChannel> {
    protected void initChannel(SocketChannel ch) throws Exception {
        //向管道加入处理器
        //得到管道
        ChannelPipeline pipeline = ch.pipeline();

        //加入一个netty提供的httpServerCodec codec => [coder - decoder]
        //1. HTTPServerCodec 是netty提供的处理http的编解码器
        pipeline.addLast("MyHttpServerCodec",new HttpServerCodec());
        //2. 增加一个自定义的handler
        pipeline.addLast("MyHttpServerHandler",new HttpServerHandler());
        System.out.println("ok....");
    }
}
```

<br>

**HttpServerHandler**: 

```java
package com.whh.netty.http;

import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelHandler;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.SimpleChannelInboundHandler;
import io.netty.handler.codec.http.*;
import io.netty.util.CharsetUtil;

import java.net.URI;

/**
 * @program: nettyPro
 * @description:
 * @author: wenyan
 * @create: 2019-12-08 14:51
 **/

/**
 * 1. SimpleChannelInboundHandler 是 ChannelInboundHandlerAdapter的子类
 * 2. HTTPObject 是客户端与服务器端相互通讯的数据被封装成 HTTPObject
 */
public class HttpServerHandler extends SimpleChannelInboundHandler {
    /**
     * 读取客户端数据
     * @param ctx
     * @param msg
     * @throws Exception
     */
    protected void channelRead0(ChannelHandlerContext ctx, Object msg) throws Exception {
        System.out.println("对应的channel=" + ctx.channel() + "pipeline=" + ctx.pipeline()
        + "通过pipeline获取channel" + ctx.pipeline().channel());

        System.out.println("当前pipeline获取channel" + ctx.handler());

        //判断msg是不是httprequest请求
        if(msg instanceof HttpRequest){
            System.out.println("ctx 类型:" + ctx.getClass());
            System.out.println("pipeline hashcode=" + ctx.pipeline().hashCode()+"HttpServerHandler hash="+this.hashCode());
            System.out.println("msg类型:" + msg.getClass());
            System.out.println("客户端地址:" + ctx.channel().remoteAddress());

            HttpRequest httpRequest = (HttpRequest)msg;
            //获取uri，过滤指定的资源
            URI uri = new URI(httpRequest.uri());
            if("/favicon.ico".equals(uri.getPath())){
                System.out.println("请求了 favicon.ico，不做响应");
            }
            //回复信息给浏览器（http协议）
            ByteBuf content = Unpooled.copiedBuffer("你好，我是服务端",CharsetUtil.UTF_8);

            //构造一个http响应，即httpresponse
            FullHttpResponse response = new DefaultFullHttpResponse(HttpVersion.HTTP_1_1, HttpResponseStatus.OK,content);

            response.headers().set(HttpHeaderNames.CONTENT_TYPE,"text/plain");
            response.headers().set(HttpHeaderNames.CONTENT_LENGTH,content.readableBytes());

            //将构建好的response返回
            ctx.writeAndFlush(response);
        }
    }
}
```

<br>