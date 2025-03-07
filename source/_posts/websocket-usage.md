---
title: WebSocket在C#与.NET中的简单实现
date: 2024-08-18 20:53:59
categories:
- Network-Protocol
tags:
- WebSocket
- .NET
---

WebSocket的基本概念、应用场景，以及服务端和客户端在C#与.NET中的简单实现。

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

WebSockek有串口、网口两种通信方式。

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
public class WebSocketServer
{
    private readonly string _serverUri;
    private readonly HttpListener _httpListener;
    private WebSocket _webSocket;

    public WebSocketServer(string serverUri)
    {
        _serverUri = serverUri;
        _httpListener = new HttpListener();
    }

    public async Task Start()
    {
        _httpListener.Prefixes.Add(_serverUri);
        _httpListener.Start();
        Console.WriteLine($"WebSocket服务已启动，服务地址：{_serverUri}，等待客户端连接...");

        while (true)
        {
            HttpListenerContext httpContext = await _httpListener.GetContextAsync();

            if (httpContext.Request.IsWebSocketRequest)
            {
                await HandleWebSocketConnection(httpContext);
            }
            else
            {
                httpContext.Response.StatusCode = 400;
                httpContext.Response.Close();
            }
        }
    }

    private async Task HandleWebSocketConnection(HttpListenerContext httpContext)
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
                Console.WriteLine(" WebSocket消息接收错误 :" + e.Message);
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

    ~WebSocketServer()
    {
        _webSocket.Dispose();
    }
}
```

然后启动WebSocket服务。

```c#
static async Task Main(string[] args)
{
    string serverUri = "http://localhost:8181/";

    var webSocketServer = new WebSocketServer(serverUri);

    await webSocketServer.Start();
}
```

#### 客户端

首先创建一个WebSocket客户端类，

```c#
 public class WebSocketClient
 {
     private readonly string _uri;
     private readonly ClientWebSocket _clientWebSocket;

     public WebSocketClient(string uri)
     {
         _uri = uri;
         _clientWebSocket = new ClientWebSocket();
     }

     public async Task Start()
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

     ~WebSocketClient()
     {
         _clientWebSocket.Dispose();
     }
 }
```

然后启动WebSocket客户端。

```c#
static async Task Main(string[] args)
{
    string serverUri = "http://localhost:8181/";

    var webSocketClient = new WebSocketClient(serverUri.Replace("http", "ws")); // http升级请求

    await webSocketClient.Start(); 
}
```

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

## 参考文档

- [RFC 6455官方文档](https://www.rfc-editor.org/rfc/rfc6455)

- [WebSocket协议(RFC 6455中文版)](https://websocket.xiniushu.com/)

- [Fleck开源项目地址](https://github.com/statianzo/Fleck)

- [WebSocket4Net开源项目地址](https://github.com/kerryjiang/WebSocket4Net)