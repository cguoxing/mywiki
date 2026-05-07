# 飞牛 NAS（fnOS）使用教程

> 更新时间：2026年5月
> 适用型号：飞牛私有云（fnOS）· 1H-ARCH 自用部署

---

## 一、系统概述

飞牛 NAS（品牌名：飞牛私有云 fnOS）是国产轻量级 NAS 操作系统，基于 Debian 开发，界面友好，适合中小团队私有部署。

**本部署环境：**

| 项目 | 配置 |
|------|------|
| 主机名 | 1H-AT |
| 内网 IP | 192.168.31.187 |
| ZeroTier IP | 10.166.37.164 |
| 系统盘 | 98GB |
| 数据盘 | 1TB + 4.3TB（外挂视频区） |
| Ollama 端口 | 11434 |

---

## 二、系统初始化

### 2.1 初始登录

1. 浏览器访问 `http://192.168.31.187` 或飞牛官方域名（需内网穿透）
2. 首次登录使用管理员账户，默认 `admin`
3. 首次使用需设置强密码，建议使用密码管理器保存

### 2.2 创建普通用户

```bash
# 通过 SSH 登录创建用户（可选）
ssh admin@192.168.31.187
# 在 Web 界面：控制面板 → 用户 → 添加用户
```

### 2.3 存储池配置

1. 进入「存储管理」
2. 选择硬盘，创建存储池（推荐 RAID1 或 SHR 冗余模式）
3. 创建共享文件夹：项目备份、媒体资源、个人资料等

---

## 三、文件共享（SMB/CIFS）

### 3.1 启用 SMB 服务

1. 进入「控制面板」→「文件服务」
2. 开启 **SMB/CIFS**
3. 选择工作组为 `WORKGROUP`（默认）

### 3.2 创建共享文件夹

1. 「存储管理」→「共享文件夹」→ 新建
2. 设置名称、描述、存储路径
3. 配置访问权限（读/写/管理员）

### 3.3 Windows 访问

```powershell
# 方式一：资源管理器直接访问
\\192.168.31.187\sharename

# 方式二：映射网络驱动器
net use Z: \\192.168.31.187\sharename /user:harry 1Hdesign

# 方式三：快速访问（永久映射）
net use Z: \\192.168.31.187\sharename /persistent:yes
```

### 3.4 macOS 访问

```bash
# Finder → 前往 → 连接服务器
smb://192.168.31.187

# 命令行挂载
mkdir -p /Volumes/mynas
mount -t smbfs //harry:1Hdesign@192.168.31.187/sharename /Volumes/mynas

# 开机自动挂载（/etc/fstab）
//harry:1Hdesign@192.168.31.187/sharename /Volumes/mynas smbfs rw,auto 0 0
```

### 3.5 Linux 访问

```bash
# 安装 smbclient
sudo apt install smbclient cifs-utils

# 命令行浏览
smbclient -U harry //192.168.31.187/sharename

# 挂载到本地目录
sudo mount -t cifs //192.168.31.187/sharename /mnt/nas -o user=harry,pass=1Hdesign

# fstab 自动挂载
//192.168.31.187/sharename /mnt/nas cifs username=harry,password=1Hdesign,iocharset=utf8 0 0
```

---

## 四、Ollama AI 服务

### 4.1 安装 Ollama

飞牛 NAS 支持 Docker，可在 Docker 中运行 Ollama：

```bash
# SSH 登录飞牛 NAS
ssh admin@192.168.31.187

# 方式一：通过 Docker 安装
docker run -d \
  --name ollama \
  -p 11434:11434 \
  -v /vol1/ollama:/root/.ollama \
  -restart unless-stopped \
  ollama/ollama

# 方式二：原生安装（已有安装）
/opt/ollama/bin/ollama serve
```

### 4.2 开放外网访问（重要）

默认 Ollama 只监听 `127.0.0.1`，需要改为 `0.0.0.0`：

```bash
# 停止旧进程
sudo pkill -f ollama

# 使用环境变量启动（监听所有接口）
OLLAMA_HOST=0.0.0.0:11434 /opt/ollama/bin/ollama serve &

# 验证是否监听所有接口
curl http://192.168.31.187:11434/api/tags
```

### 4.3 拉取模型

```bash
# 查看可用模型
ollama list

# 拉取模型
ollama pull qwen2.5:3b
ollama pull llama3.2:latest
ollama pull gemma3:4b-it-q4_K_M

# 删除模型
ollama rm modelname:tag
```

### 4.4 API 调用

```bash
# 基础对话
curl http://192.168.31.187:11434/api/generate -d '{
  "model": "qwen2.5:3b",
  "prompt": "你好",
  "stream": false
}'

# 非流式响应（适合程序调用）
curl http://192.168.31.187:11434/api/generate -d '{
  "model": "qwen2.5:3b",
  "prompt": "用一句话介绍飞牛NAS",
  "stream": false,
  "options": {
    "temperature": 0.7,
    "num_predict": 200
  }
}'

# 查看已安装模型列表
curl http://192.168.31.187:11434/api/tags
```

### 4.5 开机自启动（systemd）

```ini
[Unit]
Description=Ollama Service
After=network.target

[Service]
Type=simple
ExecStart=/opt/ollama/bin/ollama serve
Environment="OLLAMA_HOST=0.0.0.0:11434"
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

```bash
# 部署服务
sudo mv ollama.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable ollama
sudo systemctl start ollama
```

---

## 五、ZeroTier 内网穿透

### 5.1 为什么需要 ZeroTier

飞牛 NAS 位于内网（192.168.31.187），外出时无法直接访问。ZeroTier 通过 P2P VPN 技术，将所有设备组建成虚拟局域网。

### 5.2 注册 ZeroTier

1. 访问 [https://my.zerotier.com](https://my.zerotier.com)
2. 注册账号（建议用 Google/GitHub 登录）
3. 创建网络，记下 **Network ID**

### 5.3 NAS 端安装

```bash
# SSH 登录 NAS
ssh admin@192.168.31.187

# 下载并安装 ZeroTier
curl -s https://install.zerotier.com | sudo bash

# 加入网络（替换 YOUR_NETWORK_ID 为实际 Network ID）
sudo zerotier-cli join YOUR_NETWORK_ID

# 在 ZeroTier 网页后台审批设备
# 分配固定 IP（如 10.166.37.164）
```

### 5.4 验证连接

```bash
# 查看 ZeroTier 状态
sudo zerotier-cli status
sudo zerotier-cli listnetworks

# 从外网测试访问
ping 10.166.37.164
curl http://10.166.37.164:11434/api/tags
```

### 5.5 多设备组网（1H-ARCH 当前配置）

| 设备 | ZeroTier IP | 内网 IP |
|------|-------------|---------|
| 飞牛 NAS | 10.166.37.164 | 192.168.31.187 |
| 1H-harry（MacBook） | 10.166.37.40 | 动态 |
| Ub-server | 10.166.37.64 | 192.168.31.84 |

---

## 六、Docker 使用

### 6.1 安装 Docker

飞牛 NAS 管理界面支持一键安装 Docker。

### 6.2 常用容器

```bash
# Ollama（AI推理）
docker run -d --name ollama \
  -p 11434:11434 \
  -v /vol1/ollama_data:/root/.ollama \
  --restart unless-stopped \
  ollama/ollama

# Portainer（容器管理界面）
docker run -d --name portainer \
  -p 9000:9000 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /vol1/portainer_data:/data \
  --restart unless-stopped \
  portainer/portainer-ce

# Jellyfin（媒体服务器）
docker run -d --name jellyfin \
  -p 8096:8096 \
  -v /vol1/jellyfin_config:/config \
  -v /vol1/jellyfin_cache:/cache \
  -v /vol1/media:/media \
  --restart unless-stopped \
  jellyfin/jellyfin
```

---

## 七、备份策略

### 7.1 本地备份

```bash
# 定时备份到外挂硬盘
rsync -avz /vol1/important_data/ /vol1/external_backup/ --delete
```

### 7.2 跨设备备份

```bash
# 从 Ub-server 备份到 NAS
rsync -avz -e "ssh" harry@192.168.31.84:/home/harry/backups/ \
  /vol1/ub_backups/
```

### 7.3 快照保护

建议在 fnOS 中启用 Btrfs 快照，定期快照防止误删。

---

## 八、常见问题

### Q1: SMB 访问提示「无权限」

检查以下几点：
1. 用户是否添加到共享权限列表
2. 密码是否正确（SMB 密码独立于系统密码）
3. 是否启用了 SMB 服务
4. Windows 防火墙是否拦截了 445 端口

### Q2: Ollama 无法从外网访问

1. 确认 `OLLAMA_HOST=0.0.0.0:11434` 而非默认的 `127.0.0.1`
2. 检查 NAS 防火墙是否开放 11434 端口
3. ZeroTier 网络是否已审批通过

### Q3: Docker 拉取镜像失败

```bash
# 配置国内镜像加速
sudo mkdir -p /etc/docker/daemon.json
sudo tee /etc/docker/daemon.json << EOF
{
  "registry-mirrors": [
    "https://docker.mirrors.ustc.edu.cn",
    "https://hub-mirror.c.163.com"
  ]
}
EOF
sudo systemctl restart docker
```

### Q4: 如何查看系统资源占用

```bash
# SSH 登录后
htop          # 实时进程和资源
df -h         # 磁盘使用
free -h       # 内存使用
uptime        # 运行时间
```

### Q5: 忘记管理员密码

通过 SSH 使用 `sudo` 重置：
```bash
ssh admin@192.168.31.187
sudo passwd admin
```

---

## 九、安全建议

1. **修改默认端口**：SMB 445 端口暴露，建议配合 VPN 使用
2. **启用防火墙**：只开放必要端口（SSH、WebUI、SMB）
3. **定期更新**：飞牛 NAS 会推送安全更新，及时安装
4. **强密码**：所有账户使用独立强密码，推荐密码管理器
5. **关闭 SSH 密码登录**：使用 SSH Key 认证
6. **ZeroTier 私有网络**：不要泄露 Network ID，只邀请可信设备

---

## 十、快速命令速查

```bash
# SSH 登录
ssh admin@192.168.31.187

# 查看 IP
ip addr show

# 重启服务
sudo systemctl restart smbd
sudo systemctl restart docker

# 查看 Ollama 日志
sudo journalctl -u ollama -f

# ZeroTier
sudo zerotier-cli status
sudo zerotier-cli join NETWORK_ID
sudo zerotier-cli leave NETWORK_ID

# Docker
docker ps -a
docker logs ollama
docker restart ollama
```

---

*本文档由 1H-ARCH 大元Q2 整理 · 2026年5月*
*如有问题，请联系：大鱼（主理人）*
