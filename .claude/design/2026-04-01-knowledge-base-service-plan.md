# Knowledge Base Service 实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在 `projects/knowledge-base/` 下创建独立知识库微服务，提供文件上传解析、向量检索、音视频处理的 REST API。

**Architecture:** Bun + Hono 独立服务，直接 import `packages/service` 复用 FastGPT 向量库适配层、文件解析、训练队列逻辑。独立 MongoDB + 向量库 + MinIO，不依赖 FastGPT 主应用进程。

**Tech Stack:** Bun, Hono, MongoDB/Mongoose, `@fastgpt/service`（workspace import），MinIO SDK，Vitest

---

## 文件结构

```
projects/knowledge-base/
├── src/
│   ├── index.ts                 # Hono 入口，挂载路由、中间件、启动初始化
│   ├── config.ts                # 环境变量 zod 解析 + global.systemEnv 初始化
│   ├── db/
│   │   ├── connect.ts           # MongoDB 连接
│   │   └── minio.ts             # MinIO 客户端
│   ├── models/
│   │   ├── apiKey.ts            # API Key Mongoose schema
│   │   └── mediaTask.ts         # media_tasks Mongoose schema（状态机）
│   ├── middleware/
│   │   └── auth.ts              # Bearer Token 验证，注入 teamId 到 ctx
│   ├── routes/
│   │   ├── health.ts            # GET /health
│   │   ├── datasets.ts          # CRUD /v1/datasets
│   │   ├── collections.ts       # CRUD /v1/datasets/:id/collections
│   │   ├── data.ts              # POST/DELETE /v1/datasets/:id/data
│   │   ├── search.ts            # POST /v1/search
│   │   ├── files.ts             # POST /v1/files/upload（普通文档）
│   │   ├── media.ts             # POST /v1/files/upload-media（音视频）
│   │   └── tasks.ts             # GET /v1/tasks/:id 和 /stream（SSE）
│   ├── services/
│   │   ├── sse.ts               # SSE 连接管理
│   │   ├── trainingWorker.ts    # 后台向量化 worker
│   │   └── mediaProcessor.ts   # 音视频处理状态机
│   └── lib/
│       └── ffmpeg.ts            # ffmpeg 音频提取封装
├── test/
│   ├── health.test.ts
│   ├── datasets.test.ts
│   ├── search.test.ts
│   └── mediaProcessor.test.ts
├── package.json
├── tsconfig.json
└── Dockerfile
```

---

## Task 1: 项目脚手架

**Files:**
- Create: `projects/knowledge-base/package.json`
- Create: `projects/knowledge-base/tsconfig.json`
- Create: `projects/knowledge-base/src/index.ts`（空入口）

- [ ] **Step 1: 创建 package.json**

```json
{
  "name": "knowledge-base",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "dev": "bun run --watch src/index.ts",
    "start": "bun run src/index.ts",
    "build": "sh build.sh",
    "test": "vitest run",
    "test:watch": "vitest"
  },
  "engines": {
    "node": ">=20",
    "pnpm": "9.x"
  },
  "dependencies": {
    "@fastgpt/service": "workspace:*",
    "@fastgpt/global": "workspace:*",
    "hono": "^4.7.6",
    "mongoose": "catalog:",
    "minio": "^8.0.5",
    "zod": "catalog:",
    "bcryptjs": "^3.0.2",
    "uuid": "^11.1.0",
    "dotenv": "^17.3.1",
    "axios": "catalog:",
    "lodash": "catalog:",
    "date-fns": "catalog:"
  },
  "devDependencies": {
    "@types/bun": "latest",
    "@types/bcryptjs": "^2.4.6",
    "@types/uuid": "^10.0.0",
    "@types/lodash": "catalog:",
    "typescript": "catalog:",
    "vitest": "catalog:"
  }
}
```

- [ ] **Step 2: 创建 tsconfig.json**

```json
{
  "extends": "../../tsconfig.json",
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "esModuleInterop": true,
    "strict": true,
    "skipLibCheck": true,
    "outDir": "./dist",
    "rootDir": "./src",
    "declaration": true,
    "resolveJsonModule": true,
    "types": ["@types/bun"]
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "test"]
}
```

- [ ] **Step 3: 创建最小入口 src/index.ts**

```typescript
import { Hono } from 'hono';

const app = new Hono();

app.get('/health', (c) => c.json({ status: 'ok' }));

export default {
  port: process.env.PORT || 3010,
  fetch: app.fetch
};
```

- [ ] **Step 4: 安装依赖**

在 FastGPT 根目录执行：
```bash
pnpm install
```

- [ ] **Step 5: 验证服务可启动**

```bash
cd projects/knowledge-base && bun run dev
```

Expected: 输出 `Listening on http://localhost:3010`，访问 `http://localhost:3010/health` 返回 `{"status":"ok"}`

- [ ] **Step 6: Commit**

```bash
git add projects/knowledge-base/package.json projects/knowledge-base/tsconfig.json projects/knowledge-base/src/index.ts
git commit -m "feat(knowledge-base): project scaffolding with Bun + Hono"
```

---

## Task 2: 配置与 global.systemEnv 初始化

**Files:**
- Create: `projects/knowledge-base/src/config.ts`
- Modify: `projects/knowledge-base/src/index.ts`

- [ ] **Step 1: 创建 src/config.ts**

```typescript
import { z } from 'zod';

const ConfigSchema = z.object({
  PORT: z.string().default('3010'),
  KNOWLEDGE_MONGODB_URI: z.string().min(1),

  // 向量库（四选一）
  PG_ADDRESS: z.string().optional(),
  MILVUS_ADDRESS: z.string().optional(),
  OCEANBASE_ADDRESS: z.string().optional(),
  SEEKDB_ADDRESS: z.string().optional(),

  // MinIO
  KNOWLEDGE_MINIO_ENDPOINT: z.string().min(1),
  KNOWLEDGE_MINIO_ACCESS_KEY: z.string().min(1),
  KNOWLEDGE_MINIO_SECRET_KEY: z.string().min(1),
  KNOWLEDGE_MINIO_BUCKET: z.string().default('knowledge-base'),
  KNOWLEDGE_MINIO_PORT: z.string().default('9000'),
  KNOWLEDGE_MINIO_USE_SSL: z.string().default('false'),

  // 豆包 ASR
  VOLCENGINE_ASR_APP_ID: z.string().optional(),
  VOLCENGINE_ASR_ACCESS_TOKEN: z.string().optional(),
  VOLCENGINE_ASR_POLL_INTERVAL: z.string().default('5'),
  VOLCENGINE_ASR_POLL_TIMEOUT: z.string().default('3600'),

  // PDF 高级解析（可选）
  TEXTIN_APP_ID: z.string().optional(),
  TEXTIN_SECRET_CODE: z.string().optional(),
  DOC2X_API_KEY: z.string().optional(),

  // LLM（清洗/摘要）
  LLM_BASE_URL: z.string().optional(),
  LLM_API_KEY: z.string().optional(),
  LLM_MODEL: z.string().default('gpt-4o-mini'),

  // 训练 Worker 并发
  VECTOR_MAX_PROCESS: z.string().default('10'),
});

export type AppConfig = z.infer<typeof ConfigSchema>;

export const loadConfig = (): AppConfig => {
  const result = ConfigSchema.safeParse(process.env);
  if (!result.success) {
    console.error('Config validation failed:', result.error.format());
    process.exit(1);
  }
  return result.data;
};

export const initSystemEnv = (config: AppConfig) => {
  // 初始化 packages/service 依赖的 global.systemEnv
  global.systemEnv = {
    vectorMaxProcess: Number(config.VECTOR_MAX_PROCESS),
    customPdfParse: {
      textinAppId: config.TEXTIN_APP_ID,
      textinSecretCode: config.TEXTIN_SECRET_CODE,
      doc2xKey: config.DOC2X_API_KEY,
      url: undefined,
      key: undefined,
      price: 0,
    },
  } as any;

  global.feConfigs = {} as any;
  global.subPlans = undefined as any;
  global.vectorQueueLen = 0;
};
```

- [ ] **Step 2: 在 src/index.ts 启动时调用配置初始化**

```typescript
import 'dotenv/config';
import { Hono } from 'hono';
import { loadConfig, initSystemEnv } from './config';

const config = loadConfig();
initSystemEnv(config);

const app = new Hono();
app.get('/health', (c) => c.json({ status: 'ok', version: '1.0.0' }));

export default {
  port: Number(config.PORT),
  fetch: app.fetch
};
```

- [ ] **Step 3: 创建 .env.example**

在 `projects/knowledge-base/` 下创建：
```env
PORT=3010
KNOWLEDGE_MONGODB_URI=mongodb://localhost:27017/knowledge-base
PG_ADDRESS=postgresql://user:pass@localhost:5432/fastgpt_kb
KNOWLEDGE_MINIO_ENDPOINT=localhost
KNOWLEDGE_MINIO_ACCESS_KEY=minioadmin
KNOWLEDGE_MINIO_SECRET_KEY=minioadmin
KNOWLEDGE_MINIO_BUCKET=knowledge-base
KNOWLEDGE_MINIO_PORT=9000
KNOWLEDGE_MINIO_USE_SSL=false
VOLCENGINE_ASR_APP_ID=
VOLCENGINE_ASR_ACCESS_TOKEN=
TEXTIN_APP_ID=
TEXTIN_SECRET_CODE=
DOC2X_API_KEY=
LLM_BASE_URL=https://api.openai.com/v1
LLM_API_KEY=
LLM_MODEL=gpt-4o-mini
VECTOR_MAX_PROCESS=10
```

- [ ] **Step 4: Commit**

```bash
git add projects/knowledge-base/src/config.ts projects/knowledge-base/src/index.ts projects/knowledge-base/.env.example
git commit -m "feat(knowledge-base): config loading and global.systemEnv init"
```

---

## Task 3: MongoDB 连接 + MinIO 客户端

**Files:**
- Create: `projects/knowledge-base/src/db/connect.ts`
- Create: `projects/knowledge-base/src/db/minio.ts`
- Modify: `projects/knowledge-base/src/index.ts`

- [ ] **Step 1: 写失败测试（MongoDB 连接失败时进程退出）**

创建 `projects/knowledge-base/test/db.test.ts`：
```typescript
import { describe, it, expect } from 'vitest';

describe('db connect', () => {
  it('should export connectMongo function', async () => {
    const mod = await import('../src/db/connect');
    expect(typeof mod.connectMongo).toBe('function');
  });
});
```

运行：
```bash
cd projects/knowledge-base && bun test test/db.test.ts
```
Expected: FAIL（模块不存在）

- [ ] **Step 2: 创建 src/db/connect.ts**

```typescript
import mongoose from 'mongoose';
import type { AppConfig } from '../config';

export const connectMongo = async (config: AppConfig): Promise<void> => {
  await mongoose.connect(config.KNOWLEDGE_MONGODB_URI, {
    maxPoolSize: 10,
    serverSelectionTimeoutMS: 5000,
  });
  console.log('[KB] MongoDB connected:', config.KNOWLEDGE_MONGODB_URI.replace(/\/\/.*@/, '//***@'));
};
```

- [ ] **Step 3: 运行测试验证通过**

```bash
cd projects/knowledge-base && bun test test/db.test.ts
```
Expected: PASS

- [ ] **Step 4: 创建 src/db/minio.ts**

```typescript
import { Client } from 'minio';
import type { AppConfig } from '../config';

let minioClient: Client;

export const initMinio = (config: AppConfig): void => {
  minioClient = new Client({
    endPoint: config.KNOWLEDGE_MINIO_ENDPOINT,
    port: Number(config.KNOWLEDGE_MINIO_PORT),
    useSSL: config.KNOWLEDGE_MINIO_USE_SSL === 'true',
    accessKey: config.KNOWLEDGE_MINIO_ACCESS_KEY,
    secretKey: config.KNOWLEDGE_MINIO_SECRET_KEY,
  });
};

export const getMinio = (): Client => {
  if (!minioClient) throw new Error('MinIO not initialized');
  return minioClient;
};

export const ensureBucket = async (bucket: string): Promise<void> => {
  const client = getMinio();
  const exists = await client.bucketExists(bucket);
  if (!exists) {
    await client.makeBucket(bucket);
    console.log(`[KB] MinIO bucket created: ${bucket}`);
  }
};

export const putObject = async (
  bucket: string,
  key: string,
  buffer: Buffer,
  contentType = 'application/octet-stream'
): Promise<void> => {
  await getMinio().putObject(bucket, key, buffer, buffer.length, {
    'Content-Type': contentType,
  });
};

export const getPresignedUrl = async (
  bucket: string,
  key: string,
  expirySeconds = 3600
): Promise<string> => {
  return getMinio().presignedGetObject(bucket, key, expirySeconds);
};

export const removeObject = async (bucket: string, key: string): Promise<void> => {
  try {
    await getMinio().removeObject(bucket, key);
  } catch {
    console.warn(`[KB] MinIO remove failed: ${key}`);
  }
};

export const objectExists = async (bucket: string, key: string): Promise<boolean> => {
  try {
    await getMinio().statObject(bucket, key);
    return true;
  } catch {
    return false;
  }
};
```

- [ ] **Step 5: 在 index.ts 启动时初始化 DB + MinIO**

```typescript
import 'dotenv/config';
import { Hono } from 'hono';
import { loadConfig, initSystemEnv } from './config';
import { connectMongo } from './db/connect';
import { initMinio, ensureBucket } from './db/minio';

const config = loadConfig();
initSystemEnv(config);

const app = new Hono();
app.get('/health', (c) => c.json({ status: 'ok', version: '1.0.0' }));

const start = async () => {
  await connectMongo(config);
  initMinio(config);
  await ensureBucket(config.KNOWLEDGE_MINIO_BUCKET);
  console.log(`[KB] Server starting on port ${config.PORT}`);
};

start().catch((err) => {
  console.error('[KB] Startup failed:', err);
  process.exit(1);
});

export default {
  port: Number(config.PORT),
  fetch: app.fetch
};
```

- [ ] **Step 6: Commit**

```bash
git add projects/knowledge-base/src/db/ projects/knowledge-base/src/index.ts projects/knowledge-base/test/db.test.ts
git commit -m "feat(knowledge-base): MongoDB and MinIO initialization"
```

---

## Task 4: API Key 模型 + Auth 中间件

**Files:**
- Create: `projects/knowledge-base/src/models/apiKey.ts`
- Create: `projects/knowledge-base/src/middleware/auth.ts`
- Create: `projects/knowledge-base/test/auth.test.ts`

- [ ] **Step 1: 写失败测试**

创建 `projects/knowledge-base/test/auth.test.ts`：
```typescript
import { describe, it, expect } from 'vitest';

describe('auth middleware', () => {
  it('should export authMiddleware', async () => {
    const mod = await import('../src/middleware/auth');
    expect(typeof mod.authMiddleware).toBe('function');
  });
});
```

```bash
cd projects/knowledge-base && bun test test/auth.test.ts
```
Expected: FAIL

- [ ] **Step 2: 创建 src/models/apiKey.ts**

```typescript
import mongoose, { Schema } from 'mongoose';

export type ApiKeyDoc = {
  _id: mongoose.Types.ObjectId;
  key: string;       // bcrypt hash
  name: string;
  teamId: string;    // UUID
  createdAt: Date;
  lastUsedAt: Date;
};

const ApiKeySchema = new Schema<ApiKeyDoc>(
  {
    key: { type: String, required: true, index: true },
    name: { type: String, required: true },
    teamId: { type: String, required: true },
    lastUsedAt: { type: Date, default: Date.now },
  },
  { timestamps: true }
);

export const MongoApiKey = mongoose.model<ApiKeyDoc>('api_keys', ApiKeySchema);
```

- [ ] **Step 3: 创建 src/middleware/auth.ts**

```typescript
import type { MiddlewareHandler } from 'hono';
import bcrypt from 'bcryptjs';
import { MongoApiKey } from '../models/apiKey';

declare module 'hono' {
  interface ContextVariableMap {
    teamId: string;
  }
}

export const authMiddleware: MiddlewareHandler = async (c, next) => {
  const header = c.req.header('Authorization');
  if (!header?.startsWith('Bearer ')) {
    return c.json({ error: 'Missing or invalid Authorization header' }, 401);
  }

  const token = header.slice(7);

  // 查找所有 key（实际可加 Redis 缓存优化）
  const keys = await MongoApiKey.find({}).lean();
  const matched = keys.find((k) => bcrypt.compareSync(token, k.key));

  if (!matched) {
    return c.json({ error: 'Invalid API key' }, 401);
  }

  // 异步更新 lastUsedAt，不阻塞请求
  MongoApiKey.updateOne({ _id: matched._id }, { lastUsedAt: new Date() }).exec();

  c.set('teamId', matched.teamId);
  await next();
};
```

- [ ] **Step 4: 运行测试**

```bash
cd projects/knowledge-base && bun test test/auth.test.ts
```
Expected: PASS

- [ ] **Step 5: 在 index.ts 挂载 auth 中间件到 /v1 路由组**

```typescript
import { authMiddleware } from './middleware/auth';

// 在 start() 前
const v1 = new Hono();
v1.use('*', authMiddleware);
app.route('/v1', v1);
```

- [ ] **Step 6: Commit**

```bash
git add projects/knowledge-base/src/models/apiKey.ts projects/knowledge-base/src/middleware/auth.ts projects/knowledge-base/test/auth.test.ts projects/knowledge-base/src/index.ts
git commit -m "feat(knowledge-base): API key model and auth middleware"
```

---

## Task 5: Health 端点 + Dataset CRUD

**Files:**
- Create: `projects/knowledge-base/src/routes/health.ts`
- Create: `projects/knowledge-base/src/routes/datasets.ts`
- Create: `projects/knowledge-base/test/datasets.test.ts`
- Modify: `projects/knowledge-base/src/index.ts`

- [ ] **Step 1: 创建 src/routes/health.ts**

```typescript
import { Hono } from 'hono';

export const healthRouter = new Hono();

healthRouter.get('/', (c) =>
  c.json({ status: 'ok', version: '1.0.0', timestamp: new Date().toISOString() })
);
```

- [ ] **Step 2: 写失败测试（Dataset CRUD）**

创建 `projects/knowledge-base/test/datasets.test.ts`：
```typescript
import { describe, it, expect } from 'vitest';

describe('datasets router', () => {
  it('should export datasetsRouter', async () => {
    const mod = await import('../src/routes/datasets');
    expect(mod.datasetsRouter).toBeDefined();
  });
});
```

```bash
cd projects/knowledge-base && bun test test/datasets.test.ts
```
Expected: FAIL

- [ ] **Step 3: 创建 src/routes/datasets.ts**

```typescript
import { Hono } from 'hono';
import { zValidator } from '@hono/zod-validator';
import { z } from 'zod';
import { MongoDataset } from '@fastgpt/service/core/dataset/schema';
import { v4 as uuidv4 } from 'uuid';

export const datasetsRouter = new Hono();

const CreateDatasetSchema = z.object({
  name: z.string().min(1).max(100),
  vectorModel: z.string().default('text-embedding-3-small'),
  agentModel: z.string().default('gpt-4o-mini'),
});

const UpdateDatasetSchema = z.object({
  name: z.string().min(1).max(100).optional(),
  vectorModel: z.string().optional(),
  agentModel: z.string().optional(),
});

// 创建知识库
datasetsRouter.post('/', zValidator('json', CreateDatasetSchema), async (c) => {
  const teamId = c.get('teamId');
  const body = c.req.valid('json');

  const dataset = await MongoDataset.create({
    teamId,
    tmbId: teamId,
    name: body.name,
    vectorModel: body.vectorModel,
    agentModel: body.agentModel,
    type: 'dataset',
  });

  return c.json({ id: dataset._id, name: dataset.name }, 201);
});

// 列表
datasetsRouter.get('/', async (c) => {
  const teamId = c.get('teamId');
  const page = Number(c.req.query('page') || '1');
  const limit = Math.min(Number(c.req.query('limit') || '20'), 100);

  const [total, datasets] = await Promise.all([
    MongoDataset.countDocuments({ teamId }),
    MongoDataset.find({ teamId })
      .select('_id name vectorModel agentModel createdAt')
      .skip((page - 1) * limit)
      .limit(limit)
      .lean(),
  ]);

  return c.json({ total, page, limit, items: datasets });
});

// 详情
datasetsRouter.get('/:id', async (c) => {
  const teamId = c.get('teamId');
  const dataset = await MongoDataset.findOne({ _id: c.req.param('id'), teamId }).lean();
  if (!dataset) return c.json({ error: 'Not found' }, 404);
  return c.json(dataset);
});

// 更新
datasetsRouter.put('/:id', zValidator('json', UpdateDatasetSchema), async (c) => {
  const teamId = c.get('teamId');
  const body = c.req.valid('json');

  const dataset = await MongoDataset.findOneAndUpdate(
    { _id: c.req.param('id'), teamId },
    { $set: body },
    { new: true }
  ).lean();

  if (!dataset) return c.json({ error: 'Not found' }, 404);
  return c.json(dataset);
});

// 删除（含所有 Collection + 向量）
datasetsRouter.delete('/:id', async (c) => {
  const teamId = c.get('teamId');
  const datasetId = c.req.param('id');

  const dataset = await MongoDataset.findOne({ _id: datasetId, teamId });
  if (!dataset) return c.json({ error: 'Not found' }, 404);

  const { deleteDatasetDataVector } = await import(
    '@fastgpt/service/common/vectorDB/controller'
  );
  const { MongoDatasetCollection } = await import(
    '@fastgpt/service/core/dataset/collection/schema'
  );
  const { MongoDatasetData } = await import('@fastgpt/service/core/dataset/data/schema');

  await deleteDatasetDataVector({ teamId, datasetIds: [datasetId], collectionIds: [] });
  await MongoDatasetData.deleteMany({ teamId, datasetId });
  await MongoDatasetCollection.deleteMany({ teamId, datasetId });
  await MongoDataset.deleteOne({ _id: datasetId, teamId });

  return c.json({ success: true });
});
```

- [ ] **Step 4: 运行测试**

```bash
cd projects/knowledge-base && bun test test/datasets.test.ts
```
Expected: PASS

- [ ] **Step 5: 注册路由到 index.ts**

在 `src/index.ts` 的 v1 路由组中添加：
```typescript
import { healthRouter } from './routes/health';
import { datasetsRouter } from './routes/datasets';

app.route('/health', healthRouter);
v1.route('/datasets', datasetsRouter);
```

- [ ] **Step 6: Commit**

```bash
git add projects/knowledge-base/src/routes/health.ts projects/knowledge-base/src/routes/datasets.ts projects/knowledge-base/test/datasets.test.ts projects/knowledge-base/src/index.ts
git commit -m "feat(knowledge-base): health endpoint and dataset CRUD"
```

---

## Task 6: Collection CRUD + Data Push

**Files:**
- Create: `projects/knowledge-base/src/routes/collections.ts`
- Create: `projects/knowledge-base/src/routes/data.ts`
- Modify: `projects/knowledge-base/src/index.ts`

- [ ] **Step 1: 创建 src/routes/collections.ts**

```typescript
import { Hono } from 'hono';
import { zValidator } from '@hono/zod-validator';
import { z } from 'zod';
import { MongoDatasetCollection } from '@fastgpt/service/core/dataset/collection/schema';
import {
  DatasetCollectionTypeEnum,
  TrainingModeEnum,
} from '@fastgpt/global/core/dataset/constants';

export const collectionsRouter = new Hono<{ Variables: { teamId: string } }>();

const CreateCollectionSchema = z.object({
  name: z.string().min(1).max(200),
  type: z.enum(['text', 'file', 'link']).default('text'),
  rawLink: z.string().url().optional(),
});

// 创建
collectionsRouter.post('/', zValidator('json', CreateCollectionSchema), async (c) => {
  const teamId = c.get('teamId');
  const datasetId = c.req.param('datasetId');
  const body = c.req.valid('json');

  const col = await MongoDatasetCollection.create({
    teamId,
    tmbId: teamId,
    datasetId,
    name: body.name,
    type:
      body.type === 'link'
        ? DatasetCollectionTypeEnum.link
        : DatasetCollectionTypeEnum.file,
    trainingType: TrainingModeEnum.chunk,
    rawLink: body.rawLink,
  });

  return c.json({ id: col._id, name: col.name }, 201);
});

// 列表
collectionsRouter.get('/', async (c) => {
  const teamId = c.get('teamId');
  const datasetId = c.req.param('datasetId');

  const items = await MongoDatasetCollection.find({ teamId, datasetId })
    .select('_id name type trainingType createdAt')
    .lean();

  return c.json({ items });
});

// 删除
collectionsRouter.delete('/:cid', async (c) => {
  const teamId = c.get('teamId');
  const datasetId = c.req.param('datasetId');
  const collectionId = c.req.param('cid');

  const col = await MongoDatasetCollection.findOne({ _id: collectionId, teamId, datasetId });
  if (!col) return c.json({ error: 'Not found' }, 404);

  const { deleteDatasetDataVector } = await import(
    '@fastgpt/service/common/vectorDB/controller'
  );
  const { MongoDatasetData } = await import('@fastgpt/service/core/dataset/data/schema');

  await deleteDatasetDataVector({ teamId, datasetIds: [datasetId], collectionIds: [collectionId] });
  await MongoDatasetData.deleteMany({ teamId, datasetId, collectionId });
  await MongoDatasetCollection.deleteOne({ _id: collectionId });

  return c.json({ success: true });
});
```

- [ ] **Step 2: 创建 src/routes/data.ts**

```typescript
import { Hono } from 'hono';
import { zValidator } from '@hono/zod-validator';
import { z } from 'zod';
import { MongoDatasetData } from '@fastgpt/service/core/dataset/data/schema';
import { insertDatasetDataVector } from '@fastgpt/service/common/vectorDB/controller';
import { getEmbeddingModel } from '@fastgpt/service/core/ai/model';
import { MongoDataset } from '@fastgpt/service/core/dataset/schema';

export const dataRouter = new Hono<{ Variables: { teamId: string } }>();

const PushDataSchema = z.object({
  collectionId: z.string().min(1),
  q: z.string().min(1),
  a: z.string().optional(),
  indexes: z.array(z.string()).optional(),
});

// 推送文本数据
dataRouter.post('/', zValidator('json', PushDataSchema), async (c) => {
  const teamId = c.get('teamId');
  const datasetId = c.req.param('datasetId');
  const { collectionId, q, a, indexes } = c.req.valid('json');

  const dataset = await MongoDataset.findOne({ _id: datasetId, teamId }).lean();
  if (!dataset) return c.json({ error: 'Dataset not found' }, 404);

  const model = getEmbeddingModel(dataset.vectorModel);
  const texts = [q, ...(indexes || [])];

  const { insertIds } = await insertDatasetDataVector({
    teamId,
    datasetId,
    collectionId,
    model,
    inputs: texts,
  });

  const dataDoc = await MongoDatasetData.create({
    teamId,
    tmbId: teamId,
    datasetId,
    collectionId,
    q,
    a: a || '',
    indexes: (indexes || []).map((text, i) => ({
      defaultIndex: i === 0,
      type: 'custom',
      text,
      dataId: insertIds[i + 1] || insertIds[0],
    })),
    dataId: insertIds[0],
  });

  return c.json({ dataId: dataDoc._id }, 201);
});

// 删除数据条目
dataRouter.delete('/:dataId', async (c) => {
  const teamId = c.get('teamId');
  const datasetId = c.req.param('datasetId');
  const dataId = c.req.param('dataId');

  const data = await MongoDatasetData.findOne({ _id: dataId, teamId, datasetId }).lean();
  if (!data) return c.json({ error: 'Not found' }, 404);

  const { deleteDatasetDataVector } = await import(
    '@fastgpt/service/common/vectorDB/controller'
  );
  await deleteDatasetDataVector({ teamId, id: data.dataId });
  await MongoDatasetData.deleteOne({ _id: dataId });

  return c.json({ success: true });
});
```

- [ ] **Step 3: 注册到 index.ts**

```typescript
import { collectionsRouter } from './routes/collections';
import { dataRouter } from './routes/data';

// 在 v1 路由组下（collections 挂在 dataset 子路径）
v1.route('/datasets/:datasetId/collections', collectionsRouter);
v1.route('/datasets/:datasetId/data', dataRouter);
```

- [ ] **Step 4: Commit**

```bash
git add projects/knowledge-base/src/routes/collections.ts projects/knowledge-base/src/routes/data.ts projects/knowledge-base/src/index.ts
git commit -m "feat(knowledge-base): collection CRUD and data push endpoints"
```

---

## Task 7: 向量检索 API

**Files:**
- Create: `projects/knowledge-base/src/routes/search.ts`
- Create: `projects/knowledge-base/test/search.test.ts`
- Modify: `projects/knowledge-base/src/index.ts`

- [ ] **Step 1: 写失败测试**

创建 `projects/knowledge-base/test/search.test.ts`：
```typescript
import { describe, it, expect } from 'vitest';

describe('search router', () => {
  it('should export searchRouter', async () => {
    const mod = await import('../src/routes/search');
    expect(mod.searchRouter).toBeDefined();
  });
});
```

```bash
cd projects/knowledge-base && bun test test/search.test.ts
```
Expected: FAIL

- [ ] **Step 2: 创建 src/routes/search.ts**

```typescript
import { Hono } from 'hono';
import { zValidator } from '@hono/zod-validator';
import { z } from 'zod';
import { defaultSearchDatasetData } from '@fastgpt/service/core/dataset/search/controller';
import { DatasetSearchModeEnum } from '@fastgpt/global/core/dataset/constants';
import { NodeInputKeyEnum } from '@fastgpt/global/core/workflow/constants';

export const searchRouter = new Hono<{ Variables: { teamId: string } }>();

const SearchSchema = z.object({
  datasetIds: z.array(z.string()).min(1),
  collectionIds: z.array(z.string()).optional(),
  query: z.string().min(1),
  limit: z.number().int().min(1).max(100).default(10),
  similarity: z.number().min(0).max(1).optional(),
  searchMode: z
    .enum(['embedding', 'fulltext', 'mixedRecall'])
    .default('mixedRecall'),
  usingReRank: z.boolean().default(false),
  embeddingWeight: z.number().min(0).max(1).optional(),
});

searchRouter.post('/', zValidator('json', SearchSchema), async (c) => {
  const teamId = c.get('teamId');
  const body = c.req.valid('json');

  const results = await defaultSearchDatasetData({
    teamId,
    histories: [],
    model: '',
    datasetIds: body.datasetIds,
    reRankQuery: body.query,
    queries: [body.query],
    [NodeInputKeyEnum.datasetMaxTokens]: 10000,
    [NodeInputKeyEnum.datasetSimilarity]: body.similarity,
    [NodeInputKeyEnum.datasetSearchMode]: body.searchMode as any,
    [NodeInputKeyEnum.datasetSearchUsingReRank]: body.usingReRank,
    [NodeInputKeyEnum.datasetSearchEmbeddingWeight]: body.embeddingWeight,
    forbidCollectionIdList: [],
    filterCollectionIdList: body.collectionIds,
  });

  return c.json({
    results: results.searchRes.slice(0, body.limit).map((r) => ({
      id: r.id,
      q: r.q,
      a: r.a,
      score: r.score,
      collectionId: r.collectionId,
      datasetId: r.datasetId,
      sourceName: r.sourceName,
    })),
  });
});
```

- [ ] **Step 3: 运行测试**

```bash
cd projects/knowledge-base && bun test test/search.test.ts
```
Expected: PASS

- [ ] **Step 4: 注册到 index.ts**

```typescript
import { searchRouter } from './routes/search';
v1.route('/search', searchRouter);
```

- [ ] **Step 5: Commit**

```bash
git add projects/knowledge-base/src/routes/search.ts projects/knowledge-base/test/search.test.ts projects/knowledge-base/src/index.ts
git commit -m "feat(knowledge-base): vector search endpoint"
```

---

## Task 8: 训练 Worker（后台向量化）

**Files:**
- Create: `projects/knowledge-base/src/services/trainingWorker.ts`
- Modify: `projects/knowledge-base/src/index.ts`

- [ ] **Step 1: 创建 src/services/trainingWorker.ts**

从 `projects/app/src/service/core/dataset/queues/generateVector.ts` 移植核心逻辑，去除 wallet/usage 相关调用：

```typescript
import { MongoDatasetTraining } from '@fastgpt/service/core/dataset/training/schema';
import { MongoDatasetData } from '@fastgpt/service/core/dataset/data/schema';
import { TrainingModeEnum } from '@fastgpt/global/core/dataset/constants';
import {
  deleteDatasetDataVector,
  insertDatasetDataVector,
} from '@fastgpt/service/common/vectorDB/controller';
import { getEmbeddingModel } from '@fastgpt/service/core/ai/model';
import { addMinutes } from 'date-fns';

const POLL_INTERVAL_MS = 2000;
const MAX_RETRIES = 5;

async function processOneItem(): Promise<boolean> {
  const item = await MongoDatasetTraining.findOneAndUpdate(
    {
      mode: TrainingModeEnum.chunk,
      retryCount: { $gt: 0 },
      lockTime: { $lte: addMinutes(new Date(), -3) },
    },
    { lockTime: new Date(), $inc: { retryCount: -1 } }
  )
    .populate([
      { path: 'dataset', select: 'vectorModel' },
      { path: 'collection', select: 'name indexPrefixTitle' },
      { path: 'data', select: '_id indexes' },
    ])
    .lean();

  if (!item) return false;

  try {
    const pop = item as any;
    const vectorModel = getEmbeddingModel(pop.dataset?.vectorModel);
    const text = item.q + (item.a ? `\n${item.a}` : '');

    const { insertIds } = await insertDatasetDataVector({
      teamId: item.teamId,
      datasetId: item.datasetId,
      collectionId: item.collectionId,
      model: vectorModel,
      inputs: [text],
    });

    await MongoDatasetData.updateOne(
      { _id: pop.data?._id },
      { $set: { dataId: insertIds[0] } }
    );

    await MongoDatasetTraining.deleteOne({ _id: item._id });
    return true;
  } catch (err) {
    console.error('[KB] Training item failed:', err);
    if (item.retryCount <= 0) {
      await MongoDatasetTraining.deleteOne({ _id: item._id });
    }
    return false;
  }
}

export const startTrainingWorker = (): void => {
  const run = async () => {
    try {
      const processed = await processOneItem();
      setTimeout(run, processed ? 100 : POLL_INTERVAL_MS);
    } catch (err) {
      console.error('[KB] Training worker error:', err);
      setTimeout(run, POLL_INTERVAL_MS);
    }
  };

  run();
  console.log('[KB] Training worker started');
};
```

- [ ] **Step 2: 在 start() 中启动 worker**

在 `src/index.ts` 的 `start()` 函数末尾添加：
```typescript
import { startTrainingWorker } from './services/trainingWorker';

// start() 内：
startTrainingWorker();
```

- [ ] **Step 3: Commit**

```bash
git add projects/knowledge-base/src/services/trainingWorker.ts projects/knowledge-base/src/index.ts
git commit -m "feat(knowledge-base): background training worker for vectorization"
```

---

## Task 9: 文件上传（普通文档）

**Files:**
- Create: `projects/knowledge-base/src/routes/files.ts`
- Modify: `projects/knowledge-base/src/index.ts`

- [ ] **Step 1: 创建 src/routes/files.ts**

```typescript
import { Hono } from 'hono';
import { MongoDataset } from '@fastgpt/service/core/dataset/schema';
import { MongoDatasetCollection } from '@fastgpt/service/core/dataset/collection/schema';
import { MongoDatasetTraining } from '@fastgpt/service/core/dataset/training/schema';
import { MongoDatasetData } from '@fastgpt/service/core/dataset/data/schema';
import {
  DatasetCollectionTypeEnum,
  TrainingModeEnum,
} from '@fastgpt/global/core/dataset/constants';
import { readFileContentByBuffer } from '@fastgpt/service/common/file/read/utils';
import { text2Chunks } from '@fastgpt/service/worker/function';

export const filesRouter = new Hono<{ Variables: { teamId: string } }>();

const SUPPORTED_EXTENSIONS = new Set([
  'txt', 'md', 'html', 'pdf', 'docx', 'pptx', 'xlsx', 'csv',
]);

filesRouter.post('/upload', async (c) => {
  const teamId = c.get('teamId');
  const formData = await c.req.formData();

  const file = formData.get('file') as File | null;
  const datasetId = formData.get('datasetId') as string;
  const collectionName = (formData.get('collectionName') as string) || file?.name || 'upload';

  if (!file || !datasetId) {
    return c.json({ error: 'Missing file or datasetId' }, 400);
  }

  const ext = file.name.split('.').pop()?.toLowerCase() || '';
  if (!SUPPORTED_EXTENSIONS.has(ext)) {
    return c.json({ error: `Unsupported file type: .${ext}` }, 400);
  }

  const dataset = await MongoDataset.findOne({ _id: datasetId, teamId }).lean();
  if (!dataset) return c.json({ error: 'Dataset not found' }, 404);

  const buffer = Buffer.from(await file.arrayBuffer());

  // 解析文件内容（使用 packages/service，包含 Textin/Doc2x 支持）
  const { rawText } = await readFileContentByBuffer({
    teamId,
    tmbId: teamId,
    extension: ext,
    buffer,
    encoding: 'utf-8',
    customPdfParse: true,
    getFormatText: true,
  });

  // 分块
  const chunks = await text2Chunks({
    text: rawText,
    chunkLen: 512,
    overlapRatio: 0.2,
    chunkSplitMode: 'markdown',
  });

  // 创建 Collection
  const collection = await MongoDatasetCollection.create({
    teamId,
    tmbId: teamId,
    datasetId,
    name: collectionName,
    type: DatasetCollectionTypeEnum.file,
    trainingType: TrainingModeEnum.chunk,
    fileId: file.name,
  });

  // 写入训练队列（后台 worker 消费）
  const trainingDocs = chunks.map((chunk) => ({
    teamId,
    tmbId: teamId,
    datasetId,
    collectionId: collection._id.toString(),
    q: chunk.q,
    a: chunk.a || '',
    mode: TrainingModeEnum.chunk,
    retryCount: 5,
    lockTime: new Date(0),
  }));

  await MongoDatasetTraining.insertMany(trainingDocs);

  return c.json({
    collectionId: collection._id,
    chunksCount: chunks.length,
  }, 201);
});
```

- [ ] **Step 2: 注册到 index.ts**

```typescript
import { filesRouter } from './routes/files';
v1.route('/files', filesRouter);
```

- [ ] **Step 3: Commit**

```bash
git add projects/knowledge-base/src/routes/files.ts projects/knowledge-base/src/index.ts
git commit -m "feat(knowledge-base): document file upload with parsing and training queue"
```

---

## Task 10: SSE 服务 + media_tasks 模型

**Files:**
- Create: `projects/knowledge-base/src/services/sse.ts`
- Create: `projects/knowledge-base/src/models/mediaTask.ts`
- Create: `projects/knowledge-base/test/mediaProcessor.test.ts`

- [ ] **Step 1: 写失败测试**

创建 `projects/knowledge-base/test/mediaProcessor.test.ts`：
```typescript
import { describe, it, expect } from 'vitest';

describe('media task model', () => {
  it('should export MongoMediaTask', async () => {
    const mod = await import('../src/models/mediaTask');
    expect(mod.MongoMediaTask).toBeDefined();
  });
});
```

```bash
cd projects/knowledge-base && bun test test/mediaProcessor.test.ts
```
Expected: FAIL

- [ ] **Step 2: 创建 src/models/mediaTask.ts**

```typescript
import mongoose, { Schema } from 'mongoose';

export type TaskStage =
  | 'uploading'
  | 'extracting_audio'
  | 'uploading_audio'
  | 'transcribing'
  | 'cleaning'
  | 'summarizing'
  | 'training'
  | 'completed'
  | 'failed';

export type MediaTaskDoc = {
  _id: string;  // UUID
  teamId: string;
  datasetId: string;
  fileName: string;
  fileType: 'audio' | 'video';
  contentType: 'meeting' | 'tutorial';
  status: 'pending' | 'processing' | 'completed' | 'failed';
  stage: TaskStage;
  progress: number;
  videoMinioKey: string;
  audioMinioKey: string;
  transcriptText: string;
  cleanedText: string;
  retryCount: number;
  error: string;
  createdAt: Date;
  updatedAt: Date;
};

const MediaTaskSchema = new Schema<MediaTaskDoc>(
  {
    _id: { type: String },
    teamId: { type: String, required: true, index: true },
    datasetId: { type: String, required: true },
    fileName: { type: String, required: true },
    fileType: { type: String, enum: ['audio', 'video'], required: true },
    contentType: { type: String, enum: ['meeting', 'tutorial'], default: 'tutorial' },
    status: {
      type: String,
      enum: ['pending', 'processing', 'completed', 'failed'],
      default: 'pending',
      index: true,
    },
    stage: { type: String, default: 'uploading' },
    progress: { type: Number, default: 0 },
    videoMinioKey: { type: String, default: '' },
    audioMinioKey: { type: String, default: '' },
    transcriptText: { type: String, default: '' },
    cleanedText: { type: String, default: '' },
    retryCount: { type: Number, default: 0 },
    error: { type: String, default: '' },
  },
  { timestamps: true }
);

export const MongoMediaTask = mongoose.model<MediaTaskDoc>('media_tasks', MediaTaskSchema);
```

- [ ] **Step 3: 创建 src/services/sse.ts**

```typescript
import type { Context } from 'hono';
import type { TaskStage } from '../models/mediaTask';

export type TaskEvent = {
  taskId: string;
  stage: TaskStage;
  progress: number;
  error?: string;
};

// taskId → 活跃的 SSE 响应控制器集合
const connections = new Map<string, Set<(event: TaskEvent) => void>>();

export const sseSubscribe = (taskId: string, send: (event: TaskEvent) => void): (() => void) => {
  if (!connections.has(taskId)) {
    connections.set(taskId, new Set());
  }
  connections.get(taskId)!.add(send);

  // 返回 unsubscribe 函数
  return () => {
    connections.get(taskId)?.delete(send);
    if (connections.get(taskId)?.size === 0) {
      connections.delete(taskId);
    }
  };
};

export const ssePublish = (event: TaskEvent): void => {
  const subs = connections.get(event.taskId);
  if (!subs) return;
  subs.forEach((send) => {
    try {
      send(event);
    } catch {
      // 连接已断开，忽略
    }
  });
};
```

- [ ] **Step 4: 运行测试**

```bash
cd projects/knowledge-base && bun test test/mediaProcessor.test.ts
```
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add projects/knowledge-base/src/models/mediaTask.ts projects/knowledge-base/src/services/sse.ts projects/knowledge-base/test/mediaProcessor.test.ts
git commit -m "feat(knowledge-base): media task model and SSE service"
```

---

## Task 11: ffmpeg 封装 + 豆包 ASR

**Files:**
- Create: `projects/knowledge-base/src/lib/ffmpeg.ts`
- Create: `projects/knowledge-base/src/lib/asr.ts`

- [ ] **Step 1: 创建 src/lib/ffmpeg.ts**

```typescript
import { spawn } from 'bun';
import { join } from 'path';
import { tmpdir } from 'os';

export const extractAudio = async (
  videoUrl: string,
  outputPath: string
): Promise<void> => {
  const proc = spawn([
    'ffmpeg', '-y',
    '-i', videoUrl,
    '-vn',
    '-acodec', 'pcm_s16le',
    '-ar', '16000',
    '-ac', '1',
    outputPath,
  ], {
    stdout: 'ignore',
    stderr: 'pipe',
  });

  const [exitCode, stderr] = await Promise.all([
    proc.exited,
    new Response(proc.stderr).text(),
  ]);

  if (exitCode !== 0) {
    throw new Error(`ffmpeg failed (code ${exitCode}): ${stderr.slice(0, 500)}`);
  }
};

export const getTmpAudioPath = (taskId: string): string =>
  join(tmpdir(), `kb-task-${taskId}.wav`);
```

- [ ] **Step 2: 创建 src/lib/asr.ts**（移植自 ic_superman/media.py）

```typescript
import type { AppConfig } from '../config';

const SUBMIT_URL = 'https://openspeech-direct.zijieapi.com/api/v3/auc/bigmodel/submit';
const QUERY_URL = 'https://openspeech-direct.zijieapi.com/api/v3/auc/bigmodel/query';
const CODE_SUCCESS = '20000000';
const CODE_PROCESSING = new Set(['20000001', '20000002']);

export const asrSubmit = async (
  audioUrl: string,
  config: AppConfig
): Promise<{ taskId: string; logId: string }> => {
  const taskId = crypto.randomUUID();
  const headers: Record<string, string> = {
    'X-Api-App-Key': config.VOLCENGINE_ASR_APP_ID!,
    'X-Api-Access-Key': config.VOLCENGINE_ASR_ACCESS_TOKEN!,
    'X-Api-Resource-Id': 'volc.bigasr.auc',
    'X-Api-Request-Id': taskId,
    'X-Api-Sequence': '-1',
  };

  const body = JSON.stringify({
    user: { uid: 'kb-service' },
    audio: { url: audioUrl },
    request: { model_name: 'bigmodel', enable_punc: true, enable_itn: true },
  });

  const res = await fetch(SUBMIT_URL, { method: 'POST', headers, body });
  const code = res.headers.get('X-Api-Status-Code') || '';

  if (code !== CODE_SUCCESS) {
    throw new Error(
      `ASR submit failed: code=${code} ${res.headers.get('X-Api-Message') || ''}`
    );
  }

  return { taskId, logId: res.headers.get('X-Tt-Logid') || '' };
};

export const asrPoll = async (
  taskId: string,
  logId: string,
  config: AppConfig
): Promise<string> => {
  const pollInterval = Number(config.VOLCENGINE_ASR_POLL_INTERVAL) * 1000;
  const pollTimeout = Number(config.VOLCENGINE_ASR_POLL_TIMEOUT) * 1000;
  const headers: Record<string, string> = {
    'X-Api-App-Key': config.VOLCENGINE_ASR_APP_ID!,
    'X-Api-Access-Key': config.VOLCENGINE_ASR_ACCESS_TOKEN!,
    'X-Api-Resource-Id': 'volc.bigasr.auc',
    'X-Api-Request-Id': taskId,
    'X-Tt-Logid': logId,
  };

  const deadline = Date.now() + pollTimeout;

  while (Date.now() < deadline) {
    const res = await fetch(QUERY_URL, {
      method: 'POST',
      headers,
      body: JSON.stringify({}),
    });
    const code = res.headers.get('X-Api-Status-Code') || '';

    if (code === CODE_SUCCESS) {
      const data = (await res.json()) as any;
      return data?.result?.text || '';
    }

    if (!CODE_PROCESSING.has(code)) {
      throw new Error(`ASR task ${taskId} failed: code=${code}`);
    }

    await Bun.sleep(pollInterval);
  }

  throw new Error(`ASR task ${taskId} timed out`);
};
```

- [ ] **Step 3: Commit**

```bash
git add projects/knowledge-base/src/lib/ffmpeg.ts projects/knowledge-base/src/lib/asr.ts
git commit -m "feat(knowledge-base): ffmpeg audio extraction and Doubao ASR integration"
```

---

## Task 12: 音视频处理状态机

**Files:**
- Create: `projects/knowledge-base/src/services/mediaProcessor.ts`

- [ ] **Step 1: 创建 src/services/mediaProcessor.ts**

```typescript
import { unlink, exists } from 'node:fs/promises';
import { MongoMediaTask, type MediaTaskDoc } from '../models/mediaTask';
import { ssePublish } from './sse';
import { putObject, getPresignedUrl, removeObject, objectExists } from '../db/minio';
import { extractAudio, getTmpAudioPath } from '../lib/ffmpeg';
import { asrSubmit, asrPoll } from '../lib/asr';
import type { AppConfig } from '../config';

const VIDEO_EXTENSIONS = new Set(['.mp4', '.mov', '.avi', '.mkv', '.webm']);
const MAX_RETRIES = 3;

const updateTask = async (
  task: MediaTaskDoc,
  update: Partial<Pick<MediaTaskDoc, 'stage' | 'progress' | 'status' | 'error' | 'transcriptText' | 'cleanedText' | 'audioMinioKey'>>
) => {
  Object.assign(task, update);
  await MongoMediaTask.updateOne({ _id: task._id }, { $set: update });
  ssePublish({ taskId: task._id, stage: task.stage, progress: task.progress, error: task.error || undefined });
};

const llmClean = async (text: string, config: AppConfig): Promise<string> => {
  const res = await fetch(`${config.LLM_BASE_URL}/chat/completions`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      Authorization: `Bearer ${config.LLM_API_KEY}`,
    },
    body: JSON.stringify({
      model: config.LLM_MODEL,
      messages: [
        {
          role: 'system',
          content:
            '你是一个文本整理助手。将下面的语音转写文本整理为规范的书面文档：去除口语化表达和语气词，修正标点，按语义分段落，保留所有信息内容，不做删减。直接输出整理后的文档，不要解释。',
        },
        { role: 'user', content: text },
      ],
    }),
  });
  const data = (await res.json()) as any;
  return data?.choices?.[0]?.message?.content || text;
};

const llmSummarize = async (text: string, config: AppConfig): Promise<string> => {
  const res = await fetch(`${config.LLM_BASE_URL}/chat/completions`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      Authorization: `Bearer ${config.LLM_API_KEY}`,
    },
    body: JSON.stringify({
      model: config.LLM_MODEL,
      messages: [
        {
          role: 'system',
          content:
            '请为以下文档生成一个300字以内的摘要，涵盖主要议题、关键结论和重要信息点。',
        },
        { role: 'user', content: text },
      ],
    }),
  });
  const data = (await res.json()) as any;
  return data?.choices?.[0]?.message?.content || '';
};

export const processMediaTask = async (taskId: string, config: AppConfig): Promise<void> => {
  const task = await MongoMediaTask.findById(taskId);
  if (!task) return;

  const bucket = config.KNOWLEDGE_MINIO_BUCKET;
  const isVideo = VIDEO_EXTENSIONS.has('.' + task.fileName.split('.').pop()?.toLowerCase());
  let tmpAudioPath: string | null = null;

  try {
    await updateTask(task, { status: 'processing', stage: 'extracting_audio', progress: 5 });

    // ── 阶段 1: 视频提取音频 ────────────────────────────────────────
    if (isVideo) {
      const audioKey = `audio/task-${taskId}.wav`;
      task.audioMinioKey = audioKey;

      const audioExists = await objectExists(bucket, audioKey);
      if (!audioExists) {
        tmpAudioPath = getTmpAudioPath(taskId);

        // 清理残留临时文件
        if (await exists(tmpAudioPath)) await unlink(tmpAudioPath);

        const videoUrl = await getPresignedUrl(bucket, task.videoMinioKey);
        await extractAudio(videoUrl, tmpAudioPath);
        await updateTask(task, { stage: 'uploading_audio', progress: 35 });

        // ── 阶段 2: 上传音频到 MinIO ──────────────────────────────────
        const { readFile } = await import('node:fs/promises');
        const audioBuffer = await readFile(tmpAudioPath);
        await putObject(bucket, audioKey, audioBuffer, 'audio/wav');
        await updateTask(task, { audioMinioKey: audioKey, stage: 'transcribing', progress: 50 });

        await unlink(tmpAudioPath);
        tmpAudioPath = null;
      } else {
        await updateTask(task, { audioMinioKey: audioKey, stage: 'transcribing', progress: 50 });
      }
    } else {
      task.audioMinioKey = task.videoMinioKey;
      await updateTask(task, { audioMinioKey: task.audioMinioKey, stage: 'transcribing', progress: 50 });
    }

    // ── 阶段 3: 豆包 ASR ─────────────────────────────────────────────
    if (!task.transcriptText) {
      const audioUrl = await getPresignedUrl(bucket, task.audioMinioKey, 7200);
      const { taskId: asrId, logId } = await asrSubmit(audioUrl, config);
      await updateTask(task, { progress: 60 });

      const transcript = await asrPoll(asrId, logId, config);
      await updateTask(task, { transcriptText: transcript, stage: 'cleaning', progress: 75 });
    } else {
      await updateTask(task, { stage: 'cleaning', progress: 75 });
    }

    // ── 阶段 4: LLM 清洗 ─────────────────────────────────────────────
    if (!task.cleanedText) {
      const cleaned = await llmClean(task.transcriptText, config);
      await updateTask(task, { cleanedText: cleaned, stage: 'summarizing', progress: 82 });
    } else {
      await updateTask(task, { stage: 'summarizing', progress: 82 });
    }

    // ── 阶段 5: 摘要（仅 meeting 类型） ──────────────────────────────
    await updateTask(task, { stage: 'training', progress: 85 });

    const { MongoDatasetCollection } = await import(
      '@fastgpt/service/core/dataset/collection/schema'
    );
    const { MongoDatasetTraining } = await import(
      '@fastgpt/service/core/dataset/training/schema'
    );
    const { text2Chunks } = await import('@fastgpt/service/worker/function');
    const { TrainingModeEnum, DatasetCollectionTypeEnum } = await import(
      '@fastgpt/global/core/dataset/constants'
    );

    const collection = await MongoDatasetCollection.create({
      teamId: task.teamId,
      tmbId: task.teamId,
      datasetId: task.datasetId,
      name: task.fileName,
      type: DatasetCollectionTypeEnum.file,
      trainingType: TrainingModeEnum.chunk,
    });

    const trainingItems = [];

    // 会议类型：先推入摘要 chunk
    if (task.contentType === 'meeting') {
      const summary = await llmSummarize(task.cleanedText, config);
      trainingItems.push({
        teamId: task.teamId,
        tmbId: task.teamId,
        datasetId: task.datasetId,
        collectionId: collection._id.toString(),
        q: `[摘要] ${task.fileName}\n${summary}`,
        a: '',
        mode: TrainingModeEnum.chunk,
        retryCount: 5,
        lockTime: new Date(0),
      });
    }

    // 正文分块
    const chunks = await text2Chunks({
      text: task.cleanedText,
      chunkLen: 512,
      overlapRatio: 0.2,
      chunkSplitMode: 'markdown',
    });

    for (const chunk of chunks) {
      trainingItems.push({
        teamId: task.teamId,
        tmbId: task.teamId,
        datasetId: task.datasetId,
        collectionId: collection._id.toString(),
        q: chunk.q,
        a: chunk.a || '',
        mode: TrainingModeEnum.chunk,
        retryCount: 5,
        lockTime: new Date(0),
      });
    }

    await MongoDatasetTraining.insertMany(trainingItems);
    await updateTask(task, { status: 'completed', stage: 'completed', progress: 100 });

    // 清理 MinIO 临时文件
    await Promise.all([
      removeObject(bucket, task.videoMinioKey),
      isVideo ? removeObject(bucket, task.audioMinioKey) : Promise.resolve(),
    ]);
  } catch (err: unknown) {
    const errorMsg = err instanceof Error ? err.message : String(err);
    const newRetry = task.retryCount + 1;

    if (newRetry >= MAX_RETRIES) {
      await updateTask(task, { status: 'failed', stage: 'failed', error: errorMsg, progress: task.progress });
    } else {
      await MongoMediaTask.updateOne({ _id: taskId }, { $inc: { retryCount: 1 }, $set: { status: 'pending' } });
    }
  } finally {
    if (tmpAudioPath) {
      try { await unlink(tmpAudioPath); } catch {}
    }
  }
};

// 服务启动时恢复中断的任务
export const recoverInterruptedTasks = async (config: AppConfig): Promise<void> => {
  const stuck = await MongoMediaTask.find({
    status: { $in: ['pending', 'processing'] },
    retryCount: { $lt: MAX_RETRIES },
  }).lean();

  if (stuck.length === 0) return;
  console.log(`[KB] Recovering ${stuck.length} interrupted media tasks`);

  for (const t of stuck) {
    processMediaTask(t._id, config).catch((err) =>
      console.error(`[KB] Recovery failed for task ${t._id}:`, err)
    );
  }
};
```

- [ ] **Step 2: Commit**

```bash
git add projects/knowledge-base/src/services/mediaProcessor.ts
git commit -m "feat(knowledge-base): media processing state machine with recovery"
```

---

## Task 13: 音视频上传路由 + 任务状态路由

**Files:**
- Create: `projects/knowledge-base/src/routes/media.ts`
- Create: `projects/knowledge-base/src/routes/tasks.ts`
- Modify: `projects/knowledge-base/src/index.ts`

- [ ] **Step 1: 创建 src/routes/media.ts**

```typescript
import { Hono } from 'hono';
import { v4 as uuidv4 } from 'uuid';
import { MongoMediaTask } from '../models/mediaTask';
import { putObject } from '../db/minio';
import { processMediaTask } from '../services/mediaProcessor';
import type { AppConfig } from '../config';

const VIDEO_EXTENSIONS = new Set(['mp4', 'mov', 'avi', 'mkv', 'webm']);
const AUDIO_EXTENSIONS = new Set(['mp3', 'wav', 'm4a', 'aac', 'ogg', 'flac']);

export const createMediaRouter = (config: AppConfig) => {
  const router = new Hono<{ Variables: { teamId: string } }>();

  router.post('/upload-media', async (c) => {
    const teamId = c.get('teamId');
    const formData = await c.req.formData();

    const file = formData.get('file') as File | null;
    const datasetId = formData.get('datasetId') as string;
    const contentType = (formData.get('contentType') as string) || 'tutorial';

    if (!file || !datasetId) {
      return c.json({ error: 'Missing file or datasetId' }, 400);
    }

    const ext = file.name.split('.').pop()?.toLowerCase() || '';
    const fileType = VIDEO_EXTENSIONS.has(ext)
      ? 'video'
      : AUDIO_EXTENSIONS.has(ext)
      ? 'audio'
      : null;

    if (!fileType) {
      return c.json({ error: `Unsupported media type: .${ext}` }, 400);
    }

    if (!config.VOLCENGINE_ASR_APP_ID) {
      return c.json({ error: 'ASR not configured' }, 503);
    }

    const taskId = uuidv4();
    const minioKey = `media/${taskId}.${ext}`;

    const buffer = Buffer.from(await file.arrayBuffer());
    await putObject(config.KNOWLEDGE_MINIO_BUCKET, minioKey, buffer);

    await MongoMediaTask.create({
      _id: taskId,
      teamId,
      datasetId,
      fileName: file.name,
      fileType,
      contentType: contentType === 'meeting' ? 'meeting' : 'tutorial',
      status: 'pending',
      stage: 'uploading',
      progress: 0,
      videoMinioKey: minioKey,
      audioMinioKey: '',
      retryCount: 0,
    });

    // 异步处理，不阻塞响应
    processMediaTask(taskId, config).catch((err) =>
      console.error(`[KB] Media task ${taskId} error:`, err)
    );

    return c.json({ taskId }, 202);
  });

  return router;
};
```

- [ ] **Step 2: 创建 src/routes/tasks.ts**

```typescript
import { Hono } from 'hono';
import { streamSSE } from 'hono/streaming';
import { MongoMediaTask } from '../models/mediaTask';
import { sseSubscribe } from '../services/sse';

export const tasksRouter = new Hono<{ Variables: { teamId: string } }>();

// 轮询查询
tasksRouter.get('/:taskId', async (c) => {
  const teamId = c.get('teamId');
  const task = await MongoMediaTask.findOne({
    _id: c.req.param('taskId'),
    teamId,
  }).lean();

  if (!task) return c.json({ error: 'Not found' }, 404);

  return c.json({
    taskId: task._id,
    status: task.status,
    stage: task.stage,
    progress: task.progress,
    error: task.error || undefined,
  });
});

// SSE 订阅
tasksRouter.get('/:taskId/stream', async (c) => {
  const teamId = c.get('teamId');
  const taskId = c.req.param('taskId');

  const task = await MongoMediaTask.findOne({ _id: taskId, teamId }).lean();
  if (!task) return c.json({ error: 'Not found' }, 404);

  return streamSSE(c, async (stream) => {
    // 立即推送当前状态
    await stream.writeSSE({
      data: JSON.stringify({
        taskId,
        stage: task.stage,
        progress: task.progress,
        error: task.error || undefined,
      }),
    });

    if (task.status === 'completed' || task.status === 'failed') {
      return;
    }

    // 订阅后续事件
    await new Promise<void>((resolve) => {
      const unsubscribe = sseSubscribe(taskId, async (event) => {
        await stream.writeSSE({ data: JSON.stringify(event) });
        if (event.stage === 'completed' || event.stage === 'failed') {
          unsubscribe();
          resolve();
        }
      });

      stream.onAbort(() => {
        unsubscribe();
        resolve();
      });
    });
  });
});
```

- [ ] **Step 3: 注册路由到 index.ts，并在 start() 中调用 recoverInterruptedTasks**

```typescript
import { createMediaRouter } from './routes/media';
import { tasksRouter } from './routes/tasks';
import { recoverInterruptedTasks } from './services/mediaProcessor';

// 路由注册（在 v1 下）
v1.route('/files', filesRouter);
v1.route('/files', createMediaRouter(config));  // 追加 upload-media 到 /v1/files
v1.route('/tasks', tasksRouter);

// start() 末尾添加
await recoverInterruptedTasks(config);
```

- [ ] **Step 4: Commit**

```bash
git add projects/knowledge-base/src/routes/media.ts projects/knowledge-base/src/routes/tasks.ts projects/knowledge-base/src/index.ts
git commit -m "feat(knowledge-base): media upload route and SSE task status endpoints"
```

---

## Task 14: Dockerfile

**Files:**
- Create: `projects/knowledge-base/Dockerfile`

- [ ] **Step 1: 创建 Dockerfile**

```dockerfile
FROM oven/bun:1.2-alpine AS base
WORKDIR /app

# 安装 ffmpeg
RUN apk add --no-cache ffmpeg

# 复制 monorepo 必要文件
COPY package.json pnpm-workspace.yaml pnpm-lock.yaml ./
COPY packages/global/package.json ./packages/global/
COPY packages/service/package.json ./packages/service/
COPY projects/knowledge-base/package.json ./projects/knowledge-base/

# 安装依赖
RUN bun install --frozen-lockfile

# 复制源码
COPY packages/global ./packages/global
COPY packages/service ./packages/service
COPY projects/knowledge-base ./projects/knowledge-base

WORKDIR /app/projects/knowledge-base

EXPOSE 3010

CMD ["bun", "run", "src/index.ts"]
```

- [ ] **Step 2: Commit**

```bash
git add projects/knowledge-base/Dockerfile
git commit -m "feat(knowledge-base): Dockerfile with ffmpeg"
```

---

## Task 15: 完整集成测试

**Files:**
- Create: `projects/knowledge-base/test/integration.test.ts`

- [ ] **Step 1: 创建集成测试（需要运行中的服务）**

创建 `projects/knowledge-base/test/integration.test.ts`：
```typescript
import { describe, it, expect, beforeAll } from 'vitest';

const BASE = process.env.KB_TEST_URL || 'http://localhost:3010';
const API_KEY = process.env.KB_TEST_API_KEY || 'test-key';
const headers = { Authorization: `Bearer ${API_KEY}`, 'Content-Type': 'application/json' };

describe('Integration: Health', () => {
  it('GET /health returns ok', async () => {
    const res = await fetch(`${BASE}/health`);
    const body = await res.json() as any;
    expect(res.status).toBe(200);
    expect(body.status).toBe('ok');
  });
});

describe('Integration: Auth', () => {
  it('rejects request without API key', async () => {
    const res = await fetch(`${BASE}/v1/datasets`);
    expect(res.status).toBe(401);
  });
});

describe('Integration: Dataset CRUD', () => {
  let datasetId: string;

  it('creates a dataset', async () => {
    const res = await fetch(`${BASE}/v1/datasets`, {
      method: 'POST',
      headers,
      body: JSON.stringify({ name: 'Test Dataset' }),
    });
    expect(res.status).toBe(201);
    const body = await res.json() as any;
    expect(body.id).toBeDefined();
    datasetId = body.id;
  });

  it('lists datasets', async () => {
    const res = await fetch(`${BASE}/v1/datasets`, { headers });
    expect(res.status).toBe(200);
    const body = await res.json() as any;
    expect(Array.isArray(body.items)).toBe(true);
  });

  it('deletes the dataset', async () => {
    const res = await fetch(`${BASE}/v1/datasets/${datasetId}`, {
      method: 'DELETE',
      headers,
    });
    expect(res.status).toBe(200);
  });
});
```

- [ ] **Step 2: 运行集成测试（需先启动服务）**

```bash
cd projects/knowledge-base && bun dev &
KB_TEST_URL=http://localhost:3010 KB_TEST_API_KEY=<valid-key> bun test test/integration.test.ts
```

Expected: 所有测试 PASS

- [ ] **Step 3: 最终 Commit**

```bash
git add projects/knowledge-base/test/integration.test.ts
git commit -m "test(knowledge-base): integration tests for health, auth, and dataset CRUD"
git push myfork main
```

---

## 自检结果

**Spec 覆盖检查：**
- ✅ 文件上传解析（Task 9）- txt/md/html/pdf/docx/pptx/xlsx/csv
- ✅ Textin/Doc2x 高级 PDF 解析（Task 2 global.systemEnv 初始化）
- ✅ 向量检索 API（Task 7）
- ✅ 知识库 CRUD（Task 5/6）
- ✅ 训练结果推送（Task 8 training worker）
- ✅ 健康检测（Task 5）
- ✅ 音视频处理 Pipeline（Task 10/11/12/13）
- ✅ 中断恢复（Task 12 recoverInterruptedTasks）
- ✅ SSE 状态推送（Task 10/13）
- ✅ 独立 API Key 认证（Task 4）
- ✅ 独立 MongoDB + MinIO（Task 3）
- ✅ Dockerfile（Task 14）

**类型一致性：**
- `TaskStage` 类型定义于 Task 10（`mediaTask.ts`），在 Task 10（`sse.ts`）、Task 12（`mediaProcessor.ts`）和 Task 13（`tasks.ts`）中统一引用
- `AppConfig` 定义于 Task 2，贯穿所有任务
- `teamId` 通过 Hono Context Variables 传递，类型声明在 Task 4
