# V2 Envelope 签署者完整签署路径推演

> 基于 `/sign/:token` 路由，从签署者打开链接到插入签名字段再到完成签署的全流程分析。

---

## 一、整体流程概览

```
签署者打开 /sign/:token
       │
       ▼
  loader 入口 ──► 查询 recipient.token 关联的 envelope.internalVersion
       │
       ├── internalVersion === 2 ──► handleV2Loader()
       │
       └── internalVersion === 1 ──► handleV1Loader()
```

---

## 二、Loader V1/V2 分流机制

文件: [_index.tsx](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/apps/remix/app/routes/_recipient+/sign.%24token+/_index.tsx#L262-L309)

`loader` 函数首先通过 `prisma.recipient.findFirst` 查询 token 对应的 recipient，并 select 其关联 envelope 的 `internalVersion` 和 `teamId`：

```ts
const foundRecipient = await prisma.recipient.findFirst({
  where: { token },
  select: {
    envelope: {
      select: { internalVersion: true, teamId: true },
    },
  },
});
```

- **token 不存在** → 抛出 `404`
- **`internalVersion === 2`** → 调用 `handleV2Loader(loaderArgs)`，返回 `{ version: 2, payload, branding }`
- **其他 (version 1)** → 调用 `handleV1Loader(loaderArgs)`，返回 `{ version: 1, payload, branding }`

前端 `SigningPage` 组件根据 `data.version` 选择渲染 `SigningPageV2` 或 `SigningPageV1`。

---

## 三、handleV2Loader 详细流程

文件: [_index.tsx](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/apps/remix/app/routes/_recipient+/sign.%24token+/_index.tsx#L170-L260)

### 3.1 获取信封数据与 accessAuth 校验

```ts
const envelopeForSigning = await getEnvelopeForRecipientSigning({ token, userId: user?.id })
```

- **成功** → 返回 `{ isDocumentAccessValid: true, ...envelopeForSigning }`
- **抛出 `UNAUTHORIZED`** → 调用 `getEnvelopeRequiredAccessData({ token })` 获取 `{ recipientEmail, recipientHasAccount }`，返回 `{ isDocumentAccessValid: false, ... }`
- **其他错误** → 抛出 `404`

### 3.2 accessAuth 失败处理

当 `isDocumentAccessValid === false` 时，loader 直接返回该状态，前端 `SigningPageV2` 渲染 `DocumentSigningAuthPageView`，展示认证页面，要求签署者通过账户登录或 2FA 验证来获取访问权限。

该认证页面接收：
- `recipientEmail` — 签署者邮箱
- `recipientHasAccount` — 该邮箱是否已注册账户（决定显示登录还是注册引导）

### 3.3 二次 accessAuth 验证（在 getEnvelopeForRecipientSigning 成功后）

即使 `getEnvelopeForRecipientSigning` 通过了服务端 access auth 检查，loader 仍在前端侧做一次 `derivedRecipientAccessAuth` 校验：

```ts
const { derivedRecipientAccessAuth } = extractDocumentAuthMethods({
  documentAuth: envelope.authOptions,
  recipientAuth: recipient.authOptions,
});

const isAccessAuthValid = derivedRecipientAccessAuth.every((accesssAuth) =>
  match(accesssAuth)
    .with(DocumentAccessAuth.ACCOUNT, () => user && user.email === recipient.email)
    .with(DocumentAccessAuth.TWO_FACTOR_AUTH, () => true) // 2FA 不要求账户，后续在 completeDocumentWithToken 中验证
    .exhaustive(),
);
```

- **`ACCOUNT`** → 当前登录用户的 email 必须与 recipient.email 一致
- **`TWO_FACTOR_AUTH`** → 直接通过（2FA 验证推迟到签署完成阶段）
- **不通过** → 同样返回 `isDocumentAccessValid: false`，展示认证页面

### 3.4 状态跳转决策链

在 accessAuth 通过后，按以下顺序检查并跳转：

| 条件 | 跳转目标 | 说明 |
|------|---------|------|
| `!isRecipientsTurn` | `/sign/${token}/waiting` | 顺序签署中，轮到前面的人先签 |
| `isRejected` | `/sign/${token}/rejected` | 签署者已拒绝或信封被拒绝 |
| `isCompleted` | `redirectUrl \|\| /sign/${token}/complete` | 签署者已签完或信封已完成 |
| `isExpired` | `/sign/${token}/expired` | 签署者已过期 |

### 3.5 viewedDocument 标记

所有检查通过后，调用 `viewedDocument({ token, requestMetadata, recipientAccessAuth })` 记录签署者已查看文档（审计日志 + webhook `DOCUMENT_OPENED`）。

### 3.6 前端渲染

最终 loader 返回 `{ isDocumentAccessValid: true, envelopeForSigning }`，前端 `SigningPageV2` 渲染：

```tsx
<EnvelopeSigningProvider envelopeData={envelopeForSigning} ...>
  <DocumentSigningAuthProvider documentAuthOptions={envelope.authOptions} recipient={recipient} user={user}>
    <EnvelopeRenderProvider version="current" envelope={envelope} ...>
      <DocumentSigningPageViewV2 />
    </EnvelopeRenderProvider>
  </DocumentSigningAuthProvider>
</EnvelopeSigningProvider>
```

若信封被删除或状态为 `REJECTED`，展示取消页面。

---

## 四、getEnvelopeForRecipientSigning 内部逻辑

文件: [get-envelope-for-recipient-signing.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/envelope/get-envelope-for-recipient-signing.ts#L156-L299)

### 4.1 查询信封

```ts
const envelope = await prisma.envelope.findFirst({
  where: {
    type: EnvelopeType.DOCUMENT,
    status: { not: DocumentStatus.DRAFT },
    recipients: { some: { token } },
  },
  include: { user, documentMeta, recipients (with fields + signature), envelopeItems, team },
});
```

- 不存在 → 抛出 `NOT_FOUND`
- `envelopeItems.length === 0` → 抛出 `NOT_FOUND`

### 4.2 ACCESS 权限校验

```ts
const documentAccessValid = await isRecipientAuthorized({
  type: 'ACCESS',
  documentAuthOptions: envelope.authOptions,
  recipient,
  userId,
  authOptions: accessAuth,
});
```

不通过则抛出 `UNAUTHORIZED`，由上层 `_index.tsx` catch 后返回认证页面数据。

### 4.3 isRecipientsTurn 判断

对于 `SEQUENTIAL` 签署顺序：遍历当前 recipient 之前的所有 recipient，如果任一 `signingStatus !== SIGNED`，则 `isRecipientsTurn = false`。

### 4.4 返回值

通过 `ZEnvelopeForSigningResponse.parse()` 验证后返回完整签署数据，包括 envelope、recipient、recipientSignature、isCompleted、isRejected、isExpired、isRecipientsTurn、sender、settings。

---

## 五、isRecipientAuthorized 鉴权核心

文件: [is-recipient-authorized.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/document/is-recipient-authorized.ts#L53-L170)

### 5.1 提取 auth 方法

通过 `extractDocumentAuthMethods` 从 document 和 recipient 的 authOptions 中推导出：
- `derivedRecipientAccessAuth` — 访问认证方法列表
- `derivedRecipientActionAuth` — 操作（签署）认证方法列表

规则：**recipient 级别的 auth 配置优先于 document 级别的全局配置**，只有 recipient 级别为空时才回退到 document 级别。

### 5.2 三种鉴权类型

| type | 使用的 auth 方法 | 场景 |
|------|----------------|------|
| `ACCESS` | `derivedRecipientAccessAuth` | 打开文档时 |
| `ACCESS_2FA` | `derivedRecipientAccessAuth` | 完成签署时验证 2FA |
| `ACTION` | `derivedRecipientActionAuth` | 签署字段时 |

### 5.3 快速通过路径

- auth 方法列表为空或包含 `EXPLICIT_NONE` → 直接返回 `true`
- `ACCESS` 类型且所有方法都是 `TWO_FACTOR_AUTH` → 返回 `true`（2FA 延迟到 `ACCESS_2FA` 阶段验证）

### 5.4 各认证方式验证

| 认证方式 | 验证逻辑 |
|---------|---------|
| `ACCOUNT` | `userId` 存在且与 recipient.email 对应用户的 ID 一致 |
| `PASSKEY` | 通过 WebAuthn `verifyAuthenticationResponse` 验证 passkey 签名 |
| `TWO_FACTOR_AUTH` | `ACCESS` → true；`ACCESS_2FA` + email 方法 → `validateTwoFactorTokenFromEmail`；其他 → `verifyTwoFactorAuthenticationToken` (TOTP) |
| `PASSWORD` | `verifyPassword({ userId, password })` |
| `EXPLICIT_NONE` | 直接返回 `true` |

---

## 六、signEnvelopeFieldRoute — 签署字段

文件: [sign-envelope-field.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/trpc/server/envelope-router/sign-envelope-field.ts#L15-L283)

这是一个**未认证的公开 tRPC mutation 路由**。

### 6.1 输入 Schema

来自 [sign-envelope-field.types.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/trpc/server/envelope-router/sign-envelope-field.types.ts#L7-L48)：

```ts
ZSignEnvelopeFieldRequestSchema = z.object({
  token: z.string(),
  fieldId: z.number(),
  fieldValue: ZSignEnvelopeFieldValue,  // discriminatedUnion on 'type'
  authOptions: ZRecipientActionAuthSchema.optional(),
});
```

`ZSignEnvelopeFieldValue` 是基于 `type` 字段的 discriminated union，包含 CHECKBOX、RADIO、NUMBER、EMAIL、NAME、INITIALS、TEXT、DROPDOWN、DATE、SIGNATURE 十种字段类型。

### 6.2 前置校验链

#### 6.2.1 ASSISTANT 限制

```ts
const field = await prisma.field.findFirst({
  where: {
    id: fieldId,
    recipient: {
      ...(recipient.role === RecipientRole.ASSISTANT
        ? {
            signingStatus: { not: SigningStatus.SIGNED },
            signingOrder: { gte: recipient.signingOrder ?? 0 },
            envelopeId: recipient.envelopeId,
          }
        : { id: recipient.id }),
    },
  },
});
```

ASSISTANT 可以操作同一 envelope 中 `signingOrder >= 自己` 且未签署的 recipient 的字段，但**不能签署 SIGNATURE 类型字段**：

```ts
if (field.type === FieldType.SIGNATURE && recipient.id !== field.recipientId && recipient.role === RecipientRole.ASSISTANT) {
  throw new AppError(AppErrorCode.INVALID_REQUEST, { message: 'Assistant recipients cannot sign signature fields' });
}
```

#### 6.2.2 FieldType.SIGNATURE 匹配

```ts
if (fieldValue.type !== field.type) {
  throw new AppError(AppErrorCode.NOT_FOUND, { message: 'Selected values do not match the field values' });
}
```

提交的 `fieldValue.type` 必须与数据库中 `field.type` 完全一致。

#### 6.2.3 fieldValue.type 验证

`ZSignEnvelopeFieldValue` 通过 `z.discriminatedUnion('type', [...])` 在输入层就确保了 `fieldValue.type` 只能是合法的 `FieldType` 枚举值之一，并且每种类型的 `value` 结构都经过严格校验。

#### 6.2.4 readOnly 检查

```ts
if (field.fieldMeta?.readOnly) {
  throw new AppError(AppErrorCode.INVALID_REQUEST, { message: `Field ${fieldId} is read only` });
}
```

#### 6.2.5 DocumentStatus.PENDING 检查

```ts
if (envelope.status !== DocumentStatus.PENDING) {
  throw new AppError(AppErrorCode.INVALID_REQUEST, { message: `Document ${envelope.id} must be pending for signing` });
}
```

#### 6.2.6 已签署检查

```ts
if (recipient.signingStatus === SigningStatus.SIGNED || field.recipient.signingStatus === SigningStatus.SIGNED) {
  throw new AppError(AppErrorCode.INVALID_REQUEST, { message: `Recipient ${recipient.id} has already signed` });
}
```

### 6.3 extractFieldInsertionValues — 提取字段插入值

文件: [envelope-signing.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/utils/envelope-signing.ts#L33-L253)

根据 `fieldValue.type` 分别处理：

| 字段类型 | 处理逻辑 |
|---------|---------|
| `EMAIL` | `zEmail()` 校验，null → uninsert |
| `NAME` / `INITIALS` | `z.string().min(1)` 校验，null → uninsert |
| `DATE` | `value === true` 时根据 timezone + dateFormat 格式化当前日期 |
| `NUMBER` | `ZNumberFieldMeta` + `validateNumberField` 校验 |
| `TEXT` | `ZTextFieldMeta` + `validateTextField` 校验 |
| `RADIO` | `ZRadioFieldMeta` 校验索引有效性 |
| `CHECKBOX` | `ZCheckboxFieldMeta` 校验 + 可选 validationRule 长度校验 |
| `DROPDOWN` | `ZDropdownFieldMeta` + `validateDropdownField` 校验 |
| `SIGNATURE` | null/空 → uninsert；base64 图片 → 绘制签名；非 base64 → 打字签名（检查 `typedSignatureEnabled`） |

### 6.4 Uninsert 路径

当 `inserted === false` 时（用户清除字段），在事务中：
1. 将 field 的 `customText` 置空，`inserted` 置为 `false`
2. 删除该字段关联的 signature 记录
3. 记录 `DOCUMENT_FIELD_UNINSERTED` 审计日志（ASSISTANT 除外）

### 6.5 Insert 路径 — actionAuth 验证

```ts
const derivedRecipientActionAuth = await validateFieldAuth({
  documentAuthOptions: envelope.authOptions,
  recipient,
  field,
  userId: user?.id,
  authOptions,
});
```

文件: [validate-field-auth.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/document/validate-field-auth.ts#L21-L48)

**关键规则：只有 SIGNATURE 类型字段才需要 actionAuth 验证，其他字段类型直接返回 `undefined`**：

```ts
if (field.type !== FieldType.SIGNATURE) {
  return undefined;
}
```

对于 SIGNATURE 字段，调用 `isRecipientAuthorized({ type: 'ACTION', ... })`，不通过则抛出 `UNAUTHORIZED`。

### 6.6 事务写入

在 `prisma.$transaction` 中：

1. **更新 field**：`customText` + `inserted`
2. **SIGNATURE 字段**：`signature.upsert`，写入 `signatureImageAsBase64` 或 `typedSignature`
3. **审计日志**：
   - ASSISTANT 代签他人字段 → `DOCUMENT_FIELD_PREFILLED`
   - 签署自己的字段 → `DOCUMENT_FIELD_INSERTED`
   - 日志中包含 `fieldSecurity`（actionAuth 类型，如有）

---

## 七、completeDocumentWithToken — 完成签署

文件: [complete-document-with-token.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/document/complete-document-with-token.ts#L54-L495)

### 7.1 前置校验

```ts
const envelope = await prisma.envelope.findFirstOrThrow({
  where: { ...unsafeBuildEnvelopeIdQuery(id, EnvelopeType.DOCUMENT), recipients: { some: { token } } },
  include: { documentMeta: true, recipients: { where: { token } } },
});
```

- `envelope.status !== DocumentStatus.PENDING` → 抛出错误
- `recipient.signingStatus === SigningStatus.SIGNED` → 抛出错误
- `recipient.signingStatus === SigningStatus.REJECTED` → 抛出 `UNKNOWN_ERROR`
- `assertRecipientNotExpired(recipient)` → 过期则抛出错误

### 7.2 顺序签署验证

```ts
if (envelope.documentMeta?.signingOrder === DocumentSigningOrder.SEQUENTIAL) {
  const isRecipientsTurn = await getIsRecipientsTurnToSign({ token: recipient.token });
  if (!isRecipientsTurn) {
    throw new Error(`Recipient ${recipient.id} attempted to complete the document before it was their turn`);
  }
}
```

`getIsRecipientsTurnToSign` 的逻辑：对于 SEQUENTIAL 签署，检查当前 recipient 之前的所有 recipient 是否都已 `SIGNED`。对于 PARALLEL 签署，直接返回 `true`。

### 7.3 ACCESS_2FA 验证

```ts
if (derivedRecipientAccessAuth.includes(DocumentAuth.TWO_FACTOR_AUTH)) {
  if (!accessAuthOptions) {
    throw new AppError(AppErrorCode.UNAUTHORIZED, { message: 'Access authentication required' });
  }

  if (!recipient.email.trim()) {
    throw new AppError(AppErrorCode.INVALID_REQUEST, { message: 'Recipient requires an email' });
  }

  const isValid = await isRecipientAuthorized({
    type: 'ACCESS_2FA',
    documentAuthOptions: envelope.authOptions,
    recipient,
    userId,
    authOptions: accessAuthOptions,
  });

  if (!isValid) {
    // 审计日志: DOCUMENT_ACCESS_AUTH_2FA_FAILED
    throw new AppError(AppErrorCode.TWO_FACTOR_AUTH_FAILED, { message: 'Invalid 2FA authentication' });
  }

  // 审计日志: DOCUMENT_ACCESS_AUTH_2FA_VALIDATED
}
```

`ACCESS_2FA` 的特殊之处在于：
- 在 `ACCESS` 阶段（loader），2FA 被设计为"允许通过"（`return true`）
- 真正的 2FA 验证推迟到 `completeDocumentWithToken` 时以 `ACCESS_2FA` 类型执行
- 支持 `email` 方法（`validateTwoFactorTokenFromEmail`）和 `authenticator` 方法（`verifyTwoFactorAuthenticationToken` TOTP）

### 7.4 日期字段自动插入

```ts
const uninsertedDateFields = fields.filter((field) => field.type === FieldType.DATE && !field.inserted);

if (envelope.internalVersion === 2 && uninsertedDateFields.length > 0) {
  const formattedDate = DateTime.now()
    .setZone(envelope.documentMeta?.timezone ?? DEFAULT_DOCUMENT_TIME_ZONE)
    .toFormat(envelope.documentMeta?.dateFormat ?? DEFAULT_DOCUMENT_DATE_FORMAT);

  await prisma.field.updateMany({
    where: { id: { in: uninsertedDateFields.map((f) => f.id) } },
    data: { customText: formattedDate, inserted: true },
  });

  // 审计日志: DOCUMENT_FIELD_INSERTED (每个日期字段一条)
  // 更新本地 fields 数组以通过后续校验
}
```

**这是 V2 特有行为**：完成签署时，所有未插入的 DATE 字段会自动使用当前日期（基于信封配置的时区和格式）填充。

### 7.5 recipientOverride — 签署者信息覆盖

```ts
let recipientName = recipient.name;
let recipientEmail = recipient.email;

if (!recipientName) {
  recipientName = (
    recipientOverride?.name ||
    fields.find((field) => field.type === FieldType.NAME)?.customText ||
    ''
  ).trim();
}

if (!recipient.email) {
  recipientEmail = (
    recipientOverride?.email ||
    fields.find((field) => field.type === FieldType.EMAIL)?.customText ||
    ''
  ).trim().toLowerCase();
}

if (!recipientEmail) {
  throw new AppError(AppErrorCode.INVALID_BODY, { message: 'Recipient email is required' });
}
```

- 如果 recipient 没有设置 name，优先使用 `recipientOverride.name`，其次使用 NAME 类型字段的值
- 如果 recipient 没有设置 email，优先使用 `recipientOverride.email`，其次使用 EMAIL 类型字段的值
- email 是必须的，否则抛出错误

### 7.6 fieldsContainUnsignedRequiredField 检查

文件: [advanced-fields-helpers.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/utils/advanced-fields-helpers.ts#L18-L49)

```ts
if (fieldsContainUnsignedRequiredField(fields)) {
  throw new Error(`Recipient ${recipient.id} has unsigned fields`);
}
```

字段是否"必填"的判定：
- **非高级字段类型**（SIGNATURE、DATE、EMAIL、NAME、INITIALS、FREE_SIGNATURE）→ **始终必填**
- **高级字段类型**（NUMBER、TEXT、DROPDOWN、RADIO、CHECKBOX）→ 解析 `fieldMeta.required`，无 fieldMeta 或解析失败则视为**非必填**

### 7.7 DOCUMENT_RECIPIENT_COMPLETED 审计日志与状态更新

在 `prisma.$transaction` 中：

1. **更新 recipient**：`signingStatus = SIGNED`，`signedAt = new Date()`，`name`/`email` 使用最终值
2. **如果 name 或 email 发生变更** → 记录 `RECIPIENT_UPDATED` 审计日志（包含 diff）
3. **记录 `DOCUMENT_RECIPIENT_COMPLETED` 审计日志**，包含 `actionAuth` 信息

### 7.8 DOCUMENT_RECIPIENT_COMPLETED Webhook

```ts
await triggerWebhook({
  event: WebhookTriggerEvents.DOCUMENT_RECIPIENT_COMPLETED,
  data: ZWebhookDocumentSchema.parse(mapEnvelopeToWebhookDocumentPayload(envelopeWithRelations)),
  userId: envelope.userId,
  teamId: envelope.teamId,
});
```

在事务完成后，立即触发 `DOCUMENT_RECIPIENT_COMPLETED` webhook。

### 7.9 send.recipient.signed.email Job

```ts
await jobs.triggerJob({
  name: 'send.recipient.signed.email',
  payload: {
    documentId: legacyDocumentId,
    recipientId: recipient.id,
  },
});
```

发送"签署完成确认邮件"给签署者。

### 7.10 后续签署者处理

```ts
const pendingRecipients = await prisma.recipient.findMany({
  where: {
    envelopeId: envelope.id,
    signingStatus: { not: SigningStatus.SIGNED },
    role: { not: RecipientRole.CC },
  },
  orderBy: [{ signingOrder: { sort: 'asc', nulls: 'last' } }, { id: 'asc' }],
});
```

**仍有未签署的 recipient 时**：

1. 调用 `sendPendingEmail` 发送通知
2. **SEQUENTIAL 模式下**：
   - 取排序后的第一个 pending recipient 作为下一个签署者
   - 如果 `allowDictateNextSigner` 且提供了 `nextSigner`，更新下一位签署者的 name/email 并记录审计日志
   - 更新下一位 recipient 的 `sendStatus = SENT`、`sentAt = new Date()`
   - 触发 `send.signing.requested.email` Job 发送签署邀请邮件

### 7.11 全部签署完成 → DOCUMENT_SIGNED Webhook + seal-document Job

```ts
const haveAllRecipientsSigned = await prisma.envelope.findFirst({
  where: {
    id: envelope.id,
    recipients: {
      every: { OR: [{ signingStatus: SigningStatus.SIGNED }, { role: RecipientRole.CC }] },
    },
  },
});

if (haveAllRecipientsSigned) {
  await jobs.triggerJob({
    name: 'internal.seal-document',
    payload: { documentId: legacyDocumentId, requestMetadata },
  });
}
```

当**所有非 CC 的 recipient 都已签署**时：

1. 触发 `internal.seal-document` Job — 对文档进行密封（seal），生成最终签署后的 PDF
2. 触发 `DOCUMENT_SIGNED` webhook — 通知文档已全部签署完成

---

## 八、完整流程时序图

```
签署者                     前端 (Remix)                    后端 (Loader/TRPC)                         数据库 / Jobs
  │                            │                                │                                      │
  │  GET /sign/:token          │                                │                                      │
  │───────────────────────────►│                                │                                      │
  │                            │  loader()                      │                                      │
  │                            │  查询 recipient.envelope       │                                      │
  │                            │────────────────────────────────►│  prisma.recipient.findFirst          │
  │                            │                                │─────────────────────────────────────►│
  │                            │                                │◄─────────────────────────────────────│
  │                            │  internalVersion === 2         │                                      │
  │                            │                                │                                      │
  │                            │  handleV2Loader()              │                                      │
  │                            │────────────────────────────────►│  getEnvelopeForRecipientSigning()    │
  │                            │                                │─────────────────────────────────────►│
  │                            │                                │  isRecipientAuthorized(ACCESS)        │
  │                            │                                │─────────────────────────────────────►│
  │                            │                                │◄─────────────────────────────────────│
  │                            │                                │                                      │
  │                            │  [UNAUTHORIZED]                │                                      │
  │                            │◄────────────────────────────────│  getEnvelopeRequiredAccessData()     │
  │                            │                                │─────────────────────────────────────►│
  │  显示认证页面               │                                │                                      │
  │◄───────────────────────────│                                │                                      │
  │                            │                                │                                      │
  │  (签署者完成认证)           │                                │                                      │
  │                            │                                │                                      │
  │  [isRecipientsTurn=false]  │                                │                                      │
  │  redirect /waiting         │                                │                                      │
  │                            │                                │                                      │
  │  [isRejected]              │                                │                                      │
  │  redirect /rejected        │                                │                                      │
  │                            │                                │                                      │
  │  [isCompleted]             │                                │                                      │
  │  redirect /complete        │                                │                                      │
  │                            │                                │                                      │
  │  [isExpired]               │                                │                                      │
  │  redirect /expired         │                                │                                      │
  │                            │                                │                                      │
  │  [全部通过]                 │                                │                                      │
  │                            │  viewedDocument()              │                                      │
  │                            │────────────────────────────────►│  审计日志 + webhook                   │
  │                            │                                │                                      │
  │  渲染签署页面               │                                │                                      │
  │◄───────────────────────────│                                │                                      │
  │                            │                                │                                      │
  │  插入签名字段               │                                │                                      │
  │───────────────────────────►│  signEnvelopeFieldRoute        │                                      │
  │                            │────────────────────────────────►│  校验: ASSISTANT/SIGNATURE            │
  │                            │                                │  校验: fieldValue.type === field.type │
  │                            │                                │  校验: readOnly                       │
  │                            │                                │  校验: DocumentStatus.PENDING         │
  │                            │                                │  校验: signingStatus !== SIGNED       │
  │                            │                                │  extractFieldInsertionValues()        │
  │                            │                                │  validateFieldAuth(ACTION)            │
  │                            │                                │  isRecipientAuthorized(ACTION)        │
  │                            │                                │  ──── 事务开始 ────                   │
  │                            │                                │  field.update + signature.upsert      │
  │                            │                                │  审计日志                             │
  │                            │                                │  ──── 事务结束 ────                   │
  │                            │◄────────────────────────────────│                                      │
  │  (可继续插入其他字段)        │                                │                                      │
  │                            │                                │                                      │
  │  点击"完成签署"             │                                │                                      │
  │───────────────────────────►│  completeDocumentWithToken     │                                      │
  │                            │────────────────────────────────►│  校验: PENDING / 未过期 / 未签署     │
  │                            │                                │  校验: 顺序签署 isRecipientsTurn      │
  │                            │                                │  校验: ACCESS_2FA                     │
  │                            │                                │  自动插入 DATE 字段 (V2)              │
  │                            │                                │  recipientOverride 处理               │
  │                            │                                │  fieldsContainUnsignedRequiredField   │
  │                            │                                │  ──── 事务开始 ────                   │
  │                            │                                │  recipient.update (SIGNED)            │
  │                            │                                │  RECIPIENT_UPDATED 审计 (如有变更)    │
  │                            │                                │  DOCUMENT_RECIPIENT_COMPLETED 审计    │
  │                            │                                │  ──── 事务结束 ────                   │
  │                            │                                │                                      │
  │                            │                                │  webhook: DOCUMENT_RECIPIENT_COMPLETED│
  │                            │                                │  job: send.recipient.signed.email     │
  │                            │                                │                                      │
  │                            │                                │  [仍有 pending recipients]            │
  │                            │                                │  sendPendingEmail                     │
  │                            │                                │  SEQUENTIAL: 发送下一签署者邀请       │
  │                            │                                │  job: send.signing.requested.email    │
  │                            │                                │                                      │
  │                            │                                │  [全部签署完成]                       │
  │                            │                                │  job: internal.seal-document          │
  │                            │                                │  webhook: DOCUMENT_SIGNED             │
  │                            │◄────────────────────────────────│                                      │
  │  显示完成页面               │                                │                                      │
  │◄───────────────────────────│                                │                                      │
```

---

## 九、关键设计要点总结

### 9.1 双层认证架构

- **Access Auth（访问认证）**：控制签署者能否打开文档，在 loader 阶段验证
- **Action Auth（操作认证）**：控制签署者能否签署字段（仅 SIGNATURE 类型），在 `signEnvelopeFieldRoute` 中验证

### 9.2 2FA 两阶段验证

- `ACCESS` 阶段对 `TWO_FACTOR_AUTH` 返回 `true`（放行）
- 实际 2FA 验证推迟到 `completeDocumentWithToken` 中的 `ACCESS_2FA` 阶段
- 这样签署者可以先查看文档，在完成签署时才需要输入 2FA 验证码

### 9.3 ASSISTANT 角色限制

- ASSISTANT 可以代签 `signingOrder >= 自己` 的 recipient 的非 SIGNATURE 字段
- ASSISTANT **绝对不能**签署 SIGNATURE 类型字段（即使是自己之前签署顺序的 recipient）
- ASSISTANT 代签的审计日志类型为 `DOCUMENT_FIELD_PREFILLED` 而非 `DOCUMENT_FIELD_INSERTED`

### 9.4 V2 日期字段自动插入

完成签署时，V2 信封中所有未插入的 DATE 字段会自动填充当前日期。这确保了签署日期字段的完整性和一致性，签署者无需手动插入日期。

### 9.5 顺序签署的严格保证

在三个层面保障顺序签署：
1. **Loader 层**：`isRecipientsTurn` 判断，非轮到时重定向到 `/waiting`
2. **signEnvelopeField 层**：field 查询条件中 ASSISTANT 的 `signingOrder >=` 限制
3. **completeDocument 层**：`getIsRecipientsTurnToSign` 再次验证，防止并发竞争

### 9.6 签署完成后的级联动作

```
recipient.update(SIGNED)
    │
    ├── webhook: DOCUMENT_RECIPIENT_COMPLETED
    ├── job: send.recipient.signed.email
    │
    ├── [有后续签署者]
    │   ├── sendPendingEmail
    │   └── SEQUENTIAL: job send.signing.requested.email + 更新下一签署者
    │
    └── [全部完成]
        ├── job: internal.seal-document (密封文档)
        └── webhook: DOCUMENT_SIGNED
```

`DOCUMENT_SIGNED` webhook 和 `internal.seal-document` job 仅在**所有非 CC recipient 都已签署**后才触发。
