# multipart POST /envelope/create 请求全链路追踪

> 本文档追踪一次 `multipart/form-data` 的 `POST /envelope/create` 请求，从 `payload` 与 `files` 进入 `createEnvelopeRouteCaller`，到最终写入 `Envelope`、`EnvelopeItem`、`DocumentMeta`、`Recipient`、`Field` 和 `DocumentAuditLog` 的全过程。

---

## 1. 请求入口与 Zod 校验

### 1.1 路由定义

文件：[create-envelope.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/trpc/server/envelope-router/create-envelope.ts#L21-L39)

```ts
export const createEnvelopeRoute = authenticatedProcedure
  .meta(createEnvelopeMeta)
  .input(ZCreateEnvelopeRequestSchema)
  .output(ZCreateEnvelopeResponseSchema)
  .mutation(async ({ input, ctx }) => {
    return await createEnvelopeRouteCaller({
      userId: ctx.user.id,
      teamId: ctx.teamId,
      input,
      apiRequestMetadata: ctx.metadata,
      logger: ctx.logger,
    });
  });
```

OpenAPI 元信息定义在 [create-envelope.types.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/trpc/server/envelope-router/create-envelope.types.ts#L23-L32)：

```ts
export const createEnvelopeMeta: TrpcRouteMeta = {
  openapi: {
    method: 'POST',
    path: '/envelope/create',
    contentTypes: ['multipart/form-data'],
    ...
  },
};
```

### 1.2 请求 Schema 解构

文件：[create-envelope.types.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/trpc/server/envelope-router/create-envelope.types.ts#L82-L85)

```ts
export const ZCreateEnvelopeRequestSchema = zodFormData({
  payload: zfd.json(ZCreateEnvelopePayloadSchema),
  files: zfd.repeatableOfType(zfdFile()),
});
```

- **`payload`**：JSON 字符串，由 `zfd.json()` 解析后经 `ZCreateEnvelopePayloadSchema` 校验，包含 `title`、`type`（EnvelopeType.DOCUMENT | TEMPLATE）、`delegatedDocumentOwner`、`externalId`、`visibility`、`globalAccessAuth`、`globalActionAuth`、`formValues`、`folderId`、`recipients`（含 `fields` 子数组）、`meta`、`attachments`。
- **`files`**：可重复的文件上传字段，支持 PDF 与 DOCX。

### 1.3 进入 createEnvelopeRouteCaller

文件：[create-envelope.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/trpc/server/envelope-router/create-envelope.ts#L65-L72)

```ts
export const createEnvelopeRouteCaller = async ({
  userId, teamId, input, apiRequestMetadata, logger, options = {},
}: CreateEnvelopeRouteOptions) => {
  const { payload, files } = input;
  const { title, type, externalId, visibility, globalAccessAuth, globalActionAuth,
          formValues, recipients, folderId, meta, attachments, delegatedDocumentOwner } = payload;
  ...
```

---

## 2. 前置校验链（createEnvelopeRouteCaller 层）

### 2.1 getServerLimits — 获取服务端配额

文件：[server.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/ee/server-only/limits/server.ts#L16-L118)

```ts
const { remaining, maximumEnvelopeItemCount } = await getServerLimits({ userId, teamId });
```

**执行逻辑**：

1. 根据 `teamId` + `userId` 查询所属 `Organisation`，同时 include `subscription` 和 `organisationClaim`。
2. 从 `organisationClaim.envelopeItemCount` 获取 `maximumEnvelopeItemCount`。
3. 根据不同场景返回配额：
   - **自托管（`!IS_BILLING_ENABLED()`）**：返回 `SELFHOSTED_PLAN_LIMITS`（全部 Infinity）。
   - **Enterprise 内部 Claim**：返回 `PAID_PLAN_LIMITS`（全部 Infinity），绕过所有限制。
   - **订阅过期（`SubscriptionStatus.INACTIVE`）**：返回 `INACTIVE_PLAN_LIMITS`（全部 0）。
   - **`unlimitedDocuments` flag**：返回 `PAID_PLAN_LIMITS`。
   - **普通情况**：以 `FREE_PLAN_LIMITS`（5 docs / 10 recipients / 3 directTemplates）为基准，减去当月已用量得到 `remaining`。
4. 当月用量统计范围：`createdAt >= 月初`，排除 `TEMPLATE_DIRECT_LINK` 来源。

### 2.2 remaining.documents 限额检查

文件：[create-envelope.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/trpc/server/envelope-router/create-envelope.ts#L95-L100)

```ts
if (remaining.documents <= 0) {
  throw new AppError(AppErrorCode.LIMIT_EXCEEDED, { ... });
}
```

当剩余文档配额 ≤ 0 时，抛出 `LIMIT_EXCEEDED`。

### 2.3 maximumEnvelopeItemCount — 信封条目数量上限

文件：[create-envelope.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/trpc/server/envelope-router/create-envelope.ts#L102-L107)

```ts
if (files.length > maximumEnvelopeItemCount) {
  throw new AppError('ENVELOPE_ITEM_LIMIT_EXCEEDED', { ... });
}
```

`maximumEnvelopeItemCount` 来自 `organisationClaim.envelopeItemCount`（Prisma Schema 中定义的 Int 字段），限制单次请求上传文件数。

---

## 3. 文件处理管线（createEnvelopeRouteCaller 层）

每个文件并行经过以下管线：

### 3.1 convertToPdf — 格式转换

文件：[index.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/document-conversion/index.ts#L27-L43)

```ts
let pdf = await convertToPdf(file, logger);
```

- **PDF 输入**：直接返回 Buffer，无转换。
- **DOCX 输入**：调用 `convertDocxToPdf` 将其转为 PDF。
- **其他类型**：抛出 `UNSUPPORTED_FILE_TYPE`。

### 3.2 insertFormValuesInPdf — 将 formValues 写入 PDF

文件：[insert-form-values-in-pdf.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/pdf/insert-form-values-in-pdf.ts#L8-L24)

```ts
if (formValues) {
  pdf = await insertFormValuesInPdf({ pdf, formValues });
}
```

**执行逻辑**：

1. 用 `PDF.load` 加载 PDF Buffer。
2. 获取 PDF 表单 `form`。若无表单，直接返回原 Buffer。
3. 将 `formValues`（`Record<string, string | boolean | number>`）中 boolean 转为字符串，其余 `toString()`。
4. 调用 `form.fill()` 填充表单字段。
5. 以 `incremental: true` 模式保存并返回新 Buffer。

**触发条件**：仅当 payload 中 `formValues` 非空时执行。

### 3.3 normalizePdf — PDF 标准化

文件：[normalize-pdf.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/pdf/normalize-pdf.ts#L5-L34)

```ts
const normalized = await normalizePdf(pdf, {
  flattenForm: type !== EnvelopeType.TEMPLATE,
});
```

**执行逻辑**：

1. 加载 PDF，若解析失败抛出 `INVALID_DOCUMENT_FILE`。
2. 若 PDF 加密，抛出 `INVALID_DOCUMENT_FILE`。
3. 执行 `pdfDoc.flattenLayers()`。
4. 若 `flattenForm=true`（即 type 为 DOCUMENT，非 TEMPLATE）且有表单，则 `form.flatten()` + `pdfDoc.flattenAnnotations()`。
5. 保存并返回标准化后的 Buffer。

**关键**：TEMPLATE 类型保留表单字段不 flatten，DOCUMENT 类型 flatten 表单。

### 3.4 extractPdfPlaceholders — 提取并清除占位符

文件：[auto-place-fields.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/pdf/auto-place-fields.ts#L183-L195)

```ts
const { cleanedPdf, placeholders } = await extractPdfPlaceholders(normalized);
```

**执行逻辑**：

1. **`extractPlaceholdersFromPDF`**：遍历每页，用 `PLACEHOLDER_REGEX = /\{\{([^}]+)\}\}/g` 匹配文本。对每个匹配：
   - 解析内部内容，如 `{{SIGNATURE, r1, required=true}}` → `[SIGNATURE, r1, required=true]`。
   - 第一个元素为字段类型（经 `parseFieldTypeFromPlaceholder` 映射为 `FieldType` 枚举）。
   - 第二个元素必须匹配 `/^r\d+$/i`（如 `r1`、`R2`），否则跳过。
   - 后续元素为 `key=value` 形式的元数据，经 `parseFieldMetaFromPlaceholder` 转换。
   - 计算页面坐标（转换为左上角原点）。
2. **`removePlaceholdersFromPDF`**：若无占位符则返回原 Buffer；否则对每个占位符区域绘制白色矩形覆盖，然后保存。

**返回**：`{ cleanedPdf: Buffer, placeholders: PlaceholderInfo[] }`。

### 3.5 putPdfFileServerSide — 上传 PDF 并创建 DocumentData

文件：[put-file.server.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/universal/upload/put-file.server.ts#L22-L49)

```ts
const { documentData } = await putPdfFileServerSide({
  name: file.name,
  type: 'application/pdf',
  arrayBuffer: async () => Promise.resolve(cleanedPdf),
});
```

**执行逻辑**：

1. 加载 PDF 验证有效性（若解析失败抛 `INVALID_DOCUMENT_FILE`）。
2. 检查加密（当前 `isEncryptedDocumentsAllowed = false`，加密 PDF 被拒绝）。
3. 确保文件名以 `.pdf` 结尾。
4. 调用 `putFileServerSide` 根据 `NEXT_PUBLIC_UPLOAD_TRANSPORT` 环境变量选择存储：
   - **s3 / azure-blob**：上传至对象存储，返回 `DocumentDataType.S3_PATH` + key。
   - **其他**：Base64 编码存入数据库，返回 `DocumentDataType.BYTES_64` + base64 字符串。
5. 调用 `createDocumentData` 在 `DocumentData` 表创建记录。
6. 返回 `{ documentData, filePageCount }`。

### 3.6 组装 envelopeItems

文件：[create-envelope.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/trpc/server/envelope-router/create-envelope.ts#L110-L141)

每个文件处理后返回：

```ts
{
  title: file.name,
  documentDataId: documentData.id,
  placeholders,
}
```

---

## 4. 收件人字段映射（createEnvelopeRouteCaller 层）

文件：[create-envelope.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/trpc/server/envelope-router/create-envelope.ts#L143-L176)

### 4.1 field.identifier 解析

每个收件人的 `fields` 中，`identifier` 字段用于定位文件对应的 `documentDataId`：

| `identifier` 类型 | 解析方式 |
|---|---|
| `string` | 按 `envelopeItems.find(item => item.title === field.identifier)` 匹配文件名 |
| `number` | 按 `envelopeItems.at(field.identifier)` 索引匹配 |
| `undefined` | 默认取 `envelopeItems.at(0)`，即第一个文件 |

若找不到对应的 `documentDataId`，抛出 `NOT_FOUND`。

---

## 5. 核心写入：createEnvelope（lib 层）

文件：[create-envelope.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/envelope/create-envelope.ts#L110-L643)

### 5.1 assertUserNotDisabledById — 禁用账户检查

文件：[assert-user-not-disabled.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/user/assert-user-not-disabled.ts#L34-L48)

```ts
await assertUserNotDisabledById({ userId });
```

从数据库重新查询用户的 `disabled` 字段（防止缓存 session/token 绕过），若 `disabled=true` 则抛出 `ACCOUNT_DISABLED`（403）。

### 5.2 团队验证

```ts
const team = await prisma.team.findFirst({
  where: buildTeamWhereQuery({ teamId, userId }),
  include: { organisation: { select: { organisationClaim: true } } },
});
```

确保用户是团队成员。若团队不存在，抛出 `NOT_FOUND`。

### 5.3 assertOrganisationRatesAndLimits — 组织级速率与配额检查

文件：[assert-organisation-rates-and-limits.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/rate-limit/assert-organisation-rates-and-limits.ts#L29-L89)

```ts
if (type === EnvelopeType.DOCUMENT) {
  await assertOrganisationRatesAndLimits({
    organisationId: team.organisationId,
    organisationClaim: team.organisation.organisationClaim,
    type: 'document',
    count: 1,
  });
}
```

**触发条件**：仅当 `type === EnvelopeType.DOCUMENT` 时检查（TEMPLATE 不受限制）。

**执行逻辑**：

1. 若 `DANGEROUS_BYPASS_RATE_LIMITS=true`，直接返回。
2. 若 `count=0`，视为 no-op。
3. 若 `count<0`，抛出 `INVALID_REQUEST`。
4. 从 `organisationClaim` 中取出 `documentRateLimits` 和 `documentQuota`。
5. **`checkOrganisationRateLimits`**：对每个窗口（如 `10m`、`1h`），用 bucketed counter 检查速率。若超限，抛出 `TOO_MANY_REQUESTS` 并附带 `X-RateLimit-*` 和 `Retry-After` 响应头。
6. **`checkMonthlyQuota`**：
   - 若 `quota=0`，直接拒绝。
   - 原子 upsert `OrganisationMonthlyStat`，递增 `documentCount`。
   - 若 `quota=null`（无限量），允许但仍然记录统计。
   - 若 `quota` 有值且超限，抛出 `TOO_MANY_REQUESTS`。
   - 在恰好跨过阈值的那次请求上触发 `send.organisation-limit-exceeded.email` 作业。

### 5.4 folder 校验

文件：[create-envelope.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/envelope/create-envelope.ts#L171-L185)

```ts
if (folderId) {
  const folder = await prisma.folder.findUnique({
    where: {
      id: folderId,
      type: data.type === EnvelopeType.TEMPLATE ? FolderType.TEMPLATE : FolderType.DOCUMENT,
      team: buildTeamWhereQuery({ teamId, userId }),
    },
  });
  if (!folder) throw new AppError(AppErrorCode.NOT_FOUND, { message: 'Folder not found' });
}
```

**校验要点**：
- folderId 非空时才校验。
- 文件夹类型必须与信封类型匹配（DOCUMENT → FolderType.DOCUMENT，TEMPLATE → FolderType.TEMPLATE）。
- 文件夹必须属于当前用户所在的团队。

### 5.5 cfr21 actionAuth 限制

文件：[create-envelope.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/envelope/create-envelope.ts#L239-L256)

```ts
const authOptions = createDocumentAuthOptions({
  globalAccessAuth: globalAccessAuth || [],
  globalActionAuth: globalActionAuth || [],
});

const recipientsHaveActionAuth = data.recipients?.some(
  (recipient) => recipient.actionAuth && recipient.actionAuth.length > 0,
);

if (
  (authOptions.globalActionAuth.length > 0 || recipientsHaveActionAuth) &&
  !team.organisation.organisationClaim.flags.cfr21
) {
  throw new AppError(AppErrorCode.UNAUTHORIZED, { message: 'You do not have permission to set the action auth' });
}
```

**规则**：只有组织的 `organisationClaim.flags.cfr21` 为 true 时，才允许设置 `globalActionAuth` 或收件人级别的 `actionAuth`。否则抛出 `UNAUTHORIZED`。

### 5.6 delegatedDocumentOwner 校验

文件：[create-envelope.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/envelope/create-envelope.ts#L282-L310)

```ts
const getValidatedDelegatedOwner = async () => {
  if (!settings.delegateDocumentOwnership || !delegatedDocumentOwner || requestMetadata.source === 'app') {
    return null;
  }
  const delegatedOwner = await prisma.user.findFirst({ where: { email: delegatedDocumentOwner } });
  if (!delegatedOwner) throw new AppError(AppErrorCode.UNAUTHORIZED, { ... });
  const isTeamMember = await prisma.team.findFirst({
    where: buildTeamWhereQuery({ teamId, userId: delegatedOwner.id }),
  });
  if (!isTeamMember) throw new AppError(AppErrorCode.UNAUTHORIZED, { ... });
  return delegatedOwner;
};
```

**三重前置条件**（全部满足才进入校验）：
1. 组织/团队设置中 `delegateDocumentOwnership` 为 true。
2. 请求中提供了 `delegatedDocumentOwner` 邮箱。
3. 请求来源不是 `app`（即来自 API）。

**校验逻辑**：
- 该邮箱对应的用户必须存在。
- 该用户必须是当前团队的成员。

若校验通过，返回 `delegatedOwner` 对象，后续 `envelopeOwnerId = delegatedOwner.id`；否则 `envelopeOwnerId = userId`。

### 5.7 DocumentMeta 创建

文件：[create-envelope.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/envelope/create-envelope.ts#L312-L318)

```ts
const [documentMeta, secondaryId, delegatedOwner] = await Promise.all([
  prisma.documentMeta.create({
    data: extractDerivedDocumentMeta(settings, { ...meta, timezone: timezoneToUse }),
  }),
  type === EnvelopeType.DOCUMENT
    ? incrementDocumentId().then((v) => v.formattedDocumentId)
    : incrementTemplateId().then((v) => v.formattedTemplateId),
  getValidatedDelegatedOwner(),
]);
```

`extractDerivedDocumentMeta`（[document.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/utils/document.ts#L28-L64)）合并组织/团队设置与请求 meta 覆盖，优先级：meta 覆盖 > 组织设置 > 默认值。生成的字段包括：`language`、`timezone`、`dateFormat`、`signingOrder`、`distributionMethod`、签名方式开关、邮件设置、过期/提醒设置等。

---

## 6. 数据库事务写入

文件：[create-envelope.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/envelope/create-envelope.ts#L326-L622)

以下所有操作在 `prisma.$transaction` 中执行。

### 6.1 创建 Envelope

```ts
const envelope = await tx.envelope.create({
  data: {
    id: prefixedId('envelope'),
    secondaryId,
    internalVersion,
    type, title, qrToken, externalId,
    envelopeItems: { createMany: { data: envelopeItems.map(...) } },
    envelopeAttachments: { createMany: { data: (attachments || []).map(...) } },
    userId: envelopeOwnerId,
    teamId,
    authOptions,
    visibility,
    folderId,
    formValues,
    source: type === EnvelopeType.DOCUMENT ? DocumentSource.DOCUMENT : DocumentSource.TEMPLATE,
    documentMetaId: documentMeta.id,
    ...
  },
  include: { envelopeItems: true },
});
```

写入 `Envelope` 表，同时批量创建 `EnvelopeItem` 和 `EnvelopeAttachment`。

### 6.2 创建 defaultRecipients

文件：[create-envelope.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/envelope/create-envelope.ts#L376-L387)

```ts
const defaultRecipients =
  settings.defaultRecipients && !bypassDefaultRecipients
    ? ZDefaultRecipientsSchema.parse(settings.defaultRecipients)
    : [];

const mappedDefaultRecipients = defaultRecipients.map((recipient) => ({
  email: recipient.email, name: recipient.name, role: recipient.role,
}));

const allRecipients = [...(data.recipients || []), ...mappedDefaultRecipients];
```

**逻辑**：
- 从团队/组织设置中获取 `defaultRecipients`。
- 若 `bypassDefaultRecipients=true`，跳过默认收件人。
- 请求中的收件人排在前面，默认收件人追加在后面。

### 6.3 创建 Recipient 及其 Fields

文件：[create-envelope.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/envelope/create-envelope.ts#L389-L447)

对每个收件人：
1. 构造 `recipientAuthOptions`。
2. 映射 `fields`：每个 field 的 `documentDataId` 用于在 `envelope.envelopeItems` 中找到对应 `envelopeItemId`。若未指定，默认使用 `firstEnvelopeItem.id`。
3. 创建 `Recipient` 记录，同时 `createMany` 其 `Field` 记录。

**Recipient 特殊初始化**：
- CC 角色：`sendStatus = SENT`，`signingStatus = SIGNED`。
- 其他角色：`sendStatus = NOT_SENT`，`signingStatus = NOT_SIGNED`。

### 6.4 创建 placeholderRecipients 及其 Fields

文件：[create-envelope.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/envelope/create-envelope.ts#L450-L556)

**触发条件**：`envelopeItems` 中存在带 `placeholders` 的条目。

**逻辑**：

1. 收集所有占位符中的收件人引用（如 `r1`、`r2`），存入 `uniqueRecipientRefs` Map。
2. 查询已创建的收件人。
3. **`shouldCreatePlaceholderRecipients`**：当 `(!data.recipients || data.recipients.length === 0) && uniqueRecipientRefs.size > 0` 时为 true。
4. 若需要创建占位符收件人：
   - 为每个 `r{n}` 生成邮箱 `recipient.{n}@documenso.com`，名称 `Recipient {n}`，角色 `SIGNER`。
   - 过滤掉已存在的邮箱。
   - `createMany` 批量创建。
   - 重新查询所有收件人。
5. 对每个有占位符的 `envelopeItem`：
   - 调用 `convertPlaceholdersToFieldInputs` 将占位符转换为字段输入（坐标从点转为百分比）。
   - 收件人解析通过 `findRecipientByPlaceholder`：
     - 若请求提供了 `data.recipients`：`r1` → `recipients[0]`，`r2` → `recipients[1]`（索引映射）。
     - 若未提供收件人：按邮箱匹配已创建的占位符收件人。
   - `createMany` 批量创建 `Field` 记录。

### 6.5 DOCUMENT_CREATED 审计日志

文件：[create-envelope.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/envelope/create-envelope.ts#L583-L619)

```ts
if (type === EnvelopeType.DOCUMENT) {
  await tx.documentAuditLog.create({
    data: createDocumentAuditLogData({
      type: DOCUMENT_AUDIT_LOG_TYPE.DOCUMENT_CREATED,
      envelopeId: envelope.id,
      user: { id: envelopeOwnerId },
      metadata: requestMetadata,
      data: { title, source: { type: DocumentSource.DOCUMENT } },
    }),
  });

  if (delegatedOwner) {
    await tx.documentAuditLog.create({
      data: createDocumentAuditLogData({
        type: DOCUMENT_AUDIT_LOG_TYPE.DOCUMENT_DELEGATED_OWNER_CREATED,
        envelopeId: envelope.id,
        user: { id: userId },
        metadata: requestMetadata,
        data: { delegatedOwnerName: delegatedOwner.name, delegatedOwnerEmail: delegatedOwner.email, teamName: team.name },
      }),
    });
  }
}
```

**条件**：
- `DOCUMENT_CREATED`：仅当 `type === EnvelopeType.DOCUMENT` 时创建。
- `DOCUMENT_DELEGATED_OWNER_CREATED`：在 `DOCUMENT` 基础上，还要求 `delegatedOwner` 校验通过（非 null）。
- TEMPLATE 类型**不会**创建任何审计日志。

`createDocumentAuditLogData`（[document-audit-logs.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/utils/document-audit-logs.ts#L40-L72)）提取 `userId`、`email`、`name`（优先取显式 `user` 参数，其次取 `metadata.auditUser`），以及 `ipAddress` 和 `userAgent`。

---

## 7. Webhook 触发（事务外）

文件：[create-envelope.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/envelope/create-envelope.ts#L624-L640)

```ts
if (type === EnvelopeType.DOCUMENT) {
  await triggerWebhook({
    event: WebhookTriggerEvents.DOCUMENT_CREATED,
    data: ZWebhookDocumentSchema.parse(mapEnvelopeToWebhookDocumentPayload(createdEnvelope)),
    userId, teamId,
  });
} else if (type === EnvelopeType.TEMPLATE) {
  await triggerWebhook({
    event: WebhookTriggerEvents.TEMPLATE_CREATED,
    data: ZWebhookDocumentSchema.parse(mapEnvelopeToWebhookDocumentPayload(createdEnvelope)),
    userId, teamId,
  });
}
```

| Webhook 事件 | 触发条件 |
|---|---|
| `DOCUMENT_CREATED` | `type === EnvelopeType.DOCUMENT` |
| `TEMPLATE_CREATED` | `type === EnvelopeType.TEMPLATE` |

`triggerWebhook`（[trigger-webhook.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/webhooks/trigger/trigger-webhook.ts#L13-L37)）：
1. 查询该 `userId` + `teamId` 下注册了对应事件的所有 Webhook。
2. 若无注册 Webhook，直接返回。
3. 对每个 Webhook，通过 `jobs.triggerJob` 异步执行 `internal.execute-webhook` 作业。

**关键**：Webhook 在事务提交后触发，避免长事务持有数据库连接。

---

## 8. 全链路数据写入汇总

| 步骤 | 写入的模型 | 位置 |
|---|---|---|
| 文件上传 | `DocumentData` | `putPdfFileServerSide` |
| 元信息创建 | `DocumentMeta` | `prisma.documentMeta.create`（事务前） |
| 信封创建 | `Envelope` | `tx.envelope.create` |
| 信封条目 | `EnvelopeItem` | `envelope.create` 内嵌 `createMany` |
| 信封附件 | `EnvelopeAttachment` | `envelope.create` 内嵌 `createMany` |
| 显式收件人 | `Recipient` | `tx.recipient.create` |
| 收件人字段 | `Field` | `recipient.create` 内嵌 `createMany` |
| 占位符收件人 | `Recipient` | `tx.recipient.createMany` |
| 占位符字段 | `Field` | `tx.field.createMany` |
| 审计日志 | `DocumentAuditLog` | `tx.documentAuditLog.create` |
| 配额统计 | `OrganisationMonthlyStat` | `checkMonthlyQuota`（upsert） |

---

## 9. 容易遗漏的边界条件

### 边界条件 1：placeholderRecipients 与 defaultRecipients 的交互冲突

**问题描述**：

当请求**未提供** `data.recipients`，但组织/团队设置中配置了 `defaultRecipients`，同时上传的 PDF 中包含 `{{SIGNATURE, r1}}` 等占位符时，存在两条收件人创建路径的交互：

1. `allRecipients` 会先包含 `mappedDefaultRecipients`（默认收件人）。
2. `shouldCreatePlaceholderRecipients` 的判断条件是 `(!data.recipients || data.recipients.length === 0) && uniqueRecipientRefs.size > 0`。
3. 当 `data.recipients` 为空/未提供时，条件为 true，会额外创建 `recipient.{n}@documenso.com` 占位符收件人。
4. 但默认收件人已经通过 `allRecipients` 写入了，占位符收件人的字段会通过 `findRecipientByPlaceholder` 中的邮箱匹配找到的是占位符收件人，而非默认收件人。

**结果**：默认收件人没有关联字段，占位符收件人有关联字段但邮箱是虚拟地址。这可能导致发送时默认收件人收到空白信封，而占位符收件人的邮件无法送达。

**回归测试设计**：

```ts
// 测试场景：无显式 recipients + 有 defaultRecipients + PDF 有占位符
describe('placeholderRecipients with defaultRecipients interaction', () => {
  it('should not create placeholder recipients when defaultRecipients exist and data.recipients is empty', async () => {
    // 设置：团队配置 defaultRecipients: [{ email: 'admin@example.com', role: SIGNER }]
    // 上传：包含 {{SIGNATURE, r1}} 占位符的 PDF
    // 请求：不提供 recipients
    // 预期：占位符字段应关联到 defaultRecipients 中的收件人，而非创建新的虚拟收件人
  });

  it('should map r1 to first default recipient when no explicit recipients provided', async () => {
    // 验证 findRecipientByPlaceholder 在 recipients[] 为空时的行为
    // 当 data.recipients 为空但 allRecipients 包含 defaults 时
    // 传入 convertPlaceholdersToFieldInputs 的 resolver 应能正确映射
  });
});
```

### 边界条件 2：formValues 写入后 normalizePdf 的 flatten 顺序导致数据丢失

**问题描述**：

在 `createEnvelopeRouteCaller` 中，文件处理管线的顺序为：

1. `convertToPdf` → PDF
2. `insertFormValuesInPdf`（将 formValues 填入 PDF 表单字段）
3. `normalizePdf`（当 `type=DOCUMENT` 时 `flattenForm=true`）

当 `formValues` 非空且 `type=DOCUMENT` 时：
- 步骤 2 将值填入 PDF AcroForm 字段。
- 步骤 3 将表单 flatten，字段值被"烧录"到页面内容中，表单字段本身被移除。

这在视觉上是正确的，但**如果同一个 PDF 也包含 `{{SIGNATURE, r1}}` 占位符**，步骤 3 的 `flattenAnnotations` 可能在某些 PDF 库实现中改变文本层的布局，导致步骤 4 的 `extractPdfPlaceholders` 无法匹配到占位符文本。这是因为 `flattenAnnotations` 可能重新渲染注释层，改变文本坐标甚至丢失部分文本。

**结果**：占位符字段未被自动创建，信封缺少预期的签名域。

**回归测试设计**：

```ts
describe('formValues + normalizePdf + placeholder interaction', () => {
  it('should preserve placeholders after formValues insertion and normalizePdf with flattenForm=true', async () => {
    // 准备一个同时包含 AcroForm 字段和 {{SIGNATURE, r1}} 占位符文本的 PDF
    // formValues = { field1: 'value1' }
    // type = DOCUMENT
    // 执行完整管线后，验证 placeholders 仍然被正确提取
    // 且 formValues 的值已烧录到 PDF 中
  });

  it('should handle PDF with formValues but no form gracefully', async () => {
    // 提供一个无 AcroForm 的 PDF，但 payload 包含 formValues
    // 预期：insertFormValuesInPdf 返回原 Buffer，不抛错
    // 后续 normalizePdf 和 extractPdfPlaceholders 正常执行
  });

  it('should not flatten form for TEMPLATE type even with formValues', async () => {
    // type = TEMPLATE
    // formValues = { field1: 'value1' }
    // 预期：normalizePdf 的 flattenForm=false，表单字段保留
    // 占位符仍能被提取
  });
});
```

---

## 10. 附录：关键函数调用关系图

```
POST /envelope/create (multipart/form-data)
  │
  ├─ Zod 校验 (ZCreateEnvelopeRequestSchema)
  │   ├─ payload: zfd.json(ZCreateEnvelopePayloadSchema)
  │   └─ files: zfd.repeatableOfType(zfdFile())
  │
  └─ createEnvelopeRouteCaller
      │
      ├─ getServerLimits({ userId, teamId })
      │   ├─ 查询 Organisation + subscription + organisationClaim
      │   ├─ 自托管/Enterprise/过期/unlimitedDocuments 特殊处理
      │   └─ 返回 { remaining, maximumEnvelopeItemCount }
      │
      ├─ remaining.documents ≤ 0 ? → LIMIT_EXCEEDED
      ├─ files.length > maximumEnvelopeItemCount ? → ENVELOPE_ITEM_LIMIT_EXCEEDED
      │
      ├─ Promise.all(files.map(file => ...))
      │   ├─ convertToPdf(file, logger)
      │   ├─ formValues ? insertFormValuesInPdf({ pdf, formValues }) : skip
      │   ├─ normalizePdf(pdf, { flattenForm: type !== TEMPLATE })
      │   ├─ extractPdfPlaceholders(normalized) → { cleanedPdf, placeholders }
      │   └─ putPdfFileServerSide({ name, type, arrayBuffer: cleanedPdf })
      │       └─ createDocumentData → DocumentData 记录
      │
      ├─ recipients fields.identifier → documentDataId 映射
      │
      └─ createEnvelope({ userId, teamId, data, ... })
          │
          ├─ assertUserNotDisabledById({ userId })
          ├─ 团队验证 (buildTeamWhereQuery)
          ├─ type=DOCUMENT ? assertOrganisationRatesAndLimits → checkOrganisationRateLimits + checkMonthlyQuota
          ├─ folderId ? folder 校验 (type + team 归属)
          ├─ cfr21 actionAuth 限制校验
          ├─ emailId 校验
          ├─ Promise.all([DocumentMeta 创建, secondaryId 生成, delegatedOwner 校验])
          │
          └─ prisma.$transaction
              ├─ Envelope.create (含 EnvelopeItem.createMany + EnvelopeAttachment.createMany)
              ├─ defaultRecipients 处理
              ├─ Recipient.create × N (含 Field.createMany)
              ├─ placeholderRecipients 条件创建 (Recipient.createMany)
              ├─ placeholder → Field 转换 + Field.createMany
              ├─ type=DOCUMENT ? DocumentAuditLog.create(DOCUMENT_CREATED) : skip
              ├─ delegatedOwner ? DocumentAuditLog.create(DOCUMENT_DELEGATED_OWNER_CREATED) : skip
              └─ 返回 createdEnvelope
          │
          └─ 事务外 Webhook
              ├─ type=DOCUMENT → triggerWebhook(DOCUMENT_CREATED)
              └─ type=TEMPLATE → triggerWebhook(TEMPLATE_CREATED)
```
