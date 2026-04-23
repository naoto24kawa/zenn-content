---
title: "Cloudflare Queues Consumer の 2 つの落とし穴と対策"
emoji: "🪤"
type: "tech"
topics: ["cloudflare", "typescript", "個人開発"]
published: true
---

## 先に結論

Cloudflare Queues の Consumer を書くとき、2 つの落とし穴がある。

1. **ポイズンメッセージ** — `try/catch` なしで書くと、1 件の失敗以降のメッセージが全滅する
2. **重複消費** — Queues は at-least-once 配信なので、リトライ時に同じメッセージが 2 回処理される

どちらも「動いているように見えるが、特定条件で壊れる」タイプの問題だ。この記事では両方の原因と対策を、実際のコードを元に解説する。

---

## Cloudflare Queues の基本特性

対策を理解する前に、Queues の配信保証を押さえておく。

Cloudflare Queues は **at-least-once 配信** を保証する。つまり:

- メッセージは**最低 1 回は**処理される
- ただし、**複数回処理される可能性がある**

Consumer が `message.ack()` を呼ぶとメッセージは完了扱いになる。呼ばなかった場合、または `message.retry()` を呼んだ場合は、Queues がリトライをスケジュールする。

バッチ Consumer の型は以下のとおりだ。

```typescript
async queue(batch: MessageBatch<T>, env: Env): Promise<void> {
  for (const message of batch.messages) {
    // ここで各メッセージを処理する
  }
}
```

`batch.messages` を `for` ループで回し、各メッセージに `ack()` か `retry()` を呼ぶのが基本パターンだ。この構造が理解できていないと、2 つの落とし穴に落ちる。

---

## 落とし穴 1: ポイズンメッセージによる全滅

### 何が起きるか

次のような Consumer を書いたとする。

```typescript
async queue(batch: MessageBatch<T>, env: Env): Promise<void> {
  for (const message of batch.messages) {
    await process(message.body);
    message.ack();
  }
}
```

`process()` がエラーを投げると `for` ループが止まる。そのメッセージ以降はすべて未処理のまま残り、バッチ全体がリトライ対象になる。

「処理できないメッセージ」= **ポイズンメッセージ**が 1 件あるだけで、後続の健全なメッセージも何度もリトライされ続ける。

### 原因

エラーが `for` ループの外に抜けることで、`ack()` が呼ばれないメッセージが生まれる。Queues は「`ack()` が呼ばれなかった = 失敗」と判断し、バッチ全体をリトライする。

### 対策: メッセージ単位の try/catch

各メッセージを独立した `try/catch` で囲み、失敗しても後続に影響を与えないようにする。

```typescript
async queue(batch: MessageBatch<T>, env: Env): Promise<void> {
  for (const message of batch.messages) {
    try {
      await process(message.body);
      message.ack();
    } catch (err) {
      console.error("[queue] processing failed", message.id, err);
      message.retry();
    }
  }
}
```

ポイントは 2 つだ。

- `message.ack()` は**成功時のみ**呼ぶ
- `message.retry()` は**失敗時に明示的に**呼ぶ

「失敗したメッセージをリトライキューに戻す」という意図を明示することで、ポイズンメッセージが後続を巻き込まなくなる。

沈黙のまま消えるより、意図的にリトライさせる方が**監視できる**。`console.error` でログを残しておけば、特定のメッセージが繰り返し失敗していることを検知できる。

---

## 落とし穴 2: at-least-once による重複消費

### 何が起きるか

ポイズンメッセージ対策を入れても、問題は残る。

処理に成功して `ack()` を呼んだはずなのに、Queues が「受信確認がなかった」と判断してリトライすることがある。ネットワークの瞬断、Worker の強制終了、Queues 側の内部的なリトライなど、原因はさまざまだ。

結果として、同じメッセージが 2 回処理される。DB に通知ログを 2 行書いてしまう、メールを 2 通送ってしまう、といった副作用が起きる。

### 原因

at-least-once 保証は「最低 1 回処理されることを保証する」だけで、「1 回だけ処理されることを保証する」わけではない。Queues の仕様上、重複は起こりえる。

### 対策: messageId による冪等性

Queues が各メッセージに付与する `message.id` を使い、**同じメッセージを 2 回処理しない**設計にする。

鍵は `INSERT OR IGNORE` だ。

```typescript
export async function processQueueMessage(
  message: NotificationMessage,
  env: Env,
  messageId = crypto.randomUUID(),
): Promise<void> {
  // ... 通知送信処理 ...

  // INSERT OR IGNORE で冪等性を保証
  // 同じ messageId が既に存在する場合は何もしない
  const logResult = await env.DB.prepare(
    "INSERT OR IGNORE INTO notification_logs (id, app_id, title, body, url, sent_count, failed_count, created_at) VALUES (?, ?, ?, ?, ?, ?, ?, ?)",
  )
    .bind(messageId, message.app_id, message.title, message.body, message.url ?? null, sent, failed, now)
    .run();

  // changes === 0 なら重複処理 → 以降のカウント更新をスキップ
  if (logResult.meta.changes > 0 && sent > 0) {
    await env.DB.prepare(
      `INSERT INTO monthly_usage (app_id, year_month, sent_count) VALUES (?, ?, ?)
       ON CONFLICT(app_id, year_month) DO UPDATE SET sent_count = sent_count + excluded.sent_count`,
    )
      .bind(message.app_id, yearMonth, sent)
      .run();
  }
}
```

`INSERT OR IGNORE` は「同じ主キーのレコードが既に存在すれば INSERT をスキップする」SQLite の構文だ。`notification_logs` の主キーを `messageId` にしておけば、リトライで同じメッセージが来ても 2 回目以降は無視される。

`logResult.meta.changes > 0` で「今回初めて処理された」かどうかを判定し、月次カウントの更新も初回のみに限定している。

---

## 2 つを組み合わせた「防御的 Consumer」

落とし穴 1 と 2 の対策を組み合わせると、Consumer の全体像は次のようになる。

```typescript
// index.ts - Queues ハンドラ
async queue(batch: MessageBatch<NotificationMessage>, env: Env): Promise<void> {
  for (const message of batch.messages) {
    try {
      await processQueueMessage(message.body, env, message.id);
      message.ack();
    } catch (err) {
      console.error("[queue] processing failed", message.id, err);
      message.retry();
    }
  }
},
```

```typescript
// consumer.ts - 処理本体 (messageId を主キーに冪等性を保証)
export async function processQueueMessage(
  message: NotificationMessage,
  env: Env,
  messageId = crypto.randomUUID(), // Queue 経由では必ず message.id を渡す。デフォルト値はテスト用
): Promise<void> {
  // ... 処理 ...

  await env.DB.prepare(
    "INSERT OR IGNORE INTO notification_logs (id, ...) VALUES (?, ...)",
  ).bind(messageId, ...).run();
}
```

`message.id` を `processQueueMessage` に渡すことで、Queues のメッセージ ID が DB の主キーと一致する。これを「**防御的 Consumer**」と呼ぶことにする。

防御的 Consumer には 3 つの性質がある。

1. **独立性** — 各メッセージの失敗が後続に波及しない
2. **冪等性** — 同じメッセージが何度来ても DB の状態が変わらない
3. **可観測性** — 失敗ログを残すことで、どのメッセージが問題かを特定できる

---

## 実装上の注意点

### ack/retry の呼び忘れ

`try/catch` の中で `ack()` も `retry()` も呼ばずに関数を抜けると、メッセージはタイムアウトまで未処理扱いになる。必ずどちらかを呼ぶ。

### messageId のスコープ

`message.id` は Queues が付与するユニーク ID だ。Queue Consumer から `processQueueMessage` を呼ぶ際は必ず `message.id` をそのまま渡す。Consumer 内部で `crypto.randomUUID()` などを呼んで新しい ID を生成すると、リトライのたびに異なる ID になり冪等性が崩れる。

なお関数の引数にデフォルト値 `= crypto.randomUUID()` を持たせるパターンがある。これはテストから messageId なしで呼べるようにするためのものであり、Queue Consumer 経由では必ず `message.id` を渡すという前提は変わらない。

### INSERT OR IGNORE の前提

`INSERT OR IGNORE` が機能するには、対象テーブルの `id` カラムに `PRIMARY KEY` または `UNIQUE` 制約が必要だ。制約がなければ重複行が挿入されてしまう。

---

## まとめ

| 落とし穴 | 症状 | 対策 |
|---|---|---|
| ポイズンメッセージ | 1 件の失敗でバッチ全滅 | メッセージ単位の `try/catch` + `ack`/`retry` 明示 |
| 重複消費 | 同じメッセージが 2 回処理される | `INSERT OR IGNORE` + `messageId` を主キーに |

両方対策した Consumer を「防御的 Consumer」と呼んだ。独立性・冪等性・可観測性の 3 つを備えることで、Queues の at-least-once 保証の上で安全に動く。

at-least-once は「失敗しない」ではなく「必ず届ける」保証だ。Consumer 側がそれを受け止める設計になっていないと、届いたメッセージが壊れた状態を作り出す。

---

## 関連記事

- [@modelcontextprotocol/sdk が Workers で動かない理由と JSON-RPC 2.0 手書き実装](https://zenn.dev/naoto24kawa/articles/20260422-todoke-mcp-cf-workers)
