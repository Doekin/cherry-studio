# Cherry Studio AI Provider 技术架构文档 (新方案)

## 1. 核心设计理念与目标

本架构旨在重构 Cherry Studio 的 AI Provider（现称为 `aiCore`）层，以实现以下目标：

- **职责清晰**：明确划分各组件的职责，降低耦合度。
- **高度复用**：最大化业务逻辑和通用处理逻辑的复用，减少重复代码。
- **易于扩展**：方便快捷地接入新的 AI Provider (LLM供应商) 和添加新的 AI 功能 (如翻译、摘要、图像生成等)。
- **易于维护**：简化单个组件的复杂性，提高代码的可读性和可维护性。
- **标准化**：统一内部数据流和接口，简化不同 Provider 之间的差异处理。

核心思路是将纯粹的 **SDK 适配层 (`XxxApiClient`)**、**通用逻辑处理与智能解析层 (中间件)** 以及 **统一业务功能入口层 (`AiCompletionService`)** 清晰地分离开来。

## 2. 核心组件详解

### 2.1. `aiCore` (原 `AiProvider` 文件夹)

这是整个 AI 功能的核心模块。

#### 2.1.1. `XxxApiClient` (例如 `aiCore/openai/OpenAIApiClient.ts`)

- **职责**：作为特定 AI Provider SDK 的纯粹适配层。
  - **参数适配**：将应用内部统一的 `CoreRequest` 对象 (见下文) 转换为特定 SDK 所需的请求参数格式。
  - **基础响应转换**：将 SDK 返回的原始数据块 (`RawSdkChunk`，例如 `OpenAI.Chat.Completions.ChatCompletionChunk`) 转换为一组最基础、最直接的应用层 `Chunk` 对象 (定义于 `src/renderer/src/types/chunk.ts`)。
    - 例如：SDK 的 `delta.content` -> `TextDeltaChunk`；SDK 的 `delta.reasoning_content` -> `ThinkingDeltaChunk`；SDK 的 `delta.tool_calls` -> `RawToolCallChunk` (包含原始工具调用数据)。
    - **关键**：`XxxApiClient` **不处理**耦合在文本内容中的复杂结构，如 `<think>` 或 `<tool_use>` 标签。
- **特点**：极度轻量化，代码量少，易于实现和维护新的 Provider 适配。

#### 2.1.2. `ApiClient.ts` (或 `BaseApiClient.ts` 的核心接口)

- 定义了所有 `XxxApiClient` 必须实现的接口，如：
  - `getSdkInstance(): Promise<TSdkInstance> | TSdkInstance`
  - `getRequestTransformer(): RequestTransformer<TSdkParams>`
  - `getResponseChunkTransformer(): ResponseChunkTransformer<TRawChunk, TResponseContext>`
  - 其他可选的、与特定 Provider 相关的辅助方法 (如工具调用转换)。

#### 2.1.3. `ApiClientFactory.ts`

- 根据 Provider 配置动态创建和返回相应的 `XxxApiClient` 实例。

#### 2.1.4. `services/AiCompletionService.ts` (或其他如 `AiCoreService.ts`)

- **职责**：作为所有 AI 相关业务功能的统一入口。
  - 提供面向应用的高层接口，例如：
    - `executeCompletions(params: CompletionsParams): Promise<CompletionsResult>`
    - `translateText(text: string, targetLang: string, options?: ServiceOptions): Promise<string>`
    - `summarizeText(text: string, options?: ServiceOptions): Promise<string>`
    - 未来可能的 `generateImage(prompt: string): Promise<ImageResult>` 等。
  - **封装特定任务的提示工程 (Prompt Engineering)**：
    - 例如，`translateText` 方法内部会构建一个包含特定翻译指令的 `CoreRequest`。
  - **编排和调用中间件链**：根据调用的业务方法和参数，动态选择和组织合适的中间件序列。
  - 获取 `ApiClient` 实例并将其注入到中间件上游的 `Context` 中。
- **优势**：业务逻辑（如翻译、摘要的提示构建和流程控制）只需实现一次，即可支持所有通过 `ApiClient` 接入的底层 Provider。

#### 2.1.5. `coreRequestTypes.ts` (或 `types.ts`)

- 定义核心的、Provider 无关的内部请求结构，例如：
  - `CoreCompletionsRequest`: 包含标准化后的消息列表、模型配置、工具列表、最大Token数、是否流式输出等。
  - `CoreTranslateRequest`, `CoreSummarizeRequest` 等 (如果与 `CoreCompletionsRequest` 结构差异较大，否则可复用并添加任务类型标记)。

### 2.2. `middleware`

中间件层负责处理请求和响应流中的通用逻辑和特定特性。

#### 2.2.1. `middlewareTypes.ts`

- 定义中间件的核心类型，如 `AiProviderMiddlewareContext` (扩展后包含 `_apiClientInstance` 和 `_coreRequest`)、`MiddlewareAPI`、`CompletionsMiddleware` 等。

#### 2.2.2. 核心中间件 (`middleware/core/`)

- **`TransformCoreToSdkParamsMiddleware.ts`**: 调用 `ApiClient.getRequestTransformer()` 将 `CoreRequest` 转换为特定 SDK 的参数，并存入上下文。
- **`RequestExecutionMiddleware.ts`**: 调用 `ApiClient.getSdkInstance()` 获取 SDK 实例，并使用转换后的参数执行实际的 API 调用，返回原始 SDK 流。
- **`StreamAdapterMiddleware.ts`**: 将各种形态的原始 SDK 流 (如异步迭代器) 统一适配为 `ReadableStream<RawSdkChunk>`。
  - **`RawSdkChunk`**：指特定AI提供商SDK在流式响应中返回的、未经应用层统一处理的原始数据块格式 (例如 OpenAI 的 `ChatCompletionChunk`，Gemini 的 `GenerateContentResponse` 中的部分等)。
- **`RawSdkChunkToAppChunkMiddleware.ts`**: (新增) 消费 `ReadableStream<RawSdkChunk>`，在其内部对每个 `RawSdkChunk` 调用 `ApiClient.getResponseChunkTransformer()`，将其转换为一个或多个基础的应用层 `Chunk` 对象，并输出 `ReadableStream<Chunk>`。

#### 2.2.3. 特性中间件 (`middleware/feature/`)

这些中间件消费由 `RawSdkChunkToAppChunkMiddleware` 输出的、相对标准化的 `Chunk` 流，并处理更复杂的逻辑。

- **`ThinkingTagExtractionMiddleware.ts`**: 检查 `TextDeltaChunk`，解析其中可能包含的 `<think>...</think>` 文本内嵌标签，生成 `ThinkingDeltaChunk` 和 `ThinkingCompleteChunk`。
- **`ToolUseTagExtractionMiddleware.ts`**: 检查 `TextDeltaChunk`，解析其中可能包含的 `<tool_use>...</tool_use>` 文本内嵌标签，生成 `McpToolCallRequestChunk` (或其前置标准化Chunk)。如果 `ApiClient` 输出了 `RawToolCallChunk` (来自SDK原生工具调用)，此中间件或其后续也可能负责将其转换为标准格式。
- **`McpToolHandlerMiddleware.ts`**: 处理标准化的工具调用请求 (`McpToolCallRequestChunk`)，执行工具并处理响应，可能涉及递归调用 `AiCompletionService`。
- **`WebSearchHandlerMiddleware.ts`**: 处理 Web 搜索相关逻辑。
- **`TextChunkMiddleware.ts`**: 处理最终的、不含特殊标签的文本流。

#### 2.2.4. 通用中间件 (`middleware/common/`)

- **`LoggingMiddleware.ts`**: 请求和响应日志。
- **`AbortHandlerMiddleware.ts`**: 处理请求中止。
- **`ErrorHandlingMiddleware.ts`**: 统一错误处理。
- **`FinalChunkConsumerAndNotifierMiddleware.ts`**: 消费最终的 `Chunk` 流，通过 `onChunk` 回调通知应用层，并在流结束时发送 `BLOCK_COMPLETE` 及累加的 `Usage`/`Metrics`。

### 2.3. `types/chunk.ts`

- 定义应用全局统一的 `Chunk` 类型及其所有变体。这包括基础类型 (如 `TextDeltaChunk`, `ThinkingDeltaChunk`)、SDK原生数据传递类型 (如 `RawToolCallChunk`, `RawFinishChunk` - 作为 `ApiClient` 转换的中间产物)，以及功能性类型 (如 `McpToolCallRequestChunk`, `WebSearchCompleteChunk`)。

## 3. 核心执行流程 (以 `AiCompletionService.executeCompletions` 为例)

```markdown
**应用层 (例如 UI 组件)**
||
\/
**`AiCompletionService.executeCompletions` (`aiCore/services/AiCompletionService.ts`)**
(构建 `CoreCompletionsRequest`, 获取 `ApiClient` 实例, 创建 `Context`, 编排中间件链)
||
\/
**`applyMiddlewares` (或类似的通用中间件执行器)**
(开始执行中间件链)
||
\/
**`LoggingMiddleware` (`middleware/common/LoggingMiddleware.ts`)**
||
\/
**`AbortHandlerMiddleware` (`middleware/common/AbortHandlerMiddleware.ts`)**
||
\/
**`TransformCoreToSdkParamsMiddleware` (`middleware/core/TransformCoreToSdkParamsMiddleware.ts`)**
调用 --> **`ApiClient.getRequestTransformer().transform` (例如 `aiCore/openai/OpenAIApiClient.ts`)**
(将 `CoreRequest` 转换为 `SdkSpecificParams`)
|| (输出: `SdkSpecificParams` 存入 Context)
\/
**`RequestExecutionMiddleware` (`middleware/core/RequestExecutionMiddleware.ts`)**
调用 --> **`ApiClient.getSdkInstance` (例如 `aiCore/openai/OpenAIApiClient.ts`)**
调用 --> **`sdkInstance.chat.completions.create` (例如 OpenAI SDK 的方法)**
|| (输出: 原始SDK流, 如 `AsyncIterable<OpenAI.Chat.Completions.ChatCompletionChunk>`)
\/
**`StreamAdapterMiddleware` (`middleware/core/StreamAdapterMiddleware.ts`)**
(将原始SDK流转换为 `ReadableStream<RawSdkChunk>`)
|| (输出: `ReadableStream<RawSdkChunk>`)
\/
**`RawSdkChunkToAppChunkMiddleware` (`middleware/core/RawSdkChunkToAppChunkMiddleware.ts`)**
调用 --> **`ApiClient.getResponseChunkTransformer()` (例如 `aiCore/openai/OpenAIApiClient.ts`)**
(将 `RawSdkChunk` -> `Chunk[]` - 基础应用层类型)
|| (输出: `ReadableStream<Chunk>` - 包含基础应用层Chunk)
\/
**`ThinkingTagExtractionMiddleware` (`middleware/feature/ThinkingTagExtractionMiddleware.ts`)**
(处理 `<think>` 标签)
||
\/
**`ToolUseTagExtractionMiddleware` (`middleware/feature/ToolUseTagExtractionMiddleware.ts`)**
(处理 `<tool_use>` 标签或 `RawToolCallChunk`)
||
\/
**`TextChunkMiddleware` (`middleware/common/TextChunkMiddleware.ts` 或 `feature/`)
||
\/
**`McpToolHandlerMiddleware` (`middleware/feature/McpToolHandlerMiddleware.ts`)**
(可能递归调用 `AiCompletionService`)
||
\/
**`FinalChunkConsumerAndNotifierMiddleware` (`middleware/common/FinalChunkConsumerAndNotifierMiddleware.ts`)**
调用 --> **`context.onChunk(chunk)` (应用层提供的回调)**
||
\/
**`LoggingMiddleware` (响应部分)
||
\/
**`ErrorHandlingMiddleware`
||
\/
**`AiCompletionService.executeCompletions` 返回 `Promise`\*\*
```

## 4. 建议的文件/目录结构

```
src/renderer/src/
├── aiCore/                             # 原 AiProvider 文件夹
│   ├── openai/                       # 特定 Provider 的适配层
│   │   └── OpenAIApiClient.ts
│   │   └── types.ts                  # (可选) OpenAI 特有类型
│   ├── gemini/
│   │   └── GeminiApiClient.ts
│   ├── ...                            # 其他 Provider 的 ApiClient
│   ├── services/                      # 统一的业务服务层
│   │   ├── AiCompletionService.ts
│   │   └── types.ts                  # 服务层参数、选项等类型
│   ├── ApiClient.ts                    # ApiClient 接口定义
│   ├── ApiClientFactory.ts             # ApiClient 工厂
│   ├── coreRequestTypes.ts             # CoreRequest 等核心请求类型
│   └── index.ts                        # 导出主要接口和服务
│
├── middleware/                         # 中间件
│   ├── common/                         # 通用型中间件
│   ├── core/                           # 核心流转中间件
│   ├── feature/                        # 特定特性处理中间件
│   ├── middlewareTypes.ts              # 中间件核心类型
│   └── index.ts
│
├── types/
│   ├── chunk.ts                        # 全局统一 Chunk 类型定义
│   └── ...                             # 其他全局类型
└── ...                                 # 其他应用目录
```

## 5. 迁移和实施建议

- **小步快跑，逐步迭代**：优先完成核心流程的重构（例如 `completions`），再逐步迁移其他功能（`translate` 等）和其他 Provider。
- **优先定义核心类型**：`CoreRequest`, `Chunk`, `ApiClient` 接口是整个架构的基石。
- **为 `ApiClient` 瘦身**：将现有 `XxxProvider` 中的复杂逻辑剥离到新的中间件或 `AiCompletionService` 中。
- **强化中间件**：让中间件承担起更多解析和特性处理的责任。
- **编写单元测试和集成测试**：确保每个组件和整体流程的正确性。

此架构旨在提供一个更健壮、更灵活、更易于维护的 AI 功能核心，支撑 Cherry Studio 未来的发展。
