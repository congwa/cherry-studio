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
   │  与现有记忆比较  │  updateMemories()
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
