---
title: 基于Fleck和WebSocket4Net的WebSocket简单实现
date: 2024-08-18 20:53:59
categories:
- Network-Protocol
tags:
- WebSocket
- .NET
---

WebSocket的基本概念、通信方式，以及OSI七层网络模型，并基于Fleck和WebSocket4Net实现一个简单的即时通讯聊天功能。

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

## 实现

基于Fleck+WebSocket4Net实现简单通讯聊天。

### 服务端

Fleck是C#中的一个WebSocket服务器实现。下面借助Fleck来模拟一个WebSocket服务端。

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

### 客户端

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

- [Fleck开源项目地址](https://github.com/statianzo/Fleck)

- [WebSocket4Net开源项目地址](https://github.com/kerryjiang/WebSocket4Net)