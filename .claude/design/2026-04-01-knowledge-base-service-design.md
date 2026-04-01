# 独立知识库服务设计文档

**日期**: 2026-04-01  
**状态**: 待实施

---

## 1. 背景与目标

将 FastGPT 知识库能力（文件上传解析、向量检索、音视频处理）抽取为独立微服务，提供 REST API 供外部 Agent 使用。

**核心目标**：
- 独立部署，不依赖 FastGPT 主应用进程
- 复用 `packages/service` 代码，跟随上游 bug 修复自动更新
- 支持完整文件格式解析（含图文混排、Excel、双栏 PDF）
- 支持音视频转文本入库
- 提供向量检索 API 供 Agent 调用

---

## 2. 架构决策

| 维度 | 决策 | 原因 |
|------|------|------|
| 部署形态 | Monorepo 内新 Project | 复用 `packages/service`，上游修复自动生效 |
| 框架 | Bun + Hono | 与 code-sandbox 同构，轻量 |
| 认证 | 独立 API Key | 与 FastGPT 用户体系隔离 |
| 数据库 | 独立 MongoDB + 向量库 | 数据完全隔离 |
| 文件存储 | 独立 MinIO | 与主应用存储隔离 |
| 状态推送 | SSE | 管理界面实时进度，无需 WebSocket |
| 代码管理 | 私有 Fork | 私有扩展，定期 merge 上游 |

---

## 3. 整体架构

```
projects/knowledge-base/
│
│  ┌──────────────┐    ┌──────────────────────────────┐
│  │  Hono Router  │    │       Business Layer         │
│  │              │    │                              │
│  │  GET  /health│───▶│  DatasetService              │
│  │  /v1/datasets│    │  CollectionService           │
│  │  /v1/search  │    │  SearchService               │
│  │  /v1/files   │    │  FileService                 │
│  │  /v1/media   │    │  MediaProcessor (状态机)      │
│  │  /v1/tasks   │    │  SseManager                  │
│  └──────────────┘    └──────────────┬───────────────┘
│         │                           │
│  ┌──────▼──────┐           ┌────────▼────────────┐
│  │  API Key    │           │  packages/service    │
│  │  Auth       │           │  (直接 import)       │
│  │  Middleware │           │  - vectorDB 适配层   │
│  └─────────────┘           │  - 文件解析 Worker   │
│                            │  - 训练队列          │
│                            │  - Textin/Doc2x      │
│                            └─────────────────────┘
│                                      │
│               ┌──────────────────────┼──────────────┐
│               ▼                      ▼              ▼
│          MongoDB(独立)           向量库(独立)    MinIO(独立)
```

---

## 4. API 设计

### 认证
所有接口需携带：`Authorization: Bearer <api-key>`

### 路由清单

```
# 健康检测
GET  /health
→ { status: "ok", version: string }

# 知识库 CRUD
POST   /v1/datasets              创建知识库
GET    /v1/datasets              列表（支持分页）
GET    /v1/datasets/:id          详情
PUT    /v1/datasets/:id          更新名称/模型配置
DELETE /v1/datasets/:id          删除（含所有 Collection 和向量）

# Collection CRUD
POST   /v1/datasets/:id/collections         创建集合（文本/文件/链接）
GET    /v1/datasets/:id/collections         列表
DELETE /v1/datasets/:id/collections/:cid    删除集合

# 文件上传（普通文档）
POST /v1/files/upload
  Body: multipart/form-data { file, datasetId, collectionName? }
  → { collectionId, taskId? }
  支持：txt / md / html / pdf / docx / pptx / xlsx / csv

# 音视频上传
POST /v1/files/upload-media
  Body: multipart/form-data { file, datasetId, collectionName?, contentType? }
  contentType: "meeting" | "tutorial"（影响是否生成摘要 chunk）
  → { taskId: UUID }

# 检索
POST /v1/search
  Body: {
    datasetIds: string[]           // 指定检索范围
    collectionIds?: string[]       // 可选，进一步缩小范围
    query: string
    limit?: number                 // 默认 10
    similarity?: number            // 最低相似度阈值
    searchMode?: "embedding" | "fulltext" | "mixedRecall"
    usingReRank?: boolean
  }
  → { results: SearchResultItem[] }

# 直接推送文本数据
POST   /v1/datasets/:id/data
  Body: { collectionId: string, q: string, a?: string, indexes?: string[] }
  → { dataId: string }
DELETE /v1/datasets/:id/data/:dataId

# 任务状态查询（轮询）
GET /v1/tasks/:taskId
  → { taskId, stage, progress, status, error? }

# 任务状态订阅（SSE）
GET /v1/tasks/:taskId/stream
  Content-Type: text/event-stream
  → 持续推送 TaskEvent
```

### SSE 事件格式

```typescript
type TaskStage =
  | 'uploading'
  | 'extracting_audio'
  | 'uploading_audio'
  | 'transcribing'
  | 'cleaning'
  | 'summarizing'
  | 'training'
  | 'completed'
  | 'failed'

type TaskEvent = {
  taskId: string
  stage: TaskStage
  progress: number        // 0-100
  error?: string          // 仅 failed 时有值
}
```

---

## 5. 数据模型

### 5.1 api_keys 集合
```typescript
{
  _id: ObjectId
  key: string          // bcrypt hash 存储
  name: string         // 备注名
  teamId: string       // 内部租户 ID（UUID）
  createdAt: Date
  lastUsedAt: Date
}
```

### 5.2 datasets / collections / data 集合
直接复用 `packages/service/core/dataset/` 下的 Mongoose Schema，不重复定义。

### 5.3 media_tasks 集合（音视频处理状态机）
```typescript
{
  _id: string               // UUID，即 taskId
  teamId: string
  datasetId: string
  fileName: string
  fileType: 'audio' | 'video'
  contentType: 'meeting' | 'tutorial'  // 影响是否生成摘要

  // 状态机字段
  status: 'pending' | 'processing' | 'completed' | 'failed'
  stage: TaskStage
  progress: number

  // 幂等恢复用（确定性 MinIO key，重试可跳过已完成阶段）
  videoMinioKey: string     // 原始视频文件
  audioMinioKey: string     // 提取的音频（格式：audio/task-{id}.wav）
  transcriptText: string    // 豆包转写结果
  cleanedText: string       // LLM 清洗后文本

  retryCount: number        // >= 3 时标记 failed
  error: string
  createdAt: Date
  updatedAt: Date
}
```

### 5.4 SSE 连接管理（内存）
```typescript
// 不持久化，服务重启后客户端需重新订阅
const sseConnections = new Map<string, Set<SSEConnection>>()
```

---

## 6. 音视频处理 Pipeline

### 完整流程
```
上传音视频文件
    ↓
存入 MinIO（videoMinioKey）
    ↓ 返回 taskId，异步处理开始
[视频] ffmpeg 从 MinIO 预签名 URL 提取音频
    → 本地临时 wav（16kHz, 单声道）
    → 上传音频到 MinIO（audioMinioKey: audio/task-{id}.wav）← 豆包需要 URL
    → 删除本地临时文件
[音频] 直接获取 MinIO 预签名 URL
    ↓
豆包 ASR：提交任务（传入音频 URL）→ 轮询结果 → transcriptText
    ↓
LLM 清洗：去口语化、断句、补标点、分段落 → cleanedText
    ↓
[contentType === 'meeting'] 生成整篇摘要，作为首个 chunk 存入
    ↓
分块 → 向量化 → 写入训练队列（复用 packages/service）
    ↓
成功：删除 MinIO 临时文件（原始视频 + 提取的音频）
失败：保留 MinIO 文件（便于重试跳过已完成阶段）
```

### 进度对照表
| 阶段 | stage | progress |
|------|-------|----------|
| 上传完成，开始处理 | extracting_audio | 5 |
| ffmpeg 提取音频完成 | uploading_audio | 35 |
| 音频上传 MinIO 完成 | transcribing | 50 |
| 豆包提交 ASR | transcribing | 60 |
| ASR 转写完成 | cleaning | 75 |
| LLM 清洗完成 | summarizing / training | 85 |
| 训练队列写入完成 | completed | 100 |

### 中断恢复机制
- `audioMinioKey` 采用确定性命名（`audio/task-{taskId}.wav`）
- 服务重启时扫描 `status=processing` 的任务，按 `stage` 决定从哪步恢复
- `retryCount >= 3` 时标记 `failed`，通过 SSE 推送错误原因
- 各阶段产物（transcriptText、cleanedText）写入 MongoDB 后才更新 stage，保证幂等

---

## 7. 文件解析能力

| 格式 | 解析方式 | 图片支持 | 双栏支持 |
|------|---------|---------|---------|
| txt / md | 直接读取 | — | — |
| html | cheerio | — | — |
| pdf | 系统解析（基础） | ✗ | ✗ |
| pdf | Textin API（需配置） | ✅ | ✅ |
| pdf | Doc2x API（需配置） | ✅ | — |
| docx | mammoth | — | — |
| pptx | pptx2json | — | — |
| xlsx | node-xlsx → Markdown 表格 | — | — |
| csv | PapaParse → Markdown 表格 | — | — |

**图文混排**：Textin/Doc2x 解析 PDF 时提取嵌入图片，上传 MinIO 后以 Markdown 图片引用保留位置，配合 `imageIndex` 标志用 VLM 对图片建向量索引。

---

## 8. 项目目录结构

```
projects/knowledge-base/
├── src/
│   ├── index.ts                  # Hono 入口，挂载路由和中间件
│   ├── config.ts                 # 环境变量（MongoDB/MinIO/向量库/豆包/Textin/Doc2x）
│   ├── middleware/
│   │   └── auth.ts               # API Key 验证（查 MongoDB api_keys 集合）
│   ├── routes/
│   │   ├── health.ts             # GET /health
│   │   ├── datasets.ts           # CRUD /v1/datasets
│   │   ├── collections.ts        # CRUD /v1/datasets/:id/collections
│   │   ├── search.ts             # POST /v1/search
│   │   ├── files.ts              # POST /v1/files/upload（普通文档）
│   │   ├── media.ts              # POST /v1/files/upload-media（音视频）
│   │   └── tasks.ts              # GET /v1/tasks/:id 和 /stream（SSE）
│   ├── services/
│   │   ├── mediaProcessor.ts     # 音视频处理状态机
│   │   └── sse.ts                # SSE 连接管理（Map<taskId, Set<conn>>）
│   └── db/
│       └── connect.ts            # 独立 MongoDB 连接初始化
├── package.json                  # Bun runtime，依赖 hono + @fastgpt/service
└── Dockerfile
```

---

## 9. 环境变量

```env
# 服务
PORT=3010

# 独立 MongoDB
KNOWLEDGE_MONGODB_URI=mongodb://...

# 向量库（四选一，与主应用同类型但独立实例）
PG_ADDRESS=
MILVUS_ADDRESS=
OCEANBASE_ADDRESS=
SEEKDB_ADDRESS=

# 独立 MinIO
KNOWLEDGE_MINIO_ENDPOINT=
KNOWLEDGE_MINIO_ACCESS_KEY=
KNOWLEDGE_MINIO_SECRET_KEY=
KNOWLEDGE_MINIO_BUCKET=

# 豆包 ASR
VOLCENGINE_ASR_APP_ID=
VOLCENGINE_ASR_ACCESS_TOKEN=

# PDF 高级解析（可选，不配置则使用基础解析）
TEXTIN_APP_ID=
TEXTIN_SECRET_CODE=
DOC2X_API_KEY=

# LLM（清洗/摘要）
LLM_BASE_URL=
LLM_API_KEY=
LLM_MODEL=
```

---

## 10. 故障处理

| 场景 | 处理方式 |
|------|---------|
| ffmpeg 失败 | 标记 failed，保留 MinIO 文件，SSE 推送错误 |
| 豆包 ASR 超时 | retryCount++，重试最多 3 次，超限标 failed |
| LLM 清洗失败 | retryCount++，重试最多 3 次 |
| 服务重启 | 启动时扫描 processing 任务，按 stage 从断点恢复 |
| MinIO 上传失败 | 本地临时文件保留，重试时重新上传 |
| 训练队列满 | packages/service 内置背压处理，无需额外处理 |

---

## 11. 代码管理

- 仓库：Fork `labring/FastGPT` 为私有仓库
- 新增代码仅在 `projects/knowledge-base/` 目录，不修改原有文件
- 同步上游：`git fetch origin main && git merge origin/main`
- 冲突概率极低（新增目录与上游无交叉）
