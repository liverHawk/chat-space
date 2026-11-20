# イベント用雑談チャットシステム 仕様書

## 1. システム概要

イベント参加者向けのリアルタイム雑談チャットWebアプリケーション。
**全体チャット（1）＋各団体限定チャット（N）の「1+N」構成**を提供し、Discordアカウントによるログインおよび表示名の自動同期を行う。

フロントエンドのビルド環境やSPAフレームワーク（React等）を排除し、**Rust（Axum）によるサーバーサイドレンダリング（SSR）＋ HTMX ＋ SSE** による単一バイナリ・単一クレート構成で軽量かつ高信頼に運用することを目的とする。

---

## 2. 技術スタック

| レイヤー | 採用技術 / クレート | 用途 |
| --- | --- | --- |
| **言語** | Rust | バックエンド・HTML生成・運用CLIのすべてを記述 |
| **Webフレームワーク** | `axum` / `tokio` | HTTPルーティング、SSE配信、型安全な認可（Extractor） |
| **HTMLテンプレート** | `maud` | Rustマクロによる型安全・自動エスケープ付きHTML生成 |
| **フロントエンド制御** | HTMX（+ `sse` 拡張） | CDN読み込み。JSレスでのフォーム送信とSSE DOM差し替え |
| **スタイリング** | Pico CSS（または Tailwind Standalone） | ビルドツール不要の軽量CSS |
| **リアルタイム配信** | SSE + `tokio::sync::broadcast` + `DashMap` | 部屋単位のメモリ内Pub/Sub（Redis不要） |
| **データベース** | SQLite（WALモード）+ `sqlx` | 永続化・マイグレーション・コンパイル時SQL検証 |
| **認証** | Discord OAuth2（`oauth2` + `reqwest`） | ログイン認証・プロフィール同期 |
| **セッション管理** | `tower-sessions` + `tower-sessions-sqlx-store` | HttpOnly Cookie認証・SQLite永続化 |
| **リバースプロキシ（本番）** | Caddy または Nginx | 自動HTTPS化・HTTP/2配信（SSE同時接続数制限の回避） |

---

## 3. 機能要件

### 3.1 認証・ユーザー同期機能

* **Discord OAuth2 ログイン**
* スコープは `identify` を基本とする（Discordロール連動を採用する場合は `guilds.members.read` を追加）。
* 認可リクエスト時にランダムな `state` を生成してセッションに保存し、コールバック時に検証してCSRFを防止する。


* **表示名・アイコンの同期**
* コールバック時に Discord API (`GET /users/@me`) から `id`, `global_name`（未設定時は `username`）, `avatar` を取得し、`users` テーブルへ `UPSERT` する。
* イベント中の名前変更を反映するため、再ログイン時または画面上の「プロフィール再同期」操作で最新情報を上書きできる。


* **ログアウト**
* セッションレコードを破棄し、ログイン画面へリダイレクトする。



### 3.2 ルーム管理・アクセス制御（1+N構成）

* **全体チャット（1）**
* `is_global = true` のルーム。ログイン済みの全ユーザーが閲覧・投稿可能。


* **団体チャット（N）**
* `is_global = false` のルーム。`room_memberships` テーブルに紐づくメンバーのみ閲覧・投稿・SSE購読が可能。


* **団体への参加方式（招待コード方式を基本設計とする）**
* 各団体ルームに一意の `invite_code` を発行し、招待URL（`/join/:invite_code`）を配布する。
* ログイン済みユーザーがURLにアクセスすると `room_memberships` に登録され、該当ルームへ遷移する。
* 未ログインユーザーがアクセスした場合は、遷移先URLをセッションに保存してDiscordログインへ誘導し、認証完了後に自動参加・遷移させる。
* *拡張オプション（Discordロール連動）*: `rooms.discord_role_id` を設定し、ログイン時にイベント用サーバーの所属ロールを取得して `room_memberships` を自動同期する。



### 3.3 リアルタイムチャット機能

* **過去ログ表示**
* ルーム表示時、直近のメッセージ（例：最新50〜100件）をSQLiteから取得して初期レンダリングする。


* **メッセージ投稿**
* フォームから `POST /rooms/:id/messages` で送信する。
* サーバー側で空文字・最大文字数（例：500文字）をバリデーションし、`messages` テーブルへ保存する。
* 保存後、`maud` でレンダリングしたメッセージHTML断片（`<li>...</li>`）を該当ルームの `broadcast::Sender` へ送信する。


* **リアルタイム受信（SSE）**
* クライアントは `GET /rooms/:id/stream` にSSE接続する。
* サーバーは `message` イベントとしてHTML断片をPushし、HTMXの `sse-swap="message"` + `hx-swap="beforeend"` でチャット欄末尾に自動追記する。
* Keep-Aliveパケットを定期送信し、プロキシ等によるタイムアウト切断を防ぐ。



### 3.4 運用・管理機能（CLI）

* 単一クレート内の別バイナリ（`src/bin/admin.rs`）として管理用CLIを提供する。
* 団体ルームの作成および招待コードの発行・再発行
* ルーム一覧・メンバー数の確認
* イベント終了後のメッセージログのエクスポート（CSV / JSON）



---

## 4. データベース設計（SQLite）

起動時に `PRAGMA foreign_keys = ON;` および `PRAGMA journal_mode = WAL;` を有効化する。

```sql
-- ユーザー情報（Discordから同期）
CREATE TABLE IF NOT EXISTS users (
    id TEXT PRIMARY KEY,                          -- Discord User ID
    name TEXT NOT NULL,                           -- global_name or username
    avatar_url TEXT,
    updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- チャットルーム（全体1 + 団体N）
CREATE TABLE IF NOT EXISTS rooms (
    id TEXT PRIMARY KEY,                          -- 例: 'global', 'team-a'
    name TEXT NOT NULL,                           -- 表示名
    is_global BOOLEAN NOT NULL DEFAULT 0,         -- 1: 全体チャット, 0: 団体限定
    invite_code TEXT UNIQUE,                      -- 招待コード
    discord_role_id TEXT UNIQUE                   -- ロール連動用（任意）
);

-- 団体ルーム所属（認可判定の唯一のソース）
CREATE TABLE IF NOT EXISTS room_memberships (
    user_id TEXT NOT NULL,
    room_id TEXT NOT NULL,
    PRIMARY KEY (user_id, room_id),
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (room_id) REFERENCES rooms(id) ON DELETE CASCADE
);

-- メッセージ履歴
CREATE TABLE IF NOT EXISTS messages (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    room_id TEXT NOT NULL,
    user_id TEXT NOT NULL,
    body TEXT NOT NULL,
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (room_id) REFERENCES rooms(id) ON DELETE CASCADE,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

CREATE INDEX IF NOT EXISTS idx_messages_room_id ON messages(room_id, id DESC);

```

※ セッションテーブルは `tower-sessions-sqlx-store` のマイグレーションにより自動生成される。

---

## 5. エンドポイント仕様

認可はAxumのカスタムExtractor（`AuthUser`, `RoomMember`）で強制する。

| Method | Path | 認可 | 処理概要 |
| --- | --- | --- | --- |
| `GET` | `/` | なし | ログイン済みなら `/rooms/global` へリダイレクト。未ログインならログイン画面を表示 |
| `GET` | `/auth/discord` | なし | `state` をセッション保存し、Discord認可URLへリダイレクト |
| `GET` | `/auth/callback` | なし | `state` 検証 → トークン交換 → `users` へUPSERT → セッション発行 → リダイレクト |
| `POST` | `/auth/logout` | `AuthUser` | セッション破棄 → `/` へリダイレクト |
| `GET` | `/join/:invite_code` | `AuthUser` | 招待コード検証 → `room_memberships` へ登録 → 対象ルームへリダイレクト（※未ログイン時はログイン後に復帰） |
| `GET` | `/rooms/:id` | `RoomMember` | サイドバー（全体＋所属団体一覧）と対象ルームのチャット画面をHTMLで返却 |
| `POST` | `/rooms/:id/messages` | `RoomMember` | メッセージ検証・DB保存・該当ルームのSSEチャネルへHTML断片をブロードキャスト（`200 OK`） |
| `GET` | `/rooms/:id/stream` | `RoomMember` | 該当ルームの `broadcast::Receiver` を購読し、SSEストリームを返却 |

---

## 6. リアルタイム通信・UI仕様

### 6.1 画面レイアウト

* **サイドバー**: 全体チャット（固定）＋ 自分が所属している団体チャット一覧、および自身のDiscordアイコン・表示名・ログアウトボタン。
* **メインエリア**: ルーム名ヘッダー、メッセージ一覧（`<ul>`）、下部に固定されたメッセージ送信フォーム。

### 6.2 HTMX + SSE 連携仕様

* 受信コンテナに `hx-ext="sse" sse-connect="/rooms/{id}/stream"` を設定し、リスト要素に `sse-swap="message" hx-swap="beforeend"` を付与する。
* 送信フォームに `hx-post="/rooms/{id}/messages" hx-swap="none" hx-on::after-request="this.reset()"` を付与し、送信完了後に入力欄をクリアする。
* **SSE改行対策**: SSEの `data:` 仕様によるイベント分断を防ぐため、配信するHTML断片は改行を含まない1行文字列とし、メッセージ本文内の改行は `<br>` に変換するかスペースへ置換する。

---

## 7. 非機能要件・セキュリティ

* **XSS対策**: `maud` の自動HTMLエスケープを徹底し、`PreEscaped` は信頼できるHTML構造以外に使用しない。
* **認可漏れ防止**: `GET /rooms/:id`, `POST /rooms/:id/messages`, `GET /rooms/:id/stream` の3エンドポイントすべてに `RoomMember` Extractor を適用する。
* **セッション保護**: Cookieには `HttpOnly`, `SameSite=Lax`, 本番環境では `Secure` 属性を付与する。
* **同時接続数対策**: HTTP/1.1のブラウザ同一オリジン接続制限（6本）を回避するため、本番環境ではリバースプロキシ（Caddy / Nginx）で **HTTP/2 以上** を有効にする。
* **メッセージスパム対策**: 必要に応じて `tower_governor` 等で `POST /rooms/:id/messages` にユーザー単位のレートリミット（例：5秒間に3回まで）を設ける。

---

## 8. プロジェクト構成・環境変数

### 8.1 ディレクトリ構成（単一クレート）

```text
event-chat/
├── Cargo.toml
├── .env
├── migrations/
│   └── 0001_init.sql
└── src/
    ├── main.rs             # サーバー起動・ルーター構築
    ├── state.rs            # AppState (SqlitePool, DashMap, OAuth2Client)
    ├── auth.rs             # Discord OAuth2処理・AuthUser / RoomMember Extractor
    ├── views.rs            # Maud HTMLテンプレート
    ├── handlers/
    │   ├── mod.rs
    │   ├── rooms.rs        # ルーム表示・招待参加
    │   └── messages.rs     # 投稿・SSE配信
    └── bin/
        └── admin.rs        # ルーム作成・招待コード発行・ログ出力用CLI

```

### 8.2 環境変数（`.env`）

| 変数名 | 必須 | 説明 |
| --- | --- | --- |
| `DATABASE_URL` | 必須 | SQLite接続文字列（例：`sqlite:chat.db`） |
| `DISCORD_CLIENT_ID` | 必須 | Discord Developer Portal の Client ID |
| `DISCORD_CLIENT_SECRET` | 必須 | Discord Developer Portal の Client Secret |
| `DISCORD_REDIRECT_URL` | 必須 | コールバックURL（例：`[https://chat.example.com/auth/callback](https://chat.example.com/auth/callback)`） |
| `BIND_ADDR` | 任意 | リッスンアドレス（デフォルト：`127.0.0.1:3000`） |