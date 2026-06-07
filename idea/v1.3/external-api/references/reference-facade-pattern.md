---
status: draft
updated_at: 2026-06-07
updated_by: Codex
---

# Facade Pattern Reference

## Facade 是什麼

Facade 可以先理解成：

> 對外提供一個穩定入口。裡面怎麼拿資料、怎麼組資料可以換，但外面呼叫的人不用知道。

在 External API 的 Stage 2，facade / client 的目的是保留 service boundary。

現在 `external`、`fund`、`wallet` 暫時都在 `esingpay-cradle` 同一個 instance 裡，所以可以用 function call 先串起來。但概念上它們未來會拆成不同 microservice，因此 external adapter 不應直接依賴 fund / wallet 的 internal query service、composer、use case。

## 最簡單例子

假設 wallet service 本體提供一個 facade：

```ts
// wallet/facade/wallet.facade.ts
export class WalletFacade {
  constructor(private readonly walletQueryService: WalletQueryService) {}

  async getWallet(id: string) {
    return this.walletQueryService.getOneById(id);
  }
}
```

External 不直接碰 `WalletQueryService`，而是透過 external client：

```ts
// external/client/wallet.client.ts
export class ExternalWalletClient {
  constructor(private readonly walletFacade: WalletFacade) {}

  async getWallet(id: string) {
    return this.walletFacade.getWallet(id);
  }
}
```

External adapter 只知道：

```ts
const wallet = await this.walletClient.getWallet(id);
```

External adapter 不需要知道背後現在是 function call、query service、composer，還是未來 RPC。

## 如果沒有 Facade

如果 external adapter 直接依賴 wallet internal implementation：

```ts
export class ExternalWalletService {
  constructor(
    private readonly walletQueryService: WalletQueryService,
    private readonly walletComposer: WalletComposer,
  ) {}

  async getWallet(id: string) {
    const wallet = await this.walletQueryService.getOneById(id);
    return this.walletComposer.composeOne(wallet);
  }
}
```

現在看起來很方便，但未來 wallet 拆成獨立 microservice 後，這兩個 dependency 不在同一個 process 裡：

```ts
WalletQueryService
WalletComposer
```

External 不能再直接 inject 它們。原本 external adapter 的主流程就必須改成 RPC：

```ts
// 原本
const wallet = await this.walletQueryService.getOneById(id);
const composed = await this.walletComposer.composeOne(wallet);

// 未來
const result = await this.walletRpcClient.getWallet({ id });
```

如果 external 很多地方都直接用 `WalletQueryService + WalletComposer`，每個地方都要改。

## 用 Facade / Client 的好處

External 一開始就只呼叫：

```ts
const wallet = await this.walletClient.getWallet(id);
```

現在 client 背後可以是同一個 cradle instance 內的 function call：

```ts
return this.walletFacade.getWallet(id);
```

未來換成真 RPC 時，只改 client：

```ts
return this.walletRpcClient.getWallet({ id });
```

External adapter 不用改。

所以 facade / client 的價值是：

```text
把「現在怎麼拿資料」藏起來。
讓 external 只依賴一個穩定入口。
未來 function call 換 RPC 時，只改入口後面的實作。
```

## 對 External API 的套用

不要讓 external adapter 直接做：

```text
external adapter
  -> fund / wallet internal query service
  -> fund / wallet composer
```

Stage 2 暫時應做：

```text
external adapter
  -> external client
  -> fund / wallet facade
  -> fund / wallet query service / composer
```

未來 RPC 完成後抽換成：

```text
external adapter
  -> external client
  -> fund / wallet contract-rpc client
  -> fund / wallet rpc server
```

External adapter 的主流程不需要重寫。
