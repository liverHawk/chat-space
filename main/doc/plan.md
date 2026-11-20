# イベント用雑談チャットシステム (event-chat) 実装計画

本書は [init.md](init.md) および要件ヒアリングで合意した設計・実装・テスト計画をまとめたものです。

## 1. 確定仕様・設計方針

1. **クレート・バイナリ構成 (`Cargo.toml`)**
   - パッケージ名 `event-chat`（`default-run = "event-chat"`）、Webサーバー（`src/main.rs`）と管理CLI（`src/bin/admin.rs`）の2バイナリ構成とする。
   - 共通モジュール（`state`, `auth`, `views`, `handlers`）を `src/lib.rs` から公開し、サーバー・CLI・統合テストで共有する。
2. **DB・マイグレーション (`migrations/20261120040811_init.sql`)**
   - SQLite接続時に `create_if_missing(true)`、`PRAGMA foreign_keys = ON;`、`PRAGMA journal_mode = WAL;` を有効化する。
   - `users`, `rooms`, `room_memberships`, `messages` テーブルと `idx_messages_room_id` を定義する（`rooms.discord_role_id` も保持）。
   - 全体チャット（`id = 'global'`, `name = '全体チャット'`, `is_global = 1`）を `INSERT OR IGNORE` で初期投入する。
3. **認証・セッション・プロフィール再同期 (`src/auth.rs`)**
   - `tower-sessions` + `tower-sessions-sqlx-store`（`SqliteStore`）で管理し、Cookieは `HttpOnly`, `SameSite=Lax`（本番HTTPS時は `Secure`）とする。
   - **Discord OAuth2 (`identify` スコープ)**:
     - `GET /auth/discord`: `state` をセッション保存し認可URLへ遷移。サイドバーの「プロフィール再同期」もここへ遷移して最新情報を再取得する。
     - `GET /auth/callback`: `state` 検証 → トークン交換 → `GET /users/@me` 取得 → `users` へ `UPSERT` → `user_id` 保存 → `next_url` または `/rooms/global` へ遷移。
     - `POST /auth/logout`: `AuthUser` 認可でセッション破棄し `/` へ遷移。
   - **認可Extractor**:
     - `AuthUser`: セッション検証。未ログインの `/join/:invite_code` アクセス時は `next_url` を保存して `/auth/discord` へ誘導する。
     - `RoomMember`: ルーム存在確認（404）および非全体ルームでの `room_memberships` 所属確認（403）を `GET /rooms/:id`, `POST /rooms/:id/messages`, `GET /rooms/:id/stream` に強制する。
4. **チャット画面・SSE・レートリミット (`src/views.rs`, `src/handlers/`, `src/state.rs`)**
   - `GET /`（ログイン済なら `/rooms/global` へ、未ログインならログイン画面）、`GET /join/:invite_code`（招待コード検証・参加・リダイレクト）。
   - `GET /rooms/:id`: `maud` + Pico CSS + HTMX（`sse` 拡張）でサイドバー（全体＋所属団体＋再同期・ログアウト）と最新100件・送信フォームを表示する。
   - `POST /rooms/:id/messages`: `DashMap` による `user_id` 単位レートリミット（5秒3回、超過429）、空文字・500文字制限（違反400）、`maud` 自動エスケープ＋改行の `<br>` 変換＋改行コード完全除去による1行HTML断片のブロードキャストを行う。
   - `GET /rooms/:id/stream`: `broadcast::Receiver` を購読し `event: message` としてSSE配信（Keep-Alive有効）。
5. **管理用CLI (`src/bin/admin.rs`)**
   - `clap` で `create-room`, `reissue-invite`, `list-rooms`, `export-messages`（CSV/JSON）を提供する。

## 2. ディレクトリ・モジュール構成

```text
event-chat/
├── Cargo.toml
├── .env.sample
├── migrations/20261120040811_init.sql
└── src/
    ├── lib.rs            # モジュール公開・build_app ルーター構築
    ├── main.rs           # サーバー起動
    ├── state.rs          # AppState (SqlitePool, DashMap, OAuth2Client, レートリミット)
    ├── auth.rs           # Discord OAuth2 ハンドラ・AuthUser / RoomMember
    ├── views.rs          # Maud HTMLテンプレート・1行SSEメッセージ生成
    ├── handlers/
    │   ├── mod.rs
    │   ├── rooms.rs      # index, join_room, show_room
    │   └── messages.rs   # post_message, stream_messages
    └── bin/admin.rs      # 運用管理CLI
```

## 3. コミット分割計画（1コミット100行以内厳守）

1. **Commit 1**: `.gitignore`・`.env.sample`・`Cargo.toml` の更新
2. **Commit 2**: `migrations/20261120040811_init.sql` の定義
3. **Commit 3**: `src/state.rs`（DB初期化・AppState・チャネル・レートリミット）
4. **Commit 4**: `src/auth.rs` パート1（`AuthUser` / `RoomMember` Extractor）
5. **Commit 5**: `src/auth.rs` パート2（OAuth2・ログアウトハンドラ）
6. **Commit 6**: `src/views.rs` パート1（ログイン画面・1行SSEメッセージ描画）
7. **Commit 7**: `src/views.rs` パート2（チャット画面・サイドバー・HTMXフォーム）
8. **Commit 8**: `src/handlers/mod.rs`・`src/handlers/rooms.rs`
9. **Commit 9**: `src/handlers/messages.rs`（投稿・SSE配信）
10. **Commit 10**: `src/lib.rs`・`src/main.rs`（ルーター構築・起動）
11. **Commit 11**: `src/bin/admin.rs` パート1（CLI定義・ルーム管理3コマンド）
12. **Commit 12**: `src/bin/admin.rs` パート2（`export-messages` 実装）
13. **Commit 13–14**: 統合テスト実装と `cargo test` 検証

## 4. テスト・検証計画

- インメモリSQLiteを用いた統合テストで以下を検証する:
  - `RoomMember` 認可（未参加団体チャットへのアクセス・投稿・SSEは403、参加済・全体チャットは200）
  - 招待コード参加（`/join/:invite_code`）と未ログイン時の `next_url` 誘導
  - メッセージ投稿の空文字/500文字バリデーション（400）とレートリミット（429）
  - XSSエスケープ、本文改行の `<br>` 変換、SSE配信用1行HTML保証（`\r`/`\n` なし）
- `cargo check --all-targets`・`cargo test` 通過、および全コミットが100行以内であることを確認する。
