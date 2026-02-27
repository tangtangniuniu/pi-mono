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
  <a href="https://github.com/badlogic/pi-mono/actions/workflows/ci.yml"><img alt="构建状态" src="https://img.shields.io/github/actions/workflow/status/badlogic/pi-mono/ci.yml?style=flat-square&branch=main" /></a>
</p>
<p align="center">
  <a href="https://pi.dev">pi.dev</a> 域名由
  <br /><br />
  <a href="https://exe.dev"><img src="packages/coding-agent/docs/images/exy.png" alt="Exy 吉祥物" width="48" /><br />exe.dev</a>
  慷慨捐赠
</p>

# Pi 单体仓库

> **寻找 pi 编码代理？** 请参阅 **[packages/coding-agent](packages/coding-agent)** 了解安装和使用方法。

用于构建AI代理和管理LLM部署的工具集。

## 包列表

| 包 | 描述 |
|---------|-------------|
| **[@mariozechner/pi-ai](packages/ai)** | 统一的多提供商LLM API（OpenAI、Anthropic、Google等） |
| **[@mariozechner/pi-agent-core](packages/agent)** | 支持工具调用的代理运行时和状态管理 |
| **[@mariozechner/pi-coding-agent](packages/coding-agent)** | 交互式编码代理CLI |
| **[@mariozechner/pi-mom](packages/mom)** | 将消息委托给pi编码代理的Slack机器人 |
| **[@mariozechner/pi-tui](packages/tui)** | 支持差分渲染的终端UI库 |
| **[@mariozechner/pi-web-ui](packages/web-ui)** | AI聊天界面的Web组件 |
| **[@mariozechner/pi-pods](packages/pods)** | 管理GPU Pod上vLLM部署的CLI工具 |

## 贡献指南

请参阅 [CONTRIBUTING.md](CONTRIBUTING.md) 了解贡献指南，以及 [AGENTS.md](AGENTS.md) 了解项目特定规则（适用于人类和代理）。

## 开发

```bash
npm install          # 安装所有依赖
npm run build        # 构建所有包
npm run check        # 代码检查、格式化和类型检查
./test.sh            # 运行测试（没有API密钥时跳过LLM相关测试）
./pi-test.sh         # 从源代码运行pi（必须从仓库根目录运行）
```

> **注意：** `npm run check` 需要先运行 `npm run build`。web-ui包使用 `tsc`，需要依赖项编译后的 `.d.ts` 文件。

## 许可证

MIT