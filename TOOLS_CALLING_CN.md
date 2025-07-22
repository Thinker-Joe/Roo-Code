# Roo Code 工具调用核心实现（含伪代码）

本文档详细介绍 Roo Code 在 AI 交互过程中如何发现并调用工具，并给出关键实现的源码片段或伪代码，帮助理解整个执行流程。

## 1. 工具枚举与类型

所有可用工具在 `packages/types/src/tool.ts` 中枚举并以 `zod` 校验：
```ts
export const toolNames = [
  "execute_command",
  "read_file",
  "write_to_file",
  "apply_diff",
  "insert_content",
  "search_and_replace",
  "search_files",
  "list_files",
  "list_code_definition_names",
  "browser_action",
  "use_mcp_tool",
  "access_mcp_resource",
  "ask_followup_question",
  "attempt_completion",
  "switch_mode",
  "new_task",
  "fetch_instructions",
  "codebase_search",
  "update_todo_list",
] as const
```

工具参数及 `ToolUse` 结构体在 `src/shared/tools.ts` 中定义，供解析与校验使用。

## 2. 工具分组与显示名称

`src/shared/tools.ts` 内部定义 `TOOL_DISPLAY_NAMES`、`TOOL_GROUPS` 和 `ALWAYS_AVAILABLE_TOOLS`：
```ts
export const TOOL_DISPLAY_NAMES: Record<ToolName, string> = {
  execute_command: "run commands",
  read_file: "read files",
  /* ... */
}
export const TOOL_GROUPS = {
  read: { tools: ["read_file", "fetch_instructions", /* ... */] },
  edit: { tools: ["apply_diff", "write_to_file", "insert_content", "search_and_replace"] },
  command: { tools: ["execute_command"] },
  /* ... */
}
export const ALWAYS_AVAILABLE_TOOLS: ToolName[] = [
  "ask_followup_question",
  "attempt_completion",
  "switch_mode",
  "new_task",
  "update_todo_list",
]
```

## 3. 系统提示的生成

系统提示在 `src/core/prompts/system.ts` 的 `generatePrompt` 函数中拼装，伪代码如下：
```ts
function generatePrompt(mode, cwd, supportsComputerUse, ...): string {
  const modeConfig = getModeBySlug(mode)
  const roleDefinition = getModeSelection(mode).roleDefinition
  const toolSection = getToolDescriptionsForMode(mode, cwd, supportsComputerUse, ...)
  const guidelines = getToolUseGuidelinesSection()
  return `${roleDefinition}
${markdownFormattingSection()}
${getSharedToolUseSection()}
${toolSection}
${guidelines}
...`
}
```
`getToolDescriptionsForMode` 根据模式收集允许的工具描述：
```ts
function getToolDescriptionsForMode(mode, cwd, supportsComputerUse, ...): string {
  const config = getModeConfig(mode)
  const tools = new Set<string>()
  config.groups.forEach(group => {
    TOOL_GROUPS[getGroupName(group)].tools.forEach(tool => {
      if (isToolAllowedForMode(tool, mode, ...)) tools.add(tool)
    })
  })
  ALWAYS_AVAILABLE_TOOLS.forEach(t => tools.add(t))
  const descriptions = Array.from(tools).map(t => toolDescriptionMap[t](args))
  return `# Tools\n\n${descriptions.filter(Boolean).join("\n\n")}`
}
```
`getSharedToolUseSection` 指出工具调用的 XML 格式：
```ts
export function getSharedToolUseSection(): string {
  return `====
TOOL USE

<tool_name>
  <param>value</param>
</tool_name>`
}
```

## 4. 解析助手输出

`parseAssistantMessageV2` 将模型文本解析为文本和工具块。核心逻辑伪代码：
```ts
function parseAssistantMessageV2(text): AssistantMessageContent[] {
  for each char i in text:
    if insideParam:                // 正在解析参数值
      if closeTagFound(i): storeValue(); continue
      else continue
    if insideTool:                 // 在工具块内
      if startOfParam(i): enterParam(); continue
      if toolCloseFound(i): finalizeTool(); continue
      else continue
    // 普通文本区域，检查是否遇到新的工具标签
    if startOfTool(i): finalizeText(); startTool(); continue
    else ensureTextBlock()
  finalizeAnyOpenBlocks()
  return contentBlocks
}
```

## 5. 消息呈现与工具执行

`Task` 在接收流式响应时解析消息并调用 `presentAssistantMessage`：
```ts
for await (const chunk of stream) {
  assistantMessage += chunk.text
  assistantContent = parseAssistantMessage(assistantMessage)
  presentAssistantMessage(task)
}
```
`presentAssistantMessage` 伪代码如下：
```ts
async function presentAssistantMessage(cline: Task) {
  const block = cline.assistantMessageContent[cline.currentStreamingContentIndex]
  if (block.type === 'text') {
    cline.userMessageContent.push(block)
  } else {
    validateToolUse(block.name, mode, ...)
    if (toolRepetitionDetector.check(block).allowExecution) {
      await dispatchTool(block)
    } else {
      pushToolResult(toolError('repetition limit'))
    }
  }
  // 处理下一个块或等待流式输入完成
}
```
其中 `dispatchTool` 根据工具名调用 `readFileTool`、`writeToFileTool` 等实现。

## 6. 校验与重复检测

工具调用前通过 `validateToolUse` 确保在当前模式可用：
```ts
function validateToolUse(toolName: ToolName, mode: Mode, ...): void {
  if (!isToolAllowedForMode(toolName, mode, ...)) {
    throw new Error(`Tool "${toolName}" is not allowed in ${mode} mode.`)
  }
}
```
`ToolRepetitionDetector` 检测连续重复调用：
```ts
class ToolRepetitionDetector {
  previous: string | null = null
  count = 0
  limit = 3
  check(call: ToolUse) {
    const json = JSON.stringify(call)
    if (json === this.previous) this.count++
    else { this.count = 1; this.previous = json }
    if (this.count >= this.limit) {
      this.count = 0; this.previous = null
      return { allowExecution: false }
    }
    return { allowExecution: true }
  }
}
```

## 7. 工具实现示例

以 `readFileTool` 为例，伪代码如下：
```ts
async function readFileTool(cline, block) {
  const paths = parseXML(block.params.args)
  if (block.partial) {
    await cline.ask('tool', JSON.stringify({ tool: 'readFile', path: paths[0] }), true)
    return
  }
  const results = paths.map(p => readFileFromWorkspace(p))
  cline.pushToolResult(`<files>\n${results.join('\n')}\n</files>`)
}
```

## 8. 工具使用统计

`Task` 记录工具调用次数及失败次数：
```ts
recordToolUsage(name: ToolName) {
  if (!this.toolUsage[name]) this.toolUsage[name] = { attempts: 0, failures: 0 }
  this.toolUsage[name].attempts++
}
recordToolError(name: ToolName) {
  if (!this.toolUsage[name]) this.toolUsage[name] = { attempts: 0, failures: 0 }
  this.toolUsage[name].failures++
}
```

---

综合来看，Roo Code 通过在系统提示中列出工具及其示例，引导模型使用 XML 格式调用工具。模型输出经 `parseAssistantMessageV2` 解析后，`presentAssistantMessage` 依次校验并执行每个工具，同时记录调用统计并防止循环调用，形成安全、逐步的工具交互流程。
