---
title: 加密算法在JWT、SSH中的应用
date: 2024-08-27 19:44:50
categories:
- Cyber-Security
tags:
- JWT
- SSH
---

对称加密、非对称加密算法在不同场景中的应用，如使用JWT进行单点登录、使用SSH进行远程免密登录等。

<!--more-->


## 加密算法

### 对称加密

对称加密是一种加密和解密所使用的密钥是相同的加密算法。在对称加密中，发送方和接收方使用相同的密钥对数据进行加密和解密。常见的对称加密算法包括DES、3DES、AES等。

对称加密的优势在于速度快，加解密过程简单，适合用于传输大量数据。但是对称加密无法提供身份验证和数据完整性保护，即无法判断消息的发送方是否可信以及数据是否被篡改。

对称加密通常与其他的加密技术(如数字签名、消息认证码等)结合使用，以提供更高的安全性保护。

#### AES

高级加密标准(Advanced Encryption Standard, 简称AES)为最常见的对称加密算法，在密码学中又称Rijndael加密法。AES算是Rijndael算法的一种特殊实现，选的分组为128bit（16字节），密钥可以使用128、192和256bit三种，而Rijndael使用的密钥和区块长度可以是32位的整数倍，以128位为下限，256比特为上限。加密过程中使用的密钥是由Rijndael密钥生成方案产生。

在AES加密中，密钥(key)、初始化向量(Initialization Vector, 简称IV)是两个重要的组成部分。此外，还可设置加密方式(Mode, 默认为CBC)和填充方式(Padding, 默认为PKCS7)，其中CBC模式需要有IV，ECB模式不需要IV。

- key
    - 定义：key是用于AES加密和解密的密钥，它是双方协商或生成的秘密信息，用于确保加密数据的安全性。
    - 长度：AES支持多种长度的密钥，包括128位（16字节）、192位（24字节）和256位（32字节）。选择哪种长度的密钥取决于安全需求和性能考虑。
    - 重要性：密钥的保密性至关重要，如果密钥被泄露，加密的数据就可能被解密，从而失去保护。

- iv
    - 定义：iv是一个随机或伪随机的值，它与密钥一起用于加密过程中的初始化。它确保了即使使用相同的密钥加密相同的明文，每次加密得到的密文也会不同，从而增强了加密的安全性。
    - 长度：AES的iv长度通常是128位（16字节），这与AES的数据块长度相同。
    - 重要性：iv的随机性对于防止重放攻击和统计分析攻击至关重要。使用固定的iv或可预测的iv会降低加密的安全性。

在AES加密过程中，首先会将明文分成多个128位的数据块（如果最后一个数据块不足128位，则需要进行填充，常用的填充方式包括PKCS5Padding、PKCS7Padding和ZeroPadding等）。然后，使用密钥和iv对数据进行加密。加密过程通常包括多个轮次，每轮都会执行一系列复杂的操作，如字节替换、行移位、列混合和轮密钥加等。最终，所有的数据块都被加密成密文。

以下是c#中AES加密与解密的实现方式，

```c#
/// <summary>
/// ASE加密
/// </summary>
public static string AESEncrypt(string plainText, string key, string iv)
{
    byte[] keyBytes = Encoding.UTF8.GetBytes(key);
    byte[] ivBytes = Encoding.UTF8.GetBytes(iv);
    using (Aes aesAlg = Aes.Create())
    {
        aesAlg.Key = keyBytes;
        aesAlg.IV = ivBytes;
        aesAlg.Mode = CipherMode.CBC;
        aesAlg.Padding = PaddingMode.PKCS7;

        ICryptoTransform encryptor = aesAlg.CreateEncryptor(aesAlg.Key, aesAlg.IV);

        using (MemoryStream ms = new MemoryStream())
        {
            using (CryptoStream cs = new CryptoStream(ms, encryptor, CryptoStreamMode.Write))
            {
                using (StreamWriter sw = new StreamWriter(cs))
                {
                    sw.Write(plainText);
                }
                return Convert.ToBase64String(ms.ToArray());
            }
        }
    }
}

/// <summary>
/// ASE解密
/// </summary>
public static string AESDecrypt(string cipherText, string key, string iv)
{
    byte[] keyBytes = Encoding.UTF8.GetBytes(key);
    byte[] ivBytes = Encoding.UTF8.GetBytes(iv);
    byte[] cipherBytes = Convert.FromBase64String(cipherText);

    using (Aes aesAlg = Aes.Create())
    {
        aesAlg.Key = keyBytes;
        aesAlg.IV = ivBytes;
        aesAlg.Mode = CipherMode.CBC;
        aesAlg.Padding = PaddingMode.PKCS7;

        ICryptoTransform decryptor = aesAlg.CreateDecryptor(aesAlg.Key, aesAlg.IV);

        using (MemoryStream ms = new MemoryStream(cipherBytes))
        {
            using (CryptoStream cs = new CryptoStream(ms, decryptor, CryptoStreamMode.Read))
            {
                using (StreamReader sr = new StreamReader(cs))
                {
                    return sr.ReadToEnd();
                }
            }
        }
    }
}
```

### 非对称加密

非对称加密是一种加密算法，与对称加密不同，非对称加密使用一对不同的密钥来进行加密和解密。这对密钥中的一个被称为私钥(private key)，另一个被称为公钥(public key)。私钥只能由密钥的拥有者持有并保密，不对外公开，而公钥可以向任何人公开。常见的非对称加密算法包括RSA、ECC等。

在非对称加密中，加密和解密过程如下：
    
 1. 发送方使用接收方的公钥对明文进行加密，生成密文
 2. 接收方使用自己的私钥对密文进行解密，恢复为明文

非对称加密算法的特点是安全性高，能够提供身份验证和数据完整性保护。但是非对称加密的缺点是速度较慢，加解密过程相对复杂，适合处理少量的数据。

因此，通常会将对称加密和非对称加密相结合，在传输数据时使用非对称加密来交换对称加密所需的密钥，然后使用对称加密算法来加密和解密实际的数据。

## 消息认证码

消息认证码(Message Authentication Code, 简称MAC)是一种确认完整性并进行认证的技术。

消息认证码的输入包括任意长度的消息和一个发送者与接收者之间共享的密钥，它可以输出固定长度的数据，这个数据称为MAC值。要计算MAC值必须持有共享密钥，消息认证码正是利用这一性质来完成身份认证的。此外，消息认证码通过类似单向散列函数的散列值来确保数据完整性。

HMAC是一种使用单向散列函数来构造消息认证码的方法。HMAC中所使用的单向散列函数并不仅限于一种，任何高强度的单向散列函数都可以被用于HMAC，如果将来设计出新的单向散列函数，也同样可以使用。使用SHA-256、MD5、RIPEMD-160所构造的HMAC，分别称为HMAC-SHA-256、HMAC-MD5和HMAC-RlPEMD。

消息认证码中，由于发送者和接收者共享相同的密钥，因此会产生无法对第三方证明以及无法防止否认等问题。

## 数字签名

数字签名(又称公钥数字签名)是只有信息的发送者才能产生的别人无法伪造的一段数字串。数字签名中也同样会使用公钥和私钥组成的密钥对，不过这两个密钥的用法和非对称加密是相反的，即用私钥加密相当于生成签名，而用公钥解密则相当于验证签名。通常情况下，为了提高传输效率，不会直接对原始数据进行数字签名，而是对原始数据的Hash值进行签名。

数字签名的过程如下：

1. 生成签名：

    - 对原始数据进行哈希运算(使用预先约定的哈希算法)，得到Hash值

    - 使用非对称加密的私钥对Hash值加密，得到签名
 
    - 发送原始数据及签名

2. 验证签名：

    - 接收原始数据及签名

    - 对数字签名使用公钥解密, 得到Hash值

    - 对原始数据进行哈希运算得到新的Hash值，如果两者一致，则签名验证成功；如果两者不一致，则签名验证失败

## JWT

JWT(JSON Web Token)是目前最流行的跨域身份验证解决方案，适用于身份鉴权、授权、信息交换、单点登录等场景。JWT可以使用密钥(使用HMAC算法)或使用RSA或ECDSA的公钥/私钥对进行签名。

应用程序获取JWT并用于访问API等资源的过程如下：

1. 客户端向授权服务器请求授权
2. 授权服务器校验用户身份，如果校验成功，返回访问令牌(token)
3. 应用程序使用访问令牌访问受保护的资源，服务端通过验证JWT的签名来确认用户的身份，通过解析JWT中的声明信息判断用户是否有权限执行特定的操作或访问特定的资源

JWT由Header、Payload、Signature三部分组成：

- JWT的头部通常由两部分组成，分别是令牌类型(typ)和加密算法(alg)。一般情况下，头部会采用Base64编码。

    ```json
    {
        "alg": "HS256",
        "typ": "JWT"
    }
    ```
- JWT的载荷也称为声明信息(claims)，包含了一些有关实体(通常是用户)的信息以及其他元数据。通常包含预定义的字段，如iss(发行者)、sub(主题)、aud(受众)、exp(过期时间)、nbf(生效时间)、iat(发布时间)和jti(JWT ID)等，以及自定义字段。

    ```json
    {
        "sub": "1234567890",
        "jti": "5a3cd526-caa4-4952-a5ba-f44245fe1762",
        "iat": 1681102951,
        "nbf": 1681102951,
        "exp": 1681189351,
        "iss": "Jocoboy",
        "aud": "WebUser",
        "admin": true
    }
    ```

- JWT的签名是由头部、载荷和密钥(通常为32个字节)共同生成的，用于验证JWT的真实性和完整性。使用Header里面指定的签名算法(默认是HMAC-SHA256)，按照下面的公式产生签名

    ```c#
    HMACSHA256(
    base64UrlEncode(header) + "." +
    base64UrlEncode(payload),
    secret)
    ```

.NET使用HMAC-SHA256对称加密算法生成JWT的部分代码如下
```c#
    ...
    var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_authOptions.Secret));

    var credentials = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

    var token = new JwtSecurityToken(_authOptions.Issuer,
        _authOptions.Audience,
        claims,
        DateTime.Now,
        DateTime.Now.AddMinutes(_authOptions.Expiration),
        credentials);
    return new JwtSecurityTokenHandler().WriteToken(token);
    ...
```

## SSH

安全外壳协议(Secure Shell, 简称SSH)是一种建立在应用层基础上，在不安全网络上用于安全远程登录和其他安全网络服务的协议。

SSH建立在非对称加密之上，建立远程连接的过程如下：

1. 远程主机(虚拟机)收到本地主机的登录请求，把自己的公钥发给本地主机
2. 本地主机使用这个公钥，将登录密码加密后，发送给远程主机
3. 远程主机用自己的私钥，解密登录密码，如果密码正确，则同意本地主机登录

### 口令登录

Linux系统中以用户名username，登录远程主机remote_host的命令如下

`ssh username@remote_host`

初次连接会提示公钥指纹，确认指纹无误后输入密码，远程主机的公钥就会自动保存到本地主机的.ssh/known_hosts文件中。

### 公钥登录

公钥登录可以省去输入密码的步骤。用户将自己的公钥储存在远程主机上，登录时远程主机会向用户发送一段随机字符串，用户用自己的私钥加密后再发回来。远程主机用事先储存的公钥进行解密，如果成功，就证明用户是可信的，直接允许登录。

在本地主机生成一对公钥和私钥，然后将公钥添加到远程主机的 ~/.ssh/authorized_keys文件中(Windows系统中把.ssh/id_rsa.pub的内容复制出来，手动追加即可)

`ssh-keygen -t rsa -b 4096 -C "your_email@example.com"`

`ssh-copy-id username@remote_host`

## 参考文档

- [.NET中的AES类](https://learn.microsoft.com/zh-cn/dotnet/api/system.security.cryptography.aes)

- [JWT官方文档](https://jwt.io/introduction)

- [Generating new SSH key](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent)
