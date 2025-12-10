# Cherry Studio 记忆系统完整解析

## 一、架构总览

Cherry Studio 的记忆系统是一个**双向记忆系统**，包含两种不同类型的记忆机制：

1. **基于向量数据库的用户记忆** - 存储用户个人信息和偏好的长期记忆
2. **基于知识图谱的 MCP 记忆** - 使用实体-关系模型的结构化记忆

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Cherry Studio 记忆系统                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────┐         ┌─────────────────────┐           │
│  │  向量记忆系统        │         │  MCP 知识图谱记忆    │           │
│  │  (User Memory)      │         │  (Memory Server)     │           │
│  │                     │         │                      │           │
│  │  • LibSQL 数据库    │         │  • JSON 文件存储     │           │
│  │  • 向量嵌入搜索     │         │  • 实体-关系模型     │           │
│  │  • 事实提取 + 更新  │         │  • 文本搜索          │           │
│  └─────────────────────┘         └─────────────────────┘           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 二、核心组件详解

### 2.1 数据存储层 (Main Process)

#### 📦 主进程 MemoryService

```1:64:src/main/services/memory/MemoryService.ts
import type { Client } from '@libsql/client'
import { createClient } from '@libsql/client'
// ... 省略
export class MemoryService {
  private static instance: MemoryService | null = null
  private db: Client | null = null
  private isInitialized = false
  private embeddings: Embeddings | null = null
  private config: MemoryConfig | null = null
  private static readonly UNIFIED_DIMENSION = 1536
  private static readonly SIMILARITY_THRESHOLD = 0.85
```

**关键特性：**

- 使用 **LibSQL (SQLite 的分支)** 作为数据库，支持原生向量存储
- 单例模式，确保全局唯一实例
- 统一向量维度为 **1536**（兼容 OpenAI 嵌入模型）
- 相似度阈值 **0.85**，用于去重判断

#### 📊 数据库表结构

```6:38:src/main/services/memory/queries.ts
export const MemoryQueries = {
  // Table creation queries
  createTables: {
    memories: `
      CREATE TABLE IF NOT EXISTS memories (
        id TEXT PRIMARY KEY,
        memory TEXT NOT NULL,
        hash TEXT UNIQUE,
        embedding F32_BLOB(1536), -- Native vector column (1536 dimensions for OpenAI embeddings)
        metadata TEXT, -- JSON string
        user_id TEXT,
        agent_id TEXT,
        run_id TEXT,
        created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
        updated_at DATETIME DEFAULT CURRENT_TIMESTAMP,
        is_deleted INTEGER DEFAULT 0
      )
    `,

    memoryHistory: `
      CREATE TABLE IF NOT EXISTS memory_history (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        memory_id TEXT NOT NULL,
        previous_value TEXT,
        new_value TEXT,
        action TEXT NOT NULL, -- ADD, UPDATE, DELETE
        created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
        updated_at DATETIME DEFAULT CURRENT_TIMESTAMP,
        is_deleted INTEGER DEFAULT 0,
        FOREIGN KEY (memory_id) REFERENCES memories (id)
      )
    `
  },
```

**两张核心表：**

1. **memories** - 存储记忆主体

   - `embedding F32_BLOB(1536)` - 原生向量列，支持高效相似度搜索
   - `hash` - SHA256 哈希用于去重
   - 支持软删除 (`is_deleted`)

2. **memory_history** - 记忆变更历史
   - 记录 ADD/UPDATE/DELETE 操作
   - 保留 `previous_value` 用于回溯

---

### 2.2 记忆处理层 (Renderer Process)

#### 🔄 MemoryProcessor - 记忆处理器

```26:85:src/renderer/src/services/MemoryProcessor.ts
export class MemoryProcessor {
  private memoryService: MemoryService

  constructor() {
    this.memoryService = MemoryService.getInstance()
  }

  /**
   * Extract facts from conversation messages
   */
  async extractFacts(messages: AssistantMessage[], config: MemoryProcessorConfig): Promise<string[]> {
    // ... 使用 LLM 从对话中提取事实
    const [systemPrompt, userPrompt] = getFactRetrievalMessages(parsedMessages)
    const responseContent = await fetchGenerate({
      prompt: systemPrompt,
      content: userPrompt,
      model: getModel(memoryConfig.llmApiClient.model, memoryConfig.llmApiClient.provider)
    })
    // 解析 JSON 响应
    const parsed = FactRetrievalSchema.parse(dataToValidate)
    return parsed.facts
  }
```

**工作流程：**

```
用户对话 → 事实提取(LLM) → 与现有记忆比较 → 决定操作(ADD/UPDATE/DELETE/NONE) → 存储
```

#### 📝 事实提取提示词

```21:91:src/renderer/src/utils/memory-prompts.ts
export const factExtractionPrompt: string = `You are a Personal Information Organizer, specialized in accurately storing facts, user memories, and preferences. Your primary role is to extract relevant pieces of information about the user from conversations and organize them into distinct, manageable facts.

Types of Information to Remember:

1. Store Personal Preferences: Keep track of likes, dislikes, and specific preferences
2. Maintain Important Personal Details: Remember significant personal information like names, relationships
3. Track Plans and Intentions: Note upcoming events, trips, goals
4. Remember Activity and Service Preferences: Recall preferences for dining, travel, hobbies
5. Monitor Health and Wellness Preferences: Keep a record of dietary restrictions, fitness routines
6. Store Professional Details: Remember job titles, work habits, career goals
7. Miscellaneous Information Management: Keep track of favorite books, movies, brands

DO NOT EXTRACT:
- Questions or requests for information
- Technical help requests
- General inquiries about tools, methods, or procedures
```

**重要特性：**

- 只提取**个人信息**，过滤通用知识
- 支持多语言（检测输入语言并用相同语言记录）
- 排除问题和帮助请求

#### 🔄 记忆更新逻辑

```93:241:src/renderer/src/utils/memory-prompts.ts
export const updateMemorySystemPrompt: string = `You are a smart memory manager which controls the memory of a system.
You can perform four operations: (1) add into the memory, (2) update the memory, (3) delete from the memory, and (4) no change.

1. **Add**: If the retrieved facts contain new information not present in the memory
2. **Update**: If the retrieved facts contain information that is already present but different
3. **Delete**: If the retrieved facts contain information that contradicts the memory
4. **No Change**: If the retrieved facts are already present in the memory
```

---

### 2.3 搜索编排插件

#### 🎯 searchOrchestrationPlugin

```239:409:src/renderer/src/aiCore/plugins/searchOrchestrationPlugin.ts
export const searchOrchestrationPlugin = (assistant: Assistant, topicId: string) => {
  return definePlugin({
    name: 'search-orchestration',
    enforce: 'pre', // 确保在其他插件之前执行

    /**
     * 🔍 Step 1: 意图识别阶段
     */
    onRequestStart: async (context: AiRequestContext) => {
      // 判断是否需要各种搜索
      const shouldMemorySearch = globalMemoryEnabled && assistant.enableMemory
      // ...
    },

    /**
     * 🔧 Step 2: 工具配置阶段
     */
    transformParams: async (params: any, context: AiRequestContext) => {
      // 🧠 记忆搜索工具配置
      const globalMemoryEnabled = selectGlobalMemoryEnabled(store.getState())
      if (globalMemoryEnabled && assistant.enableMemory) {
        params.tools['builtin_memory_search'] = memorySearchTool()
      }
      return params
    },

    /**
     * 💾 Step 3: 记忆存储阶段
     */
    onRequestEnd: async (context: AiRequestContext) => {
      // 在对话结束后，异步处理记忆存储
      await storeConversationMemory(messages, assistant, context)
    }
  })
}
```

**三阶段处理：**

1. **onRequestStart** - 意图识别，判断是否需要搜索
2. **transformParams** - 动态注入记忆搜索工具
3. **onRequestEnd** - 异步存储对话记忆（不阻塞 UI）

---

### 2.4 向量搜索实现

#### 🔍 混合搜索算法

```753:808:src/main/services/memory/MemoryService.ts
  private async hybridSearch(
    _: string,
    queryEmbedding: number[],
    options: VectorSearchOptions = {}
  ): Promise<SearchResult> {
    // 向量搜索 SQL
    const hybridQuery = `
      SELECT * FROM (
        SELECT
          m.id, m.memory, m.hash, m.metadata,
          CASE
            WHEN m.embedding IS NULL THEN 2.0
            ELSE vector_distance_cos(m.embedding, vector32(?))
          END as distance,
          CASE
            WHEN m.embedding IS NULL THEN 0.0
            ELSE (1 - vector_distance_cos(m.embedding, vector32(?)))
          END as vector_similarity
        FROM memories m
        WHERE m.is_deleted = 0
      ) AS results
      WHERE vector_similarity >= ?
      ORDER BY vector_similarity DESC
      LIMIT ?
    `
```

**搜索特性：**

- 使用 `vector_distance_cos` 计算余弦相似度
- 支持阈值过滤（默认 0.5）
- 支持用户和助手 ID 过滤
- 返回结果按相似度排序

---

### 2.5 MCP 记忆服务器（知识图谱）

#### 🌐 KnowledgeGraphManager

```35:337:src/main/mcpServers/memory.ts
class KnowledgeGraphManager {
  private memoryPath: string
  private entities: Map<string, Entity> // 实体存储
  private relations: Set<string>        // 关系存储

  // 实体结构
  interface Entity {
    name: string
    entityType: string
    observations: string[]  // 观察记录
  }

  // 关系结构
  interface Relation {
    from: string
    to: string
    relationType: string
  }

  // 支持的操作
  async createEntities(entities: Entity[]): Promise<Entity[]>
  async createRelations(relations: Relation[]): Promise<Relation[]>
  async addObservations(observations: []): Promise<[]>
  async deleteEntities(entityNames: string[]): Promise<void>
  async searchNodes(query: string): Promise<KnowledgeGraph>
}
```

**这是一个独立的 MCP Server，提供以下工具：**

- `create_entities` - 创建实体
- `create_relations` - 创建关系
- `add_observations` - 添加观察
- `delete_entities` - 删除实体
- `read_graph` - 读取整个图
- `search_nodes` - 搜索节点
- `open_nodes` - 打开特定节点

---

### 2.6 状态管理 (Redux Store)

```1:119:src/renderer/src/store/memory.ts
export interface MemoryState {
  /** The current memory configuration */
  memoryConfig: MemoryConfig
  /** The currently selected user ID for memory operations */
  currentUserId: string
  /** Global memory enabled state - when false, memory is disabled for all assistants */
  globalMemoryEnabled: boolean
}

const defaultMemoryConfig: MemoryConfig = {
  embedderDimensions: 1536,
  isAutoDimensions: true,
  customFactExtractionPrompt: factExtractionPrompt,
  customUpdateMemoryPrompt: updateMemorySystemPrompt
}
```

**状态结构：**

- `memoryConfig` - 嵌入模型配置、维度、自定义提示词
- `currentUserId` - 当前用户 ID（支持多用户）
- `globalMemoryEnabled` - 全局开关

---

### 2.7 IPC 通信桥接

```282:297:src/preload/index.ts
  memory: {
    add: (messages: string | AssistantMessage[], options?: AddMemoryOptions) =>
      ipcRenderer.invoke(IpcChannel.Memory_Add, messages, options),
    search: (query: string, options: MemorySearchOptions) =>
      ipcRenderer.invoke(IpcChannel.Memory_Search, query, options),
    list: (options?: MemoryListOptions) => ipcRenderer.invoke(IpcChannel.Memory_List, options),
    delete: (id: string) => ipcRenderer.invoke(IpcChannel.Memory_Delete, id),
    update: (id: string, memory: string, metadata?: Record<string, any>) =>
      ipcRenderer.invoke(IpcChannel.Memory_Update, id, memory, metadata),
    get: (id: string) => ipcRenderer.invoke(IpcChannel.Memory_Get, id),
    setConfig: (config: MemoryConfig) => ipcRenderer.invoke(IpcChannel.Memory_SetConfig, config),
    deleteUser: (userId: string) => ipcRenderer.invoke(IpcChannel.Memory_DeleteUser, userId),
    deleteAllMemoriesForUser: (userId: string) =>
      ipcRenderer.invoke(IpcChannel.Memory_DeleteAllMemoriesForUser, userId),
    getUsersList: () => ipcRenderer.invoke(IpcChannel.Memory_GetUsersList)
  },
```

---

## 三、完整工作流程图

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                          记忆系统完整工作流程                                  │
└──────────────────────────────────────────────────────────────────────────────┘

1️⃣ 对话开始
   ┌─────────────────┐
   │  用户发送消息    │
   └────────┬────────┘
            │
            ▼
   ┌─────────────────┐
   │ 搜索编排插件     │  onRequestStart()
   │ 检查是否启用记忆 │
   └────────┬────────┘
            │
            ▼
2️⃣ 记忆检索（如果启用）
   ┌─────────────────┐
   │ 生成查询嵌入向量 │
   │ (Embeddings API)│
   └────────┬────────┘
            │
            ▼
   ┌─────────────────┐
   │  向量相似度搜索  │  hybridSearch()
   │  (LibSQL)       │
   └────────┬────────┘
            │
            ▼
   ┌─────────────────┐
   │ 将相关记忆注入   │
   │ 到对话上下文    │
   └────────┬────────┘
            │
            ▼
3️⃣ AI 生成响应
   ┌─────────────────┐
   │ LLM 生成响应    │  (带有记忆上下文)
   └────────┬────────┘
            │
            ▼
4️⃣ 记忆存储（异步）
   ┌─────────────────┐
   │ 事实提取        │  extractFacts()
   │ (LLM 分析对话)  │
   └────────┬────────┘
            │
            ▼
   ┌─────────────────┐
   │  与现有记忆比较  │  updateMemories() -- 代码只负责：拿旧记忆 + 新 facts → 喂给模型 → 根据模型输出执行操作
   │  决定操作类型   │
   │  ADD/UPDATE/    │
   │  DELETE/NONE    │
   └────────┬────────┘
            │
            ▼
   ┌─────────────────────────────────────┐
   │              执行操作               │
   ├─────────┬─────────┬─────────┬──────┤
   │   ADD   │ UPDATE  │ DELETE  │ NONE │
   │  新增   │  更新   │  删除   │ 跳过 │
   └─────────┴─────────┴─────────┴──────┘
            │
            ▼
   ┌─────────────────┐
   │ 生成嵌入向量    │
   │ 存入 LibSQL    │
   │ 记录变更历史   │
   └─────────────────┘
```

---

## 四、关键技术特性

### 4.1 去重机制

```144:239:src/main/services/memory/MemoryService.ts
// 1. 哈希去重 - 完全相同的文本
const hash = crypto.createHash('sha256').update(trimmedMemory).digest('hex')
const existing = await this.db.execute({
  sql: MemoryQueries.memory.checkExistsIncludeDeleted,
  args: [hash]
})

// 2. 向量相似度去重 - 语义相似的内容
const similarMemories = await this.hybridSearch(trimmedMemory, embedding, {
  limit: 5,
  threshold: 0.1
})
if (highestSimilarity >= MemoryService.SIMILARITY_THRESHOLD) {  // 0.85
  logger.debug('Skipping memory addition due to high similarity')
  continue
}
```

### 4.2 多用户支持

```56:68:src/renderer/src/services/MemoryService.ts
public setCurrentUser(userId: string): void {
  this.currentUserId = userId
}

// 所有操作都自动附加当前用户 ID
public async list(config?: MemoryListOptions): Promise<MemorySearchResult> {
  const configWithUser = {
    ...config,
    userId: this.currentUserId
  }
  // ...
}
```

### 4.3 软删除 + 历史追踪

```440:471:src/main/services/memory/MemoryService.ts
public async delete(id: string): Promise<void> {
  // 获取当前值用于历史记录
  const currentMemory = (current.rows[0] as any).memory as string

  // 软删除
  await this.db.execute({
    sql: MemoryQueries.memory.softDelete,
    args: [new Date().toISOString(), id]
  })

  // 记录历史
  await this.addHistory(id, currentMemory, null, 'DELETE')
}
```

### 4.4 向量维度自动归一化

```695:706:src/main/services/memory/MemoryService.ts
private normalizeEmbedding(embedding: number[]): number[] {
  if (embedding.length === MemoryService.UNIFIED_DIMENSION) {
    return embedding
  }
  if (embedding.length < MemoryService.UNIFIED_DIMENSION) {
    // 补零
    return [...embedding, ...new Array(MemoryService.UNIFIED_DIMENSION - embedding.length).fill(0)]
  } else {
    // 截断
    return embedding.slice(0, MemoryService.UNIFIED_DIMENSION)
  }
}
```

---

## 五、类型定义

```907:938:src/renderer/src/types/index.ts
export interface MemoryConfig {
  embedderModel?: Model  // @deprecated
  embedderDimensions?: number
  embedderApiClient?: ApiClient
  llmApiClient?: ApiClient
  customFactExtractionPrompt?: string
  customUpdateMemoryPrompt?: string
  isAutoDimensions?: boolean
}

export interface MemoryItem {
  id: string
  memory: string
  hash?: string
  createdAt?: string
  updatedAt?: string
  score?: number  // 搜索相似度分数
  metadata?: Record<string, any>
}

export interface MemorySearchResult {
  results: MemoryItem[]
  relations?: any[]
}
```

---

## 六、总结

Cherry Studio 的记忆系统是一个**工程完善度很高**的实现：

| 特性       | 实现方式                            |
| ---------- | ----------------------------------- |
| **存储**   | LibSQL + 原生向量列                 |
| **嵌入**   | 支持多种嵌入模型 (OpenAI, 自定义等) |
| **搜索**   | 余弦相似度向量搜索                  |
| **去重**   | 哈希 + 向量相似度双重校验           |
| **提取**   | LLM 驱动的事实提取                  |
| **更新**   | 智能 ADD/UPDATE/DELETE/NONE 决策    |
| **多用户** | 用户级隔离                          |
| **历史**   | 完整变更追踪                        |
| **异步**   | 记忆存储不阻塞 UI                   |
| **可配置** | 支持自定义提示词                    |

这是一个参考了 [Mem0](https://github.com/mem0ai/mem0) 等开源项目思路，但针对桌面应用场景进行了优化的实现。

---

## 七、向量记忆系统详解：注入、召回与更新

### 7.1 记忆召回机制（如何判断召回）

#### 触发条件

记忆召回在 `searchOrchestrationPlugin` 的 `transformParams` 阶段触发：

```typescript
// src/renderer/src/aiCore/plugins/searchOrchestrationPlugin.ts
const globalMemoryEnabled = selectGlobalMemoryEnabled(store.getState());
if (globalMemoryEnabled && assistant.enableMemory) {
  params.tools["builtin_memory_search"] = memorySearchTool();
}
```

**召回条件：**

1. 全局记忆功能已启用 (`globalMemoryEnabled = true`)
2. 当前助手启用了记忆 (`assistant.enableMemory = true`)

#### 召回流程

```
┌──────────────────────────────────────────────────────────────────┐
│                     记忆召回完整流程                              │
└──────────────────────────────────────────────────────────────────┘

1. 用户发送消息
   │
   ▼
2. AI 自动判断是否需要搜索记忆（通过 Tool Calling）
   │  工具: builtin_memory_search
   │  参数: { query: "搜索关键词", limit: 5 }
   │
   ▼
3. 执行向量搜索
   │  a) 生成查询嵌入向量: embedding = await embeddings.embedQuery(query)
   │  b) 向量相似度搜索: SELECT ... WHERE vector_similarity >= threshold
   │
   ▼
4. 返回相关记忆
   │  格式: MemoryItem[] (id, memory, createdAt, score...)
   │
   ▼
5. 注入到上下文（作为 Tool Result 返回给 LLM）
```

#### 向量搜索 SQL

```sql
-- src/main/services/memory/queries.ts
SELECT
  m.id, m.memory, m.hash, m.metadata,
  CASE
    WHEN m.embedding IS NULL THEN 0.0
    ELSE (1 - vector_distance_cos(m.embedding, vector32(?)))
  END as vector_similarity
FROM memories m
WHERE m.is_deleted = 0 AND m.user_id = ?
  AND vector_similarity >= 0.5  -- 默认阈值
ORDER BY vector_similarity DESC
LIMIT ?
```

### 7.2 记忆注入格式

#### 方式一：Tool Calling 模式（推荐）

Cherry Studio 使用 **AI SDK Tool Calling** 机制，记忆作为工具执行结果返回给 LLM：

```typescript
// src/renderer/src/aiCore/tools/MemorySearchTool.ts
export const memorySearchTool = () => {
  return tool({
    name: "builtin_memory_search",
    description:
      "Search through conversation memories and stored facts for relevant context",
    inputSchema: z.object({
      query: z.string().describe("Search query to find relevant memories"),
      limit: z.number().min(1).max(20).default(5),
    }),
    execute: async ({ query, limit = 5 }) => {
      // ... 执行搜索
      return relevantMemories; // 返回 MemoryItem[]
    },
  });
};
```

**注入格式（Tool Result）：**

```json
// AI 收到的 Tool Result 格式
{
  "type": "tool-result",
  "toolCallId": "call_xxx",
  "toolName": "builtin_memory_search",
  "result": [
    {
      "id": "uuid-1",
      "memory": "用户喜欢 Python 编程",
      "createdAt": "2024-01-15T10:30:00Z",
      "score": 0.92
    },
    {
      "id": "uuid-2",
      "memory": "用户的名字是小明",
      "createdAt": "2024-01-10T08:00:00Z",
      "score": 0.87
    }
  ]
}
```

#### 方式二：引用提示词注入（Legacy 模式）

在旧版 API Client 中，记忆通过 `REFERENCE_PROMPT` 模板注入：

```typescript
// src/renderer/src/aiCore/legacy/clients/BaseApiClient.ts
private getMemoryReferencesFromCache(message: Message) {
  const memories = window.keyv.get(`memory-search-${message.id}`) as MemoryItem[]
  if (memories) {
    return memories.map((mem, index) => ({
      id: index + 1,
      content: `${mem.memory} -- Created at: ${mem.createdAt}`,
      sourceUrl: '',
      type: 'memory'
    }))
  }
  return []
}
```

**REFERENCE_PROMPT 模板：**

```
// src/renderer/src/config/prompts.ts
Please answer the question based on the reference materials

## Citation Rules:
- Please cite the context at the end of sentences when appropriate.
- Please use the format of citation number [number] to reference the context.
- If a sentence comes from multiple contexts, list all relevant citation numbers.
- If all reference content is not relevant, please answer based on your knowledge.

## My question is:

{question}

## Reference Materials:

{references}

Please respond in the same language as the user's question.
```

**实际注入示例：**

````markdown
Please answer the question based on the reference materials

## My question is:

我喜欢什么编程语言？

## Reference Materials:

```json
[
  {
    "number": 1,
    "content": "用户喜欢 Python 编程 -- Created at: 2024-01-15T10:30:00Z",
    "type": "memory"
  },
  {
    "number": 2,
    "content": "用户正在学习 TypeScript -- Created at: 2024-01-20T14:00:00Z",
    "type": "memory"
  }
]
```
````

Please respond in the same language as the user's question.

````

### 7.3 记忆更新机制（如何判断是否更新）

#### 更新流程触发点

在 `searchOrchestrationPlugin` 的 `onRequestEnd` 阶段，对话结束后**异步**处理记忆存储：

```typescript
// src/renderer/src/aiCore/plugins/searchOrchestrationPlugin.ts
onRequestEnd: async (context: AiRequestContext) => {
  await storeConversationMemory(messages, assistant, context)
}
````

#### 完整更新流程

```
┌──────────────────────────────────────────────────────────────────┐
│                     记忆更新完整流程                              │
└──────────────────────────────────────────────────────────────────┘

1. 对话结束后，提取最近的用户和助手消息
   │
   ▼
2. 事实提取 (extractFacts)
   │  使用 LLM 分析对话，提取用户个人信息
   │  输出: ["用户喜欢咖啡", "用户住在北京"]
   │
   ▼
3. 获取现有相关记忆
   │  从缓存获取: window.keyv.get(`memory-search-${lastMessageId}`)
   │  转换格式: [{ id: "xxx", text: "..." }, ...]
   │
   ▼
4. 记忆更新决策 (updateMemories)
   │  使用 LLM 比较新事实与现有记忆
   │  决定每条记忆的操作: ADD / UPDATE / DELETE / NONE
   │
   ▼
5. 执行数据库操作
   │  ADD: memoryService.add(fact, { userId, agentId })
   │  UPDATE: memoryService.update(id, newText, metadata)
   │  DELETE: memoryService.delete(id)
```

#### 事实提取提示词

```typescript
// src/renderer/src/utils/memory-prompts.ts
export const factExtractionPrompt = `You are a Personal Information Organizer...

Types of Information to Remember:
1. Store Personal Preferences: likes, dislikes, specific preferences
2. Maintain Important Personal Details: names, relationships, important dates
3. Track Plans and Intentions: upcoming events, trips, goals
4. Remember Activity and Service Preferences: dining, travel, hobbies
5. Monitor Health and Wellness Preferences: dietary restrictions, fitness routines
6. Store Professional Details: job titles, work habits, career goals
7. Miscellaneous Information: favorite books, movies, brands

DO NOT EXTRACT:
- Questions or requests for information
- Technical help requests
- General inquiries

Return format: {"facts": ["fact1", "fact2", ...]}
`;
```

**输入输出示例：**

```
输入对话:
user: 我明天要去上海出差，帮我订一家酒店
assistant: 好的，您有什么偏好吗？
user: 我喜欢住五星级酒店，最好靠近浦东机场

输出:
{
  "facts": [
    "明天要去上海出差",
    "喜欢住五星级酒店",
    "偏好靠近浦东机场的酒店"
  ]
}
```

#### 记忆更新决策提示词

```typescript
// src/renderer/src/utils/memory-prompts.ts
export const updateMemorySystemPrompt = `You are a smart memory manager...

Operations:
1. ADD: New information not present in memory → generate new ID
2. UPDATE: Information exists but different → keep same ID, update content
3. DELETE: Information contradicts existing memory → remove
4. NONE: Information already present → no change

Return format:
[
  {"id": "0", "text": "...", "event": "ADD/UPDATE/DELETE/NONE", "old_memory": "..."}
]
`;
```

**更新决策示例：**

```
现有记忆:
[
  {"id": "0", "text": "用户喜欢喝咖啡"},
  {"id": "1", "text": "用户住在北京"}
]

新提取的事实:
["用户现在喜欢喝茶", "用户明天要去上海"]

LLM 决策:
[
  {"id": "0", "text": "用户现在喜欢喝茶", "event": "UPDATE", "old_memory": "用户喜欢喝咖啡"},
  {"id": "1", "text": "用户住在北京", "event": "NONE"},
  {"id": "2", "text": "用户明天要去上海", "event": "ADD"}
]
```

#### 去重机制

在添加新记忆前，会进行双重去重检查：

```typescript
// src/main/services/memory/MemoryService.ts

// 1. 哈希去重（完全相同的文本）
const hash = crypto.createHash("sha256").update(trimmedMemory).digest("hex");
const existing = await this.db.execute({
  sql: "SELECT id, is_deleted FROM memories WHERE hash = ?",
  args: [hash],
});
if (existing.rows.length > 0 && !isDeleted) {
  continue; // 跳过，已存在
}

// 2. 语义相似度去重（内容相似但表述不同）
const similarMemories = await this.hybridSearch(trimmedMemory, embedding, {
  limit: 5,
  threshold: 0.1,
});
const highestSimilarity = Math.max(
  ...similarMemories.memories.map((m) => m.score || 0)
);
if (highestSimilarity >= 0.85) {
  // SIMILARITY_THRESHOLD
  continue; // 跳过，语义太相似
}
```

---

## 八、MCP 知识图谱记忆详解

### 8.1 设计目的与适用场景

**MCP Memory Server 是一个独立的知识图谱系统**，与向量记忆系统有本质区别：

| 特性         | 向量记忆系统          | MCP 知识图谱记忆       |
| ------------ | --------------------- | ---------------------- |
| **存储方式** | 向量数据库 (LibSQL)   | JSON 文件              |
| **数据模型** | 扁平化事实列表        | 实体-关系图结构        |
| **搜索方式** | 语义向量相似度        | 关键词文本匹配         |
| **主要用途** | 记住用户偏好/个人信息 | 构建领域知识网络       |
| **触发方式** | 自动（插件驱动）      | 手动（AI 主动调用）    |
| **典型用例** | "用户喜欢咖啡"        | "张三 → 是 → 产品经理" |

### 8.2 知识图谱数据模型

```typescript
// src/main/mcpServers/memory.ts

// 实体结构
interface Entity {
  name: string; // 实体名称，如 "张三"
  entityType: string; // 实体类型，如 "人物", "公司", "概念"
  observations: string[]; // 观察记录，如 ["喜欢喝咖啡", "住在北京"]
}

// 关系结构
interface Relation {
  from: string; // 起始实体，如 "张三"
  to: string; // 目标实体，如 "ABC公司"
  relationType: string; // 关系类型，如 "就职于"
}

// 知识图谱
interface KnowledgeGraph {
  entities: Entity[];
  relations: Relation[];
}
```

**存储位置：** `{configDir}/memory.json`

### 8.3 MCP 工具集

MCP Memory Server 向 AI 暴露以下工具：

| 工具名                | 描述               | 输入参数                                       |
| --------------------- | ------------------ | ---------------------------------------------- |
| `create_entities`     | 创建新实体         | `entities: [{name, entityType, observations}]` |
| `create_relations`    | 创建实体间关系     | `relations: [{from, to, relationType}]`        |
| `add_observations`    | 为现有实体添加观察 | `observations: [{entityName, contents}]`       |
| `delete_entities`     | 删除实体及相关关系 | `entityNames: string[]`                        |
| `delete_observations` | 删除特定观察       | `deletions: [{entityName, observations}]`      |
| `delete_relations`    | 删除特定关系       | `relations: [{from, to, relationType}]`        |
| `read_graph`          | 读取整个知识图谱   | 无                                             |
| `search_nodes`        | 搜索节点           | `query: string`                                |
| `open_nodes`          | 获取特定实体       | `names: string[]`                              |

### 8.4 注入格式

MCP 工具的结果以 **JSON 文本**形式返回给 AI：

```typescript
// 工具调用返回格式
{
  content: [
    {
      type: "text",
      text: JSON.stringify(
        {
          entities: [
            {
              name: "张三",
              entityType: "人物",
              observations: ["产品经理", "喜欢技术分享"],
            },
            {
              name: "ABC公司",
              entityType: "公司",
              observations: ["互联网公司", "成立于2015年"],
            },
          ],
          relations: [
            {
              from: "张三",
              to: "ABC公司",
              relationType: "就职于",
            },
          ],
        },
        null,
        2
      ),
    },
  ];
}
```

### 8.5 召回机制

**搜索方式：简单的文本匹配**

```typescript
// src/main/mcpServers/memory.ts
async searchNodes(query: string): Promise<KnowledgeGraph> {
  const lowerCaseQuery = query.toLowerCase()

  // 在实体名称、类型、观察记录中搜索
  const filteredEntities = Array.from(this.entities.values()).filter(e =>
    e.name.toLowerCase().includes(lowerCaseQuery) ||
    e.entityType.toLowerCase().includes(lowerCaseQuery) ||
    e.observations.some(o => o.toLowerCase().includes(lowerCaseQuery))
  )

  // 只返回筛选后实体之间的关系
  const filteredEntityNames = new Set(filteredEntities.map(e => e.name))
  const filteredRelations = Array.from(this.relations)
    .map(rStr => this._deserializeRelation(rStr))
    .filter(r => filteredEntityNames.has(r.from) && filteredEntityNames.has(r.to))

  return { entities: filteredEntities, relations: filteredRelations }
}
```

### 8.6 ⚠️ 关键真相：MCP 知识图谱不自动提取实体

**这是一个非常重要的区别！** Cherry Studio 的 MCP 知识图谱记忆系统**完全不会自动提取实体**。

与向量记忆系统不同，MCP 知识图谱的设计哲学是：

```
┌─────────────────────────────────────────────────────────────────┐
│                    两种记忆系统的本质区别                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  向量记忆系统 (自动化)              MCP 知识图谱 (工具驱动)      │
│  ┌──────────────────┐              ┌──────────────────┐        │
│  │ 对话结束后       │              │ 对话过程中       │        │
│  │ 系统自动提取事实 │              │ AI 主动调用工具  │        │
│  │ 无需 AI 参与决策 │              │ AI 决定何时存储  │        │
│  └──────────────────┘              └──────────────────┘        │
│                                                                 │
│  实现方式:                          实现方式:                   │
│  • searchOrchestrationPlugin       • MCP Server 暴露工具       │
│  • onRequestEnd 钩子               • 工具被添加到 LLM 请求     │
│  • 后台异步处理                    • LLM 自行决定是否调用      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**为什么不自动提取？**

1. **语义复杂性**：实体和关系的识别比简单的事实提取复杂得多
2. **上下文依赖**：需要理解对话语境才能正确建立关系
3. **灵活性**：让 AI 决定何时/何地存储，更智能
4. **成本控制**：不需要每轮对话都调用额外的 LLM

### 8.7 MCP 工具如何注入到 LLM

#### Step 1: MCP 服务器启动时注册工具

```typescript
// src/main/mcpServers/memory.ts - MCP Server 定义工具
this.server.setRequestHandler(ListToolsRequestSchema, async () => {
  return {
    tools: [
      {
        name: "create_entities",
        description: "Create multiple new entities in the knowledge graph",
        inputSchema: {
          type: "object",
          properties: {
            entities: {
              type: "array",
              items: {
                type: "object",
                properties: {
                  name: {
                    type: "string",
                    description: "The name of the entity",
                  },
                  entityType: {
                    type: "string",
                    description: "The type of the entity",
                  },
                  observations: {
                    type: "array",
                    items: { type: "string" },
                    description:
                      "Observation contents associated with the entity",
                  },
                },
                required: ["name", "entityType"],
              },
            },
          },
          required: ["entities"],
        },
      },
      {
        name: "search_nodes",
        description: "Search nodes in memory based on a query",
        inputSchema: {
          type: "object",
          properties: {
            query: {
              type: "string",
              description:
                "Search query to match entity names, types, and observations",
            },
          },
          required: ["query"],
        },
      },
      // ... 其他工具
    ],
  };
});
```

#### Step 2: MCP 工具转换为 AI SDK 格式

```typescript
// src/renderer/src/aiCore/utils/mcp.ts
export function convertMcpToolsToAiSdkTools(mcpTools: MCPTool[]): ToolSet {
  const tools: ToolSet = {};

  for (const mcpTool of mcpTools) {
    // 使用 mcpTool.id 作为键，确保唯一性
    tools[mcpTool.id] = tool({
      description: mcpTool.description || `Tool from ${mcpTool.serverName}`,
      inputSchema: jsonSchema(mcpTool.inputSchema as JSONSchema7),
      execute: async (params, { toolCallId }) => {
        // 用户确认（如果需要）
        const isAutoApproveEnabled = isToolAutoApproved(mcpTool, server);
        if (!isAutoApproveEnabled) {
          const confirmed = await requestToolConfirmation(toolCallId);
          if (!confirmed) {
            return { content: [{ type: "text", text: "User declined" }] };
          }
        }

        // 调用 MCP 服务器执行工具
        const result = await callMCPTool(toolResponse);
        return result;
      },
    });
  }
  return tools;
}
```

#### Step 3: 工具被添加到 LLM 请求

```typescript
// LLM 请求时，工具被作为参数传入
const response = await generateText({
  model: openai("gpt-4"),
  messages: conversationMessages,
  tools: {
    // MCP Memory Server 的工具
    "create_entities-memory-server": createEntitiesTool,
    "create_relations-memory-server": createRelationsTool,
    "search_nodes-memory-server": searchNodesTool,
    // ... 其他工具
  },
});
```

### 8.8 LLM 如何决定调用知识图谱工具

**关键：完全依赖 LLM 的智能判断**

LLM 会根据以下因素决定是否调用知识图谱工具：

1. **用户明确要求**："帮我记住张三是产品经理"
2. **对话中出现重要信息**：LLM 判断这是值得存储的知识
3. **需要查询已有知识**："张三是做什么的？"

**实际对话流程示例：**

```
┌─────────────────────────────────────────────────────────────────┐
│                    MCP 知识图谱实际工作流程                       │
└─────────────────────────────────────────────────────────────────┘

用户: "张三是ABC公司的产品经理，他负责电商业务"

                    ┌───────────────────┐
                    │   LLM 接收消息    │
                    │   看到可用工具:   │
                    │   • create_entities│
                    │   • create_relations│
                    │   • search_nodes   │
                    └─────────┬─────────┘
                              │
              ┌───────────────▼───────────────┐
              │  LLM 内部决策（不透明）        │
              │  "这里有人物和公司的关系,     │
              │   我应该用知识图谱工具存储"    │
              └───────────────┬───────────────┘
                              │
                    ┌─────────▼─────────┐
                    │  LLM 生成工具调用  │
                    └─────────┬─────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│ Tool Call #1  │    │ Tool Call #2  │    │ Tool Call #3  │
│create_entities│    │create_entities│    │create_relations│
│{              │    │{              │    │{              │
│ entities: [{  │    │ entities: [{  │    │ relations: [{ │
│  name: "张三" │    │  name:"ABC公司"│    │  from: "张三" │
│  type: "人物" │    │  type: "公司" │    │  to: "ABC公司"│
│  observations:│    │  observations:│    │  relationType:│
│  ["产品经理", │    │  ["电商业务"] │    │  "就职于"     │
│   "负责电商"] │    │ }]            │    │ }]            │
│ }]            │    │}              │    │}              │
│}              │    │               │    │               │
└───────┬───────┘    └───────┬───────┘    └───────┬───────┘
        │                    │                     │
        └─────────────┬──────┴─────────────────────┘
                      │
              ┌───────▼────────┐
              │ MCP Server 执行│
              │ 写入 memory.json│
              └───────┬────────┘
                      │
              ┌───────▼────────┐
              │ 返回结果给 LLM │
              │ LLM 继续对话   │
              └────────────────┘
```

### 8.9 知识图谱召回的完整流程

**当用户询问已存储的信息时：**

```
用户: "张三是做什么的？"

┌──────────────────────────────────────────────────────────────┐
│                       召回流程                                │
└──────────────────────────────────────────────────────────────┘

1. LLM 接收到问题
   │
   ▼
2. LLM 判断: "这是关于人物的查询，我应该搜索知识图谱"
   │
   ▼
3. LLM 生成工具调用:
   ┌─────────────────────────────────────────┐
   │ {                                       │
   │   "name": "search_nodes",               │
   │   "arguments": { "query": "张三" }      │
   │ }                                       │
   └─────────────────────────────────────────┘
   │
   ▼
4. MCP Server 执行搜索 (简单文本匹配):
   ┌─────────────────────────────────────────┐
   │ // 在实体名、类型、观察中搜索 "张三"    │
   │ entities.filter(e =>                    │
   │   e.name.includes("张三") ||            │
   │   e.entityType.includes("张三") ||      │
   │   e.observations.some(o =>              │
   │     o.includes("张三"))                 │
   │ )                                       │
   └─────────────────────────────────────────┘
   │
   ▼
5. 返回搜索结果 (JSON 格式):
   ┌─────────────────────────────────────────┐
   │ {                                       │
   │   "entities": [{                        │
   │     "name": "张三",                     │
   │     "entityType": "人物",               │
   │     "observations": [                   │
   │       "产品经理",                       │
   │       "负责电商业务"                    │
   │     ]                                   │
   │   }],                                   │
   │   "relations": [{                       │
   │     "from": "张三",                     │
   │     "to": "ABC公司",                    │
   │     "relationType": "就职于"            │
   │   }]                                    │
   │ }                                       │
   └─────────────────────────────────────────┘
   │
   ▼
6. LLM 收到工具结果，生成回复:
   "张三是ABC公司的产品经理，负责电商业务。"
```

### 8.10 ⚠️ MCP 知识图谱的局限性

由于 Cherry Studio 的 MCP 知识图谱设计，它存在以下局限：

| 局限性            | 详细说明                               |
| ----------------- | -------------------------------------- |
| **无自动提取**    | 不会像向量记忆那样自动从对话中提取信息 |
| **无向量嵌入**    | 使用简单文本匹配，语义理解能力有限     |
| **依赖 LLM 判断** | 如果 LLM 不主动调用，信息不会被存储    |
| **无去重机制**    | 可能创建重复的实体                     |
| **无历史追踪**    | 没有变更历史记录                       |

### 8.11 如何让 LLM 更好地使用知识图谱

如果你想让 LLM 更积极地使用知识图谱，可以在系统提示词中添加指导：

```
你有权访问一个知识图谱记忆系统。当用户提到以下信息时，请主动使用工具存储：

1. 人物信息（姓名、职位、关系）
2. 公司/组织信息
3. 项目和任务的关联
4. 重要事件和日期

可用工具：
- create_entities: 创建实体
- create_relations: 创建关系
- search_nodes: 搜索知识图谱

在回答问题前，如果问题涉及已知实体，请先搜索知识图谱获取信息。
```

### 8.12 向量记忆 vs 知识图谱完整对比

```
┌──────────────────────────────────────────────────────────────────────────┐
│                      两种记忆系统完整对比                                 │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  向量记忆系统 (User Memory)              MCP 知识图谱 (Memory Server)    │
│  ┌─────────────────────────┐            ┌─────────────────────────┐     │
│  │ 扁平化事实存储           │            │ 图结构知识存储           │     │
│  │                         │            │                         │     │
│  │ • "用户喜欢咖啡"        │            │  [张三]                 │     │
│  │ • "用户住在北京"        │            │    ├─就职于→[ABC公司]   │     │
│  │ • "用户是程序员"        │            │    └─认识→[李四]        │     │
│  │                         │            │                         │     │
│  └─────────────────────────┘            └─────────────────────────┘     │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐│
│  │ 特性           │ 向量记忆                │ MCP 知识图谱            ││
│  ├─────────────────────────────────────────────────────────────────────┤│
│  │ 存储方式       │ LibSQL 向量数据库       │ JSON 文件               ││
│  │ 数据模型       │ 扁平化事实列表          │ 实体-关系图结构         ││
│  │ 搜索方式       │ 余弦相似度向量搜索      │ 简单文本匹配            ││
│  │ 实体提取       │ ✅ 自动 (LLM)           │ ❌ 不自动               ││
│  │ 更新触发       │ ✅ 自动 (onRequestEnd)  │ ❌ AI 主动调用          ││
│  │ 召回方式       │ ✅ 自动 (Tool Calling)  │ ❌ AI 主动调用          ││
│  │ 向量嵌入       │ ✅ 支持                 │ ❌ 不支持               ││
│  │ 去重机制       │ ✅ 哈希+向量相似度      │ ❌ 无                   ││
│  │ 历史追踪       │ ✅ memory_history 表    │ ❌ 无                   ││
│  │ 多用户支持     │ ✅ user_id 隔离         │ ❌ 单一文件             ││
│  └─────────────────────────────────────────────────────────────────────┘│
│                                                                          │
│  适用场景:                               适用场景:                       │
│  • 用户偏好记录                          • 人物关系网络                   │
│  • 个人信息存储                          • 项目知识图谱                   │
│  • 对话上下文补充                        • 领域概念关联                   │
│  • 需要语义搜索                          • 需要结构化关系                 │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### 8.13 在自己项目中实现知识图谱记忆

如果你想在自己的项目中实现类似 Cherry Studio 的知识图谱记忆，有两种方案：

#### 方案 A：纯工具驱动（Cherry Studio 方案）

```typescript
// 1. 定义知识图谱工具
const knowledgeGraphTools = {
  create_entity: tool({
    description: "Create an entity in the knowledge graph",
    parameters: z.object({
      name: z.string(),
      type: z.string(),
      observations: z.array(z.string()).optional(),
    }),
    execute: async ({ name, type, observations }) => {
      // 存储到数据库或文件
      await db.entities.create({
        name,
        type,
        observations: observations || [],
      });
      return { success: true, entity: { name, type } };
    },
  }),

  search_entities: tool({
    description: "Search entities in the knowledge graph",
    parameters: z.object({
      query: z.string(),
    }),
    execute: async ({ query }) => {
      const results = await db.entities.search(query);
      return results;
    },
  }),
};

// 2. 在 LLM 请求中提供工具
const response = await generateText({
  model: openai("gpt-4"),
  messages: messages,
  tools: knowledgeGraphTools,
  // LLM 会自行决定是否调用
});
```

**优点**：简单，LLM 自主决策
**缺点**：依赖 LLM 判断，可能遗漏重要信息

#### 方案 B：自动提取 + 知识图谱（增强方案）

```typescript
// 1. 定义实体提取提示词
const ENTITY_EXTRACTION_PROMPT = `从对话中提取实体和关系。

输出格式：
{
  "entities": [
    {"name": "实体名", "type": "类型", "observations": ["观察1"]}
  ],
  "relations": [
    {"from": "实体A", "to": "实体B", "type": "关系类型"}
  ]
}

只提取明确提到的实体和关系。`;

// 2. 对话结束后自动提取
async function extractKnowledgeGraph(messages: Message[]) {
  const response = await generateText({
    model: openai("gpt-4"),
    messages: [
      { role: "system", content: ENTITY_EXTRACTION_PROMPT },
      { role: "user", content: JSON.stringify(messages) },
    ],
  });

  const { entities, relations } = JSON.parse(response.text);

  // 3. 存储到知识图谱
  for (const entity of entities) {
    await upsertEntity(entity); // 使用 upsert 避免重复
  }
  for (const relation of relations) {
    await upsertRelation(relation);
  }
}

// 4. 可选：添加向量嵌入支持语义搜索
async function upsertEntity(entity: Entity) {
  const embedding = await generateEmbedding(
    `${entity.name} ${entity.type} ${entity.observations.join(" ")}`
  );

  await db.execute({
    sql: `INSERT OR REPLACE INTO entities (name, type, observations, embedding) 
          VALUES (?, ?, ?, vector32(?))`,
    args: [
      entity.name,
      entity.type,
      JSON.stringify(entity.observations),
      embedding,
    ],
  });
}
```

**优点**：自动化，不遗漏信息，支持语义搜索
**缺点**：额外的 LLM 调用成本

---

## 九、如何在自己项目中实现

### 9.1 向量记忆系统实现要点

#### 核心依赖

```json
{
  "dependencies": {
    "@libsql/client": "^0.x.x", // 支持向量的 SQLite
    "ai": "^4.x.x" // AI SDK (Tool Calling)
  }
}
```

#### 最小化实现步骤

**Step 1: 数据库初始化**

```typescript
// 创建支持向量的表
const db = createClient({ url: "file:./memories.db" });
await db.execute(`
  CREATE TABLE IF NOT EXISTS memories (
    id TEXT PRIMARY KEY,
    memory TEXT NOT NULL,
    hash TEXT UNIQUE,
    embedding F32_BLOB(1536),
    user_id TEXT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
  )
`);
```

**Step 2: 添加记忆**

```typescript
async function addMemory(text: string, userId: string) {
  // 1. 生成哈希去重
  const hash = crypto.createHash("sha256").update(text).digest("hex");
  const existing = await db.execute({
    sql: "SELECT id FROM memories WHERE hash = ?",
    args: [hash],
  });
  if (existing.rows.length > 0) return null;

  // 2. 生成嵌入向量
  const embedding = await openai.embeddings.create({
    model: "text-embedding-3-small",
    input: text,
  });

  // 3. 向量去重检查
  const similar = await searchMemory(text, userId, 0.85);
  if (similar.length > 0) return null;

  // 4. 插入数据库
  const id = crypto.randomUUID();
  await db.execute({
    sql: "INSERT INTO memories (id, memory, hash, embedding, user_id) VALUES (?, ?, ?, ?, ?)",
    args: [
      id,
      text,
      hash,
      `[${embedding.data[0].embedding.join(",")}]`,
      userId,
    ],
  });
  return id;
}
```

**Step 3: 搜索记忆**

```typescript
async function searchMemory(
  query: string,
  userId: string,
  threshold = 0.5,
  limit = 5
) {
  const embedding = await openai.embeddings.create({
    model: "text-embedding-3-small",
    input: query,
  });
  const vector = `[${embedding.data[0].embedding.join(",")}]`;

  const result = await db.execute({
    sql: `
      SELECT id, memory, (1 - vector_distance_cos(embedding, vector32(?))) as score
      FROM memories
      WHERE user_id = ? AND score >= ?
      ORDER BY score DESC
      LIMIT ?
    `,
    args: [vector, userId, threshold, limit],
  });
  return result.rows;
}
```

**Step 4: 创建记忆搜索工具**

```typescript
import { tool } from "ai";

const memorySearchTool = tool({
  name: "memory_search",
  description: "Search user memories for relevant context",
  parameters: z.object({
    query: z.string().describe("Search query"),
  }),
  execute: async ({ query }) => {
    return await searchMemory(query, currentUserId);
  },
});
```

**Step 5: 对话后自动存储记忆**

```typescript
async function processConversation(messages: Message[]) {
  // 1. 使用 LLM 提取事实
  const facts = await extractFacts(messages);

  // 2. 对每个事实，决定是否添加/更新/删除
  for (const fact of facts) {
    await addMemory(fact, currentUserId);
  }
}
```

### 9.2 知识图谱实现要点

#### 最小化实现

```typescript
// 数据结构
interface Entity {
  name: string;
  type: string;
  observations: string[];
}
interface Relation {
  from: string;
  to: string;
  type: string;
}
interface KnowledgeGraph {
  entities: Entity[];
  relations: Relation[];
}

// 存储
const graph: KnowledgeGraph = { entities: [], relations: [] };

// MCP 工具
const tools = {
  create_entity: (entity: Entity) => {
    if (!graph.entities.find((e) => e.name === entity.name)) {
      graph.entities.push(entity);
      saveToFile();
    }
  },
  create_relation: (relation: Relation) => {
    // 确保两端实体都存在
    if (
      graph.entities.find((e) => e.name === relation.from) &&
      graph.entities.find((e) => e.name === relation.to)
    ) {
      graph.relations.push(relation);
      saveToFile();
    }
  },
  search: (query: string) => {
    const q = query.toLowerCase();
    return {
      entities: graph.entities.filter(
        (e) =>
          e.name.toLowerCase().includes(q) ||
          e.observations.some((o) => o.toLowerCase().includes(q))
      ),
      relations: graph.relations.filter(
        (r) =>
          r.from.toLowerCase().includes(q) || r.to.toLowerCase().includes(q)
      ),
    };
  },
};
```

### 9.3 关键设计决策

| 决策点     | 推荐方案                   | 原因                   |
| ---------- | -------------------------- | ---------------------- |
| 向量数据库 | LibSQL / Postgres+pgvector | 原生向量支持，性能好   |
| 嵌入模型   | text-embedding-3-small     | 性价比高，1536 维      |
| 去重阈值   | 0.85                       | 平衡精确性和召回率     |
| 搜索阈值   | 0.5                        | 较低阈值，保证召回     |
| 事实提取   | 专用 LLM 调用              | 与主对话分离，异步处理 |
| 记忆注入   | Tool Calling               | 标准化，AI 可控        |
