# iPhoneアプリ タスク一覧

ステータス: 🔲 未着手 / 🔄 進行中 / ✅ 完了

## Phase 1: プロジェクト基盤

- 🔲 IOS-001: Xcodeプロジェクト作成 (Swift 6, iOS 17+)
- 🔲 IOS-002: SwiftCheck (プロパティテスト) 導入
- 🔲 IOS-003: SwiftPackage依存関係 (Auth0 SDK)
- 🔲 IOS-004: OMV APIクライアント骨格実装

## Phase 2: 認証

- 🔲 IOS-011: Auth0 Authorization Code Flow (PKCE) 実装
- 🔲 IOS-012: Keychain へのトークン保存実装
- 🔲 IOS-013: ログイン画面UI

## Phase 3: ギャラリー表示

- 🔲 IOS-021: グリッドビュー実装 (LazyVGrid)
- 🔲 IOS-022: サムネイルキャッシュ実装 (L1メモリ + L2ディスク)
  - プロパティテスト: キャッシュ容量制限を超えないこと
  - プロパティテスト: LRU削除ポリシーの正確性
- 🔲 IOS-023: 無限スクロール・ページネーション
- 🔲 IOS-024: 日付グルーピング

## Phase 4: 詳細表示

- 🔲 IOS-031: フル解像度表示 (ピンチズーム)
- 🔲 IOS-032: 動画再生 (AVFoundation)
- 🔲 IOS-033: EXIFメタデータ・AIタグ表示

## Phase 5: 検索

- 🔲 IOS-041: キーワード検索UI + APIクライアント実装
- 🔲 IOS-042: 自然言語検索UI実装
  - テスト: 様々なクエリでクラッシュしないこと
- 🔲 IOS-043: 検索結果のリアルタイム更新

## Phase 6: 統合テスト

- 🔲 IOS-051: モックAPIに対するUIテスト (XCUITest)
- 🔲 IOS-052: Snapshot テスト (主要画面)
