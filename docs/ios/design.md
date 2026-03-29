# iPhoneアプリ 設計文書

## 技術スタック

- **言語**: Swift 6
- **UI**: SwiftUI
- **状態管理**: `@Observable` (Swift 5.9+)
- **キャッシュ**: NSCache + ディスクキャッシュ (独自実装)
- **認証**: Auth0 Swift SDK
- **動画**: AVFoundation

## ディレクトリ構成

```
NASPhotoiOS/
├── App/
│   └── NASPhotoApp.swift
├── Features/
│   ├── Gallery/
│   │   ├── GalleryView.swift           # グリッド一覧
│   │   ├── GalleryViewModel.swift
│   │   └── ThumbnailCell.swift
│   ├── Detail/
│   │   ├── PhotoDetailView.swift       # 詳細表示
│   │   └── VideoPlayerView.swift
│   ├── Search/
│   │   ├── SearchView.swift
│   │   └── SearchViewModel.swift       # キーワード+自然言語検索
│   └── Auth/
│       └── LoginView.swift
├── Infrastructure/
│   ├── APIClient.swift                 # OMV APIクライアント
│   ├── ThumbnailCache.swift            # NSCache + ディスクキャッシュ
│   └── Auth0Client.swift
└── Tests/
    ├── Unit/
    └── Property/                       # SwiftCheck使用
```

## キャッシュ設計

```swift
// 2段階キャッシュ
struct ThumbnailCache {
    // L1: メモリキャッシュ (NSCache, 自動削除)
    private let memoryCache: NSCache<NSString, UIImage>
    // L2: ディスクキャッシュ (最大500MB, LRU削除)
    private let diskCache: DiskCache
}
```

## 自然言語検索フロー

```
1. ユーザーが自然言語テキスト入力
2. POST /v1/query/natural へ送信
3. APIサーバーがClaude Haikuに [スキーマ情報 + クエリ] を送信
4. Claude HaikuがSQLクエリを生成
5. APIサーバーがSupabaseでSQLを実行
6. 結果をiPhoneに返却
7. GalleryViewで結果表示
```

## Auth0 統合

- Authorization Code Flow with PKCE
- アクセストークンをKeychainに保存
- 全APIリクエストに `Authorization: Bearer <token>` を付与
