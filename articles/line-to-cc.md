---
title: "スマホの LINE から Claude Code を遠隔操作できるようにした"
emoji: "📱"
type: "tech"
topics: ["claudecode", "line", "mcp", "typescript", "bun"]
published: false
---

## この記事で作るもの

LINE にメッセージを送ると、自宅 PC で動いている Claude Code が受信・処理して、結果を LINE に返してくれる仕組みです。

**ツール実行の承認も LINE のボタン1タップでできます。**

![Permission Relay の Flex Message カード](/images/line-to-cc/permission-card.png)

https://github.com/elchika-inc/line-to-cc

起動コマンド1つで、トンネル構築から Webhook 設定、疎通テストまで全自動で完了します。

```bash
claude --dangerously-load-development-channels server:line
```

```
[line] Webhook server listening on http://127.0.0.1:8788/webhook
[line] Starting cloudflared tunnel (killing existing tunnels first)...
[line] Tunnel URL: https://xxxxx.trycloudflare.com
[line] Setting webhook URL: https://xxxxx.trycloudflare.com/webhook
[line] Webhook URL set successfully
[line] Webhook URL verified in LINE
[line] Testing webhook connectivity...
[line] Webhook connectivity test PASSED (status: 200)
```

:::message
Claude Code Channels は Research Preview 段階のため、`--dangerously-load-development-channels` フラグが必要です。
:::

## なぜ作ったか

Claude Code は強力ですが、ターミナルの前にいないと使えません。

**「外出中にスマホから指示を出して、帰ったら終わっている」**

Telegram 版は公式が出していますが、日本のユーザーには LINE のほうが身近です。それなら自分で作ろうと思いました。

## アーキテクチャ

```
┌──────────┐     Webhook POST     ┌──────────────────┐
│ LINE App │ ──────────────────> │  LINE Platform   │
└──────────┘                     └────────┬─────────┘
      ▲                                   │ HTTPS
      │ Push API                          ▼
      │                          ┌──────────────────┐
      │                          │ Cloudflare Tunnel │
      │                          │ (Quick Tunnel)    │
      │                          └────────┬─────────┘
      │                                   │ localhost:8788
      │                                   ▼
      │                          ┌──────────────────┐
      │                          │   Hono Server    │
      │                          │ POST /webhook    │
      │                          │ ┌──────────────┐ │
      │                          │ │署名検証      │ │
      │                          │ │sender gating │ │
      │                          │ │verdict判定   │ │
      │                          │ └──────┬───────┘ │
      │                          └────────┼─────────┘
      │                                   │ MCP (stdio)
      │                                   ▼
      │                          ┌──────────────────┐
      └───────── line_reply ──── │  Claude Code     │
                  tool           │  Session         │
                                 └──────────────────┘
```

LINE → Cloudflare Tunnel → ローカルの Hono サーバー → MCP 通知 → Claude Code という一直線の構成です。返信は Claude Code が `line_reply` tool を呼び出し、Push API で LINE に送ります。

### Claude Code Channels とは

Claude Code v2.1.80 で追加された機能で、外部メッセージングサービスを Claude Code セッションに接続できます。MCP Server として `claude/channel` capability を宣言することで、外部イベントをセッションに push できます。

公式には [Telegram プラグイン](https://github.com/anthropics/claude-plugins-official/tree/main/external_plugins/telegram) がリファレンス実装として公開されています。Telegram はポーリング方式ですが、LINE は Webhook 方式なので、Cloudflare Tunnel で外部公開する必要がある点が大きな違いです。

## 機能紹介

### 1. 双方向テキストチャット

LINE からメッセージを送ると Claude Code セッションに届き、Claude Code の返信が LINE に返ります。

![LINE でのチャットの様子](/images/line-to-cc/chat.png)

### 2. Permission Relay -- ボタン1タップで承認

Claude Code がツール実行(Bash, Write 等) の承認を求めると、LINE に **Flex Message カード**が届きます。

![Flex Message カード](/images/line-to-cc/permission-card.png)

「Allow」「Deny」ボタンをタップするだけ。テキストで `yes` / `no` と入力しても OK です。

当初は `yes abcde` (5文字の request_id 付き) というテキスト入力方式でしたが、スマホでは面倒すぎたので改善しました。

| 方式 | 操作 |
|------|------|
| ~~テキスト + request_id~~ | ~~`yes abcde` と入力~~ |
| ~~Quick Reply ボタン~~ | ~~画面下部のフローティングボタン(ダサい)~~ |
| **Flex Message footer ボタン** | **カード内の「Allow」「Deny」を1タップ** |

### 3. Sender Gating -- ペアリングコード方式

Bot にメッセージを送ると6文字のペアリングコードが返信されます。Claude Code セッションでそのコードを入力すると、以降そのユーザーだけが操作できるようになります。

![ペアリングコードの発行](/images/line-to-cc/pairing.png)

- 有効期限: 1時間
- 最大試行: 2回
- 同時ペアリング: 1ユーザーのみ

### 4. 全自動セットアップ

`claude --dangerously-load-development-channels server:line` で起動すると:

1. HTTP サーバーが localhost:8788 で起動
2. 既存の cloudflared プロセスを自動で kill
3. 新しい Quick Tunnel を構築
4. LINE API で Webhook URL を自動設定 (バックオフ付きリトライ)
5. LINE API で疎通テストを実行
6. セッションに「LINE channel ready」を通知

**LINE Developers Console を毎回開く必要はありません。**

![LINE channel ready 通知](/images/line-to-cc/ready.png)

## ハマりどころ -- 実装で苦労した3つのポイント

ここからが本題です。「作ってみた」記事は多いですが、**ハマりどころの一次情報**こそが価値だと思っているので、詳しく書きます。

### 1. cloudflared の URL 取得 != 外部到達可能

cloudflared の Quick Tunnel は起動すると stderr に URL を出力します。この URL をパースして LINE API で Webhook URL を設定する...のですが、**URL が出力された時点ではまだ外部から到達できない**ことがあります。

Cloudflare のエッジへの接続確立にラグがあるためです。

さらに厄介なのが、**LINE の PUT `/v2/bot/channel/webhook/endpoint` API は、設定時に URL の到達性を内部で検証する**ということ。到達できないと `400 Invalid webhook endpoint URL` で拒否されます。

解決策: LINE API のリトライ自体を到達性チェックとして使う。

```ts
for (let i = 0; i < 5; i++) {
  if (i > 0) {
    const waitSec = 3 + i * 2  // 3s, 5s, 7s, 9s
    await new Promise((r) => setTimeout(r, waitSec * 1000))
  }
  setOk = await lineClient.setWebhookUrl(webhookUrl)
  if (setOk) break
}
```

最初は `fetch()` でローカルからトンネル URL にアクセスして到達性を確認しようとしましたが、**macOS の DNS リゾルバが IPv6 優先で `*.trycloudflare.com` を解決できない**問題にハマりました。LINE のサーバーからは正常に到達できるので、LINE API のリトライに任せるのが正解でした。

### 2. MCP 接続とトンネルセットアップの順序問題

MCP Server は `mcp.connect(transport)` で Claude Code と stdio 接続します。この呼び出しは **blocking** です。

最初の実装ではトンネルセットアップを `mcp.connect()` の前に `await` していました。すると:

- トンネル起動に 5-10 秒かかる
- Webhook URL 設定のリトライで 10-20 秒かかる
- 合計 30 秒近く MCP 接続が遅延する
- Claude Code 側がタイムアウトする可能性

解決策: MCP 接続を先に完了し、トンネルセットアップは非同期で実行。結果を MCP notification でセッションに通知します。

```ts
// MCP 接続を先に完了
const transport = new StdioServerTransport()
await mcp.connect(transport)

// トンネルは非同期で (MCP 接続後なので notification が使える)
setupTunnelAndWebhook()
  .then(async () => {
    await mcp.notification({
      method: 'notifications/claude/channel',
      params: {
        content: 'LINE channel ready.',
        meta: { status: 'ready' },
      },
    })
  })
  .catch(async (err) => {
    await mcp.notification({
      method: 'notifications/claude/channel',
      params: {
        content: `LINE tunnel setup failed: ${err.message}`,
        meta: { status: 'error' },
      },
    })
  })
```

### 3. Reply Token の罠

LINE の Webhook には `replyToken` が含まれますが、**数秒で失効**します。Claude Code がメッセージを処理して返信を生成するのを待っていると、確実に期限切れになります。

そのため、全ての返信に **Push API** (`POST /v2/bot/message/push`) を使っています。Reply API と違い Push API は月 200 通の制限がありますが、個人利用なら十分です。

## 署名検証の実装

LINE は Webhook リクエストに HMAC-SHA256 署名を付与します。Web Crypto API の `crypto.subtle.verify` を使えば、タイミングセーフな検証がワンライナーで書けます。

```ts
export async function verifySignature(
  body: string, secret: string, signature: string,
): Promise<boolean> {
  const encoder = new TextEncoder()
  const key = await crypto.subtle.importKey(
    'raw', encoder.encode(secret),
    { name: 'HMAC', hash: 'SHA-256' }, false, ['verify'],
  )
  const sigBuf = Uint8Array.from(atob(signature), (c) => c.charCodeAt(0))
  return crypto.subtle.verify(
    'HMAC', key, sigBuf.buffer as ArrayBuffer, encoder.encode(body),
  )
}
```

**重要: JSON パース前の生文字列で検証すること。** パース後だとバイト列が変わり、検証が壊れます。Hono なら `c.req.text()` で生ボディを取得できます。

## セットアップ (5分で完了)

### 前提

| ソフトウェア | インストール |
|-------------|-------------|
| Claude Code v2.1.80+ | [公式サイト](https://claude.ai/claude-code) |
| Bun | `curl -fsSL https://bun.sh/install \| bash` |
| cloudflared | `brew install cloudflared` |

### 手順

```bash
# 1. Clone
git clone https://github.com/elchika-inc/line-to-cc.git
cd line-to-cc
bun install

# 2. LINE Developers Console で Messaging API チャネルを作成
#    Channel Secret と Channel Access Token を取得

# 3. Credentials
cp .env.example .env
# .env を編集

# 4. 起動
claude --dangerously-load-development-channels server:line

# 5. LINE で Bot を友だち追加してメッセージ送信 -> ペアリング
```

## Telegram プラグインとの比較

| 項目 | Telegram (公式) | LINE (本プラグイン) |
|------|----------------|---------------------|
| メッセージ取得 | Bot API ポーリング | Webhook (HTTPS POST) |
| 外部公開 | 不要 | 必要 (Cloudflare Tunnel) |
| トンネル設定 | - | 全自動 |
| 署名検証 | Bot token ベース | HMAC-SHA256 |
| Permission UI | インラインボタン | Flex Message カード |
| 日本での普及率 | 低い | ほぼ全員 |

## 制約事項

- Claude Code Channels は Research Preview (`--dangerously-load-development-channels` フラグ必須)
- LINE 無料プランは月 200 通 (個人利用なら十分)
- テキストメッセージのみ (画像・ファイル非対応)
- マシンがスリープすると接続が切れる

## まとめ

MCP の `claude/channel` capability を使えば、任意のメッセージングサービスを Claude Code に接続できます。LINE 版を作ったことで、**スマホから Claude Code を遠隔操作する**ワークフローが実現しました。

実装のコアは 8 ファイル、テスト 46 件。Bun + Hono + MCP SDK の構成で、外部ライブラリへの依存は最小限です。

ソースコードは MIT ライセンスで公開しています。

https://github.com/elchika-inc/line-to-cc

:::message
Channels 機能が GA (一般公開) されれば、`--dangerously-load-development-channels` フラグなしで使えるようになるはずです。
:::
