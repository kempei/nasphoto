# NASPhoto

個人写真・動画の管理システム。iPhone・ミラーレス一眼で撮影した写真はまず Mac mini の **写真.app (Photos.app)** に取り込まれる。Mac mini 上で常駐する独自アプリ **NasPhotos.app** が写真.app から写真・動画を読み取り、メタデータを解析・補完してSupabaseに保存しつつ、ファイル本体を OMV NAS へ転送する。iPhoneアプリからは OMV API Server 経由で閲覧・検索ができる。

## システム構成

```
[iPhone/カメラ] ──取り込み──→ [写真.app (Photos.app)]
                                        ↓ PhotosKit で読み取り
                               [NasPhotos.app (Mac mini)]
                                ↙ ファイル転送          ↘ メタデータ保存
                       [OMV API Server]              [Supabase (DB)]
                       ↕ ファイル保存/配信                   ↑ クエリ中継
                       [OMV NAS (2TB+)]          [OMV API Server]
                                                        ↑
                                               [iPhone アプリ]
```

## コンポーネント

| ディレクトリ | 役割 |
|---|---|
| `macos/` | **NasPhotos.app** — バックアップ・メタデータ解析 (Swift/SwiftUI) |
| `omvapi/` | OMV上の軽量APIサーバー (Go) |
| `ios/` | iPhone ビューアーアプリ (Swift/SwiftUI) |
| `db/` | Supabase マイグレーション・スキーマ |

## ドキュメント

- [Macアプリ 要件](docs/macos/requirements.md) / [デザイン](docs/macos/design.md) / [タスク](docs/macos/tasks.md)
- [APIサーバー 要件](docs/omvapi/requirements.md) / [デザイン](docs/omvapi/design.md) / [タスク](docs/omvapi/tasks.md)
- [iPhoneアプリ 要件](docs/ios/requirements.md) / [デザイン](docs/ios/design.md) / [タスク](docs/ios/tasks.md)
- [データベース 要件](docs/db/requirements.md) / [デザイン](docs/db/design.md) / [タスク](docs/db/tasks.md)

## 技術スタック

- **Mac/iOSアプリ**: Swift 6, SwiftUI, PhotosKit, Vision framework, Core ML
- **APIサーバー**: Go (軽量HTTP + カスタムUDPプロトコル)
- **DB**: Supabase (PostgreSQL, 無料枠)
- **認証**: Auth0
- **シークレット管理**: AWS Parameter Store
- **LLM**: Claude Haiku (自然言語→SQLクエリ変換)

## 前提条件

- Mac mini (Apple Silicon 推奨) + Photos.app
- OMV (Open Media Vault) NAS: ローカルネットワーク内
- Supabase アカウント (無料枠)
- Auth0 アカウント
- AWS アカウント (Parameter Store)
- Anthropic API キー (Claude Haiku)
