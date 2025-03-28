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

基于.NET实现大文件上传中的分片上传与断点续传功能，并在WinForm中实现上传进度反馈、并行上传、分片大小动态调整等辅助功能。

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

UploadChunk接口实现了分片的临时存储(存储的临时文件名为fileId_chunkNumber)，以及分片的最终合并。

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

首先创建一个ChunkedUploadService类，用来处理本地的文件流，并调用服务端的分片上传接口。

```c#
public class ChunkedUploadService
{
    private readonly HttpClient _httpClient;
    private readonly string _serviceUri;
    private const string UploadChunkApiUri = "/api/Upload/UploadChunk";
    private const int ChunkSize = 1024 * 1024; // 1MB 每块

    public ChunkedUploadService(string serviceUri)
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

然后在客户端的UI线程中使用异步方式调用ChunkedUploadService类中的分片上传方法。

```c#
var serviceUri = "http://localhost:5001";
var filePath = $"{disk}:\\{srcDir}\\{srcFile}";
var uploader = new ChunkedUploadService(serviceUri);
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

首先创建一个ResumableUploadService类，调用服务端的文件分片上传状态接口，对于已上传的分片跳过处理，然后调用服务端的分片上传接口继续上传，以实现断点续传功能。

```c#
public class ResumableUploadService
{
    private readonly HttpClient _httpClient;
    private readonly string _serviceUri;
    private const string UploadChunkApiUri = "/api/Upload/UploadChunk";
    private const string GetUploadStatusApiUri = "/api/Upload/GetUploadStatus";
    private const int ChunkSize = 1024 * 1024; // 1MB 每块

    public ResumableUploadService(string serviceUri)
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

然后在客户端的UI线程中使用异步方式调用ResumableUploadService类中的断点续传方法。

```c#
var serviceUri = "http://localhost:5001";
var filePath = $"{disk}:\\{srcDir}\\{srcFile}";
var uploader = new ResumableUploadService(serviceUri);
await uploader.UploadWithResumeAsync(filePath, CancellationToken.None);
```

## 上传进度反馈与上传取消

通常情况下，大文件上传时需要给用户提供上传进度信息以及取消上传的功能，下面使用WinForm简单实现上述功能。

### 服务端实现

服务端代码延用[断点续传](#断点续传)，不再展示。

### 客户端实现

客户端的ResumableUploadService中的主要变化是添加了三个委托事件，分别用来更新上传百分比、状态消息、上传与取消按钮禁用状态。

```c#
public class ResumableUploadService
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
    private readonly ResumableUploadService _uploader;
    private CancellationTokenSource _cts;
    private readonly string _serviceUri;

    public MainForm()
    {
        InitializeComponent();

        _serviceUri = "http://localhost:5001";
        _uploader = new ResumableUploadService(_serviceUri);

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

## 并行上传

并行上传是提升大文件传输效率的有效手段，并行上传可将总时间缩短为单线程上传时间/N(N为并行度)。并行上传适合在不稳定的网络环境(如高延迟)中使用，可充分利用间歇性网络带宽。

并行上传时多个分片同时上报上传进度，进度条呈现更平滑，更适合需要实时进度反馈的场景。

### 服务端实现

服务端代码延用[断点续传](#断点续传)，不再展示。

### 客户端实现

客户端基于ResumableUploadService，创建了一个并行上传服务类ParallelUploadService，使用`Parallel.ForEachAsync`实现并行上传，并设置了最大并行度(默认为4)。

```c#
public class ParallelUploadService
{
    ...
    private readonly int _maxParallelism;

    public ParallelUploadService(string serviceUri, int maxParallelism = 4)
    {
        ...
        _maxParallelism = maxParallelism;
    }

    public async Task ParallelUploadFileAsync(string filePath, CancellationToken cancellationToken)
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

            var chunksToUpload = Enumerable.Range(0, totalChunks).Where(chunk => !uploadedChunks.Contains(chunk)).ToList();

            var progressLock = new object();
            var uploadedCount = 0;
            var totalToUpload = chunksToUpload.Count;

            StatusChanged?.Invoke($"准备上传 {totalToUpload} 个分片...");

            var parallelOptions = new ParallelOptions
            {
                MaxDegreeOfParallelism = _maxParallelism,
                CancellationToken = cancellationToken
            };

            using var fileStream = File.OpenRead(filePath);

            await Parallel.ForEachAsync(chunksToUpload, parallelOptions, async (chunkNumber, cancellationToken) =>
            {
                cancellationToken.ThrowIfCancellationRequested();

                fileStream.Seek(chunkNumber * ChunkSize, SeekOrigin.Begin);

                var chunkData = new byte[ChunkSize];
                var bytesRead = await fileStream.ReadAsync(chunkData, 0, ChunkSize, cancellationToken);

                var actualChunkData = new byte[bytesRead];
                Array.Copy(chunkData, actualChunkData, bytesRead);

                StatusChanged?.Invoke($"正在上传分片 {chunkNumber + 1}/{totalChunks}");

                await UploadChunkAsync(fileId, chunkNumber, totalChunks, fileInfo.Name, actualChunkData, cancellationToken);

                // 使用锁保证进度更新的原子性
                lock (progressLock)
                {
                    uploadedCount++;
                    var progress = (int)((double)uploadedCount / totalToUpload * 100);
                    ProgressChanged?.Invoke(progress);
                }
            });

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

主窗体MainForm中对于ParallelUploadService服务类的调用方式与ResumableUploadService类似，不再展示。

## 动态调整分片大小

根据网络状态动态调整分片大小，可以优化分片上传效率。

### 服务端实现

由于分片的大小是动态计算得出的，存储的临时文件名由fileId_chunkNumber变更为{fileId}/chunk_{offsetValue}。

服务端的分片存储结构如下：

>uploads/
├── {fileId}/          // 每个文件一个独立目录
│   ├── chunk_{offsetValue1}        // 分片文件按偏移量命名
│   ├── chunk_{offsetValue2}
│   └── ...
└── completed/       // 最终合并的文件

由于上传是并行的，服务端在检查分片上传完整性时，必须以所有分片大小总和等于文件大小为标志，而不能以其中某一分片的偏移量+分片大小等于文件大小为标志。

```c#
[ApiController]
[Route("[controller]/[action]")]
[Authorize]
public class UploadController: ControllerBase
{
    private readonly IWebHostEnvironment _env;
    private readonly string _uploadPath;
    private readonly ILogger<UploadController> _logger;

    public UploadController(IWebHostEnvironment env, ILogger<UploadController> logger)
    {
        _env = env;
        _uploadPath = Path.Combine(_env.WebRootPath, "uploads");
        _logger = logger;
    }

    internal class UploadedChunk
    {
        public long Offset { get; set; }
        public int Size { get; set; }
    }

    [HttpGet]
    [AllowAnonymous]
    public IActionResult Ping()
    {
        return Ok(DateTime.UtcNow);
    }

    [HttpGet]
    [AllowAnonymous]
    public IActionResult GetUploadStatus(string fileId)
    {
        var uploadDir = Path.Combine(_uploadPath, fileId);
        if (!Directory.Exists(uploadDir))
        {
            return Ok(new { UploadedChunks = Array.Empty<UploadedChunk>() });
        }

        var chunks = Directory.GetFiles(uploadDir, "chunk_*")
            .Select(f => new FileInfo(f))
            .Select(f => new UploadedChunk
            {
                Offset = long.Parse(f.Name.Split('_')[1]),
                Size = (int)f.Length
            })
            .OrderBy(c => c.Offset)
            .ToList();

        return Ok(new { UploadedChunks = chunks });
    }

    [HttpPost]
    [AllowAnonymous]
    public async Task<IActionResult> UploadChunkAsync(CancellationToken cancellationToken)
    {
        var form = await Request.ReadFormAsync(cancellationToken);
        var fileId = form["fileId"].ToString();
        var chunkIndex = int.Parse(form["chunkIndex"].ToString());
        var chunkOffset = int.Parse(form["chunkOffset"].ToString());
        var chunkSize = int.Parse(form["chunkSize"].ToString());
        var fileSize = int.Parse(form["fileSize"].ToString());
        var fileName = form["fileName"].ToString();
        var chunk = form.Files["chunk"];

        try
        {
            // 验证分片
            if (chunk == null || chunk.Length == 0)
            {
                return BadRequest("无效的分片数据");
            }

            // 创建上传目录
            var uploadDir = Path.Combine(_uploadPath, fileId);
            Directory.CreateDirectory(uploadDir);

            // 保存分片
            var chunkPath = Path.Combine(uploadDir, $"chunk_{chunkOffset}");
            using (var stream = new FileStream(chunkPath, FileMode.Create))
            {
                await chunk.CopyToAsync(stream, cancellationToken);
            }

            _logger.LogInformation($"已接收分片 {chunkIndex} (偏移: {chunkOffset}, 大小: {chunk.Length}, 剩余大小: {fileSize - chunkOffset})");

            // 检查是否完成
            if (IsUploadComplete(fileId, fileSize))
            {
                _logger.LogInformation($"文件 {fileName} 所有分片已上传");
                await MergeChunksAsync(fileId, fileName, cancellationToken);
                return Ok(new { Completed = true });
            }

            return Ok(new { Completed = false });
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "分片上传失败");
            return StatusCode(500, new { Error = ex.Message });
        }
    }

    private bool IsUploadComplete(string fileId, long fileSize)
    {
        var status = GetUploadStatus(fileId) as OkObjectResult;
        var data = status?.Value as dynamic;
        var chunks = data?.UploadedChunks as IEnumerable<UploadedChunk>;

        if (chunks == null || !chunks.Any())
        {
            return false;
        }

        // 获取文件总大小(最后一个分片的结束位置)
        // 请勿使用chunks.Max(c => c.Offset + c.Size), 该值在单线程环境中可以代表所有分片都已上传完毕, 但在多线程环境中是错误的
        long currentTotalSize = chunks.Sum(c => c.Size);

        // 检查是否覆盖了所有字节
        return currentTotalSize >= fileSize;
    }

    private async Task MergeChunksAsync(string fileId, string fileName, CancellationToken cancellationToken)
    {
        var uploadDir = Path.Combine(_uploadPath, fileId);
        var finalDir = Path.Combine(_uploadPath, "completed");
        var finalPath = Path.Combine(finalDir, fileName);
        if(!Directory.Exists(finalDir))
        {
            Directory.CreateDirectory(finalDir);
        }

        // 获取所有分片并按偏移量排序
        var chunkFiles = Directory.GetFiles(uploadDir, "chunk_*")
            .Select(f => new
            {
                Path = f,
                Offset = long.Parse(Path.GetFileName(f).Split('_')[1])
            })
            .OrderBy(x => x.Offset)
            .ToList();

        // 合并文件
        using (var finalStream = new FileStream(finalPath, FileMode.Create))
        {
            foreach (var chunk in chunkFiles)
            {
                using (var chunkStream = System.IO.File.OpenRead(chunk.Path))
                {
                    await chunkStream.CopyToAsync(finalStream, cancellationToken);
                }
                System.IO.File.Delete(chunk.Path);
            }
        }

        // 清理临时目录
        Directory.Delete(uploadDir);

        _logger.LogInformation($"文件合并完成: {finalPath}");
    }
}
```

调用服务端上传时的日志记录如下，可以看到分片的上传是乱序的：

{% asset_img adaptive_resumable_upload_console.png  服务端日志记录输出 %}

### 客户端实现

客户端实现了AdaptiveResumableUploadService服务类，在并行上传与断点续传的基础上，添加了实时更新网络指标(上行速度/网络延迟)并动态计算当前分片大小的功能，并使用了异步锁SemaphoreSlim确保分片信息更新时的原子性。

```c#
public class AdaptiveResumableUploadService
{
    private readonly HttpClient _httpClient;
    private readonly string _serviceUri;
    private const string UploadChunkApiUri = "/api/Upload/UploadChunk";
    private const string GetUploadStatusApiUri = "/api/Upload/GetUploadStatus";
    private const string PingApiUri = "/api/Ping";
    private readonly int _maxParallelism;
    private readonly int _minChunkSize;
    private readonly int _maxChunkSize;
    private readonly int _initialChunkSize;

    // 网络状况监测
    private double _averageUploadSpeedMbps = 1.0; // 初始假设1Mbps
    private double _averageLatencyMs = 100; // 初始延迟100ms
    private readonly object _networkLock = new object();
    private readonly SemaphoreSlim _semaphoreSlim = new SemaphoreSlim(1, 1);

    // 进度和状态事件
    public event Action<int> ProgressChanged; // 上传百分比
    public event Action<string> StatusChanged; // 状态消息
    public event Action<bool> UploadCompleted; // 完成状态

    public AdaptiveResumableUploadService(string serviceUri, 
        int maxParallelism = 4,
        int minChunkSize = 256 * 1024,    // 256KB
        int maxChunkSize = 10 * 1024 * 1024, // 10MB
        int initialChunkSize = 1 * 1024 * 1024) // 1MB
    {
        _serviceUri = serviceUri;
        _httpClient = new HttpClient();
        _httpClient.Timeout = Timeout.InfiniteTimeSpan;
        _maxParallelism = maxParallelism;
        _minChunkSize = minChunkSize;
        _maxChunkSize = maxChunkSize;
        _initialChunkSize = initialChunkSize;
    }

    // 主上传方法
    public async Task UploadFileAdaptiveAsync(string filePath, CancellationToken cancellationToken)
    {
        try
        {
            StatusChanged?.Invoke("正在准备上传...");

            var fileInfo = new FileInfo(filePath);
            var fileId = GetFileMD5(filePath); // 基于文件内容生成唯一ID

            StatusChanged?.Invoke("正在检查已上传分片...");

            // 获取已上传的分片信息
            var uploadedChunks = await GetUploadedChunksAsync(fileId, cancellationToken);

            // 准备上传任务
            var currentChunkSize = _initialChunkSize;
            var fileSize = fileInfo.Length;

            var progressLock = new object();
            long totalUploaded = uploadedChunks.Values.Sum();
            long totalToUpload = fileSize - totalUploaded;

            StatusChanged?.Invoke($"需要上传 {totalToUpload} 字节");

            var parallelOptions = new ParallelOptions
            {
                MaxDegreeOfParallelism = _maxParallelism,
                CancellationToken = cancellationToken
            };

            using var fileStream = File.OpenRead(filePath);

            await Parallel.ForEachAsync(GenerateChunks(fileSize, currentChunkSize, uploadedChunks), parallelOptions, async (chunk, cancellationToken) =>
            {
                cancellationToken.ThrowIfCancellationRequested();

                bool success = await UploadChunkAsync(fileId, chunk.Index, chunk.Offset, chunk.Size, fileSize, fileInfo.Name, fileStream, cancellationToken);

                if (success)
                {
                    // 使用锁保证进度更新的原子性
                    lock (progressLock)
                    {
                        totalUploaded += chunk.Size;
                        int progress = (int)((double)totalUploaded / totalToUpload * 100);
                        ProgressChanged?.Invoke(progress);
                    }
                }
            });

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

    private  IEnumerable<FileChunk> GenerateChunks(long fileSize, int currentChunkSize, ConcurrentDictionary<long, int> uploadedChunks)
    {
        long position = 0;
        int chunkIndex = 0;

        while (position < fileSize)
        {
            _semaphoreSlim.WaitAsync(); // 使用异步锁确保分片信息更新时的原子性
            if (!uploadedChunks.TryGetValue(position, out var chunkSize))
            {
                // 动态调整分片大小
                currentChunkSize = CalculateDynamicChunkSize();
                var actualChunkSize = (int)Math.Min(currentChunkSize, fileSize - position);

                yield return new FileChunk
                {
                    Index = chunkIndex,
                    Offset = position,
                    Size = actualChunkSize,
                    RemainingSize = fileSize - (position + actualChunkSize)
                };

                position += actualChunkSize;
            }
            else
            {
                position += chunkSize;
            }

            chunkIndex++;

            _semaphoreSlim.Release();
        }
    }

    private string GetFileMD5(string filePath)
    {
        using var md5 = MD5.Create();
        using var stream = File.OpenRead(filePath);
        var hash = md5.ComputeHash(stream);
        return BitConverter.ToString(hash).Replace("-", "").ToLower();
    }

    // 获取已上传的分片信息
    private async Task<ConcurrentDictionary<long, int>> GetUploadedChunksAsync(string fileId, CancellationToken cancellationToken)
    {
        try
        {
            var response = await _httpClient.GetAsync($"{_serviceUri}{GetUploadStatusApiUri}?fileId={fileId}", cancellationToken);
            response.EnsureSuccessStatusCode();

            var result = await response.Content.ReadFromJsonAsync<UploadStatusResponse>(cancellationToken);
            return new ConcurrentDictionary<long, int>(result.UploadedChunks.ToDictionary(x => x.Offset, x => x.Size));
        }
        catch
        {
            return new ConcurrentDictionary<long, int>();
        }
    }

    // 上传单个分片
    private async Task<bool> UploadChunkAsync(string fileId, int chunkIndex, long chunkOffset, int chunkSize, long fileSize,  string fileName, Stream fileStream, CancellationToken cancellationToken)
    {
        var stopwatch = Stopwatch.StartNew();

        try
        {
            // 测量请求延迟
            var latencyStopwatch = Stopwatch.StartNew();
            var pingResponse = await _httpClient.GetAsync($"{_serviceUri}{PingApiUri}", cancellationToken);
            latencyStopwatch.Stop();

            // 准备分片数据
            var chunkData = new byte[chunkSize];
            fileStream.Seek(chunkOffset, SeekOrigin.Begin);
            var bytesRead = await fileStream.ReadAsync(chunkData, 0, chunkSize, cancellationToken);

            if (bytesRead == 0) return false;

            using var content = new MultipartFormDataContent
            {
                { new StringContent(fileId), "fileId" },
                { new StringContent(chunkIndex.ToString()), "chunkIndex" },
                { new StringContent(chunkOffset.ToString()), "chunkOffset" },
                { new StringContent(bytesRead.ToString()), "chunkSize"},
                { new StringContent(fileSize.ToString()), "fileSize"},
                { new StringContent(fileName), "fileName" },
                { new ByteArrayContent(chunkData, 0, bytesRead), "chunk", "chunk.dat" }
            };

            // 上传
            var uploadStopwatch = Stopwatch.StartNew();
            var response = await _httpClient.PostAsync($"{_serviceUri}{UploadChunkApiUri}", content, cancellationToken);
            uploadStopwatch.Stop();

            response.EnsureSuccessStatusCode();

            // 更新网络指标
            UpdateNetworkMetrics(bytesRead, uploadStopwatch.Elapsed, latencyStopwatch.Elapsed);
            return true;
        }
        finally
        {
            stopwatch.Stop();
        }
    }

    #region 更新网络指标并动态计算当前分片大小
    // 动态计算当前分片大小
    private int CalculateDynamicChunkSize()
    {
        lock (_networkLock)
        {
            // 基于当前上传速度和延迟计算理想分片大小
            double targetUploadTimeSec = 2.0; // 目标每个分片上传时间
            double speedFactor = _averageUploadSpeedMbps* 125000; // Mbps -> bytes/sec
            double latencyFactor = Math.Max(1, _averageLatencyMs / 50);

            int idealSize = (int)(speedFactor * targetUploadTimeSec / latencyFactor);

            // 限制在最小和最大值之间
            return (int)Math.Clamp(idealSize, _minChunkSize, _maxChunkSize);
        }
    }

    // 更新网络指标
    private void UpdateNetworkMetrics(long chunkSize, TimeSpan uploadTime, TimeSpan latency)
    {
        lock (_networkLock)
        {
            // 计算速度 (Mbps)
            double speedMbps = (chunkSize * 8 / uploadTime.TotalSeconds) / 1_000_000;
            // 平滑处理速度值（加权平均）
            _averageUploadSpeedMbps = 0.7 * _averageUploadSpeedMbps + 0.3 * speedMbps;
            // 更新延迟
            _averageLatencyMs = 0.8 * _averageLatencyMs + 0.2 * latency.TotalMilliseconds;

            StatusChanged?.Invoke($"网络: {_averageUploadSpeedMbps:F1}Mbps, " +
                                $"延迟: {_averageLatencyMs:F0}ms, " +
                                $"分片: {CalculateDynamicChunkSize() / 1024}KB");
        }
    }
    #endregion


    #region record
    private record UploadedChunk
    {
        public long Offset { get; init; }
        public int Size { get; init; }
    }

    private record UploadStatusResponse
    {
        public List<UploadedChunk> UploadedChunks { get; init; }
    }

    private record FileChunk
    {
        public int Index { get; init; }
        public long Offset { get; init; }
        public int Size { get; init; }
        public long RemainingSize { get; init; }
    }
    #endregion
}
```

## 分片完整性校验

未完待续...

## 分片清理

未完待续...

## 参考文档

- [阿里云OSS断点续传上传操作指南](https://help.aliyun.com/zh/oss/user-guide/resumable-upload)

- [Control.InvokeRequired属性参考文档](https://learn.microsoft.com/zh-cn/dotnet/api/system.windows.forms.control.invokerequired)