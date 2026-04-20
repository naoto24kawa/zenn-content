---
title: "個人開発で監視SaaSを1ヶ月で作って正式リリースした話"
emoji: "🚀"
type: "tech"
topics: ["個人開発", "cloudflare", "claudecode", "monitoring", "typescript"]
published: true
---

## はじめに — なぜ監視SaaSを作ったのか

個人開発で Web サービスを運営していると、「落ちてたけど気づかなかった」という経験が一度はある。

自分もそうだった。以前運営していたサービスが深夜にダウンして、翌朝ユーザーから連絡をもらって初めて気づいた。無料の監視ツールは試したことがある。UptimeRobot、Uptime Kuma、Checkly — どれも優秀だけど、いくつか不満が残った。

- 日本語 UI がない (あっても中途半端)
- 死活監視・SSL・Cron・Web 変更検知が別々のツール
- MCP / AI 連携に対応しているものが見当たらない

「なければ作ればいい」。個人開発者にとって、これは最も危険で、最も楽しい判断だ。

こうして [Manako](https://manako.dev) が生まれた。日本語 UI のオールインワン監視ダッシュボード。HTTP / TCP / Ping / Heartbeat / Web 変更検知 / SSL / ドメイン、7 種の監視を 1 つの画面で管理できる。

この記事では、ゼロからリリースまでの約 1 ヶ月を振り返る。技術選定の判断、開発中に踏んだ失敗、Claude Code にどこまで助けてもらったか — 全部書く。

## なぜ Manako を作ったか — 先に結論

3 つの問いに答える形で作った。

**問 1: 個人開発者が「監視、面倒だな」と思う理由は何か？**

ツールが多すぎること、日本語がないこと、そして「どのツールが何をカバーするか」を覚えていないといけないこと。UptimeRobot で死活監視、Checkly でブラウザテスト、別のサービスで SSL 期限チェック — これを全部設定したあと、どれかのアラートが来たとき「どのダッシュボードを見ればいいか」を思い出す認知負荷がある。

**問 2: 日本の個人開発者に最適化された監視ツールは存在するか？**

調べた限り、存在しなかった。日本語 UI + オールインワン + 個人開発者価格帯 — この 3 つが揃っているものが見当たらなかった。

**問 3: AI 時代の監視ツールはどうあるべきか？**

Claude Code や Cursor から自然言語で監視を操作できるべきだと思った。「manako.dev の死活監視を追加して」と打って、HTTP モニターが作成される。これが 2026 年の監視ツールのあり方だと判断した。

この 3 問に「自分が一番困っている当事者」として答えた結果が Manako だ。

## Manako の概要

HTTP / TCP / SSL / Heartbeat など 7 種の監視を 1 つの日本語ダッシュボードで管理できる SaaS だ。通知は Slack / LINE / PagerDuty など 10 チャンネルに対応している。

一番尖っているのは MCP Server 対応で、Claude Code から「manako.dev の死活監視を追加して」と打つだけで HTTP モニターが作成される。この話は別の記事に書いた → [MCP vs API vs CLI 論争の中で、監視SaaSのMCP serverを設計した話](https://zenn.dev/naoto24kawa/articles/manako-mcp-server-design)

## 技術選定 — Cloudflare 100% という判断

最大の技術判断は「Cloudflare Workers スタック 100% で組む」ことだった。

| コンポーネント | 選定技術 |
|---|---|
| ランタイム | Cloudflare Workers |
| API フレームワーク | Hono |
| DB | Cloudflare D1 (SQLite) + Drizzle ORM |
| 時系列データ | Analytics Engine |
| KV | Cloudflare KV (セッション, レート制限) |
| オブジェクトストレージ | R2 (Web 変更検知のスクリーンショット) |
| キュー | Cloudflare Queues (通知配信) |
| フロントエンド | React + Vite |
| CI/CD | GitHub Actions |

AWS や GCP を使わない選択は「インフラ費を気にしない」という心理的安全性に直結する。監視 SaaS はチェッカーが 1 分ごとに動くので、従量課金だとコストの読みが難しい。Cloudflare Workers の Paid プラン ($5/月) で基本的な running cost が固定されるのは大きかった。

### Analytics Engine という選択

一番悩んだのは時系列データの保存先だ。レスポンスタイムを D1 (SQLite) に入れると書き込み頻度が高すぎる。Analytics Engine は書き込み無制限・クエリ回数課金で、監視 SaaS のユースケースにぴったりだった。

トレードオフはサンプリング。Analytics Engine はデータをサンプリングして返すので、「稼働率 99.98% と表示して実は 99.97% だった」という誤差が理論上ありうる。

結論: 個人開発者向け価格帯でこの精度を気にするユーザーはいない。正確さよりも「月 $5 で動く」ほうが価値がある。

### ID 設計: UUID より ULID を選んだ理由

全テーブルの主キーに ULID を採用した。時刻ソート可能 + B-Tree フレンドリーという特性が監視ログの時系列クエリに刺さる。

```typescript
import { ulid } from 'ulidx';

// モニター作成時
const monitorId = ulid(); // 例: 01HVZX3K2JPXNB8PXZJTQ5M3NS
// → タイムスタンプが先頭に入るため、WHERE created_at BETWEEN より
//   WHERE id BETWEEN '01HVZ...' AND '01HW0...' の方がインデックス効率が良い
```

UUID v4 (完全ランダム) だと B-Tree インデックスの挿入コストが高くなる。ULID はモノトニックに増加するため、D1 の SQLite インデックスと相性が良かった。

### D1 の地雷

D1 (SQLite) はテーブル 24 個まで育ったが、いくつか踏んだ。(技術選定の経緯はこちら → [月$5で監視SaaSを動かす -- Cloudflare Workers + Hono の技術選定](https://zenn.dev/naoto24kawa/articles/manako-cloudflare-tech-stack))

**`PRAGMA foreign_keys=OFF` が無視される**: テーブルの再構築 (カラム追加・型変更) で FK 制約があるテーブルを DROP できない。SQLite の仕様だが、Cloudflare D1 の環境では `PRAGMA` が一部制限されている。

回避策として「CREATE → INSERT SELECT → RENAME」の 3 ステップパターンを確立した:

```sql
-- ① 新テーブルを作る (FK 制約なし)
CREATE TABLE monitors_new AS SELECT * FROM monitors;
ALTER TABLE monitors_new ADD COLUMN new_col TEXT DEFAULT '';

-- ② データを移行
INSERT INTO monitors_new SELECT *, '' FROM monitors;

-- ③ 差し替え
DROP TABLE monitors;
ALTER TABLE monitors_new RENAME TO monitors;
```

このパターンを本番ランブックに書くことになった。個人開発で本番ランブックを書く日が来るとは思っていなかった。

## 1 ヶ月の開発タイムライン

ゼロから正式リリースまで、約 4 週間。commits 数は約 2,000。

### Week 1: 基盤

- モノレポ構成 (pnpm workspaces)
- 認証 (Email/Password + Google/GitHub OAuth)
- 7 種のモニター CRUD API
- HTTP / TCP / Ping / Heartbeat チェッカー

### Week 2: プロダクション投入

- 本番デプロイ (Workers 8 個 + Pages 4 個)
- Stripe Billing 統合 (Pro / Business プラン)
- 10 チャンネル通知システム
- MCP Server (4 ツール)
- CLI (npm publish)
- ドキュメントサイト (Astro Starlight)

### Week 3-4: 磨き込みとリリース

- セキュリティ強化 (MFA, Turnstile CAPTCHA, メール認証)
- UI/UX 改善 (設定サイドバー, ヘルプペイン)
- 年間課金オプション
- Cookie 同意 + GTM
- E2E テスト 55 シナリオ

振り返ると、Week 2 の密度が異常だった。Stripe 統合と 10 チャンネル通知と MCP Server を同じ週にやったのは無謀だったかもしれない。でも「リリースしてからやる」のスタンスだと永遠にリリースできないので、腹を括った。

## 失敗談 — 踏んだ地雷たち

Building in public なので、失敗も全部書く。

### 1. データモデルを 3 回書き直した

最初に設計したデータモデルは、監視の「結果」と「インシデント」が同じテーブルに混在していた。これが Week 2 で崩壊した。

通知チャンネルを 10 種類に増やした時点で、「1 回の監視失敗」が「複数チャンネルへの通知」を発火するという多対多の関係を既存テーブルで表現できないことがわかった。

2 回目の設計では `incidents` テーブルと `notifications` テーブルを分離した。しかし今度は「どのインシデントがどの通知を送ったか」の追跡が複雑になり、デバッグが辛くなった。

3 回目でようやく安定した。`monitor_checks` (全チェック記録) → `incidents` (障害集約) → `notification_logs` (送信履歴) という 3 層構造にした。最初からこの設計にしていれば 1 週間は節約できた。

### 2. 通知チャンネル拡張が想定より大変だった

最初は Slack と Email だけのつもりだった。「どうせ個人開発者は Slack か LINE でしょ」と思っていた。

しかし実装を進めると「PagerDuty 対応してほしい」という声を想定し始め、結果的に 10 チャンネルに膨らんだ。

問題は各チャンネルの認証フローがバラバラなことだ。Slack は OAuth、PagerDuty は API キー、LINE は Webhook URL — それぞれ異なる検証ロジックが必要だった。「共通インターフェースを定義する」という当たり前の設計判断を最初にサボったツケが、後から来た。

```typescript
// 最終的に落ち着いたインターフェース
interface NotificationChannel {
  type: ChannelType;
  send(alert: Alert): Promise<SendResult>;
  validate(): Promise<ValidationResult>;
}
```

このインターフェースを最初に定義していれば、10 チャンネルの実装はテンプレート化できた。

### 3. Cloudflare Workers.dev ルーティングバグ

ある日、Worker から Worker を呼ぶ内部 API が断続的に 404 を返すようになった。10 時間半にわたって 3 つのインシデントが同時発生。

調べると `*.workers.dev` サブドメイン経由の Worker-to-Worker 呼び出しで Cloudflare のエッジルーティングにバグがあった。カスタムドメイン経由なら 200、`workers.dev` 経由だと 404。

**対策**: Fly.io 上にチェッカープロキシを立て、SSL / ドメイン監視は Cloudflare ネットワーク外から実行するフォールバックを実装した。100% Cloudflare と言いつつ、1 コンポーネントだけ外に出ている。これが現実。

### 4. LP と実装のズレ

ランディングページのプラン比較表に「IP 許可リスト: Coming Soon」と書いてあった。実装を確認したら、もう実装済みだった。逆に、Free プランの通知チャンネル種別制限が LP に反映されていなかった。

LP と実装が乖離するのは個人開発あるあるだけど、SaaS の場合はユーザーの購買判断に直結するので致命的。これ以降、プラン関連の変更は LP とコードを同じ PR で更新するルールにした。

### 5. 価格設定がわからない問題

一番悩んだのは技術じゃなくて価格だった。Free / Pro / Business の 3 段階にしたけど、Pro をいくらにするかで 1 週間は悩んだ。

結論としては「個人開発者が月のランチ 1 回分で使える価格」を目安にした。正解かどうかはユーザーが教えてくれる。

## Claude Code との協働

自分一人で 4 週間、2,000 commits。これは Claude Code なしでは不可能だった。コードレビューの話は別途書いたのでこちらも → [一人で書いたコード、誰かに「大丈夫」と言ってほしかった](https://zenn.dev/naoto24kawa/articles/manako-solo-code-review)

### 助かったこと

- **コードレビュー**: 一人で書いたコードを「大丈夫」と言ってもらえる安心感。実際に存在しない API を自信満々に提案されたこともあるが、設計判断を任せなければ問題ない
- **MCP Server の自己回帰**: Manako の MCP Server 自体を Claude Code で開発し、開発中に Claude Code から Manako を操作してデバッグする — この自己回帰的なループが生産性を上げた
- **リファクタリング**: 24 テーブル、233 ユニットテスト。手動でリネームや型変更をやっていたら何日もかかる作業が数時間で終わる

### 気をつけていること

- **意思決定者は自分**: AI はレビュアー。大きな設計判断 (D1 を選ぶか、Analytics Engine を使うか、Stripe の価格体系をどうするか) は全部自分で決めた
- **完了宣言を疑う**: Claude Code が「完了しました」と言ったとき、本当に完了しているか検証する習慣をつけた。Read せずに Edit する、テストを自己申告する — こういうパターンを知っておくと事故が減る
- **CLAUDE.md にガードレールを書く**: プロジェクト固有のルール (tenant isolation の WHERE 句必須、D1 の FK 制約回避手順) を明文化しておくと、AI がそれを守ってくれる

## 現在の構成

リリース時点の本番構成:

- **Cloudflare Workers**: 8 個 (API, モニターワーカー, 通知キュー, Heartbeat 受信, ステータスページ, クリーンアップ, MCP Server, テストターゲット)
- **Cloudflare Pages**: 4 個 (ダッシュボード, 管理画面, ドキュメント, LP)
- **外部**: Fly.io チェッカープロキシ 1 個
- **テスト**: ユニット 233+、E2E 55 シナリオ
- **インフラ費**: 月 $5〜$10

## これから

リリースから 7 日が経った。正直に言うと、ユーザー数はまだ少ない。

フォロワー 0 のアカウントから始めた個人開発の広報は、想像以上に孤独だ。記事を書いて X で告知しても、初動のいいねは 2〜3。それでも書き続ける理由は、「作った」という事実が一番強い広報だと信じているから。

次にやることは明確だ。

- ユーザーの声を聞いて機能を磨く
- Browser Rendering API で Web 変更検知を強化する
- 英語圏向けに Product Hunt / Hacker News でローンチする

監視は地味だけど、サービスが動いている限り必要なもの。個人開発者が「監視、面倒だな」と思ったときに、Manako を思い出してもらえたら嬉しい。

作ったものを使ってもらうためには、まず知ってもらわないといけない。この記事がその最初の一歩になれば。

Free プラン (クレカ不要) で始められる。

https://manako.dev
