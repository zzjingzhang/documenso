# Envelope.distribute 与 Document.distribute 复用逻辑分析

## 一、总体架构对比

### 1.1 代码文件位置

| 功能模块 | 文件路径 |
|---------|---------|
| envelope.distribute | [distribute-envelope.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/trpc/server/envelope-router/distribute-envelope.ts) |
| document.distribute | [distribute-document.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/trpc/server/document-router/distribute-document.ts) |
| 核心复用逻辑 sendDocument | [send-document.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/document/send-document.ts) |

### 1.2 复用关系图

```
distributeEnvelopeRoute ──┐
                           ├─► updateDocumentMeta ──┐
distributeDocumentRoute ──┘                        │
                                                    ├─► sendDocument
                                                    │
                          meta参数差异化处理 ◄──────┘
```

## 二、核心复用机制：sendDocument

### 2.1 入口参数设计

[sendDocument](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/document/send-document.ts#L45-L51) 通过 `EnvelopeIdOptions` 实现兼容映射：

```typescript
export type SendDocumentOptions = {
  id: EnvelopeIdOptions;        // 关键：支持两种ID类型
  userId: number;
  teamId: number;
  sendEmail?: boolean;
  requestMetadata: ApiRequestMetadata;
};
```

**EnvelopeIdOptions 类型定义**：
```typescript
// 支持 envelopeId 或 documentId 两种标识方式
{ type: 'envelopeId', id: string } | { type: 'documentId', id: number }
```

### 2.2 两个 distribute 路由的差异化处理

| 处理环节 | distributeEnvelope (L16-70) | distributeDocument (L16-58) |
|---------|----------------------------|-----------------------------|
| **ID类型** | `type: 'envelopeId'` | `type: 'documentId'` |
| **返回映射** | 直接返回 envelope + formatSigningLink | 通过 `mapEnvelopeToDocumentLite` 转换 |
| **meta传递** | 直接透传 | 直接透传 |

**关键代码对比**：

```typescript
// distribute-envelope.ts L48-56
const envelope = await sendDocument({
  id: { type: 'envelopeId', id: envelopeId },  // 使用 envelopeId
  userId: ctx.user.id,
  teamId,
  requestMetadata: ctx.metadata,
});

// distribute-document.ts L48-56
const envelope = await sendDocument({
  id: { type: 'documentId', id: documentId },  // 使用 documentId
  userId: ctx.user.id,
  teamId,
  requestMetadata: ctx.metadata,
});
```

## 三、Meta 更新流程分析

### 3.1 updateDocumentMeta 调用时机

[updateDocumentMeta](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/document-meta/upsert-document-meta.ts#L35-L136) 在 **sendDocument 之前** 被调用：

```
调用顺序：
1. updateDocumentMeta (meta 非空时)
2. sendDocument
```

**代码位置**：
- distribute-envelope.ts [L26-46](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/trpc/server/envelope-router/distribute-envelope.ts#L26-L46)
- distribute-document.ts [L26-46](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/trpc/server/document-router/distribute-document.ts#L26-L46)

### 3.2 Meta 支持的字段

| 字段 | 说明 |
|------|------|
| subject | 邮件主题 |
| message | 邮件正文 |
| dateFormat | 日期格式 |
| timezone | 时区 |
| redirectUrl | 签署后跳转URL |
| distributionMethod | 分发方式 (EMAIL/NONE) |
| emailSettings | 邮件设置 |
| language | 语言 |
| emailId | 发件邮箱ID |
| emailReplyTo | 回复邮箱 |

## 四、状态流转与核心枚举

### 4.1 DocumentStatus 状态流转

[schema.prisma L373-378](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/prisma/schema.prisma#L373-L378)

```
DRAFT (草稿)
    │
    ▼  sendDocument [send-document.ts L294-300]
PENDING (签署中)
    │
    ├─► 全部签署完成 ──► COMPLETED
    │
    └─► 有人拒绝 ──────► REJECTED
```

**关键代码** [send-document.ts L294-300](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/document/send-document.ts#L294-L300)：
```typescript
return await tx.envelope.update({
  where: { id: envelope.id },
  data: { status: DocumentStatus.PENDING },  // DRAFT → PENDING
  include: { documentMeta: true, recipients: true },
});
```

### 4.2 DocumentSigningOrder 签署顺序

[schema.prisma L531-534](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/prisma/schema.prisma#L531-L534)

| 模式 | 行为 | 关键代码 |
|------|------|---------|
| **PARALLEL** (并行) | 所有收件人同时收到通知 | 默认值 |
| **SEQUENTIAL** (顺序) | 仅第一个未签收人收到通知 | [send-document.ts L131-136](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/document/send-document.ts#L131-L136) |

**顺序签署核心逻辑**：
```typescript
if (signingOrder === DocumentSigningOrder.SEQUENTIAL) {
  recipientsToNotify = envelope.recipients
    .filter((r) => r.signingStatus === SigningStatus.NOT_SIGNED 
                   && r.role !== RecipientRole.CC)
    .slice(0, 1);  // 只取第一个
}
```

### 4.3 RecipientRole 角色处理

[schema.prisma L612-618](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/prisma/schema.prisma#L612-L618)

| 角色 | 邮件通知 | 过期处理 | 需要签名字段 |
|------|---------|---------|------------|
| **CC** (抄送) | ❌ 不发送 [L318](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/document/send-document.ts#L318) | ❌ 排除 [L284](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/document/send-document.ts#L284) | ❌ |
| **SIGNER** (签署者) | ✅ 发送 | ✅ 包含 | ✅ 需要 |
| **VIEWER** (查看者) | ✅ 发送 | ✅ 包含 | ❌ |
| **APPROVER** (审批者) | ✅ 发送 | ✅ 包含 | ❌ |
| **ASSISTANT** (助理) | ✅ 发送 | ✅ 包含 | ❌ |

### 4.4 SendStatus 发送状态

[schema.prisma L601-604](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/prisma/schema.prisma#L601-L604)

```
NOT_SENT (未发送)
    │
    ▼  邮件job触发
SENT (已发送)
```

**关键判断** [send-document.ts L318](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/document/send-document.ts#L318)：
```typescript
if (recipient.sendStatus === SendStatus.SENT || recipient.role === RecipientRole.CC) {
  return;  // 已发送或抄送者跳过
}
```

### 4.5 SigningStatus 签署状态

[schema.prisma L606-610](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/prisma/schema.prisma#L606-L610)

```
NOT_SIGNED (未签署)
    ├─► 签署完成 ──► SIGNED
    └─► 拒绝签署 ──► REJECTED
```

## 五、执行顺序详细分析

### 5.1 sendDocument 完整执行链路

```
1. 前置校验
   ├─ assertUserNotDisabledById [L57]
   ├─ getEnvelopeWhereInput + 查询信封 [L59-101]
   ├─ 检查收件人数量 [L107-109]
   ├─ 检查收件人上限 [L114-119]
   └─ 检查文档是否已完成 [L121-123]

2. 收件人通知列表确定
   └─ 根据 signingOrder 确定 recipientsToNotify [L127-136]
      ├─ PARALLEL: 全部收件人
      └─ SEQUENTIAL: 第一个 NOT_SIGNED 且非 CC 的收件人

3. 表单值注入 (如有)
   └─ injectFormValuesIntoDocument [L142-148]

4. 收件人校验
   ├─ Auth 收件人邮箱校验 [L151-166]
   │  └─ extractDocumentAuthMethods + isRecipientEmailValidForSending
   └─ 必填字段校验 [L169-179]
      └─ getRecipientsWithMissingFields [recipients.ts L23-38]

5. 特殊情况：全部无需操作
   └─ 触发 internal.seal-document job [L181-204]

6. 字段自动填充 (V2信封)
   └─ extractFieldAutoInsertValues [L206-226, L384-496]
      ├─ EMAIL: 收件人邮箱
      ├─ TEXT: 预设文本
      ├─ NUMBER: 预设数值
      ├─ RADIO: 预选项
      ├─ DROPDOWN: 默认值
      └─ CHECKBOX: 预选项

7. 数据库事务 (关键) [L228-306]
   ├─ DOCUMENT_SENT 审计日志 [L229-238] (仅 DRAFT 状态时)
   ├─ DOCUMENT_FIELDS_AUTO_INSERTED 审计日志 [L240-270]
   ├─ envelopeExpirationPeriod 过期时间设置 [L272-292]
   │  └─ resolveExpiresAt + 更新 recipient.expiresAt
   │     └─ 排除: SIGNED/REJECTED 状态, CC 角色
   └─ 更新信封状态为 PENDING [L294-305]

8. 发送邮件 (如启用) [L308-333]
   └─ 对 recipientsToNotify 中每个收件人
      ├─ 跳过: sendStatus=SENT 或 role=CC
      └─ 触发 send.signing.requested.email job

9. Webhook 触发
   └─ DOCUMENT_SENT webhook [L335-340]
      └─ triggerWebhook + mapEnvelopeToWebhookDocumentPayload
```

### 5.2 关键节点时间线

```
时序 ───────────────────────────────────────────────────────────►
     │          │          │           │           │           │
  Meta更新   事务开始    字段填充    状态PENDING   邮件发送   Webhook
(updateDoc  │          │           │           │           │
  Meta)   DOCUMENT_   DOCUMENT_   expiresAt   send.sign.  DOCUMENT_
          SENT审计    FIELDS_     设置        requested.  SENT
                      AUTO_                   email job
                      INSERTED
```

## 六、核心辅助函数分析

### 6.1 getIsRecipientsTurnToSign

[get-is-recipient-turn.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/recipient/get-is-recipient-turn.ts)

**功能**：判断当前收件人是否轮到签署

```typescript
// 非顺序签署：总是返回 true
if (envelope.documentMeta?.signingOrder !== SEQUENTIAL) {
  return true;
}

// 顺序签署：检查前面所有收件人是否都已 SIGNED
for (let i = 0; i < currentRecipientIndex; i++) {
  if (recipients[i].signingStatus !== SigningStatus.SIGNED) {
    return false;
  }
}
return true;
```

### 6.2 getNextPendingRecipient

[get-next-pending-recipient.ts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/recipient/get-next-pending-recipient.ts)

**功能**：获取当前收件人的下一个待签署人（用于顺序签署流转）

```typescript
// 按 signingOrder 升序排列
const currentIndex = recipients.findIndex((r) => r.id === currentRecipientId);
if (currentIndex === -1 || currentIndex === recipients.length - 1) {
  return null;  // 已是最后一个
}
return recipients[currentIndex + 1];
```

### 6.3 extractFieldAutoInsertValues

[send-document.ts L384-496](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/server-only/document/send-document.ts#L384-L496)

**支持自动填充的字段类型**：

| 字段类型 | 填充值来源 | 条件 |
|---------|-----------|------|
| EMAIL | recipient.email | 邮箱格式有效且非模板收件人 |
| TEXT | fieldMeta.text | text 非空 |
| NUMBER | fieldMeta.value | value 非空 |
| RADIO | 第一个选中项的索引 | 有项被选中 |
| DROPDOWN | fieldMeta.defaultValue | defaultValue 存在于 values |
| CHECKBOX | 所有选中项索引 | 选中项通过校验规则 |

### 6.4 getRecipientsWithMissingFields

[recipients.ts L23-38](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/lib/utils/recipients.ts#L23-L38)

**功能**：找出缺少必填字段的收件人

```typescript
// 仅 SIGNER 角色需要检查
if (recipient.role === RecipientRole.SIGNER) {
  const hasSignatureField = fields.some(
    (field) => field.recipientId === recipient.id 
               && isSignatureFieldType(field.type)
  );
  return !hasSignatureField;  // 没有签名字段则缺失
}
```

## 七、数据库模型关键定义

### 7.1 Envelope 模型

[schema.prisma L426-485](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/prisma/schema.prisma#L426-L485)

```prisma
model Envelope {
  id              String         @id
  secondaryId     String         @unique  // 用于兼容旧 documentId
  type            EnvelopeType   // DOCUMENT / TEMPLATE
  status          DocumentStatus @default(DRAFT)
  internalVersion Int            // V1 / V2
  recipients      Recipient[]
  fields          Field[]
  documentMeta    DocumentMeta   @relation(...)
}
```

### 7.2 DocumentMeta 模型

[schema.prisma L550-576](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/prisma/schema.prisma#L550-L576)

```prisma
model DocumentMeta {
  id                       String               @id
  signingOrder             DocumentSigningOrder @default(PARALLEL)
  envelopeExpirationPeriod Json?                // 过期周期设置
  emailSettings            Json?                // 邮件开关设置
  distributionMethod       DocumentDistributionMethod @default(EMAIL)
}
```

### 7.3 Recipient 模型

[schema.prisma L621-657](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/documenso/packages/prisma/schema.prisma#L621-L657)

```prisma
model Recipient {
  id              Int            @id
  envelopeId      String
  email           String
  token           String
  role            RecipientRole  @default(SIGNER)
  signingOrder    Int?           // 顺序签署序号
  signingStatus   SigningStatus  @default(NOT_SIGNED)
  sendStatus      SendStatus     @default(NOT_SENT)
  expiresAt       DateTime?      // 过期时间
  sentAt          DateTime?
  signedAt        DateTime?
}
```

## 八、顺序签署回归测试用例设计

### 8.1 测试目标

验证 **顺序签署模式下只通知第一个未签收件人** 的核心逻辑。

### 8.2 测试用例集

#### 用例 1：首次分发 - 只通知第一个收件人

**前置条件**：
- 文档状态：DRAFT
- 签署顺序：SEQUENTIAL
- 收件人列表：
  - R1 (SIGNER, signingOrder=1, NOT_SIGNED)
  - R2 (SIGNER, signingOrder=2, NOT_SIGNED)
  - R3 (SIGNER, signingOrder=3, NOT_SIGNED)
  - R4 (CC, signingOrder=4, NOT_SIGNED)

**执行步骤**：
1. 调用 distributeDocument / distributeEnvelope
2. 检查邮件 job 触发情况

**预期结果**：
- ✅ 仅 R1 触发 `send.signing.requested.email` job
- ❌ R2、R3、R4 不触发邮件
- ✅ 文档状态变为 PENDING
- ✅ R1、R2、R3 的 expiresAt 被设置（R4 作为 CC 排除）

---

#### 用例 2：第一个签署完成后重新分发 - 通知第二个收件人

**前置条件**：
- 文档状态：PENDING
- 签署顺序：SEQUENTIAL
- 收件人状态：
  - R1: SIGNED, sendStatus=SENT
  - R2: NOT_SIGNED, sendStatus=NOT_SENT
  - R3: NOT_SIGNED, sendStatus=NOT_SENT

**执行步骤**：
1. 调用 sendDocument (模拟重新分发)
2. 检查邮件 job 触发情况

**预期结果**：
- ✅ 仅 R2 触发邮件 job
- ❌ R1（已签署）、R3（未轮到）不触发邮件

---

#### 用例 3：混合角色场景 - 跳过 CC 角色

**前置条件**：
- 签署顺序：SEQUENTIAL
- 收件人列表（按 signingOrder 排序）：
  - R1: CC, NOT_SIGNED
  - R2: SIGNER, NOT_SIGNED
  - R3: SIGNER, NOT_SIGNED

**执行步骤**：
1. 首次分发文档

**预期结果**：
- ✅ 仅 R2 触发邮件（第一个非 CC 的 NOT_SIGNED 收件人）
- ❌ R1 作为 CC 角色被跳过

---

#### 用例 4：并行模式对比 - 所有人收到通知

**前置条件**：
- 签署顺序：PARALLEL
- 收件人：R1、R2、R3（均为 SIGNER，NOT_SIGNED）

**执行步骤**：
1. 分发文档

**预期结果**：
- ✅ R1、R2、R3 均触发邮件 job
- ❌ CC 角色仍被跳过

---

#### 用例 5：字段自动填充 - 仅未发送收件人生效

**前置条件**：
- 签署顺序：SEQUENTIAL
- internalVersion: 2
- R1: sendStatus=SENT, 有 EMAIL 字段
- R2: sendStatus=NOT_SENT, 有 EMAIL 字段

**执行步骤**：
1. 重新分发文档

**预期结果**：
- ✅ R2 的 EMAIL 字段被自动填充
- ❌ R1 的字段不更新（已发送过）
- ✅ DOCUMENT_FIELDS_AUTO_INSERTED 审计日志只包含 R2 的字段

---

#### 用例 6：收件人轮次校验 - getIsRecipientsTurnToSign

**前置条件**：
- 签署顺序：SEQUENTIAL
- 收件人：R1 (order 1), R2 (order 2), R3 (order 3)

**执行步骤**：
1. R1 未签署时，R2 访问签署页
2. R1 签署完成后，R2 访问签署页
3. R1、R2 签署完成后，R3 访问签署页

**预期结果**：
- ❌ 步骤 1：R2 返回 false（未轮到）
- ✅ 步骤 2：R2 返回 true（轮到）
- ✅ 步骤 3：R3 返回 true（轮到）

---

#### 用例 7：最后一个收件人签署 - 无下一个待签署人

**前置条件**：
- 签署顺序：SEQUENTIAL
- 共 3 个收件人
- 当前签署人：R3 (order 3)

**执行步骤**：
1. 调用 `getNextPendingRecipient(documentId, R3.id)`

**预期结果**：
- ✅ 返回 `null`（无下一个收件人）

---

#### 用例 8：过期时间设置 - 排除特定条件

**前置条件**：
- envelopeExpirationPeriod: 7 天
- 收件人状态：
  - R1: NOT_SIGNED, SIGNER
  - R2: SIGNED, SIGNER
  - R3: REJECTED, SIGNER
  - R4: NOT_SIGNED, CC

**执行步骤**：
1. 分发文档

**预期结果**：
- ✅ R1: expiresAt 被设置
- ❌ R2: 已签署，不设置
- ❌ R3: 已拒绝，不设置
- ❌ R4: CC 角色，不设置

## 九、关键代码定位索引

| 功能点 | 文件位置 | 行号 |
|-------|---------|------|
| 顺序签署收件人过滤 | send-document.ts | L131-136 |
| 邮件发送跳过逻辑 | send-document.ts | L318 |
| CC 角色过期排除 | send-document.ts | L284 |
| 状态 DRAFT→PENDING | send-document.ts | L299 |
| DOCUMENT_SENT 审计 | send-document.ts | L230-238 |
| DOCUMENT_FIELDS_AUTO_INSERTED | send-document.ts | L256-269 |
| DOCUMENT_SENT webhook | send-document.ts | L335-340 |
| send.signing.requested.email | send-document.ts | L322-331 |
| 字段自动填充函数 | send-document.ts | L384-496 |
| 收件人邮箱校验 | send-document.ts | L151-166 |
| 缺少字段校验 | send-document.ts | L169-179 |
| 是否轮到签署 | get-is-recipient-turn.ts | L8-46 |
| 下一个待签署人 | get-next-pending-recipient.ts | L6-42 |
| mapEnvelopeToDocumentLite | document.ts | L71-96 |
| getRecipientsWithMissingFields | recipients.ts | L23-38 |
| isRecipientEmailValidForSending | recipients.ts | L111-113 |
| updateDocumentMeta | upsert-document-meta.ts | L35-136 |
