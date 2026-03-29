# データベース タスク一覧

ステータス: 🔲 未着手 / 🔄 進行中 / ✅ 完了

## Phase 1: 環境構築

- 🔲 DB-001: Supabase プロジェクト作成 (無料枠)
- 🔲 DB-002: Supabase CLI セットアップ
- 🔲 DB-003: `db/supabase/config.toml` 設定
- 🔲 DB-004: ローカル開発用 Supabase 環境起動確認 (`supabase start`)

## Phase 2: スキーマ実装

- 🔲 DB-011: PostGIS 拡張の有効化 (`CREATE EXTENSION postgis`)
- 🔲 DB-012: `001_initial_schema.sql` マイグレーション作成 (`media_items`)
  - テスト: マイグレーションの冪等性確認
- 🔲 DB-013: `002_add_face_clusters.sql` マイグレーション作成
- 🔲 DB-014: `003_add_backup_cursors.sql` マイグレーション作成
- 🔲 DB-015: `004_add_tag_vocabulary_and_scores.sql` マイグレーション作成 + シードデータ投入
- 🔲 DB-016: RLS ポリシー設定

## Phase 3: インデックス・パフォーマンス

- 🔲 DB-021: GINインデックス追加 (object_tags, scene_tags)
- 🔲 DB-022: テストデータ (`seeds/test_data.sql`) 作成 (10万件)
- 🔲 DB-023: 検索クエリのパフォーマンス確認 (EXPLAIN ANALYZE)

## Phase 4: AWS Parameter Store 連携

- 🔲 DB-031: Supabase 接続情報を AWS Parameter Store に登録
- 🔲 DB-032: MacアプリからのParameter Store取得テスト
- 🔲 DB-033: OMV APIサーバーからの Parameter Store 取得テスト (`/tmp/aws/credential.json` 経由)

## Phase 5: Claude Haiku 統合テスト

- 🔲 DB-041: スキーマ情報をClaude Haikuのプロンプトに埋め込む形式を決定
- 🔲 DB-042: 典型的な自然言語クエリに対するSQL変換精度の評価
- 🔲 DB-043: 不正なSQLインジェクションが生成されないことの確認
