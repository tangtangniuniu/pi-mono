# 开发规则

## 第一条消息
如果用户在第一条消息中没有给您具体的任务，请先阅读README.md，然后询问要处理哪个模块。根据回答，并行读取相关的README.md文件：
- packages/ai/README.md
- packages/tui/README.md
- packages/agent/README.md
- packages/coding-agent/README.md
- packages/mom/README.md
- packages/pods/README.md
- packages/web-ui/README.md

## 代码质量
- 除非绝对必要，否则不使用 `any` 类型
- 在 node_modules 中查找外部 API 类型定义，而不是猜测
- **绝对不要使用内联导入** - 不要使用 `await import("./foo.js")`，不要在类型位置使用 `import("pkg").Type`，不要使用动态导入类型。始终使用标准的顶层导入。
- 永远不要删除或降级代码来修复过时依赖的类型错误；而是升级依赖
- 删除功能或看起来是有意设计的代码时，一定要先询问
- 永远不要硬编码按键检查，例如 `matchesKey(keyData, "ctrl+x")`。所有按键绑定必须是可配置的。将默认值添加到匹配对象（`DEFAULT_EDITOR_KEYBINDINGS` 或 `DEFAULT_APP_KEYBINDINGS`）中

## 命令
- 代码更改后（不是文档更改）：运行 `npm run check`（获取完整输出，不要截断）。在提交前修复所有错误、警告和信息提示。
- 注意：`npm run check` 不运行测试。
- 绝对不要运行：`npm run dev`、`npm run build`、`npm test`
- 只有在用户指示时才运行特定测试：`npx tsx ../../node_modules/vitest/dist/cli.js --run test/specific.test.ts`
- 从包根目录运行测试，而不是从仓库根目录。
- 编写测试时，运行它们，在测试或实现中识别问题，然后迭代直到修复。
- 除非用户要求，否则绝对不要提交

## GitHub 问题
阅读问题时：
- 始终阅读问题的所有评论
- 使用此命令一次性获取所有内容：
  ```bash
  gh issue view <number> --json title,body,comments,labels,state
  ```

创建问题时：
- 添加 `pkg:*` 标签来表明问题影响了哪个包
  - 可用标签：`pkg:agent`、`pkg:ai`、`pkg:coding-agent`、`pkg:mom`、`pkg:pods`、`pkg:tui`、`pkg:web-ui`
- 如果问题跨多个包，添加所有相关标签

通过提交关闭问题时：
- 在提交消息中包含 `fixes #<number>` 或 `closes #<number>`
- 这会在提交合并时自动关闭问题

## PR 工作流
- 先不拉取本地代码，先分析 PR
- 如果用户批准：创建功能分支，拉取 PR，在 main 上变基，应用调整，提交，合并到 main，推送，关闭 PR，并以用户的语气留下评论
- 您永远不会自己打开 PR。我们在功能分支中工作，直到一切都符合用户的要求，然后合并到 main，然后推送。

## 工具
- 使用 GitHub CLI 处理问题/PR
- 为问题/PR 添加包标签：pkg:agent、pkg:ai、pkg:coding-agent、pkg:mom、pkg:pods、pkg:tui、pkg:web-ui

## 使用 tmux 测试 pi 交互模式

在受控终端环境中测试 pi 的 TUI：

```bash
# 创建具有特定尺寸的 tmux 会话
tmux new-session -d -s pi-test -x 80 -y 24

# 从源代码启动 pi
tmux send-keys -t pi-test "cd /Users/badlogic/workspaces/pi-mono && ./pi-test.sh" Enter

# 等待启动，然后捕获输出
sleep 3 && tmux capture-pane -t pi-test -p

# 发送输入
tmux send-keys -t pi-test "your prompt here" Enter

# 发送特殊键
tmux send-keys -t pi-test Escape
tmux send-keys -t pi-test C-o  # ctrl+o

# 清理
tmux kill-session -t pi-test
```

## 风格
- 保持回答简短简洁
- 在提交、问题、PR 评论或代码中不使用表情符号
- 不使用 fluff 或欢快的填充文本
- 仅使用技术散文，友善但直接（例如，"Thanks @user" 而不是 "Thanks so much @user!"）

## 变更日志
位置：`packages/*/CHANGELOG.md`（每个包都有自己的）

### 格式
在 `## [Unreleased]` 下使用这些部分：
- `### Breaking Changes` - 需要迁移的 API 更改
- `### Added` - 新功能
- `### Changed` - 对现有功能的更改
- `### Fixed` - 错误修复
- `### Removed` - 删除的功能

### 规则
- 在添加条目之前，阅读完整的 `[Unreleased]` 部分，查看哪些子部分已经存在
- 新条目始终放在 `## [Unreleased]` 部分下
- 附加到现有子部分（例如 `### Fixed`），不要创建重复项
- 永远不要修改已发布的版本部分（例如 `## [0.12.2]`）
- 每个版本部分一旦发布就是不可变的

### 署名
- **内部更改（来自问题）**：`Fixed foo bar ([#123](https://github.com/badlogic/pi-mono/issues/123))`
- **外部贡献**：`Added feature X ([#456](https://github.com/badlogic/pi-mono/pull/456) by [@username](https://github.com/username))`

## 添加新的 LLM 提供商（packages/ai）

添加新提供商需要在多个文件中进行更改：

### 1. 核心类型（`packages/ai/src/types.ts`）
- 向 `Api` 类型联合添加 API 标识符（例如 `"bedrock-converse-stream"`）
- 创建扩展 `StreamOptions` 的选项接口
- 向 `ApiOptionsMap` 添加映射
- 向 `KnownProvider` 类型联合添加提供商名称

### 2. 提供商实现（`packages/ai/src/providers/`）
创建提供商文件，导出：
- 返回 `AssistantMessageEventStream` 的 `stream<Provider>()` 函数
- 消息/工具转换函数
- 发出标准化事件的响应解析（`text`、`tool_call`、`thinking`、`usage`、`stop`）

### 3. 流集成（`packages/ai/src/stream.ts`）
- 导入提供商的流函数和选项类型
- 在 `getEnvApiKey()` 中添加凭据检测
- 在 `mapOptionsForApi()` 中为 `SimpleStreamOptions` 映射添加 case
- 向 `streamFunctions` 映射添加提供商

### 4. 模型生成（`packages/ai/scripts/generate-models.ts`）
- 添加从提供商源获取/解析模型的逻辑
- 映射到标准化的 `Model` 接口

### 5. 测试（`packages/ai/test/`）
向以下测试添加提供商：`stream.test.ts`、`tokens.test.ts`、`abort.test.ts`、`empty.test.ts`、`context-overflow.test.ts`、`image-limits.test.ts`、`unicode-surrogate.test.ts`、`tool-call-without-result.test.ts`、`image-tool-result.test.ts`、`total-tokens.test.ts`、`cross-provider-handoff.test.ts`。

对于 `cross-provider-handoff.test.ts`，至少添加一个提供商/模型对。如果提供商暴露了多个模型系列（例如 GPT 和 Claude），每个系列至少添加一个对。

对于非标准身份验证，创建实用工具（例如 `bedrock-utils.ts`）并包含凭据检测。

### 6. 编码代理（`packages/coding-agent/`）
- `src/core/model-resolver.ts`：向 `DEFAULT_MODELS` 添加默认模型 ID
- `src/cli/args.ts`：添加环境变量文档
- `README.md`：添加提供商设置说明

### 7. 文档
- `packages/ai/README.md`：添加到提供商表，记录选项/身份验证，添加环境变量
- `packages/ai/CHANGELOG.md`：在 `## [Unreleased]` 下添加条目

## 发布

**同步版本控制**：所有包始终共享相同的版本号。每次发布都会一起更新所有包。

**版本语义**（无主要版本）：
- `patch`：错误修复和新功能
- `minor`：API 破坏性更改

### 步骤

1. **更新 CHANGELOGs**：确保自上次发布以来的所有更改都记录在每个受影响包的 CHANGELOG.md 的 `[Unreleased]` 部分中

2. **运行发布脚本**：
   ```bash
   npm run release:patch    # 修复和添加
   npm run release:minor    # API 破坏性更改
   ```

该脚本处理：版本号提升、CHANGELOG 最终确定、提交、标记、发布，以及添加新的 `[Unreleased]` 部分。

## **关键** 工具使用规则 **关键**
- 永远不要使用 sed/cat 读取文件或文件范围。始终使用读取工具（使用 offset + limit 进行范围读取）。
- 您必须在编辑前完整阅读您要修改的每个文件。

## **关键** 并行代理的 Git 规则 **关键**

多个代理可能同时在同一个工作树中的不同文件上工作。您必须遵循这些规则：

### 提交
- **仅提交您在此会话中更改的文件**
- 当有相关问题或 PR 时，始终在提交消息中包含 `fixes #<number>` 或 `closes #<number>`
- 永远不要使用 `git add -A` 或 `git add .` - 这些会包含其他代理的更改
- 始终使用 `git add <specific-file-paths>` 只列出您修改的文件
- 提交前，运行 `git status` 并验证您只暂存了您的文件
- 跟踪您在会话期间创建/修改/删除的文件

### 禁止的 Git 操作
这些命令可能会破坏其他代理的工作：
- `git reset --hard` - 破坏未提交的更改
- `git checkout .` - 破坏未提交的更改
- `git clean -fd` - 删除未跟踪的文件
- `git stash` - 隐藏所有更改，包括其他代理的工作
- `git add -A` / `git add .` - 暂存其他代理的未提交工作
- `git commit --no-verify` - 绕过必需的检查，绝对不允许

### 安全工作流
```bash
# 1. 首先检查状态
git status

# 2. 只添加您的特定文件
git add packages/ai/src/providers/transform-messages.ts
git add packages/ai/CHANGELOG.md

# 3. 提交
git commit -m "fix(ai): description"

# 4. 推送（如果需要，pull --rebase，但永远不要 reset/checkout）
git pull --rebase && git push
```

### 如果变基冲突发生
- 仅在您的文件中解决冲突
- 如果冲突在您未修改的文件中，中止并询问用户
- 永远不要强制推送

---

## 技术说明

为了帮助不熟悉这些技术的开发者，下面对项目中使用的关键技术进行详细说明：

### npm workspaces（monorepo）

**什么是 npm workspaces？**

npm workspaces 是 npm 官方提供的 monorepo（单体仓库）管理方案。它允许您在单个仓库中管理多个相互关联的 npm 包，而不必为每个包单独设置仓库。

**为什么使用 monorepo？**

1. **统一管理依赖**：所有包共享相同的 node_modules，避免重复安装
2. **简化开发流程**：可以同时修改多个包并立即看到变化
3. **版本同步**：更容易保持包之间的版本一致性
4. **共享代码**：包之间可以直接相互引用，无需发布到 npm

**项目结构**

```
pi-mono/
├── packages/
│   ├── ai/
│   ├── agent/
│   ├── coding-agent/
│   ├── tui/
│   ├── web-ui/
│   ├── mom/
│   └── pods/
├── package.json          # 根 package.json
└── node_modules/        # 共享的 node_modules
```

**常用命令**

```bash
# 安装所有依赖（从根目录运行）
npm install

# 构建所有包
npm run build

# 检查所有包的代码质量
npm run check

# 在特定包中运行命令
# 方式1：进入包目录
cd packages/ai
npm run build

# 方式2：使用 npm workspace
npm run build --workspace packages/ai

# 运行所有包的测试
npm test
```

**工作原理**

在根目录的 package.json 中配置：

```json
{
  "private": true,
  "workspaces": [
    "packages/*"
  ]
}
```

这样，npm 会将 packages/ 下的所有目录识别为工作区，它们之间可以相互引用，而不会在每个包中重复安装 node_modules。

### Biome（格式化、linting）

**什么是 Biome？**

Biome 是一个快速、现代化的代码格式化和 linting 工具，是 Prettier + ESLint 的替代品。它使用 Rust 编写，速度非常快，并且可以同时处理格式化和 linting。

**为什么使用 Biome？**

1. **速度快**：比 Prettier + ESLint 组合快 10-100 倍
2. **单一工具**：同时处理格式化和 linting，配置更简单
3. **零配置**：开箱即用的合理默认设置
4. **TypeScript 优先**：对 TypeScript 支持非常好

**项目中的配置**

在根目录有 `biome.json` 配置文件。

**常用命令**

```bash
# 检查代码（格式化 + linting）
npm run check

# 自动修复可修复的问题
npx biome check --write

# 仅格式化代码
npx biome format --write .

# 仅 lint 检查
npx biome lint
```

**Biome 的主要功能**

1. **代码格式化**
   - 自动格式化 JavaScript/TypeScript 代码
   - 统一缩进、换行、引号等
   - 支持 .json、.md 等多种文件

2. **代码检查（Linting）**
   - 检查潜在的 bug
   - 检查代码风格问题
   - 检查安全问题
   - 检查性能问题

**配置示例（biome.json）**

```json
{
  "formatter": {
    "indentStyle": "tab",
    "lineWidth": 100
  },
  "linter": {
    "enabled": true,
    "rules": {
      "recommended": true
    }
  }
}
```

### Vitest（测试框架）

**什么是 Vitest？**

Vitest 是一个现代化的 JavaScript/TypeScript 测试框架，API 与 Jest 兼容，但速度更快，并且对 Vite 生态系统集成很好。

**为什么使用 Vitest？**

1. **速度快**：使用 esbuild 即时转换代码
2. **TypeScript 优先**：原生支持 TypeScript，无需额外配置
3. **Jest 兼容**：API 与 Jest 几乎相同，学习成本低
4. **内置功能**：内置断言、模拟、覆盖率等功能
5. **即时反馈**：支持监听模式，文件变化时自动重新运行测试

**项目中的使用**

测试文件放在每个包的 `test/` 目录中。

**常用命令**

```bash
# 运行所有测试（不推荐，项目规则要求只运行特定测试）
npm test

# 运行特定测试文件（从包根目录运行）
cd packages/ai
npx tsx ../../node_modules/vitest/dist/cli.js --run test/stream.test.ts

# 以监听模式运行（开发时使用）
npx tsx ../../node_modules/vitest/dist/cli.js test/stream.test.ts
```

**测试文件结构**

```typescript
import { describe, expect, it } from "vitest";
import { someFunction } from "../src/some-file.js";

describe("someFunction", () => {
  it("should work correctly", () => {
    const result = someFunction("input");
    expect(result).toBe("expected output");
  });

  it("should handle edge cases", () => {
    expect(() => someFunction(null)).toThrow();
  });
});
```

**核心概念**

1. **测试套件（describe）**
   - 组织相关的测试
   - 可以嵌套使用

2. **测试用例（it/test）**
   - 单个测试函数
   - 描述测试的预期行为

3. **断言（expect）**
   - 验证结果是否符合预期
   - 常用方法：`toBe()`、`toEqual()`、`toThrow()`、`toBeTruthy()` 等

4. **模拟（mock）**
   - 模拟外部依赖
   - 隔离被测试的代码

**完整示例**

```typescript
import { describe, expect, it, vi } from "vitest";
import { Calculator } from "../src/calculator.js";

describe("Calculator", () => {
  it("should add two numbers", () => {
    const calc = new Calculator();
    expect(calc.add(2, 3)).toBe(5);
  });

  it("should subtract two numbers", () => {
    const calc = new Calculator();
    expect(calc.subtract(5, 3)).toBe(2);
  });

  it("should call external service", async () => {
    // 模拟外部服务
    const mockService = {
      fetch: vi.fn().mockResolvedValue({ data: "test" })
    };
    
    const calc = new Calculator(mockService);
    const result = await calc.fetchData();
    
    expect(mockService.fetch).toHaveBeenCalled();
    expect(result).toEqual({ data: "test" });
  });
});
```

**运行测试覆盖率**

```bash
# 生成覆盖率报告
npx vitest --coverage
```

**项目测试策略**

根据 AGENTS.md 的规则：
- 只在用户明确要求时运行测试
- 运行特定测试而不是所有测试
- 从包根目录运行测试
- 编写测试后立即运行，迭代修复问题

---

## 总结

这三个工具（npm workspaces、Biome、Vitest）共同构成了一个现代化、高效的开发工作流：

1. **npm workspaces** 管理多个相关包的依赖和构建
2. **Biome** 保证代码质量和一致性
3. **Vitest** 快速、可靠地运行测试

通过学习和使用这些工具，您可以更高效地参与 Pi 项目的开发！
