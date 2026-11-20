# AGENTS.md

このリポジトリで作業するAIエージェント向けの行動規範および開発ガイドラインです。

## 1. コミュニケーション・報告ルール

- **使用言語**: ユーザーへの説明・質問・報告はすべて**日本語**で行うこと。
- **逐次報告**: 作業の進捗、実施した内容、次に進めるステップをこまめに逐次ユーザーへ報告すること。

## 2. Git・コミットルール

- **定期的なコミット**: 作業の区切りごとに定期的にコミットを行うこと。
- **1コミットの編集行数上限**: **1コミットあたりの編集行数（追加行＋削除行の合計）が100行を超えないようにする**こと。
  - 変更規模が大きくなる場合は、機能・ファイル・レイヤー単位で細かく分割してコミットする。
  - コミット前に `git diff --cached --stat` 等で変更行数を確認する。
- **コミット対象外**: `.env` やローカルDBファイル（`chat.db` 等）などの機密情報・生成物はコミットしないこと。

## 3. プロジェクト概要・仕様

- **概要**: イベント参加者向けリアルタイム雑談チャットシステム（全体チャット1＋団体チャットNの「1+N」構成）。
- **詳細仕様**: 実装・設計の詳細は必ず [`doc/init.md`](doc/init.md) を参照すること。
- **技術スタック**:
  - 言語 / Web: Rust (`axum` / `tokio`)
  - HTML生成 / フロント制御: `maud` + HTMX（+ `sse` 拡張）+ Pico CSS
  - リアルタイム配信: SSE + `tokio::sync::broadcast` + `DashMap`
  - DB / セッション: SQLite（WALモード）+ `sqlx`、`tower-sessions` + `tower-sessions-sqlx-store`
  - 認証: Discord OAuth2（`oauth2` + `reqwest`）
- **構成**: 単一クレート構成（Webサーバー: `src/main.rs`、運用管理CLI: `src/bin/admin.rs`）。

## 4. 実装・セキュリティ上の重要事項

- **XSS対策**: `maud` の自動HTMLエスケープを徹底し、`PreEscaped` は信頼できるHTML構造以外に使用しない。
- **認可の徹底**: `GET /rooms/:id`、`POST /rooms/:id/messages`、`GET /rooms/:id/stream` のすべてに `RoomMember` Extractor を適用し、認可漏れを防ぐ。
- **SSE改行対策**: SSEで配信するHTML断片には改行を含めず1行文字列とし、本文内の改行は `<br>` へ変換する等してイベント分断を防ぐ。
- **DB設定**: SQLite起動時に `PRAGMA foreign_keys = ON;` および `PRAGMA journal_mode = WAL;` を有効化する。
- **動作確認**: 実装・変更後は可能な範囲で `cargo check` や `cargo test` を実行して問題がないことを確認する。
