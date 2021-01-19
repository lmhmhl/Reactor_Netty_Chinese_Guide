# 4 TCP客户端

Reactor Netty提供了容易使用和容易配置的TcpClient，隐藏了大部分的在需要创建TCP客户端所使用的Netty函数，并且增加了反应式流的回压机制。

## 4.1 建立连接和断开连接

为了使TCP客户端和指定端点建立连接，你必须创建和配置一个`TcpClient`的实例，默认情况下，host主机名称为localhost，端口号为12012。下面的例子展示了怎样创建TcpClient。

```java
import reactor.netty.Connection;
import reactor.netty.tcp.TcpClient;

public class Application {

	public static void main(String[] args) {
		Connection connection =
				TcpClient.create()      //1
				         .connectNow(); //2

		connection.onDispose()
		          .block();
	}
}
```

1. 创建一个TcpClient实例，可以基于此进行配置。
2. 以阻塞的方式建立连接，直到完成初始化。

返回的Connection提供了一个简单的连接API，包括disposeNow（），以阻塞的方式关闭客户端连接。

### 4.1.1. 主机名和端口号

为了连接到特定的主机名和端口号，你可以使用如下的方式配置TCP客户端。下面的例子展示了怎样配置TcpClient。

```java
import reactor.netty.Connection;
import reactor.netty.tcp.TcpClient;

public class Application {

	public static void main(String[] args) {
		Connection connection =
				TcpClient.create()
				         .host("example.com") //1
				         .port(80)            //2
				         .connectNow();

		connection.onDispose()
		          .block();
	}
}
```

1. 配置TCP主机名
2. 配置TCP端口号

## 4.2 饿汉式初始化

默认情况下，TcpClient根据需要进行初始化。这意味着connection operation将会在初始化和加载时花费额外的时间。

* 事件循环组
* 主机名称解析器
* 本地传输库
* 本地安全性库

当你想预加载如下的资源，你需要按照下面的方式进行配置TcpClient。

```java
import reactor.core.publisher.Mono;
import reactor.netty.Connection;
import reactor.netty.tcp.TcpClient;

public class Application {

	public static void main(String[] args) {
		TcpClient tcpClient =
				TcpClient.create()
				         .host("example.com")
				         .port(80)
				         .handle((inbound, outbound) -> outbound.sendString(Mono.just("hello")));

		tcpClient.warmup() //1
		         .block();

		Connection connection = tcpClient.connectNow(); //2

		connection.onDispose()
		          .block();
	}
}
```

1. 初始化和加载事件循环组，主机名称解析器，本地传输库，本地安全性库。
2. 当连接到端服务器时，会进行主机名称的解析。

## 4.3 写数据

为了向给定的端点写数据，你必须添加一个I/O handler。这个I/O handler能够访问NettyOutbound进行写数据。

```java
import reactor.core.publisher.Mono;
import reactor.netty.Connection;
import reactor.netty.tcp.TcpClient;

public class Application {

	public static void main(String[] args) {
		Connection connection =
				TcpClient.create()
				         .host("example.com")
				         .port(80)
				         .handle((inbound, outbound) -> outbound.sendString(Mono.just("hello"))) 
				         .connectNow();

		connection.onDispose()
		          .block();
	}
}
```

## 4.4 消费数据

为了从给定端点接收数据，你必须添加一个I/O handler，这个handler能够访问NettyInbound进行数据的读取。

```java
import reactor.netty.Connection;
import reactor.netty.tcp.TcpClient;

public class Application {

	public static void main(String[] args) {
		Connection connection =
				TcpClient.create()
				         .host("example.com")
				         .port(80)
				         .handle((inbound, outbound) -> inbound.receive().then()) //1
				         .connectNow();

		connection.onDispose()
		          .block();
	}
}
```

1. 从给定端点接收数据

## 4.5 生命周期内的回调

提供以下生命周期回调，以便您对TCP客户端进行扩展。

- `doOnConnect`: 在channel即将被连接时调用。
- `doOnConnected`: 在channel被连接之后调用。
- `doOnDisconnected`: channel断开连接之后进行调用。

以下示例使用`doOnConnected`回调：

```java
import io.netty.handler.timeout.ReadTimeoutHandler;
import reactor.netty.Connection;
import reactor.netty.tcp.TcpClient;
import java.util.concurrent.TimeUnit;

public class Application {

	public static void main(String[] args) {
		Connection connection =
				TcpClient.create()
				         .host("example.com")
				         .port(80)
				         .doOnConnected(conn ->
				             conn.addHandler(new ReadTimeoutHandler(10, TimeUnit.SECONDS))) //1
				         .connectNow();

		connection.onDispose()
		          .block();
	}
}
```

1. ，当管道被连接时，Netty 管道使用了ReadTimeoutHandler进行扩展。

## 4.6 TCP级别的配置

这一章节介绍了在TCP级别使用的三种配置。

- [Channel Options](https://projectreactor.io/docs/netty/release/reference/index.html#client-tcp-level-configurations-channel-options)
- [Wire Logger](https://projectreactor.io/docs/netty/release/reference/index.html#client-tcp-level-configurations-event-wire-logger)
- [Event Loop Group](https://projectreactor.io/docs/netty/release/reference/index.html#client-tcp-level-configurations-event-loop-group)

### 4.6.1. 管道选项

默认情况下，TCP客户端使用下面的配置方式。

```java
TcpClientConnect(ConnectionProvider provider) {
	this.config = new TcpClientConfig(
			provider,
			Collections.singletonMap(ChannelOption.AUTO_READ, false),
			() -> AddressUtils.createUnresolved(NetUtil.LOCALHOST.getHostAddress(), DEFAULT_PORT));
}

```

如果需要添加额外的配置，你可以使用下面的方式进行配置。

```java
import io.netty.channel.ChannelOption;
import reactor.netty.Connection;
import reactor.netty.tcp.TcpClient;

public class Application {

	public static void main(String[] args) {
		Connection connection =
				TcpClient.create()
				         .host("example.com")
				         .port(80)
				         .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 10000)
				         .connectNow();

		connection.onDispose()
		          .block();
	}
}
```

您可以在以下链接中找到有关Netty channel选项配置的更多信息：

- [`ChannelOption`](https://netty.io/4.1/api/io/netty/channel/ChannelOption.html)
- [Socket Options](https://docs.oracle.com/javase/8/docs/technotes/guides/net/socketOpt.html)

### 4.6.2. wire Logger

当需要检查对等点之间的流量时，Reactor Netty提供有日志打印的功能。默认情况下，禁用日志记录功能。若要启用它，必须将react.netty.tcp.TcpClient级别设置为DEBUG并应用以下配置：

```java
import reactor.netty.Connection;
import reactor.netty.tcp.TcpClient;

public class Application {

    public static void main(String[] args) {
        Connection connection =
                TcpClient.create()
                         .wiretap(true) 
                         .host("example.com")
                         .port(80)
                         .connectNow();

        connection.onDispose()
                  .block();
    }
}
```

#### 4.5.3. Event Loop Group

默认情况下，TCP客户端使用“事件循环组”，其中工作线程数等于初始化时可用于运行时的处理器数（但最小值为4）。需要其他配置时，可以使用LoopResource＃create方法。

以下列表显示了事件循环组的默认配置：

```java
/**
 * Default worker thread count, fallback to available processor
 * (but with a minimum value of 4)
 */
public static final String IO_WORKER_COUNT = "reactor.netty.ioWorkerCount";
/**
 * Default selector thread count, fallback to -1 (no selector thread)
 */
public static final String IO_SELECT_COUNT = "reactor.netty.ioSelectCount";
/**
 * Default worker thread count for UDP, fallback to available processor
 * (but with a minimum value of 4)
 */
public static final String UDP_IO_THREAD_COUNT = "reactor.netty.udp.ioThreadCount";
/**
 * Default quiet period that guarantees that the disposal of the underlying LoopResources
 * will not happen, fallback to 2 seconds.
 */
public static final String SHUTDOWN_QUIET_PERIOD = "reactor.netty.ioShutdownQuietPeriod";
/**
 * Default maximum amount of time to wait until the disposal of the underlying LoopResources
 * regardless if a task was submitted during the quiet period, fallback to 15 seconds.
 */
public static final String SHUTDOWN_TIMEOUT = "reactor.netty.ioShutdownTimeout";

/**
 * Default value whether the native transport (epoll, kqueue) will be preferred,
 * fallback it will be preferred when available
 */
public static final String NATIVE = "reactor.netty.native";
```

如果需要更改这些设置，则可以应用以下配置：

```java
import reactor.netty.Connection;
import reactor.netty.resources.LoopResources;
import reactor.netty.tcp.TcpClient;

public class Application {

    public static void main(String[] args) {
        LoopResources loop = LoopResources.create("event-loop", 1, 4, true);

        Connection connection =
                TcpClient.create()
                         .host("example.com")
                         .port(80)
                         .runOn(loop)
                         .connectNow();

        connection.onDispose()
                  .block();
    }
}
```

## 4.7 连接池

默认情况下，TCP客户端使用固定大小的连接池，channel的最大数量为500。允许最大的请求数量为1000，多余的请求将被放入等待队列中。如果想要获取的channel没有在连接池中，将会创建一个新的channel。当连接池中允许的channel的数量到达了上限，请求将会延迟，直到连接池中有可用的channel。

```java
/**
 * Default max connections. Fallback to
 * available number of processors (but with a minimum value of 16)
 */
public static final String POOL_MAX_CONNECTIONS = "reactor.netty.pool.maxConnections";
/**
 * Default acquisition timeout (milliseconds) before error. If -1 will never wait to
 * acquire before opening a new
 * connection in an unbounded fashion. Fallback 45 seconds
 */
public static final String POOL_ACQUIRE_TIMEOUT = "reactor.netty.pool.acquireTimeout";
/**
 * Default max idle time, fallback - max idle time is not specified.
 */
public static final String POOL_MAX_IDLE_TIME = "reactor.netty.pool.maxIdleTime";
/**
 * Default max life time, fallback - max life time is not specified.
 */
public static final String POOL_MAX_LIFE_TIME = "reactor.netty.pool.maxLifeTime";
/**
 * Default leasing strategy (fifo, lifo), fallback to fifo.
 * <ul>
 *     <li>fifo - The connection selection is first in, first out</li>
 *     <li>lifo - The connection selection is last in, first out</li>
 * </ul>
 */
public static final String POOL_LEASING_STRATEGY = "reactor.netty.pool.leasingStrategy";
/**
 * Default {@code getPermitsSamplingRate} (between 0d and 1d (percentage))
 * to be used with a {@link SamplingAllocationStrategy}.
 * This strategy wraps a {@link PoolBuilder#sizeBetween(int, int) sizeBetween} {@link AllocationStrategy}
 * and samples calls to {@link AllocationStrategy#getPermits(int)}.
 * Fallback - sampling is not enabled.
 */
public static final String POOL_GET_PERMITS_SAMPLING_RATE = "reactor.netty.pool.getPermitsSamplingRate";
/**
 * Default {@code returnPermitsSamplingRate} (between 0d and 1d (percentage))
 * to be used with a {@link SamplingAllocationStrategy}.
 * This strategy wraps a {@link PoolBuilder#sizeBetween(int, int) sizeBetween} {@link AllocationStrategy}
 * and samples calls to {@link AllocationStrategy#returnPermits(int)}.
 * Fallback - sampling is not enabled.
 */
public static final String POOL_RETURN_PERMITS_SAMPLING_RATE = "reactor.netty.pool.returnPermitsSamplingRate";
```

如果需要禁用连接池，则可以应用以下配置：

```java
import reactor.netty.Connection;
import reactor.netty.tcp.TcpClient;

public class Application {

	public static void main(String[] args) {
		Connection connection =
				TcpClient.newConnection()
				         .host("example.com")
				         .port(80)
				         .connectNow();

		connection.onDispose()
		          .block();
	}
}
```

如果需要为连接池中的通道指定空闲时间，可以应用以下配置：

```java
import reactor.netty.Connection;
import reactor.netty.resources.ConnectionProvider;
import reactor.netty.tcp.TcpClient;
import java.time.Duration;

public class Application {

	public static void main(String[] args) {
		ConnectionProvider provider =
				ConnectionProvider.builder("fixed")
				                  .maxConnections(50)
				                  .pendingAcquireTimeout(Duration.ofMillis(30000))
				                  .maxIdleTime(Duration.ofMillis(60))
				                  .build();

		Connection connection =
				TcpClient.create(provider)
				         .host("example.com")
				         .port(80)
				         .connectNow();

		connection.onDispose()
		          .block();
	}
}
```

```java
import reactor.netty.Connection;
import reactor.netty.resources.ConnectionProvider;
import reactor.netty.tcp.TcpClient;
import java.time.Duration;

public class Application {

	public static void main(String[] args) {
		ConnectionProvider provider =
				ConnectionProvider.builder("fixed")
				                  .maxConnections(50)
				                  .pendingAcquireTimeout(Duration.ofMillis(30000))
				                  .maxIdleTime(Duration.ofMillis(60))
				                  .build();

		Connection connection =
				TcpClient.create(provider)
				         .host("example.com")
				         .port(80)
				         .connectNow();

		connection.onDispose()
		          .block();
	}
}
```

当您期望高负载时，请谨慎使用最大连接值非常高的连接池。你可能会经历reactor.netty.http.client.PrematureCloseException异常由于打开/获取的并发连接太多，导致根本原因为“连接超时”的异常。

#### 4.6.1. Metrics

池中的ConnectionProvider支持与Micrometer的内置集成。它使用前缀react.netty.connection.provider公开所有度量。

Pooled `ConnectionProvider` metrics

| metric name                                           | type  | description                                                  |
| :---------------------------------------------------- | :---- | :----------------------------------------------------------- |
| reactor.netty.connection.provider.total.connections   | Gauge | The number of all connections, active or idle                |
| reactor.netty.connection.provider.active.connections  | Gauge | The number of the connections that have been successfully acquired and are in active use |
| reactor.netty.connection.provider.idle.connections    | Gauge | The number of the idle connections                           |
| reactor.netty.connection.provider.pending.connections | Gauge | The number of requests that are waiting for a connection     |

```java
import reactor.netty.Connection;
import reactor.netty.resources.ConnectionProvider;
import reactor.netty.tcp.TcpClient;

public class Application {

    public static void main(String[] args) {
        ConnectionProvider provider =
                ConnectionProvider.builder("fixed")
                                  .maxConnections(50)
                                  .metrics(true) 
                                  .build();

        Connection connection =
                TcpClient.create(provider)
                         .host("example.com")
                         .port(80)
                         .connectNow();

        connection.onDispose()
                  .block();
    }
}
```

## 4.8 SSL和SSL

当需要SSL或TLS时，可以应用以下配置。默认情况下，如果OpenSSL可用，则将SslProvider.OPENSSL提供程序用作提供程序。否则，提供程序为SslProvider.JDK。您可以通过使用SslContextBuilder或通过设置-Dio.netty.handler.ssl.noOpenSsl = true来切换提供程序。

以下示例使用SslContextBuilder：

```java
import io.netty.handler.ssl.SslContextBuilder;
import reactor.netty.Connection;
import reactor.netty.tcp.TcpClient;

public class Application {

    public static void main(String[] args) {
        SslContextBuilder sslContextBuilder = SslContextBuilder.forClient();

        Connection connection =
                TcpClient.create()
                         .host("example.com")
                         .port(443)
                         .secure(spec -> spec.sslContext(sslContextBuilder))
                         .connectNow();

        connection.onDispose()
                  .block();
    }
}
```

#### 4.7.1. Server Name Indication

默认情况下，TCP客户端将远程主机名作为SNI服务器名发送。当需要更改此默认设置时，可以按以下方式配置TCP客户端：

```java
import io.netty.handler.ssl.SslContext;
import io.netty.handler.ssl.SslContextBuilder;
import reactor.netty.Connection;
import reactor.netty.tcp.TcpClient;

import javax.net.ssl.SNIHostName;

public class Application {

    public static void main(String[] args) throws Exception {
        SslContext sslContext = SslContextBuilder.forClient().build();

        Connection connection =
                TcpClient.create()
                         .host("127.0.0.1")
                         .port(8080)
                         .secure(spec -> spec.sslContext(sslContext)
                                             .serverNames(new SNIHostName("test.com")))
                         .connectNow();

        connection.onDispose()
                  .block();
    }
}
```

### 4.9. 代理支持

TCP客户端支持Netty提供的代理功能，并提供了一种通过ProxyProvider构建器指定“非代理主机”的方法。以下示例使用ProxyProvider：

```java
import reactor.netty.Connection;
import reactor.netty.transport.ProxyProvider;
import reactor.netty.tcp.TcpClient;

public class Application {

	public static void main(String[] args) {
		Connection connection =
				TcpClient.create()
				         .host("example.com")
				         .port(80)
				         .proxy(spec -> spec.type(ProxyProvider.Proxy.SOCKS4)
				                            .host("proxy")
				                            .port(8080)
				                            .nonProxyHosts("localhost"))
				        .connectNow();

		connection.onDispose()
		          .block();
	}
}
```

### 4.10. 指标

TCP客户端支持与Micrometer的内置集成。它使用前缀react.netty.tcp.client公开所有度量。

下表提供了有关TCP客户端指标的信息：

| metric name                                 | type                | description                                     |
| :------------------------------------------ | :------------------ | :---------------------------------------------- |
| reactor.netty.tcp.client.data.received      | DistributionSummary | Amount of the data received, in bytes           |
| reactor.netty.tcp.client.data.sent          | DistributionSummary | Amount of the data sent, in bytes               |
| reactor.netty.tcp.client.errors             | Counter             | Number of errors that occurred                  |
| reactor.netty.tcp.client.tls.handshake.time | Timer               | Time spent for TLS handshake                    |
| reactor.netty.tcp.client.connect.time       | Timer               | Time spent for connecting to the remote address |
| reactor.netty.tcp.client.address.resolver   | Timer               | Time spent for resolving the address            |

这些其他指标也可用：

Pooled `ConnectionProvider` metrics

| metric name                                           | type  | description                                                  |
| :---------------------------------------------------- | :---- | :----------------------------------------------------------- |
| reactor.netty.connection.provider.total.connections   | Gauge | The number of all connections, active or idle                |
| reactor.netty.connection.provider.active.connections  | Gauge | The number of the connections that have been successfully acquired and are in active use |
| reactor.netty.connection.provider.idle.connections    | Gauge | The number of the idle connections                           |
| reactor.netty.connection.provider.pending.connections | Gauge | The number of requests that are waiting for a connection     |

`ByteBufAllocator` metrics

| metric name                                             | type  | description                                                  |
| :------------------------------------------------------ | :---- | :----------------------------------------------------------- |
| reactor.netty.bytebuf.allocator.used.heap.memory        | Gauge | The number of the bytes of the heap memory                   |
| reactor.netty.bytebuf.allocator.used.direct.memory      | Gauge | The number of the bytes of the direct memory                 |
| reactor.netty.bytebuf.allocator.used.heap.arenas        | Gauge | The number of heap arenas (when `PooledByteBufAllocator`)    |
| reactor.netty.bytebuf.allocator.used.direct.arenas      | Gauge | The number of direct arenas (when `PooledByteBufAllocator`)  |
| reactor.netty.bytebuf.allocator.used.threadlocal.caches | Gauge | The number of thread local caches (when `PooledByteBufAllocator`) |
| reactor.netty.bytebuf.allocator.used.tiny.cache.size    | Gauge | The size of the tiny cache (when `PooledByteBufAllocator`)   |
| reactor.netty.bytebuf.allocator.used.small.cache.size   | Gauge | The size of the small cache (when `PooledByteBufAllocator`)  |
| reactor.netty.bytebuf.allocator.used.normal.cache.size  | Gauge | The size of the normal cache (when `PooledByteBufAllocator`) |
| reactor.netty.bytebuf.allocator.used.chunk.size         | Gauge | The chunk size for an arena (when `PooledByteBufAllocator`)  |

以下示例启用了该集成：

```java
import reactor.netty.Connection;
import reactor.netty.tcp.TcpClient;

public class Application {

    public static void main(String[] args) {
        Connection connection =
                TcpClient.create()
                         .host("example.com")
                         .port(80)
                         .metrics(true) 
                         .connectNow();

        connection.onDispose()
                  .block();
    }
}
```



当需要TCP客户端度量标准来与Micrometer以外的系统集成时，或者您想要提供自己与Micrometer的集成时，可以提供自己的度量记录器，如下所示：

```java
import reactor.netty.Connection;
import reactor.netty.channel.ChannelMetricsRecorder;
import reactor.netty.tcp.TcpClient;

import java.net.SocketAddress;
import java.time.Duration;

public class Application {

    public static void main(String[] args) {
        Connection connection =
                TcpClient.create()
                         .host("example.com")
                         .port(80)
                         .metrics(true, CustomChannelMetricsRecorder::new) 
                         .connectNow();

        connection.onDispose()
                  .block();
    }
```



|      | Enables TCP client metrics and provides [`ChannelMetricsRecorder`](https://projectreactor.io/docs/netty/release/api/reactor/netty/channel/ChannelMetricsRecorder.html) implementation. |
| :--- | :----------------------------------------------------------- |
|      |                                                              |

### 4.11. Unix Domain Sockets

使用本地传输时，TCP客户端支持Unix域套接字（UDS）。

以下示例显示了如何使用UDS支持：

```java
import io.netty.channel.unix.DomainSocketAddress;
import reactor.netty.Connection;
import reactor.netty.tcp.TcpClient;

public class Application {

    public static void main(String[] args) {
        Connection connection =
                TcpClient.create()
                         .remoteAddress(() -> new DomainSocketAddress("/tmp/test.sock")) 
                         .connectNow();

        connection.onDispose()
                  .block();
    }
```

## 4.12 主机名称解析

默认情况下，TcpClient使用Netty的域名查找机制异步解析域名。这是JVM内置的阻塞解析器的一种替代方法。

当需要更改默认设置时，可以按以下方式配置TcpClient：

```java
import reactor.netty.Connection;
import reactor.netty.tcp.TcpClient;

import java.time.Duration;

public class Application {

	public static void main(String[] args) {
		Connection connection =
				TcpClient.create()
				         .host("example.com")
				         .port(80)
				         .resolver(spec -> spec.queryTimeout(Duration.ofMillis(500))) 
				         .connectNow();

		connection.onDispose()
		          .block();
	}
}
```

| Configuration name        | Description                                                  |
| :------------------------ | :----------------------------------------------------------- |
| `cacheMaxTimeToLive`      | 缓存的DNS资源记录的最长生存时间（分辨率：秒）。如果DNS服务器返回的DNS资源记录的生存时间大于此最大生存时间，则此解析程序将忽略来自DNS服务器的生存时间，并使用此最大生存时间。默认为Integer.MAX\u值. |
| `cacheMinTimeToLive`      | 缓存的DNS资源记录的最短生存时间（分辨率：秒）。如果DNS服务器返回的DNS资源记录的生存时间小于此最小生存时间，则此解析程序将忽略来自DNS服务器的生存时间，并使用此最小生存时间。默认值：0。 |
| `cacheNegativeTimeToLive` | 失败DNS查询的缓存的生存时间（分辨率：秒）。默认值：0。       |
| `disableOptionalRecord`   | 禁用自动包含可选记录，该记录尝试向远程DNS服务器提示解析程序每个响应可以读取多少数据。默认情况下，此设置处于启用状态。 |
| `disableRecursionDesired` | 指定此解析程序是否必须发送设置了递归所需（RD）标志的DNS查询。默认情况下，此设置处于启用状态。 |
| `maxPayloadSize`          | 设置数据报数据包缓冲区的容量（字节）。默认值：4096。         |
| `maxQueriesPerResolve`    | 设置解析主机名时允许发送的最大DNS查询数。默认值：16。        |
| `ndots`                   | 设置在进行初始绝对查询之前必须出现在名称中的点数。默认值：-1（用于确定Unix上操作系统的值，否则使用值1）。 |
| `queryTimeout`            | 设置此解析程序执行的每个DNS查询的超时时间（分辨率：毫秒）。默认值：5000。 |
| `resolvedAddressTypes`    | 解析地址的协议系列的列表。                                   |
| `roundRobinSelection`     | 启用[DnsNameResolverDnsAddResolverGroup'](https://netty.io/4.1/api/io/netty/resolver/dns/DnsAddressResolverGroup.html) |
| `runOn`                   | 在给定的[`LoopResources`](https://projectreactor.io/docs/netty/release/api/reactor/netty/resources/LoopResources.html)上执行与DNS服务器的通信. 默认情况下，将使用在客户端级别指定的LoopResources。 |
| `searchDomains`           | 解析程序的搜索域列表。默认情况下，使用系统DNS搜索域填充有效搜索域列表。 |
| `trace`                   | 在解析失败时生成详细跟踪信息时，此解析程序将使用的特定记录器和日志级别。 |

Sometimes, you may want to switch to the JVM built-in resolver. To do so, you can configure the `TcpClient` as follows:

```java
import io.netty.resolver.DefaultAddressResolverGroup;
import reactor.netty.Connection;
import reactor.netty.tcp.TcpClient;

public class Application {

	public static void main(String[] args) {
		Connection connection =
				TcpClient.create()
				         .host("example.com")
				         .port(80)
				         .resolver(DefaultAddressResolverGroup.INSTANCE) 
				         .connectNow();

		connection.onDispose()
		          .block();
	}
}
```

有时，您可能需要切换到JVM内置解析器。为此，可以按如下方式配置TcpClient：

```java
import io.netty.resolver.DefaultAddressResolverGroup;
import reactor.netty.Connection;
import reactor.netty.tcp.TcpClient;

public class Application {

	public static void main(String[] args) {
		Connection connection =
				TcpClient.create()
				         .host("example.com")
				         .port(80)
				         .resolver(DefaultAddressResolverGroup.INSTANCE) //1
				         .connectNow();

		connection.onDispose()
		          .block();
	}
```

1. 设置JVM的内置解析器