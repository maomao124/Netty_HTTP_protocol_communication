



## http协议

### 服务端

```java
package mao.t1;

import io.netty.bootstrap.ServerBootstrap;
import io.netty.buffer.ByteBuf;
import io.netty.channel.*;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.channel.socket.nio.NioSocketChannel;
import io.netty.handler.codec.http.*;
import io.netty.handler.logging.LogLevel;
import io.netty.handler.logging.LoggingHandler;
import io.netty.util.concurrent.Future;
import io.netty.util.concurrent.GenericFutureListener;
import lombok.extern.slf4j.Slf4j;

import java.nio.charset.StandardCharsets;
import java.util.Map;
import java.util.function.Consumer;

import static io.netty.handler.codec.http.HttpHeaderNames.CONTENT_LENGTH;

/**
 * Project name(项目名称)：Netty_HTTP协议通信
 * Package(包名): mao.t1
 * Class(类名): Server
 * Author(作者）: mao
 * Author QQ：1296193245
 * GitHub：https://github.com/maomao124/
 * Date(创建日期)： 2023/3/25
 * Time(创建时间)： 21:55
 * Version(版本): 1.0
 * Description(描述)： http协议
 */

@Slf4j
public class Server
{
    public static void main(String[] args)
    {
        NioEventLoopGroup boss = new NioEventLoopGroup(2);
        NioEventLoopGroup worker = new NioEventLoopGroup();
        ChannelFuture channelFuture = new ServerBootstrap()
                .group(boss, worker)
                .channel(NioServerSocketChannel.class)
                .childHandler(new ChannelInitializer<NioSocketChannel>()
                {
                    @Override
                    protected void initChannel(NioSocketChannel ch) throws Exception
                    {
                        ch.pipeline().addLast(new LoggingHandler(LogLevel.DEBUG))
                                .addLast(new HttpServerCodec())
                                .addLast(new SimpleChannelInboundHandler<HttpRequest>()
                                {
                                    /**
                                     * 处理请求行和请求头
                                     *
                                     * @param ctx         ctx
                                     * @param httpRequest http请求
                                     * @throws Exception 异常
                                     */
                                    @Override
                                    protected void channelRead0(ChannelHandlerContext ctx, HttpRequest httpRequest)
                                            throws Exception
                                    {
                                        log.debug("请求的uri：" + httpRequest.uri());
                                        //请求头
                                        HttpHeaders headers = httpRequest.headers();
                                        headers.forEach(new Consumer<Map.Entry<String, String>>()
                                        {
                                            @Override
                                            public void accept(Map.Entry<String, String> stringStringEntry)
                                            {
                                                String key = stringStringEntry.getKey();
                                                String value = stringStringEntry.getValue();
                                                log.debug(key + " ---> " + value);
                                            }
                                        });
                                        DefaultFullHttpResponse httpResponse = new
                                                DefaultFullHttpResponse(HttpVersion.HTTP_1_1,
                                                HttpResponseStatus.OK);
                                        String respMsg = "<h1>Hello, world!</h1>";
                                        byte[] bytes = respMsg.getBytes(StandardCharsets.UTF_8);
                                        httpResponse.headers().setInt(CONTENT_LENGTH, bytes.length);
                                        httpResponse.headers().set("content-type", "text/html;charset=utf-8");
                                        httpResponse.content().writeBytes(bytes);
                                        //写回响应
                                        ctx.writeAndFlush(httpResponse);
                                    }
                                })
                                .addLast(new SimpleChannelInboundHandler<HttpContent>()
                                {
                                    /**
                                     * 处理请求体
                                     *
                                     * @param ctx ctx
                                     * @param httpContent HttpContent
                                     * @throws Exception 异常
                                     */
                                    @Override
                                    protected void channelRead0(ChannelHandlerContext ctx, HttpContent httpContent)
                                            throws Exception
                                    {
                                        ByteBuf byteBuf = httpContent.content();
                                        String s = byteBuf.toString(StandardCharsets.UTF_8);
                                        log.debug("请求体：" + s);
                                    }
                                });
                    }
                }).bind(8080);
        Channel channel = channelFuture.channel();

        channelFuture.addListener(new GenericFutureListener<Future<? super Void>>()
        {
            @Override
            public void operationComplete(Future<? super Void> future) throws Exception
            {
                if (future.isSuccess())
                {
                    log.info("服务启动完成");
                }
                else
                {
                    log.warn("服务启动失败：" + future.cause().getMessage());
                }
            }
        });

        channel.closeFuture().addListener(new GenericFutureListener<Future<? super Void>>()
        {
            @Override
            public void operationComplete(Future<? super Void> future) throws Exception
            {
                log.info("关闭客户端");
                boss.shutdownGracefully();
                worker.shutdownGracefully();
            }
        });
    }
}

```





### 客户端

```java
package mao.t1;

import lombok.SneakyThrows;
import lombok.extern.slf4j.Slf4j;

import java.io.*;
import java.net.URL;
import java.net.URLConnection;
import java.nio.charset.StandardCharsets;
import java.util.Scanner;

/**
 * Project name(项目名称)：Netty_HTTP协议通信
 * Package(包名): mao.t1
 * Class(类名): Client
 * Author(作者）: mao
 * Author QQ：1296193245
 * GitHub：https://github.com/maomao124/
 * Date(创建日期)： 2023/3/25
 * Time(创建时间)： 21:55
 * Version(版本): 1.0
 * Description(描述)： 无
 */

@Slf4j
public class Client
{
    @SneakyThrows
    public static void main(String[] args)
    {
        Scanner input = new Scanner(System.in);
        while (true)
        {
            String requestBody = input.nextLine();
            if ("q".equals(requestBody))
            {
                return;
            }
            log.debug("请求体：" + requestBody);
            String resp = httpGet("http://localhost:8080/index.html", requestBody);
            log.info("响应内容：" + resp);
        }
    }

    /**
     * http get请求
     *
     * @param urlString   url字符串
     * @param requestBody 请求体
     * @return {@link String}
     */
    private static String httpGet(String urlString, String requestBody) throws IOException
    {
        BufferedReader bufferedReader = null;
        InputStreamReader inputStreamReader = null;
        InputStream inputStream = null;
        OutputStream outputStream = null;
        try
        {
            URL url = new URL(urlString);
            URLConnection urlConnection = url.openConnection();
            urlConnection.setDoOutput(true);
            //连接
            urlConnection.connect();
            outputStream = urlConnection.getOutputStream();
            //写请求体
            outputStream.write(requestBody.getBytes(StandardCharsets.UTF_8));
            inputStream = urlConnection.getInputStream();
            inputStreamReader = new InputStreamReader(inputStream, StandardCharsets.UTF_8);
            bufferedReader = new BufferedReader(inputStreamReader);
            StringBuilder stringBuilder = new StringBuilder();
            String resp;
            while ((resp = bufferedReader.readLine()) != null)
            {
                stringBuilder.append(resp);
            }
            return stringBuilder.toString();
        }
        finally
        {
            if (bufferedReader != null)
            {
                bufferedReader.close();
            }
            if (inputStreamReader != null)
            {
                inputStreamReader.close();
            }
            if (inputStream != null)
            {
                inputStream.close();
            }
            if (outputStream != null)
            {
                outputStream.close();
            }
        }
    }
}

```





### 运行结果

浏览器访问

```sh
2023-03-25  21:43:03.421  [nioEventLoopGroup-2-1] INFO  mao.t2.Server:  服务启动完成
2023-03-25  21:43:08.374  [nioEventLoopGroup-3-1] DEBUG io.netty.buffer.AbstractByteBuf:  -Dio.netty.buffer.checkAccessible: true
2023-03-25  21:43:08.374  [nioEventLoopGroup-3-1] DEBUG io.netty.buffer.AbstractByteBuf:  -Dio.netty.buffer.checkBounds: true
2023-03-25  21:43:08.375  [nioEventLoopGroup-3-1] DEBUG io.netty.util.ResourceLeakDetectorFactory:  Loaded default ResourceLeakDetector: io.netty.util.ResourceLeakDetector@6cf2b2c2
2023-03-25  21:43:08.380  [nioEventLoopGroup-3-1] DEBUG io.netty.handler.logging.LoggingHandler:  [id: 0x7d5b7c34, L:/[0:0:0:0:0:0:0:1]:8080 - R:/[0:0:0:0:0:0:0:1]:65438] REGISTERED
2023-03-25  21:43:08.380  [nioEventLoopGroup-3-2] DEBUG io.netty.handler.logging.LoggingHandler:  [id: 0x3598ea6e, L:/[0:0:0:0:0:0:0:1]:8080 - R:/[0:0:0:0:0:0:0:1]:65439] REGISTERED
2023-03-25  21:43:08.381  [nioEventLoopGroup-3-2] DEBUG io.netty.handler.logging.LoggingHandler:  [id: 0x3598ea6e, L:/[0:0:0:0:0:0:0:1]:8080 - R:/[0:0:0:0:0:0:0:1]:65439] ACTIVE
2023-03-25  21:43:08.381  [nioEventLoopGroup-3-1] DEBUG io.netty.handler.logging.LoggingHandler:  [id: 0x7d5b7c34, L:/[0:0:0:0:0:0:0:1]:8080 - R:/[0:0:0:0:0:0:0:1]:65438] ACTIVE
2023-03-25  21:43:08.382  [nioEventLoopGroup-3-2] DEBUG io.netty.util.Recycler:  -Dio.netty.recycler.maxCapacityPerThread: 4096
2023-03-25  21:43:08.382  [nioEventLoopGroup-3-2] DEBUG io.netty.util.Recycler:  -Dio.netty.recycler.maxSharedCapacityFactor: 2
2023-03-25  21:43:08.382  [nioEventLoopGroup-3-2] DEBUG io.netty.util.Recycler:  -Dio.netty.recycler.linkCapacity: 16
2023-03-25  21:43:08.382  [nioEventLoopGroup-3-2] DEBUG io.netty.util.Recycler:  -Dio.netty.recycler.ratio: 8
2023-03-25  21:43:08.388  [nioEventLoopGroup-3-2] DEBUG io.netty.handler.logging.LoggingHandler:  [id: 0x3598ea6e, L:/[0:0:0:0:0:0:0:1]:8080 - R:/[0:0:0:0:0:0:0:1]:65439] READ: 846B
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 47 45 54 20 2f 69 6e 64 65 78 2e 68 74 6d 6c 20 |GET /index.html |
|00000010| 48 54 54 50 2f 31 2e 31 0d 0a 48 6f 73 74 3a 20 |HTTP/1.1..Host: |
|00000020| 6c 6f 63 61 6c 68 6f 73 74 3a 38 30 38 30 0d 0a |localhost:8080..|
|00000030| 43 6f 6e 6e 65 63 74 69 6f 6e 3a 20 6b 65 65 70 |Connection: keep|
|00000040| 2d 61 6c 69 76 65 0d 0a 43 61 63 68 65 2d 43 6f |-alive..Cache-Co|
|00000050| 6e 74 72 6f 6c 3a 20 6d 61 78 2d 61 67 65 3d 30 |ntrol: max-age=0|
|00000060| 0d 0a 73 65 63 2d 63 68 2d 75 61 3a 20 22 4d 69 |..sec-ch-ua: "Mi|
|00000070| 63 72 6f 73 6f 66 74 20 45 64 67 65 22 3b 76 3d |crosoft Edge";v=|
|00000080| 22 31 31 31 22 2c 20 22 4e 6f 74 28 41 3a 42 72 |"111", "Not(A:Br|
|00000090| 61 6e 64 22 3b 76 3d 22 38 22 2c 20 22 43 68 72 |and";v="8", "Chr|
|000000a0| 6f 6d 69 75 6d 22 3b 76 3d 22 31 31 31 22 0d 0a |omium";v="111"..|
|000000b0| 73 65 63 2d 63 68 2d 75 61 2d 6d 6f 62 69 6c 65 |sec-ch-ua-mobile|
|000000c0| 3a 20 3f 30 0d 0a 73 65 63 2d 63 68 2d 75 61 2d |: ?0..sec-ch-ua-|
|000000d0| 70 6c 61 74 66 6f 72 6d 3a 20 22 57 69 6e 64 6f |platform: "Windo|
|000000e0| 77 73 22 0d 0a 55 70 67 72 61 64 65 2d 49 6e 73 |ws"..Upgrade-Ins|
|000000f0| 65 63 75 72 65 2d 52 65 71 75 65 73 74 73 3a 20 |ecure-Requests: |
|00000100| 31 0d 0a 55 73 65 72 2d 41 67 65 6e 74 3a 20 4d |1..User-Agent: M|
|00000110| 6f 7a 69 6c 6c 61 2f 35 2e 30 20 28 57 69 6e 64 |ozilla/5.0 (Wind|
|00000120| 6f 77 73 20 4e 54 20 31 30 2e 30 3b 20 57 69 6e |ows NT 10.0; Win|
|00000130| 36 34 3b 20 78 36 34 29 20 41 70 70 6c 65 57 65 |64; x64) AppleWe|
|00000140| 62 4b 69 74 2f 35 33 37 2e 33 36 20 28 4b 48 54 |bKit/537.36 (KHT|
|00000150| 4d 4c 2c 20 6c 69 6b 65 20 47 65 63 6b 6f 29 20 |ML, like Gecko) |
|00000160| 43 68 72 6f 6d 65 2f 31 31 31 2e 30 2e 30 2e 30 |Chrome/111.0.0.0|
|00000170| 20 53 61 66 61 72 69 2f 35 33 37 2e 33 36 20 45 | Safari/537.36 E|
|00000180| 64 67 2f 31 31 31 2e 30 2e 31 36 36 31 2e 35 31 |dg/111.0.1661.51|
|00000190| 0d 0a 41 63 63 65 70 74 3a 20 74 65 78 74 2f 68 |..Accept: text/h|
|000001a0| 74 6d 6c 2c 61 70 70 6c 69 63 61 74 69 6f 6e 2f |tml,application/|
|000001b0| 78 68 74 6d 6c 2b 78 6d 6c 2c 61 70 70 6c 69 63 |xhtml+xml,applic|
|000001c0| 61 74 69 6f 6e 2f 78 6d 6c 3b 71 3d 30 2e 39 2c |ation/xml;q=0.9,|
|000001d0| 69 6d 61 67 65 2f 77 65 62 70 2c 69 6d 61 67 65 |image/webp,image|
|000001e0| 2f 61 70 6e 67 2c 2a 2f 2a 3b 71 3d 30 2e 38 2c |/apng,*/*;q=0.8,|
|000001f0| 61 70 70 6c 69 63 61 74 69 6f 6e 2f 73 69 67 6e |application/sign|
|00000200| 65 64 2d 65 78 63 68 61 6e 67 65 3b 76 3d 62 33 |ed-exchange;v=b3|
|00000210| 3b 71 3d 30 2e 37 0d 0a 53 65 63 2d 46 65 74 63 |;q=0.7..Sec-Fetc|
|00000220| 68 2d 53 69 74 65 3a 20 6e 6f 6e 65 0d 0a 53 65 |h-Site: none..Se|
|00000230| 63 2d 46 65 74 63 68 2d 4d 6f 64 65 3a 20 6e 61 |c-Fetch-Mode: na|
|00000240| 76 69 67 61 74 65 0d 0a 53 65 63 2d 46 65 74 63 |vigate..Sec-Fetc|
|00000250| 68 2d 55 73 65 72 3a 20 3f 31 0d 0a 53 65 63 2d |h-User: ?1..Sec-|
|00000260| 46 65 74 63 68 2d 44 65 73 74 3a 20 64 6f 63 75 |Fetch-Dest: docu|
|00000270| 6d 65 6e 74 0d 0a 41 63 63 65 70 74 2d 45 6e 63 |ment..Accept-Enc|
|00000280| 6f 64 69 6e 67 3a 20 67 7a 69 70 2c 20 64 65 66 |oding: gzip, def|
|00000290| 6c 61 74 65 2c 20 62 72 0d 0a 41 63 63 65 70 74 |late, br..Accept|
|000002a0| 2d 4c 61 6e 67 75 61 67 65 3a 20 7a 68 2d 43 4e |-Language: zh-CN|
|000002b0| 2c 7a 68 3b 71 3d 30 2e 39 2c 65 6e 3b 71 3d 30 |,zh;q=0.9,en;q=0|
|000002c0| 2e 38 2c 65 6e 2d 47 42 3b 71 3d 30 2e 37 2c 65 |.8,en-GB;q=0.7,e|
|000002d0| 6e 2d 55 53 3b 71 3d 30 2e 36 0d 0a 43 6f 6f 6b |n-US;q=0.6..Cook|
|000002e0| 69 65 3a 20 49 64 65 61 2d 32 33 34 37 65 36 38 |ie: Idea-2347e68|
|000002f0| 33 3d 65 37 63 31 39 34 64 36 2d 63 65 66 31 2d |3=e7c194d6-cef1-|
|00000300| 34 61 30 36 2d 38 32 31 35 2d 30 33 39 36 34 34 |4a06-8215-039644|
|00000310| 65 34 39 66 30 31 3b 20 48 6d 5f 6c 76 74 5f 38 |e49f01; Hm_lvt_8|
|00000320| 61 63 65 66 36 36 39 65 61 36 36 66 34 37 39 38 |acef669ea66f4798|
|00000330| 35 34 65 63 64 33 32 38 64 31 66 33 34 38 66 3d |54ecd328d1f348f=|
|00000340| 31 36 37 39 35 35 35 34 34 36 0d 0a 0d 0a       |1679555446....  |
+--------+-------------------------------------------------+----------------+
2023-03-25  21:43:08.395  [nioEventLoopGroup-3-2] DEBUG mao.t2.Server:  请求的uri：/index.html
2023-03-25  21:43:08.400  [nioEventLoopGroup-3-2] DEBUG mao.t2.Server:  Host ---> localhost:8080
2023-03-25  21:43:08.400  [nioEventLoopGroup-3-2] DEBUG mao.t2.Server:  Connection ---> keep-alive
2023-03-25  21:43:08.400  [nioEventLoopGroup-3-2] DEBUG mao.t2.Server:  Cache-Control ---> max-age=0
2023-03-25  21:43:08.400  [nioEventLoopGroup-3-2] DEBUG mao.t2.Server:  sec-ch-ua ---> "Microsoft Edge";v="111", "Not(A:Brand";v="8", "Chromium";v="111"
2023-03-25  21:43:08.400  [nioEventLoopGroup-3-2] DEBUG mao.t2.Server:  sec-ch-ua-mobile ---> ?0
2023-03-25  21:43:08.400  [nioEventLoopGroup-3-2] DEBUG mao.t2.Server:  sec-ch-ua-platform ---> "Windows"
2023-03-25  21:43:08.400  [nioEventLoopGroup-3-2] DEBUG mao.t2.Server:  Upgrade-Insecure-Requests ---> 1
2023-03-25  21:43:08.400  [nioEventLoopGroup-3-2] DEBUG mao.t2.Server:  User-Agent ---> Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/111.0.0.0 Safari/537.36 Edg/111.0.1661.51
2023-03-25  21:43:08.400  [nioEventLoopGroup-3-2] DEBUG mao.t2.Server:  Accept ---> text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
2023-03-25  21:43:08.400  [nioEventLoopGroup-3-2] DEBUG mao.t2.Server:  Sec-Fetch-Site ---> none
2023-03-25  21:43:08.401  [nioEventLoopGroup-3-2] DEBUG mao.t2.Server:  Sec-Fetch-Mode ---> navigate
2023-03-25  21:43:08.401  [nioEventLoopGroup-3-2] DEBUG mao.t2.Server:  Sec-Fetch-User ---> ?1
2023-03-25  21:43:08.401  [nioEventLoopGroup-3-2] DEBUG mao.t2.Server:  Sec-Fetch-Dest ---> document
2023-03-25  21:43:08.401  [nioEventLoopGroup-3-2] DEBUG mao.t2.Server:  Accept-Encoding ---> gzip, deflate, br
2023-03-25  21:43:08.401  [nioEventLoopGroup-3-2] DEBUG mao.t2.Server:  Accept-Language ---> zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6
2023-03-25  21:43:08.401  [nioEventLoopGroup-3-2] DEBUG mao.t2.Server:  Cookie ---> Idea-2347e683=e7c194d6-cef1-4a06-8215-039644e49f01; Hm_lvt_8acef669ea66f479854ecd328d1f348f=1679555446
2023-03-25  21:43:08.402  [nioEventLoopGroup-3-2] DEBUG io.netty.handler.logging.LoggingHandler:  [id: 0x3598ea6e, L:/[0:0:0:0:0:0:0:1]:8080 - R:/[0:0:0:0:0:0:0:1]:65439] WRITE: 100B
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 48 54 54 50 2f 31 2e 31 20 32 30 30 20 4f 4b 0d |HTTP/1.1 200 OK.|
|00000010| 0a 63 6f 6e 74 65 6e 74 2d 6c 65 6e 67 74 68 3a |.content-length:|
|00000020| 20 32 32 0d 0a 63 6f 6e 74 65 6e 74 2d 74 79 70 | 22..content-typ|
|00000030| 65 3a 20 74 65 78 74 2f 68 74 6d 6c 3b 63 68 61 |e: text/html;cha|
|00000040| 72 73 65 74 3d 75 74 66 2d 38 0d 0a 0d 0a 3c 68 |rset=utf-8....<h|
|00000050| 31 3e 48 65 6c 6c 6f 2c 20 77 6f 72 6c 64 21 3c |1>Hello, world!<|
|00000060| 2f 68 31 3e                                     |/h1>            |
+--------+-------------------------------------------------+----------------+
2023-03-25  21:43:08.403  [nioEventLoopGroup-3-2] DEBUG io.netty.handler.logging.LoggingHandler:  [id: 0x3598ea6e, L:/[0:0:0:0:0:0:0:1]:8080 - R:/[0:0:0:0:0:0:0:1]:65439] FLUSH
2023-03-25  21:43:08.403  [nioEventLoopGroup-3-2] DEBUG mao.t2.Server:  请求体：
2023-03-25  21:43:08.403  [nioEventLoopGroup-3-2] DEBUG io.netty.handler.logging.LoggingHandler:  [id: 0x3598ea6e, L:/[0:0:0:0:0:0:0:1]:8080 - R:/[0:0:0:0:0:0:0:1]:65439] READ COMPLETE
2023-03-25  21:43:08.433  [nioEventLoopGroup-3-2] DEBUG io.netty.handler.logging.LoggingHandler:  [id: 0x3598ea6e, L:/[0:0:0:0:0:0:0:1]:8080 - R:/[0:0:0:0:0:0:0:1]:65439] READ: 746B
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 47 45 54 20 2f 66 61 76 69 63 6f 6e 2e 69 63 6f |GET /favicon.ico|
|00000010| 20 48 54 54 50 2f 31 2e 31 0d 0a 48 6f 73 74 3a | HTTP/1.1..Host:|
|00000020| 20 6c 6f 63 61 6c 68 6f 73 74 3a 38 30 38 30 0d | localhost:8080.|
|00000030| 0a 43 6f 6e 6e 65 63 74 69 6f 6e 3a 20 6b 65 65 |.Connection: kee|
|00000040| 70 2d 61 6c 69 76 65 0d 0a 73 65 63 2d 63 68 2d |p-alive..sec-ch-|
|00000050| 75 61 3a 20 22 4d 69 63 72 6f 73 6f 66 74 20 45 |ua: "Microsoft E|
|00000060| 64 67 65 22 3b 76 3d 22 31 31 31 22 2c 20 22 4e |dge";v="111", "N|
|00000070| 6f 74 28 41 3a 42 72 61 6e 64 22 3b 76 3d 22 38 |ot(A:Brand";v="8|
|00000080| 22 2c 20 22 43 68 72 6f 6d 69 75 6d 22 3b 76 3d |", "Chromium";v=|
|00000090| 22 31 31 31 22 0d 0a 73 65 63 2d 63 68 2d 75 61 |"111"..sec-ch-ua|
|000000a0| 2d 6d 6f 62 69 6c 65 3a 20 3f 30 0d 0a 55 73 65 |-mobile: ?0..Use|
|000000b0| 72 2d 41 67 65 6e 74 3a 20 4d 6f 7a 69 6c 6c 61 |r-Agent: Mozilla|
|000000c0| 2f 35 2e 30 20 28 57 69 6e 64 6f 77 73 20 4e 54 |/5.0 (Windows NT|
|000000d0| 20 31 30 2e 30 3b 20 57 69 6e 36 34 3b 20 78 36 | 10.0; Win64; x6|
|000000e0| 34 29 20 41 70 70 6c 65 57 65 62 4b 69 74 2f 35 |4) AppleWebKit/5|
|000000f0| 33 37 2e 33 36 20 28 4b 48 54 4d 4c 2c 20 6c 69 |37.36 (KHTML, li|
|00000100| 6b 65 20 47 65 63 6b 6f 29 20 43 68 72 6f 6d 65 |ke Gecko) Chrome|
|00000110| 2f 31 31 31 2e 30 2e 30 2e 30 20 53 61 66 61 72 |/111.0.0.0 Safar|
|00000120| 69 2f 35 33 37 2e 33 36 20 45 64 67 2f 31 31 31 |i/537.36 Edg/111|
|00000130| 2e 30 2e 31 36 36 31 2e 35 31 0d 0a 73 65 63 2d |.0.1661.51..sec-|
|00000140| 63 68 2d 75 61 2d 70 6c 61 74 66 6f 72 6d 3a 20 |ch-ua-platform: |
|00000150| 22 57 69 6e 64 6f 77 73 22 0d 0a 41 63 63 65 70 |"Windows"..Accep|
|00000160| 74 3a 20 69 6d 61 67 65 2f 77 65 62 70 2c 69 6d |t: image/webp,im|
|00000170| 61 67 65 2f 61 70 6e 67 2c 69 6d 61 67 65 2f 73 |age/apng,image/s|
|00000180| 76 67 2b 78 6d 6c 2c 69 6d 61 67 65 2f 2a 2c 2a |vg+xml,image/*,*|
|00000190| 2f 2a 3b 71 3d 30 2e 38 0d 0a 53 65 63 2d 46 65 |/*;q=0.8..Sec-Fe|
|000001a0| 74 63 68 2d 53 69 74 65 3a 20 73 61 6d 65 2d 6f |tch-Site: same-o|
|000001b0| 72 69 67 69 6e 0d 0a 53 65 63 2d 46 65 74 63 68 |rigin..Sec-Fetch|
|000001c0| 2d 4d 6f 64 65 3a 20 6e 6f 2d 63 6f 72 73 0d 0a |-Mode: no-cors..|
|000001d0| 53 65 63 2d 46 65 74 63 68 2d 44 65 73 74 3a 20 |Sec-Fetch-Dest: |
|000001e0| 69 6d 61 67 65 0d 0a 52 65 66 65 72 65 72 3a 20 |image..Referer: |
|000001f0| 68 74 74 70 3a 2f 2f 6c 6f 63 61 6c 68 6f 73 74 |http://localhost|
|00000200| 3a 38 30 38 30 2f 69 6e 64 65 78 2e 68 74 6d 6c |:8080/index.html|
|00000210| 0d 0a 41 63 63 65 70 74 2d 45 6e 63 6f 64 69 6e |..Accept-Encodin|
|00000220| 67 3a 20 67 7a 69 70 2c 20 64 65 66 6c 61 74 65 |g: gzip, deflate|
|00000230| 2c 20 62 72 0d 0a 41 63 63 65 70 74 2d 4c 61 6e |, br..Accept-Lan|
|00000240| 67 75 61 67 65 3a 20 7a 68 2d 43 4e 2c 7a 68 3b |guage: zh-CN,zh;|
|00000250| 71 3d 30 2e 39 2c 65 6e 3b 71 3d 30 2e 38 2c 65 |q=0.9,en;q=0.8,e|
|00000260| 6e 2d 47 42 3b 71 3d 30 2e 37 2c 65 6e 2d 55 53 |n-GB;q=0.7,en-US|
|00000270| 3b 71 3d 30 2e 36 0d 0a 43 6f 6f 6b 69 65 3a 20 |;q=0.6..Cookie: |
|00000280| 49 64 65 61 2d 32 33 34 37 65 36 38 33 3d 65 37 |Idea-2347e683=e7|
|00000290| 63 31 39 34 64 36 2d 63 65 66 31 2d 34 61 30 36 |c194d6-cef1-4a06|
|000002a0| 2d 38 32 31 35 2d 30 33 39 36 34 34 65 34 39 66 |-8215-039644e49f|
|000002b0| 30 31 3b 20 48 6d 5f 6c 76 74 5f 38 61 63 65 66 |01; Hm_lvt_8acef|
|000002c0| 36 36 39 65 61 36 36 66 34 37 39 38 35 34 65 63 |669ea66f479854ec|
|000002d0| 64 33 32 38 64 31 66 33 34 38 66 3d 31 36 37 39 |d328d1f348f=1679|
|000002e0| 35 35 35 34 34 36 0d 0a 0d 0a                   |555446....      |
+--------+-------------------------------------------------+----------------+
2023-03-25  21:43:08.433  [nioEventLoopGroup-3-2] DEBUG mao.t2.Server:  请求的uri：/favicon.ico
2023-03-25  21:43:08.433  [nioEventLoopGroup-3-2] DEBUG mao.t2.Server:  Host ---> localhost:8080
2023-03-25  21:43:08.433  [nioEventLoopGroup-3-2] DEBUG mao.t2.Server:  Connection ---> keep-alive
2023-03-25  21:43:08.433  [nioEventLoopGroup-3-2] DEBUG mao.t2.Server:  sec-ch-ua ---> "Microsoft Edge";v="111", "Not(A:Brand";v="8", "Chromium";v="111"
2023-03-25  21:43:08.433  [nioEventLoopGroup-3-2] DEBUG mao.t2.Server:  sec-ch-ua-mobile ---> ?0
2023-03-25  21:43:08.433  [nioEventLoopGroup-3-2] DEBUG mao.t2.Server:  User-Agent ---> Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/111.0.0.0 Safari/537.36 Edg/111.0.1661.51
2023-03-25  21:43:08.433  [nioEventLoopGroup-3-2] DEBUG mao.t2.Server:  sec-ch-ua-platform ---> "Windows"
2023-03-25  21:43:08.433  [nioEventLoopGroup-3-2] DEBUG mao.t2.Server:  Accept ---> image/webp,image/apng,image/svg+xml,image/*,*/*;q=0.8
2023-03-25  21:43:08.433  [nioEventLoopGroup-3-2] DEBUG mao.t2.Server:  Sec-Fetch-Site ---> same-origin
2023-03-25  21:43:08.433  [nioEventLoopGroup-3-2] DEBUG mao.t2.Server:  Sec-Fetch-Mode ---> no-cors
2023-03-25  21:43:08.433  [nioEventLoopGroup-3-2] DEBUG mao.t2.Server:  Sec-Fetch-Dest ---> image
2023-03-25  21:43:08.433  [nioEventLoopGroup-3-2] DEBUG mao.t2.Server:  Referer ---> http://localhost:8080/index.html
2023-03-25  21:43:08.434  [nioEventLoopGroup-3-2] DEBUG mao.t2.Server:  Accept-Encoding ---> gzip, deflate, br
2023-03-25  21:43:08.434  [nioEventLoopGroup-3-2] DEBUG mao.t2.Server:  Accept-Language ---> zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6
2023-03-25  21:43:08.434  [nioEventLoopGroup-3-2] DEBUG mao.t2.Server:  Cookie ---> Idea-2347e683=e7c194d6-cef1-4a06-8215-039644e49f01; Hm_lvt_8acef669ea66f479854ecd328d1f348f=1679555446
2023-03-25  21:43:08.434  [nioEventLoopGroup-3-2] DEBUG io.netty.handler.logging.LoggingHandler:  [id: 0x3598ea6e, L:/[0:0:0:0:0:0:0:1]:8080 - R:/[0:0:0:0:0:0:0:1]:65439] WRITE: 100B
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 48 54 54 50 2f 31 2e 31 20 32 30 30 20 4f 4b 0d |HTTP/1.1 200 OK.|
|00000010| 0a 63 6f 6e 74 65 6e 74 2d 6c 65 6e 67 74 68 3a |.content-length:|
|00000020| 20 32 32 0d 0a 63 6f 6e 74 65 6e 74 2d 74 79 70 | 22..content-typ|
|00000030| 65 3a 20 74 65 78 74 2f 68 74 6d 6c 3b 63 68 61 |e: text/html;cha|
|00000040| 72 73 65 74 3d 75 74 66 2d 38 0d 0a 0d 0a 3c 68 |rset=utf-8....<h|
|00000050| 31 3e 48 65 6c 6c 6f 2c 20 77 6f 72 6c 64 21 3c |1>Hello, world!<|
|00000060| 2f 68 31 3e                                     |/h1>            |
+--------+-------------------------------------------------+----------------+
2023-03-25  21:43:08.434  [nioEventLoopGroup-3-2] DEBUG io.netty.handler.logging.LoggingHandler:  [id: 0x3598ea6e, L:/[0:0:0:0:0:0:0:1]:8080 - R:/[0:0:0:0:0:0:0:1]:65439] FLUSH
2023-03-25  21:43:08.434  [nioEventLoopGroup-3-2] DEBUG mao.t2.Server:  请求体：
2023-03-25  21:43:08.434  [nioEventLoopGroup-3-2] DEBUG io.netty.handler.logging.LoggingHandler:  [id: 0x3598ea6e, L:/[0:0:0:0:0:0:0:1]:8080 - R:/[0:0:0:0:0:0:0:1]:65439] READ COMPLETE
```



客户端访问

```sh
hello
2023-03-25  21:44:17.314  [main] DEBUG mao.t2.Client:  请求体：hello
2023-03-25  21:44:17.324  [main] INFO  mao.t2.Client:  响应内容：<h1>Hello, world!</h1>
你好
2023-03-25  21:44:27.771  [main] DEBUG mao.t2.Client:  请求体：你好
2023-03-25  21:44:27.777  [main] INFO  mao.t2.Client:  响应内容：<h1>Hello, world!</h1>
654
2023-03-25  21:44:32.211  [main] DEBUG mao.t2.Client:  请求体：654
2023-03-25  21:44:32.213  [main] INFO  mao.t2.Client:  响应内容：<h1>Hello, world!</h1>
```

```sh
2023-03-25  21:44:17.319  [nioEventLoopGroup-3-3] DEBUG io.netty.handler.logging.LoggingHandler:  [id: 0x9b850779, L:/127.0.0.1:8080 - R:/127.0.0.1:65455] REGISTERED
2023-03-25  21:44:17.319  [nioEventLoopGroup-3-3] DEBUG io.netty.handler.logging.LoggingHandler:  [id: 0x9b850779, L:/127.0.0.1:8080 - R:/127.0.0.1:65455] ACTIVE
2023-03-25  21:44:17.322  [nioEventLoopGroup-3-3] DEBUG io.netty.handler.logging.LoggingHandler:  [id: 0x9b850779, L:/127.0.0.1:8080 - R:/127.0.0.1:65455] READ: 235B
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 50 4f 53 54 20 2f 69 6e 64 65 78 2e 68 74 6d 6c |POST /index.html|
|00000010| 20 48 54 54 50 2f 31 2e 31 0d 0a 55 73 65 72 2d | HTTP/1.1..User-|
|00000020| 41 67 65 6e 74 3a 20 4a 61 76 61 2f 31 36 2e 30 |Agent: Java/16.0|
|00000030| 2e 32 0d 0a 48 6f 73 74 3a 20 6c 6f 63 61 6c 68 |.2..Host: localh|
|00000040| 6f 73 74 3a 38 30 38 30 0d 0a 41 63 63 65 70 74 |ost:8080..Accept|
|00000050| 3a 20 74 65 78 74 2f 68 74 6d 6c 2c 20 69 6d 61 |: text/html, ima|
|00000060| 67 65 2f 67 69 66 2c 20 69 6d 61 67 65 2f 6a 70 |ge/gif, image/jp|
|00000070| 65 67 2c 20 2a 3b 20 71 3d 2e 32 2c 20 2a 2f 2a |eg, *; q=.2, */*|
|00000080| 3b 20 71 3d 2e 32 0d 0a 43 6f 6e 6e 65 63 74 69 |; q=.2..Connecti|
|00000090| 6f 6e 3a 20 6b 65 65 70 2d 61 6c 69 76 65 0d 0a |on: keep-alive..|
|000000a0| 43 6f 6e 74 65 6e 74 2d 74 79 70 65 3a 20 61 70 |Content-type: ap|
|000000b0| 70 6c 69 63 61 74 69 6f 6e 2f 78 2d 77 77 77 2d |plication/x-www-|
|000000c0| 66 6f 72 6d 2d 75 72 6c 65 6e 63 6f 64 65 64 0d |form-urlencoded.|
|000000d0| 0a 43 6f 6e 74 65 6e 74 2d 4c 65 6e 67 74 68 3a |.Content-Length:|
|000000e0| 20 35 0d 0a 0d 0a 68 65 6c 6c 6f                | 5....hello     |
+--------+-------------------------------------------------+----------------+
2023-03-25  21:44:17.323  [nioEventLoopGroup-3-3] DEBUG mao.t2.Server:  请求的uri：/index.html
2023-03-25  21:44:17.323  [nioEventLoopGroup-3-3] DEBUG mao.t2.Server:  User-Agent ---> Java/16.0.2
2023-03-25  21:44:17.323  [nioEventLoopGroup-3-3] DEBUG mao.t2.Server:  Host ---> localhost:8080
2023-03-25  21:44:17.323  [nioEventLoopGroup-3-3] DEBUG mao.t2.Server:  Accept ---> text/html, image/gif, image/jpeg, *; q=.2, */*; q=.2
2023-03-25  21:44:17.323  [nioEventLoopGroup-3-3] DEBUG mao.t2.Server:  Connection ---> keep-alive
2023-03-25  21:44:17.323  [nioEventLoopGroup-3-3] DEBUG mao.t2.Server:  Content-type ---> application/x-www-form-urlencoded
2023-03-25  21:44:17.323  [nioEventLoopGroup-3-3] DEBUG mao.t2.Server:  Content-Length ---> 5
2023-03-25  21:44:17.323  [nioEventLoopGroup-3-3] DEBUG io.netty.handler.logging.LoggingHandler:  [id: 0x9b850779, L:/127.0.0.1:8080 - R:/127.0.0.1:65455] WRITE: 100B
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 48 54 54 50 2f 31 2e 31 20 32 30 30 20 4f 4b 0d |HTTP/1.1 200 OK.|
|00000010| 0a 63 6f 6e 74 65 6e 74 2d 6c 65 6e 67 74 68 3a |.content-length:|
|00000020| 20 32 32 0d 0a 63 6f 6e 74 65 6e 74 2d 74 79 70 | 22..content-typ|
|00000030| 65 3a 20 74 65 78 74 2f 68 74 6d 6c 3b 63 68 61 |e: text/html;cha|
|00000040| 72 73 65 74 3d 75 74 66 2d 38 0d 0a 0d 0a 3c 68 |rset=utf-8....<h|
|00000050| 31 3e 48 65 6c 6c 6f 2c 20 77 6f 72 6c 64 21 3c |1>Hello, world!<|
|00000060| 2f 68 31 3e                                     |/h1>            |
+--------+-------------------------------------------------+----------------+
2023-03-25  21:44:17.323  [nioEventLoopGroup-3-3] DEBUG io.netty.handler.logging.LoggingHandler:  [id: 0x9b850779, L:/127.0.0.1:8080 - R:/127.0.0.1:65455] FLUSH
2023-03-25  21:44:17.324  [nioEventLoopGroup-3-3] DEBUG mao.t2.Server:  请求体：hello
2023-03-25  21:44:17.325  [nioEventLoopGroup-3-3] DEBUG io.netty.handler.logging.LoggingHandler:  [id: 0x9b850779, L:/127.0.0.1:8080 - R:/127.0.0.1:65455] READ COMPLETE
2023-03-25  21:44:22.337  [nioEventLoopGroup-3-3] DEBUG io.netty.handler.logging.LoggingHandler:  [id: 0x9b850779, L:/127.0.0.1:8080 - R:/127.0.0.1:65455] READ COMPLETE
2023-03-25  21:44:22.337  [nioEventLoopGroup-3-3] DEBUG io.netty.handler.logging.LoggingHandler:  [id: 0x9b850779, L:/127.0.0.1:8080 ! R:/127.0.0.1:65455] INACTIVE
2023-03-25  21:44:22.337  [nioEventLoopGroup-3-3] DEBUG io.netty.handler.logging.LoggingHandler:  [id: 0x9b850779, L:/127.0.0.1:8080 ! R:/127.0.0.1:65455] UNREGISTERED
2023-03-25  21:44:27.773  [nioEventLoopGroup-3-4] DEBUG io.netty.handler.logging.LoggingHandler:  [id: 0x68cd2b82, L:/127.0.0.1:8080 - R:/127.0.0.1:65456] REGISTERED
2023-03-25  21:44:27.773  [nioEventLoopGroup-3-4] DEBUG io.netty.handler.logging.LoggingHandler:  [id: 0x68cd2b82, L:/127.0.0.1:8080 - R:/127.0.0.1:65456] ACTIVE
2023-03-25  21:44:27.776  [nioEventLoopGroup-3-4] DEBUG io.netty.handler.logging.LoggingHandler:  [id: 0x68cd2b82, L:/127.0.0.1:8080 - R:/127.0.0.1:65456] READ: 236B
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 50 4f 53 54 20 2f 69 6e 64 65 78 2e 68 74 6d 6c |POST /index.html|
|00000010| 20 48 54 54 50 2f 31 2e 31 0d 0a 55 73 65 72 2d | HTTP/1.1..User-|
|00000020| 41 67 65 6e 74 3a 20 4a 61 76 61 2f 31 36 2e 30 |Agent: Java/16.0|
|00000030| 2e 32 0d 0a 48 6f 73 74 3a 20 6c 6f 63 61 6c 68 |.2..Host: localh|
|00000040| 6f 73 74 3a 38 30 38 30 0d 0a 41 63 63 65 70 74 |ost:8080..Accept|
|00000050| 3a 20 74 65 78 74 2f 68 74 6d 6c 2c 20 69 6d 61 |: text/html, ima|
|00000060| 67 65 2f 67 69 66 2c 20 69 6d 61 67 65 2f 6a 70 |ge/gif, image/jp|
|00000070| 65 67 2c 20 2a 3b 20 71 3d 2e 32 2c 20 2a 2f 2a |eg, *; q=.2, */*|
|00000080| 3b 20 71 3d 2e 32 0d 0a 43 6f 6e 6e 65 63 74 69 |; q=.2..Connecti|
|00000090| 6f 6e 3a 20 6b 65 65 70 2d 61 6c 69 76 65 0d 0a |on: keep-alive..|
|000000a0| 43 6f 6e 74 65 6e 74 2d 74 79 70 65 3a 20 61 70 |Content-type: ap|
|000000b0| 70 6c 69 63 61 74 69 6f 6e 2f 78 2d 77 77 77 2d |plication/x-www-|
|000000c0| 66 6f 72 6d 2d 75 72 6c 65 6e 63 6f 64 65 64 0d |form-urlencoded.|
|000000d0| 0a 43 6f 6e 74 65 6e 74 2d 4c 65 6e 67 74 68 3a |.Content-Length:|
|000000e0| 20 36 0d 0a 0d 0a e4 bd a0 e5 a5 bd             | 6..........    |
+--------+-------------------------------------------------+----------------+
2023-03-25  21:44:27.776  [nioEventLoopGroup-3-4] DEBUG mao.t2.Server:  请求的uri：/index.html
2023-03-25  21:44:27.776  [nioEventLoopGroup-3-4] DEBUG mao.t2.Server:  User-Agent ---> Java/16.0.2
2023-03-25  21:44:27.776  [nioEventLoopGroup-3-4] DEBUG mao.t2.Server:  Host ---> localhost:8080
2023-03-25  21:44:27.777  [nioEventLoopGroup-3-4] DEBUG mao.t2.Server:  Accept ---> text/html, image/gif, image/jpeg, *; q=.2, */*; q=.2
2023-03-25  21:44:27.777  [nioEventLoopGroup-3-4] DEBUG mao.t2.Server:  Connection ---> keep-alive
2023-03-25  21:44:27.777  [nioEventLoopGroup-3-4] DEBUG mao.t2.Server:  Content-type ---> application/x-www-form-urlencoded
2023-03-25  21:44:27.777  [nioEventLoopGroup-3-4] DEBUG mao.t2.Server:  Content-Length ---> 6
2023-03-25  21:44:27.777  [nioEventLoopGroup-3-4] DEBUG io.netty.handler.logging.LoggingHandler:  [id: 0x68cd2b82, L:/127.0.0.1:8080 - R:/127.0.0.1:65456] WRITE: 100B
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 48 54 54 50 2f 31 2e 31 20 32 30 30 20 4f 4b 0d |HTTP/1.1 200 OK.|
|00000010| 0a 63 6f 6e 74 65 6e 74 2d 6c 65 6e 67 74 68 3a |.content-length:|
|00000020| 20 32 32 0d 0a 63 6f 6e 74 65 6e 74 2d 74 79 70 | 22..content-typ|
|00000030| 65 3a 20 74 65 78 74 2f 68 74 6d 6c 3b 63 68 61 |e: text/html;cha|
|00000040| 72 73 65 74 3d 75 74 66 2d 38 0d 0a 0d 0a 3c 68 |rset=utf-8....<h|
|00000050| 31 3e 48 65 6c 6c 6f 2c 20 77 6f 72 6c 64 21 3c |1>Hello, world!<|
|00000060| 2f 68 31 3e                                     |/h1>            |
+--------+-------------------------------------------------+----------------+
2023-03-25  21:44:27.777  [nioEventLoopGroup-3-4] DEBUG io.netty.handler.logging.LoggingHandler:  [id: 0x68cd2b82, L:/127.0.0.1:8080 - R:/127.0.0.1:65456] FLUSH
2023-03-25  21:44:27.778  [nioEventLoopGroup-3-4] DEBUG mao.t2.Server:  请求体：你好
2023-03-25  21:44:27.778  [nioEventLoopGroup-3-4] DEBUG io.netty.handler.logging.LoggingHandler:  [id: 0x68cd2b82, L:/127.0.0.1:8080 - R:/127.0.0.1:65456] READ COMPLETE
2023-03-25  21:44:32.212  [nioEventLoopGroup-3-4] DEBUG io.netty.handler.logging.LoggingHandler:  [id: 0x68cd2b82, L:/127.0.0.1:8080 - R:/127.0.0.1:65456] READ: 233B
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 50 4f 53 54 20 2f 69 6e 64 65 78 2e 68 74 6d 6c |POST /index.html|
|00000010| 20 48 54 54 50 2f 31 2e 31 0d 0a 55 73 65 72 2d | HTTP/1.1..User-|
|00000020| 41 67 65 6e 74 3a 20 4a 61 76 61 2f 31 36 2e 30 |Agent: Java/16.0|
|00000030| 2e 32 0d 0a 48 6f 73 74 3a 20 6c 6f 63 61 6c 68 |.2..Host: localh|
|00000040| 6f 73 74 3a 38 30 38 30 0d 0a 41 63 63 65 70 74 |ost:8080..Accept|
|00000050| 3a 20 74 65 78 74 2f 68 74 6d 6c 2c 20 69 6d 61 |: text/html, ima|
|00000060| 67 65 2f 67 69 66 2c 20 69 6d 61 67 65 2f 6a 70 |ge/gif, image/jp|
|00000070| 65 67 2c 20 2a 3b 20 71 3d 2e 32 2c 20 2a 2f 2a |eg, *; q=.2, */*|
|00000080| 3b 20 71 3d 2e 32 0d 0a 43 6f 6e 6e 65 63 74 69 |; q=.2..Connecti|
|00000090| 6f 6e 3a 20 6b 65 65 70 2d 61 6c 69 76 65 0d 0a |on: keep-alive..|
|000000a0| 43 6f 6e 74 65 6e 74 2d 74 79 70 65 3a 20 61 70 |Content-type: ap|
|000000b0| 70 6c 69 63 61 74 69 6f 6e 2f 78 2d 77 77 77 2d |plication/x-www-|
|000000c0| 66 6f 72 6d 2d 75 72 6c 65 6e 63 6f 64 65 64 0d |form-urlencoded.|
|000000d0| 0a 43 6f 6e 74 65 6e 74 2d 4c 65 6e 67 74 68 3a |.Content-Length:|
|000000e0| 20 33 0d 0a 0d 0a 36 35 34                      | 3....654       |
+--------+-------------------------------------------------+----------------+
2023-03-25  21:44:32.212  [nioEventLoopGroup-3-4] DEBUG mao.t2.Server:  请求的uri：/index.html
2023-03-25  21:44:32.212  [nioEventLoopGroup-3-4] DEBUG mao.t2.Server:  User-Agent ---> Java/16.0.2
2023-03-25  21:44:32.212  [nioEventLoopGroup-3-4] DEBUG mao.t2.Server:  Host ---> localhost:8080
2023-03-25  21:44:32.212  [nioEventLoopGroup-3-4] DEBUG mao.t2.Server:  Accept ---> text/html, image/gif, image/jpeg, *; q=.2, */*; q=.2
2023-03-25  21:44:32.212  [nioEventLoopGroup-3-4] DEBUG mao.t2.Server:  Connection ---> keep-alive
2023-03-25  21:44:32.212  [nioEventLoopGroup-3-4] DEBUG mao.t2.Server:  Content-type ---> application/x-www-form-urlencoded
2023-03-25  21:44:32.212  [nioEventLoopGroup-3-4] DEBUG mao.t2.Server:  Content-Length ---> 3
2023-03-25  21:44:32.212  [nioEventLoopGroup-3-4] DEBUG io.netty.handler.logging.LoggingHandler:  [id: 0x68cd2b82, L:/127.0.0.1:8080 - R:/127.0.0.1:65456] WRITE: 100B
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 48 54 54 50 2f 31 2e 31 20 32 30 30 20 4f 4b 0d |HTTP/1.1 200 OK.|
|00000010| 0a 63 6f 6e 74 65 6e 74 2d 6c 65 6e 67 74 68 3a |.content-length:|
|00000020| 20 32 32 0d 0a 63 6f 6e 74 65 6e 74 2d 74 79 70 | 22..content-typ|
|00000030| 65 3a 20 74 65 78 74 2f 68 74 6d 6c 3b 63 68 61 |e: text/html;cha|
|00000040| 72 73 65 74 3d 75 74 66 2d 38 0d 0a 0d 0a 3c 68 |rset=utf-8....<h|
|00000050| 31 3e 48 65 6c 6c 6f 2c 20 77 6f 72 6c 64 21 3c |1>Hello, world!<|
|00000060| 2f 68 31 3e                                     |/h1>            |
+--------+-------------------------------------------------+----------------+
2023-03-25  21:44:32.212  [nioEventLoopGroup-3-4] DEBUG io.netty.handler.logging.LoggingHandler:  [id: 0x68cd2b82, L:/127.0.0.1:8080 - R:/127.0.0.1:65456] FLUSH
2023-03-25  21:44:32.212  [nioEventLoopGroup-3-4] DEBUG mao.t2.Server:  请求体：654
2023-03-25  21:44:32.213  [nioEventLoopGroup-3-4] DEBUG io.netty.handler.logging.LoggingHandler:  [id: 0x68cd2b82, L:/127.0.0.1:8080 - R:/127.0.0.1:65456] READ COMPLETE
2023-03-25  21:44:37.791  [nioEventLoopGroup-3-4] DEBUG io.netty.handler.logging.LoggingHandler:  [id: 0x68cd2b82, L:/127.0.0.1:8080 - R:/127.0.0.1:65456] READ COMPLETE
2023-03-25  21:44:37.791  [nioEventLoopGroup-3-4] DEBUG io.netty.handler.logging.LoggingHandler:  [id: 0x68cd2b82, L:/127.0.0.1:8080 ! R:/127.0.0.1:65456] INACTIVE
2023-03-25  21:44:37.791  [nioEventLoopGroup-3-4] DEBUG io.netty.handler.logging.LoggingHandler:  [id: 0x68cd2b82, L:/127.0.0.1:8080 ! R:/127.0.0.1:65456] UNREGISTERED
```





