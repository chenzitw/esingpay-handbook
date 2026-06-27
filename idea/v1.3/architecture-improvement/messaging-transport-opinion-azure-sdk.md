---
status: draft
updated_at: 2026-06-27
updated_by: Codex
---

# Messaging Transport — Azure SDK Adapter Opinion

> **定位**:本文是 [messaging-transport-proposal.md](messaging-transport-proposal.md) 的補充 opinion。proposal 已定調 cross-service event broadcast 採 Azure Service Bus Topic / Subscription；本文處理下游 service-kit runtime 的 adapter 取向：是否直接採用第三方 NestJS messaging 套件，或以官方 `@azure/service-bus` SDK 建立內部封裝。
>
> **結論先行**:建議採用 **官方 `@azure/service-bus` SDK + 內部 NestJS 封裝**，不要把第三方 NestJS messaging extension 當成 Azure Service Bus 的核心流程控制層。

## Recommendation

service-kit 的 Azure Service Bus runtime 應採用：

```txt
Application Service
  -> Internal Messaging Abstraction
  -> Azure Service Bus Adapter
  -> @azure/service-bus
```

而不是：

```txt
Application Service
  -> Third-party Messaging Abstraction
  -> Third-party Azure Extension
  -> @azure/service-bus
```

內部封裝不應只是把 SDK method 包一層，而應明確擁有以下決策：

- publish message 的統一介面。
- session-aware consume 流程。
- handler error classification。
- message settlement：`complete` / `abandon` / `deadLetter`。
- retry / DLQ policy。
- graceful shutdown。
- logging / tracing / metrics。

這些能力是 message infrastructure 的可靠性核心。若把它們壓在維護度不明或 Azure semantics 抽象不足的第三方 extension 下，後續 debug、升版與可靠性治理成本都會升高。

## Context

目前已知需求包含：

- 後端框架使用 NestJS。
- cross-service event broadcast 採 Azure Service Bus。
- publish message 時需要保留 `sessionId` 能力，以支援未來具名 FIFO / mutual exclusion 場景。
- consumer/subscriber 必須能正確處理 session-enabled entity。
- handler error 必須能映射到明確 settlement 行為。
- application layer 不應直接散落 Azure SDK 型別與 receiver 操作。
- runtime 需要有一致的 observability 與 graceful shutdown 行為。

曾評估的第三方選項包含：

- `@nestjstools/messaging`
- `@nestjstools/messaging-azure-service-bus-extension`
- `nestjs-azure-service-bus`
- `@niur/nestjs-service-bus`

判斷是：這些套件可作為 API ergonomics 或 application-level pattern 的參考，但不適合作為本案 Azure Service Bus adapter 的核心可靠性層。

## Why Not Third-party Azure Extension

`@nestjstools/messaging` 的 application-level 抽象有參考價值，例如 named bus、routing message、handler decorator、middleware、normalizer、exception listener。

但其 Azure extension 對 Azure Service Bus 的支援偏薄，尤其卡在 publish metadata、session consumer 與 settlement control。

### Publish Surface

目前 Azure publish 形狀偏向只轉出 message body 與 routing key：

```ts
const serviceBusMessage: ServiceBusMessage = {
  body: message.message,
  applicationProperties: {
    [ROUTING_KEY_ATTRIBUTE_NAME]: message.messageRoutingKey,
  },
};
```

這不足以覆蓋 service-kit 需要保留的 Azure message surface：

- `sessionId`
- `messageId`
- `correlationId`
- `replyTo`
- `timeToLive`
- `scheduledEnqueueTimeUtc`
- custom application properties merge strategy
- batch publish

只補 publish `sessionId` 的成本不高，但那只解送出端；可靠消費的主要問題仍在 consumer pipeline。

### Session Consumer

Azure Service Bus entity 若啟用 `requiresSession`，consumer 不能用一般 receiver：

```ts
client.createReceiver(queueName);
```

而應使用 session receiver：

```ts
await client.acceptSession(queueName, sessionId);
await client.acceptNextSession(queueName);
```

大多數 worker pool 情境應使用 `acceptNextSession` 接下一個可用 session。若第三方 extension 只建立一般 receiver，它就沒有真正支援 Azure Service Bus sessions consumer。

### Settlement Control

Azure Service Bus 的可靠消費流程需要把 handler 結果映射到 settlement：

- handler 成功：`completeMessage`。
- handler 發生可重試錯誤：`abandonMessage`。
- handler 發生不可重試錯誤：`deadLetterMessage`。
- delivery count 達應用層門檻：`deadLetterMessage`。

若第三方 core consumer flow 在內層吃掉 handler error，再改走 exception listener / `onError()`，Azure adapter 就不容易根據 handler 結果做精準 settlement。要補到 production-ready，可能不只是改 Azure extension，而要介入 core consumer flow；這會讓維護成本與升版風險上升。

## Azure Service Bus Semantics That Must Stay Visible

### Session Is Receiver-level Semantics

publish 時帶 `sessionId`：

```ts
await sender.sendMessages({
  body,
  sessionId,
});
```

consume 時不是「從 message 指定 sessionId」，而是取得 session receiver：

```ts
const receiver = await client.acceptSession(queueName, sessionId);
```

或接下一個可用 session：

```ts
const receiver = await client.acceptNextSession(queueName);
```

若 queue/subscription 未啟用 `requiresSession`，帶 `sessionId` 的 message 仍可被一般 `createReceiver()` 收到，但此時 `sessionId` 只是 message 屬性，不提供同 session 的順序與互斥保證。

若 queue/subscription 啟用 `requiresSession`：

- publish message 必須帶 `sessionId`。
- consume 必須使用 `acceptSession` 或 `acceptNextSession`。
- 不應使用一般 `createReceiver`。

### Non-session Receiver Can Break Causal Order

若只是把 `sessionId` 當 metadata，但仍用一般 receiver，同一 session 的 message 可能被不同 worker 並行處理：

```txt
message A1 sessionId=order-1
message A2 sessionId=order-1
message A3 sessionId=order-1

consumer-1 處理 A1 很慢
consumer-2 先處理完成 A2
consumer-2 又先處理完成 A3
consumer-1 最後才完成 A1
```

實際完成順序可能變成：

```txt
A2 -> A3 -> A1
```

若業務需要同一 order / tenant / workflow / aggregate 保序，必須使用 Azure Service Bus sessions，而不是只在 message 上放 `sessionId`。

## Recommended Adapter Shape

### Publisher Contract

application layer 應依賴 service-kit 定義的 publish request，不直接依賴 Azure SDK 型別：

```ts
export interface PublishMessage<T = unknown> {
  entityName: string;
  body: T;
  routingKey?: string;
  sessionId?: string;
  messageId?: string;
  correlationId?: string;
  applicationProperties?: Record<string, unknown>;
}
```

使用端只表達 transport-neutral intent：

```ts
await publisher.publish({
  entityName: 'fund.deposit.completed',
  routingKey: 'fund.deposit.completed',
  body: payload,
  sessionId: aggregateId,
  correlationId,
  applicationProperties: {
    source: 'fund-service',
  },
});
```

adapter 內部再映射到 Azure SDK：

```ts
await sender.sendMessages({
  body: message.body,
  sessionId: message.sessionId,
  messageId: message.messageId,
  correlationId: message.correlationId,
  applicationProperties: {
    routingKey: message.routingKey,
    ...message.applicationProperties,
  },
});
```

### Consumer Context

handler 應取得必要 metadata，但不直接操作 Azure receiver：

```ts
export interface MessageContext {
  entityName: string;
  routingKey?: string;
  sessionId?: string;
  messageId?: string;
  correlationId?: string;
  deliveryCount: number;
  enqueuedTimeUtc?: Date;
}
```

handler 介面維持簡單：

```ts
export interface MessageHandler<T = unknown> {
  handle(message: T, context: MessageContext): Promise<void>;
}
```

settlement 預設由 consumer pipeline 統一控制，不交給每個 handler 自行呼叫。只有在明確有業務需求時，才另開 manual settlement escape hatch。

### Settlement Policy

consumer pipeline 應採類似以下的核心 flow：

```ts
try {
  await handler.handle(message.body, context);
  await receiver.completeMessage(message);
} catch (error) {
  if (isNonRetryable(error)) {
    await receiver.deadLetterMessage(message, {
      deadLetterReason: error.name,
      deadLetterErrorDescription: error.message,
    });
    return;
  }

  if (message.deliveryCount >= maxDeliveryCountBeforeDeadLetter) {
    await receiver.deadLetterMessage(message, {
      deadLetterReason: 'MaxDeliveryExceededInApplication',
      deadLetterErrorDescription: error.message,
    });
    return;
  }

  await receiver.abandonMessage(message);
}
```

這裡的重點不是固定上述程式碼，而是把 handler success / retryable failure / non-retryable failure / poison message 清楚投影到 Azure settlement。

### Error Classification

至少要能區分：

- retryable error。
- non-retryable error。
- validation error。
- downstream timeout。
- poison message。

可用 application error 類型：

```ts
export class NonRetryableMessageError extends Error {}
export class RetryableMessageError extends Error {}
```

也可用 policy object：

```ts
export interface MessageErrorClassifier {
  classify(error: unknown, context: MessageContext): 'retry' | 'deadLetter';
}
```

避免所有錯誤一律 `abandon`。不可重試錯誤若反覆消費到 Azure max delivery 才進 DLQ，會拉高觀測與排錯成本。

### Session Consumer Loop

session-enabled entity 應以 `acceptNextSession` 建立 worker loop。概念如下：

```ts
while (running) {
  try {
    const receiver = await client.acceptNextSession(entityName, {
      maxAutoLockRenewalDurationInMs,
    });

    receiver.subscribe(
      {
        processMessage: async (message) => {
          await pipeline.handle(message, receiver);
        },
        processError: async (args) => {
          logger.error(args.error);
        },
      },
      {
        autoCompleteMessages: false,
        maxConcurrentCalls: 1,
      },
    );
  } catch (error) {
    await backoff();
  }
}
```

實作時需顯式處理：

- `maxConcurrentSessions`。
- 每個 session 內 `maxConcurrentCalls: 1`。
- 無可用 session 時 backoff，避免 busy loop。
- receiver lifecycle 與 graceful shutdown。
- session lock renewal。
- active session / active receiver 數量觀測。
- 同一 process 建立過多 receiver 的上限控制。

## Trade-offs

### Pros

- 完整掌握 Azure Service Bus semantics：`sessionId`、`acceptNextSession`、`completeMessage`、`abandonMessage`、`deadLetterMessage`、lock renewal、delivery count、DLQ reason / description 都留在 service-kit 可治理的範圍。
- 避免第三方 abstraction 限制：不受既有 `MessageOptions`、consumer flow、metadata discovery 或 decorator 設計綁住。
- 更容易滿足 production reliability：retry、DLQ、idempotency、observability、graceful shutdown、backpressure、concurrency control 都可按本系統需求設計。
- 降低低維護套件風險：官方 SDK 由 Microsoft 維護；內部 adapter 的功能範圍可控制得比第三方 extension 小而明確。
- 仍可保留 NestJS 使用體驗：自行封裝不等於裸用 SDK，可提供 module、publisher、handler registry、config module integration 與 lifecycle hooks。

### Cons

- 初期設計成本較高：需要自行定義 publisher interface、consumer pipeline、handler registry、error policy、settlement policy、module config。
- 測試責任回到 service-kit：至少要覆蓋 publish mapping、`sessionId` mapping、handler success / retryable failure / non-retryable failure / delivery count 超限、no session backoff、graceful shutdown。
- 若未來支援多 broker，需要重新審視 abstraction 邊界。現階段 Azure Service Bus 已是 event broadcast 的確定基礎建設，優先做好 Azure semantics 比過早抽象多 broker 更實際。

## Implementation Phasing

### Publisher

- 建立 `AzureServiceBusModule`。
- 建立 singleton `ServiceBusClient`。
- 建立 `AzureServiceBusPublisher`。
- 支援 `sessionId`、`messageId`、`correlationId`、`applicationProperties`。
- sender 快取或集中管理，避免每次 publish 都 create / close sender。

### Basic Consumer

- 支援 non-session receiver。
- `autoCompleteMessages: false`。
- handler 成功後 `completeMessage`。
- retryable error → `abandonMessage`。
- non-retryable error → `deadLetterMessage`。
- 補上 unit tests。

### Session Consumer

- 支援 `acceptNextSession`。
- 支援 `maxConcurrentSessions`。
- session 內 `maxConcurrentCalls: 1`。
- 無 session 時 backoff。
- session receiver lifecycle 管理。
- lock renewal 設定。

### Observability And Governance

- logging。
- metrics。
- tracing。
- DLQ reason 規範。
- handler timeout。
- idempotency key。
- poison message dashboard / runbook。

## Open Dependencies

- **codebase survey**:正式 design 前仍須 survey `<codebase>/service-kit`、`contract-event`、既有 NestJS module / lifecycle patterns，以及是否已有 event runtime precedent。
- **entity provisioning**:`requiresSession` 是 queue / subscription 的 provisioning-time 屬性；是否啟用 session 必須在 service-local plan 中具名，不應由 runtime 自動推測。
- **event default**:依 proposal，目前 event broadcast 預設不開 session，consumer 靠 idempotency + version-stamping 維持 order-tolerance；session 主要保留給 job 端或具名 per-key 因果序需求。
- **third-party pattern reuse**:`@nestjstools/messaging` 可參考 handler registry、middleware pipeline、routing key 等 application-level 概念，但不直接依賴其 Azure extension。

## Decision Impact

若採納本文 opinion，下游 service-kit SB runtime design 應以「官方 SDK + 內部 adapter」作為 frozen input，並把第三方 NestJS messaging extension 降格為參考材料，而非 implementation dependency。
