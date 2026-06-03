# Replicating Cline's Architecture in Your Project

This guide shows you how to build a high-performance AI coding agent system using the same architectural patterns as Cline.

## Core Principles

Cline achieves extreme speed and efficiency through:

1. **Streaming & Delta-based Processing** - Never wait for complete responses
2. **Layered Architecture** - Clear separation of concerns (WebviewProvider → Controller → Task)
3. **Lazy Loading & Smart Chunking** - Only load what's needed
4. **Session Persistence** - Resume from checkpoints, avoid recomputation
5. **Non-blocking Concurrency** - All I/O is async
6. **Provider Abstraction** - Swap LLM providers without changing core logic
7. **Modular Tool System** - Add/remove tools via interfaces

---

## Architecture Layers

### Layer 1: Entry Point (Platform Specific)
```
src/
├── extension.ts          # VS Code specific
├── hosts/                # Host adapters (VSCode, CLI, JetBrains)
│   ├── vscode/
│   ├── cli/
│   └── jetbrains/
└── common.ts             # Shared initialization
```

**Responsibility**: Platform integration, not business logic.

### Layer 2: Webview/UI Provider
```
src/core/webview/
├── index.ts              # Lifecycle management
├── VscodeWebviewProvider.ts
└── protocol.ts           # Message contracts
```

**Responsibility**: Handle platform-specific UI communication, stream updates to UI.

```typescript
// Example: VS Code specific webview handling
export class VscodeWebviewProvider implements vscode.WebviewViewProvider {
  resolveWebviewView(webviewView: vscode.WebviewView) {
    webviewView.webview.onDidReceiveMessage((message) => {
      this.controller.handleMessage(message)
    })
  }
}
```

### Layer 3: Controller (Business Logic Hub)
```
src/core/controller/
├── index.ts              # Main controller class
├── state/
│   ├── updateSettings.ts
│   └── state-helpers.ts
├── task/
│   ├── newTask.ts
│   ├── getTaskHistory.ts
│   └── resumeTask.ts
└── ui/
    └── various event handlers
```

**Responsibility**: Orchestrate everything, manage state, coordinate between UI and Task.

```typescript
export class Controller {
  task?: Task
  stateManager: StateManager
  mcpHub: McpHub
  
  // Single source of truth for all state
  async initTask(userContent: string, images?: string[]) {
    this.task = new Task(this)
    return this.task.run()
  }
  
  // Handle webview messages
  async handleMessage(message: WebviewInboundMessage) {
    switch (message.type) {
      case "send_message":
        await this.task?.handleUserMessage(message)
        break
      case "abort":
        await this.task?.abort()
        break
      // ... more handlers
    }
  }
}
```

### Layer 4: Task Executor (Core AI Loop)
```
src/core/task/
├── index.ts              # Main task loop
├── TaskLoop.ts           # Streaming loop
├── tools/
│   ├── handlers/
│   └── executor.ts
├── api/
│   ├── transform/
│   └── stream-handler.ts
└── messaging/
    └── parse-assistant-message.ts
```

**Responsibility**: Execute AI requests, handle tool calls, stream responses.

```typescript
export class Task {
  async initiateTaskLoop(userMessage: string) {
    while (!this.abort) {
      // 1. Stream API response
      const stream = this.attemptApiRequest()
      
      // 2. Parse and present as deltas
      for await (const chunk of stream) {
        this.assistantMessageContent = parseAssistantMessage(chunk)
        await this.presentAssistantMessage() // Send delta to UI
      }
      
      // 3. Extract tool calls
      const toolCalls = extractToolCalls(this.assistantMessageContent)
      
      // 4. Execute tools
      const toolResults = await this.executeTools(toolCalls)
      
      // 5. Continue loop with results
      const didEndLoop = await this.recursivelyMakeClineRequests(toolResults)
      if (didEndLoop) break
    }
  }
}
```

---

## Key Implementation Patterns

### 1. Streaming Architecture (Speed Trick #1)

**Instead of:**
```typescript
const response = await api.complete(messages)
return response
```

**Use:**
```typescript
const stream = api.streamComplete(messages)

for await (const chunk of stream) {
  // Process each chunk immediately
  const delta = parse(chunk)
  
  // Send to UI immediately
  await ui.postMessage({
    type: "assistant_delta",
    text: delta.text
  })
  
  // Update local state for tool call extraction
  this.buffer += delta.text
}

// Extract tools from buffer when complete
const tools = extractTools(this.buffer)
```

**Result**: UI updates appear in real-time, not after completion.

### 2. Smart Context Window Management (Speed Trick #2)

```typescript
// File: src/core/context/ContextManager.ts
export class ContextManager {
  // Never load entire files
  async readRelevantSections(
    filePath: string,
    startLine: number,
    endLine: number
  ) {
    return this.readFile(filePath, startLine, endLine)
  }
  
  // Automatic chunking for large files
  async prepareFileForContext(filePath: string) {
    const content = await fs.readFile(filePath, 'utf-8')
    const lineCount = content.split('\n').length
    
    // If > 5000 lines, send summary + current context
    if (lineCount > 5000) {
      return {
        summary: await generateFileSummary(filePath),
        currentContext: content.slice(0, 50000) // First 50k chars
      }
    }
    return content
  }
  
  // Track context usage
  trackTokens(filePath: string, tokens: number) {
    this.contextUsage[filePath] = tokens
  }
  
  // Prioritize files by relevance
  getSortedFilesByRelevance() {
    return Object.entries(this.contextUsage)
      .sort((a, b) => b[1] - a[1])
  }
}
```

### 3. Session Persistence & Checkpointing (Speed Trick #3)

```typescript
// File: src/core/checkpoint/CheckpointManager.ts
export class CheckpointManager {
  async createCheckpoint(task: Task) {
    const checkpoint = {
      id: generateId(),
      timestamp: Date.now(),
      state: {
        messages: task.messages,
        context: task.contextState,
        fileEdits: task.pendingEdits,
        workspaceState: await this.captureWorkspace()
      }
    }
    
    // Save to disk
    await this.store.save(checkpoint)
    return checkpoint.id
  }
  
  async restoreCheckpoint(id: string) {
    const checkpoint = await this.store.get(id)
    
    // Restore git state
    await git.checkout(checkpoint.state.workspaceState.gitRef)
    
    // Restore task state
    return {
      messages: checkpoint.state.messages,
      context: checkpoint.state.context
    }
  }
}
```

### 4. Tool Execution Framework (Speed Trick #4)

```typescript
// File: src/core/task/tools/ToolExecutor.ts
export class ToolExecutor {
  private handlers = new Map<string, ToolHandler>()
  
  registerTool(tool: Tool) {
    this.handlers.set(tool.name, {
      execute: tool.execute,
      schema: tool.inputSchema
    })
  }
  
  // Execute tools in parallel when possible
  async executeTools(toolCalls: ToolCall[]) {
    const results = await Promise.all(
      toolCalls.map(async (call) => {
        const handler = this.handlers.get(call.name)
        if (!handler) throw new Error(`Unknown tool: ${call.name}`)
        
        try {
          const result = await handler.execute(call.input)
          
          // Stream progress to UI
          await this.ui.postMessage({
            type: "tool_completed",
            toolName: call.name,
            output: result
          })
          
          return {
            toolCallId: call.id,
            output: result,
            isError: false
          }
        } catch (error) {
          return {
            toolCallId: call.id,
            output: error.message,
            isError: true
          }
        }
      })
    )
    
    return results
  }
}
```

### 5. Provider Abstraction (Speed Trick #5)

```typescript
// File: src/core/api/providers/ApiHandler.ts
export interface ApiHandler {
  complete(messages: Message[], options: ApiOptions): AsyncGenerator<string>
  getModel(): { id: string; info: ModelInfo }
}

export class ApiGateway {
  private handlers = new Map<string, ApiHandler>()
  
  registerHandler(provider: string, handler: ApiHandler) {
    this.handlers.set(provider, handler)
  }
  
  async stream(
    provider: string,
    messages: Message[],
    options: ApiOptions
  ) {
    const handler = this.handlers.get(provider)
    if (!handler) throw new Error(`Unknown provider: ${provider}`)
    
    // Handler decides how to stream (OpenAI format, Anthropic format, etc.)
    return handler.complete(messages, options)
  }
}

// Providers implement their own format
export class AnthropicHandler implements ApiHandler {
  async *complete(messages: Message[], options: ApiOptions) {
    const stream = await anthropic.messages.create({
      stream: true,
      messages: convertToAnthropicFormat(messages),
      // ...
    })
    
    for await (const event of stream) {
      if (event.type === 'content_block_delta') {
        yield event.delta.text
      }
    }
  }
}

export class OpenAIHandler implements ApiHandler {
  async *complete(messages: Message[], options: ApiOptions) {
    const stream = await openai.chat.completions.create({
      stream: true,
      messages: convertToOpenAIFormat(messages),
      // ...
    })
    
    for await (const chunk of stream) {
      yield chunk.choices[0].delta.content || ''
    }
  }
}
```

### 6. Non-Blocking Concurrency Pattern (Speed Trick #6)

```typescript
// File: src/core/task/TaskLoop.ts
export class TaskLoop {
  private userMessageReady = new Promise<void>()
  private resolveUserMessage?: () => void
  
  async runLoop(initialMessage: string) {
    let userContent = initialMessage
    
    while (!this.shouldStop) {
      // 1. API request (non-blocking)
      const streamPromise = this.apiManager.stream(userContent)
      
      // 2. Parse response (non-blocking)
      const parsePromise = this.parseStream(streamPromise)
      
      // 3. Wait for user interaction OR tool execution (whichever comes first)
      const [assistantMessage, userFeedback] = await Promise.race([
        parsePromise,
        this.waitForUserInput()
      ])
      
      if (userFeedback) {
        userContent = userFeedback
        continue
      }
      
      // 4. Execute tools in background
      const toolResults = await this.executeTools(assistantMessage.toolCalls)
      userContent = toolResults
    }
  }
  
  // UI can interrupt anytime without blocking
  async receiveUserMessage(message: string) {
    this.userMessage = message
    this.resolveUserMessage?.()
  }
}
```

### 7. State Management (Speed Trick #7)

```typescript
// File: src/core/state/StateManager.ts
export class StateManager {
  // In-memory cache (populated on initialization)
  private cache = new Map<string, any>()
  
  async initialize(context: ExtensionContext) {
    // Load state once
    this.cache.set('settings', context.globalState.get('settings'))
    this.cache.set('apiKeys', context.secrets.get('apiKeys'))
    this.cache.set('taskHistory', await readTaskHistoryFile())
  }
  
  // Fast lookups
  getGlobalStateKey<T>(key: string): T {
    return this.cache.get(key)
  }
  
  // Async updates
  async setGlobalState(key: string, value: any) {
    this.cache.set(key, value)
    
    // Persist in background (don't block)
    await this.persistAsync(key, value)
  }
  
  // Cross-window synchronization
  private persistAsync(key: string, value: any) {
    // Write to disk (file-based for cross-window consistency)
    return fs.writeFile(
      path.join(this.dataDir, `${key}.json`),
      JSON.stringify(value)
    )
  }
}
```

---

## Implementation Checklist

### Phase 1: Core Architecture
- [ ] Create `Controller` class (single source of truth)
- [ ] Create `Task` class (execution loop)
- [ ] Implement streaming API gateway
- [ ] Setup message streaming to UI

### Phase 2: Performance
- [ ] Add context chunking
- [ ] Implement checkpoint system
- [ ] Add parallel tool execution
- [ ] Setup non-blocking concurrency

### Phase 3: Provider Support
- [ ] Anthropic provider
- [ ] OpenAI provider
- [ ] Google provider
- [ ] OpenRouter fallback

### Phase 4: Advanced Features
- [ ] MCP server support
- [ ] Custom tool registration
- [ ] Plugin system
- [ ] Cross-window state sync

### Phase 5: IDE Integration
- [ ] VS Code extension
- [ ] CLI support
- [ ] JetBrains plugin

---

## Performance Metrics

Track these to verify your implementation matches Cline:

```typescript
interface PerformanceMetrics {
  // Time to first token
  ttft: number
  
  // Context loading time
  contextLoadTime: number
  
  // Tool execution parallelization
  maxConcurrentTools: number
  
  // File read efficiency (lines per ms)
  fileReadSpeed: number
  
  // Streaming chunk size (bytes)
  avgChunkSize: number
}
```

---

## File Structure Template

```
src/
├── common.ts                    # Shared initialization
├── hosts/
│   ├── host-provider.ts         # Provider interface
│   ├── vscode/
│   │   ├── VscodeWebviewProvider.ts
│   │   └── vscode-specific.ts
│   └── cli/
│       └── cli-specific.ts
├── core/
│   ├── webview/
│   │   ├── index.ts
│   │   └── protocol.ts
│   ├── controller/
│   │   ├── index.ts
│   │   ├── state/
│   │   └── task/
│   ├── task/
│   │   ├── index.ts
│   │   ├── TaskLoop.ts
│   │   ├── tools/
│   │   ├── api/
│   │   └── messaging/
│   ├── api/
│   │   ├── providers/
│   │   │   ├── anthropic.ts
│   │   │   ├── openai.ts
│   │   │   └── base.ts
│   │   └── gateway.ts
│   ├── checkpoint/
│   │   └── CheckpointManager.ts
│   └── context/
│       └── ContextManager.ts
├── services/
│   ├── mcp/
│   ├── auth/
│   └── telemetry/
├── shared/
│   ├── types.ts
│   ├── api.ts
│   └── tools.ts
└── utils/
    ├── fs.ts
    └── path.ts
```

---

## Key Takeaways

1. **Streaming First**: Every response should be streamed, not batched
2. **Layered Abstraction**: Clear boundaries between UI, business logic, and execution
3. **State Centralization**: One Controller manages all state
4. **Provider Agnostic**: Support multiple LLM providers via interfaces
5. **Tool Framework**: Generic tool execution with parallel support
6. **Persistence**: Checkpoint every decision for recovery
7. **Non-blocking**: Async everywhere, never block the UI
8. **Observable**: Every action should emit events for logging/debugging

This architecture lets you scale from a simple CLI tool to a multi-IDE agent platform.
