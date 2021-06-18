
Socks 协议本身是基于 TCP 的协议，位于应用层与传输层之间的会话层，将应用层的数据透明的传输到 TCP 层。更详细的介绍参见 [SOCKS - 维基百科，自由的百科全书](https://zh.wikipedia.org/wiki/SOCKS)

<!--more-->

## Socks5协议

Socks5 协议在进行代理通信时，首先需要进行连接、认证等步骤。

  - 协商：主要是 Client 端请求代理服务对认证方式进行协商
  - 认证：如果第一步确认需要进行认证，则发送对应密钥进行认证
  - 连接：通过代理连接目标网站
  - 通信：进行 Http 等应用层通信，与原始 TCP 通信一致

下面就每一步进行编码。

## 协商

在协商阶段请求格式为：

```
+--------------------------------+
|   VER    | NMETHODS |  METHODS |
|----------|----------|----------|
|    1     |     1    |     1    |
+--------------------------------+
```

  - MehotdsVER：是SOCKS版本，这里应该是0x05；
  - NMETHODS：是METHODS部分的长度；
  - METHODS：是客户端支持的认证方式列表，每个方法占1字节。当前的定义是：
    - 0x00 不需要认证
    - 0x01 GSSAPI
    - 0x02 用户名、密码认证
    - 0x03 - 0x7F由IANA分配（保留）
    - 0x80 - 0xFE为私人方法保留
    - 0xFF 无可接受的方法

```
buffer.put((byte) 5);
buffer.put((byte) 1);
buffer.put((byte) 0);
```

> 我自己的代理本身没有认证，所以最后一个字段为 `0`

代理返回的结果：

```
+---------------------+
|   VER    | NMETHODS |
|----------|----------|
|    1     |     1    |
+---------------------+
```

  - VER：是SOCKS版本，这里应该是0x05；
  - METHOD：是服务端选中的方法。如果返回0xFF表示没有一个认证方法被选中，客户端需要关闭连接。

## 认证

请求格式：

```
+----------+----------+----------+----------+----------+
|   VER    |   NLEN   |   NAME   |   PLEN   |    PW    |
+------------------------------------------------------+
|    1     |     1    |   VAR    |     1    |    VAR   |
+----------+----------+----------+----------+----------+

```

代理响应结果：

```
+---------------------+
|   VER    |  STATES  |
|----------|----------|
|    1     |     1    |
+---------------------+
```

> 其中STATES 0x00 表示成功，0x01 表示失败

## 连接

连接请求格式如下：
```
+----------+----------+----------+----------+----------+----------+
|   VER    |   CMD    |    RSV   |   ATYP   | DST.ADDR | DST.PORT |
+-----------------------------------------------------------------+
|    1     |     1    |   0X00   |     1    |    VAR   |    2   |
+----------+----------+----------+----------+----------+----------+
```

  - VER：SOCKS版本，这里应该是0x05；
  - CMD：SOCK的命令码
    - 0x01：表示CONNECT请求
    - 0x02：表示BIND请求
    - 0x03：表示UDP转发
  - RSV：0x00，保留
  - ATYP：DST.ADDR类型
    - 0x01：IPv4地址，DST.ADDR部分4字节长度
    - 0x03：域名，DST.ADDR部分第一个字节为域名长度，DST.ADDR剩余的内容为域名，没有\0结尾。
    - 0x04：IPv6地址，16个字节长度。
  - DST.ADDR：目的地址
  - DST.PORT：网络字节序表示的目的端口

代码如下：

```
buffer.put((byte) 5);
buffer.put((byte) 1);
buffer.put((byte) 0);
buffer.put((byte) 3);
//BND.ADDR
buffer.put((byte) toHex(HTTP_HOST.length())); //HTTP 域名
buffer.put(HTTP_HOST.getBytes());
//BND.PORT
buffer.put((byte) 0);
buffer.put((byte) toHex(80));
```

代理响应结果如下：

```
+----------+----------+----------+----------+----------+----------+
|   VER    |   CMD    |    RSV   |   ATYP   | BND.ADDR | BND.PORT |
+-----------------------------------------------------------------+
|    1     |     1    |   0X00   |     1    |    VAR   |     2    |
+----------+----------+----------+----------+----------+----------+
```

## 通信

在通过代理建立远程连接之后，直接可以通过该链路进行通信，这里选用 HTTP 进行简单的 GET 请求。完整代码如下：


## Socks 代理 & Http

```java
public class Main {
    private static final ByteBuffer buffer = ByteBuffer.allocate(1024);

    private static final AtomicInteger step = new AtomicInteger(0);
    public static final String HTTP_HOST = "www.baidu.com";

    public static void main(String[] args) throws Exception {
        Selector selector = Selector.open();

        SocketChannel channel = SocketChannel.open();
        channel.socket().connect(new InetSocketAddress("127.0.0.1", 1086));
        System.out.println("与代理建立连接");

        channel.configureBlocking(false);
        channel.register(selector, SelectionKey.OP_READ | SelectionKey.OP_WRITE);

        while (channel.isConnected()) {
            TimeUnit.MILLISECONDS.sleep(100);
            if (selector.select() == 0) {
                continue;
            }

            Set<SelectionKey> selectionKeys = selector.selectedKeys();
            for (SelectionKey selectionKey : selectionKeys) {
                if (selectionKey.isWritable()) {
                    onWritable(selectionKey);
                } else if (selectionKey.isReadable()) {
                    onReadable(selectionKey);
                }
            }
        }
    }

    private static void onWritable(SelectionKey selectionKey) {
        SocketChannel channel = (SocketChannel) selectionKey.channel();
        try {
            synchronized (buffer) {
                buffer.clear();

                switch (step.get()) {
                    case 0:
                        buffer.put((byte) 5);
                        buffer.put((byte) 1);
                        buffer.put((byte) 0);
                        break;
                    case 1:
                        buffer.put((byte) 5);
                        buffer.put((byte) 1);
                        buffer.put((byte) 0);
                        buffer.put((byte) 3);
                        
                        buffer.put((byte) toHex(HTTP_HOST.length()));
                        buffer.put(HTTP_HOST.getBytes());
                        
                        buffer.put((byte) 0);
                        buffer.put((byte) toHex(80));
                        break;
                    case 2:
                        String httpGet = "GET / HTTP/1.1\r\n" +
                                "host: " + HTTP_HOST + "\r\n" +
                                "User-Agent: curl/7.54.0\r\n" +
                                "Accept: */*\r\n" +
                                "\r\n";

                        buffer.put(httpGet.getBytes());
                        break;
                    case 3:
                        return;
                }
                System.out.println("step:" + step.get() + ", write:" + Arrays.toString(Arrays.copyOf(buffer.array(), buffer.position())));

                buffer.flip();
                channel.write(buffer);
                buffer.compact();
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    static int toHex(int source) {
        String hexString = Integer.toHexString(source);

        return Integer.parseInt(hexString, 16);
    }

    private static void onReadable(SelectionKey selectionKey) {
        SocketChannel channel = (SocketChannel) selectionKey.channel();
        try {
            synchronized (buffer) {
                buffer.clear();
                channel.read(buffer);
                byte[] res = Arrays.copyOf(buffer.array(), buffer.position());
                System.out.println("step:" + step.get() + ", rec:" + Arrays.toString(res));

                switch (step.get()) {
                    case 0:
                        if (res[1] == 0) {
                            step.incrementAndGet();
                        }
                        break;
                    case 1:
                        if (res[1] == 0) {
                            step.incrementAndGet();
                        }
                        break;
                    case 2:
                        System.out.println("httpResponse:" + new String(res, Charset.forName("UTF-8")));
                        step.incrementAndGet();
                        break;
                }

            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```
