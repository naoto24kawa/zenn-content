---
title: "@modelcontextprotocol/sdk が Cloudflare Workers で動かない理由と JSON-RPC 2.0 手書き実装"
emoji: "🔌"
type: "tech"
topics: ["cloudflare", "mcp", "typescript", "個人開発"]
published: true
---

## 先に結論

`@modelcontextprotocol/sdk` は Cloudflare Workers で動かない。内部依存の AJV が JSON Schema ファイルを動的 `require()` しており、Workers のバンドラ（esbuild ベース）がそれを解決できないのが原因だ。

解決策は **SDK を使わずに JSON-RPC 2.0 を手書きすること**。MCP の実体は JSON-RPC 2.0 なので、`switch` 文ベースで素直に書ける。

---

## 背景

新しいプロダクトを作っている。バックエンドは Cloudflare Workers で組んでおり、MCP サーバーを実装しようと `@modelcontextprotocol/sdk` をインストールしたところ、即座に壁にぶつかった。

---

## 何が起きたか

`wrangler deploy` を実行すると、`ajv/dist/compile/codegen` が解決できないというビルドエラーで止まる。

### 原因を調べた

SDK の依存関係を辿ると AJV (Another JSON Validator) が入っており、AJV が内部で JSON Schema ファイルを動的 `require()` している。

```
@modelcontextprotocol/sdk
  └─ ajv
       └─ ajv/dist/compile/codegen  ← 動的 require
```

Cloudflare Workers のバンドラは静的解析でモジュールグラフを解決する。動的 `require()` は解析できないため、バンドルが失敗する。

### 試したが解決しなかったこと

- `wrangler.toml` に `node_compat = true` を追加 → 解決しない（Node.js 互換レイヤーの問題ではない）
- AJV をバージョン固定で追加 → SDK が要求するバージョンと競合

---

## 解決策: JSON-RPC 2.0 を手書きする

MCP プロトコルの通信層の実体は [JSON-RPC 2.0](https://www.jsonrpc.org/specification) だ。SDK はその上にある便利なラッパーにすぎない。

仕様書を見るとリクエスト/レスポンスの構造はシンプルで、手書きで十分実装できる。

### 型定義

まず JSON-RPC 2.0 のリクエストと、MCP のツール結果の型を定義する。

```typescript
type JsonRpcRequest = {
  jsonrpc: "2.0";
  id?: string | number | null;
  method: string;
  params?: unknown;
};

type TextContent = { type: "text"; text: string };
type ToolResult = { content: TextContent[]; isError?: true };
```

### レスポンスヘルパー

成功・エラーのレスポンスを返す関数は 2 つだけでよい。

```typescript
function jsonrpc(id: string | number | null | undefined, result: unknown) {
  return { jsonrpc: "2.0", id: id ?? null, result };
}

function jsonrpcError(id: string | number | null | undefined, code: number, message: string) {
  return { jsonrpc: "2.0", id: id ?? null, error: { code, message } };
}
```

### メソッドディスパッチャー

MCP が規定するメソッドを `switch` で振り分けるだけだ。

```typescript
switch (method) {
  case "initialize":
    return c.json(
      jsonrpc(id, {
        protocolVersion: "2024-11-05",
        capabilities: { tools: {} },
        serverInfo: { name: "todoke", version: "1.0.0" },
      }),
    );

  case "notifications/initialized":
    return c.newResponse(null, 204);

  case "ping":
    return c.json(jsonrpc(id, {}));

  case "tools/list":
    return c.json(jsonrpc(id, { tools: TOOLS }));

  case "tools/call": {
    const p = (params ?? {}) as { name?: string; arguments?: Record<string, unknown> };
    if (typeof p.name !== "string") {
      return c.json(jsonrpcError(id, -32602, "Invalid params: missing 'name'"));
    }
    const { name, arguments: args = {} } = p;
    try {
      let result: ToolResult;
      if (name === "send_notification") result = await callSendNotification(args, env, appId);
      else if (name === "get_stats")     result = await callGetStats(env, appId);
      // ... 他のツール
      else return c.json(jsonrpcError(id, -32602, `Unknown tool: ${name}`));
      return c.json(jsonrpc(id, result));
    } catch (err) {
      const msg = err instanceof Error ? err.message : "Tool execution failed";
      return c.json(jsonrpc(id, { content: [{ type: "text", text: `Error: ${msg}` }], isError: true }));
    }
  }

  default:
    return c.json(jsonrpcError(id, -32601, `Method not found: ${method}`));
}
```

`initialize` → `tools/list` → `tools/call` の 3 つが実装できれば、Claude Desktop や Cursor から接続できる。

### ツール定義

MCP ツールは JSON Schema で定義する。SDK なしでも同じ構造を手書きすればよい。

```typescript
const TOOLS = [
  {
    name: "send_notification",
    description: "Send a push notification to all active subscribers of the app.",
    inputSchema: {
      type: "object",
      required: ["title", "body"],
      properties: {
        title: { type: "string", description: "Notification title" },
        body:  { type: "string", description: "Notification body text" },
        url:   { type: "string", format: "uri", description: "URL to open on click" },
      },
    },
  },
  // ... 他のツール
];
```

ツール定義は純粋なオブジェクトなので、AJV も SDK も不要だ。

---

## 実際の構成

todoke の MCP サーバー (`apps/api/src/routes/mcp.ts`) は 295 行で、以下の 5 つのツールを実装している。

| ツール | 概要 | 必要スコープ |
|---|---|---|
| `send_notification` | Push 通知を送信 | notify 以上 |
| `get_stats` | 購読者数・送信数の統計 | notify 以上 |
| `list_api_keys` | API キー一覧 | full のみ |
| `create_api_key` | API キー作成 | full のみ |
| `delete_api_key` | API キー削除 | full のみ |

スコープ制御も SDK なしで実装している。`tools/call` の中で呼び出し元のキースコープを確認し、`full` が必要なツールへのアクセスを弾く。

```typescript
const FULL_SCOPE_TOOLS = ["list_api_keys", "create_api_key", "delete_api_key"];
if (FULL_SCOPE_TOOLS.includes(name) && keyScope !== "full") {
  return c.json(
    jsonrpc(id, {
      content: [{ type: "text", text: "Error: This tool requires full scope" }],
      isError: true,
    }),
  );
}
```

---

## SDK ありと手書きの比較

| | `@modelcontextprotocol/sdk` | JSON-RPC 2.0 手書き |
|---|---|---|
| Cloudflare Workers 対応 | ❌ ビルド失敗 | ✅ |
| 外部依存 | AJV 等 複数 | なし |
| 型安全性 | SDK が保証 | 自前で型定義 |
| MCP バージョン追従 | SDK 更新で自動 | 仕様書を見て手動 |
| 実装量 | 少ない | 約 300 行 |

型安全性は TypeScript の型定義で補える。MCP の仕様変更への追従は手動になるが、JSON-RPC 2.0 の基本構造は安定しており、実際の変更は `protocolVersion` の文字列と capabilities の内容程度だ。

---

## まとめ

- `@modelcontextprotocol/sdk` は AJV の動的インポートにより Cloudflare Workers で動かない
- JSON-RPC 2.0 は `switch` + ヘルパー 2 関数で手書き実装できる
- ツール定義は純粋なオブジェクトなので SDK なしで書ける
- Workers 上で MCP サーバーを動かしたいなら、最初から手書きを選んだ方が早い

同じ問題で詰まっている人の参考になれば。

---

## 関連記事

- [個人開発で監視 SaaS を 1 ヶ月で作ってリリースした話](https://zenn.dev/naoto24kawa/articles/20260417-manako-launch-story)
