# OMV APIサーバー 設計文書

## 技術スタック

- **言語**: Go 1.22+
- **HTTPフレームワーク**: 標準 `net/http` (軽量のため)
- **カスタムプロトコル**: UDP + 独自ヘッダー (大容量転送用)
- **認証**: Auth0 JWT検証 (github.com/auth0/go-jwt-middleware)
- **Supabase連携**: REST API経由

## ディレクトリ構成

```
omvapi/
├── cmd/
│   └── server/
│       └── main.go
├── internal/
│   ├── auth/
│   │   ├── jwt.go          # Auth0 JWT検証
│   │   └── jwt_test.go
│   ├── transfer/
│   │   ├── receiver.go     # ファイル受信 (HTTP multipart + UDP)
│   │   ├── sender.go       # ファイル送信 (Range対応)
│   │   ├── chunker.go      # 分割処理
│   │   └── chunker_test.go
│   ├── thumbnail/
│   │   ├── generator.go    # サムネイル生成
│   │   └── cache.go        # キャッシュ管理
│   ├── proxy/
│   │   └── supabase.go     # Supabaseクエリ中継
│   ├── nlsearch/
│   │   ├── text2sql.go     # Claude Haiku呼び出し・プロンプト構築
│   │   └── validator.go    # 生成SQLのバリデーション (書き込み系・インジェクション禁止)
│   └── secrets/
│       └── aws.go          # AWS Parameter Store からのシークレット取得 (fileCredentialsProvider)
├── api/
│   └── router.go           # ルーティング定義
└── go.mod
```

## APIエンドポイント

| メソッド | パス | 説明 | 認可 |
|---|---|---|---|
| POST | `/v1/upload` | ファイルアップロード | Mac |
| POST | `/v1/upload/chunk` | チャンクアップロード | Mac |
| GET | `/v1/files/{id}` | ファイル取得 | iPhone/Mac |
| GET | `/v1/thumbnails/{id}` | サムネイル取得 | iPhone/Mac |
| POST | `/v1/query` | Supabaseクエリ中継 | iPhone/Mac |
| POST | `/v1/query/natural` | 自然言語クエリ (→Claude Haiku→SQL) | iPhone |
| GET | `/health` | ヘルスチェック | なし |

## Text2SQL 処理フロー

```text
1. POST /v1/query/natural でiPhoneから自然言語クエリを受信
2. 地名を含むか判定し、含む場合はジオコーディング API で座標を取得 (後述)
3. プロンプトを組み立てる (後述のプロンプト設計を参照)
4. Claude Haiku に送信しSQLを生成
5. 生成されたSQLをバリデーション (SELECT以外・危険なキーワードを拒否)
6. Supabase REST APIでSQLを実行
7. 結果をiPhoneに返却
```

### ジオコーディングステップ

クエリに地名・施設名・駅名が含まれる場合、SQL 生成前に座標を解決する。

**使用サービス**: Apple Maps Server API
- 日本語地名・駅名の精度が高い
- 個人利用は無料枠で十分
- シークレットは他の API キーと同様に `/tmp/nasphoto-secrets.json` で管理

**処理フロー**:
```text
「三浦海岸駅周辺で撮影した写真」
  ↓ Claude Haiku で地名を抽出: "三浦海岸駅"
  ↓ Apple Maps Server API: → {lat: 35.1528, lng: 139.6138}
  ↓ プロンプトに座標を付加してSQL生成
  ↓ ST_DWithin(location, ST_MakePoint(139.6138, 35.1528)::geography, 1000)
```

地名が抽出できない、またはジオコーディングが失敗した場合は、
座標なしのままSQL生成を続行し（`place_name` の LIKE 検索にフォールバック）、
その旨をレスポンスに含める。

## Text2SQL プロンプト設計

プロンプトには以下を含めることで、タグ語彙が不明なまま SQL を生成する問題を防ぐ。

### 組み込む情報

| 要素 | 内容 |
|---|---|
| スキーマ定義 | `media_items` の CREATE TABLE 全文 |
| タグ語彙 | `tag_vocabulary` テーブルから起動時に取得してキャッシュした語彙一覧 |
| SQL生成ルール | 下記参照 |
| ユーザークエリ | iPhone から受け取った自然言語文字列 |

### タグ語彙の取得とキャッシュ

```go
// サーバー起動時に一度だけ取得し、プロセスメモリにキャッシュする
// tag_vocabulary テーブルが正のソースであり、コード側に語彙をハードコードしない
SELECT tag, tag_type, category FROM tag_vocabulary ORDER BY tag_type, category, tag;
```

起動時キャッシュのため Supabase API 呼び出しは 1回のみ。語彙を更新した場合はサーバーを再起動する。

### SQL生成ルール (プロンプトに明示する)

```
- SELECT 文のみ生成すること (INSERT/UPDATE/DELETE/DROP 等は絶対に生成しない)
- 必ず LIMIT 100 を付与すること
- タグ絞り込みには 'tag' = ANY(object_tags) または 'tag' = ANY(scene_tags) を使うこと
- 関連度順ソートには (object_tag_scores->>'tag')::numeric DESC を使うこと
- 日本語クエリでも英語タグ名を使うこと (例: 「犬」→ 'dog'、「海」→ 'ocean' または 'beach')
- 日付範囲は shot_at BETWEEN ... AND ... を使うこと
- 結果カラムは id, filename, thumbnail_path, shot_at, place_name のみ選択すること
```

### プロンプト構造 (概略)

```text
あなたはPostgreSQLのクエリ生成エキスパートです。
以下のスキーマとタグ語彙を使って、ユーザーの自然言語クエリをSQLに変換してください。

## スキーマ
CREATE TABLE media_items (
    id UUID, filename TEXT, thumbnail_path TEXT, shot_at TIMESTAMPTZ,
    place_name TEXT, camera_model TEXT,
    object_tags TEXT[], object_tag_scores JSONB,
    scene_tags  TEXT[], scene_tag_scores  JSONB,
    ...
);

## object_tags / object_tag_scores に使用できるタグ名
人物: person, people, child, baby, crowd
動物: dog, cat, bird, horse, fish, rabbit, wildlife
乗り物: car, bicycle, motorcycle, bus, truck, train, airplane, boat
...（tag_vocabulary から生成）

## scene_tags / scene_tag_scores に使用できるタグ名
屋内外: outdoor, indoor
自然: nature, forest, beach, mountain, desert, field, park, garden
...（tag_vocabulary から生成）

## ルール
- SELECT文のみ生成すること
- LIMIT 100を常に付与すること
- タグ検索: 'dog' = ANY(object_tags)
- 関連度ソート: (object_tag_scores->>'dog')::numeric DESC
- 日本語クエリでも英語タグ名を使うこと

## ユーザークエリ
{{query}}
```

## カスタムUDPプロトコル (大容量転送)

```
パケットフォーマット:
[4B: セッションID][4B: チャンクNo][4B: 総チャンク数][2B: ペイロード長][nB: ペイロード]

フロー:
1. HTTPで転送セッション開始 (ファイルサイズ・チェックサム送信)
2. UDPでチャンク転送
3. 欠落チャンクを選択的再送
4. HTTP で完了確認・チェックサム検証
```

## シークレット管理

AWS Parameter Store からシークレットを取得する。クレデンシャルは `/tmp/aws/credential.json` から読み込み、有効期限切れ時に自動リフレッシュする（`refreshablesession.py` と同様のパターン）。

### クレデンシャルファイル

`/tmp/aws/credential.json`（外部の仕組みで定期更新される）:

```json
{
    "AccessKeyId": "...",
    "SecretAccessKey": "...",
    "Token": "...",
    "Expiration": "2026-01-01T00:00:00Z"
}
```

### Parameter Store パス

リージョン: `ap-northeast-1`

| パス | 内容 | 備考 |
|---|---|---|
| `/auth0/nasphoto` | `{"clientId":"...","clientSecret":"...","domain":"dev-t8fsj7om1qpw4uzq.us.auth0.com"}` | ドメインは既存テナントを共有 |
| `/nasphoto/supabase` | `{"url":"...","key":"..."}` | service_role キー |
| `/nasphoto/anthropic_api_key` | Anthropic API キー | SecureString |
| `/nasphoto/apple_maps_server_key` | Apple Maps Server API キー | SecureString |
| `/omv/port/nasphoto` | OMV API サーバーのポート番号 | 既存 `/omv/port/*` 命名規則に準拠 |

### 実装方針 (`internal/secrets/aws.go`)

- `fileCredentialsProvider` が `aws.CredentialsProvider` を実装
  - `Retrieve()` のたびに `/tmp/aws/credential.json` を読み直す
  - `aws.NewCredentialsCache` でラップし、有効期限内はキャッシュ・期限切れで自動再読み込み
- 起動時に各パラメータを取得してプロセスメモリに保持する

```go
type Secrets struct {
    SupabaseURL        string
    SupabaseKey        string
    Auth0Domain        string
    Auth0ClientID      string
    Auth0ClientSecret  string
    AnthropicAPIKey    string
    AppleMapsServerKey string
    Port               int
}
```
