# Macアプリ 設計文書

## 技術スタック

- **言語**: Swift 6
- **UI**: SwiftUI + AppKit (メニューバー)
- **フレームワーク**: PhotosKit, Vision, Core ML, Network.framework
- **認証**: Auth0 Swift SDK
- **AWS**: AWS SDK for Swift (Parameter Store)

## アーキテクチャ

```
NASPhotoMac
├── App
│   ├── NASPhotoMacApp.swift        # エントリーポイント
│   └── MenuBarView.swift           # メニューバーUI
├── Features
│   ├── Backup
│   │   ├── BackupCoordinator.swift  # バックアップ全体制御
│   │   ├── PhotosLibraryWatcher.swift # Photos.app変更検知
│   │   ├── BackupStateStore.swift   # バックアップ済み管理 (UserDefaults + 最終日時)
│   │   └── TransferClient.swift    # OMV APIクライアント (UDP/HTTP)
│   ├── Metadata
│   │   ├── MetadataExtractor.swift  # EXIF抽出
│   │   ├── VisionAnalyzer.swift    # Vision framework AI解析
│   │   └── MetadataSupplementor.swift # スキャン写真補完
│   └── Settings
│       └── SettingsView.swift
├── Infrastructure
│   ├── SupabaseClient.swift        # Supabase REST API
│   ├── AWSParameterStore.swift    # シークレット取得
│   └── Auth0Client.swift          # 認証
└── Tests
    ├── Unit/
    └── Property/                  # SwiftCheck使用
```

## バックアップフロー

```
1. PhotosLibraryWatcher が Photos.app の変更を検知 (PHPhotoLibraryChangeObserver)
2. 新規/更新アセットを取得
3. BackupStateStore で「最終バックアップ取り込み日時」以降のアセットを抽出
4. TransferClient でOMV APIへ転送 (大ファイルは分割)
5. 転送成功後にメタデータ解析キューに追加
6. MetadataExtractor + VisionAnalyzer で解析
7. SupabaseClient でメタデータ保存
8. BackupStateStore の最終日時を更新
```

## メタデータスキーマ (Supabase)

`docs/db/design.md` 参照。

## シークレット管理

`AWSParameterStore.swift` が AWS Parameter Store からシークレットを取得する。

### クレデンシャル

macOS アプリは環境のデフォルト認証情報（`~/.aws/credentials` または IAM ロール）を使用する。AWS SDK for Swift のデフォルトクレデンシャルチェーンに任せる。

### Parameter Store パス

リージョン: `ap-northeast-1`。omvapi と同じパスを参照する。

| パス | 取得する値 |
|---|---|
| `/auth0/nasphoto` | `domain`, `clientId`, `clientSecret` |
| `/nasphoto/supabase` | `url`, `key` |

※ macOS アプリは Anthropic / Apple Maps には直接アクセスしない（omvapi 経由）。

### 実装方針 (`Infrastructure/AWSParameterStore.swift`)

- `actor AWSParameterStore` として実装
- アプリ起動時に一度だけ取得し、メモリに保持する
- AWS SDK for Swift (`AWSSSM`) の `SSMClient` を使用

```swift
struct AppSecrets {
    let supabaseURL: String
    let supabaseKey: String
    let auth0Domain: String
    let auth0ClientID: String
    let auth0ClientSecret: String
}
```

## Auth0 統合

- macOSアプリはDevice Authorization Flowで初回認証
- トークンをKeychain に保存
- トークン失効時は自動リフレッシュ

## VisionAnalyzer タグ生成

`VisionAnalyzer.swift` は `VNClassifyImageRequest` を実行し、以下の処理でタグとスコアを生成する:

1. アプリ起動時に Supabase の `tag_vocabulary` テーブルを取得してメモリにキャッシュする
2. 各アセットに対して `VNClassifyImageRequest` を実行
3. 信頼スコア 0.5 以上の `VNClassificationObservation` を抽出
4. キャッシュした語彙に含まれるラベルのみを採用し、`tag_type` で object / scene に振り分ける
5. タグ名配列を `object_tags` / `scene_tags` に、`{タグ名: スコア}` を `object_tag_scores` / `scene_tag_scores` に格納して Supabase へ保存する

語彙の追加・削除は `tag_vocabulary` テーブルへのマイグレーションで行う。コード変更は不要。

## エラーハンドリング

| エラー種別 | 対応 |
|---|---|
| ネットワーク断絶 | 指数バックオフでリトライ (最大32秒間隔) |
| Supabase APIエラー | ローカルキューに保存し後でリトライ |
| Vision解析失敗 | ログ記録後にスキップ (バックアップは続行) |
| Auth0トークン失効 | 自動リフレッシュ、失敗時はユーザー通知 |
