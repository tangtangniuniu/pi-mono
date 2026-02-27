# @mariozechner/pi-agent-core

支持工具执行和事件流的状态化代理。基于 `@mariozechner/pi-ai` 构建。

## 安装

```bash
npm install @mariozechner/pi-agent-core
```

## 快速开始

```typescript
import { Agent } from "@mariozechner/pi-agent-core";
import { getModel } from "@mariozechner/pi-ai";

const agent = new Agent({
  initialState: {
    systemPrompt: "你是一个有用的助手。",
    model: getModel("anthropic", "claude-sonnet-4-20250514"),
  },
});

agent.subscribe((event) => {
  if (event.type === "message_update" && event.assistantMessageEvent.type === "text_delta") {
    // 仅流式传输新的文本块
    process.stdout.write(event.assistantMessageEvent.delta);
  }
});

await agent.prompt("你好！");
```

## 核心概念

### AgentMessage vs LLM Message

代理使用 `AgentMessage`，这是一种灵活的类型，可以包含：
- 标准LLM消息（`user`、`assistant`、`toolResult`）
- 通过声明合并的自定义应用特定消息类型

LLM只理解 `user`、`assistant` 和 `toolResult`。`convertToLlm` 函数通过在每次LLM调用之前过滤和转换消息来弥合这一差距。

### 消息流

```
AgentMessage[] → transformContext() → AgentMessage[] → convertToLlm() → Message[] → LLM
                    (可选)                           (必需)
```

1. **transformContext**: 修剪旧消息，注入外部上下文
2. **convertToLlm**: 过滤掉仅用于UI的消息，将自定义类型转换为LLM格式

## 事件流

代理发出用于UI更新的事件。了解事件序列有助于构建响应式界面。

### prompt() 事件序列

当您调用 `prompt("你好")` 时：

```
prompt("你好")
├─ agent_start
├─ turn_start
├─ message_start   { message: userMessage }      // 您的提示
├─ message_end     { message: userMessage }
├─ message_start   { message: assistantMessage } // LLM开始响应
├─ message_update  { message: partial... }       // 流式块
├─ message_update  { message: partial... }
├─ message_end     { message: assistantMessage } // 完整响应
├─ turn_end        { message, toolResults: [] }
└─ agent_end       { messages: [...] }
```

### 包含工具调用

如果助手调用工具，循环将继续：

```
prompt("读取config.json")
├─ agent_start
├─ turn_start
├─ message_start/end  { userMessage }
├─ message_start      { assistantMessage with toolCall }
├─ message_update...
├─ message_end        { assistantMessage }
├─ tool_execution_start  { toolCallId, toolName, args }
├─ tool_execution_update { partialResult }           // 如果工具流式传输
├─ tool_execution_end    { toolCallId, result }
├─ message_start/end  { toolResultMessage }
├─ turn_end           { message, toolResults: [toolResult] }
│
├─ turn_start                                        // 下一轮
├─ message_start      { assistantMessage }           // LLM响应工具结果
├─ message_update...
├─ message_end
├─ turn_end
└─ agent_end
```

### continue() 事件序列

`continue()` 从现有上下文恢复，不添加新消息。用于错误后的重试。

```typescript
// 错误后，从当前状态重试
await agent.continue();
```

上下文中的最后一条消息必须是 `user` 或 `toolResult`（不能是 `assistant`）。

### 事件类型

| 事件 | 描述 |
|-------|-------------|
| `agent_start` | 代理开始处理 |
| `agent_end` | 代理完成，包含所有新消息 |
| `turn_start` | 新一轮开始（一次LLM调用 + 工具执行） |
| `turn_end` | 轮次完成，包含助手消息和工具结果 |
| `message_start` | 任何消息开始（user、assistant、toolResult） |
| `message_update` | **仅限助手。** 包含带有增量的 `assistantMessageEvent` |
| `message_end` | 消息完成 |
| `tool_execution_start` | 工具开始 |
| `tool_execution_update` | 工具流式传输进度 |
| `tool_execution_end` | 工具完成 |

## 代理选项

```typescript
const agent = new Agent({
  // 初始状态
  initialState: {
    systemPrompt: string,
    model: Model<any>,
    thinkingLevel: "off" | "minimal" | "low" | "medium" | "high" | "xhigh",
    tools: AgentTool<any>[],
    messages: AgentMessage[],
  },

  // 将AgentMessage[]转换为LLM Message[]（对于自定义消息类型必需）
  convertToLlm: (messages) => messages.filter(...),

  // 在convertToLlm之前转换上下文（用于修剪、压缩）
  transformContext: async (messages, signal) => pruneOldMessages(messages),

  // 转向模式："one-at-a-time"（默认）或"all"
  steeringMode: "one-at-a-time",

  // 后续消息模式："one-at-a-time"（默认）或"all"
  followUpMode: "one-at-a-time",

  // 工具执行选项
  toolExecution: {
    // 工具执行超时（毫秒）
    timeout: 30000,
    
    // 工具执行期间是否允许用户中断
    allowInterruption: true,
  },
});
```

## 工具执行

### 定义工具

工具使用TypeBox模式定义，提供类型安全和运行时验证：

```typescript
import { Type } from "@mariozechner/pi-ai";

const tools = [{
  name: "search_files",
  description: "在项目中搜索文件",
  parameters: Type.Object({
    query: Type.String({ description: "搜索查询" }),
    file_type: Type.Optional(Type.String({ description: "文件类型过滤器" })),
  }),
  execute: async (args) => {
    // 实现搜索逻辑
    return { content: [{ type: "text", text: "搜索结果..." }] };
  }
}];
```

### 工具执行生命周期

工具执行发出详细的事件，允许实时UI更新：

```typescript
agent.subscribe((event) => {
  switch (event.type) {
    case "tool_execution_start":
      console.log(`工具开始: ${event.toolName}`);
      console.log(`参数: ${JSON.stringify(event.args)}`);
      break;
      
    case "tool_execution_update":
      console.log(`工具进度: ${event.partialResult}`);
      break;
      
    case "tool_execution_end":
      console.log(`工具完成: ${event.isError ? "错误" : "成功"}`);
      console.log(`结果: ${event.result.content}`);
      break;
  }
});
```

## 自定义消息类型

### 通过声明合并扩展

通过声明合并添加自定义消息类型：

```typescript
// 在您的项目中声明
import { AgentMessage } from "@mariozechner/pi-agent-core";

declare module "@mariozechner/pi-agent-core" {
  interface AgentMessage {
    role: "notification" | "status" | "custom";
    content: string;
    timestamp: number;
  }
}

// 现在可以使用自定义消息类型
const notification: AgentMessage = {
  role: "notification",
  content: "任务开始执行",
  timestamp: Date.now()
};
```

### 转换自定义消息

在 `convertToLlm` 中处理自定义消息：

```typescript
const convertToLlm = (messages: AgentMessage[]): Message[] => {
  return messages.flatMap(msg => {
    switch (msg.role) {
      case "notification":
        // 过滤掉仅用于UI的通知
        return [];
        
      case "status":
        // 将状态消息转换为用户消息
        return [{
          role: "user",
          content: `状态更新: ${msg.content}`,
          timestamp: msg.timestamp
        }];
        
      default:
        // 传递标准消息类型
        return [msg];
    }
  });
};
```

## 高级用法

### 上下文转换

使用 `transformContext` 进行高级上下文管理：

```typescript
const transformContext = async (
  messages: AgentMessage[], 
  signal?: AbortSignal
): Promise<AgentMessage[]> => {
  // 修剪旧消息以保持在令牌限制内
  const pruned = pruneOldMessages(messages, 100000);
  
  // 注入外部上下文（例如，当前时间、系统状态）
  pruned.unshift({
    role: "user",
    content: `当前时间: ${new Date().toISOString()}`,
    timestamp: Date.now()
  });
  
  return pruned;
};
```

### 转向和后续消息

处理转向和后续消息：

```typescript
// 添加转向消息（中断当前处理）
await agent.steer("请停止当前任务，先处理这个紧急问题");

// 添加后续消息（在当前任务完成后处理）
await agent.followUp("完成后，请总结一下所做的工作");
```

### 错误处理

健壮的错误处理：

```typescript
try {
  await agent.prompt("执行这个任务");
} catch (error) {
  if (error.name === "AbortError") {
    console.log("请求被用户中止");
  } else {
    console.error("代理错误:", error);
    
    // 从中断点继续
    await agent.continue();
  }
}
```

## 性能优化

### 消息压缩

实现自定义压缩策略：

```typescript
const compressMessages = (messages: AgentMessage[]): AgentMessage[] => {
  // 将长对话历史压缩为摘要
  if (messages.length > 50) {
    const summary: AgentMessage = {
      role: "user",
      content: "之前的对话摘要: ...",
      timestamp: Date.now()
    };
    return [summary, ...messages.slice(-10)]; // 保留最后10条消息
  }
  return messages;
};
```

### 流式优化

优化流式处理性能：

```typescript
// 批量处理更新以减少UI重绘
let updateTimer: NodeJS.Timeout;
agent.subscribe((event) => {
  if (event.type === "message_update") {
    clearTimeout(updateTimer);
    updateTimer = setTimeout(() => {
      // 批量更新UI
      updateUI(event.message);
    }, 50); // 50ms去抖动
  }
});
```

## 集成示例

### 与React集成

```typescript
import { useState, useEffect } from "react";
import { Agent } from "@mariozechner/pi-agent-core";

function ChatComponent() {
  const [messages, setMessages] = useState<AgentMessage[]>([]);
  const [agent] = useState(() => new Agent({/* 配置 */}));

  useEffect(() => {
    const subscription = agent.subscribe((event) => {
      switch (event.type) {
        case "message_start":
          setMessages(prev => [...prev, event.message]);
          break;
        case "message_update":
          setMessages(prev => 
            prev.map(msg => 
              msg.timestamp === event.message.timestamp ? event.message : msg
            )
          );
          break;
      }
    });

    return () => subscription.unsubscribe();
  }, [agent]);

  return <div>{/* 渲染消息 */}</div>;
}
```

### 与Node.js CLI集成

```typescript
import { Agent } from "@mariozechner/pi-agent-core";
import readline from "readline";

const rl = readline.createInterface({
  input: process.stdin,
  output: process.stdout
});

const agent = new Agent({/* 配置 */});

agent.subscribe((event) => {
  if (event.type === "message_update" && event.assistantMessageEvent.type === "text_delta") {
    process.stdout.write(event.assistantMessageEvent.delta);
  }
});

async function chat() {
  while (true) {
    const input = await new Promise<string>(resolve => 
      rl.question("\n您: ", resolve)
    );
    
    if (input === "/exit") break;
    
    await agent.prompt(input);
  }
  
  rl.close();
}

chat();
```

## 最佳实践

1. **始终处理中止**：使用AbortController处理用户中断
2. **合理使用转换**：仅在必要时转换消息以减少开销
3. **监控令牌使用**：定期检查令牌使用情况以避免超出限制
4. **实现适当的错误处理**：为工具执行和LLM调用提供有意义的错误消息
5. **优化UI更新**：使用去抖动和批量更新提高性能

## 许可证

MIT