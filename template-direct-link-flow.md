# 模板直链功能完整流程分析

## 概述

本文档详细分析了 Documenso 中模板生成直链后，匿名或已登录用户通过 `directTemplateToken` 提交 `signedFieldValues` 并创建 Document 的全过程。

## 一、创建模板直链

### 1.1 DIRECT_TEMPLATE_RECIPIENT_EMAIL 替换或创建

在 [create-template-direct-link.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/template/create-template-direct-link.ts) 中：

- **常量定义**：`DIRECT_TEMPLATE_RECIPIENT_EMAIL` 是一个特殊的占位符邮箱
- **两种模式**：
  1. **指定收件人模式**（提供 `directRecipientId`）：
     - 更新现有收件人的 name 和 email 为 `DIRECT_TEMPLATE_RECIPIENT_NAME` 和 `DIRECT_TEMPLATE_RECIPIENT_EMAIL`
  2. **自动创建模式**（不提供 `directRecipientId`）：
     - 创建一个新的收件人，使用占位符 name 和 email

**关键代码** (第62-85行)：
```typescript
const createdDirectLink = await prisma.$transaction(async (tx) => {
  let recipient: Recipient | undefined;

  if (directRecipientId) {
    recipient = await tx.recipient.update({
      where: {
        envelopeId: envelope.id,
        id: directRecipientId,
      },
      data: {
        name: DIRECT_TEMPLATE_RECIPIENT_NAME,
        email: DIRECT_TEMPLATE_RECIPIENT_EMAIL,
      },
    });
  } else {
    recipient = await tx.recipient.create({
      data: {
        envelopeId: envelope.id,
        name: DIRECT_TEMPLATE_RECIPIENT_NAME,
        email: DIRECT_TEMPLATE_RECIPIENT_EMAIL,
        token: nanoid(),
      },
    });
  }
  // ...
});
```

### 1.2 directTemplateRecipientId

在 `TemplateDirectLink` 表中存储 `directTemplateRecipientId`，用于标识哪个收件人是"直签收件人"。

**数据库模型** ([schema.prisma](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/prisma/schema.prisma#L1015-L1025))：
```prisma
model TemplateDirectLink {
  id         String   @id @unique @default(cuid())
  envelopeId String   @unique
  token      String   @unique
  createdAt  DateTime @default(now())
  enabled    Boolean

  directTemplateRecipientId Int  // 关键字段

  envelope Envelope @relation(fields: [envelopeId], references: [id], onDelete: Cascade)
}
```

## 二、通过直链创建文档

### 2.1 入口点

在 [router.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/trpc/server/template-router/router.ts#L633-L679) 中，`createDocumentFromDirectTemplate` 端点：
- 使用 `maybeAuthenticatedProcedure`：支持匿名或已登录用户
- 接收参数：`directTemplateToken`, `signedFieldValues`, `templateUpdatedAt`, `nextSigner` 等

### 2.2 templateUpdatedAt 防旧版本机制

**目的**：防止用户使用缓存的旧版本模板进行签名

**验证逻辑** ([create-document-from-direct-template.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/template/create-document-from-direct-template.ts#L173-L175))：
```typescript
if (directTemplateEnvelope.updatedAt.getTime() !== templateUpdatedAt.getTime()) {
  throw new AppError(AppErrorCode.INVALID_REQUEST, { message: 'Template no longer matches' });
}
```

### 2.3 allowDictateNextSigner 与 SEQUENTIAL 的组合限制

**规则**：只有同时满足以下两个条件时，才允许指定下一个签名者：
1. `allowDictateNextSigner` 为 `true`
2. `signingOrder` 为 `DocumentSigningOrder.SEQUENTIAL`

**验证逻辑** (第140-148行)：
```typescript
if (
  nextSigner &&
  (!directTemplateEnvelope.documentMeta?.allowDictateNextSigner ||
    directTemplateEnvelope.documentMeta?.signingOrder !== DocumentSigningOrder.SEQUENTIAL)
) {
  throw new AppError(AppErrorCode.INVALID_REQUEST, {
    message: 'You need to enable allowDictateNextSigner and sequential signing to dictate the next signer',
  });
}
```

### 2.4 ACCOUNT 和 TWO_FACTOR_AUTH 访问认证限制

**直链不支持的认证方式**：
- `DocumentAccessAuth.ACCOUNT`：要求用户登录且邮箱匹配
- `DocumentAccessAuth.TWO_FACTOR_AUTH`：直链场景完全不支持

**验证逻辑** (第184-192行)：
```typescript
const isAccessAuthValid = match(derivedRecipientAccessAuth.at(0))
  .with(DocumentAccessAuth.ACCOUNT, () => user && user?.email === directRecipientEmail)
  .with(DocumentAccessAuth.TWO_FACTOR_AUTH, () => false) // 不支持直链
  .with(undefined, () => true)
  .exhaustive();

if (!isAccessAuthValid) {
  throw new AppError(AppErrorCode.UNAUTHORIZED, { message: 'You must be logged in' });
}
```

### 2.5 recipientCount 限制

**目的**：根据组织订阅计划限制收件人数量

**验证逻辑** (第204-212行)：
```typescript
const maximumRecipientCount = directTemplateEnvelope.team.organisation.organisationClaim.recipientCount;
const resultingRecipientCount = nonDirectTemplateRecipients.length + 1;

if (maximumRecipientCount > 0 && resultingRecipientCount > maximumRecipientCount) {
  throw new AppError('RECIPIENT_LIMIT_EXCEEDED', {
    message: `You cannot send a document with more than ${maximumRecipientCount} recipients`,
    statusCode: 400,
  });
}
```

> **注意**：此检查在 `sendDocument` 中也会执行，但由于直链流程异常会被吞掉，必须在此处提前验证。

### 2.6 字段必填验证与签名处理

#### 必填字段验证 (第235-239行)：
```typescript
if (isRequiredField(templateField) && !signedFieldValue) {
  throw new AppError(AppErrorCode.INVALID_BODY, {
    message: 'Invalid, missing or changed fields',
  });
}
```

#### 签名字段处理 (第266-294行)：
- **Base64 签名**：`isBase64 = true`，存储为 `signatureImageAsBase64`
- **Typed 签名**：`isBase64 = false`，存储为 `typedSignature`

```typescript
const { value, isBase64 } = signedFieldValue;

const isSignatureField =
  templateField.type === FieldType.SIGNATURE || templateField.type === FieldType.FREE_SIGNATURE;

let customText = !isSignatureField ? value : '';

const signatureImageAsBase64 = isSignatureField && isBase64 ? value : undefined;
const typedSignature = isSignatureField && !isBase64 ? value : undefined;
```

#### 字段认证验证 (第245-255行)：
调用 [validate-field-auth.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/document/validate-field-auth.ts)：
- 仅对 `SIGNATURE` 类型字段进行认证验证
- 非签名字段直接返回 `undefined`（不要求认证）

## 三、文档创建事务

### 3.1 DocumentSource.TEMPLATE_DIRECT_LINK

新创建的文档来源标记为 `TEMPLATE_DIRECT_LINK` ([create-document-from-direct-template.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/template/create-document-from-direct-template.ts#L356))：
```typescript
source: DocumentSource.TEMPLATE_DIRECT_LINK,
```

### 3.2 directRecipient 状态设置为 SIGNED

直签收件人在创建时就被标记为已签名 (第458-460行)：
```typescript
signingStatus: SigningStatus.SIGNED,
sendStatus: SendStatus.SENT,
signedAt: initialRequestTime,
```

### 3.3 非直签收件人复制

非直签收件人从模板复制到新文档，保留原有配置 (第376-393行)：
```typescript
recipients: {
  createMany: {
    data: nonDirectTemplateRecipients.map((recipient) => {
      const authOptions = ZRecipientAuthOptionsSchema.parse(recipient?.authOptions);

      return {
        email: recipient.email,
        name: recipient.name,
        role: recipient.role,
        authOptions: createRecipientAuthOptions({
          accessAuth: authOptions.accessAuth,
          actionAuth: authOptions.actionAuth,
        }),
        sendStatus: recipient.role === RecipientRole.CC ? SendStatus.SENT : SendStatus.NOT_SENT,
        signingStatus: recipient.role === RecipientRole.CC ? SigningStatus.SIGNED : SigningStatus.NOT_SIGNED,
        signingOrder: recipient.signingOrder,
        token: nanoid(),
      };
    }),
  },
},
```

### 3.4 附件复制

模板附件被复制到新文档 (第724-739行)：
```typescript
const templateAttachments = await tx.envelopeAttachment.findMany({
  where: {
    envelopeId: directTemplateEnvelope.id,
  },
});

if (templateAttachments.length > 0) {
  await tx.envelopeAttachment.createMany({
    data: templateAttachments.map((attachment) => ({
      envelopeId: createdEnvelope.id,
      type: attachment.type,
      label: attachment.label,
      data: attachment.data,
    })),
  });
}
```

## 四、审计日志

### 4.1 DOCUMENT_CREATED

记录文档创建事件，包含来源信息 (第551-568行)：
```typescript
createDocumentAuditLogData({
  type: DOCUMENT_AUDIT_LOG_TYPE.DOCUMENT_CREATED,
  envelopeId: createdEnvelope.id,
  user: {
    id: user?.id,
    name: user?.name,
    email: directRecipientEmail,
  },
  metadata: requestMetadata,
  data: {
    title: createdEnvelope.title,
    source: {
      type: DocumentSource.TEMPLATE_DIRECT_LINK,
      templateId: directTemplateEnvelopeLegacyId,
      directRecipientEmail,
    },
  },
}),
```

### 4.2 DOCUMENT_OPENED

记录文档打开事件 (第569-585行)。

### 4.3 DOCUMENT_FIELD_INSERTED

为每个已插入的字段创建审计日志 (第586-630行)。

### 4.4 DOCUMENT_RECIPIENT_COMPLETED

记录直签收件人完成签署 (第631-648行)。

## 五、后置操作

### 5.1 ownerDocumentCreated 邮件通知

如果启用了 `ownerDocumentCreated` 邮件设置，触发邮件通知 (第750-758行)：
```typescript
if (emailSettings.ownerDocumentCreated) {
  await jobs.triggerJob({
    name: 'send.document.created.from.direct.template.email',
    payload: {
      envelopeId: createdEnvelope.id,
      recipientId,
    },
  });
}
```

### 5.2 sendDocument 与 DOCUMENT_SIGNED Webhook

调用 [send-document.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/document/send-document.ts) 发送文档，成功后触发 `DOCUMENT_SIGNED` Webhook (第760-788行)：

```typescript
try {
  await sendDocument({/* ... */});

  const refetchedEnvelope = await prisma.envelope.findFirstOrThrow({/* ... */});

  await triggerWebhook({
    event: WebhookTriggerEvents.DOCUMENT_SIGNED,
    data: ZWebhookDocumentSchema.parse(mapEnvelopeToWebhookDocumentPayload(refetchedEnvelope)),
    userId: refetchedEnvelope.userId,
    teamId: refetchedEnvelope.teamId ?? undefined,
  });
} catch (err) {
  console.error('[CREATE_DOCUMENT_FROM_DIRECT_TEMPLATE]:', err);
  // 异常被吞掉，不抛出
}
```

### 5.3 sendDocument 异常被吞掉的风险含义

**风险点** (第789-794行)：
```typescript
catch (err) {
  console.error('[CREATE_DOCUMENT_FROM_DIRECT_TEMPLATE]:', err);
  // Don't launch an error since the document has already been created.
  // Log and reseal as required until we configure middleware.
}
```

**风险含义**：
1. **文档状态不一致**：文档已创建且直签收件人已标记为 SIGNED，但 `sendDocument` 失败可能导致：
   - 其他收件人未收到签名请求邮件
   - 文档未正确密封（如果是最后一个签署者）
   - Webhook 未触发

2. **用户无感知**：用户会收到成功响应，但实际后续流程可能失败

3. **需要后台补偿**：注释中提到 "Log and reseal as required"，暗示需要后台任务进行补偿

**设计权衡**：
- 由于数据库事务已提交，文档已永久创建
- 选择"静默失败"而非回滚，避免丢失用户签名数据
- 依赖日志监控和后台重试机制来解决一致性问题

## 六、完整流程图

```
用户访问直链页面
    │
    ▼
加载模板信息 (含 templateUpdatedAt)
    │
    ▼
用户填写字段并签名
    │
    ▼
调用 createDocumentFromDirectTemplate
    │
    ├─► 验证 templateUpdatedAt（防旧版本）
    ├─► 验证 allowDictateNextSigner + SEQUENTIAL
    ├─► 验证访问认证（ACCOUNT/2FA 限制）
    ├─► 验证 recipientCount 限制
    └─► 验证必填字段 + 签名格式
    │
    ▼
开始数据库事务
    │
    ├─► 创建 Envelope (source=TEMPLATE_DIRECT_LINK)
    ├─► 复制非直签收件人
    ├─► 复制非直签收件人字段
    ├─► 创建直签收件人 (signingStatus=SIGNED)
    ├─► 创建直签收件人字段（含签名）
    ├─► 创建审计日志（4种类型）
    ├─► 更新下一个签署者（如果允许）
    └─► 复制模板附件
    │
    ▼
提交事务
    │
    ▼
触发 ownerDocumentCreated 邮件
    │
    ▼
调用 sendDocument
    │
    ├─► 成功：触发 DOCUMENT_SIGNED Webhook
    └─► 失败：记录日志，静默失败
    │
    ▼
返回成功响应
```

## 七、关键文件索引

| 文件 | 描述 |
|------|------|
| [router.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/trpc/server/template-router/router.ts) | tRPC 路由定义 |
| [create-template-direct-link.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/template/create-template-direct-link.ts) | 创建模板直链 |
| [create-document-from-direct-template.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/template/create-document-from-direct-template.ts) | 通过直链创建文档 |
| [send-document.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/document/send-document.ts) | 发送文档 |
| [validate-field-auth.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/document/validate-field-auth.ts) | 字段认证验证 |
| [schema.prisma](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/prisma/schema.prisma) | 数据库模型 |
