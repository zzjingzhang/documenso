# PDF 从上传到下载的完整生命周期分析

## 1. 上传阶段：`/upload-pdf`

### 1.1 请求入口

路由定义在 [files.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/apps/remix/server/api/files/files.ts#L32-L56)，使用 Hono 框架注册 `POST /upload-pdf`，通过 `sValidator('form', ZUploadPdfRequestSchema)` 校验表单，要求上传字段名为 `file`，类型为 `File` 实例。

### 1.2 文件大小限制

```typescript
const MAX_FILE_SIZE = APP_DOCUMENT_UPLOAD_SIZE_LIMIT * 1024 * 1024;
if (file.size > MAX_FILE_SIZE) {
  return c.json({ error: 'File too large' }, 400);
}
```

- [APP_DOCUMENT_UPLOAD_SIZE_LIMIT](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/constants/app.ts#L3) 读取环境变量 `NEXT_PUBLIC_DOCUMENT_SIZE_UPLOAD_LIMIT`，默认值为 **50**（即 50 MB）
- 限制公式：`MAX_FILE_SIZE = 50 * 1024 * 1024 = 52,428,800 bytes`
- 超出限制直接返回 `400 { error: 'File too large' }`

### 1.3 `putNormalizedPdfFileServerSide` 流程

定义在 [put-file.server.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/universal/upload/put-file.server.ts#L54-L71)，具体步骤：

1. **读取 Buffer**：`Buffer.from(await file.arrayBuffer())`
2. **PDF 标准化**：调用 [normalizePdf](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/pdf/normalize-pdf.ts#L5-L34)
   - 加载 PDF 并校验有效性（无效则抛 `INVALID_DOCUMENT_FILE`）
   - 检查加密：`pdfDoc.isEncrypted` 为 true 则拒绝
   - 扁平化图层：`pdfDoc.flattenLayers()`
   - 扁平化表单：`form.flatten()` + `pdfDoc.flattenAnnotations()`（默认 `flattenForm: true`）
   - 保存为新的 Buffer 返回
3. **存储文件**：调用 `putFileServerSide` 将标准化后的 Buffer 存入对应存储
4. **创建数据库记录**：调用 [createDocumentData](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/document-data/create-document-data.ts#L16-L23)

> 注意：与 `putPdfFileServerSide`（旧路径）不同，`putNormalizedPdfFileServerSide` **不传入 `initialData`**，因此 `createDocumentData` 中 `initialData` 为 `undefined`，最终 `initialData: initialData || data`，即 **initialData 与 data 相同**，都指向标准化后的原始 PDF。

### 1.4 `NEXT_PUBLIC_UPLOAD_TRANSPORT` 与 DocumentData 变化

`putFileServerSide` 根据 `NEXT_PUBLIC_UPLOAD_TRANSPORT` 环境变量分发存储策略：

| Transport 值 | 存储函数 | `DocumentData.type` | `DocumentData.data` | `DocumentData.initialData` |
|---|---|---|---|---|
| `database`（默认/fallback） | [putFileInDatabase](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/universal/upload/put-file.server.ts#L85-L96) | `BYTES_64` | Base64 编码的 PDF 全文 | 同 `data`（上传时未单独指定） |
| `s3` | [putFileInObjectStorage](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/universal/upload/put-file.server.ts#L98-L112) | `S3_PATH` | S3 对象 key（如 `uploads/xxx.pdf`） | 同 `data` |
| `azure-blob` | `putFileInObjectStorage`（与 s3 共用） | `S3_PATH` | Blob 对象 key | 同 `data` |

具体过程：
- **database 模式**：将 `ArrayBuffer` 转为 `Uint8Array`，再用 `base64.encode` 编码为 ASCII 字符串存储，type 为 `BYTES_64`
- **s3/azure-blob 模式**：将 Buffer 包装为 `File` 对象，调用 `uploadS3File` 上传到对象存储，返回的 key 作为 data，type 为 `S3_PATH`

> 注意：`BYTES` 类型（原始二进制字符串）在当前上传流程中 **不再产生**，它属于旧版遗留格式，仅在读取时兼容。

---

## 2. 读取阶段：`getFileServerSide`

定义在 [get-file.server.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/universal/upload/get-file.server.ts#L12-L18)，使用 `ts-pattern` 的 `match` 做穷尽匹配：

| DocumentDataType | 读取函数 | 处理逻辑 |
|---|---|---|
| `BYTES` | [getFileFromBytes](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/universal/upload/get-file.server.ts#L20-L26) | `TextEncoder.encode(data)` — 将二进制字符串编码为 `Uint8Array` |
| `BYTES_64` | [getFileFromBytes64](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/universal/upload/get-file.server.ts#L28-L32) | `base64.decode(data)` — 将 Base64 字符串解码为 `Uint8Array` |
| `S3_PATH` | [getFileFromS3](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/universal/upload/get-file.server.ts#L34-L49) | 调用 `getPresignGetUrl(key)` 获取预签名 URL → `fetch(url)` → `response.arrayBuffer()` → `Uint8Array` |

返回值统一为 `Uint8Array`，调用方无需关心底层存储差异。

---

## 3. 访问控制：四种访问方式的查询条件对比

### 3.1 Session 访问（已登录用户）

路由：`GET /envelope/:envelopeId/envelopeItem/:envelopeItemId`

```typescript
const session = await getOptionalSession(c);
let userId = session.user?.id;
```

**查询条件**：
```typescript
prisma.envelope.findFirst({
  where: { id: envelopeId },
  include: {
    envelopeItems: { where: { id: envelopeItemId }, include: { documentData: true } }
  }
})
```

**鉴权**：通过 [checkEnvelopeFileAccess](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/apps/remix/server/api/files/files.helpers.ts#L251-L291) 校验：
1. 用户是否为 `envelope.teamId` 对应 Team 的成员（`getTeamById`）
2. 若非成员且信封类型为 `TEMPLATE` + `ORGANISATION`，则降级检查用户是否属于同一 Organisation 的任何 Team

### 3.2 Embedding Presign Token 访问

路由：`GET /envelope/:envelopeId/envelopeItem/:envelopeItemId?token=xxx`

```typescript
if (token) {
  const presignToken = await verifyEmbeddingPresignToken({ token }).catch(() => undefined);
  userId = presignToken?.userId;
}
```

**Token 验证**（[verifyEmbeddingPresignToken](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/embedding-presign/verify-embedding-presign-token.ts#L12-L133)）：
1. JWT 解码获取 `sub`（API Token ID）和 `aud`（User ID / Team ID）
2. 从数据库查找对应 `ApiToken` 记录
3. 校验 `aud` 与 `apiToken.teamId` 或 `apiToken.userId` 匹配
4. 使用 `apiToken.token` 作为密钥通过 `jwtVerify` 验证签名和过期时间
5. 返回 `userId`，后续查询条件与 Session 访问相同

**查询条件**：与 Session 相同（通过 `userId` + `checkEnvelopeFileAccess`），但身份来源不同。

### 3.3 Recipient Token 访问

路由：`GET /token/:token/envelopeItem/:envelopeItemId`

**查询条件**：
```typescript
prisma.envelopeItem.findUnique({
  where: {
    id: envelopeItemId,
    envelope: {
      recipients: { some: { token } }
    }
  },
  include: { envelope: true, documentData: true }
})
```

**特点**：
- 通过 `recipient.token` 关联查找，无需 Session
- 不调用 `checkEnvelopeFileAccess`，只要 token 匹配即可访问
- 默认 version 为 `signed`

### 3.4 QR Token 访问

路由：`GET /token/:token/envelopeItem/:envelopeItemId`（与 Recipient Token 共用路由）

**查询条件**（当 `token.startsWith('qr_')` 时）：
```typescript
prisma.envelopeItem.findUnique({
  where: {
    id: envelopeItemId,
    envelope: { qrToken: token }
  },
  include: { envelope: true, documentData: true }
})
```

**特点**：
- 通过 `envelope.qrToken` 查找，对应完成签署后证书上的二维码
- 同样无需 Session，不调用 `checkEnvelopeFileAccess`
- 默认 version 为 `signed`

### 3.5 对比总结

| 访问方式 | 身份来源 | 查询入口 | Prisma 查询方式 | 是否需要 Team 鉴权 | 默认 version |
|---|---|---|---|---|---|
| Session | `getOptionalSession` | `envelope.findFirst` by ID | 先找 Envelope 再取 Item | ✅ `checkEnvelopeFileAccess` | `signed` |
| Embedding Presign | JWT → ApiToken → userId | 同 Session | 同 Session | ✅ `checkEnvelopeFileAccess` | `signed` |
| Recipient Token | URL 中的 `token` | `envelopeItem.findUnique` by recipient token | 直接查找 EnvelopeItem | ❌ | `signed` |
| QR Token | URL 中 `qr_` 前缀的 token | `envelopeItem.findUnique` by qrToken | 直接查找 EnvelopeItem | ❌ | `signed` |

### 3.6 硬缓存路由（item.pdf）

[get-envelope-item-pdf](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/apps/remix/server/api/files/routes/get-envelope-item-pdf.ts) 和 [get-envelope-item-pdf-by-token](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/apps/remix/server/api/files/routes/get-envelope-item-pdf-by-token.ts) 提供了基于 `documentDataId` 的硬缓存路由：

- Session/Presign Token 路由：`cacheStrategy: 'private'`
- Recipient/QR Token 路由：`cacheStrategy: 'private'`（当前代码中 token 路由也使用 `private`）
- 查询额外包含 `documentDataId` 条件，使 URL 与特定数据版本绑定，配合 `immutable` 缓存策略实现长期缓存

---

## 4. 版本选择：`signed` / `original` / `pending`

### 4.1 版本定义

在 [files.types.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/apps/remix/server/api/files/files.types.ts#L52) 中：

- **Session download** 支持 `signed`、`original`、`pending`（默认 `signed`）
- **Token download** 仅支持 `signed`、`original`（默认 `signed`），**不支持 `pending`**

### 4.2 data 与 initialData 的选择逻辑

定义在 [handleStaticFileRequest](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/apps/remix/server/api/files/files.helpers.ts#L79-L134) 和 [handlePendingFileRequest](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/apps/remix/server/api/files/files.helpers.ts#L138-L235)：

| Version | 使用的 DocumentData 字段 | 含义 |
|---|---|---|
| `signed` | `documentData.data` | 签署/封存后的 PDF（含签名、证书页等） |
| `original` | `documentData.initialData` | 原始上传的标准化 PDF（无任何签署内容） |
| `pending` | `documentData.initialData` | 以原始 PDF 为底板，动态叠加已插入字段 |

> **关键区别**：`pending` 版本虽然读取的是 `initialData`，但它不是直接返回，而是作为底板传给 `generatePartialSignedPdf` 进行实时渲染。

### 4.3 硬缓存路由的版本映射

[item.pdf 路由](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/apps/remix/server/api/files/routes/get-envelope-item-pdf.ts#L126-L127) 使用不同的版本名：

| URL version | 使用的字段 |
|---|---|
| `current` | `documentData.data`（等同于 `signed`） |
| `initial` | `documentData.initialData`（等同于 `original`） |

---

## 5. ETag 机制

### 5.1 静态文件（signed/original）

```typescript
const documentDataToUse = version === 'signed' ? documentData.data : documentData.initialData;
const etag = Buffer.from(sha256(documentDataToUse)).toString('hex');
```

- 对所使用的 data 字符串做 SHA-256 哈希，转为十六进制作为 ETag
- 由于 `data`/`initialData` 在封存后不再变化，ETag 天然稳定

### 5.2 Pending 文件

```typescript
const etag = Buffer.from(
  sha256(JSON.stringify({
    envelopeStatus: envelope.status,
    fields: fields.map((field) => ({
      id: field.id,
      customText: field.customText,
      signatureId: field.signature?.id ?? null,
      signatureCreated: field.signature?.created ?? null,
    })),
  })),
).toString('hex');
```

- ETag 基于 **信封状态 + 所有已插入字段的摘要** 计算
- 任何字段被签署或修改都会导致 ETag 变化，客户端需重新获取

### 5.3 硬缓存路由

```typescript
const etag = Buffer.from(sha256(documentDataToUse)).toString('hex');
```

- 与静态文件相同，基于 data 字符串哈希

---

## 6. Cache-Control 策略

### 6.1 静态文件 — 查看（isDownload = false）

| Envelope 状态 | Cache-Control |
|---|---|
| `COMPLETED` | `public, max-age=31536000, immutable` |
| 其他（DRAFT/PENDING/REJECTED） | `public, max-age=0, must-revalidate` |

### 6.2 静态文件 — 下载（isDownload = true）

```
Cache-Control: no-cache, no-store, must-revalidate
Pragma: no-cache
Expires: 0
```

完全禁止缓存，确保每次下载获取最新数据。

### 6.3 Pending 文件

```
Cache-Control: no-store, private
```

- `no-store`：不缓存（因为内容可能随字段签署而变化）
- `private`：仅客户端可缓存（不应被共享缓存/CDN 缓存）

### 6.4 硬缓存路由（item.pdf）

```
Cache-Control: {private|public}, max-age=31536000, immutable
```

- 因为 URL 包含 `documentDataId`，数据不变则 URL 不变，可安全长期缓存
- Session 访问用 `private`，Token 访问也用 `private`

---

## 7. Content-Disposition 生成（下载模式）

在 [handleStaticFileRequest](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/apps/remix/server/api/files/files.helpers.ts#L119-L131) 和 [handlePendingFileRequest](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/apps/remix/server/api/files/files.helpers.ts#L229-L232) 中：

### 7.1 文件名生成规则

| Version | 文件名模式 | 示例 |
|---|---|---|
| `signed` | `{baseTitle}_signed.pdf` | `合同_signed.pdf` |
| `original` | `{baseTitle}.pdf` | `合同.pdf` |
| `pending` | `{baseTitle}_pending.pdf` | `合同_pending.pdf` |

其中 `baseTitle = title.replace(/\.pdf$/, '')`（signed/original）或 `title.replace(/\.pdf$/i, '')`（pending，多了 `i` 标志）。

### 7.2 Header 设置

使用 `content-disposition` 库生成：

```typescript
c.header('Content-Disposition', contentDisposition(filename));
```

对于 ASCII 文件名，生成 `Content-Disposition: attachment; filename="合同_signed.pdf"`；对于非 ASCII 字符，库会自动生成 `filename*` 参数（RFC 5987 编码）。

---

## 8. Pending 下载的限制条件

定义在 [handlePendingFileRequest](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/apps/remix/server/api/files/files.helpers.ts#L138-L164)：

### 8.1 必须为 PENDING 状态

```typescript
if (envelope.status !== DocumentStatus.PENDING) {
  const errorCode = match(envelope.status)
    .with(DocumentStatus.DRAFT, () => AppErrorCode.ENVELOPE_DRAFT)
    .with(DocumentStatus.COMPLETED, () => AppErrorCode.ENVELOPE_COMPLETED)
    .with(DocumentStatus.REJECTED, () => AppErrorCode.ENVELOPE_REJECTED)
    .otherwise(() => AppErrorCode.INVALID_REQUEST);
  throw new AppError(errorCode, { ... });
}
```

**原因**：
- `DRAFT`：信封尚未发送，没有签署流程，不存在"部分签署"
- `COMPLETED`：信封已完成，应使用 `signed` 版本（含完整签署和证书页）
- `REJECTED`：信封已被拒绝，不应提供部分签署预览

### 8.2 必须为 internalVersion 2

```typescript
if (envelope.internalVersion !== 2) {
  throw new AppError(AppErrorCode.ENVELOPE_LEGACY, { ... });
}
```

**原因**：
- `internalVersion: 1` 的旧版信封使用遗留字段插入逻辑（`useLegacyFieldInsertion: true`），其字段坐标、渲染方式与 V2 不兼容
- `generatePartialSignedPdf` 使用 [insertFieldInPDFV2](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/pdf/insert-field-in-pdf-v2.ts)，仅适用于 V2 信封的字段格式
- V1 信封的字段布局可能无法正确渲染到 PDF 上

### 8.3 Token 路由不支持 pending

在 [files.types.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/apps/remix/server/api/files/files.types.ts#L57-L61) 中，Token download 的 version 枚举只有 `signed` 和 `original`：

```typescript
version: z.enum(['signed', 'original']).default('signed')
```

这意味着收件人（Recipient Token）和 QR 扫码者无法下载 pending 版本，只有通过 Session 认证的团队内部用户才能查看部分签署预览。

---

## 9. `insertedFields` 生成部分签署 PDF 的流程

### 9.1 字段查询

```typescript
const fields = await prisma.field.findMany({
  where: {
    envelopeItemId,
    inserted: true,  // 仅查询已插入（已签署）的字段
  },
  include: {
    signature: true,  // 包含签名图像数据
  },
  orderBy: { id: 'asc' },
});
```

- 只取 `inserted: true` 的字段，即收件人已经完成签署/填写的字段
- `signature` 关联包含 `signatureImageAsBase64`（手写/上传签名图）和 `typedSignature`（打字签名）

### 9.2 PDF 生成过程

[generatePartialSignedPdf](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/pdf/generate-partial-signed-pdf.ts#L20-L76)：

1. **加载原始 PDF**：`PDF.load(pdfData)` — 使用 `initialData` 解码后的 `Uint8Array`
2. **扁平化**：`pdfDoc.flattenAll()` — 消除所有 PDF 图层
3. **升级版本**：`pdfDoc.upgradeVersion('1.7')` — 确保兼容性
4. **按页分组字段**：`groupBy(fields, (field) => field.page)`
5. **逐页渲染**：
   - 获取页面尺寸（`pageWidth`、`pageHeight`）
   - 调用 `insertFieldInPDFV2` 生成覆盖层 PDF 字节：
     - 为每个字段在对应位置渲染签名图像、文本、日期等内容
     - 返回仅包含该页覆盖内容的 PDF Buffer
   - 将覆盖层 PDF 嵌入主文档：`pdfDoc.embedPage(overlayPdf, 0)`
   - 处理页面旋转：根据 `page.rotation`（0°/90°/180°/270°）计算平移偏移
   - 将嵌入页面绘制到主文档页面上：`page.drawPage(embeddedPage, { x, y, rotate })`
6. **最终扁平化**：`pdfDoc.flattenAll()` — 确保所有注解/表单不可编辑
7. **保存**：`pdfDoc.save({ useXRefStream: true })` — 使用 XRef 流式存储，返回 `Uint8Array`

### 9.3 生成结果特点

- **无 PKI 签名**：不是合法的数字签名文档
- **无证书页**：不包含签署证书
- **无审计日志附录**：不包含审计追踪
- 仅作为签署进度预览，不可作为最终法律文件

---

## 10. 完整生命周期时序图

```
上传            存储                    签署中                     封存                     下载
 │               │                       │                         │                       │
 ▼               ▼                       ▼                         ▼                       ▼
POST /upload-pdf  putFileServerSide    签署者签署字段         seal job 执行          GET .../download/:version
 │               │                     │                         │                       │
 ├─ size <= 50MB? ├─ database:          ├─ Field.inserted=true   ├─ data 更新为           ├─ signed → data
 │  └─ 否 → 400  │  BYTES_64 + base64  │  └─ Signature 创建     │  完整签署PDF          ├─ original → initialData
 ├─ normalizePdf ├─ s3/azure:           │                         │  (含证书页/审计)       ├─ pending → initialData + insertedFields
 │  ├─ 校验有效   │  S3_PATH + key      │                         │  initialData 不变      │   └─ 仅 PENDING + V2
 │  ├─ 拒绝加密   │                     │                         │                       │
 │  ├─ flatten    │                     │                         │                       │
 │  └─ 保存      │                     │                         │                       │
 │               │                     │                         │                       │
 ▼               ▼                       ▼                         ▼                       ▼
DocumentData  DocumentData            Envelope.status:       Envelope.status:        ETag/Cache-Control/
.type/data    .initialData=data       PENDING                COMPLETED               Content-Disposition
.initialData=data                                            data=签署后PDF           按规则设置
```

---

## 11. 关键数据模型关系

```
Envelope
  ├── internalVersion (1=legacy, 2=current)
  ├── status (DRAFT → PENDING → COMPLETED/REJECTED)
  ├── qrToken (完成签署后生成)
  ├── recipients[] → token (收件人访问令牌)
  └── envelopeItems[]
        ├── title (文件名来源)
        └── documentData (DocumentData)
              ├── type (BYTES / BYTES_64 / S3_PATH)
              ├── data (签署后PDF / S3 key / base64)
              └── initialData (原始PDF / 同data)
                    └── fields[]
                          ├── inserted (是否已签署)
                          ├── page, positionX, positionY, width, height
                          ├── type (SIGNATURE/TEXT/DATE/...)
                          └── signature
                                ├── signatureImageAsBase64
                                └── typedSignature
```
