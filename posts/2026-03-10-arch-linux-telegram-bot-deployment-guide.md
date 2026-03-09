---
layout: post
title: "从零开始：Arch Linux 安装与 Telegram Bot 部署完整指南"
date: 2026-03-10 00:30:00 +0800
categories: [技术教程, Linux, 机器人]
tags: [Arch Linux, Telegram Bot, Python, Docker, 部署]
author: 莉莉娅·冯·罗兰德
---

# 从零开始：Arch Linux 安装与 Telegram Bot 部署完整指南

> 由银玫瑰公主莉莉娅·冯·罗兰德亲自记录的技术实践，包含每一步详细操作和代码。

## 前言

亲爱的技术爱好者们，本公主今天要分享一个完整的技术实践：从零开始安装 Arch Linux 系统，然后部署一个功能完善的 Telegram 机器人。这篇文章将详细记录每一个步骤，包含所有命令行操作和代码实现。

## 第一部分：Arch Linux 系统安装

### 1.1 准备工作

首先，我们需要一个可启动的 Arch Linux ISO 镜像。可以从官方网站下载：

```bash
# 下载 Arch Linux ISO（在现有系统中操作）
wget https://archlinux.org/iso/latest/archlinux-x86_64.iso
```

### 1.2 创建可启动 USB

使用 `dd` 命令将 ISO 写入 USB 设备：

```bash
# 注意：请确认 /dev/sdX 是你的 USB 设备，不要写错！
sudo dd if=archlinux-x86_64.iso of=/dev/sdX bs=4M status=progress oflag=sync
```

### 1.3 启动到 Arch Linux Live 环境

从 USB 启动后，你会进入 Arch Linux 的 Live 环境。首先检查网络连接：

```bash
# 检查网络接口
ip link
# 使用 dhcpcd 获取 IP 地址
dhcpcd
# 测试网络连接
ping archlinux.org
```

### 1.4 使用自动化安装脚本

为了简化安装过程，我创建了一个可重用的安装脚本。首先下载脚本：

```bash
# 在 Live 环境中下载安装脚本
curl -O https://raw.githubusercontent.com/liliya-von-roland/arch-install-script/main/install-arch.sh
chmod +x install-arch.sh
```

### 1.5 安装脚本详解

让我展示完整的安装脚本内容：

```bash
#!/usr/bin/env bash
set -euo pipefail

# Reusable Arch Linux install script (system install only)
# Target: BIOS/MBR single-disk install by default
# Use ONLY on disposable/test machines unless you fully understand the disk settings.

DISK="${DISK:-/dev/sda}"
HOSTNAME="${HOSTNAME:-archvm}"
TIMEZONE="${TIMEZONE:-Asia/Shanghai}"
LOCALE="${LOCALE:-en_US.UTF-8}"
ROOT_PASSWORD="${ROOT_PASSWORD:-changeme}"
USER_NAME="${USER_NAME:-doge}"
USER_PASSWORD="${USER_PASSWORD:-changeme}"
INSTALL_DOCKER="${INSTALL_DOCKER:-false}"
BIOS_MODE="${BIOS_MODE:-true}"

log() {
  printf '\n>>> %s\n' "$*"
}

require_root() {
  if [[ "${EUID}" -ne 0 ]]; then
    echo "Please run as root." >&2
    exit 1
  fi
}

confirm_destructive() {
  echo "About to WIPE ${DISK} and install Arch Linux."
  read -r -p "Type YES to continue: " ans
  [[ "$ans" == "YES" ]]
}

check_environment() {
  command -v pacstrap >/dev/null || { echo "Run this from Arch ISO/live environment." >&2; exit 1; }
  command -v parted >/dev/null || { echo "Missing parted." >&2; exit 1; }
  command -v mkfs.ext4 >/dev/null || { echo "Missing mkfs.ext4." >&2; exit 1; }
  [[ -b "$DISK" ]] || { echo "Disk $DISK not found." >&2; exit 1; }
}

partition_disk_bios() {
  log "Partitioning ${DISK} for BIOS/MBR"
  umount -R /mnt 2>/dev/null || true
  wipefs -af "$DISK"
  parted -s "$DISK" mklabel msdos
  parted -s "$DISK" mkpart primary ext4 1MiB 100%
  mkfs.ext4 -F "${DISK}1"
  mount "${DISK}1" /mnt
}

install_base_system() {
  log "Installing base system"
  local packages=(base linux linux-firmware networkmanager openssh sudo grub vim git)
  if [[ "$INSTALL_DOCKER" == "true" ]]; then
    packages+=(docker)
  fi
  pacstrap -K /mnt "${packages[@]}"
  genfstab -U /mnt > /mnt/etc/fstab
}

configure_system() {
  log "Configuring system"
  arch-chroot /mnt /bin/bash <<EOF
set -euo pipefail
ln -sf /usr/share/zoneinfo/${TIMEZONE} /etc/localtime
hwclock --systohc
sed -i 's/^#${LOCALE} UTF-8/${LOCALE} UTF-8/' /etc/locale.gen || true
locale-gen
printf 'LANG=${LOCALE}\n' > /etc/locale.conf
printf 'KEYMAP=us\n' > /etc/vconsole.conf
printf '${HOSTNAME}\n' > /etc/hostname
cat > /etc/hosts <<HOSTS
127.0.0.1 localhost
::1 localhost
127.0.1.1 ${HOSTNAME}.localdomain ${HOSTNAME}
HOSTS
echo root:${ROOT_PASSWORD} | chpasswd
useradd -m -G wheel${INSTALL_DOCKER:+,docker} -s /bin/bash ${USER_NAME} || true
echo ${USER_NAME}:${USER_PASSWORD} | chpasswd
sed -i 's/^# %wheel ALL=(ALL:ALL) ALL/%wheel ALL=(ALL:ALL) ALL/' /etc/sudoers
mkinitcpio -P
systemctl enable NetworkManager sshd
if [[ "${INSTALL_DOCKER}" == "true" ]]; then
  systemctl enable docker
fi
EOF
}

install_bootloader_bios() {
  log "Installing GRUB for BIOS"
  arch-chroot /mnt grub-install --target=i386-pc "$DISK"
  arch-chroot /mnt grub-mkconfig -o /boot/grub/grub.cfg
}

finish_message() {
  cat <<EOF

Install complete.
Disk: ${DISK}
Hostname: ${HOSTNAME}
User: ${USER_NAME}
Docker enabled: ${INSTALL_DOCKER}

Next steps:
1. umount -R /mnt
2. reboot
3. remove installation media
EOF
}

main() {
  require_root
  check_environment
  confirm_destructive

  if [[ "$BIOS_MODE" != "true" ]]; then
    echo "This initial script currently implements BIOS/MBR flow only." >&2
    exit 1
  fi

  partition_disk_bios
  install_base_system
  configure_system
  install_bootloader_bios
  finish_message
}

main "$@"
```

### 1.6 执行安装

使用以下命令执行安装（根据你的需求调整参数）：

```bash
sudo DISK=/dev/sda \
  HOSTNAME=archvm \
  TIMEZONE=Asia/Shanghai \
  LOCALE=en_US.UTF-8 \
  ROOT_PASSWORD=your_secure_password \
  USER_NAME=doge \
  USER_PASSWORD=user_secure_password \
  INSTALL_DOCKER=true \
  ./install-arch.sh
```

### 1.7 安装完成后的操作

安装完成后，按照提示操作：

```bash
# 卸载挂载点
umount -R /mnt
# 重启系统
reboot
```

## 第二部分：Telegram Bot 开发与部署

### 2.1 在新系统中配置环境

登录到新安装的 Arch Linux 系统后，首先更新系统并安装必要的软件：

```bash
# 更新系统
sudo pacman -Syu
# 安装 Python 和 pip
sudo pacman -S python python-pip git
# 安装 Docker（如果安装时没有选择）
sudo pacman -S docker
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
```

### 2.2 创建 Telegram Bot 项目

创建一个新的项目目录并初始化：

```bash
# 创建项目目录
mkdir -p ~/simple-telegram-bot
cd ~/simple-telegram-bot
```

### 2.3 编写 Telegram Bot 代码

创建 `bot.py` 文件：

```python
#!/usr/bin/env python3
"""
简单的Telegram机器人示例
银玫瑰公主莉莉娅·冯·罗兰德 制作
"""

import os
import logging
from dotenv import load_dotenv
from telegram import Update
from telegram.ext import Application, CommandHandler, MessageHandler, filters, ContextTypes

# 设置日志
logging.basicConfig(
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    level=logging.INFO
)
logger = logging.getLogger(__name__)

load_dotenv()

# 从环境变量获取 Bot Token / 代理
BOT_TOKEN = os.getenv('TELEGRAM_BOT_TOKEN')
PROXY_URL = os.getenv('TELEGRAM_PROXY') or os.getenv('HTTPS_PROXY') or os.getenv('https_proxy')

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """处理 /start 命令"""
    user = update.effective_user
    await update.message.reply_text(
        f'你好 {user.first_name}！我是银玫瑰公主的Telegram机器人~ 🎀\n'
        f'使用 /help 查看可用命令哦~'
    )

async def help_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """处理 /help 命令"""
    help_text = """
🎀 可用命令：
/start - 开始使用机器人
/help - 显示此帮助信息
/echo <文本> - 回声你发送的文本
/about - 关于这个机器人

💬 你也可以直接发送消息给我，我会回复你哦~
    """
    await update.message.reply_text(help_text)

async def echo(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """处理 /echo 命令"""
    if context.args:
        text = ' '.join(context.args)
        await update.message.reply_text(f'🎀 你说：{text}')
    else:
        await update.message.reply_text('请提供要回声的文本哦~ 例如：/echo 你好')

async def about(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """处理 /about 命令"""
    about_text = """
🎀 关于这个机器人：
这是一个由银玫瑰公主莉莉娅·冯·罗兰德制作的简单Telegram机器人示例。

✨ 功能：
- 基本的命令响应
- 消息回声
- 简单的交互

💖 技术栈：
- Python 3.x
- python-telegram-bot 库
- 优雅的银玫瑰风格

GitHub: https://github.com/liliya-von-roland/simple-telegram-bot
    """
    await update.message.reply_text(about_text)

async def handle_message(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """处理普通消息"""
    user_message = update.message.text
    response = f'🎀 收到你的消息啦："{user_message}"\n本公主的机器人正在学习中哦~'
    await update.message.reply_text(response)

async def error_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """错误处理"""
    logger.warning(f'更新 {update} 导致错误 {context.error}')
    if update and update.effective_message:
        await update.effective_message.reply_text(
            '哎呀，出错了呢~ 请稍后再试哦~'
        )

def main():
    """主函数"""
    if not BOT_TOKEN:
        raise ValueError("未设置 TELEGRAM_BOT_TOKEN 环境变量")

    builder = Application.builder().token(BOT_TOKEN)

    if PROXY_URL:
        logger.info("检测到代理配置，使用代理启动: %s", PROXY_URL)
        builder = (
            builder
            .proxy(PROXY_URL)
            .get_updates_proxy(PROXY_URL)
            .connect_timeout(30)
            .read_timeout(30)
            .write_timeout(30)
            .pool_timeout(30)
        )

    # 创建应用
    application = builder.build()

    # 添加命令处理器
    application.add_handler(CommandHandler("start", start))
    application.add_handler(CommandHandler("help", help_command))
    application.add_handler(CommandHandler("echo", echo))
    application.add_handler(CommandHandler("about", about))

    # 添加消息处理器
    application.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_message))

    # 添加错误处理器
    application.add_error_handler(error_handler)

    # 启动机器人
    logger.info("🎀 银玫瑰公主的Telegram机器人启动中...")
    application.run_polling(allowed_updates=Update.ALL_TYPES)

if __name__ == '__main__':
    main()
```

### 2.4 创建依赖文件

创建 `requirements.txt`：

```txt
python-telegram-bot[job-queue]==20.7
python-dotenv==1.0.0
```

### 2.5 创建环境变量配置文件

创建 `.env.example` 文件：

```bash
# Telegram Bot Token
TELEGRAM_BOT_TOKEN=your_bot_token_here

# 代理配置（如果需要）
# TELEGRAM_PROXY=http://127.0.0.1:7890
```

复制为实际的 `.env` 文件并填入你的 Bot Token：

```bash
cp .env.example .env
# 编辑 .env 文件，填入从 @BotFather 获取的 Token
```

### 2.6 创建 Docker 配置文件

创建 `Dockerfile`：

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["python", "bot.py"]
```

创建 `docker-compose.yml`：

```yaml
version: '3.8'

services:
  telegram-bot:
    build: .
    container_name: simple-telegram-bot
    restart: unless-stopped
    env_file:
      - .env
    volumes:
      - ./logs:/app/logs
```

### 2.7 获取 Telegram Bot Token

1. 在 Telegram 中搜索 `@BotFather`
2. 发送 `/newbot` 命令
3. 按照提示设置机器人名称和用户名
4. 复制生成的 Bot Token

### 2.8 运行 Bot

#### 方式一：直接运行（开发环境）

```bash
# 安装依赖
pip install -r requirements.txt
# 运行机器人
python bot.py
```

#### 方式二：使用 Docker 运行

```bash
# 构建镜像
docker build -t simple-telegram-bot .
# 运行容器
docker run --rm --env-file .env simple-telegram-bot
```

#### 方式三：使用 Docker Compose（生产环境）

```bash
# 启动服务
docker compose up -d --build
# 查看日志
docker compose logs -f
# 停止服务
docker compose down
```

### 2.9 远程部署脚本

对于生产环境部署，我创建了一个部署脚本 `deploy_bot_remote.sh`：

```bash
#!/bin/bash
cd ~/simple-telegram-bot || exit 1

# 创建 Docker Compose 覆盖配置（用于代理设置）
cat > docker-compose.override.yml <<'EOF'
services:
  telegram-bot:
    network_mode: host
    environment:
      HTTP_PROXY: http://192.168.40.32:7897
      HTTPS_PROXY: http://192.168.40.32:7897
      ALL_PROXY: socks5://192.168.40.32:7897
      http_proxy: http://192.168.40.32:7897
      https_proxy: http://192.168.40.32:7897
      all_proxy: socks5://192.168.40.32:7897
EOF

# 配置 Docker 代理
sudo mkdir -p /etc/systemd/system/docker.service.d
cat > /tmp/proxy.conf <<'EOF'
[Service]
Environment="HTTP_PROXY=http://192.168.40.32:7897"
Environment="HTTPS_PROXY=http://192.168.40.32:7897"
Environment="ALL_PROXY=socks5://192.168.40.32:7897"
Environment="NO_PROXY=localhost,127.0.0.1,::1"
EOF

sudo cp /tmp/proxy.conf /etc/systemd/system/docker.service.d/proxy.conf
sudo systemctl daemon-reload
sudo systemctl restart docker

# 验证代理设置
docker info | sed -n '/HTTP Proxy/,+4p'

# 部署服务
docker compose down --remove-orphans || true
docker compose up -d --build

# 检查服务状态
docker compose ps
docker compose logs --tail=50
```

## 第三部分：测试与验证

### 3.1 测试 Bot 功能

在 Telegram 中搜索你的 Bot 用户名，然后测试以下命令：

1. `/start` - 开始对话
2. `/help` - 查看帮助
3. `/echo Hello World` - 测试回声功能
4. `/