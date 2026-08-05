# AI 工具在 Termux 环境中的避坑指南

> 从实战经验中总结的 Termux + AI 工具踩坑记录，持续更新。

---

## 目录

1. [Hermes Agent 在 Termux 上运行](#1-hermes-agent-在-termux-上运行)
2. [Python 版本兼容性问题](#2-python-版本兼容性问题)
3. [搜索工具 ddgs 无法安装](#3-搜索工具-ddgs-无法安装)
4. [浏览器自动化限制](#4-浏览器自动化限制)
5. [Telegram Bot 配置](#5-telegram-bot-配置)
6. [GitHub CLI 配置](#6-github-cli-配置)
7. [Termux 上的 AI 编码工具](#7-termux-上的-ai-编码工具)
8. [Android SDK 与构建环境](#8-android-sdk-与构建环境)
9. [聚合新闻 Skills 安装](#9-聚合新闻-skills-安装)
10. [Skills 管理](#10-skills-管理)
11. [当前 AI 工具安装方式一览表](#11-当前-ai-工具安装方式一览表)
12. [搜索 Skills 安装（重要）](#12-搜索-skills-安装重要)

---

## 1. Hermes Agent 在 Termux 上运行

### 问题

Hermes Agent 官方支持 Termux，但有几个已知坑。

### 解决方案

```bash
# 安装依赖
pkg install git python clang rust make pkg-config libffi openssl nodejs ripgrep ffmpeg

# 创建 venv（不要用 python -m venv，改用 uv）
uv venv venv --python python3
source venv/bin/activate
uv pip install -e ".[termux-all]"
```

### ⚠️ 已知警告（可忽略）

以下警告是 Termux 环境正常现象，不影响核心功能：

- `browser-cdp (system dependency not met)` — Termux 无 Chrome，正常
- `browser-dialog (system dependency not met)` — 需要 GUI 桌面
- `computer_use (system dependency not met)` — 需要桌面环境
- `x_search (missing XAI_API_KEY)` — 未配置 xAI
- `No env user allowlists configured` — 已配对用户能用就行

### 隐藏工具不可用警告

```bash
hermes config set display.hide_unavailable_tools true
```

### 重启 Gateway 的正确方式

Hermes 不能自己重启自己（会先杀当前进程），必须在**另一个 Termux 窗口**执行：

```bash
hermes gateway restart
```

---

## 2. Python 版本兼容性问题

### 问题

Hermes Agent 要求 Python `<3.14,>=3.11`，但 Termux 的 `pkg upgrade` 会自动把 Python 升到 3.14+，导致 Hermes 无法启动。

### 解决方案

```bash
# 降级 Python
apt-get install --allow-downgrades -y python=3.13.12-3

# 锁定版本防止自动升级
apt-mark hold python
apt-mark hold python-ensurepip-wheels

# 重建 venv
cd ~/hermes-agent
rm -rf venv
uv venv venv --python python3
source venv/bin/activate
uv pip install -e ".[termux-all]"
```

### ⚠️ 重要提醒

每次 `pkg upgrade` 后都要检查 Python 版本：

```bash
python3 --version
```

如果是 3.14，必须再次降级。

### 自动防护

可以在 `~/.bashrc` 添加：

```bash
alias pkg-upgrade='pkg upgrade -y && apt-get install --allow-downgrades -y python=3.13.12-3 && apt-mark hold python'
```

---

## 3. 搜索工具 ddgs 无法安装

### 问题

`ddgs`（DuckDuckGo 搜索）依赖 `primp`，而 `primp` 是 Rust 写的 Python 扩展。在 Termux ARM64 上编译会卡住或报错。

```
error: failed to run custom build command for `proc-macro2 v1.0.106`
Caused by: could not execute process ... (never executed)
Caused by: Text file busy (os error 26)
```

即使成功安装预编译轮子，也会因为 glibc vs bionic libc 不兼容而无法加载：

```
ImportError: dlopen failed: library "libgcc_s.so.1" not found
```

### 结论

**ddgs 目前无法在 Termux 上使用**，这是 Rust 扩展在 Android 上的已知限制。

### 替代方案

使用 Hermes 内置的 `web_search`（firecrawl 后端）+ `web_extract`：

```
我：搜索 Android 16 新特性
Hermes：调用 web_search → 返回搜索结果

我：打开这个网页看看内容
Hermes：调用 web_extract → 返回网页正文
```

90% 的场景够用。

---

## 4. 浏览器自动化限制

### 问题

Termux 没有 Chrome/Chromium，无法使用 Hermes 的 `browser` 工具（`browser_navigate`、`browser_type` 等）。

### 可用方案对比

| 方案 | 费用 | 说明 |
|------|------|------|
| **内置 web_search + web_extract** | 免费 | 90% 场景够用 |
| **Browser Use 云端** | 付费 | 需要 API key |
| **Termux Browser Pilot** | 免费 | 本地 Firefox/Chromium 自动化 |
| **termux-agent-browser** | 免费 | 专为 Hermes 设计的 CDP 直连 |
| **本地 Chromium + CDP** | 免费 | 需要 proot-distro |

### 推荐方案

先用 `web_search` + `web_extract`，需要真实浏览器渲染时再考虑 Termux Browser Pilot。

---

## 5. Telegram Bot 配置

### 配置步骤

```bash
# 1. 设置 bot token
hermes config set telegram.token "YOUR_BOT_TOKEN"

# 2. 重启 gateway（在另一个窗口）
hermes gateway restart

# 3. 获取配对码，让用户在 Telegram 上发送 /start
hermes pairing list

# 4. 批准配对
hermes pairing approve <PAIRING_CODE>
```

### 关闭 Telegram

```bash
# 编辑 ~/.hermes/config.yaml
# 把 telegram: 下的 enabled: true 改成 enabled: false
python3 -c "
with open('/data/data/com.termux/files/home/.hermes/config.yaml', 'r') as f:
    content = f.read()
content = content.replace('telegram:\n      enabled: true', 'telegram:\n      enabled: false')
with open('/data/data/com.termux/files/home/.hermes/config.yaml', 'w') as f:
    f.write(content)
"
```

---

## 6. GitHub CLI 配置

### 登录

```bash
gh auth login
# 选择 HTTPS → 用 token 登录
```

### 常用命令

```bash
gh search repos "termux" --limit 10
gh search code "hermes agent" --language python
gh repo view owner/repo
gh release download TAG --repo owner/repo --pattern "*.deb"
```

---

## 7. Termux 上的 AI 编码工具

### MiMoCode-Termux（小米 MiMo）

```bash
# 下载最新 deb 包
curl -sL -o ~/mimocode.deb "https://github.com/Hope2333/MiMoCode-Termux/releases/latest/download/mimocode_X.Y.Z_aarch64.deb"

# 安装
cd ~ && ar x mimocode.deb && tar xf data.tar.xz
cp ./data/data/com.termux/files/usr/bin/mimo /data/data/com.termux/files/usr/bin/
cp -r ./data/data/com.termux/files/usr/lib/mimocode /data/data/com.termux/files/usr/lib/
```

### OpenCode-Termux

```bash
# 下载最新 deb 包
curl -sL -o ~/opencode.deb "https://github.com/Hope2333/opencode-termux/releases/latest/download/opencode_X.Y.Z_aarch64.deb"

# 安装
cd ~ && ar x opencode.deb && tar xf data.tar.xz
cp ./data/data/com.termux/files/usr/bin/opencode /data/data/com.termux/files/usr/bin/
cp -r ./data/data/com.termux/files/usr/lib/opencode /data/data/com.termux/files/usr/lib/
```

⚠️ `mimo --version` 显示的 `0.1.0` 是启动器版本号，不是模型版本。实际装的是 MiMo-Code v0.1.9。

---

## 8. Android SDK 与构建环境

### 安装

```bash
pkg install openjdk-21

# 下载 Android SDK Command Line Tools
ANDROID_HOME="$HOME/android-sdk"
mkdir -p "$ANDROID_HOME/cmdline-tools"
curl -fsSL "https://dl.google.com/android/repository/commandlinetools-linux-11076708_latest.zip" -o ~/cmdline-tools.zip
unzip ~/cmdline-tools.zip -d "$ANDROID_HOME/cmdline-tools/"
mv "$ANDROID_HOME/cmdline-tools/cmdline-tools" "$ANDROID_HOME/cmdline-tools/latest"
```

### ⚠️ 关键坑：ARM64 aapt2

Google 的 Android SDK 只提供 x86_64 原生二进制，在 Termux ARM64 上跑不了。

**解决方案**：用 lzhiyong 的 ARM64 预编译版本：

```bash
curl -fsSL "https://github.com/lzhiyong/android-sdk-tools/releases/download/35.0.2/android-sdk-tools-static-aarch64.zip" -o ~/lzhiyong.zip
unzip -o ~/lzhiyong.zip -d ~/lzhiyong-tools

# 替换所有 build-tools 版本
for BT in $ANDROID_HOME/build-tools/*/; do
  cp ~/lzhiyong-tools/build-tools/aapt2 "$BT/aapt2"
  cp ~/lzhiyong-tools/build-tools/aidl "$BT/aidl"
  cp ~/lzhiyong-tools/build-tools/zipalign "$BT/zipalign"
  chmod +x "$BT"/*
done
```

### ⚠️ 关键坑：AGP 下载自己的 x86_64 aapt2

即使替换了 SDK 二进制，AGP 还会从 Maven 下载 x86_64 版本。

**解决方案**：在 `gradle.properties` 设置：

```properties
android.aapt2FromMavenOverride=/data/data/com.termux/files/home/android-sdk/build-tools/35.0.1/aapt2
```

### NDK 配置

```bash
# 下载 ARM64 NDK r29
curl --http1.1 -L -o ~/termux-ndk-r29.7z \
  "https://github.com/lzhiyong/termux-ndk/releases/download/android-ndk/android-ndk-r29-aarch64.7z"
mkdir -p ~/termux-ndk && cd ~/termux-ndk && 7z x ~/termux-ndk-r29.7z -y
mv ~/termux-ndk/android-ndk-r29 ~/android-sdk/ndk/29.0.14206865
```

### CMake 配置

```bash
pkg install cmake ninja
rm -rf $ANDROID_HOME/cmake/3.22.1/bin/*
ln -sf /data/data/com.termux/files/usr/bin/cmake $ANDROID_HOME/cmake/3.22.1/bin/cmake
ln -sf /data/data/com.termux/files/usr/bin/ninja $ANDROID_HOME/cmake/3.22.1/bin/ninja
```

### 构建命令

```bash
export JAVA_HOME=/data/data/com.termux/files/usr/lib/jvm/java-21-openjdk
./gradlew assembleArm64Debug --no-daemon
```

---

## 9. 聚合新闻 Skills 安装

### 安装 Skills

```bash
# Hacker News
skillhub install hackernews --dir ~/.hermes/skills

# GitHub Trending
skillhub install github-trending --dir ~/.hermes/skills

# 新闻聚合（8 大源）
skillhub install news-aggregator-skill --dir ~/.hermes/skills
```

### ⚠️ 已知问题

- `gif-search`：Tenor API 需要 API key
- `arxiv`、`polymarket`、`blogwatcher`：安装后可能缺脚本文件
- `web-search-free`：缺少脚本文件

### 可用 Skills

| Skills | 功能 | 状态 |
|--------|------|------|
| hackernews | Hacker News 热门/搜索 | ✅ |
| github-trending | GitHub 热门项目 | ✅ |
| news-aggregator | 8 大源聚合 | ✅ |

---

## 10. Skills 管理

### 更新 Skills

```bash
hermes skills update
```

### 从 SkillHub 安装

```bash
# 搜索
skillhub search abc

# 安装到 Hermes
skillhub install abc --dir ~/.hermes/skills
```

### SkillHub 优先策略

国内用户建议将 SkillHub 设为优先源（CN 加速），不可用时回退到 GitHub。

---

## 11. 当前 AI 工具安装方式一览表

| 工具 | 安装方式 | 命令 | 来源 |
|------|---------|------|------|
| **Hermes Agent** | Git 源码 + venv | `git clone` → `uv venv` → `uv pip install -e ".[termux-all]"` | GitHub (NousResearch/hermes-agent) |
| **bun** (JS 运行时) | pkg 包 | `pkg install bun` 或 Hope2333/bun-termux | Termux 仓库 / GitHub |
| **opencode** (AI 编码) | deb 包 | `curl` → `ar x` → `tar xf` → 手动复制 | GitHub (Hope2333/opencode-termux) |
| **mimocode** (MiMo) | deb 包 | `curl` → `ar x` → `tar xf` → 手动复制 | GitHub (Hope2333/MiMoCode-Termux) |
| **MiMo 模型** | API 调用 | 无需本地安装，通过 API 调用 | api.longcat.chat (LongCat-2.0) |
| **uv** (Python 包管理器) | pkg 包 | `pkg install uv` 或 `pip install uv` | Termux 仓库 / PyPI |
| **gh** (GitHub CLI) | pkg 包 | `pkg install gh` | Termux 仓库 |
| **hackernews** (Skill) | skillhub | `skillhub install hackernews --dir ~/.hermes/skills` | SkillHub |
| **github-trending** (Skill) | skillhub | `skillhub install github-trending --dir ~/.hermes/skills` | SkillHub |
| **news-aggregator** (Skill) | skillhub | `skillhub install news-aggregator-skill --dir ~/.hermes/skills` | SkillHub |
| **skillhub** (技能商店) | 官方脚本 | `curl -fsSL .../install.sh \| bash` | skillhub.cn |

---

## 12. 搜索 Skills 安装（重要）

### MCP Tavily 搜索

```bash
# 安装 mcp 包
cd ~/hermes-agent && source venv/bin/activate && pip install mcp

# 添加 mcp_servers 到 ~/.hermes/config.yaml
mcp_servers:
  tavily:
    enabled: true
    headers:
      Authorization: Bearer YOUR_TAVILY_API_KEY
    url: https://tavily.ivanli.cc/mcp
```

重启 Hermes 后自动注册为 `mcp_tavily_tavily_search` 工具。

### 百度搜索（Baidu AI Search）

```bash
# 安装 SkillHub CLI
curl -fsSL https://skillhub-1388575217.cos.ap-guangzhou.myqcloud.com/install/install.sh | bash -s -- --cli-only

# 安装百度搜索 skill
skillhub install baidu-search --namespace ide-rea --dir ~/.hermes/skills/

# 配置 API Key
echo "BAIDU_API_KEY=YOUR_BAIDU_API_KEY" >> ~/.hermes/.env
```

API Key 申请：https://console.bce.baidu.com/ai-search/qianfan/ais/console/apiKey

---

## 持续更新

本仓库根据实战经验持续更新。欢迎 Issue / PR。

---

## 许可证

MIT
