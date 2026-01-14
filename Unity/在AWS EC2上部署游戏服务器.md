# # **Amazon EC2 上部署 C# Console 游戏服务器完整指南（Markdown）**

---

## 🌐 1. 创建 EC2 实例

### 1. 登录 AWS 控制台

进入：**EC2 → Instances → Launch Instance**

### 2. 基础配置

- **OS**：Ubuntu 22.04 LTS
    
- **Instance type**：t3.micro / t3.small / t3.medium（按需求）
    
- **Key pair**：选择创建并下载 `.pem` 文件
    
- **Storage**：建议 20GB SSD
    

### 3. 配置安全组（Security Group）

需要开放：

|用途|端口|Source|
|---|---|---|
|SSH 登录|22|**你的 IP/32** → 安全|
|游戏服务器|**5500**|**0.0.0.0/0** → 全球玩家可连|
|HTTP（可选）|80|0.0.0.0/0|
|HTTPS（可选）|443|0.0.0.0/0|

---

## 🔑 2. SSH 连接 EC2

把 key 文件权限设为安全：

```bash
chmod 400 mykey.pem
```

连接：

```bash
ssh -i mykey.pem ubuntu@你的EC2公网DNS
```

若看到提示：

```
The authenticity of host ... can't be established
```

输入：

```
yes
```

正常现象。

---

## 📦 3. 安装 Git 与 .NET SDK

更新软件源：

```bash
sudo apt update
```

安装 Git：

```bash
sudo apt install git
```

安装 .NET 8 SDK（LTS）：

```bash
sudo apt install dotnet-sdk-8.0
```

验证：

```bash
dotnet --version
```

—

## 📥 4. 拉取你的游戏服务器项目

### 公有仓库：

```bash
git clone https://github.com/yourname/repo.git
```

### 私有仓库（使用 token）：

```bash
git clone https://<token>@github.com/yourname/repo.git
```

---

## ⚙️ 5. 构建 C# Console 游戏服务器（使用 .NET 8）

进入项目：

```bash
cd repo
```

修改 `.csproj`：

```xml
<TargetFramework>net8.0</TargetFramework>
```

然后：

```bash
dotnet clean
dotnet restore
dotnet build -c Release
```

启动测试：

```bash
dotnet bin/Release/net8.0/YourServer.dll
```

如果看到：

```
Server listening on port 5500
```

则说明成功启动。

---

## 🌍 6. 放行游戏服务器端口（关键）

在 EC2 控制台 → Security Group → Inbound rules → Add rule：

```
Custom TCP | 5500 | 0.0.0.0/0
```

如果使用 UDP，也添加 UDP。

---

## 🔎 7. 测试外网是否能连接（用 nc）

Mac 使用：

```bash
nc -vz <EC2公网IP> 5500
```

成功会显示：

```
Connection to <IP> port 5500 succeeded!
```

表示玩家可以连你的游戏服务器。

---

## 🧱 8. 持久运行（服务器不会因 SSH 退出而停止）

## ⭐ 推荐方式：systemd（专业、稳定）

创建服务：

```bash
sudo nano /etc/systemd/system/gameserver.service
```

内容：

```
[Unit]
Description=C# Game Server
After=network.target

[Service]
WorkingDirectory=/home/ubuntu/repo
ExecStart=/usr/bin/dotnet /home/ubuntu/repo/bin/Release/net8.0/YourServer.dll
Restart=always
RestartSec=5
User=ubuntu

[Install]
WantedBy=multi-user.target
```

载入服务：

```bash
sudo systemctl daemon-reload
sudo systemctl start gameserver
sudo systemctl enable gameserver
```

查看状态：

```bash
systemctl status gameserver
```

日志：

```bash
journalctl -u gameserver -f
```

---

## 🧪 9. Unity 客户端连接服务器

```csharp
client.Connect("你的EC2公网IP", 5500);
```

若能成功连接 → 部署完成 🎉

---

## 🛠 10. nano 使用方法（编辑文件时）

|操作|快捷键|
|---|---|
|保存|**Ctrl + O** → Enter|
|退出|**Ctrl + X**|

---

# 🟥 你实际遇到的问题 + 原因 + 解决方法（完整整理）

---

## **❌ 问题 1：SSH 连接超时**

错误：

```
ssh: connect to host ... port 22: Operation timed out
```

### ✔ 原因

你将 SSH 限制成 “My IP”，但你的公网 IP 变了。  
AWS 安全组只允许旧 IP 连接，因此被防火墙挡掉。

### ✔ 解决方法

临时改为：

```
SSH | 22 | 0.0.0.0/0
```

连上后再改回 “My IP”。

---

## **❌ 问题 2：.NET 8 不能运行 .NET 7 程序**

错误：

```
You must install or update .NET to run this application.
Framework: 7.0.0
Found: 8.0.22
```

### ✔ 原因

.NET **不向下兼容**。  
.NET 8 不能运行 .NET 7 的 DLL。

### ✔ 解决方法

将项目从：

```
net7.0
```

升级到：

```
net8.0
```

然后重新构建、部署。

---

## **❌ 问题 3：无法安装 dotnet-runtime-7.0**

错误：

```
Some packages could not be installed...
libicu not installable
```

### ✔ 原因

.NET 7 在 Ubuntu 22.04 的 apt 源被移除，依赖不再可用。

### ✔ 解决方法

你选择了更好的方案：  
**升级项目到 .NET 8**  
（不再需要安装 .NET 7 runtime）

---

## **❌ 问题 4：Mac 上找不到 telnet**

错误：

```
zsh: command not found: telnet
```

### ✔ 原因

macOS 10.13 后移除了 telnet。

### ✔ 解决方法（最推荐）

使用 Mac 自带的：

```
nc -vz <IP> <port>
```

例如：

```
nc -vz 51.20.35.52 5500
```

---

## **❌ 问题 5：不知道 nano 如何保存**

### ✔ 解决方法

- 保存：`Ctrl + O`
    
- 退出：`Ctrl + X`
    

---

#  🎉 总结：你现在已经掌握的能力

你已经会：

- 创建并 SSH 登录 EC2
    
- 安装 Git / .NET / 构建服务器
    
- 使用 .NET 8 部署 C# Console 游戏服务器
    
- 打开安全组端口
    
- 使用 nc 测试服务器连通性
    
- 使用 systemd 让服务器永远运行
    
- 让 Unity 客户端连接 EC2
    

这是一套 **完整的线上游戏服务器部署能力**，已经是专业程序员级别。