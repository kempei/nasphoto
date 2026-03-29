# CLAUDE.md — NASPhoto 開発ガイド

このファイルはClaudeがこのリポジトリで作業する際の指針です。

## プロジェクト概要

個人写真管理システム。以下4コンポーネントで構成される:
1. **NasPhotos.app** (`macos/`) — バックアップ & メタデータ解析。写真.app (Photos.app) に取り込まれた写真・動画を PhotosKit で読み取り、Vision・Core ML で解析後に OMV NAS へ転送する Mac mini 常駐アプリ
2. **OMV API サーバー** (`omvapi/`) — 軽量Go APIサーバー
3. **iOS アプリ** (`ios/`) — ビューアー & 検索
4. **Supabase DB** (`db/`) — スキーマ & マイグレーション

詳細は `ideaseed.md` と `docs/` 配下の要件/デザイン文書を参照。

## 開発方針

### テスト戦略
- **プロパティテスト必須**: 各コンポーネントにプロパティベーステスト (Swift: SwiftCheck, Go: gopter または testing/quick) を含める
- ユニットテスト → プロパティテスト → 統合テストの順に実装
- 外部サービス (Supabase, Auth0, AWS) はモック/スタブで分離
- 小さく動作確認しながら進める (大きなリファクタはしない)

### コーディング規約
- Swift: Swift 6 strict concurrency, `async/await` を基本とする
- Go: 標準的なGoのイディオムに従う, エラーは `errors.As/Is` で処理
- SQL: Supabase マイグレーションは `db/migrations/` に連番で管理
- シークレットはコードにハードコードしない (AWS Parameter Store / 環境変数)

### コンポーネント間の依存関係
- iOSアプリはSupabaseに直接アクセスしない → 必ずOMV APIサーバー経由 (OMV APIがクエリを中継)
- MacアプリはSupabaseに直接アクセスしてよい。ファイル転送はOMV APIサーバー経由
- Mac/OMV APIのみAWS Parameter Storeに直接アクセス
- 認証はAuth0を通じて行う

## 各コンポーネントの開始方法

### macOS アプリ
```bash
cd macos
# Xcode でプロジェクトを開く
open NASPhotoMac.xcodeproj
```

### OMV API サーバー
```bash
cd omvapi
go test ./...
go run ./cmd/server
```

### iOS アプリ
```bash
cd ios
open NASPhotoiOS.xcodeproj
```

### DB マイグレーション
```bash
cd db
# Supabase CLI を使用
supabase db push
```

## ドキュメント構成

```
docs/
├── macos/     要件・デザイン・タスク
├── omvapi/    要件・デザイン・タスク
├── ios/       要件・デザイン・タスク
└── db/        要件・デザイン・タスク
```

変更を加えた場合は対応する文書を更新すること。

## 文書の一貫性ルール

- アーキテクチャや設計方針に変更が生じた場合、**必ず関連するすべての文書を同時に更新する**。一部だけ更新して矛盾を生じさせてはならない
- 更新対象の確認順: `README.md` → `CLAUDE.md` → `docs/<component>/requirements.md` → `docs/<component>/design.md` → `docs/<component>/tasks.md`
- コンポーネント間の依存関係 (誰が何にアクセスするか) を変更した場合は、**両側のコンポーネントの文書と README.md の図を同時に更新する**
- 文書を編集する前に、変更が他の文書の記述と矛盾しないかを確認する
- **`tasks.md` の同期は指示がなくても行う**: `design.md` や `requirements.md` を更新した場合は、必ず対応する `tasks.md` も同じタイミングで更新する。指示を待たない

## 実装着手のルール

- **`tasks.md` の更新が完了するまで実装コードを書き始めない**。設計変更があった場合は `requirements.md` → `design.md` → `tasks.md` の順で文書を揃えてから実装に入る
- ユーザーから実装の指示を受けた場合も、まず関連文書が最新かどうかを確認し、不足があれば先に文書を更新する

## 重要な制約

- Supabase は**無料枠**で運用: ストレージ容量・API呼び出し数を節約する設計にする
- 同時接続数は**通常1、最大2程度**を前提としてよい (個人利用)
- NASのパフォーマンスを最大限引き出すためにUDP/非同期転送を活用
- iPhoneはAWSに直接アクセスしない
