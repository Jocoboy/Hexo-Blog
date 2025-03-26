---
title: .NET实现分片上传与断点续传
date: 2025-03-25 17:28:11
categories:
- Framework
tags:
- .NET
- ASP.NET Core
- WinForm
---

基于.NET实现大文件上传中的分片上传与断点续传功能，并在WinForm中实现上传进度反馈等辅助功能。

<!--more-->

## 前言

在文件上传场景中，分片上传和断点续传是处理大文件上传的重要技术。

除了基本的断点续传功能，大文件上传还应提供以下辅助功能：

- 分片大小调整：根据网络状况动态调整分片大小
- 并行上传：可以并行上传多个分片以提高速度
- 完整性校验：合并文件后校验MD5等哈希值确保文件完整
- 清理机制：定期清理未完成上传的临时分片
- 进度反馈：提供上传进度信息给用户

下面我们来简单实现上述功能。

## 分片上传

分片上传流程如下：

- 客户端首次上传前生成文件唯一ID(通常使用文件内容哈希，此处为了演示重新上传是从0开始的，使用的是Guid)
- 客户端将文件分割为固定大小的块，每块单独上传到服务器
- 服务器接收并临时保存每个分片
- 当所有分片上传完成后，服务器合并分片为完整文件

### 服务端实现

UploadChunk接口实现了分片的临时存储(存储的临时文件名为fileMD5_chunkNumber)，以及分片的最终合并。

```c#
[HttpPost]
[AllowAnonymous]
public async Task<IActionResult> UploadChunkAsync(CancellationToken cancellationToken)
{
    var form = await Request.ReadFormAsync(cancellationToken);
    var fileId = form["fileId"].ToString();
    var chunkNumber = int.Parse(form["chunkNumber"].ToString());
    var totalChunks = int.Parse(form["totalChunks"].ToString());
    var fileName = form["fileName"].ToString();
    var chunk = form.Files["chunk"];

    // 确保上传目录存在
    Directory.CreateDirectory(_uploadPath);

    // 临时保存分片
    var chunkPath = Path.Combine(_uploadPath, $"{fileId}_{chunkNumber}");
    using (var stream = new FileStream(chunkPath, FileMode.Create))
    {
        await chunk.CopyToAsync(stream, cancellationToken);
    }

    // 如果是最后一个分片，合并文件
    if (chunkNumber == totalChunks - 1)
    {
        await MergeChunksAsync(fileId, totalChunks, fileName, cancellationToken);
        return Ok(new { Message = "Upload complete", FileName = fileName });
    }

    return Ok(new { Message = "Chunk uploaded", ChunkNumber = chunkNumber });
}

private async Task MergeChunksAsync(string fileId, int totalChunks, string fileName, CancellationToken cancellationToken)
{
    var finalPath = Path.Combine(_uploadPath, fileName);

    using (var finalStream = new FileStream(finalPath, FileMode.Create))
    {
        for (int i = 0; i < totalChunks; i++)
        {
            var chunkPath = Path.Combine(_uploadPath, $"{fileId}_{i}");
            using (var chunkStream = System.IO.File.OpenRead(chunkPath))
            {
                await chunkStream.CopyToAsync(finalStream, cancellationToken);
            }
            System.IO.File.Delete(chunkPath); // 合并后删除分片
        }
    }
}
```

### 客户端实现

首先创建一个ChunkedUploader类，用来处理本地的文件流，并调用服务端的分片上传接口。

```c#
public class ChunkedUploader
{
    private readonly HttpClient _httpClient;
    private readonly string _serviceUri;
    private const string UploadChunkApiUri = "/api/Upload/UploadChunk";
    private const int ChunkSize = 1024 * 1024; // 1MB 每块

    public ChunkedUploader(string serviceUri)
    {
        _serviceUri = serviceUri;
        _httpClient = new HttpClient();
    }

    public async Task UploadInChunksAsync(string filePath, CancellationToken cancellationToken)
    {
        var fileInfo = new FileInfo(filePath);
        var totalChunks = (int)Math.Ceiling((double)fileInfo.Length / ChunkSize);
        var fileId = Guid.NewGuid().ToString(); // 唯一文件标识

        using var fileStream = File.OpenRead(filePath);

        for (int chunkNumber = 0; chunkNumber < totalChunks; chunkNumber++)
        {
            var chunkData = new byte[ChunkSize];
            var bytesRead = await fileStream.ReadAsync(chunkData, 0, ChunkSize, cancellationToken);

            if (bytesRead == 0) break;

            var actualChunkData = new byte[bytesRead]; // 只取实际读取的字节
            Array.Copy(chunkData, actualChunkData, bytesRead);

            await UploadChunkAsync(fileId, chunkNumber, totalChunks, fileInfo.Name, actualChunkData, cancellationToken);
        }
    }

    private async Task UploadChunkAsync(string fileId, int chunkNumber, int totalChunks, string fileName, byte[] chunkData, CancellationToken cancellationToken)
    {
        using var content = new MultipartFormDataContent
        {
            { new StringContent(fileId), "fileId" },
            { new StringContent(chunkNumber.ToString()), "chunkNumber" },
            { new StringContent(totalChunks.ToString()), "totalChunks" },
            { new StringContent(fileName), "fileName" },
            { new ByteArrayContent(chunkData), "chunk", "chunk.dat" }
        };

        var response = await _httpClient.PostAsync($"{_serviceUri}{UploadChunkApiUri}", content, cancellationToken);
        response.EnsureSuccessStatusCode();
    }
}
```

然后在客户端的UI线程中使用异步方式调用ChunkedUploader类中的分片上传方法。

```c#
var serviceUri = "http://localhost:5001";
var filePath = $"{disk}:\\{srcDir}\\{srcFile}";
var uploader = new ChunkedUploader(serviceUri);
await uploader.UploadInChunksAsync(filePath, CancellationToken.None);
```

## 断点续传

断点续传流程如下：

- 客户端首次上传前生成文件唯一ID
- 上传前根据文件文件唯一ID查询服务器已接收的分片，继续上传时只上传缺失的分片
- 当所有分片上传完成后合并分片

### 服务端实现

GetUploadStatus接口根据文件标识fileMD5查询已上传的分片编号，UploadChunk接口的功能同上。

```c#
[ApiController]
[Route("[controller]/[action]")]
[Authorize]
public class UploadController: ControllerBase
{
    private readonly IWebHostEnvironment _env;
    private readonly string _uploadPath;

    public UploadController(IWebHostEnvironment env)
    {
        _env = env;
        _uploadPath = Path.Combine(_env.WebRootPath, "uploads");
    }

    [HttpGet]
    [AllowAnonymous]
    public IActionResult GetUploadStatus(string fileId)
    {
        var uploadedChunks = new List<int>();

        if (Directory.Exists(_uploadPath))
        {
            var chunkFiles = Directory.GetFiles(_uploadPath, $"{fileId}_*");
            foreach (var chunkFile in chunkFiles)
            {
                if (int.TryParse(Path.GetFileName(chunkFile).Split('_').Last(), out var chunkNumber))
                {
                    uploadedChunks.Add(chunkNumber);
                }
            }
        }

        return Ok(uploadedChunks);
    }

    [HttpPost]
    [AllowAnonymous]
    public async Task<IActionResult> UploadChunkAsync(CancellationToken cancellationToken)
    {
        // 同分片上传
    }

    private async Task MergeChunksAsync(string fileId, int totalChunks, string fileName, CancellationToken cancellationToken)
    {
        // 同分片上传
    }
}
```

### 客户端实现

首先创建一个ResumableUploader类，调用服务端的文件分片上传状态接口，对于已上传的分片跳过处理，然后调用服务端的分片上传接口继续上传，以实现断点续传功能。

```c#
public class ResumableUploader
{
    private readonly HttpClient _httpClient;
    private readonly string _serviceUri;
    private const string UploadChunkApiUri = "/api/Upload/UploadChunk";
    private const string GetUploadStatusApiUri = "/api/Upload/GetUploadStatus";
    private const int ChunkSize = 1024 * 1024; // 1MB 每块

    public ResumableUploader(string serviceUri)
    {
        _serviceUri = serviceUri;
        _httpClient = new HttpClient();
    }

    public async Task UploadWithResumeAsync(string filePath, CancellationToken cancellationToken)
    {
        var fileInfo = new FileInfo(filePath);
        var totalChunks = (int)Math.Ceiling((double)fileInfo.Length / ChunkSize);
        var fileId = GetFileMD5(filePath); // 基于文件内容生成唯一ID

        // 获取已上传的分片信息
        var uploadedChunks = await GetUploadedChunksAsync(fileId, cancellationToken);

        using var fileStream = File.OpenRead(filePath);

        for (int chunkNumber = 0; chunkNumber < totalChunks; chunkNumber++)
        {
            if (uploadedChunks.Contains(chunkNumber))  continue; // 跳过已上传的分片

            fileStream.Seek(chunkNumber * ChunkSize, SeekOrigin.Begin);

            var chunkData = new byte[ChunkSize];
            var bytesRead = await fileStream.ReadAsync(chunkData, 0, ChunkSize, cancellationToken);

            if (bytesRead == 0) break;

            var actualChunkData = new byte[bytesRead];
            Array.Copy(chunkData, actualChunkData, bytesRead);

            await UploadChunkAsync(fileId, chunkNumber, totalChunks, fileInfo.Name, actualChunkData, cancellationToken);
        }
    }

    private string GetFileMD5(string filePath)
    {
        using var md5 = MD5.Create();
        using var stream = File.OpenRead(filePath);
        var hash = md5.ComputeHash(stream);
        return BitConverter.ToString(hash).Replace("-", "").ToLower();
    }

    private async Task<List<int>> GetUploadedChunksAsync(string fileId, CancellationToken cancellationToken)
    {
        try
        {
            var response = await _httpClient.GetAsync($"{_serviceUri}{GetUploadStatusApiUri}?fileId={fileId}");
            response.EnsureSuccessStatusCode();

            var json = await response.Content.ReadAsStringAsync(cancellationToken);
            return JsonSerializer.Deserialize<List<int>>(json);
        }
        catch
        {
            return new List<int>();
        }
    }

    private async Task UploadChunkAsync(string fileId, int chunkNumber, int totalChunks, string fileName, byte[] chunkData, CancellationToken cancellationToken)
    {
       // 同分片上传
    }
}
```

然后在客户端的UI线程中使用异步方式调用ResumableUploader类中的断点续传方法。

```c#
var serviceUri = "http://localhost:5001";
var filePath = $"{disk}:\\{srcDir}\\{srcFile}";
var uploader = new ResumableUploader(serviceUri);
await uploader.UploadWithResumeAsync(filePath, CancellationToken.None);
```

## 上传进度反馈与上传取消

通常情况下，大文件上传时需要给用户提供上传进度信息以及取消上传的功能，下面使用WinForm简单实现上述功能。

### 服务端实现

服务端代码延用断点续传，不再展示。

### 客户端实现

客户端的ResumableUploader中的主要变化是添加了三个委托事件，分别用来更新上传百分比、状态消息、上传与取消按钮禁用状态。

```c#
public class ResumableUploader
{
    ...
    // 进度和状态事件
    public event Action<int> ProgressChanged; // 上传百分比
    public event Action<string> StatusChanged; // 状态消息
    public event Action<bool> UploadCompleted; // 完成状态

    public async Task UploadWithResumeAsync(string filePath, CancellationToken cancellationToken)
    {
        try
        {
            StatusChanged?.Invoke("正在准备上传...");

            var fileInfo = new FileInfo(filePath);
            var totalChunks = (int)Math.Ceiling((double)fileInfo.Length / ChunkSize);
            var fileId = GetFileMD5(filePath); // 基于文件内容生成唯一ID

            StatusChanged?.Invoke("正在检查已上传分片...");

            // 获取已上传的分片信息
            var uploadedChunks = await GetUploadedChunksAsync(fileId, cancellationToken);

            using var fileStream = File.OpenRead(filePath);

            for (int chunkNumber = 0; chunkNumber < totalChunks; chunkNumber++)
            {
                cancellationToken.ThrowIfCancellationRequested();

                if (uploadedChunks.Contains(chunkNumber))
                {
                    ProgressChanged?.Invoke((int)((double)chunkNumber / totalChunks * 100));
                    continue;
                }

                fileStream.Seek(chunkNumber * ChunkSize, SeekOrigin.Begin);

                var chunkData = new byte[ChunkSize];
                var bytesRead = await fileStream.ReadAsync(chunkData, 0, ChunkSize, cancellationToken);

                if (bytesRead == 0) break;

                var actualChunkData = new byte[bytesRead];
                Array.Copy(chunkData, actualChunkData, bytesRead);

                StatusChanged?.Invoke($"正在上传分片 {chunkNumber + 1}/{totalChunks}");

                await UploadChunkAsync(fileId, chunkNumber, totalChunks, fileInfo.Name, actualChunkData, cancellationToken);

                var progress = (int)((double)(chunkNumber + 1) / totalChunks * 100);
                ProgressChanged?.Invoke(progress);
            }

            StatusChanged?.Invoke("上传完成！");
            UploadCompleted?.Invoke(true);
        }
        catch (OperationCanceledException)
        {
            StatusChanged?.Invoke("上传已取消");
            UploadCompleted?.Invoke(false);
        }
        catch (Exception ex)
        {
            StatusChanged?.Invoke($"上传失败: {ex.Message}");
            UploadCompleted?.Invoke(false);
        }
    }
    ...
}
```

主窗体MainForm中分别绑定三个委托事件到相应的UI线程上，使用Invoke跨线程更新UI，实现上传过程中实时更新上传进度及状态信息。

Control.InvokeRequired属性指示调用方在对控件进行方法调用时是否必须调用Invoke方法，因为调用方位于创建控件所在的线程以外的线程中。Windows窗体中的控件绑定到特定线程，并且不是线程安全的。因此，如果要从其他线程调用控件的方法，则必须使用控件的调用方法之一来封送对正确线程的调用。

```c#
public partial class MainForm : Form
{
    private readonly ResumableUploader _uploader;
    private CancellationTokenSource _cts;
    private readonly string _serviceUri;

    public MainForm()
    {
        InitializeComponent();

        _serviceUri = "http://localhost:5001";
        _uploader = new ResumableUploader(_serviceUri);

        // 绑定事件
        _uploader.ProgressChanged += UpdateProgress;
        _uploader.StatusChanged += UpdateStatus;
        _uploader.UploadCompleted += UploadCompleted;
    }

    private void UpdateProgress(int percent)
    {
        if (progressBar.InvokeRequired)
        {
            progressBar.Invoke(new Action<int>(UpdateProgress), percent);
            return;
        }
        progressBar.Value = percent;
    }

    private void UpdateStatus(string message)
    {
        if (lblStatus.InvokeRequired)
        {
            lblStatus.Invoke(new Action<string>(UpdateStatus), message);
            return;
        }
        lblStatus.Text = message;
    }

    private void UploadCompleted(bool success)
    {
        if (btnUpload.InvokeRequired)
        {
            btnUpload.Invoke(new Action<bool>(UploadCompleted), success);
            return;
        }
        btnUpload.Enabled = true;
        btnCancel.Enabled = false;
    }

    private async void btnUpload_Click(object sender, EventArgs e)
    {
        if (string.IsNullOrEmpty(textBox.Text) || !File.Exists(textBox.Text))
        {
            MessageBox.Show("请选择有效的文件");
            return;
        }

        btnUpload.Enabled = false;
        btnCancel.Enabled = true;
        progressBar.Value = 0;

        _cts = new CancellationTokenSource();
        await _uploader.UploadWithResumeAsync(textBox.Text,_cts.Token);
    }

    private void btnCancel_Click(object sender, EventArgs e)
    {
        _cts?.Cancel();
    }

    private void btnBrowse_Click(object sender, EventArgs e)
    {
        using var openFileDialog = new OpenFileDialog();
        if (openFileDialog.ShowDialog() == DialogResult.OK)
        {
            textBox.Text = openFileDialog.FileName;
        }
    }

    protected override void OnFormClosing(FormClosingEventArgs e)
    {
        _cts?.Cancel();
        base.OnFormClosing(e);
    }
}
```

最终实现的效果如下：

{% asset_img resumable_upload_in_winform.png  在WinForm中实现断点续传 %}

## 参考文档

- [阿里云OSS断点续传上传操作指南](https://help.aliyun.com/zh/oss/user-guide/resumable-upload)

- [Control.InvokeRequired属性参考文档](https://learn.microsoft.com/zh-cn/dotnet/api/system.windows.forms.control.invokerequired)