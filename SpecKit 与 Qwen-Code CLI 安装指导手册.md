# SpecKit 与 Qwen-Code CLI 安装指导手册

> **适用系统**：Linux / macOS / Windows（支持 PowerShell）  
> **最后更新时间**：2026年1月28日  

本手册将引导您依次安装 [SpecKit](https://github.com/github/spec-kit) 和 [Qwen-Code CLI](https://developer.aliyun.com/article/1680177)，帮助您快速搭建 AI 驱动的开发环境。

---

## 一、前置依赖

在开始安装前，请确保您的系统已安装以下基础工具：

| 工具 | 说明 | 安装方式 |
|------|------|----------|
| **Python 3.11+** | SpecKit 的运行环境 | [官网下载](https://www.python.org/downloads/) |
| **uv** | Python 包管理器（替代 pip + venv） | `pip install uv` 或 [官方安装指南](https://docs.astral.sh/uv/) |
| **Git** | 版本控制工具 | [官网下载](https://git-scm.com/downloads) |
| **Node.js 20+** | Qwen-Code CLI 的运行环境 | [官网下载](https://nodejs.org/) 或通过 Homebrew (`brew install node`) |

### 1. 安装**Python 3.11+**
- 我使用的旧版的ubuntu20.04，所以只能编译安装，新版的Linux系统可以选择直接下载安装或使用 deadsnakes PPA 安装
```bash
# 编译安装

# 1. 安装编译依赖
sudo apt update
sudo apt install -y build-essential zlib1g-dev libncurses5-dev libgdbm-dev \
    libnss3-dev libssl-dev libreadline-dev libffi-dev libsqlite3-dev wget libbz2-dev

# 2. 下载 Python 3.11 源码（以 3.11.10 为例）
cd /tmp
wget https://www.python.org/ftp/python/3.11.10/Python-3.11.10.tgz
tar -xf Python-3.11.10.tgz
cd Python-3.11.10

# 3. 配置并编译（--enable-optimizations 提升性能）
./configure --enable-optimizations --prefix=/usr/local
make -j $(nproc)

# 4. 安装（不会覆盖系统 Python）
sudo make altinstall

```

```bash
# 使用 deadsnakes PPA 安装

sudo apt update # 更新包列表

sudo apt install software-properties-common # 安装必要的依赖

sudo add-apt-repository ppa:deadsnakes/ppa # 添加 deadsnakes PPA

sudo apt update # 再次更新包列表

sudo apt install python3.11 # 安装 Python 3.11

sudo apt install python3.11-pip # 安装 pip（如果需要）

sudo apt install python3.11-dev python3.11-venv # 安装其他有用的工具
``` 

> 💡 **提示**：Windows 用户无需 WSL，PowerShell 脚本已原生支持。

---

## 二、安装 SpecKit

### 1. 初始化新项目

使用 `uvx` 直接从 GitHub 仓库初始化 SpecKit 项目（推荐方式）：

```bash
# 在新目录中初始化
uvx --from git+https://github.com/github/spec-kit.git specify init <PROJECT_NAME>

# 或在当前目录初始化
uvx --from git+https://github.com/github/spec-kit.git specify init .
# 等价于
uvx --from git+https://github.com/github/spec-kit.git specify init --here
```

### 2. （可选）指定 AI 编码代理

SpecKit 支持多种 AI 工具。初始化时可显式指定：

```bash
# 示例：使用 Claude
uvx --from git+https://github.com/github/spec-kit.git specify init myproj --ai claude

# 其他选项：gemini, copilot, codebuddy
```

### 3. （可选）指定脚本类型

默认行为：
- Windows → 使用 PowerShell (`.ps1`)
- Linux/macOS → 使用 Bash (`.sh`)

强制指定脚本类型：

```bash
uvx ... specify init myproj --script sh   # 强制 Bash
uvx ... specify init myproj --script ps   # 强制 PowerShell
```

### 4. 跳过工具检查（高级）

若仅需模板，不验证本地是否安装对应 AI 工具：

```bash
uvx ... specify init myproj --ai claude --ignore-agent-tools
```

### 5. 验证安装

成功初始化后，您将在项目根目录看到：

- `.specify/scripts/`：包含 `.sh` 和 `.ps1` 自动化脚本
- 可用的 AI 命令（在您的 AI 代理中）：
  - `/speckit.specify` — 创建规格说明
  - `/speckit.plan` — 生成实现计划
  - `/speckit.tasks` — 拆解为可执行任务

---

## 三、安装 Qwen-Code CLI

> 📌 Qwen-Code 是阿里云通义千问推出的 AI 编码 CLI 工具，支持代码生成、文档处理、Excel 操作等。

### 1. 获取 API Key

#### 步骤 1：选择平台（根据地理位置）

| 用户地区 | 平台 |
|--------|------|
| 中国内地 | [阿里云百炼](http://bailian.console.aliyun.com) 或 [魔搭 ModelScope](https://modelscope.cn) |
| 国际用户 | [Alibaba Cloud Model Studio](http://modelstudio.console.alibabacloud.com) |

#### 步骤 2：创建 API Key

1. 登录对应平台
2. 激活 `qwen-coder-plus` 模型（通常有免费额度）
3. 进入 **API Key 管理** 页面 → **创建 Key** → 复制密钥（形如 `sk-xxxxxx`）

### 2. 安装 Qwen-Code CLI

#### 方法一：全局 npm 安装（推荐）

```bash
# 确保已安装 Node.js 20+
npm install -g @qwen-code/qwen-code

# 验证安装
qwen --version
```

#### 方法二：从源码安装（开发者）

```bash
git clone https://github.com/QwenLM/qwen-code.git
cd qwen-code
npm install
npm install -g .
```

### 3. 配置 API 认证

Qwen-Code 使用 OpenAI 兼容的环境变量命名（**注意前缀为 `OPENAI_`**）。

#### 推荐方式：项目级 `.env` 文件

在您的项目根目录创建 `.env` 文件：

```env
# 必填项
OPENAI_API_KEY="sk-xxxxxxxxxxxxxxxxxxxxxxxx"
OPENAI_MODEL="qwen3-coder-plus"

# 根据地区选择 BASE URL
# 中国内地用户：
OPENAI_BASE_URL="https://dashscope.aliyuncs.com/compatible-mode/v1"
# 国际用户：
# OPENAI_BASE_URL="https://dashscope-intl.aliyuncs.com/compatible-mode/v1"
```

> ✅ 优势：配置隔离、便于团队共享（可提交 `.env.example`）

#### 替代方式：系统环境变量

- **Linux/macOS**:
  ```bash
  export OPENAI_API_KEY="sk-..."
  export OPENAI_BASE_URL="https://dashscope.aliyuncs.com/compatible-mode/v1"
  export OPENAI_MODEL="qwen3-coder-plus"
  ```
- **Windows (CMD)**:
  ```cmd
  setx OPENAI_API_KEY "sk-..."
  setx OPENAI_BASE_URL "https://dashscope.aliyuncs.com/compatible-mode/v1"
  setx OPENAI_MODEL "qwen3-coder-plus"
  ```

### 4. 验证 Qwen-Code

启动交互式会话：

```bash
qwen
```

应看到提示符 `qwen >`，表示已成功连接云端模型。

常用操作：
- `@file.txt` — 指定文件上下文
- `!ls` — 执行 shell 命令
- `/help` — 查看内置命令
- `Ctrl+C`（两次）— 退出

---

## 四、常见问题

### Q1: Git 在 Linux 上认证失败？
安装 Git Credential Manager：

```bash
wget https://github.com/git-ecosystem/git-credential-manager/releases/download/v2.6.1/gcm-linux_amd64.2.6.1.deb
sudo dpkg -i gcm-linux_amd64.2.6.1.deb
git config --global credential.helper manager
rm gcm-linux_amd64.2.6.1.deb
```

### Q2: Qwen-Code 报错 “Invalid API Key”？
- 确认 Key 来自正确平台（国内 vs 国际）
- 检查 `.env` 文件是否位于当前工作目录
- 确保 `OPENAI_BASE_URL` 与地区匹配

---

## 五、下一步

- 使用 SpecKit 编写 `/speckit.specify` 规格
- 在 Qwen-Code 中输入自然语言指令，如：
  ```text
  @src/ 请为这个目录生成 README.md
  ```
- 结合两者，构建完整的 AI 驱动开发工作流！

> 🎉 恭喜！您已成功配置 SpecKit + Qwen-Code 开发环境。
