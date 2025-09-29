# Neovate Code 源码深度解析

## 📋 目录

1. [项目概述](#项目概述)
2. [技术架构](#技术架构)
3. [核心模块详解](#核心模块详解)
4. [工作流程示例](#工作流程示例)
5. [复现指南](#复现指南)

## 项目概述

**Neovate Code** 是一个基于AI的编码助手工具，通过命令行界面（CLI）与开发者交互，利用多个AI模型（如DeepSeek、OpenAI、Anthropic等）来帮助编写、修改和管理代码。

### 核心特性

- 🤖 **多模型支持**：集成主流AI提供商（OpenAI、Anthropic、DeepSeek等）
- 🔧 **丰富的工具集**：文件读写、代码编辑、命令执行等
- 🔌 **插件系统**：高度可扩展的架构设计
- 💬 **流式交互**：实时显示AI响应
- ✅ **安全审批**：所有修改操作需要用户确认
- 📝 **会话管理**：支持会话保存和恢复

### 技术栈

- **运行时**: Node.js + Bun
- **开发语言**: TypeScript
- **AI SDK**: @ai-sdk系列（统一的AI接口）
- **UI框架**: React + Ink（终端UI）
- **状态管理**: Zustand + Valtio
- **构建工具**: Bun + API Extractor

## 技术架构

### 项目结构

```
/workspace/
├── src/                    # 核心源码
│   ├── index.ts           # 主入口，runNeovate函数
│   ├── cli.ts             # CLI启动文件
│   ├── context.ts         # 上下文管理器（核心）
│   ├── project.ts         # 项目/会话管理
│   ├── loop.ts            # AI对话循环引擎
│   ├── model.ts           # AI模型配置和管理
│   ├── tool.ts            # 工具系统基础架构
│   ├── tools/             # 具体工具实现
│   │   ├── read.ts        # 文件读取
│   │   ├── write.ts       # 文件写入
│   │   ├── edit.ts        # 文件编辑
│   │   ├── bash.ts        # 命令执行
│   │   └── ...
│   ├── ui/                # 终端UI组件
│   │   ├── App.tsx        # 主应用组件
│   │   └── store.ts       # Zustand状态管理
│   ├── messageBus.ts      # 消息总线系统
│   └── session.ts         # 会话管理
├── browser/               # Web界面（可选）
└── vscode-extension/      # VS Code扩展
```

### 架构设计模式

1. **插件架构模式**
   - 使用钩子系统实现扩展性
   - 允许插件修改配置、工具、提示词等

2. **分层配置管理**
   - 优先级：默认配置 < 全局配置 < 项目配置 < 命令行参数
   - 使用 `defu` 库进行深度合并

3. **流式处理模式**
   - 实时处理AI响应，提供更好的用户体验
   - 检测不完整的XML标签，避免截断工具调用

4. **工具抽象**
   - 统一的工具接口
   - 参数验证使用Zod schema
   - 支持审批机制

5. **消息总线模式**
   - UI层和业务层解耦
   - 支持请求/响应和事件模式

## 核心模块详解

### 1. Context 上下文管理器 (`context.ts`)

Context是整个系统的核心，管理着全局配置、插件、路径等：

```typescript
export class Context {
  cwd: string;                    // 当前工作目录
  productName: string;            // 产品名称
  version: string;                // 版本号
  config: Config;                 // 配置对象
  paths: Paths;                   // 路径管理器
  mcpManager: MCPManager;         // MCP协议管理器
  #pluginManager: PluginManager; // 插件管理器（私有）
  
  // 核心方法：创建Context实例
  static async create(opts: ContextCreateOpts) {
    // 1. 初始化路径管理
    const paths = new Paths({ productName, cwd });
    
    // 2. 加载配置（支持多层级：默认->全局->项目->命令行参数）
    const configManager = new ConfigManager(cwd, productName, opts.argvConfig);
    
    // 3. 扫描和加载插件（三个来源）
    const plugins = [
      ...buildInPlugins,           // 内置插件
      ...globalPlugins,            // 全局插件目录
      ...projectPlugins,           // 项目插件目录
    ];
    
    // 4. 应用插件钩子，允许插件修改配置
    const resolvedConfig = await apply({
      hook: 'config',
      args: [{ config: initialConfig }],
      type: PluginHookType.SeriesMerge
    });
    
    // 5. 初始化MCP管理器
    const mcpManager = MCPManager.create(mcpServers);
    
    return new Context({...});
  }
}
```

**关键设计要点**：
- 使用插件系统实现可扩展性
- 配置分层管理，优先级清晰
- 支持MCP（Model Context Protocol）协议

### 2. Project 项目管理器 (`project.ts`)

管理AI会话和消息流：

```typescript
export class Project {
  session: Session;    // 会话对象
  context: Context;    // 上下文引用
  
  async send(message: string, opts = {}) {
    // 1. 解析可用工具
    let tools = await resolveTools({
      context: this.context,
      sessionId: this.session.id,
      write: true,      // 允许写操作的工具
      todo: true        // 包含TODO管理工具
    });
    
    // 2. 应用插件钩子，允许修改工具列表
    tools = await this.context.apply({
      hook: 'tool',
      memo: tools,
      type: PluginHookType.SeriesMerge
    });
    
    // 3. 生成系统提示词
    let systemPrompt = generateSystemPrompt({
      todo: this.context.config.todo,
      productName: this.context.productName,
      language: this.context.config.language,
      outputStyle
    });
    
    // 4. 进入对话循环
    return this.sendWithSystemPromptAndTools(message, {
      ...opts,
      tools,
      systemPrompt
    });
  }
}
```

### 3. Loop 对话循环引擎 (`loop.ts`)

这是AI交互的核心循环，处理消息流和工具调用：

```typescript
export async function runLoop(opts: RunLoopOpts): Promise<LoopResult> {
  const history = new History({ messages });
  const maxTurns = opts.maxTurns ?? 50;
  
  while (true) {
    turnsCount++;
    
    // 1. 检查是否超过最大轮次
    if (turnsCount > maxTurns) {
      return { success: false, error: { type: 'max_turns_exceeded' }};
    }
    
    // 2. 压缩历史记录（如果需要）
    if (opts.autoCompact) {
      const compressed = await history.compress(opts.model);
    }
    
    // 3. 创建AI Agent并发送请求
    const agent = new Agent({
      name: 'code',
      model: opts.model.model.id,
      instructions: systemPrompt + toolsPrompt
    });
    
    // 4. 流式处理响应
    for await (const chunk of result.toStream()) {
      switch (chunk.data.event.type) {
        case 'text-delta':
          // 处理文本增量，检查XML标签完整性
          if (!hasIncompleteXmlTag(text)) {
            await opts.onTextDelta?.(textDelta);
          }
          break;
          
        case 'reasoning':
          // 处理推理过程（如DeepSeek R1的思考链）
          await opts.onReasoning?.(chunk.data.event.textDelta);
          break;
      }
    }
    
    // 5. 解析工具调用
    const parsed = parseMessage(text);
    const toolUse = parsed.find(item => item.type === 'tool_use');
    
    if (toolUse) {
      // 6. 请求用户批准
      const approved = await opts.onToolApprove?.(toolUse);
      
      if (approved) {
        // 7. 执行工具
        const toolResult = await opts.tools.invoke(
          toolUse.name, 
          JSON.stringify(toolUse.params)
        );
        
        // 8. 将结果添加到历史
        await history.addMessage({
          role: 'user',
          content: [{ type: 'tool_result', result: toolResult }]
        });
        
        // 继续循环
        turnsCount--;  // 工具调用不计入轮次
      } else {
        return { success: false, error: { type: 'tool_denied' }};
      }
    } else {
      // 没有工具调用，结束循环
      break;
    }
  }
  
  return { success: true, data: { text: finalText, history, usage }};
}
```

**关键特性**：
- 支持流式响应处理
- XML标签不完整检测（避免截断工具调用）
- 工具调用审批机制
- Token使用统计

### 4. Tool 工具系统 (`tool.ts` + `tools/`)

工具系统是Neovate的核心能力，允许AI执行实际操作：

```typescript
// 工具定义基础结构
export function createTool<T extends z.ZodType>(opts: {
  name: string;
  description: string;
  parameters: T;              // Zod schema定义参数
  execute: (params) => Promise<ToolResult>;
  approval?: { category: 'read' | 'write' };
  getDescription?: (opts) => string;
}) {
  return {
    ...opts,
    displayName: opts.name,
    execute: async (params) => {
      // 参数验证
      const result = opts.parameters.safeParse(params);
      if (!result.success) {
        return { isError: true, llmContent: 'Invalid parameters' };
      }
      
      // 执行工具逻辑
      return opts.execute(result.data);
    }
  };
}
```

**内置工具列表**：

| 工具名 | 功能描述 | 需要审批 |
|--------|----------|----------|
| `read` | 读取文件内容 | 否 |
| `write` | 写入文件 | 是 |
| `edit` | 编辑文件（查找替换） | 是 |
| `bash` | 执行shell命令 | 是 |
| `ls` | 列出目录内容 | 否 |
| `glob` | 文件搜索 | 否 |
| `grep` | 内容搜索 | 否 |
| `fetch` | 网络请求 | 否 |
| `todo` | 任务管理 | 是 |

### 5. 消息总线系统 (`messageBus.ts`)

实现UI层和Node层的通信：

```typescript
export class MessageBus {
  private transport?: MessageTransport;
  private handlers = new Map<string, MessageHandler>();
  private pendingRequests = new Map<MessageId, PendingRequest>();
  
  // 发送请求并等待响应
  async request<T>(method: string, params: any): Promise<T> {
    const id = randomUUID();
    
    return new Promise((resolve, reject) => {
      // 存储待处理请求
      this.pendingRequests.set(id, {
        resolve, reject,
        timeout: setTimeout(() => {
          reject(new Error('Request timeout'));
        }, 30000)
      });
      
      // 发送请求消息
      this.transport?.send({
        type: 'request',
        id, method, params,
        timestamp: Date.now()
      });
    });
  }
  
  // 注册方法处理器
  registerHandler(method: string, handler: MessageHandler) {
    this.handlers.set(method, handler);
  }
}
```

### 6. 模型管理系统 (`model.ts`)

支持多个AI提供商的统一接口：

```typescript
export const providers: ProvidersMap = {
  deepseek: {
    id: 'deepseek',
    name: 'DeepSeek',
    env: ['DEEPSEEK_API_KEY'],
    models: {
      'deepseek-v3-1': { 
        name: 'DeepSeek V3.1',
        reasoning: true,
        tool_call: true,
        limit: { context: 163840, output: 163840 }
      },
      'deepseek-r1-0528': { 
        name: 'DeepSeek R1',
        reasoning: true,  // 支持推理链
        tool_call: true
      }
    },
    createModel(name: string) {
      return createDeepSeek({
        apiKey: process.env.DEEPSEEK_API_KEY,
        baseURL: 'https://api.deepseek.com'
      })(name);
    }
  },
  openai: { /* OpenAI配置 */ },
  anthropic: { /* Claude配置 */ },
  // ... 更多提供商
};
```

## 工作流程示例

### 场景：用户要求"给 utils.js 文件添加一个计算平均值的函数"

#### 完整流程图

```
用户输入: "给 utils.js 文件添加一个计算平均值的函数"
   ↓
CLI参数解析 (cli.ts)
   ↓
Context创建 (context.ts)
   ├─ 配置加载
   ├─ 插件初始化
   └─ 工具注册
   ↓
UI初始化 (App.tsx)
   ↓
消息总线建立 (messageBus.ts)
   ↓
Project创建 (project.ts)
   ↓
Loop循环 (loop.ts)
   ├─ 第1轮: AI决定读取文件
   │   ├─ 发送请求到AI
   │   ├─ AI响应: 需要先查看文件
   │   ├─ 解析工具调用: read工具
   │   ├─ 请求用户批准
   │   ├─ 执行read工具
   │   └─ 返回文件内容
   │
   ├─ 第2轮: AI生成修改
   │   ├─ 发送文件内容给AI
   │   ├─ AI响应: 生成average函数
   │   ├─ 解析工具调用: edit工具
   │   ├─ 显示diff预览
   │   ├─ 请求用户批准
   │   ├─ 执行edit工具
   │   └─ 文件修改成功
   │
   └─ 第3轮: AI总结
       ├─ AI响应: 总结完成的工作
       └─ 无工具调用，结束循环
   ↓
结果持久化
   ├─ JSONL日志记录
   ├─ 文件系统更新
   └─ UI显示结果
```

#### 详细执行步骤

##### 步骤1：启动和初始化

```bash
$ neovate "给 utils.js 文件添加一个计算平均值的函数"
```

系统解析命令行参数，创建Context上下文，加载配置和插件。

##### 步骤2：第一轮循环 - 读取文件

**AI请求**：
```javascript
// 发送给AI的消息
{
  role: 'user',
  content: "给 utils.js 文件添加一个计算平均值的函数"
}
```

**AI响应**：
```xml
好的，我来帮你给 utils.js 文件添加一个计算平均值的函数。
首先让我查看一下当前的 utils.js 文件内容。

<use_tool>
<tool_name>read</tool_name>
<arguments>
{
  "file_path": "utils.js"
}
</arguments>
</use_tool>
```

**工具执行**：
```javascript
// read工具返回
{
  llmContent: `
    function add(a, b) {
      return a + b;
    }
    
    function subtract(a, b) {
      return a - b;
    }
    
    module.exports = { add, subtract };
  `
}
```

##### 步骤3：第二轮循环 - 编辑文件

**AI响应**：
```xml
我看到了 utils.js 文件的内容。现在我将添加一个计算平均值的函数。

<use_tool>
<tool_name>edit</tool_name>
<arguments>
{
  "file_path": "utils.js",
  "old_string": "function subtract(a, b) {\n  return a - b;\n}\n\nmodule.exports = { add, subtract };",
  "new_string": "function subtract(a, b) {\n  return a - b;\n}\n\nfunction average(numbers) {\n  if (!Array.isArray(numbers) || numbers.length === 0) {\n    return 0;\n  }\n  const sum = numbers.reduce((acc, num) => acc + num, 0);\n  return sum / numbers.length;\n}\n\nmodule.exports = { add, subtract, average };"
}
</arguments>
</use_tool>
```

**UI显示Diff预览**：
```diff
@@ -5,3 +5,11 @@
   return a - b;
 }
 
+function average(numbers) {
+  if (!Array.isArray(numbers) || numbers.length === 0) {
+    return 0;
+  }
+  const sum = numbers.reduce((acc, num) => acc + num, 0);
+  return sum / numbers.length;
+}
+
-module.exports = { add, subtract };
+module.exports = { add, subtract, average };
```

##### 步骤4：第三轮循环 - 总结

**AI最终响应**：
```markdown
✅ 我已成功为你的 utils.js 文件添加了 average 函数！

**添加的功能：**
- 函数名：`average`
- 参数：`numbers` - 数字数组
- 功能：计算数组中所有数字的平均值
- 错误处理：如果输入不是数组或数组为空，返回0

**使用示例：**
```javascript
const { average } = require('./utils');

console.log(average([1, 2, 3, 4, 5]));  // 输出: 3
console.log(average([10, 20, 30]));     // 输出: 20
console.log(average([]));               // 输出: 0
```

函数已经添加到 module.exports 中，可以直接使用了。
```

##### 步骤5：结果持久化

**JSONL日志文件** (`~/.config/neovate/sessions/abc12345.jsonl`)：
```jsonl
{"type":"config","config":{"approvalMode":"default","model":"openai/gpt-4o"}}
{"type":"message","role":"user","content":"给 utils.js 文件添加一个计算平均值的函数","uuid":"msg-001","timestamp":"2025-09-29T10:00:00Z"}
{"type":"message","role":"assistant","content":[{"type":"text","text":"好的，我来帮你..."},{"type":"tool_use","name":"read","input":{"file_path":"utils.js"}}],"uuid":"msg-002"}
{"type":"message","role":"user","content":[{"type":"tool_result","result":{"llmContent":"// 文件内容..."}}],"uuid":"msg-003"}
{"type":"message","role":"assistant","content":[{"type":"text","text":"我看到了..."},{"type":"tool_use","name":"edit","input":{...}}],"uuid":"msg-004"}
{"type":"message","role":"user","content":[{"type":"tool_result","result":{"llmContent":"File successfully edited"}}],"uuid":"msg-005"}
{"type":"message","role":"assistant","content":"✅ 我已成功为你的 utils.js 文件添加了 average 函数！...","uuid":"msg-006"}
{"type":"usage","promptTokens":1250,"completionTokens":380,"totalTokens":1630}
```

**最终文件内容** (`utils.js`)：
```javascript
function add(a, b) {
  return a + b;
}

function subtract(a, b) {
  return a - b;
}

function average(numbers) {
  if (!Array.isArray(numbers) || numbers.length === 0) {
    return 0;
  }
  const sum = numbers.reduce((acc, num) => acc + num, 0);
  return sum / numbers.length;
}

module.exports = { add, subtract, average };
```

## 复现指南

如果你想自己实现类似的AI编码助手，以下是关键步骤：

### 1. 基础架构搭建

```typescript
// 1. 创建Context管理器
class Context {
  constructor(public cwd: string, public config: Config) {}
  
  static async create(opts) {
    // 加载配置、插件等
    return new Context(opts.cwd, config);
  }
}

// 2. 实现消息历史管理
class History {
  messages: Message[] = [];
  
  addMessage(msg: Message) {
    this.messages.push(msg);
  }
  
  toAgentInput() {
    // 转换为AI可理解的格式
    return this.messages.map(/* ... */);
  }
}

// 3. 创建工具系统
function createTool(opts) {
  return {
    name: opts.name,
    execute: async (params) => {
      // 验证参数并执行
      return opts.execute(params);
    }
  };
}
```

### 2. 实现核心循环

```typescript
async function runLoop(opts) {
  const history = new History();
  
  while (true) {
    // 1. 发送请求到AI
    const response = await callAI({
      messages: history.toAgentInput(),
      tools: opts.tools
    });
    
    // 2. 处理响应
    if (response.toolCall) {
      // 3. 执行工具
      const result = await executeTool(response.toolCall);
      
      // 4. 添加到历史
      history.addMessage({ role: 'tool', content: result });
      
      // 继续循环
    } else {
      // 纯文本响应，结束循环
      break;
    }
  }
}
```

### 3. 插件系统实现

```typescript
// 插件接口定义
interface Plugin {
  name: string;
  
  // 生命周期钩子
  async config?(opts: { config: Config }): Promise<Config>;
  async initialized?(opts: { cwd: string }): Promise<void>;
  async tool?(opts: { tools: Tool[] }): Promise<Tool[]>;
  async systemPrompt?(opts: { prompt: string }): Promise<string>;
  async destroy?(): Promise<void>;
}

// 插件管理器
class PluginManager {
  plugins: Plugin[] = [];
  
  async apply(opts: PluginApplyOpts) {
    let result = opts.memo;
    
    for (const plugin of this.plugins) {
      const hook = plugin[opts.hook];
      if (hook) {
        if (opts.type === PluginHookType.SeriesMerge) {
          // 合并结果
          result = { ...result, ...(await hook(opts.args)) };
        } else if (opts.type === PluginHookType.SeriesLast) {
          // 覆盖结果
          result = await hook(opts.args);
        }
      }
    }
    
    return result;
  }
}
```

### 4. UI系统（基于Ink）

```typescript
// React组件用于终端UI
import { Box, Text } from 'ink';
import React, { useState } from 'react';

function App() {
  const [messages, setMessages] = useState([]);
  const [input, setInput] = useState('');
  
  return (
    <Box flexDirection="column">
      {/* 消息历史 */}
      <Box flexDirection="column">
        {messages.map(msg => (
          <Message key={msg.id} message={msg} />
        ))}
      </Box>
      
      {/* 输入框 */}
      <TextInput
        value={input}
        onChange={setInput}
        onSubmit={handleSubmit}
      />
    </Box>
  );
}

// 渲染到终端
import { render } from 'ink';
render(<App />);
```

## 关键优化点

1. **历史压缩**：当对话过长时自动压缩历史消息
2. **并行工具执行**：支持同时执行多个工具
3. **缓存机制**：缓存模型配置和工具定义
4. **错误恢复**：工具执行失败时的优雅降级

## 创新特性

1. **MCP协议支持**：可以接入外部工具服务
2. **多模型支持**：统一接口支持多个AI提供商
3. **推理链支持**：支持显示AI的思考过程（如DeepSeek R1）
4. **@ 引用机制**：可以在提示词中引用文件和目录

## 总结

Neovate Code 的设计非常优雅，通过插件系统实现了高度的可扩展性，同时保持了核心功能的简洁。其关键成功因素包括：

- **分层架构**：UI层、业务层、工具层清晰分离
- **插件系统**：通过钩子机制实现高度可扩展性
- **流式处理**：实时显示AI响应，提升用户体验
- **审批机制**：工具调用需要用户确认，保证安全性
- **状态管理**：使用Zustand管理UI状态，消息总线同步数据
- **持久化**：JSONL格式记录完整会话，支持恢复和回放

如果你想复现类似的项目，建议从最小可行版本开始：

1. 实现基本的AI对话循环
2. 添加文件读写工具
3. 实现工具审批机制
4. 添加更多工具和优化用户体验

这个项目为AI辅助编程提供了一个优秀的参考实现，其架构设计和实现细节都值得深入学习和借鉴。