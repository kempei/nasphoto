# OMV APIサーバー タスク一覧

ステータス: 🔲 未着手 / 🔄 進行中 / ✅ 完了

## Phase 1: プロジェクト基盤

- 🔲 API-001: Goモジュール初期化 (`go mod init`)
- 🔲 API-002: ディレクトリ構成作成
- 🔲 API-003: プロパティテストライブラリ導入 (gopter または testing/quick)
- 🔲 API-004: 基本的なHTTPサーバー起動確認 + `/health` エンドポイント
- 🔲 API-005: systemdサービスファイル作成

## Phase 2: 認証

- 🔲 API-011: Auth0 JWT 検証ミドルウェア実装
  - テスト: 有効・無効・期限切れトークンのテスト
  - プロパティテスト: 任意のJWTペイロードで検証が正しく動作すること
- 🔲 API-012: ロールベース認可 (Mac vs iPhone) 実装
- 🔲 API-013: AWS Parameter Store からのシークレット取得実装
  - `/tmp/aws/credential.json` を読み込む `fileCredentialsProvider` 実装 (有効期限切れ時自動リフレッシュ)
  - Parameter Store から `/auth0/nasphoto`・`/nasphoto/supabase` 等を一括取得
  - テスト: credential ファイルのモックで取得・リフレッシュ動作を確認
- 🔲 API-014: Parameter Store パラメータの初期登録 (AWS コンソールまたは CLI)

## Phase 3: ファイル転送

- 🔲 API-021: HTTP マルチパートアップロード実装
- 🔲 API-022: チャンク分割受信・結合実装
  - プロパティテスト: 任意チャンクサイズで分割→結合が冪等
- 🔲 API-023: チェックサム検証実装
- 🔲 API-024: UDPカスタムプロトコル実装
- 🔲 API-025: 欠落チャンク選択的再送実装
- 🔲 API-026: Range リクエスト対応ファイル配信

## Phase 4: サムネイル

- 🔲 API-031: サムネイル生成 (libvips または純Go実装)
- 🔲 API-032: サムネイルキャッシュ実装

## Phase 5: Supabase連携

- 🔲 API-041: Supabase REST APIクライアント実装
- 🔲 API-042: クエリ中継エンドポイント実装 (`/v1/query`)
- 🔲 API-043: Claude Haiku統合・プロンプト構築実装 (`/v1/query/natural`)
  - テスト: 既知の自然言語クエリに対するSQL変換の確認
- 🔲 API-044: 生成SQLバリデーション実装
  - テスト: SELECT以外のクエリ・危険なキーワードが拒否されること
  - プロパティテスト: 任意の生成SQLに対してバリデーションが副作用を起こさないこと

## Phase 6: 統合テスト

- 🔲 API-051: 全エンドポイントのHTTP統合テスト (httptest使用)
- 🔲 API-052: Mac/iPhoneモッククライアントによるE2Eテスト
