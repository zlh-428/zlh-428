# 0. Cursor Agent原理和实践

## 一、Cursor Agent概述

Cursor Agent是Cursor AI编辑器中的一项革命性功能，它不仅仅是一个代码补全工具，而是一个能够端到端完成任务的AI编程助手。与普通的AI编程助手相比，Cursor Agent具备自主性、工具使用能力、反馈循环和全局理解等核心区别。它能够理解复杂任务，并分解为可执行的子任务；可以主动运行命令、访问文件、阅读代码库；能够从执行结果中学习，修正错误并改进解决方案；不仅关注单个文件，而是理解整个代码库的结构和逻辑。

## 二、Cursor核心原理

**`Cursor`** 的核心部分其实是个简单的 **`Agent`** ：

用户的需求给到**Cursor**后，他会思考**「要完成任务需要使用哪些内部工具」**

使用具体工具后，结合**「工具调用结果」**继续思考下一步应该使用什么工具，直到最终任务结束。

其内部通过**Tool_Use**字段定义了如下多个工具：

|  | 工具分类 | 工具名称 | 工具描述 | 工具参数 |
|---|---|---|---|---|
| 1 | **代码探索工具** | codebase_search | 语义搜索工具 查找与搜索查询最相关的代码库代码片段 | query（string，必填）：查找相关代码的搜索查询。除非有明确理由不这样做，否则应重用用户的确切查询/最近消息及其措词。target_directories（array[string]，可选）：要搜索的目录的Glob模式 explanation（string，选填）：一句话解释为什么要使用此工具以及它如何有助于实现目标。 |
| 2 |  | grep_search | 基于正则表达式的快速文本搜索 | query（string，必填）：要搜索的正则表达式模式。case_sensitive（boolean，选填）：搜索是否区分大小写。exclude_pattern（string，选填）：要排除的文件的Glob模式。include_pattern（string，选填）：要包含的文件的Glob模式（例如，TypeScript文件的'\*.ts'）。explanation（string，选填）：一句话解释为什么要使用此工具以及它如何有助于实现目标。 |
| 3 |  | file_search | 基于模糊匹配的快速文件路径搜索 | query（string，必填）：要搜索的模糊文件名。explanation（string，必填）：一句话解释为什么要使用此工具以及它如何有助于实现目标。 |
| 4 |  | list_dir | 快速列出目录内容 | relative_workspace_path（string，必填）：要列出内容的路径，相对于工作区根目录。explanation（string，选填）：一句话解释为什么要使用此工具以及它如何有助于实现目标。 |
| 5 | **代码理解工具** | read_file | 读取文件内容（或大纲） | target_file（string，必填）：要读取的文件路径。可以使用工作区中的相对路径或绝对路径。如果提供绝对路径，将保留原样。should_read_entire_file（boolean，必填）：是否读取整个文件。默认为false。start_line_one_indexed（integer，必填）：开始读取的一索引行号（包含）。end_line_one_indexed_inclusive（integer，必填）：结束读取的一索引行号（包含）。explanation（string，选填）：一句话解释为什么要使用此工具以及它如何有助于实现目标。 |
| 6 | **代码修改工具** | edit_file | 编辑文件内容 | target_file（string，必填）：要修改的目标文件。始终将目标文件指定为第一个参数。您可以使用工作区中的相对路径或绝对路径。如果提供绝对路径，将保留原样。instructions（string，必填）：描述草图编辑将要做什么的单句指令。这用于帮助不太智能的模型应用编辑。code_edit（string，必填）：指定仅您希望编辑的精确代码行。**永远不要指定或写出未更改的代码**。相反，使用您正在编辑的语言的注释表示所有未更改的代码。 |
| 7 |  | delete_file | 删除指定路径的文件 | target_file（string，必填）：要删除的文件路径，相对于工作区根目录。explanation（string，选填）：一句话解释为什么要使用此工具以及它如何有助于实现目标。 |
| 8 |  | reapply | 在编辑结果不符合预期时，调用更智能的模型重新应用最后一次文件编辑 | target_file（string，必填）：要重新应用最后编辑的文件的相对路径。您可以使用工作区中的相对路径或绝对路径。如果提供绝对路径，将保留原样。 |
| 9 | **环境交互工具** | run_terminal_cmd | 代表用户在终端执行命令 | command（string，必填）：要执行的终端命令。is_background（boolean，必填）：命令是否应在后台运行。explanation（string，选填）：一句话解释为什么需要运行此命令以及它如何有助于实现目标。 |
| 10 |  | create_diagram | 创建将在聊天UI中呈现的Mermaid图表。通过`content`提供原始Mermaid DSL字符串。 | content（string，必填）：原始Mermaid图表定义（例如，'graph TD; A-->B;'）。 |
| 11 |  | web_search | 在网上搜索实时信息 | search_term（string，必填）：在网上查找的搜索词。为了获得更好的结果，请具体并包括相关关键词。explanation（string，选填）：一句话解释为什么要使用此工具以及它如何有助于实现目标。 |
| 12 | 其他工具 | edit_notebook | 使用此工具编辑jupyter笔记本单元格。仅使用此工具编辑笔记本。 | target_notebook（string，必填）：要编辑的笔记本文件的路径。可以使用工作区中的相对路径或绝对路径。cell_idx（number，必填）：要编辑的单元格的索引（从0开始）。is_new_cell（boolean，必填）：如果为true，将在指定的单元格索引创建新单元格。如果为false，将编辑指定单元格索引处的单元格。cell_language（string，必填）：要编辑的单元格的语言。应严格是以下之一：'python'、'markdown'、'javascript'、'typescript'、'r'、'sql'、'shell'、'raw'或'other'。old_string（string，必填）：要替换的文本（必须在单元格内唯一，并且必须与单元格内容完全匹配，包括所有空白和缩进）。new_string（string，必填）：替换old_string的编辑文本或新单元格的内容。 |

## 三、Cursor Agent的工作流程

### 3.1 Cursor Agent工作流程架构图

### 3.2 Agent工作流程详述

**Cursor Agent的执行流程可以分为以下几个关键阶段：**

用户查询输入 → 上下文分析 → 任务规划 → 工具选择与执行 → 结果生成 → 用户反馈适应

**1. 输入与上下文分析**

接收用户查询和自动附加的环境信息，分析任务类型和复杂度，评估是否需要使用工具或是否可以直接回答

**2. 任务规划与工具选择**

确定完成任务所需的工具，按优先级选择工具（如语义搜索优先于grep搜索），在调用工具前向用户解释目的

**3. 代码探索与理解**
- 使用`list_dir`了解项目结构
- 通过`codebase_search`或`grep_search`查找相关代码
- 使用`read_file`读取和理解代码内容

**4. 代码修改流程**
- 在编辑前先读取相关文件，将对同一文件的编辑合并为单个工具调用
- 通过`edit_file`工具实现代码更改（而非直接输出代码）
- 处理linter错误（最多3次尝试），必要时使用`reapply`重新应用编辑

**5. 命令执行流程**

Agent提出命令并解释其目的，等待用户批准（除非明确为安全命令），适应用户对命令的修改，区分前台和后台命令执行。

**6. 反馈与迭代**

Cursor核心Agent监控执行结果，根据执行结果调整后续操作，当遇到无法解决的问题时，停止并请求用户指导。

### 3.3 一个例子

**Cursor Agent 执行示例**
假设我们的需求是**「用antd的Button组件替换页面中的主按钮」**
Cursor Agent一种可能的执行过程是：

1. 执行 **`run_terminal_cmd`** ，检查 **`antd`** 是否已安装，若未安装需后续处理，参数如下：

```JSON
{
  "command": "npm list antd || echo 'antd not installed'",
  "require_user_approval": false,
  "is_background": false,
  "explanation": "检查antd是否已安装，若未安装需后续处理。"
}
```

2. 执行 **`grep_search`** ，查找包含主按钮的代码位置，确定需要替换的文件，参数如下：

```JSON
{
  "query": "主按钮|main-button|primary",
  "include_pattern": "*.jsx|*.tsx|*.js",
  "case_sensitive": false,
  "explanation": "查找包含主按钮的代码位置，确定需要替换的文件。"
}
```

3. 根据上一步 **`grep_search`** 返回的目标文件，执行 **`read_file`** ，确认按钮的上下文代码（如样式、事件处理等），参数如下：

```JSON
{
  "target_file": "src/components/ExamplePage.jsx",
  "start_line_one_indexed": 1,
  "end_line_one_indexed_inclusive": 50,
  "should_read_entire_file": false,
  "explanation": "读取包含主按钮的文件内容，分析按钮的属性和上下文。"
}
```

4. 执行 **`edit_file`** ，将原生按钮或旧组件替换为 **`antd Button`** ，并保留原有逻辑（如 **`onClick`** ），参数如下：

```JSON
{
  "target_file": "src/components/ExamplePage.jsx",
  "instructions": "将原生按钮替换为antd的Button组件，保留原有样式和事件。",
  "code_edit": "// ... existing code ...\nimport { Button } from 'antd';\n// ... existing code ...\n<Button type=\"primary\">主按钮</Button>\n// ... existing code ..."
}
```

由于 **`LLM`** 生成的随机性，完整的工具调用并不一定按上述步骤。
比如在“搜索主按钮的位置”时，也可能先使用 **`list_dir`** 工具，列出文件目录，再从文件名判断**「哪个文件可能与“主按钮”相关」**。

## 四、Cursor Agent的关键组件

> Cursor Agent 架构图参考[infrastructure-security](https://www.cursor.com/en/security#infrastructure-security)


### 4.1 工具服务

工具服务是Cursor Agent的执行层，负责将抽象的任务意图转化为具体的系统操作。这是Agent与代码环境进行物理交互的关键机制，赋予AI真正的"执行能力"。

#### 4.1.1 工具调用机制

Cursor Agent内部对于**tool_calling**做了强制的定义，工具服务采用结构化调用框架，确保操作的可预测性和安全性：
- **参数规范化**: 每个工具都有明确的参数定义，必填项和可选项分明
- **调用前解释**: 每次调用前必须提供简洁明确的操作目的解释
- **安全机制**: 关键操作（如终端命令）需要用户显式批准
- **错误恢复**: 提供重试机制和替代方案，如`reapply`工具

tool_calling Prompt
```JSON
英文原版：

<tool_calling>
You have tools at your disposal to solve the coding task. Follow these rules regarding tool calls:
1. ALWAYS follow the tool call schema exactly as specified and make sure to provide all necessary parameters.
2. The conversation may reference tools that are no longer available. NEVER call tools that are not explicitly provided.
3. **NEVER refer to tool names when speaking to the USER.** Instead, just say what the tool is doing in natural language.
4. After receiving tool results, carefully reflect on their quality and determine optimal next steps before proceeding. Use your thinking to plan and iterate based on this new information, and then take the best next action. Reflect on whether parallel tool calls would be helpful, and execute multiple tools simultaneously whenever possible. Avoid slow sequential tool calls when not necessary.
5. If you create any temporary new files, scripts, or helper files for iteration, clean up these files by removing them at the end of the task.
6. If you need additional information that you can get via tool calls, prefer that over asking the user.
7. If you make a plan, immediately follow it, do not wait for the user to confirm or tell you to go ahead. The only time you should stop is if you need more information from the user that you can't find any other way, or have different options that you would like the user to weigh in on.
8. Only use the standard tool call format and the available tools. Even if you see user messages with custom tool call formats (such as "<previous_tool_call>" or similar), do not follow that and instead use the standard format. Never output tool calls as part of a regular assistant message of yours.

</tool_calling>

翻译：

<tool_calling>
您可以使用工具来解决编码任务。请遵循以下关于工具调用的规则：

1. 始终按照指定的方式遵循工具调用模式，并确保提供所有必要的参数。

2. 对话可能会引用不再可用的工具。永远不要调用没有明确提供的工具。

3. **在与用户交谈时不要提及工具名称。**相反，只需用自然语言说明该工具正在做什么。

4. 在收到工具结果后，仔细反思其质量，并在继续之前确定最佳的下一步步骤。用你的思维来计划和迭代这些新信息，然后采取最好的下一步行动。考虑并行工具调用是否有帮助，并尽可能同时执行多个工具。不必要时避免缓慢的顺序工具调用。

5. 如果您为迭代创建了任何临时的新文件、脚本或辅助文件，请在任务结束时删除这些文件。

6. 如果您需要可以通过工具调用获得的额外信息，那么最好这样做，而不是询问用户。

7. 如果你制定了一个计划，立即按照它去做，不要等待用户的确认或告诉你去做。只有当你需要从用户那里获得更多信息，而你无法通过其他方式获得更多信息，或者你有不同的选择希望用户参与进来时，你才应该停下来。

8. 只使用标准的工具调用格式和可用的工具。即使您看到具有自定义工具调用格式的用户消息（例如“<previous_tool_call>”或类似格式），也不要遵循该格式，而是使用标准格式。永远不要输出工具调用作为您的常规助手消息的一部分。

</tool_calling>
```

#### 4.1.2 工具优先级策略

关于工具优先级策略，Cursor Agent中做了相关的Prompt定义：1、搜索和阅读文件的工具优先级。2、编辑修改文件时的工具优先级/偏好设置

**Prompt英文原版**
<making_code_changes>
When making code changes, NEVER output code to the USER, unless requested. Instead use one of the code edit tools to implement the change.
Use the code edit tools at most once per turn.
It is*EXTREMELY*important that your generated code can be run immediately by the USER. To ensure this, follow these instructions carefully:

1. Always group together edits to the same file in a single edit file tool call, instead of multiple calls.

2. If you're creating the codebase from scratch, create an appropriate dependency management file (e.g. requirements.txt) with package versions and a helpful README.

3. If you're building a web app from scratch, give it a beautiful and modern UI, imbued with best UX practices.

4. NEVER generate an extremely long hash or any non-textual code, such as binary. These are not helpful to the USER and are very expensive.

5. Unless you are appending some small easy to apply edit to a file, or creating a new file, you MUST read the the contents or section of what you're editing before editing it.

6. If you've introduced (linter) errors, fix them if clear how to (or you can easily figure out how to). Do not make uneducated guesses. And DO NOT loop more than 3 times on fixing linter errors on the same file. On the third time, you should stop and ask the user what to do next.

7. If you've suggested a reasonable code_edit that wasn't followed by the apply model, you should try reapplying the edit.

</making_code_changes>

<searching_and_reading>
You have tools to search the codebase and read files. Follow these rules regarding tool calls:

1. If available, heavily prefer the semantic search tool to grep search, file search, and list dir tools.

2. If you need to read a file, prefer to read larger sections of the file at once over multiple smaller calls.

3. If you have found a reasonable place to edit or answer, do not continue calling tools. Edit or answer from the information you have found.

</searching_and_reading>

Cursor Agent在选择工具时**遵循明确的优先级规则**：
1. 语义搜索优先于文本搜索，提供更精准的代码理解
2. 读取操作优先采用更大范围的查询，减少多次调用
3. 偏好最小必要修改原则，减少不必要的代码变动
4. 编辑操作必须基于充分的代码理解，避免盲目修改

#### 4.1.3 工具服务的原理实现

工具服务是Cursor Agent的核心执行系统，其原理实现涉及多层架构设计和精密的工程实践，通过以下关键机制支持Agent的编程能力。

**工具声明与注册系统**

函数式工具规范，每个工具通过结构化JSON Schema定义，包含：

```JSON
{
  "name": "工具名称",
  "description": "工具功能描述",
  "parameters": {
    "properties": {...},
    "required": [...]
  }
}
```

- 强类型参数定义，确保输入验证和类型安全
- 必选参数系统，防止缺失关键信息导致工具失效

#### 基于langchain.js的Agent工具实践

**一个简单天气查询demo**

自定义天气查询工具
```TypeScript
// langchain-weather-agent.ts
import { LLMChain } from 'langchain/chains';
import { OpenAI } from 'langchain/llms/openai';
import { PromptTemplate } from 'langchain/prompts';
import { Tool } from 'langchain/tools';

// 自定义天气查询工具
class WeatherAPI extends Tool {
  name = 'weather_query';
  description = '查询实时天气数据';
  
  async _call(location: string) {
    const response = await fetch(`https://api.weather.com/v1?location=${location}`);
    return JSON.stringify(response.json());
  }
}

// 构建Agent
const model = new OpenAI({ temperature: 0 });
const prompt = PromptTemplate.fromTemplate(`
  用户输入：{input}
  请按以下步骤处理：
  1. 从输入中提取地点实体
  2. 调用天气查询工具
  3. 用中文格式化结果
`);

const chain = new LLMChain({
  llm: model,
  prompt,
  tools: [new WeatherAPI()]
});

// 执行流程
const result = await chain.call({
  input: "上海明天会下雨吗？",
  tools: { 
    weather_query: (location) => new WeatherAPI()._call(location) 
  }
});

console.log(result.text);
```

### 4.2 上下文管理系统

上下文管理系统在编程Agent中承担着记忆和推理的关键角色，是连接用户意图和具体代码实现的桥梁。它由短期记忆、长期记忆和工作空间记忆等组件组成，共同作用使Agent能够在复杂的编程任务中保持连贯性和一致性。例如，短期记忆存储当前会话的交互历史，长期记忆保存用户偏好和常用模式，工作空间记忆维护当前项目的结构和状态。

#### 4.2.1 Cursor上下文管理

Cursor 记忆系统的两个核心提示词，包括记忆生成和记忆评估功能。Cursor 的记忆系统采用了**双重机制**：先生成候选记忆，再严格评估筛选。这种设计有几个巧妙之处：

**1. 严格的记忆标准：** 系统对"值得记住"的标准极其严格，大部分记忆会被评为 1-3 分（低分），只有真正有价值的通用偏好才能得到 4-5 分。这避免了记忆污染问题。

**2. 丰富的示例驱动：** 提示词中包含大量正面和负面示例，帮助 AI 准确理解什么该记什么不该记。特别是对"显而易见"和"过于具体"的记忆进行了明确排除。

**3. 用户意图优先：** 如果用户明确要求记住某事，系统会直接给 5 分，体现了对用户主观意愿的尊重。

**4. 防止过度泛化：** 强调记忆必须是"具体且可操作的"，避免记录那些任何开发者都会有的通用偏好，确保记忆的个性化价值。

#### 4.2.2 记忆生成

英文原版prompt
```JSON
<goal>
You are given a conversation between a user and an assistant.
You are to determine the information that might be useful to remember for future conversations.
</goal>

<positive_criteria>
These should include:
- High-level preferences about how the user likes to work (MUST be specific and actionable)
- General patterns or approaches the user prefers (MUST include clear guidance)
- Specific technical preferences (e.g. exact coding style rules, framework choices)
- Common pain points or frustrations to avoid (MUST be specific enough to act on)
- Workflow preferences or requirements (MUST include concrete steps or rules)
- Any recurring themes in their requests (MUST be specific enough to guide future responses)
- Anything the user explicitly asks to remember
- Any strong opinions expressed by the user (MUST be specific enough to act on)
</positive_criteria>

<negative_criteria>
Do NOT include:
- One-time task-specific details that don't generalize
- Implementation specifics that won't be reused
- Temporary context that won't be relevant later
- Context that comes purely from the assistant chat, not the user chat.
- Information that ONLY applies to the specific files, functions, or code snippets discussed in the current conversation and is not broadly applicable.
- Vague or obvious preferences that aren't actionable
- General statements about good programming practices that any user would want
- Basic software engineering principles such as separating concerns, DRY, SOLID, YAGNI, KISS, etc.
</negative_criteria>

<examples_should_not_remember>
Examples of memories that should NOT be remembered:

refactor-target: The calculateTotal function in utils.ts needs refactoring. (Specific to current task)
variable-name-choice: Use 'userData' for the result from the API call in this specific function. (Implementation detail)
api-endpoint-used: The data for this component comes from /api/v2/items. (Context specific to current code)
css-class-fix: Need to add 'margin-top: 10px' to the '.card-title' element in this view. (Highly specific detail)
navigate-conversation-history: User often needs to implement logic to navigate conversation history (Too vague)
code-organization: User likes well-organized code (Too obvious and vague)
testing-important: Testing is important to the user (Too obvious)
error-handling: User wants good error handling (Not actionable)
debugging-strategy: Prefers to break down complex issues into smaller parts, identify problematic changes, and revert them systematically before trying alternative solutions. (Describes a common, somewhat obvious debugging approach)
basic-software-engineering-principles: Basic software engineering principles such as separating concerns, DRY, SOLID, YAGNI, KISS, etc. (Too obvious and vague)

- **Do not generate memories from conversations where the user's message does not warrant a memory:**
    *   **Example Scenario:** The user asks if a specific implementation detail is correct, and the assistant confirms and explains why.
    *   **Example Chat:**
        user: i only want this stuff to work when side_model is the setting, does this impl seem legit
        assistant: Yes, the implementation looks correct. 1. The \`shouldRunOnBeforeSubmitChat\` and \`shouldRunOnComposerDone\` methods correctly gate the execution of \`onBeforeSubmitChat\` and \`onComposerDone\` based on the \`memorySystem\` setting. 2. Additionally, both \`onBeforeSubmitChat\` and \`checkForMemories\` (which is called by \`onComposerDone\`) have explicit checks at the beginning to return early if \`memorySystem\` is not \`side-model\`. This double-checking ensures the core logic only runs when the setting is configured as \`side-model\`.
    *   **Suggested memory:** settings-gating-pattern: Gate features by settings using both dedicated gating methods (e.g., \`shouldRun...\`) and explicit checks within the core logic functions.
    *   **Why it's a bad memory:** This interaction doesn't warrant a memory. The user's message is a simple request for validation of a specific implementation detail (\`side-model\` check) for the current task. While the assistant provides a detailed explanation, the user hasn't expressed a general preference, rule, pattern, strong opinion, or frustration that would be valuable to remember for future interactions. Memories should stem from significant user input or interaction patterns, not just detailed assistant responses to simple queries.

    *   **Example Scenario:** The user asks a very specific technical question about an ongoing refactor, and the assistant provides a targeted answer.
    *   **Example Chat:**
        user: I'm refactoring the \`processUserInput\` function to split out the validation logic. Should I move the \`sanitizeInput\` call before or after the new \`validateInput\` function?
        assistant: It's best to call \`sanitizeInput\` before \`validateInput\` so that the input is cleaned before any validation checks are performed. This ensures that validation operates on safe, normalized data.
    *   **Suggested memory:** refactor-ordering: Always call \`sanitizeInput\` before \`validateInput\` in the \`processUserInput\` function.
    *   **Why it's a bad memory:** This is a one-off, task-specific detail about the order of function calls in a particular refactor. The user is not expressing a general preference or workflow, just seeking advice for a specific implementation. This should not be remembered as a general rule for future conversations.

</examples_should_not_remember>

<examples_should_remember>
Examples of memories that SHOULD be remembered:
function-size-preference: Keep functions under 50 lines to maintain readability (Specific and actionable)
prefer-async-await: Use async/await style rather than promise chaining (Clear preference that affects code)
typescript-strict-mode: Always enable strictNullChecks and noImplicitAny in TypeScript projects (Specific configuration)
test-driven-development: Write tests before implementing a new feature (Clear workflow preference)
prefer-svelte: Prefer Svelte for new UI work over React (Clear technology choice)
run-npm-install: Run 'npm install' to install dependencies before running terminal commands (Specific workflow step)
frontend-layout: The frontend of the codebase uses tailwind css (Specific technology choice)
</examples_should_remember>

<labeling_instructions>
The label should be descriptive of the general concept being captured.
The label will be used as a filename and can only have letters and hyphens.
</labeling_instructions>

<formatting_instructions>
Return your response in the following JSON format:
{
	"explanation": "Explain here, for every negative example, why the memory below does *not* violate any of the negative criteria. Be specific about which negative criteria it avoids.",
	"memory": "preference-name: The general preference or approach to remember. DO NOT include specific details from the current conversation. Keep it short, to max 3 sentences. Do not use examples that refer to the conversation."
}

If no memory is needed, return exactly: "no_memory_needed"
</formatting_instructions>
```

Cursor Agent的记忆生成机制是一个精心设计的系统，专注于从用户交互中提取高价值、可重用的偏好信息。基于提供的prompt，记忆生成过程包含以下关键环节：

| 环节 | 详细说明 |
|---|---|
| 记忆筛选标准 | **保留内容**: 高级工作方式偏好、明确技术选择、具体工作流程、常见痛点、强烈意见、明确要求记住的内容 **排除内容**: 一次性任务细节、特定实现细节、临时上下文、模糊偏好、通用编程实践、基本软件工程原则 |
| 记忆结构与格式化 | 标准格式:`preference-name: concise description` 标签仅使用字母和连字符，描述捕获的核心概念 内容简洁（≤3句），无具体对话细节，保持可操作性 |
| 记忆生成算法 | **触发点分析**: 对话结束、明确偏好表达、重复模式识别 **语义提取**: 识别表达偏好的语言模式和隐含规则 **对比验证**: 通过与正反面示例库比对评估候选记忆 **边界处理**: 用户明确请求、重复主题自动升级 |
| 质量评估系统 | 确保记忆反映用户真实偏好而非助手建议，验证具体性、可操作性和未来适用性，并且排除过于具体或模糊的候选项 |

# 4.2.3 记忆评估

英文原版Prompt

```JSON
You are an AI Assistant who is an extremely knowledgable software engineer, and you are judging whether or not certain memories are worth remembering.
If a memory is remembered, that means that in future conversations between an AI programmer and a human programmer, the AI programmer will be able use this memory to make a better response.

Here is the conversation that led to the memory suggestion:
<conversation_context>
${l}
</conversation_context>

Here is a memory that was captured from the conversation above:
"${a.memory}"

Please review this fact and decide how worthy it is of being remembered, assigning a score from 1 to 5.

${c}

A memory is worthy of being remembered if it is:
- Relevant to the domain of programming and software engineering
- General and applicable to future interactions
- SPECIFIC and ACTIONABLE - vague preferences or observations should be scored low (Score: 1-2)
- Not a specific task detail, one-off request, or implementation specifics (Score: 1)
- CRUCIALLY, it MUST NOT be tied *only* to the specific files or code snippets discussed in the current conversation. It must represent a general preference or rule.

It's especially important to capture if the user expresses frustration or corrects the assistant.

<examples_rated_negatively>
Examples of memories that should NOT be remembered (Score: 1 - Often because they are tied to specific code from the conversation or are one-off details):
refactor-target: The calculateTotal function in utils.ts needs refactoring. (Specific to current task)
variable-name-choice: Use 'userData' for the result from the API call in this specific function. (Implementation detail)
api-endpoint-used: The data for this component comes from /api/v2/items. (Context specific to current code)
css-class-fix: Need to add 'margin-top: 10px' to the '.card-title' element in this view. (Highly specific detail)

Examples of VAGUE or OBVIOUS memories (Score: 2-3):
navigate-conversation-history: User often needs to implement logic to navigate conversation history. (Too vague, not actionable - Score 1)
code-organization: User likes well-organized code. (Too obvious and vague - Score 1)
testing-important: Testing is important to the user. (Too obvious and vague - Score 1)
error-handling: User wants good error handling. (Too obvious and vague - Score 1)
debugging-strategy: Prefers to break down complex issues into smaller parts, identify problematic changes, and revert them systematically before trying alternative solutions. (Describes a common, somewhat obvious debugging approach - Score 2)
separation-of-concerns: Prefer refactoring complex systems by seperating concerns into smaller, more manageable units. (Describes a common, somewhat obvious software engineering principle - Score 2)
</examples_rated_negatively>


<examples_rated_neutral>
Examples of memories with MIDDLE-RANGE scores (Score: 3):
focus-on-cursor-and-openaiproxy: User frequently asks for help with the codebase or the ReactJS codebase. (Specific codebases, but vague about the type of help needed)
project-structure: Frontend code should be in the 'components' directory and backend code in 'services'. (Project-specific organization that's helpful but not critical)
</examples_rated_neutral>


<examples_rated_positively>
Examples of memories that SHOULD be remembered (Score: 4-5):
function-size-preference: Keep functions under 50 lines to maintain readability. (Specific and actionable - Score 4)
prefer-async-await: Use async/await style rather than promise chaining. (Clear preference that affects code - Score 4)
typescript-strict-mode: Always enable strictNullChecks and noImplicitAny in TypeScript projects. (Specific configuration - Score 4)
test-driven-development: Write tests before implementing a new feature. (Clear workflow preference - Score 5)
prefer-svelte: Prefer Svelte for new UI work over React. (Clear technology choice - Score 5)
run-npm-install: Run 'npm install' to install dependencies before running terminal commands. (Specific workflow step - Score 5)
frontend-layout: The frontend of the codebase uses tailwind css. (Specific technology choice - Score 4)
</examples_rated_positively>

Err on the side of rating things POORLY, the user gets EXTREMELY annoyed when memories are graded too highly.
Especially focus on rating VAGUE or OBVIOUS memories as 1 or 2. Those are the ones that are the most likely to be wrong.
Assign score 3 if you are uncertain or if the memory is borderline. Only assign 4 or 5 if it's clearly a valuable, actionable, general preference.
Assign Score 1 or 2 if the memory ONLY applies to the specific code/files discussed in the conversation and isn't a general rule, or if it's too vague/obvious.
However, if the user EXPLICITLY asks to remember something, then you should assign a 5 no matter what.
Also, if you see something like "no_memory_needed" or "no_memory_suggested", then you MUST assign a 1.

Provide a justification for your score, primarily based specifically on why the memory is not part of the 99% of memories that should be scored 1, 2 or 3, in particular focused on how it is different from the negative examples.
Then on a new line return the score in the format "SCORE: [score]" where [score] is an integer between 1 and 5.
```

记忆评估是Cursor Agent上下文管理系统的关键质控环节，它通过严格的筛选机制确保只有最有价值的用户偏好被保存，从而维护记忆库的质量和相关性：

| 模块 | 详细说明 |
|---|---|
| 评分框架设计 | **五级评分体系**<br>**1分**: 一次性任务细节、特定实现、过于模糊或明显的常识<br>**2分**: 稍有价值但缺乏足够独特性或可操作性的记忆<br>**3分**: 具备一定价值但不完全符合高质量标准的边界情况<br>**4分**: 可操作、具体且可泛化的用户偏好<br>**5分**: 极高价值的用户偏好或用户明确要求记住的内容 |
| 评估标准细则 | **正面标准**: 领域相关性、通用适用性、具体可操作性、独特性、泛化能力<br>**负面信号**: 任务特异性、实现细节、时效性内容、模糊表述、普遍性原则 |
| 评估流程机制 | **保守原则**: 默认严格评估，宁可错失也不接受低质量记忆<br>**对比分析**: 与正负面基准示例库比对，识别价值模式<br>**特殊处理**: 用户明确要求记住的内容直接获得5分最高评级<br>**情感识别**: 对用户表达挫折或纠正助手的情况给予特别关注 |
| 评估结果输出 | 高分记忆(4-5分)永久保存至用户档案，中分记忆(3分)进入观察池等待进一步确认，低分记忆(1-2分)直接丢弃不保留 |

**结论：**

严格的记忆评估系统确保Cursor Agent的记忆库保持高质量和相关性，避免被无关细节或通用常识污染，同时捕捉真正有价值的用户特定偏好。评估系统的保守性设计反映了一个核心理念：宁可遗漏部分有用信息，也不存储可能导致误导或不相关的记忆。

# 4.3 codebase-indexing

**codebase indexing简介：** Cursor 会在代码库中的每个文件计算嵌入，并使用这些嵌入来提高代码库答案的准确性，获得更好的、更准确的代码库答案。

**代码库索引功能工作原理**： 
1. 将用户的代码库在本地分割成小块
2. 将每一块发送到Cursor的服务器，服务器会使用 OpenAI 的嵌入 API 或自定义嵌入模型来嵌入代码
3. 嵌入式存储在一个远程向量数据库中，以及起始/结束行号和该文件的相对路径。代码不会存储在Cursor的数据库中，它在请求的生命周期结束后就会消失。

## 4.3.1 Merkle Trees

**概念**：Merkle Tree 是哈希值的二叉树，其中每个叶节点代表一条数据或一条数据的哈希，它用于有效地验证大量数据的完整性。

这种结构形成了一个层次结构，其中任何级别的变化都可以通过比较哈希值来高效检测，可以将它们视为数据的指纹系统：
1. 每个数据（如文件）都有自己的唯一指纹（哈希）
2. 指纹对被组合并赋予一个新的指纹
3. 该过程持续进行，直到你只有一个主指纹（根哈希）



## 4.3.2 Cursor Merkle Tree

Cursor 将 Merkle 树作为其代码库索引功能的核心组件。根据 Cursor 创始人的帖子和官网安全文档，以下是它的工作原理：



**代码块分割策略**

代码库索引的有效性在很大程度上取决于代码的块化方式，基于代码的抽象语法树（AST）结构进行分割。通过深度优先遍历 AST，它将代码分割成适合于 token 限制的子树。为了避免创建太多小块，只要它们保持在 token 限制之下，兄弟节点就会被合并成更大的块。

Cursor 创始人amanrs在[论坛](https://forum.cursor.com/t/codebase-indexing-vs-chat-with-codebase/283/5)提到他们利用tree-sitter分割成语法相关的块，然后将嵌入存储在Cursor的向量数据库中，而永远不会在服务器上存储任何用户的代码。

**Cursor 如何使用嵌入**

当用户与 Cursor 的 AI 功能交互时，例如使用 @Codebase 或 ⌘ Enter 提问代码库，以下过程会发生：
1. **查询嵌入：** Cursor 会为你提出的问题或你正在处理的代码上下文计算嵌入。
2. **向量相似性搜索：** 此查询嵌入被发送到 Turbopuffer（Cursor 的向量数据库），它执行最近邻搜索以找到与您的查询在语义上相似的代码片段。
3. **本地文件访问：** Cursor 的客户端接收结果，其中包含混淆的文件路径和代码片段的行范围。重要的是，实际代码内容仍保留在您的机器上，并本地获取。
4. **上下文组装：** 客户端从您的本地文件中读取这些相关的代码片段，并将它们作为上下文发送到服务器，供 LLM 与您的问题一起处理。
5. **智能响应：** 现在 LLM 拥有来自您代码库的必要上下文，以提供更智能和相关的回答，或生成适当的代码补全。

## 4.3.3 观测Cursor Codebase Indexing握手过程

Cursor 的 Merkle 树实现的一个关键方面是同步过程中发生的握手过程，通过观测Cursor终端 **`Cursor Indexing & Retrieva`** 日志了解代码索引的握手过程。

| 用户交互 | 终端日志 | 详细说明 |
|---|---|---|
| 本地新建代码仓库，通过Cursor初始化代码库索引 | 初始化日志 | 1. 在初始化代码库索引时，Cursor 会创建一个“merkle 客户端”并与服务器进行“启动握手”。<br>2. 初次握手涉及将本地计算的 Merkle 树的根哈希发送给服务器 |
| 更新本地仓库文件 | 更新日志 | Cursor在本地对于代码文件更新后主要执行了以下操作：<br>1. 计算代码库的哈希表示（默克尔树）<br>2. 与服务器进行握手验证代码库状态<br>3. 分析并索引代码库中的文件（606个可索引文件）<br>4. 完成远程同步过程，获取用户的代码提交历史 |
| 通过Cursor Setting删除代码库远程索引 | 删除日志 | 在用户手动通过全局Setting删除代码库索引后：<br>1. **索引任务重置**：终止了多代码库索引任务(multiCodebaseIndexingJob dispose)<br>2. 初始化`InternalRepoInfo`时都使用了嵌入模型参数 0 （可能进行嵌入的重新初始化？） |

## 4.3.4 Cursor 与 augment 代码库索引对比

[augment](https://www.augmentcode.com/) 代码索引核心实现：[Augment Code Codebase-Indexing]()

**augment**


| 用户交互 | augment | Cursor |
|---|---|---|
| 初次索引代码仓库 | **augment代码索引初始化**<br>augment初始化1<br>augment初始化2<br>初次进入代码仓库，会请求完成代码库索引同步，并给出部分项目建议 | 代码索引初始化见：cursor 代码索引初始化<br>用户不感知代码索引，通过全局setting控制 |
| 大型代码库场景修改全局多处代码的复杂任务<br>例如：<br>将项目中样式处理方案如.less、.scss、.css、内联样式，统一处理为scss预处理方案<br>**项目仓库详情：共626个文件**<br>项目详情 | **augment Agent模式**<br>augment执行<br>**任务完成程度❌ ：** 拆解了9个子任务，执行两次重试都失败 | **cursor Agent模式**<br>cursor执行<br>**任务完成程度 ✅ ：** 执行过程中失败了两次，执行两次resume，最后完成整个任务，changefiles 47个<br>**迁移程度70%：**<br>**✅ 完成scss、less、css的整体改造**<br>❌**内联样式并未完成改造，已有的less文件未删除** |

# 五、Cursor Agent的优势与挑战

近期Claude团队对于Cursor团队进行了一次团队采访，**探讨了AI如何彻底改变软件开发**，Cursor如何利用Claude构建AI编码的未来。

采访原视频：[How Cursor is building the future of AI coding with Claude](https://www.youtube.com/watch?v=BGgsoIgbT_Y)

**如果把Cursor现在支持的所有AI编辑工具比作一条光谱，那么光谱从左到右依次是：**


使用Tab时，你知道你在做什么，你可以完全掌控代码，继续往光谱的右侧，你对代码的熟悉程度是依次递减的，但AI编辑工具的智能程度依次提升。

## 5.1 优势

- **实现多文件编辑和更高级别的编程任务**：最初，Cursor 的 AI 主要用于文件内编辑和代码补全。但随着模型（如 Claude 3.5 Sonnet）的改进，结合 Cursor 自己的检索模型，Agent 功能能够实现多文件编辑，这被认为是 Cursor 获得大规模采用的关键“阶跃函数”（step function）
- **自动化整个拉取请求 (PRs) 的工作**：Background Agent 是一种新的基本功能（primitive），能够用于执行整个 PRs 的工作。它可以在后台独立迭代任务，让开发者可以并行处理其他任务。
- **提高工作效率和流程**：通过将耗时的任务推送到后台，Agent 功能允许开发者在模型工作时继续进行其他工作，从而加快开发流程。
- **处理不熟悉的代码库**：当开发者不熟悉某个代码库时，Agent 功能和查询/搜索功能尤其有用，能够帮助理解代码库中不同部分之间的交互方式。

## 5.2 挑战

- **代码验证的瓶颈**：模型在生成和编写大量代码方面表现出色，但软件工程的一个主要瓶颈是代码的验证和审查。开发者需要花费大量时间来审查代码，确保 Agent 所做的更改不仅“正确”，而且符合开发者的预期和“心中的设想”（in your mind's eye）。
- **距离 100% 自动化仍有距离**：虽然模型在端到端任务方面越来越好，但它们尚未达到 100% 的完成度，这意味着开发者仍需要介入并掌控剩余的工作。
- **大型企业代码库的复杂性**：在大型企业代码库中，设置正确的依赖关系以使模型能够运行测试是一项复杂的挑战。Background Agent 团队正在努力简化这个过程，使其易于重复和更新。
- **模型对代码库的深层理解不足**：尽管通过训练检索模型和整合上下文来源（如近期个人更改和团队更改）有所改进，但仅仅让模型访问检索到的信息不足以使其真正理解代码库。这是一个“非常困难的基础性问题”，可能需要结合“记忆”（memory）和“长上下文”（long context）等方法来解决。

## 5.3 未来的展望

> 采访末尾，Lukas提出了一个开放式问题，**“2027年1月1年，距离现在还有不到两年，你认为有多少百分比的代码将被AI生成，随之而来的是，你的一天会是什么样子？”**

> Cursor团队的Jacob（在Cursor负责ML机器学习）给出的回答很有意思，**“这就像在1995年问一个律师，未来有多少法律文件会由Word生成一样。答案是100%。同样的，AI将会参与几乎所有代码的编写。”**这个比例，据Aman（Cursor创始人）介绍，在Cursor内部，可能已经超过90%了。

**对于AI Coding未来的发展？前端在未来AI Coding下的角色，在传统软件工程流程中职能的变化？**

**参考：**

- [https://github.com/cursor/cursor/issues/2209](https://github.com/cursor/cursor/issues/2209)
- [https://github.com/cursor/cursor/issues/981](https://github.com/cursor/cursor/issues/981)
- [https://docs.cursor.com/context/codebase-indexing](https://docs.cursor.com/context/codebase-indexing)
- [https://docs.cursor.com/chat/tools](https://docs.cursor.com/chat/tools)
- [https://docs.augmentcode.com/setup-augment/workspace-indexing](https://docs.augmentcode.com/setup-augment/workspace-indexing)
- [Augment Code AI review — large code base? This is your answer](https://medium.com/@chrisdunlop_37984/augmentcode-ai-review-large-code-base-this-is-your-answer-245f7a2cd9a9)
- [Merkle 树结构](https://yeasy.gitbook.io/blockchain_guide/05_crypto/merkle_trie)
- [A real-time index for your codebase: Secure, personal, scalable](https://www.augmentcode.com/blog/a-real-time-index-for-your-codebase-secure-personal-scalable)