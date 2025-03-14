---
title: WebSocket在C#与.NET中的简单实现
date: 2024-08-18 20:53:59
categories:
- Network-Protocol
tags:
- WebSocket
- .NET
---

WebSocket的基本概念、应用场景，以及服务端和客户端在C#与.NET中的简单实现(包含心跳检测与自动重连)。

<!--more-->

## 前言

WebSocket是一种在单个TCP连接上进行全双工通信的协议，它可以让客户端和服务器之间进行实时的双向通信。WebSocket使用一个长连接，在客户端和服务器之间保持持久的连接，从而可以实时地发送和接收数据。

## 基本概念

### OSI七层网络模型

OSI(Open System Interconnect)七层网络模型是一种将计算机网络体系结构按照功能划分为七层的标准模型。

- 应用层(Application Layer)，负责提供应用程序之间的通信服务，使得不同的应用程序可以在网络上进行数据交换和通信。常见协议有HTTP、HTTPS、FTP、POP3、SSH、DNS等。
- 表示层(Presentation Layer)，负责处理数据在网络上传输时的格式和编码，以确保不同系统之间的数据交换能够有效地进行。常见协议有JPEG、PNG、MP3等。
- 会话层(Session Layer)，负责建立、管理和终止应用程序之间的会话。常见协议有NetBIOS(网络基本输入/输出系统)、RPC等。
- 传输层(Transport Layer)，负责在不可靠的网络上提供可靠的数据传输服务。常见协议有TCP、UDP、SSL(安全套接层协议)、TLS(传输层安全性协议)等。TCP协议面向连接、可靠，UDP协议无连接、不可靠。
- 网络层(Network Layer)，负责将数据包从源主机传输到目标主机。常见协议有IP等。
- 数据链路层(Data Link Layer)，负责将网络层传输过来的数据包进行分帧，并在物理介质上进行传输。常见协议有IEEE802.2(逻辑链路控制标准)、PPP(点对点通信)等。
- 物理层(Physical Layer)，负责将数字数据转换成物理信号并在网络中传输。常见协议有RS232(串行通信接口标准)、IEEE802.3(以太网标准)等。

### 串口通信与网口通信

WebSocket有串口、网口两种通信方式。

- 串口方式‌主要基于串行接口进行数据传输，采用串口通信协议(如RS232、RS485等)。这种方式适用于点对点的数据传输，使用有限连接，只能连接两台设备，不支持网络中的多台设备之间的通信。串口通信传输速度较慢，传输距离较长，比较稳定，可以确保数据传输的可靠性。

- 网口方式‌则基于网络通信协议(如TCP/IP、UDP等)进行数据传输。这种方式使用无限连接，适用于网络中的多台设备之间的通信。网口通信传输速度较快，传输距离有限。

### 与TCP和HTTP的关系

WebSocket协议是独立的基于TCP的协议。它和HTTP的唯一关系是建立连接的握手操作的升级请求是基于HTTP服务器的。

WebSocket默认使用80端口进行连接，而基于TLS(RFC2818)的WebSocket连接是基于443端口的。

## 应用场景

创建一个客户端和服务端的双向数据Web应用(例如IM应用和游戏应用)需要向服务端频繁发送不同于一般HTTP请求的HTTP轮询请求来从服务端上游更新数据，这个方法存在许多问题：

- 服务端被迫使用大量的的潜在的TCP连接与客户端进行交互：一部分是用来发送数据，而另一部分是用来接收数据
- 应用层无线传输协议(HTTP)开销较大，每一个客户端到服务端的消息都有一个HTTP头
- 客户端脚本必须包含一个发送和接收对应的映射表来进行对应数据处理

一个简单的解决方案是使用一个简单的TCP链接来进行双向数据传输，这就是WebSocket提供的能力。结合WebSocket的API，它能够提供一个可以替代HTTP轮询的方法来满足Web页面和远端服务器的双向数据通信。

## 实现

### 原生实现

C#中可以通过System.Net.WebSockets命名空间提供的类来实现WebSocket通信。

#### 服务端

首先创建一个WebSocket服务端类，

```c#
public class WebSocketServer: IDisposable
{
    private readonly string _serverUri;
    private readonly HttpListener _httpListener;
    private WebSocket _webSocket;
    private bool disposedValue;

    public WebSocketServer(string serverUri)
    {
        _serverUri = serverUri;
        _httpListener = new HttpListener();
    }

    public async Task StartAsync()
    {
        _httpListener.Prefixes.Add(_serverUri);
        _httpListener.Start();
        Console.WriteLine($"WebSocket服务已启动，服务地址：{_serverUri}，等待客户端连接...");

        while (true)
        {
            HttpListenerContext httpContext = await _httpListener.GetContextAsync();

            if (httpContext.Request.IsWebSocketRequest)
            {
                await HandleWebSocketConnectionAsync(httpContext);
            }
            else
            {
                httpContext.Response.StatusCode = 400;
                httpContext.Response.Close();
            }
        }
    }

    private async Task HandleWebSocketConnectionAsync(HttpListenerContext httpContext)
    {
        WebSocketContext webSocketContext = null;
        try
        {
            webSocketContext = await httpContext.AcceptWebSocketAsync(subProtocol: null);
            Console.WriteLine("接收到客户端连接，WebSocket连接已建立，等待客户端消息...");
        }
        catch (Exception e)
        {
            httpContext.Response.StatusCode = 500;
            httpContext.Response.Close();
            Console.WriteLine("WebSocket连接失败: " + e.Message);
            return;
        }

        _webSocket = webSocketContext.WebSocket;
        byte[] buffer = new byte[1024];

        while (_webSocket.State == WebSocketState.Open)
        {
            WebSocketReceiveResult result = null;

            try
            {
                result = await _webSocket.ReceiveAsync(new ArraySegment<byte>(buffer), CancellationToken.None);
            }
            catch (Exception e)
            {
                Console.WriteLine("WebSocket消息接收错误 :" + e.Message);
                break;
            }

            if (result.MessageType == WebSocketMessageType.Close)
            {
                await _webSocket.CloseAsync(WebSocketCloseStatus.NormalClosure, "Closing", CancellationToken.None);
                Console.WriteLine("WebSocket连接已关闭!");
            }
            else if (result.MessageType == WebSocketMessageType.Text)
            {
                // 处理文本消息
                string message = Encoding.UTF8.GetString(buffer, 0, result.Count);
                Console.WriteLine("接收到客户端的消息: " + message);

                byte[] response = Encoding.UTF8.GetBytes("服务端已成功收到消息" + message);
                await _webSocket.SendAsync(new ArraySegment<byte>(response), WebSocketMessageType.Text, true, CancellationToken.None);
            }
            else if (result.MessageType == WebSocketMessageType.Binary)
            {
                // 处理二进制消息
            }
        }
    }

    protected virtual void Dispose(bool disposing)
    {
        if (!disposedValue)
        {
            if (disposing)
            {
                // TODO: 释放托管状态(托管对象)
            }

            // TODO: 释放未托管的资源(未托管的对象)并重写终结器
            // TODO: 将大型字段设置为 null
            disposedValue = true;
            _webSocket.Dispose();
        }
    }

    // TODO: 仅当“Dispose(bool disposing)”拥有用于释放未托管资源的代码时才替代终结器
    ~WebSocketServer()
    {
        // 不要更改此代码。请将清理代码放入“Dispose(bool disposing)”方法中
        Dispose(disposing: false);
    }

    public void Dispose()
    {
        // 不要更改此代码。请将清理代码放入“Dispose(bool disposing)”方法中
        Dispose(disposing: true);
        GC.SuppressFinalize(this);
    }
}
```

然后启动WebSocket服务。

```c#
static async Task Main(string[] args)
{
    string serverUri = "http://localhost:8181/";

    var webSocketServer = new WebSocketServer(serverUri);

    await webSocketServer.StartAsync();
}
```

#### 客户端

首先创建一个WebSocket客户端类，

```c#
public class WebSocketClient: IDisposable
{
    private readonly string _uri;
    private readonly ClientWebSocket _clientWebSocket;
    private bool disposedValue;

    public WebSocketClient(string uri)
    {
        _uri = uri;
        _clientWebSocket = new ClientWebSocket();
    }

    public async Task StartAsync()
    {
        await _clientWebSocket.ConnectAsync(new Uri(_uri), CancellationToken.None);
        Console.WriteLine("WebSocket客户端已连接至：" + _uri);

        byte[] buffer = new byte[1024];

        var input = Console.ReadLine();
        while (input != "exit")
        {
            byte[] messageBytes = Encoding.UTF8.GetBytes(input);
            await _clientWebSocket.SendAsync(new ArraySegment<byte>(messageBytes), WebSocketMessageType.Text, true, CancellationToken.None);
            Console.WriteLine("客户端发送消息: " + input);

            WebSocketReceiveResult result = await _clientWebSocket.ReceiveAsync(new ArraySegment<byte>(buffer), CancellationToken.None);
            string response = Encoding.UTF8.GetString(buffer, 0, result.Count);
            Console.WriteLine("客户端接收到消息: " + response);

            input = Console.ReadLine();
        }

        await _clientWebSocket.CloseAsync(WebSocketCloseStatus.NormalClosure, "Closing", CancellationToken.None);
        Console.WriteLine("WebSocket客户端已关闭!");
    }

    protected virtual void Dispose(bool disposing)
    {
        if (!disposedValue)
        {
            if (disposing)
            {
                // TODO: 释放托管状态(托管对象)
            }

            // TODO: 释放未托管的资源(未托管的对象)并重写终结器
            // TODO: 将大型字段设置为 null
            disposedValue = true;
            _clientWebSocket.Dispose();
        }
    }

    // TODO: 仅当“Dispose(bool disposing)”拥有用于释放未托管资源的代码时才替代终结器
    ~WebSocketServer()
    {
        // 不要更改此代码。请将清理代码放入“Dispose(bool disposing)”方法中
        Dispose(disposing: false);
    }

    public void Dispose()
    {
        // 不要更改此代码。请将清理代码放入“Dispose(bool disposing)”方法中
        Dispose(disposing: true);
        GC.SuppressFinalize(this);
    }
}
```

然后启动WebSocket客户端。

```c#
static async Task Main(string[] args)
{
    string serverUri = "http://localhost:8181/";

    var webSocketClient = new WebSocketClient(serverUri.Replace("http", "ws")); // http升级请求

    await webSocketClient.StartAsync(); 
}
```

#### 支持WSS

若要使用WebSocket Secure(WSS)，即在WebSocket上使用TLS/SSL加密通信，则需要对服务端和客户端进行一些调整。

首先服务端的uri前缀需要改为https，

```c#
static async Task Main(string[] args)
{
    string serverUri = "https://localhost:8182/";

    var webSocketServer = new WebSocketServer(serverUri);

    await webSocketServer.StartAsync();
}
```

然后客户端的http升级请求需要改为https，即从http-ws调整为https-wss，

```c#
static async Task Main(string[] args)
{
    string serverUri = "https://localhost:8182/";

    var webSocketClient = new WebSocketClient(serverUri.Replace("https", "wss")); // https升级请求

    await webSocketClient.StartAsync(); 
}
```

如果服务器使用自签名证书(未经过CA认证)，客户端默认会抛出SSL错误，测试环境中可通过在代码中忽略证书验证的方式解决。

```c#
public class WebSocketClient
{
    ...
    public async Task StartAsync()
    {
        // 忽略自签名证书验证
        ServicePointManager.ServerCertificateValidationCallback += (sender, certificate, chain, sslPolicyErrors) => true;
        ...
    }
}
```

自签名证书可以使用OpenSSL生成，

`openssl req -x509 -newkey rsa:2048 -keyout key.pem -out cert.pem -days 365 -nodes`

这将生成cert.pem(证书)和key.pem(私钥)，然后使用以下命令将PEM文件转换为PFX文件。(此PFX文件导入后颁发者默认为Internet Widgits Pty Ltd)

`openssl pkcs12 -export -out certificate.pfx -inkey key.pem -in cert.pem`

若服务端使用HttpListener，需确保证书已绑定到服务器的端口。例如，在Windows上可以使用netsh命令，(注：IP一般使用通配地址0.0.0.0)

`netsh http add sslcert ipport=<IP>:<PORT> certhash=<Certificate Thumbprint> appid={<Application GUID>}`

证书指纹可通过`win+r`输入`mmc`并添加证书单元格后在受信任的根证书颁发机构中查看，也可通过openssl命令获取。(注：通过命令获取到的指纹中的`:`需要删除，否则绑定时会报参数错误)

`openssl x509 -in cert.pem -noout -fingerprint`

可使用netsh命令检查证书是否绑定成功。

`netsh http show sslcert`

#### 支持心跳检测与自动重连

WebSocket心跳检测(Heartbeat)是一种机制，用于确保客户端和服务器之间的连接仍然活跃，并且能够及时检测到任何连接问题。在WebSocket连接中，由于网络不稳定或服务器重启等原因，可能会导致连接意外断开，而双方并不知情。心跳检测通过定期发送小的数据包(如Ping/Pong帧)来验证连接是否仍然存在。

在ClientWebSocket中，SendAsync和ReceiveAsync是异步操作，但它们有一个重要的限制：每个操作(SendAsync和ReceiveAsync)在同一时间只能有一个未完成的任务。如果尝试同时发起多个SendAsync或ReceiveAsync调用，就会抛出异常

>There is already one outstanding 'SendAsync' call for this WebSocket instance. ReceiveAsync and SendAsync can be called simultaneously, but at most one outstanding operation for each of them is allowed at the same time.

ClientWebSocket的设计是线程安全的，但它不允许同时发起多个SendAsync或ReceiveAsync操作。如果你在一个线程中调用SendAsync，而在同一个操作完成之前又在另一个线程中调用SendAsync，就会触发上述异常。(ReceiveAsync同理)

在ServerWebSocket的心跳检测中，如果使用异步方式调用SendAsync，可能会导致服务在"Aborted"状态下仍然被用于通信(理论上只有"Open"状态下才应当执行)，从而引发异常

>The 'System.Net.WebSockets.ServerWebSocket' instance cannot be used for communication because it has been transitioned into the 'Aborted' state

可以通过加锁的方式确保线程安全。在异步方法中可以使用SemaphoreSlim来实现异步锁，避免阻塞线程。

下面我们来实现WebSocket服务端与客户端各自的心跳检测机制，以及客户端自动重连机制。首先服务端需要启动心跳检测任务，并对客户端发送过来的Ping消息作出Pong回应。服务端完整代码如下：

```c#
public class WebSocketServer: IDisposable
{
    private readonly string _serverUri;
    private readonly HttpListener _httpListener;
    private WebSocket _webSocket;
    private CancellationTokenSource _cancellationTokenSource;
    private CancellationToken _cancellationToken { get { return _cancellationTokenSource == null ? CancellationToken.None : _cancellationTokenSource.Token; } }
    private const int HeartbeatInterval = 5000;  // 心跳间隔时间
    private const string PingMessage = "Ping";
    private const string PongMessage = "Pong";
    //private readonly object _sendLock = new object();
    private readonly SemaphoreSlim _sendSemaphore = new SemaphoreSlim(1, 1);  // 使用SemaphoreSlim实现异步锁，初始化时设置最大并发数为 1
    //private readonly object _receiveLock = new object();
    private readonly SemaphoreSlim _receiveSemaphore = new SemaphoreSlim(1, 1);  // 使用SemaphoreSlim实现异步锁，初始化时设置最大并发数为 1
    private bool disposedValue;

    public WebSocketServer(string serverUri)
    {
        _serverUri = serverUri;
        _httpListener = new HttpListener();
    }

    public async Task StartAsync()
    {
        _httpListener.Prefixes.Add(_serverUri);
        _httpListener.Start();
        Console.WriteLine($"WebSocket服务已启动，服务地址：{_serverUri}，等待客户端连接...");

        while (true)
        {
            HttpListenerContext httpContext = await _httpListener.GetContextAsync();

            if (httpContext.Request.IsWebSocketRequest)
            {
                await HandleWebSocketConnectionAsync(httpContext);
            }
            else
            {
                httpContext.Response.StatusCode = 400;
                httpContext.Response.Close();
            }
        }
    }

    private async Task HandleWebSocketConnectionAsync(HttpListenerContext httpContext)
    {
        WebSocketContext webSocketContext = null;
        try
        {
            webSocketContext = await httpContext.AcceptWebSocketAsync(subProtocol: null);
            Console.WriteLine("接收到客户端连接，WebSocket连接已建立，等待客户端消息...");
        }
        catch (Exception e)
        {
            httpContext.Response.StatusCode = 500;
            httpContext.Response.Close();
            Console.WriteLine("WebSocket连接失败: " + e.Message);
            return;
        }

        _webSocket = webSocketContext.WebSocket;
        _cancellationTokenSource = new CancellationTokenSource();
        byte[] buffer = new byte[1024];

        // 开始心跳检测
        var heartbeatTask = HeartbeatAsync();

        while (_webSocket.State == WebSocketState.Open)
        {
            WebSocketReceiveResult result = null;

            try
            {
                await _receiveSemaphore.WaitAsync(_cancellationToken); // 等待锁
                result = await _webSocket.ReceiveAsync(new ArraySegment<byte>(buffer), _cancellationToken);
            }
            catch (Exception e)
            {
                Console.WriteLine("WebSocket消息接收错误 :" + e.Message);
                break;
            }
            finally
            {
                _receiveSemaphore.Release(); // 释放锁
            }

            if (result.MessageType == WebSocketMessageType.Close)
            {
                await _webSocket.CloseAsync(WebSocketCloseStatus.NormalClosure, "Closing", _cancellationToken);
                Console.WriteLine("WebSocket连接已关闭!");
            }
            else if (result.MessageType == WebSocketMessageType.Text)
            {
                // 处理文本消息
                string message = Encoding.UTF8.GetString(buffer, 0, result.Count);
                Console.WriteLine("接收到来自客户端的消息: " + message);

                await SendMessageAsync("服务端已成功收到消息" + message);

                if (message == PingMessage)
                {
                    await SendMessageAsync(PongMessage);
                    Console.WriteLine($"{PongMessage}已发送.");
                }
            }
            else if (result.MessageType == WebSocketMessageType.Binary)
            {
                // 处理二进制消息
            }
        }

        _cancellationTokenSource.Cancel();
        await heartbeatTask;
    }

    private async Task SendMessageAsync(string message)
    {
        try
        {
            await _sendSemaphore.WaitAsync(_cancellationToken); // 等待锁
            var msg = Encoding.UTF8.GetBytes(message);
            await _webSocket.SendAsync(new ArraySegment<byte>(msg), WebSocketMessageType.Text, true, _cancellationToken);
        }
        finally
        {
            _sendSemaphore.Release(); // 释放锁
        }
    }

    private async Task HeartbeatAsync()
    {
        while (!_cancellationToken.IsCancellationRequested)
        {
            try
            {
                await Task.Delay(HeartbeatInterval, _cancellationToken); 
                if (_webSocket.State == WebSocketState.Open)
                {
                    await SendMessageAsync(PingMessage);
                    Console.WriteLine($"{PingMessage}已发送.");
                }
                else
                {
                    Console.WriteLine("WebSocket已关闭，心跳已停止.");
                    break;
                }
            }
            catch (OperationCanceledException)
            {
                // 任务被取消，跳出循环
                Console.WriteLine("心跳检测任务被取消，跳出循环");
                break;
            }
            catch (Exception ex)
            {
                Console.WriteLine($"心跳检测错误: {ex.Message}");
                break;
            }
        }
    }

    protected virtual void Dispose(bool disposing)
    {
        if (!disposedValue)
        {
            if (disposing)
            {
                // TODO: 释放托管状态(托管对象)
            }

            // TODO: 释放未托管的资源(未托管的对象)并重写终结器
            // TODO: 将大型字段设置为 null
            disposedValue = true;
            _webSocket.Dispose();
            _cancellationTokenSource.Dispose();
        }
    }

    // TODO: 仅当“Dispose(bool disposing)”拥有用于释放未托管资源的代码时才替代终结器
    ~WebSocketServer()
    {
        // 不要更改此代码。请将清理代码放入“Dispose(bool disposing)”方法中
        Dispose(disposing: false);
    }

    public void Dispose()
    {
        // 不要更改此代码。请将清理代码放入“Dispose(bool disposing)”方法中
        Dispose(disposing: true);
        GC.SuppressFinalize(this);
    }
}
```

然后客户端同样需要启动心跳检测任务，并对服务端发送过来的Ping消息作出Pong回应。此外，客户端还需要确保在连接断开后能自动重连。客户端完整代码如下：

```c#
public class WebSocketClient : IDisposable
{
    private readonly string _uri;
    private ClientWebSocket _clientWebSocket;
    private CancellationTokenSource _cancellationTokenSource;
    private CancellationToken _cancellationToken { get { return _cancellationTokenSource.Token; } }
    private const int HeartbeatInterval = 5000;  // 心跳间隔时间
    private const int ReconnectDelay = 10000; // 重连延迟时间
    private const string PingMessage = "Ping";
    private const string PongMessage = "Pong";
    //private readonly object _sendLock = new object();
    private readonly SemaphoreSlim _sendSemaphore = new SemaphoreSlim(1, 1); // 使用SemaphoreSlim实现异步锁，初始化时设置最大并发数为 1
    //private readonly object _receiveLock = new object();
    private readonly SemaphoreSlim _receiveSemaphore = new SemaphoreSlim(1, 1); // 使用SemaphoreSlim实现异步锁，初始化时设置最大并发数为 1
    private bool disposedValue;

    public WebSocketClient(string uri)
    {
        _uri = uri;
    }

    public async Task StartAsync()
    {
        // 忽略自签名证书验证
        ServicePointManager.ServerCertificateValidationCallback += (sender, certificate, chain, sslPolicyErrors) => true;

        while (true)
        {
            try
            {
                _clientWebSocket = new ClientWebSocket(); // 对于每一次连接尝试，都要重新创建一个ClientWebSocket实例，否则会一直提示"The WebSocket has already been started"
                _cancellationTokenSource = new CancellationTokenSource(); // 对于每一次连接尝试，都要重新创建一个CancellationTokenSource实例，否则会一直提示"A task was canceled"
                await _clientWebSocket.ConnectAsync(new Uri(_uri), _cancellationToken); // 无法对一个已经启动或已经关闭的ClientWebSocket实例再次调用ConnectAsync 

                // 开始心跳检测
                var heartbeatTask = HeartbeatAsync();
                Console.WriteLine($"客户端已连接上WebSocket服务器，握手请求后的服务地址为{_uri}");

                // 接收消息
                await ReceiveMessagesAsync();

                _cancellationTokenSource.Cancel();
                await heartbeatTask;
            }
            catch (Exception ex)
            {
                Console.WriteLine($"WebSocket错误：{ex.Message}");
            }
            Console.WriteLine($"在{ReconnectDelay / 1000}秒后尝试重连...");
            await Task.Delay(ReconnectDelay);
        }
    }

    private async Task ReceiveMessagesAsync()
    {
        var buffer = new byte[1024];

        while (_clientWebSocket.State == WebSocketState.Open)
        {
            try
            {
                    await _receiveSemaphore.WaitAsync(_cancellationToken); // 等待锁

                    var result = await _clientWebSocket.ReceiveAsync(new ArraySegment<byte>(buffer), _cancellationToken);
                    if (result.MessageType == WebSocketMessageType.Close)
                    {
                        await _clientWebSocket.CloseAsync(WebSocketCloseStatus.NormalClosure, "Closing", _cancellationToken);
                        Console.WriteLine("WebSocket客户端已关闭!");
                    }
                    else
                    {
                        var message = Encoding.UTF8.GetString(buffer, 0, result.Count);
                        Console.WriteLine($"接收到来自服务端的消息: {message}");

                        if (message == PingMessage)
                        {
                            await SendMessageAsync(PongMessage);
                            Console.WriteLine($"{PongMessage}已发送.");
                        }
                    }
            }
            catch (OperationCanceledException)
            {
                // 任务被取消，跳出循环
                Console.WriteLine("消息接收任务被取消，跳出循环");
                break;
            }
            catch (Exception ex)
            {
                Console.WriteLine($"客户端接收消息发生错误: {ex.Message}");
                break;
            }
            finally
            {
                _receiveSemaphore.Release(); // 释放锁
            }
        }

        await Task.CompletedTask;
    }

    private async Task SendMessageAsync(string message)
    {
        try
        {
            await _sendSemaphore.WaitAsync(_cancellationToken); // 等待锁
            var msg = Encoding.UTF8.GetBytes(message);
            await _clientWebSocket.SendAsync(new ArraySegment<byte>(msg), WebSocketMessageType.Text, true, _cancellationToken);
        }
        finally
        {
            _sendSemaphore.Release(); // 释放锁
        }
    }

    private async Task HeartbeatAsync()
    {
        while (!_cancellationToken.IsCancellationRequested)
        {
            try
            {
                await Task.Delay(HeartbeatInterval, _cancellationToken);
                if (_clientWebSocket.State == WebSocketState.Open)
                {
                    await SendMessageAsync(PingMessage);
                    Console.WriteLine($"{PingMessage}已发送.");
                }
                else
                {
                    Console.WriteLine("WebSocket已关闭，心跳已停止.");
                    break;
                }
            }
            catch (OperationCanceledException)
            {
                // 任务被取消，跳出循环
                Console.WriteLine("心跳检测任务被取消，跳出循环");
                break;
            }
            catch (Exception ex)
            {
                Console.WriteLine($"心跳检测错误: {ex.Message}");
                break;
            }
        }
    }

    protected virtual void Dispose(bool disposing)
    {
        if (!disposedValue)
        {
            if (disposing)
            {
                // TODO: 释放托管状态(托管对象)
            }

            // TODO: 释放未托管的资源(未托管的对象)并重写终结器
            // TODO: 将大型字段设置为 null
            disposedValue = true;
            _clientWebSocket.Dispose();
            _cancellationTokenSource.Dispose();
        }
    }

    // TODO: 仅当“Dispose(bool disposing)”拥有用于释放未托管资源的代码时才替代终结器
    ~WebSocketClient()
    {
        // 不要更改此代码。请将清理代码放入“Dispose(bool disposing)”方法中
        Dispose(disposing: false);
    }

    public void Dispose()
    {
        // 不要更改此代码。请将清理代码放入“Dispose(bool disposing)”方法中
        Dispose(disposing: true);
        GC.SuppressFinalize(this);
    }
}
```

下图展示了客户端反复掉线重连时，服务端控制台的输出情况。

{% asset_img websocket_server_console.png  服务端控制台的输出情况 %}

下图展示了服务端反复重启时，客户端控制台的输出情况。

{% asset_img websocket_client_console.png  客户端控制台的输出情况 %}

### 第三方库实现

.NET中也可以引入Nuget包来实现WebSocket通信。

#### 服务端

Fleck是C#中的一个WebSocket服务器实现，Fleck不依赖于HttpListener。下面借助Fleck来模拟一个WebSocket服务端。

```c#
using Fleck;

class Program
{
    static void Main(string[] args)
    {
        FleckLog.Level = LogLevel.Debug;
        var allSockets = new List<IWebSocketConnection>();
        var server = new WebSocketServer("ws://127.0.0.1:8181");
        server.Start(socket =>
        {
            socket.OnOpen = () =>
            {
                Console.WriteLine("客户端连接成功!");
                allSockets.Add(socket);
                Console.WriteLine("当前客户端数量：" + allSockets.ToList().Count);
            };
            socket.OnClose = () =>
            {
                Console.WriteLine("客户端已经关闭!");
                allSockets.Remove(socket);
                Console.WriteLine("当前客户端数量：" + allSockets.ToList().Count);
            };
            socket.OnMessage = message =>
            {
                Console.WriteLine(message);
                allSockets.ToList().ForEach(s => s.Send(message));
            };
        });

        var input = Console.ReadLine();
        while (input != "exit")
        {
            foreach (var socket in allSockets.ToList())
            {
                socket.Send(input);
            }
            input = Console.ReadLine();
        }
    }
}
```

#### 客户端

WebSocket4Net是基于.NET的一个WebSocket客户端实现。下面借助WebSocket4Net来模拟一个WebSocket客户端。

```c#
using WebSocket4Net;

class Program
{
    public static WebSocket webSocket4Net = null;

    private static void WebSocket4Net_Opened(object sender, EventArgs e)
    {
        webSocket4Net.Send($"客户端连接成功，准备发送数据！");
    }

    private static void WebSocket4Net_MessageReceived(object sender, MessageReceivedEventArgs e)
    {
        Console.WriteLine($"服务端回复数据:{e.Message}");
    }

    public static void ClientSendMsgToServer(object input)
    {
        webSocket4Net.Send((string)input);
    }

    static void Main(string[] args)
    {
        webSocket4Net = new WebSocket("ws://127.0.0.1:8181");
        webSocket4Net.Opened += WebSocket4Net_Opened;
        webSocket4Net.MessageReceived += WebSocket4Net_MessageReceived;
        webSocket4Net.Open();

        var input = Console.ReadLine();
        while (input != "exit")
        {
            Thread thread = new Thread(new ParameterizedThreadStart(ClientSendMsgToServer));
            thread.Start(input);
            input = Console.ReadLine();
        }

        webSocket4Net.Dispose();
    }
}
```

#### 支持WSS

首先在基于Fleck实现的服务端中加载域名证书和密码，

```c#
var server = new WebSocketServer("wss://127.0.0.1:8182");
server.Certificate = new X509Certificate2("certificate.pfx", "", X509KeyStorageFlags.Exportable | X509KeyStorageFlags.MachineKeySet | X509KeyStorageFlags.PersistKeySet); // 自签名证书密码默认为""
```

然后在基于WebSocket4Net实现的客户端中修改握手后的升级请求(ws前缀改为wss)。

```c#
webSocket4Net = new WebSocket("wss://127.0.0.1:8182");
```

## 参考文档

- [RFC 6455官方文档](https://www.rfc-editor.org/rfc/rfc6455)

- [WebSocket协议(RFC 6455中文版)](https://websocket.xiniushu.com/)

- [Fleck开源项目地址](https://github.com/statianzo/Fleck)

- [WebSocket4Net开源项目地址](https://github.com/kerryjiang/WebSocket4Net)