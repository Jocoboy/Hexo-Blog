---
title: 基于Docker的应用软件离线许可证颁发方案
date: 2026-05-14 17:34:09
categories:
- Software-License
tags:
- Docker
- Python
- Fast API
- Asymmetric Encryption
---

为构建并部署在Docker容器中的FastAPI应用设计一个离线的软件许可证颁发方案。

<!--more-->

## 前言

FastAPI是一个现代、快速（高性能）的Web框架，用于基于标准Python类型提示构建API。
在商业化场景中，需要为FastAPI添加软件许可验证机制，而现代化的FastAPI应用通常部署在Docker容器中(与宿主机隔离)。
为此，本文设计了一种基于Docker的应用软件离线许可证颁发方案，使用非对称加密技术，无需通过联网环境即可完成软件/产品的许可验证。


## 许可证生成架构图

{% asset_img product-license-generate-diagram.png 许可证生成架构图 %}

## 基本流程

### 私钥与公钥生成

使用非对称加密算法，生成一组私钥和公钥。其中私钥用于生成License.lic，用户的FastAPI应用里只用公钥验证，整个过程完全离线。

```python
# key_generate.py

# 运行一次，生成 private_key.pem 和 public_key.pem
from cryptography.hazmat.primitives import serialization
from cryptography.hazmat.primitives.asymmetric import rsa

# 生成私钥
private_key = rsa.generate_private_key(public_exponent=65537, key_size=2048)
public_key = private_key.public_key()

# 保存私钥 (你自己留着，用来给客户生成License文件)
with open("private_key.pem", "wb") as f:
    f.write(private_key.private_bytes(
        encoding=serialization.Encoding.PEM,
        format=serialization.PrivateFormat.PKCS8,
        encryption_algorithm=serialization.NoEncryption()
    ))

# 保存公钥 (这个和下面的代码一起给客户)
with open("public_key.pem", "wb") as f:
    f.write(public_key.public_bytes(
        encoding=serialization.Encoding.PEM,
        format=serialization.PublicFormat.SubjectPublicKeyInfo
    ))
```

运行后你将得到private_key.pem与public_key.pem两个文件。

### 许可证信息准备

在配置文件license_info.json中配置机器码machine_id和许可证过期时间expire_days。

```json
{     
    "machine_id": "e29633fade0fc20cbf6a4ff8144f8b09ea97db1e83830e4a9551cef7ae482b94",     
    "expire_days": 365 
}
```

其中machine_id可用跨平台机器ID库py-machineid获取。

`pip install py-machineid`

该库在被调用时，会直接尝试读取宿主机的硬件信息（如Windows注册表、Linux的/etc/machine-id或macOS的硬件序列号）。这部分操作完全由Python代码执行，绕过了Kubernetes的环境注入机制。
由于容器内的Python进程看到的是独立的文件系统和注册表，py-machineid会读取容器内部的machine-id。这个ID通常是Docker为容器生成的虚拟ID，会随容器的重建而发生改变。

为了使py-machineid读取的始终为宿主机的ID，常规方法是通过挂载将宿主机的machine-id文件覆盖容器内的同名文件。例如宿主机为Linux系统，在运行容器时添加-v参数:

`docker run -d -v /etc/machine-id:/etc/machine-id:ro your-image:latest`

由于不同操作系统文件路径差异较大，最稳妥的方式还是通过环境变量传递machine-id，可在运行容器时添加-e参数:

`docker run -d -e HOST_MACHINE_ID="your-actual-host-machine-id-here" your-image:latest`

```python
import os
import machineid
import hashlib

def get_machine_fingerprint() -> str:
    # 1. 优先从环境变量获取（适用于容器）

    env_id = os.environ.get("HOST_MACHINE_ID")
    if env_id:
        print("Using machine ID from environment variable")
        return env_id
    
    # 2. 备选：使用 py-machineid 直接读取
    #    （如果文件没有被挂载覆盖，在容器内读取到的就是容器 ID）

    # 获取原始机器ID（不包含应用标识）
    machine_id = machineid.id()

    # 或者获取匿名化的哈希版本（推荐用于License场景）
    # 传入你的应用ID，确保不同应用获取不同的ID
    hashed_id = machineid.hashed_id('your_app_name')
    
    print("Using direct machine ID from system")
    # 返回SHA256哈希作为指纹
    return hashlib.sha256(hashed_id.encode()).hexdigest()
```

技术人员可使用license_info_generate.py脚本在docker应用程序的宿主机上自动生成license_info.json文件，并用私钥private_key.pem生成对应的软件许可证license.lic颁发给用户。

用户将license.lic文件放在相应目录后，每次FastAPI应用启动时都将使用public_key.pem对其解析并验证，验证过程中同样使用get_machine_fingerprint方法。

```python
# license_info_generate.py

import hashlib
import json
import machineid

def get_machine_fingerprint() -> str:
    # 同上省略

machine_id = get_machine_fingerprint()

# 定义数据
data = {
    "machine_id": machine_id,
    "expire_days": 365
}

# 写入JSON文件
with open('license_info.json', 'w', encoding='utf-8') as f:
    json.dump(data, f, indent=4)

print("license_info.json file generated")
```

### 许可证颁发

使用前面生成的私钥private_key.pem生成license.lic文件，用于后续的FastAPI应用许可验证。

```python
# license_issue.py

import json
import base64
from cryptography.hazmat.primitives import hashes, serialization
from cryptography.hazmat.primitives.asymmetric import padding
from datetime import datetime, timedelta, timezone

def generate_license(customer_machine_id: str, expire_days: int = 365):
    # 1. 加载你的私钥
    with open("private_key.pem", "rb") as f:
        private_key = serialization.load_pem_private_key(f.read(), password=None)

    # 2. 构建授权数据
    payload = {
        "machine_id": customer_machine_id,  # 绑定的机器ID
        "expire": (datetime.now(timezone.utc) + timedelta(days=expire_days)).isoformat(),
        #"features": ["basic", "export"]     # 可以控制开放的功能模块
    }
    payload_bytes = json.dumps(payload).encode('utf-8')

    # 3. 用私钥签名
    signature = private_key.sign(
        payload_bytes,
        padding.PKCS1v15(),
        hashes.SHA256()
    )

    # 4. 组合数据并生成最终License字符串
    license_data = base64.b64encode(payload_bytes).decode('utf-8')
    license_signature = base64.b64encode(signature).decode('utf-8')
    final_license = f"{license_data}.{license_signature}"

    print("This license string is provided to the customer:")
    print(final_license)

    return final_license

def write_license_simple(license_content):
    """最简单的写入方式"""
    with open("license.lic", "w") as f:
        f.write(license_content)
    
    print("License file created: license.lic")

# 读取JSON配置文件
with open('license_info.json', 'r') as f:
    config = json.load(f)

# 提取参数到变量
machine_id = config['machine_id']
expire_days = config['expire_days']

lic = generate_license(machine_id, expire_days)
write_license_simple(lic)
```

### 应用许可验证

```python
# server_validate.py

import platform
import uuid
from fastapi import FastAPI, Depends, HTTPException, Request
from cryptography.hazmat.primitives import hashes, serialization
from cryptography.hazmat.primitives.asymmetric import padding
import json
import base64
from datetime import datetime, timezone

app = FastAPI()

# 加载公钥 (你需要把生成的 public_key.pem 随代码一起提供给客户)
with open("public_key.pem", "rb") as f:
    PUBLIC_KEY = serialization.load_pem_public_key(f.read())

import subprocess
import hashlib

import hashlib
import uuid

def get_machine_fingerprint() -> str:
    # 同上省略

def verify_license(license_str: str):
    """验证License字符串是否有效、是否过期、是否匹配本机"""
    try:
        # 1. 拆分数据和签名
        data_b64, signature_b64 = license_str.split('.')
        payload_bytes = base64.b64decode(data_b64)
        signature = base64.b64decode(signature_b64)

        # 2. 用公钥验证签名 (确保License未被篡改)
        PUBLIC_KEY.verify(
            signature,
            payload_bytes,
            padding.PKCS1v15(),
            hashes.SHA256()
        )

        # 3. 解析授权数据并验证有效期
        payload = json.loads(payload_bytes)
        expire_time = datetime.fromisoformat(payload["expire"])
        if expire_time < datetime.now(timezone.utc):
            raise HTTPException(status_code=403, detail="License is expired")

        # 4. 验证机器指纹
        current_machine_id = get_machine_fingerprint()  
        print("current machine id:" + current_machine_id)
        if payload["machine_id"] != current_machine_id:
            raise HTTPException(status_code=403, detail="The license does not match the current device")

        return payload

    except Exception as e:
        raise HTTPException(status_code=403, detail=f"License validate failed: {str(e)}")

# 加载license
with open("license.lic", "r") as f:
    lic = f.read()
    print(f"read content: '{lic}'")

res = verify_license(lic)
print(res)
```

## Docker集成测试

demo目录结构如下，

```
docker
├─ Dockerfile
├─ license.lic
├─ public_key.pem
├─ requirements.txt
└─ server_validate.py
```

其中许可证书license.lic和公钥public_key.pem的生成方式如前文所述，不再赘述。

requirements.txt包含FastAPI运行所需的依赖，本demo示例需要的依赖如下，

```
fastapi==0.104.1
uvicorn==0.24.0
cryptography==41.0.7
pydantic==2.5.0
python-multipart==0.0.6
py-machineid==1.0.0
```

server_validate.py是一个简单的FastAPI示例，该应用会自动检查py-machineid依赖库是否加载成功，自动加载许可证书license.lic和公钥public_key.pem

```python
import json
import base64
import hashlib
import os
from datetime import datetime, timezone
from fastapi import FastAPI, Depends, HTTPException, Request
from cryptography.hazmat.primitives import hashes, serialization
from cryptography.hazmat.primitives.asymmetric import padding
from pydantic import BaseModel
from typing import Optional

# 导入 py-machineid 库
try:
    import machineid
    MACHINEID_AVAILABLE = True
    print("✓ py-machineid 库加载成功")
except ImportError:
    MACHINEID_AVAILABLE = False
    print("⚠️ py-machineid 库未安装，将使用备用方案")

app = FastAPI(title="License Validation Service")

# 加载公钥
def load_public_key():
    """从文件加载公钥"""
    current_dir = os.path.dirname(os.path.abspath(__file__))
    public_key_path = os.path.join(current_dir, "public_key.pem")
    
    try:
        with open(public_key_path, "rb") as f:
            public_key = serialization.load_pem_public_key(f.read())
        print("✓ 公钥加载成功")
        return public_key
    except Exception as e:
        print(f"✗ 公钥加载失败: {e}")
        return None

PUBLIC_KEY = load_public_key()

# 加载License文件
def load_license():
    """从文件加载License"""
    current_dir = os.path.dirname(os.path.abspath(__file__))
    license_path = os.path.join(current_dir, "license.lic")
    
    try:
        with open(license_path, "r") as f:
            license_str = f.read().strip()
        print("✓ License文件加载成功")
        return license_str
    except Exception as e:
        print(f"✗ License文件加载失败: {e}")
        return None

LICENSE_STR = load_license()

def get_machine_fingerprint() -> str:
    # 1. 优先从环境变量获取（适用于容器）

    env_id = os.environ.get("HOST_MACHINE_ID")
    if env_id:
        print("Using machine ID from environment variable")
        return env_id
    
    # 2. 备选：使用 py-machineid 直接读取
    #    （如果文件没有被挂载覆盖，在容器内读取到的就是容器 ID）

    # 获取原始机器ID（不包含应用标识）
    machine_id = machineid.id()

    # 或者获取匿名化的哈希版本（推荐用于License场景）
    # 传入你的应用ID，确保不同应用获取不同的ID
    hashed_id = machineid.hashed_id('vv')
    
    print("Using direct machine ID from system")
    # 返回SHA256哈希作为指纹
    return hashlib.sha256(hashed_id.encode()).hexdigest()


def verify_license(license_str : str) -> dict:
    """验证License并返回授权信息"""
    if not PUBLIC_KEY:
        raise HTTPException(status_code=500, detail="公钥未正确加载")
    
    try:
        # 1. 拆分License数据和签名
        if '.' not in license_str:
            raise ValueError("Invalid license format")
        
        data_b64, signature_b64 = license_str.split('.')
        payload_bytes = base64.b64decode(data_b64)
        signature = base64.b64decode(signature_b64)
        
        # 2. 验证签名
        PUBLIC_KEY.verify(
            signature,
            payload_bytes,
            padding.PKCS1v15(),
            hashes.SHA256()
        )
        
        # 3. 解析授权数据
        payload = json.loads(payload_bytes)
        
        # 4. 验证有效期
        expire_time = datetime.fromisoformat(payload["expire"])
        current_time = datetime.now(timezone.utc)
        
        if expire_time < current_time:
            raise HTTPException(status_code=403, detail="License已过期")
        
        # 5. 验证机器指纹（关键步骤）
        current_fingerprint = get_machine_fingerprint()
        bound_fingerprint = payload.get("machine_id")
        
        if bound_fingerprint and current_fingerprint != bound_fingerprint:
            raise HTTPException(
                status_code=403, 
                detail=f"License与当前设备不匹配\n期望: {bound_fingerprint[:32]}...\n实际: {current_fingerprint[:32]}..."
            )
        
        return payload
        
    except HTTPException:
        raise
    except Exception as e:
        raise HTTPException(status_code=403, detail=f"License验证失败: {str(e)}")


def get_current_license(request: Request) -> dict:
    """从请求中获取并验证License"""
    # 支持从Header或Query参数获取License
    license_key = request.headers.get("X-License-Key")
    if not license_key:
        license_key = request.query_params.get("license_key")

    if not license_key:
        license_key = LICENSE_STR

    if not license_key:
    raise HTTPException(status_code=401, detail="请提供License (X-License-Key header 或 license_key参数)")
    
    return verify_license(license_key)


# ==================== API 路由 ====================

@app.get("/")
async def root():
    """根路径"""
    return {
        "service": "License Validation Service",
        "status": "running",
        "version": "2.0.0",
        "machineid_available": MACHINEID_AVAILABLE
    }


@app.get("/health")
async def health_check():
    """健康检查接口"""
    return {
        "status": "healthy",
        "public_key_loaded": PUBLIC_KEY is not None,
        "license_loaded": LICENSE_STR is not None,
        "machineid_available": MACHINEID_AVAILABLE
    }


@app.get("/machine-fingerprint")
async def get_fingerprint():
    """获取当前机器的指纹（用于生成License）"""
    fingerprint = get_machine_fingerprint()
    return {
        "machine_fingerprint": fingerprint,
        "machineid_available": MACHINEID_AVAILABLE,
        "note": "请将此指纹提供给管理员以生成License"
    }


@app.get("/validate")
async def validate_license(license_info: dict = Depends(get_current_license)):
    """验证License接口"""
    return {
        "valid": True,
        "message": "License验证通过",
        "license_info": {
            "machine_id": license_info.get("machine_id", ""),
            "expire_days": license_info.get("expire")
        }
    }


@app.get("/protected")
async def protected_endpoint(license_info: dict = Depends(get_current_license)):
    """受保护的API示例"""
    return {
        "message": "访问成功！",
        "license_valid": True,
        "expires_at": license_info.get("expire")
    }


if __name__ == "__main__":
    import uvicorn
    print("=" * 50)
    print("License Validation Service Starting...")
    print(f"Public Key: {'✓ Loaded' if PUBLIC_KEY else '✗ Failed'}")
    print(f"License: {'✓ Loaded' if LICENSE_STR else '✗ Not found'}")
    print(f"py-machineid: {'✓ Available' if MACHINEID_AVAILABLE else '✗ Not installed'}")
    print(f"Machine Fingerprint: {get_machine_fingerprint()[:48]}...")
    print("=" * 50)
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

Dockerfile文件如下，使用命令`docker build -t your-image:latest .`以构建镜像，

使用命令`docker run -d -e HOST_MACHINE_ID="your-actual-host-machine-id-here" --name your-image -p 8000:8000 your-image:latest `以创建并运行容器，同时传入环境变量HOST_MACHINE_ID

```Dockerfile
# 使用官方Python镜像
FROM python:3.11-slim

# 设置工作目录
WORKDIR /app

# 设置环境变量
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    TZ=Asia/Shanghai

# 安装系统依赖（py-machineid 在某些系统上需要）
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    gcc \
    && rm -rf /var/lib/apt/lists/*

# 复制依赖文件
COPY requirements.txt .

# 安装Python依赖
RUN pip install --no-cache-dir -r requirements.txt

# 复制应用程序文件
COPY server_validate.py .
COPY public_key.pem .
COPY license.lic .

# 创建非root用户运行应用
RUN useradd -m -u 1000 appuser && \
    chown -R appuser:appuser /app
USER appuser

# 暴露端口
EXPOSE 8000

# 健康检查
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD python -c "import requests; requests.get('http://localhost:8000/health')" || exit 1

# 启动命令
CMD ["uvicorn", "server_validate:app", "--host", "0.0.0.0", "--port", "8000"]
```


使用curl命令`curl http://localhost:8000/machine-fingerprint`测试获取机器指纹(若容器启动时未添加环境变量，则获取的为容器的ID而非宿主机)。

使用curl命令`curl http://localhost:8000/validate`测试校验许可证是否有效(许可证机器指纹与宿主机一致，且未过有效期)。

也可以使用curl命令`curl http://localhost:8000/validate?license_key="your-license-key" `从请求中获取并验证License。

## 参考文档