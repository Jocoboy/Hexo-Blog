---
title: FastAPI+Vue+PyInstaller一体化部署与打包方案
date: 2026-08-12 15:22:28
categories:
- Package-Tool
tags:
- Vue
- Fast API
- PyInstaller
- Python
---

实现FastAPI + Vue一体化部署(跳过HTTP代理服务器)，并使用PyInstaller打包。

<!--more-->

## FastAPI + Vue一体化部署

使用Fast API后端可以代替HTTP服务器(如Nignx)托管前端Vue dist包，这样前端页面和后台接口就在同一个地址和端口上，既解决了跨域问题，也方便一体化部署。

项目目录结构如下，

my-app/
├── app
│   ├─ \__init__.py
│   ├─ license  
│   └─ ...
│
├── dist/
│   ├── assets
│   ├── resource
│   ├── index.html
│   └── ...
├── vite.config.js
├── run.py
├── config.yml
└── ...

首先修改vite.config.js文件，

```js
...
build: {
    // outDir: OUTPUT_DIR || 'dist',
    outDir: '../dist',
    ...
},
...
```

然后修改app下的__init__.py文件，

```python
import mimetypes
import sys
import os
...

# 必须在挂载 StaticFiles 之前执行，强制覆盖 .js 和 .mjs 的 MIME 类型映射
if sys.platform == "win32":
    mimetypes.add_type("application/javascript", ".js")
    mimetypes.add_type("application/javascript", ".mjs")

# 获取当前文件所在目录 (app 目录)
CURRENT_DIR = Path(__file__).parent.absolute()
PROJECT_ROOT = CURRENT_DIR.parent
DIST_DIR = os.path.join(PROJECT_ROOT, "dist")
app.mount("/", StaticFiles(directory=DIST_DIR, html=True), name="vue-app")
```

可将上述修改进一步优化，引入config.yml配置文件，对开发环境与打包环境作区分：
```yml
# 运行环境配置
env:
  # 环境标识：0-开发环境(默认)，1-打包环境
  profile: 1
```

app下的__init__.py文件对应代码修改为：
```python
import mimetypes
import sys
import os
...

def resource_path(relative_path):
    """获取资源绝对路径，兼容开发环境和 PyInstaller 打包环境"""
    if getattr(sys, 'frozen', False):
        # 运行于 PyInstaller 打包环境：使用sys.executable(exe 所在目录)
        base_path = os.path.dirname(sys.executable)
    else:
        # 运行于普通 Python 开发环境：使用根目录
        project_root = os.path.abspath(os.path.join(os.path.dirname(__file__), os.pardir))
        base_path = os.path.abspath(os.path.join(project_root, os.pardir)) 
    return os.path.join(base_path, relative_path) if relative_path else base_path

# 必须在挂载 StaticFiles 之前执行，强制覆盖 .js 和 .mjs 的 MIME 类型映射
if sys.platform == "win32":
    mimetypes.add_type("application/javascript", ".js")
    mimetypes.add_type("application/javascript", ".mjs")

# 获取当前文件所在目录 (app 目录)
CURRENT_DIR = Path(__file__).parent.absolute()
PROJECT_ROOT = CURRENT_DIR.parent

# 在你的配置加载逻辑中，使用 resource_path 来获取路径
config_path = resource_path('config.yml')
# 然后用 config_path 去打开文件
with open(config_path, "r", encoding="utf-8") as f:
    _config = yaml.safe_load(f)
PROFILE = _config.get("env", {}).get("profile", 0)
IS_PRODUCTION = PROFILE == 1

# 只有打包环境才挂载 Vue 的静态文件
if IS_PRODUCTION:
    DIST_DIR = os.path.join(PROJECT_ROOT, "dist")
    app.mount("/", StaticFiles(directory=DIST_DIR, html=True), name="vue-app")
```

若要在app目录下引入license/license.lic验证，需要区分parent_dir目录的获取方式。
注：与config.yml文件类似，license目录同样不能通过spec的datas添加，否则已经构建好的exe内部归档并不会自动更新（读取的永远是打包时的快照）。

```python
def get_parent_dir():
    """
    获取持久化的基础目录（证书应从外部读取，不走 _MEIPASS）
    - 打包后:   exe 所在目录
    - 源码运行: 当前文件所在目录
    """
    if getattr(sys, 'frozen', False):
        # 打包后：sys.executable 是 exe 的真实路径
        exe_dir = os.path.dirname(sys.executable)
        return os.path.join(exe_dir, "app")
    else:
        # 源码运行
        script_dir = os.path.dirname(os.path.abspath(__file__))
        return os.path.abspath(os.path.join(script_dir, ".."))

parent_dir = get_parent_dir()
```


若app中存在审计中间件，对于特定路径需要跳过审计。

```python
# 定义需要跳过审计的路径前缀
SKIP_PATHS = ['/', '/assets/', '/resource/', '/favicon.ico']

async def after_request(self, request: Request, response: Response, process_time: int):
    # 检查是否应该跳过审计
    path = request.url.path
    for skip_path in self.SKIP_PATHS:
        if path.startswith(skip_path):
            return  # 直接返回，不记录审计日志
    ...
```

## PyInstaller

PyInstaller是一个将Python程序打包成独立可执行文件的工具，让你的程序能在没有Python环境的电脑上直接运行。它通过分析你的代码，将Python解释器、依赖库和脚本代码打包在一起，生成一个可执行文件或文件夹。

### 简单打包(以FastAPI应用为例)

对于一个最简单的FastAPI应用，如main.py:

```python
# main.py
from fastapi import FastAPI
import uvicorn
import multiprocessing
app = FastAPI()

@app.get("/")
async def root():
    return {"message": "Hello World"}

if __name__ == "__main__":
    multiprocessing.freeze_support()  # <-- 关键！防止Windows上启动多个进程
    uvicorn.run(app, host="0.0.0.0", port=8000)

```

使用命令`pyinstaller main.py --nocofirm`即可完成打包，打包结果会生成在dist目录内(单独的exe文件)。

上述命令默认使用`--onedir`单目录打包方式，也可以使用`--onefile`单文件打包方式。


### 进阶打包(指定spec文件)

.spec文件是PyInstaller的核心配置文件，它用Python语法定义了打包的全部规则，相当于一个可编程的打包说明书。当你有复杂需求（如添加资源文件、处理动态导入）时，直接修改这个文件会比在命令行输入一长串参数要清晰、可控得多。

一个标准的 .spec 文件主要由以下几个部分构成，它们共同协作完成打包：
- Analysis：这是最核心的分析与配置阶段。它会扫描主脚本，分析所有的依赖项，并允许你自定义需要包含的二进制文件（binaries）、数据文件（datas）、隐藏导入（hiddenimports）以及需要排除的模块（excludes）。
- PYZ：一个压缩包，将Analysis阶段收集到的所有Python字节码文件打包进去，以减少磁盘占用。
- EXE：定义最终生成的.exe可执行文件的配置，比如输出文件名、图标（icon）、是否显示控制台（console）以及是否启用UPX压缩（upx）等。
- COLLECT：只有在单文件夹模式（默认模式）下才会使用。它的作用是把EXE和Analysis阶段收集到的所有依赖文件（.dll、.pyd、数据文件等）复制到最终的输出文件夹dist里。

```python
# -*- mode: python ; coding: utf-8 -*-
import sys
import os
import glob
import site
import importlib
import pkgutil
from PyInstaller.utils.hooks import collect_data_files, collect_submodules, collect_all

block_cipher = None

# ============ 递归收集子模块函数 ============
def collect_submodules_recursive(package_name, exclude_patterns=None, max_depth=10):
    """
    递归收集包的所有子模块（包括多层嵌套）
    
    Args:
        package_name: 包名，如 'app'
        exclude_patterns: 排除的模式列表，如 ['*.tests', '*.sanic']
        max_depth: 最大递归深度
    
    Returns:
        list: 所有子模块名称列表
    """
    import fnmatch
    
    modules = []
    seen = set()
    
    if exclude_patterns is None:
        exclude_patterns = ['*.tests', '*.test', '*.examples']
    
    def should_exclude(module_name):
        """检查模块是否应该被排除"""
        for pattern in exclude_patterns:
            if fnmatch.fnmatch(module_name, pattern):
                return True
        return False
    
    def _collect(pkg_name, pkg_path, depth=0):
        """递归收集子模块"""
        if depth > max_depth:
            print(f"达到最大深度 {max_depth}，停止递归 {pkg_name}")
            return
        
        indent = "  " * depth
        
        # 添加当前包
        if pkg_name not in seen and not should_exclude(pkg_name):
            seen.add(pkg_name)
            modules.append(pkg_name)
            print(f"{indent} 收集包: {pkg_name}")
        
        try:
            # 迭代子模块
            for module_info in pkgutil.iter_modules([pkg_path]):
                module_name = f"{pkg_name}.{module_info.name}"
                
                if should_exclude(module_name):
                    print(f"{indent}  跳过: {module_name}")
                    continue
                
                if module_name not in seen:
                    seen.add(module_name)
                    modules.append(module_name)
                    print(f"{indent}  收集模块: {module_name}")
                    
                    # 如果是包，递归收集
                    if module_info.ispkg:
                        try:
                            sub_module = importlib.import_module(module_name)
                            if hasattr(sub_module, '__path__'):
                                sub_path = sub_module.__path__[0]
                                _collect(module_name, sub_path, depth + 1)
                        except Exception as e:
                            print(f"{indent} 无法导入子包 {module_name}: {e}")
                            
        except Exception as e:
            print(f"{indent} 迭代 {pkg_name} 时出错: {e}")
    
    try:
        pkg = importlib.import_module(package_name)
        if hasattr(pkg, '__path__'):
            package_path = pkg.__path__[0]
            print(f"\n开始递归收集 {package_name} (路径: {package_path})")
            _collect(package_name, package_path)
            print(f"收集完成，共 {len(modules)} 个模块\n")
        else:
            print(f"{package_name} 不是包")
            return []
    except Exception as e:
        print(f"无法导入 {package_name}: {e}")
        return []
    
    return modules

# datas示例：收集 pyDOE3 的所有数据文件（orthogonal_arrays/*.oa 等，打包后运行时按 __file__ 相对路径读取）
pyDOE3_datas = collect_data_files('pyDOE3')

# hiddenimports示例：收集 app 的所有子模块
app_modules = collect_submodules_recursive(
    'app', 
    exclude_patterns=['*.tests', '*.test', '*.pyc']
)

# ============ Analysis 配置 ============
a = Analysis(
    ['main.py'],
    pathex=[],
    binaries=[],
    datas=[
        ('dist', 'dist'),
        ('app/algorithm/libs', 'app/algorithm/libs') if os.path.exists('app/algorithm/libs') else None
        # 将数据文件打包到根目录，对应命令行方式为：pyinstaller --add-data "your_data_file;." main.py  
        # 注：Windows下源文件与目标路径之间用分号 ";" 分隔，目标路径写 "." 表示将文件放在打包根目录下
    ] + pyDOE3_datas,
    hiddenimports=[
        # FastAPI 和 Uvicorn
        'uvicorn',
        'uvicorn.logging',
        'uvicorn.loops.auto',
        'uvicorn.protocols.http',
        'uvicorn.protocols.http.auto',
        'fastapi',
        'fastapi.routing',
    ] + app_modules,
    hookspath=[],
    hooksconfig={},
    runtime_hooks=[],
    excludes=[],
    noarchive=False,
    optimize=0,
)
# ============ 生成 PYZ ============
pyz = PYZ(a.pure, a.zipped_data, cipher=block_cipher)

# ========== 单目录模式 ==========
exe = EXE(
    pyz,
    a.scripts,
    [],
    exclude_binaries=True,
    name='main',
    debug=False,
    bootloader_ignore_signals=False,
    strip=False,
    upx=True,
    console=True,
    disable_windowed_traceback=False,
    argv_emulation=False,
    target_arch=None,
    codesign_identity=None,
    entitlements_file=None,
)
coll = COLLECT(
    exe,
    a.binaries,
    a.datas,
    strip=False,
    upx=True,
    upx_exclude=[],
    name='main',
)
```

使用命令`pyinstaller main.spec --noconfirm`即可通过spec文件完成打包。

此外，通过添加`--distpath`参数可以更改默认的dist目录为指定目录, 例如`pyinstaller main.spec --distpath .\your-pyinstaller-dist --noconfirm`。

注: 通过显示指定添加的py文件夹(如'app/algorithm/libs')，源码可能未被加密。

可使用加密工具pyarmor对整个目录的py文件进行加密，使用命令`pyarmor gen -r -O ./your-pyarmor-dist app`即可输出加密文件到指定目录。

其中生成的pyarmor_runtime_*文件夹是运行加密代码所必需的，不能删除。需要手动将它移动到与入口脚本相同的目录下(示例为app目录下)。

## 参考文档

- [PyInstaller官方文档](https://pyinstaller.org/en/stable/)