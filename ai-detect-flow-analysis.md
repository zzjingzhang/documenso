# AI 字段/收件人检测请求流程分析

## 整体架构

```
/api/ai/detect-fields | /api/ai/detect-recipients
        │
        ▼
┌─────────────────────────┐
│  aiRateLimit 限流        │ [router.ts#L111]
└─────────────────────────┘
        │
        ▼
┌─────────────────────────┐
│  Session 会话验证        │ [detect-recipients.ts#L24]
└─────────────────────────┘
        │
        ▼
┌─────────────────────────┐
│  Team 访问权限验证       │ [detect-recipients.ts#L33-L42]
└─────────────────────────┘
        │
        ▼
┌─────────────────────────┐
│  aiFeaturesEnabled       │ [detect-recipients.ts#L45-L51]
└─────────────────────────┘
        │
        ▼
┌─────────────────────────┐
│  IS_AI_FEATURES_CONFIGURED │ [detect-recipients.ts#L53-L57]
└─────────────────────────┘
        │
        ▼
┌─────────────────────────┐
│  NDJSON 流式响应         │
│  ├─ keepalive 心跳       │
│  ├─ progress 进度        │
│  ├─ complete 完成        │
│  └─ error 错误           │
└─────────────────────────┘
```

---

## 1. 请求前置检查流程

### 1.1 aiRateLimit 限流

**位置**: [router.ts#L111](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/apps/remix/server/router.ts#L111-L112)

```typescript
// AI route.
app.use('/api/ai/*', aiRateLimitMiddleware);
app.route('/api/ai', aiRoute);
```

- 所有 `/api/ai/*` 路由统一应用 `aiRateLimitMiddleware`
- 限流中间件通过 `createRateLimitMiddleware(aiRateLimit)` 创建 [router.ts#L55]
- 在路由匹配前执行，防止未授权用户消耗 AI 资源

### 1.2 Session 会话验证

**位置**: [detect-recipients.ts#L24-L30](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/apps/remix/server/api/ai/detect-recipients.ts#L24-L30)

```typescript
const session = await getSession(c);

if (!session.user) {
  throw new AppError(AppErrorCode.UNAUTHORIZED, {
    message: 'You must be logged in to detect recipients',
  });
}
```

- 调用 `getSession(c)` 从请求中提取会话
- 验证 `session.user` 存在，确保用户已登录
- 失败返回 `UNAUTHORIZED` 错误

### 1.3 Team 访问权限验证

**位置**: [detect-recipients.ts#L33-L42](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/apps/remix/server/api/ai/detect-recipients.ts#L33-L42)

```typescript
const team = await getTeamById({
  userId: session.user.id,
  teamId,
}).catch(() => null);

if (!team) {
  throw new AppError(AppErrorCode.UNAUTHORIZED, {
    message: 'You do not have access to this team',
  });
}
```

- 通过 `getTeamById` 验证用户对该团队的访问权限
- 传入 `userId` 和 `teamId` 进行关联验证
- 团队不存在或用户无权限时返回 `UNAUTHORIZED`

### 1.4 aiFeaturesEnabled 团队 AI 开关

**位置**: [detect-recipients.ts#L45-L51](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/apps/remix/server/api/ai/detect-recipients.ts#L45-L51)

```typescript
const { aiFeaturesEnabled } = team.derivedSettings;

if (!aiFeaturesEnabled) {
  throw new AppError(AppErrorCode.UNAUTHORIZED, {
    message: 'AI features are not enabled for this team',
  });
}
```

- 从团队 `derivedSettings` 中读取 `aiFeaturesEnabled` 配置
- 团队级别的 AI 功能开关，可能受许可证控制
- 未启用时返回 `UNAUTHORIZED`

### 1.5 IS_AI_FEATURES_CONFIGURED 全局配置检查

**位置**: [detect-recipients.ts#L53-L57](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/apps/remix/server/api/ai/detect-recipients.ts#L53-L57)

```typescript
if (!IS_AI_FEATURES_CONFIGURED()) {
  throw new AppError(AppErrorCode.INVALID_REQUEST, {
    message: 'AI features are not configured. Please contact support to enable AI features.',
  });
}
```

- 全局级别的 AI 配置检查，验证 AI 服务商（如 Google Vertex）是否正确配置
- 检查 API Key、模型配置等必要参数
- 未配置时返回 `INVALID_REQUEST`，提示联系支持

---

## 2. NDJSON 流式响应

两个接口都使用 `hono/streaming` 的 `streamText` 实现 NDJSON（Newline Delimited JSON）流式响应。

**位置**: [detect-recipients.ts#L67-L134](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/apps/remix/server/api/ai/detect-recipients.ts#L67-L134)

### 2.1 keepalive 心跳事件

```typescript
const KEEPALIVE_INTERVAL_MS = 5000;

let interval: NodeJS.Timeout | null = setInterval(() => {
  void stream.writeln(JSON.stringify({ type: 'keepalive' }));
}, KEEPALIVE_INTERVAL_MS);
```

**事件结构**:
```json
{ "type": "keepalive" }
```

- 每 5000ms 发送一次，防止连接超时
- 处理完成或出错时通过 `clearInterval(interval)` 清除
- 确保长时间运行的 AI 任务不会因连接超时而中断

### 2.2 progress 进度事件

**detect-recipients**:
```typescript
onProgress: (progress) => {
  void stream.writeln(
    JSON.stringify({
      type: 'progress',
      pagesProcessed: progress.pagesProcessed,
      totalPages: progress.totalPages,
      recipientsDetected: progress.recipientsDetected,
    }),
  );
}
```

**detect-fields**:
```typescript
onProgress: (progress) => {
  void stream.writeln(
    JSON.stringify({
      type: 'progress',
      pagesProcessed: progress.pagesProcessed,
      totalPages: progress.totalPages,
      fieldsDetected: progress.fieldsDetected,
    }),
  );
}
```

**事件结构**:
```json
{
  "type": "progress",
  "pagesProcessed": 5,
  "totalPages": 10,
  "recipientsDetected": 3  // 或 fieldsDetected
}
```

- 每处理完一页（或一批）后触发
- 前端可实时展示处理进度

### 2.3 complete 完成事件

**detect-recipients**:
```typescript
await stream.writeln(
  JSON.stringify({
    type: 'complete',
    recipients,
  }),
);
```

**detect-fields**:
```typescript
await stream.writeln(
  JSON.stringify({
    type: 'complete',
    fields: allFields,
  }),
);
```

**事件结构**:
```json
{
  "type": "complete",
  "recipients": [...]  // 或 "fields": [...]
}
```

- AI 检测完成后发送最终结果
- 发送前先清除 keepalive 定时器

### 2.4 error 错误事件

```typescript
const message = error instanceof AppError ? error.message : 'Failed to detect recipients';

await stream.writeln(
  JSON.stringify({
    type: 'error',
    message,
  }),
);
```

**事件结构**:
```json
{
  "type": "error",
  "message": "错误信息"
}
```

- 流式处理过程中发生错误时发送
- 优先使用 `AppError.message`，否则使用通用错误信息
- 发送前清除 keepalive 定时器

---

## 3. pdfToImages 实现

**位置**: [pdf-to-images.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/ai/pdf-to-images.ts)

### 3.1 skia-canvas 集成

```typescript
import { Canvas, Image, Path2D } from 'skia-canvas';

// @ts-expect-error napi-rs/canvas satisfies the requirements
globalThis.Path2D = Path2D;
// @ts-expect-error napi-rs/canvas satisfies the requirements
globalThis.Image = Image;

class SkiaCanvasFactory {
  _createCanvas(width: number, height: number) {
    const canvas = new Canvas(width, height);
    canvas.gpu = false;  // 禁用 GPU 加速，避免环境依赖问题
    return canvas;
  }
  // ... create, reset, destroy 方法
}
```

- 使用 `skia-canvas` 替代默认 canvas，提供更好的服务端性能
- 全局注入 `Path2D` 和 `Image` 以满足 pdf.js 的依赖
- 显式禁用 GPU 加速（`canvas.gpu = false`），确保在无 GPU 环境下稳定运行

### 3.2 pdfjs 渲染

```typescript
import * as pdfjsLib from 'pdfjs-dist/legacy/build/pdf.mjs';

const task = await pdfjsLib.getDocument({
  data: pdfBytes,
  CanvasFactory: SkiaCanvasFactory,  // 使用自定义 Skia Canvas 工厂
});

const pdf = await task.promise;
```

- 使用 `pdfjs-dist/legacy/build/pdf.mjs` 兼容服务端环境
- 传入自定义 `SkiaCanvasFactory` 替换默认的 canvas 实现
- 支持 scale 参数控制渲染清晰度（默认 scale=2）

### 3.3 并发控制

```typescript
const images = await pMap(
  Array.from({ length: pdf.numPages }),
  async (_, index) => {
    const pageNumber = index + 1;
    const page = await pdf.getPage(pageNumber);
    // ... 单页渲染逻辑
  },
  { concurrency: 10 },  // 并发渲染 10 页
);
```

- 使用 `pMap` 控制并发数为 10
- 平衡渲染速度和内存占用
- 每页渲染完成后调用 `page.cleanup()` 释放资源
- 全部完成后调用 `pdf.destroy()` 和 `task.destroy()` 清理

---

## 4. detectRecipients 核心逻辑

**位置**: [detect-recipients/index.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/ai/envelope/detect-recipients/index.ts)

### 4.1 MAX_PAGES_PER_CHUNK 分块处理

```typescript
const MAX_PAGES_PER_CHUNK = 10;

const imageChunks = chunk(images, MAX_PAGES_PER_CHUNK);
const totalChunks = imageChunks.length;
```

- 每 10 页作为一个 chunk 发送给 AI 模型
- 避免单次请求超出模型上下文窗口限制
- 对于长文档，分批次逐步处理

### 4.2 messages 累积机制

```typescript
const messages: ModelMessage[] = [];
let allRecipients: TDetectedRecipientSchema[] = [];

for (const [chunkIndex, currentChunk] of imageChunks.entries()) {
  const promptText = buildPromptText({
    chunkIndex,
    totalChunks,
    totalPages,
    startPage,
    endPage,
    detectedRecipients: allRecipients,  // 传入已检测到的收件人
  });

  // 添加用户消息（包含图片）
  messages.push({
    role: 'user',
    content: [
      { type: 'text', text: promptText },
      ...createImageContentParts(currentChunk),
    ],
  });

  const result = await generateObject({
    model: vertex('gemini-3-flash-preview'),
    system: SYSTEM_PROMPT,
    schema: ZDetectedRecipientsSchema,
    messages,  // 累积的完整对话历史
    temperature: 0.5,
  });

  // 添加助手响应作为下一轮上下文
  messages.push({
    role: 'assistant',
    content: `Detected recipients: ${JSON.stringify(allRecipients)}`,
  });
}
```

- `messages` 数组在整个循环中持续累积
- 每轮请求包含：所有历史用户消息 + 所有历史助手响应
- 让 AI 了解之前已检测到的收件人，避免重复检测
- 非首块的 prompt 会明确要求只检测 NEW recipients

### 4.3 去重策略

```typescript
const isDuplicateRecipient = (recipient: TDetectedRecipientSchema, existing: TDetectedRecipientSchema) => {
  if (recipient.email && existing.email) {
    return recipient.email.toLowerCase() === existing.email.toLowerCase();
  }

  if (recipient.name && existing.name) {
    return recipient.name.toLowerCase() === existing.name.toLowerCase();
  }

  return false;
};

const mergeRecipients = (existingRecipients: TDetectedRecipientSchema[], newRecipients: TDetectedRecipientSchema[]) => {
  const merged = [...existingRecipients];

  for (const recipient of newRecipients) {
    const isDuplicate = merged.some((existing) => isDuplicateRecipient(recipient, existing));

    if (!isDuplicate) {
      merged.push(recipient);
    }
  }

  return merged;
};
```

- 优先级：先按 email 去重（大小写不敏感），再按 name 去重
- 多文档时，跨文档合并也使用相同的去重逻辑
- 即使 AI 在 prompt 中被要求只返回新收件人，代码层面仍做去重保障

---

## 5. detectFields 核心逻辑

**位置**: [detect-fields/index.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/ai/envelope/detect-fields/index.ts)

### 5.1 mask 已有字段

```typescript
const maskFieldsOnImage = async ({ image, width, height, fields }: MaskFieldsOnImageOptions) => {
  if (fields.length === 0) {
    return image;
  }

  const img = await loadImage(image);
  const canvas = createCanvas(width, height);
  const ctx = canvas.getContext('2d');

  ctx.drawImage(img, 0, 0, width, height);

  ctx.fillStyle = '#000000';

  for (const field of fields) {
    // field positions and width,height are on a 0-100 percentage scale
    const x = (field.positionX.toNumber() / 100) * width;
    const y = (field.positionY.toNumber() / 100) * height;
    const w = (field.width.toNumber() / 100) * width;
    const h = (field.height.toNumber() / 100) * height;

    ctx.fillRect(x, y, w, h);
  }

  return canvas.encode('jpeg');
};
```

- 先渲染原始图片，然后用黑色矩形覆盖已有字段
- 防止 AI 重复检测已存在的字段
- prompt 中也会明确提示 "IGNORE any solid black rectangles"
- 使用 `@napi-rs/canvas` 进行图像处理

### 5.2 buildRecipientContextMessage 构建收件人上下文

**位置**: [helpers.ts#L7-L19](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/ai/envelope/detect-fields/helpers.ts#L7-L19)

```typescript
export const buildRecipientContextMessage = (recipients: RecipientContext[]) => {
  if (recipients.length === 0) {
    return 'No recipients have been specified for this document. Leave recipientKey empty for all fields.';
  }

  const recipientList = recipients.map((r) => `- ${formatRecipientKey(r)}`).join('\n');

  return `The following recipients will sign/fill this document. Use their recipientKey when assigning fields:

${recipientList}

When you detect a field that should be filled by a specific recipient (based on nearby labels like "Tenant Signature", "Landlord", "Buyer", etc.), set the recipientKey to match one of the above. If no recipient can be determined, leave recipientKey empty.`;
};
```

- 为 AI 提供收件人列表，格式为 `id|name|email`
- 指导 AI 根据附近标签（如 "Tenant Signature"）将字段分配给正确的收件人
- 无收件人时明确要求留空 `recipientKey`

### 5.3 box2d 归一化

**位置**: [helpers.ts#L77-L91](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/ai/envelope/detect-fields/helpers.ts#L77-L91)

```typescript
export const normalizeDetectedField = (field: DetectedField): NormalizedField => {
  const { box2d } = field;

  const [yMin, xMin, yMax, xMax] = box2d;

  return {
    type: field.type,
    recipientKey: field.recipientKey,
    positionX: xMin / 10,
    positionY: yMin / 10,
    width: (xMax - xMin) / 10,
    height: (yMax - yMin) / 10,
    confidence: field.confidence,
  };
};
```

**坐标转换**:
- AI 返回的 box2d 格式: `[yMin, xMin, yMax, xMax]`，范围 0-1000
- 系统使用的格式: 百分比 0-100
- 转换方式: 直接除以 10
- 注意 box2d 的顺序是 **y 坐标在前**，需要正确映射到 positionX/positionY

### 5.4 resolveRecipientFromKey 失败处理

**位置**: [helpers.ts#L37-L72](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/ai/envelope/detect-fields/helpers.ts#L37-L72)

```typescript
export const resolveRecipientFromKey = (recipientKey: string, recipients: RecipientContext[]) => {
  if (recipients.length === 0) {
    return null;  // 无收件人时返回 null，触发创建逻辑
  }

  if (!recipientKey) {
    return recipients[0];  // 空 key 默认使用第一个收件人
  }

  const [idStr, name, email] = recipientKey.split('|');
  const id = Number(idStr);

  // 1. 按 ID 匹配
  if (!Number.isNaN(id)) {
    const matchById = recipients.find((r) => r.id === id);
    if (matchById) return matchById;
  }

  // 2. 按 email + name 匹配
  if (email && name) {
    const matchByEmailAndName = recipients.find((r) => r.email === email && r.name === name);
    if (matchByEmailAndName) return matchByEmailAndName;
  }

  // 3. 都不匹配时默认第一个
  return recipients[0];
};
```

**创建空 SIGNER 收件人**:

**位置**: [detect-fields/index.ts#L83-L101](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/ai/envelope/detect-fields/index.ts#L83-L101)

```typescript
let resolvedRecipient = resolveRecipientFromKey(recipientKey, recipients);

// If no recipients exist, create a blank recipient
if (!resolvedRecipient) {
  const { recipients: createdRecipients } = await createEnvelopeRecipients({
    id: {
      id: envelope.id,
      type: 'envelopeId',
    },
    recipients: [
      {
        name: '',
        email: '',
        role: RecipientRole.SIGNER,
      },
    ],
    userId,
    teamId,
  });

  resolvedRecipient = createdRecipients[0];
}
```

- 当 `resolveRecipientFromKey` 返回 `null`（即 `recipients.length === 0`）时
- 自动创建一个空的 SIGNER 收件人（name='', email=''）
- 确保所有检测到的字段都有对应的 recipientId
- 用户后续可以编辑这个自动创建的收件人

---

## 6. 状态限制设计原因

### 6.1 detectFields 只允许 DRAFT

**位置**: [detect-fields/index.ts#L44-L48](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/ai/envelope/detect-fields/index.ts#L44-L48)

```typescript
if (envelope.status !== DocumentStatus.DRAFT) {
  throw new AppError(AppErrorCode.INVALID_REQUEST, {
    message: 'Cannot detect fields for a non-draft envelope',
  });
}
```

**设计原因**:
1. **字段修改权限**: 只有 DRAFT 状态的信封允许添加/修改字段
2. **数据一致性**: 非草稿状态（如 PENDING, COMPLETED）的信封可能已有签署活动，添加字段会破坏签署流程
3. **业务逻辑**: 字段检测通常发生在编辑信封阶段，此时用户正在准备发送文档
4. **自动创建收件人**: detectFields 可能自动创建收件人，这在非草稿状态下是不允许的

### 6.2 detectRecipients 只拒绝 COMPLETED

**位置**: [detect-recipients/index.ts#L53-L57](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/ai/envelope/detect-recipients/index.ts#L53-L57)

```typescript
if (envelope.status === DocumentStatus.COMPLETED) {
  throw new AppError(AppErrorCode.INVALID_REQUEST, {
    message: 'Cannot detect recipients for a completed envelope',
  });
}
```

**设计原因**:
1. **已完成文档不可变**: COMPLETED 状态的信封所有签署已完成，收件人列表不可更改
2. **允许在 PENDING 状态使用**: 用户可能在发送后发现遗漏了收件人，此时检测收件人可帮助补充
3. **只读操作**: detectRecipients 是纯分析操作，不修改信封状态，因此限制更宽松
4. **对比 detectFields**: detectFields 会修改数据（创建收件人、添加字段），而 detectRecipients 只返回检测结果

---

## 关键差异总结

| 特性 | detectRecipients | detectFields |
|------|------------------|--------------|
| 状态限制 | 只拒绝 COMPLETED | 只允许 DRAFT |
| 处理方式 | 按 chunk 累积 messages，串行 | 按页面并发处理（concurrency: 5） |
| 修改数据 | 否（只读） | 是（可能创建收件人） |
| 去重逻辑 | 跨 chunk、跨文档去重 | 无需去重（mask 已有字段） |
| 坐标处理 | 无 | 0-1000 → 0-100 归一化 |
| 收件人解析 | 无 | resolveRecipientFromKey，失败则创建空 SIGNER |
