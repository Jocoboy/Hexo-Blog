---
title: 安全扫描技术及其应用场景
date: 2024-12-13 14:36:57
categories:
- Cyber-Security
tags:
- Security-Scan
---

三种常用的安全扫描技术原理、相关工具及应用场景。

<!--more-->

## 前言

安全扫描技术是网络安全领域中用于检测系统、网络和应用程序中潜在安全漏洞的重要手段，分为端口扫描技术、漏洞扫描技术、Web应用扫描技术等，其中漏洞扫描技术分为基于网络的漏洞扫描和基于主机的漏洞扫描。其工作流程分为信息收集、扫描执行、结果分析、漏洞报告四个部分。

## 端口扫描技术

### 原理

通过向目标主机的一系列端口发送数据包，根据目标主机的响应来判断端口是开放还是关闭的。例如，TCP 连接扫描（全连接扫描）会尝试与目标端口建立完整的 TCP 三次握手。如果能够成功完成三次握手，说明端口是开放的；如果在握手过程中收到复位（RST）信号，则表示端口是关闭的。

### 相关工具

Nmap是一款非常著名的端口扫描工具。它可以快速扫描目标主机上大量端口，并且能够识别端口对应的服务。例如，使用命令`nmap -p 1 - 1000 <target_ip_address>`可以扫描目标主机上1到1000号端口的开放情况。

## 基于网络的漏洞扫描技术

### 原理

通过网络向目标系统发送特定的探测数据包，模拟黑客攻击行为，检查目标系统是否存在已知的安全漏洞。这些扫描器通常拥有一个庞大的漏洞数据库，其中包含各种软件、操作系统和网络设备的已知漏洞信息。

### 相关工具

OpenVAS是一个开源的漏洞扫描器，它能够扫描网络中的各种设备，包括服务器、防火墙等，检测诸如操作系统漏洞、Web 应用漏洞等多种类型的漏洞。

## 基于主机的漏洞扫描技术

### 原理

在目标主机本地安装扫描软件，对主机系统本身进行扫描，检查系统配置、文件权限、安装的软件等方面是否存在安全隐患。它主要关注主机内部的安全状态，例如检查是否存在弱口令、未及时更新的软件等。

### 相关工具

MBSA（Microsoft Baseline Security Analyzer）是一款针对Windows系统的漏洞扫描工具，可以检测Windows操作系统以及微软的其他软件（如 IIS、SQL Server 等）是否存在安全漏洞，包括安全更新缺失、账户策略不安全等问题。

### 应用场景

#### IIS隐藏Server信息

IIS服务器端返回信息中包含有软件版本等详细信息，攻击者利用这些信息可以实现更有目的性的攻击。因此隐藏server版本信息，在一定程度上能够提高服务器的安全性。

可通过下载[Rewrite插件](https://www.iis.net/downloads/microsoft/url-rewrite)，然后修改`C:\Windows\System32\inetsrv\config\applicationHost.config`配置文件来实现，配置必须写在system.webServer节点内。

```xml
<rewrite>
    <allowedServerVariables>
        <add name="REMOTE_ADDR" />
    </allowedServerVariables>            
    <outboundRules>
        <rule name="REMOVE_RESPONSE_SERVER">
            <match serverVariable="RESPONSE_SERVER" pattern=".*" />
            <action type="Rewrite" />
        </rule>
    </outboundRules>
</rewrite>
```

## Web应用扫描技术

### 原理

主要用于检测Web应用程序中的安全漏洞，如SQL注入、跨站脚本攻击（XSS）、跨站请求伪造（CSRF）等。扫描器会对Web应用的页面、表单、链接等进行分析，发送各种特制的请求来测试应用程序的安全性。

### 相关工具

Acunetix是一款专业的Web应用安全扫描工具，它能够自动检测多种Web漏洞。例如，在扫描一个包含用户登录功能的Web页面时，它可以检测输入框是否容易受到SQL注入攻击，以及登录后的页面是否存在 XSS 漏洞等。

### 应用场景

#### URL参数安全防护

对所有用户输入进行严格的验证和过滤是防范XSS攻击的关键，当应用程序将用户输入的数据嵌入到HTML页面中时，必须对数据进行适当的编码，确保只接受预期的字符集和格式。

例如，可将请求参数放到Headers中，并按指定的字符集(如ISO8859-1)进行编码。

```c#
public IActionResult Login([FromBody] LoginInput input)
{
    StringValues para = new StringValues();
    if (Request.Headers.TryGetValue("username", out var headerValues) && headerValues.Count > 0)
    {
        para = headerValues[0];
    }
    else
    {
        return NotFound("参数username不存在");
    }

    string strs = null;
    try
    {
        var bytes = Encoding.GetEncoding("ISO8859-1").GetBytes(para);
        strs = Encoding.GetEncoding("ISO8859-1").GetString(bytes);
    }
    catch (ArgumentException e)
    {
        return BadRequest($"编码转换异常: {e.Message}");
    }
    ...
}
```

#### 设置Cookie的HTTPOnly属性

Session和Cookie是两种常用的客户端存储技术，它们在Web开发中用于存储和管理用户状态，以下是二者的区别：

| 区别 | Cookie | Session | 
| ----------- | ----------- | ----------- |
| 存储位置 | 存储在客户端浏览器中，以键值对的形式存在 | 存储在服务器端，每个用户会话对应一个唯一的Session对象 |
| 存储方式 | 通过HTTP响应头发送给客户端，并保存在浏览器中 | 通过Session ID（通常存储在Cookie中）在服务器端进行标识和存储 |
| 存储容量 | 4KB | 无限制 |
| 安全性   | 容易受到XSS和CSRF等攻击，可以通过设置HttpOnly、Secure和SameSite属性来提高安全性 | 存储在服务器端，相对来说更安全，但也需要注意Session劫持和修复等安全问题 |
| 生命周期 | 可以设置过期时间，到期后浏览器会自动删除 | 依赖于服务器设置的超时时间，用户不活跃超过这个时间后，Session会失效 |
| 跨域问题 | 可以通过设置Domain属性实现跨域访问 | 通常与特定的域绑定，不支持跨域访问 |
| 访问方式 | 可以通过JavaScript的document.cookie属性直接访问和修改 | 需要通过服务器端代码来访问和修改 |
| 传输 | 每次HTTP请求都会自动发送到服务器，增加了请求的负载 | 不需要在每次请求中发送，只在建立Session和通过Session ID获取Session数据时与服务器交互 |
| 用途 | 常用于保存用户的偏好设置、会话标识等 | 用于跟踪用户的状态，如登录状态、购物车内容等 |


WWW服务依赖于Http协议实现，Http是无状态的协议，所以为了在各个会话之间传递信息，就需要使用Cookie来标记访问者的状态，以便服务器端识别用户信息。Cookie由变量名与值组成，其属性里有标准的cookie变量，也有用户自定义的属性。Cookie保存在浏览器的document对象中，对于存在XSS漏洞的网站，入侵者可以插入简单的XSS语句执行任意的JS脚本，以XSS攻击的手段获取网站其余用户的Cookie。

为了防止用户Cookie被JS脚本恶意读取，可以在服务端设置HTTP - only属性，NET Core下可通过中间件添加到应用程序管道来实现。


```c#
// HttpOnlyCookieMiddleware.cs
public class HttpOnlyCookieMiddleware
 {
     private readonly RequestDelegate _next;

     public HttpOnlyCookieMiddleware(RequestDelegate next)
     {
         _next = next;
     }

     public async Task Invoke(HttpContext context)
     {
         // 在响应发送之前设置Cookie选项
         context.Response.OnStarting(() =>
         {
             var cookies = context.Response.Cookies;
             var options = new CookieOptions
             {
                 HttpOnly = true
             };
             var authToken = context.Request.Cookies["AuthToken"];
             if (!string.IsNullOrEmpty(authToken))
             {
                 cookies.Append("AuthToken", authToken, options);
             }

             return Task.CompletedTask;
         });
         await _next(context);
     }

 }

 public static class HttpOnlyCookieMiddlewareExtensions
 {
     public static IApplicationBuilder UseHttpOnlyCookieMiddleware(this IApplicationBuilder builder)
     {
         return builder.UseMiddleware<HttpOnlyCookieMiddleware>();
     }
 }

// Startup.cs
public void Configure(IApplicationBuilder app)
{
    ...
    app.UseHttpOnlyCookieMiddleware();
    ...
}
```

## 参考文档

- [字符串编码和解码在线工具](https://www.toolhelper.cn/EncodeDecode/EncodeDecode)

- [ApplicationHost.config简介](https://learn.microsoft.com/zh-cn/iis/get-started/planning-your-iis-architecture/introduction-to-applicationhostconfig)

- [.NET Core中间件使用参考](https://learn.microsoft.com/zh-cn/aspnet/core/migration/http-modules)