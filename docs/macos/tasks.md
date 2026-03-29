# Macアプリ タスク一覧

ステータス: 🔲 未着手 / 🔄 進行中 / ✅ 完了

## Phase 1: プロジェクト基盤

- 🔲 MAC-001: Xcodeプロジェクト作成 (Swift 6, macOS 14+)
- 🔲 MAC-002: SwiftCheck (プロパティテスト) 導入
- 🔲 MAC-003: SwiftPackage依存関係設定 (Auth0 SDK, AWS SDK for Swift, Supabase Swift)
- 🔲 MAC-004: メニューバーアプリの骨格実装

## Phase 2: バックアップ基盤

- 🔲 MAC-011: PhotosKit でライブラリ変更を監視する `PhotosLibraryWatcher` 実装
  - テスト: PHAsset のモックで変更検知を確認
- 🔲 MAC-012: `BackupStateStore` 実装 (最終取り込み日時の永続化)
  - プロパティテスト: 日時の順序関係が保持されること
- 🔲 MAC-013: OMV APIクライアント HTTP 部分の実装
  - テスト: モックサーバーに対して接続・転送確認
- 🔲 MAC-014: 大容量ファイルの分割転送実装
  - プロパティテスト: 任意サイズのファイルが正しく分割・再構成されること
- 🔲 MAC-015: リトライロジック実装 (指数バックオフ)
  - プロパティテスト: 失敗回数に応じた待機時間が単調増加すること

## Phase 3: メタデータ解析

- 🔲 MAC-021: EXIF抽出 `MetadataExtractor` 実装
  - プロパティテスト: 任意のEXIFバイナリでクラッシュしないこと
- 🔲 MAC-022: Vision framework によるオブジェクト認識実装
- 🔲 MAC-023: シーン分類実装
- 🔲 MAC-024: スキャン写真メタデータ補完UI実装
  - テスト: 前後写真からの日時類推ロジック

## Phase 4: クラウド連携

- 🔲 MAC-031: Supabase クライアント実装 (メタデータ保存)
- 🔲 MAC-032: AWS Parameter Store 統合 (`AWSParameterStore.swift`)
  - デフォルトクレデンシャルチェーンで SSMClient を初期化
  - `/auth0/nasphoto`・`/nasphoto/supabase` を取得して `AppSecrets` に詰める
  - テスト: SSMClient をモックして取得・エラー処理を確認
- 🔲 MAC-033: Auth0 Device Authorization Flow 実装

## Phase 5: UI・設定

- 🔲 MAC-041: メニューバーUI (進捗表示・最終バックアップ日時)
- 🔲 MAC-042: 設定画面 (接続先、除外フォルダ)
- 🔲 MAC-043: macOS起動時の自動起動設定

## Phase 6: 統合テスト

- 🔲 MAC-051: バックアップ→メタデータ解析→Supabase保存の E2E テスト (モック使用)
