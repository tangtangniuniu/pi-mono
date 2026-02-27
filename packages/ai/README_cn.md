# @mariozechner/pi-ai

统一的LLM API，具有自动模型发现、提供商配置、令牌和成本跟踪功能，以及简单的上下文持久化和会话中途切换到其他模型的能力。

**注意**：此库仅包含支持工具调用（函数调用）的模型，这对于代理工作流至关重要。

## 目录

- [支持的提供商](#支持的提供商)
- [安装](#安装)
- [快速开始](#快速开始)
- [工具](#工具)
  - [定义工具](#定义工具)
  - [处理工具调用](#处理工具调用)
  - [流式工具调用与部分JSON](#流式工具调用与部分json)
  - [验证工具参数](#验证工具参数)
  - [完整事件参考](#完整事件参考)
- [图像输入](#图像输入)
- [思考/推理](#思考推理)
  - [统一接口](#统一接口-streamsimplecompletesimple)
  - [提供商特定选项](#提供商特定选项-streamcomplete)
  - [流式思考内容](#流式思考内容)
- [停止原因](#停止原因)
- [错误处理](#错误处理)
  - [中止请求](#中止请求)
  - [中止后继续](#中止后继续)
- [API、模型和提供商](#api模型和提供商)
  - [提供商和模型](#提供商和模型)
  - [查询提供商和模型](#查询提供商和模型)
  - [自定义模型](#自定义模型)
  - [OpenAI兼容性设置](#openai兼容性设置)
  - [类型安全](#类型安全)
- [跨提供商切换](#跨提供商切换)
- [上下文序列化](#上下文序列化)
- [浏览器使用](#浏览器使用)
  - [环境变量](#环境变量-仅限nodejs)
  - [检查环境变量](#检查环境变量)
- [OAuth提供商](#oauth提供商)
  - [Vertex AI (ADC)](#vertex-ai-adc)
  - [CLI登录](#cli登录)
  - [编程式OAuth](#编程式oauth)
  - [登录流程示例](#登录流程示例)
  - [使用OAuth令牌](#使用oauth令牌)
  - [提供商说明](#提供商说明)
- [许可证](#许可证)

## 支持的提供商

- **OpenAI**
- **Azure OpenAI (Responses)**
- **OpenAI Codex**（ChatGPT Plus/Pro订阅，需要OAuth，见下文）
- **Anthropic**
- **Google**
- **Vertex AI**（通过Vertex AI的Gemini）
- **Mistral**
- **Groq**
- **Cerebras**
- **xAI**
- **OpenRouter**
- **Vercel AI Gateway**
- **MiniMax**
- **GitHub Copilot**（需要OAuth，见下文）
- **Google Gemini CLI**（需要OAuth，见下文）
- **Antigravity**（需要OAuth，见下文）
- **Amazon Bedrock**
- **Kimi For Coding**（Moonshot AI，使用Anthropic兼容的API）
- **任何OpenAI兼容的API**：Ollama、vLLM、LM Studio等

## 安装

```bash
npm install @mariozechner/pi-ai
```

TypeBox导出从 `@mariozechner/pi-ai` 重新导出：`Type`、`Static` 和 `TSchema`。

## 快速开始

```typescript
import { Type, getModel, stream, complete, Context, Tool, StringEnum } from '@mariozechner/pi-ai';

// 完全类型化，支持提供商和模型的自动完成
const model = getModel('openai', 'gpt-4o-mini');

// 使用TypeBox模式定义工具，确保类型安全和验证
const tools: Tool[] = [{
  name: 'get_time',
  description: '获取当前时间',
  parameters: Type.Object({
    timezone: Type.Optional(Type.String({ description: '可选时区（例如：America/New_York）' }))
  })
}];

// 构建对话上下文（可轻松序列化并在模型之间传输）
const context: Context = {
  systemPrompt: '你是一个有用的助手。',
  messages: [{ role: 'user', content: '现在几点了？' }],
  tools
};

// 选项1：流式传输，包含所有事件类型
const s = stream(model, context);

for await (const event of s) {
  switch (event.type) {
    case 'start':
      console.log(`开始使用 ${event.partial.model}`);
      break;
    case 'text_start':
      console.log('\n[文本开始]');
      break;
    case 'text_delta':
      process.stdout.write(event.delta);
      break;
    case 'text_end':
      console.log('\n[文本结束]');
      break;
    case 'thinking_start':
      console.log('[模型正在思考...]');
      break;
    case 'thinking_delta':
      process.stdout.write(event.delta);
      break;
    case 'thinking_end':
      console.log('[思考完成]');
      break;
    case 'toolcall_start':
      console.log(`\n[工具调用开始：索引 ${event.contentIndex}]`);
      break;
    case 'toolcall_delta':
      // 工具参数正在流式传输
      const partialCall = event.partial.content[event.contentIndex];
      if (partialCall.type === 'toolCall') {
        console.log(`[为 ${partialCall.name} 流式传输参数]`);
      }
      break;
    case 'toolcall_end':
      console.log(`\n调用的工具：${event.toolCall.name}`);
      console.log(`参数：${JSON.stringify(event.toolCall.arguments)}`);
      break;
    case 'done':
      console.log(`\n完成：${event.reason}`);
      break;
    case 'error':
      console.error(`错误：${event.error}`);
      break;
  }
}

// 流式传输后获取最终消息，添加到上下文
const finalMessage = await s.result();
context.messages.push(finalMessage);

// 处理任何工具调用
const toolCalls = finalMessage.content.filter(b => b.type === 'toolCall');
for (const call of toolCalls) {
  // 执行工具
  const result = call.name === 'get_time'
    ? new Date().toLocaleString('zh-CN', {
        timeZone: call.arguments.timezone || 'UTC',
        dateStyle: 'full',
        timeStyle: 'long'
      })
    : '未知工具';

  // 将工具结果添加到上下文（支持文本和图像）
  context.messages.push({
    role: 'toolResult',
    toolCallId: call.id,
    toolName: call.name,
    content: [{ type: 'text', text: result }],
    isError: false,
    timestamp: Date.now()
  });
}

// 如果有工具调用，继续对话
if (toolCalls.length > 0) {
  const continuation = await complete(model, context);
  context.messages.push(continuation);
  console.log('工具执行后：', continuation.content);
}

console.log(`总令牌数：输入 ${finalMessage.usage.input}，输出 ${finalMessage.usage.output}`);
console.log(`成本：$${finalMessage.usage.cost.total.toFixed(4)}`);

// 选项2：无需流式传输获取完整响应
const response = await complete(model, context);

for (const block of response.content) {
  if (block.type === 'text') {
    console.log(block.text);
  } else if (block.type === 'toolCall') {
    console.log(`工具：${block.name}(${JSON.stringify(block.arguments)})`);
  }
}
```

## 工具

### 定义工具

工具使用TypeBox模式定义，提供类型安全和运行时验证：

```typescript
import { Type } from '@mariozechner/pi-ai';

const tools: Tool[] = [{
  name: 'search_web',
  description: '搜索网络获取信息',
  parameters: Type.Object({
    query: Type.String({ description: '搜索查询' }),
    max_results: Type.Optional(Type.Number({ minimum: 1, maximum: 10 }))
  })
}, {
  name: 'calculate',
  description: '执行数学计算',
  parameters: Type.Object({
    expression: Type.String({ description: '数学表达式' })
  })
}];
```

### 处理工具调用

当LLM调用工具时，您需要执行相应的函数并返回结果：

```typescript
const toolCalls = finalMessage.content.filter(b => b.type === 'toolCall');
for (const call of toolCalls) {
  let result: string;
  let isError = false;
  
  try {
    switch (call.name) {
      case 'search_web':
        result = await searchWeb(call.arguments.query, call.arguments.max_results);
        break;
      case 'calculate':
        result = eval(call.arguments.expression).toString();
        break;
      default:
        result = `未知工具: ${call.name}`;
        isError = true;
    }
  } catch (error) {
    result = `工具执行错误: ${error.message}`;
    isError = true;
  }
  
  context.messages.push({
    role: 'toolResult',
    toolCallId: call.id,
    toolName: call.name,
    content: [{ type: 'text', text: result }],
    isError,
    timestamp: Date.now()
  });
}
```

### 流式工具调用与部分JSON

某些提供商支持工具参数的流式传输，允许您逐步解析JSON：

```typescript
case 'toolcall_delta':
  const partialCall = event.partial.content[event.contentIndex];
  if (partialCall.type === 'toolCall') {
    // 参数正在流式传输，partialCall.arguments包含部分解析的JSON
    console.log('流式参数:', JSON.stringify(partialCall.arguments));
  }
  break;
```

### 验证工具参数

使用内置的验证函数确保工具参数符合模式：

```typescript
import { validateToolArguments } from '@mariozechner/pi-ai';

for (const call of toolCalls) {
  try {
    const validatedArgs = validateToolArguments(tools, call.name, call.arguments);
    // 使用验证后的参数执行工具
  } catch (error) {
    // 处理验证错误
  }
}
```

### 完整事件参考

流式API发出以下事件类型：

- `start` - 流开始
- `text_start` - 文本块开始
- `text_delta` - 文本增量
- `text_end` - 文本块结束
- `thinking_start` - 思考块开始
- `thinking_delta` - 思考增量
- `thinking_end` - 思考块结束
- `toolcall_start` - 工具调用开始
- `toolcall_delta` - 工具参数增量
- `toolcall_end` - 工具调用结束
- `done` - 流完成
- `error` - 发生错误

## 图像输入

支持图像输入作为用户消息的一部分：

```typescript
const context: Context = {
  messages: [{
    role: 'user',
    content: [
      { type: 'text', text: '请描述这张图片：' },
      { 
        type: 'image', 
        data: 'base64编码的图像数据',
        mimeType: 'image/jpeg'
      }
    ],
    timestamp: Date.now()
  }]
};
```

## 思考/推理

### 统一接口 (streamSimple/completeSimple)

对于大多数用例，使用统一的接口：

```typescript
import { streamSimple, completeSimple } from '@mariozechner/pi-ai';

// 自动处理提供商差异
const stream = streamSimple(model, context);
const response = await completeSimple(model, context);
```

### 提供商特定选项 (stream/complete)

对于高级用例，可以使用提供商特定的选项：

```typescript
// OpenAI推理努力
const response = await complete(model, context, {
  reasoningEffort: 'high'
});

// Anthropic思考级别
const response = await complete(model, context, {
  thinking: { type: 'enabled', budgetTokens: 4096 }
});
```

### 流式思考内容

思考内容可以像文本一样流式传输：

```typescript
case 'thinking_delta':
  // 思考内容正在流式传输
  process.stdout.write(event.delta);
  break;
```

## 停止原因

LLM响应包含停止原因，帮助了解为什么生成停止：

- `stop` - 正常完成
- `length` - 达到令牌限制
- `toolUse` - 调用了工具
- `error` - 发生错误
- `aborted` - 请求被中止

## 错误处理

### 中止请求

使用AbortController中止长时间运行的请求：

```typescript
const controller = new AbortController();
setTimeout(() => controller.abort(), 30000); // 30秒超时

const response = await complete(model, context, {
  signal: controller.signal
});
```

### 中止后继续

从中止点继续对话：

```typescript
// 移除中止的消息
context.messages.pop();

// 继续对话
const continuation = await complete(model, context);
```

## API、模型和提供商

### 提供商和模型

获取可用的提供商和模型：

```typescript
import { getProviders, getModels } from '@mariozechner/pi-ai';

const providers = getProviders();
const models = getModels('openai'); // 特定提供商的所有模型
```

### 查询提供商和模型

按功能过滤模型：

```typescript
import { queryModels } from '@mariozechner/pi-ai';

// 查找支持推理和图像输入的模型
const reasoningModels = queryModels({
  reasoning: true,
  input: ['text', 'image']
});
```

### 自定义模型

添加自定义模型或覆盖现有模型：

```typescript
import { registerModel } from '@mariozechner/pi-ai';

registerModel({
  id: 'custom-model',
  name: '自定义模型',
  api: 'openai-completions',
  provider: 'custom-provider',
  baseUrl: 'https://api.example.com',
  reasoning: true,
  input: ['text'],
  cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 }
});
```

### OpenAI兼容性设置

为自定义API配置兼容性设置：

```typescript
const model = getModel('custom-provider', 'custom-model', {
  compat: {
    supportsDeveloperRole: false,      // 使用"system"而不是"developer"
    supportsReasoningEffort: false,    // 禁用reasoning_effort参数
    maxTokensField: 'max_tokens',      // 而不是"max_completion_tokens"
    requiresToolResultName: true,      // 工具结果需要name字段
    requiresMistralToolIds: true       // 工具ID必须是9个字母数字字符
  }
});
```

### 类型安全

完整的TypeScript类型支持，包括自动完成：

```typescript
// 类型安全的模型选择
const model = getModel('anthropic', 'claude-sonnet-4-20250514');

// 类型安全的工具定义
const tools: Tool[] = [{
  name: 'search',
  description: '搜索信息',
  parameters: Type.Object({
    query: Type.String(),
    limit: Type.Optional(Type.Number())
  })
}];
```

## 跨提供商切换

在会话中途切换到不同的模型：

```typescript
// 开始与OpenAI对话
let context: Context = { /* ... */ };
const openaiResponse = await complete(getModel('openai', 'gpt-4'), context);
context.messages.push(openaiResponse);

// 切换到Anthropic继续
const anthropicResponse = await complete(getModel('anthropic', 'claude-3-opus'), context);
context.messages.push(anthropicResponse);
```

## 上下文序列化

上下文对象可以轻松序列化和反序列化：

```typescript
// 序列化
const serialized = JSON.stringify(context);

// 反序列化
const deserialized: Context = JSON.parse(serialized);

// 继续对话
const response = await complete(model, deserialized);
```

## 浏览器使用

在浏览器环境中使用：

```typescript
import { getModel, complete } from '@mariozechner/pi-ai';

// 配置CORS代理（如果需要）
const model = getModel('openai', 'gpt-4o-mini', {
  baseUrl: '/api/proxy/openai' // 您的CORS代理端点
});

const response = await complete(model, context);
```

### 环境变量（仅限Node.js）

在Node.js中，API密钥通过环境变量配置：

```bash
export OPENAI_API_KEY=sk-...
export ANTHROPIC_API_KEY=sk-ant-...
```

### 检查环境变量

验证环境变量是否已设置：

```typescript
import { checkEnvApiKeys } from '@mariozechner/pi-ai';

const missing = checkEnvApiKeys(['openai', 'anthropic']);
if (missing.length > 0) {
  console.log('缺少API密钥：', missing);
}
```

## OAuth提供商

### Vertex AI (ADC)

使用应用程序默认凭据（ADC）进行身份验证：

```typescript
const model = getModel('google-vertex', 'gemini-1.5-pro');
// 自动使用ADC进行身份验证
```

### CLI登录

支持交互式CLI登录流程：

```typescript
import { oauthLogin } from '@mariozechner/pi-ai';

const credentials = await oauthLogin('github-copilot');
// 保存凭据供以后使用
```

### 编程式OAuth

以编程方式处理OAuth流程：

```typescript
import { OAuthProvider } from '@mariozechner/pi-ai';

const provider = new OAuthProvider({
  name: 'custom-provider',
  async login(callbacks) {
    // 实现登录逻辑
  },
  async refreshToken(credentials) {
    // 实现令牌刷新
  }
});
```

### 登录流程示例

完整的OAuth登录流程：

```typescript
const credentials = await oauthLogin('github-copilot', {
  onDeviceCode: (deviceCode) => {
    console.log('请在浏览器中访问：', deviceCode.verification_uri);
    console.log('输入代码：', deviceCode.user_code);
  },
  onPolling: () => {
    console.log('等待用户授权...');
  }
});

console.log('登录成功！');
```

### 使用OAuth令牌

使用OAuth凭据进行API调用：

```typescript
const model = getModel('github-copilot', 'claude-3-opus', {
  oauthCredentials: credentials
});
```

### 提供商说明

**GitHub Copilot**：需要GitHub Copilot订阅和OAuth授权
**Google Gemini CLI**：需要Google Cloud项目和服务账户
**OpenAI Codex**：需要ChatGPT Plus/Pro订阅

## 许可证

MIT