---
title: HttpClient使用详解
date: 2025-03-04 17:03:08
categories:
- Remote-Procedure-Call
tags:
- HttpClient
- .NET
- ASP.NET Core
---

使用HttpClient发送HTTP请求的方式介绍。

<!--more-->

## 前言

在c#中常见发送http请求的方式有以下三种：

- HttpWebRequest: .NET平台原生提供，这是.NET创建者最初开发用于使用HTTP请求的标准类。使用HttpWebRequest可以让开发者控制请求/响应流程的各个方面，如timeouts,cookies,headers,protocols
- WebClient: .NET平台原生提供，WebClient是一种更高级别的抽象，是HttpWebRequest为了简化最常见任务而创建的，但也因此缺少了HttpWebRequest的灵活性
- HttpClient: .NET平台原生提供，目前主流使用的Http请求工具类，直接基于Socket开发，提供了异步友好的代码编写方式

## HttpClient使用配置

当我们发送http请求时，可以使用SocketsHttpHandler处理一些事情，比如是否自动处理cookie，是否自动重定向以及最多重定向几次等。以下展示了常见的配置项：

```c#
// 在.net core 2.1之后，默认所有的http请求都会交给SocketsHttpHandler处理
var socketsHttpHandler = new SocketsHttpHandler()
{
    UseCookies = false, //是否自动处理cookie

    AllowAutoRedirect = true,//是否自动重定向, 默认true
    MaxAutomaticRedirections = 50,//自动重定向的最大次数, 默认50

    MaxConnectionsPerServer = 100, //每个请求连接的最大数量, 默认是int.MaxValue
    PooledConnectionIdleTimeout = TimeSpan.FromMinutes(2), //连接池中TCP连接最多可以闲置多久, 默认2分钟
    PooledConnectionLifetime = Timeout.InfiniteTimeSpan, //连接最长的存活时间, 默认是不限制的

    AutomaticDecompression = DecompressionMethods.GZip, //是否压缩，默认是None，即不压缩

   ConnectTimeout = Timeout.InfiniteTimeSpan,  //建立TCP连接时的超时时间, 默认不限制
   Expect100ContinueTimeout = TimeSpan.FromSeconds(1), //等待服务返回statusCode=100的超时时间, 默认1秒

   MaxResponseHeadersLength = 64, //响应头大小限制，单位kb
};

using var httpClient = new HttpClient(socketsHttpHandler);

```

此外，可以给HttpClient设置基地址，当HttpClient发送的请求不包含前缀时，将自动拼接上，否则不予拼接。还可以设置默认的http版本、请求头。

```c#
httpClient.BaseAddress = new Uri("http://127.0.0.1:3060/");
httpClient.DefaultRequestVersion = HttpVersion.Version20;
httpClient.DefaultRequestHeaders.Add("dnname", "Tony");
```

## HttpClient请求方法

### GET

通过GET请求数据的示例如下：

```c#
using var httpClient = new HttpClient(socketsHttpHandler);

var url = "http://127.0.0.1:3060/index.html";
var response = await httpClient.GetAsync(url);
var str = await response.Content.ReadAsStringAsync();
```

如果要通过GET方法下载文件，可以通过如下方式：

```c#
using var httpClient = new HttpClient();
var response = await httpClient.GetAsync("http://localhost:3060/my.txt");
var fileStream = new FileStream("C:\\my.txt", FileMode.OpenOrCreate, FileAccess.Write);
await response.Content.CopyToAsync(fileStream);
fileStream.Close();
```

如果要下载超过2GB的文件，则需要通过如下方式：

```c#
using var httpClient = new HttpClient();
var url = "http://localhost:3060/my.mp4";
var response = await httpClient.GetAsync(url, HttpCompletionOption.ResponseHeadersRead); // 拿到响应头就返回
var fileStream = new FileStream("C:\\my.mp4", FileMode.OpenOrCreate, FileAccess.Write);
await response.Content.CopyToAsync(fileStream);
fileStream.Close();
```

### POST

根据Content-Type类型，常见的请求类型有application/json、multipart/form-data、application/x-www-form-urlencoded等。

#### application/json

使用POST请求application/json类型的示例如下：

```c#
using var httpClient = new HttpClient();
var url = "http://localhost:5001/ImportFromUrl";
var stringContent = new StringContent(JsonConvert.SerializeObject(new { DownloadFileName = "test.txt", Url = "http://localhost:3060/test.txt" }), Encoding.UTF8, "application/json");
var response = await httpClient.PostAsync(url, stringContent);
var responseContent = await response.Content.ReadAsStringAsync();
var res = JsonConvert.DeserializeObject<DocInfo>(responseContent);
```

#### multipart/form-data

使用POST请求multipart/form-data类型的示例如下：

```c#
using var httpClient = new HttpClient();
var url = "http://localhost:5001/ImportFromUrl";
using var formData = new MultipartFormDataContent
{
    { new StringContent("test.txt"), "downloadFileName" },
    { new StringContent("http://localhost:3060/test.txt"), "url" }
};
var response = await httpClient.PostAsync(url, formData);
var responseContent = await response.Content.ReadAsStringAsync();
var res = JsonConvert.DeserializeObject<DocInfo>(responseContent);
```

## 参考文档

- [使用HttpClient类发出HTTP请求](https://learn.microsoft.com/zh-cn/dotnet/fundamentals/networking/http/httpclient)