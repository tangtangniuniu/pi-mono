# 🏖️ OSS 休假中

**问题跟踪和PR将于2026年2月23日重新开放。**

在此之前，所有PR将自动关闭。已批准的贡献者可以在休假后提交PR而无需重新审批。如需支持，请加入 [Discord](https://discord.com/invite/3cU7Bz4UPx)。

---

<p align="center">
  <a href="https://shittycodingagent.ai">
    <img src="https://shittycodingagent.ai/logo.svg" alt="pi logo" width="128">
  </a>
</p>
<p align="center">
  <a href="https://discord.com/invite/3cU7Bz4UPx"><img alt="Discord" src="https://img.shields.io/badge/discord-社区-5865F2?style=flat-square&logo=discord&logoColor=white" /></a>
  <a href="https://www.npmjs.com/package/@mariozechner/pi-coding-agent"><img alt="npm" src="https://img.shields.io/npm/v/@mariozechner/pi-coding-agent?style=flat-square" /></a>
  <a href="https://github.com/badlogic/pi-mono/actions/workflows/ci.yml"><img alt="构建状态" src="https://img.shields.io/github/actions/workflow/status/badlogic/pi-mono/ci.yml?style=flat-square&branch=main" /></a>
</p>
<p align="center">
  <a href="https://pi.dev">pi.dev</a> 域名由
  <br /><br />
  <a href="https://exe.dev"><img src="docs/images/exy.png" alt="Exy 吉祥物" width="48" /><br />exe.dev</a>
  慷慨捐赠
</p>

Pi是一个极简的终端编码工具。让pi适应您的工作流程，而不是反过来，无需fork和修改pi的内部代码。通过TypeScript[扩展](#扩展)、[技能](#技能)、[提示模板](#提示模板)和[主题](#主题)来扩展它。将您的扩展、技能、提示模板和主题放入[Pi包](#pi包)中，并通过npm或git与他人分享。

Pi提供了强大的默认功能，但跳过了子代理和计划模式等功能。相反，您可以要求pi构建您想要的功能，或者安装与您工作流程匹配的第三方pi包。

Pi以四种模式运行：交互式、打印或JSON、用于进程集成的RPC，以及用于嵌入到您自己应用中的SDK。请参阅[openclaw/openclaw](https://github.com/openclaw/openclaw)了解真实的SDK集成示例。

## 目录

- [快速开始](#快速开始)
- [提供商和模型](#提供商和模型)
- [交互模式](#交互模式)
  - [编辑器](#编辑器)
  - [命令](#命令)
  - [键盘快捷键](#键盘快捷键)
  - [消息队列](#消息队列)
- [会话](#会话)
  - [分支](#分支)
  - [压缩](#压缩)
- [设置](#设置)
- [上下文文件](#上下文文件)
- [自定义](#自定义)
  - [提示模板](#提示模板)
  - [技能](#技能)
  - [扩展](#扩展)
  - [主题](#主题)
  - [Pi包](#pi包)
- [编程使用](#编程使用)
- [设计理念](#设计理念)
- [CLI参考](#cli参考)

---

## 快速开始

```bash
npm install -g @mariozechner/pi-coding-agent
```

使用API密钥进行身份验证：

```bash
export ANTHROPIC_API_KEY=sk-ant-...
pi
```

或使用您现有的订阅：

```bash
pi
/login  # 然后选择提供商
```

然后只需与pi对话。默认情况下，pi为模型提供四个工具：`read`、`write`、`edit`和`bash`。模型使用这些工具来满足您的请求。通过[技能](#技能)、[提示模板](#提示模板)、[扩展](#扩展)或[pi包](#pi包)添加功能。

**平台说明：** [Windows](docs/windows.md) | [Termux (Android)](docs/termux.md) | [终端设置](docs/terminal-setup.md) | [Shell别名](docs/shell-aliases.md)

---

## 提供商和模型

对于每个内置提供商，pi维护一个支持工具的模型列表，每次发布都会更新。通过订阅（`/login`）或API密钥进行身份验证，然后通过`/model`（或Ctrl+L）选择该提供商的任何模型。

**订阅：**
- Anthropic Claude Pro/Max
- OpenAI ChatGPT Plus/Pro (Codex)
- GitHub Copilot
- Google Gemini CLI
- Google Antigravity

**API密钥：**
- Anthropic
- OpenAI
- Azure OpenAI
- Google Gemini
- Google Vertex
- Amazon Bedrock
- Mistral
- Groq
- Cerebras
- xAI
- OpenRouter
- Vercel AI Gateway
- ZAI
- OpenCode Zen
- Hugging Face
- Kimi For Coding
- MiniMax

请参阅[docs/providers.md](docs/providers.md)获取详细的设置说明。

**自定义提供商和模型：** 如果它们支持兼容的API（OpenAI、Anthropic、Google），可以通过`~/.pi/agent/models.json`添加提供商。对于自定义API或OAuth，请使用扩展。请参阅[docs/models.md](docs/models.md)和[docs/custom-provider.md](docs/custom-provider.md)。

---

## 交互模式

<p align="center"><img src="docs/images/interactive-mode.png" alt="交互模式" width="600"></p>

从上到下的界面：

- **启动头** - 显示快捷键（`/hotkeys`查看全部）、加载的AGENTS.md文件、提示模板、技能和扩展
- **消息** - 您的消息、助手响应、工具调用和结果、通知、错误和扩展UI
- **编辑器** - 您输入的地方；边框颜色表示思考级别
- **页脚** - 工作目录、会话名称、总令牌/缓存使用量、成本、上下文使用量、当前模型

编辑器可以被其他UI临时替换，比如内置的`/settings`或来自扩展的自定义UI（例如，一个问答工具，允许用户以结构化格式回答模型问题）。[扩展](#扩展)也可以替换编辑器，在其上方/下方添加小部件、状态行、自定义页脚或覆盖层。

### 编辑器

| 功能 | 使用方法 |
|---------|-----|
| 文件引用 | 输入`@`进行模糊搜索项目文件 |
| 路径补全 | Tab键补全路径 |
| 多行 | Shift+Enter（或在Windows Terminal上使用Ctrl+Enter） |
| 图像 | Ctrl+V粘贴，或拖放到终端 |
| Bash命令 | `!command`运行并将输出发送给LLM，`!!command`运行但不发送 |

标准的编辑键绑定，用于删除单词、撤销等。请参阅[docs/keybindings.md](docs/keybindings.md)。

### 命令

在编辑器中输入`/`触发命令。[扩展](#扩展)可以注册自定义命令，[技能](#技能)可作为`/skill:name`使用，[提示模板](#提示模板)通过`/templatename`展开。

| 命令 | 描述 |
|---------|-------------|
| `/login`, `/logout` | OAuth身份验证 |
| `/model` | 选择模型 |
| `/thinking` | 切换思考级别 |
| `/settings` | 打开设置界面 |
| `/hotkeys` | 显示所有键盘快捷键 |
| `/session` | 会话管理 |
| `/branch` | 创建会话分支 |
| `/compact` | 压缩会话历史 |
| `/tree` | 显示文件树 |
| `/clear` | 清除屏幕 |
| `/exit` | 退出pi |

### 键盘快捷键

| 快捷键 | 功能 |
|--------|------|
| `Ctrl+L` | 选择模型 |
| `Ctrl+T` | 切换思考级别 |
| `Ctrl+E` | 打开外部编辑器 |
| `Ctrl+F` | 添加后续消息 |
| `Ctrl+D` | 取消排队消息 |
| `Ctrl+W` | 展开/折叠工具输出 |
| `Ctrl+Q` | 退出 |

### 消息队列

您可以排队多个消息，pi将按顺序处理它们：

```
pi
> /queue
Queuing mode enabled. Type /dequeue to stop.
> Fix the bug in utils.ts
> Then add tests for the fix
> And update the documentation
```

---

## 会话

### 分支

创建会话分支以探索不同的解决方案：

```
pi
> Implement feature X
... pi implements feature X ...
> /branch try-alternative-approach
Branch created: try-alternative-approach
> Let's try a different implementation
```

### 压缩

压缩会话历史以减少令牌使用：

```
pi
> /compact
Compacting session history...
Session compressed from 45 messages to 12 messages.
```

---

## 设置

通过`/settings`命令或编辑`~/.pi/agent/settings.json`进行配置：

```json
{
  "showImages": true,
  "autoCompact": true,
  "maxContextTokens": 128000,
  "theme": "dark"
}
```

---

## 上下文文件

pi可以读取上下文文件来了解项目结构：

- `AGENTS.md` - 项目特定的规则和指南
- `.piignore` - 要从文件搜索中排除的文件和目录
- `pi-package.json` - Pi包配置

---

## 自定义

### 提示模板

创建可重用的提示模板：

```bash
# 在 ~/.pi/agent/prompt-templates/ 中创建文件
echo '你是一个专业的代码审查员。请审查以下代码：' > ~/.pi/agent/prompt-templates/code-review
```

使用：`/code-review`

### 技能

技能是工作流特定的CLI工具：

```typescript
// 在扩展中注册技能
pi.registerSkill('deploy', {
  description: '部署应用到生产环境',
  execute: async (args) => {
    // 部署逻辑
  }
});
```

使用：`/skill:deploy`

### 扩展

TypeScript扩展可以添加各种功能：

```typescript
// 简单的扩展示例
export default function myExtension(pi: ExtensionContext) {
  // 注册工具、命令、事件处理程序等
  pi.registerTool(/* ... */);
  pi.registerCommand(/* ... */);
}
```

### 主题

自定义UI主题：

```typescript
const myTheme: Theme = {
  editor: {
    border: 'cyan',
    background: 'black',
    foreground: 'white'
  }
};
```

### Pi包

将扩展、技能、提示模板和主题打包为Pi包：

```json
{
  "name": "my-pi-package",
  "version": "1.0.0",
  "pi": {
    "extensions": ["./dist/extension.js"],
    "skills": ["./dist/skills.js"],
    "promptTemplates": "./templates",
    "themes": "./themes"
  }
}
```

---

## 编程使用

### SDK模式

将pi嵌入到您的应用程序中：

```typescript
import { PiSDK } from '@mariozechner/pi-coding-agent';

const pi = new PiSDK({
  workingDir: process.cwd(),
  model: 'anthropic:claude-sonnet-4-20250514'
});

const response = await pi.prompt('帮我修复这个bug');
console.log(response);
```

### RPC模式

通过标准输入/输出与pi进程通信：

```bash
# 启动RPC模式
pi --mode rpc

# 发送JSON命令
echo '{"type": "prompt", "message": "Hello"}' | pi --mode rpc
```

---

## 设计理念

- **最小化** - 核心功能强大，但可扩展
- **适应性** - 让工具适应您的工作流程
- **可组合性** - 通过扩展组合功能
- **透明性** - 清晰的执行过程和结果

---

## CLI参考

### 基本用法

```bash
pi [options] [prompt]
```

### 选项

| 选项 | 描述 |
|------|------|
| `--mode interactive` | 交互模式（默认） |
| `--mode print` | 打印模式，输出纯文本 |
| `--mode json` | JSON模式，输出结构化JSON |
| `--mode rpc` | RPC模式，通过stdin/stdout通信 |
| `--no-session` | 不使用会话历史 |
| `--working-dir <path>` | 设置工作目录 |
| `--extension <path>` | 加载扩展 |
| `--theme <name>` | 设置主题 |

### 环境变量

| 变量 | 描述 |
|------|------|
| `ANTHROPIC_API_KEY` | Anthropic API密钥 |
| `OPENAI_API_KEY` | OpenAI API密钥 |
| `PI_WORKING_DIR` | 默认工作目录 |
| `PI_EXTENSIONS` | 要加载的扩展路径 |

---

*有关更详细的信息，请参阅各个文档文件：*
- [扩展文档](docs/extensions.md)
- [技能文档](docs/skills.md)
- [会话管理](docs/session.md)
- [RPC协议](docs/rpc.md)
- [SDK使用](docs/sdk.md)