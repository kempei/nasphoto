# データベース 設計文書

## テーブル設計

### `media_items` テーブル

```sql
CREATE TABLE media_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    filename        TEXT NOT NULL,
    nas_path        TEXT NOT NULL UNIQUE,
    file_size       BIGINT NOT NULL,
    mime_type       TEXT NOT NULL,
    sha256          TEXT NOT NULL UNIQUE,

    -- 撮影情報
    shot_at         TIMESTAMPTZ,          -- EXIF撮影日時 (NULLの場合あり: スキャン写真)
    imported_at     TIMESTAMPTZ NOT NULL, -- Photos.app取り込み日時

    -- カメラ情報
    camera_make     TEXT,
    camera_model    TEXT,
    lens_model      TEXT,
    focal_length_mm NUMERIC,
    aperture        NUMERIC,
    shutter_speed   TEXT,
    iso             INTEGER,

    -- 位置情報
    latitude        DOUBLE PRECISION,
    longitude       DOUBLE PRECISION,
    place_name      TEXT,             -- リバースジオコーディング結果 (表示用)
    location        GEOGRAPHY(POINT, 4326), -- 空間インデックス用 (latitude/longitude から生成)

    -- AI解析タグ (タグ名は tag_vocabulary.tag を参照)
    object_tags        TEXT[] DEFAULT '{}',  -- タグ名配列: GIN検索用
    object_tag_scores  JSONB  DEFAULT '{}',  -- {"dog": 0.92, "person": 0.87}: 関連度ソート用
    scene_tags         TEXT[] DEFAULT '{}',  -- タグ名配列: GIN検索用
    scene_tag_scores   JSONB  DEFAULT '{}',  -- {"outdoor": 0.95, "beach": 0.81}: 関連度ソート用
    face_cluster_ids   UUID[] DEFAULT '{}',

    -- サムネイル
    thumbnail_path  TEXT,                 -- NAS APIサーバーの相対パス

    -- 管理情報
    backed_up_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- インデックス
CREATE INDEX idx_media_shot_at        ON media_items (shot_at);
CREATE INDEX idx_media_imported_at    ON media_items (imported_at);
CREATE INDEX idx_media_object_tags    ON media_items USING GIN (object_tags);
CREATE INDEX idx_media_scene_tags     ON media_items USING GIN (scene_tags);
CREATE INDEX idx_media_object_scores  ON media_items USING GIN (object_tag_scores);
CREATE INDEX idx_media_scene_scores   ON media_items USING GIN (scene_tag_scores);
CREATE INDEX idx_media_camera         ON media_items (camera_model);
CREATE INDEX idx_media_location_gist  ON media_items USING GIST (location)
    WHERE location IS NOT NULL;
```

#### 位置情報の登録

`location` カラムは Mac アプリがメタデータを Supabase に保存する際、
`latitude` / `longitude` から `ST_MakePoint` で生成して同時に書き込む。

```sql
-- Mac アプリからの INSERT 例
INSERT INTO media_items (..., latitude, longitude, location, ...)
VALUES (..., 35.1528, 139.6138,
        ST_MakePoint(139.6138, 35.1528)::geography, ...);
```

#### 近傍検索クエリの例

```sql
-- 「三浦海岸駅周辺 1km 以内」(座標はジオコーディング API が解決)
SELECT id, filename, thumbnail_path, shot_at, place_name
FROM media_items
WHERE ST_DWithin(
    location,
    ST_MakePoint(139.6138, 35.1528)::geography,
    1000   -- メートル単位
)
ORDER BY ST_Distance(location, ST_MakePoint(139.6138, 35.1528)::geography)
LIMIT 100;
```

`place_name` は表示用の自由テキスト（リバースジオコーディング結果）であり、
場所名での検索には使用しない。場所名検索は必ずジオコーディング → 座標 → `ST_DWithin` の
パイプラインで処理する。

#### スコアを使った関連度ソートの例

```sql
-- 「犬が写っている写真」を犬の信頼スコア順に取得
SELECT id, filename, thumbnail_path, shot_at,
       (object_tag_scores->>'dog')::numeric AS dog_score
FROM media_items
WHERE 'dog' = ANY(object_tags)
ORDER BY dog_score DESC
LIMIT 100;
```

タグの絞り込みには `TEXT[]` + GIN インデックスを使い、ソートには `JSONB` スコアを参照する。
この二重持ちにより、フィルタ性能とランキング機能を両立する。

### `tag_vocabulary` テーブル

許可されたタグを一元管理するマスターテーブル。
`VisionAnalyzer.swift` の許可リストと Text2SQL のプロンプト語彙はここを正とする。

```sql
CREATE TABLE tag_vocabulary (
    tag         TEXT PRIMARY KEY,
    tag_type    TEXT NOT NULL CHECK (tag_type IN ('object', 'scene')),
    category    TEXT NOT NULL,  -- グループ名 (例: '人物', '動物', 'シーン')
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- シードデータ (object タグ)
INSERT INTO tag_vocabulary (tag, tag_type, category) VALUES
    ('person',     'object', '人物'),
    ('people',     'object', '人物'),
    ('child',      'object', '人物'),
    ('baby',       'object', '人物'),
    ('crowd',      'object', '人物'),
    ('dog',        'object', '動物'),
    ('cat',        'object', '動物'),
    ('bird',       'object', '動物'),
    ('horse',      'object', '動物'),
    ('fish',       'object', '動物'),
    ('rabbit',     'object', '動物'),
    ('wildlife',   'object', '動物'),
    ('car',        'object', '乗り物'),
    ('bicycle',    'object', '乗り物'),
    ('motorcycle', 'object', '乗り物'),
    ('bus',        'object', '乗り物'),
    ('truck',      'object', '乗り物'),
    ('train',      'object', '乗り物'),
    ('airplane',   'object', '乗り物'),
    ('boat',       'object', '乗り物'),
    ('flower',     'object', '植物'),
    ('tree',       'object', '植物'),
    ('plant',      'object', '植物'),
    ('grass',      'object', '植物'),
    ('leaf',       'object', '植物'),
    ('food',       'object', '食べ物'),
    ('drink',      'object', '食べ物'),
    ('meal',       'object', '食べ物'),
    ('cake',       'object', '食べ物'),
    ('fruit',      'object', '食べ物'),
    ('building',   'object', '建物'),
    ('bridge',     'object', '建物'),
    ('tower',      'object', '建物'),
    ('sign',       'object', '建物'),
    ('house',      'object', '建物'),
    ('rock',       'object', '自然物'),
    ('water',      'object', '自然物'),
    ('cloud',      'object', '自然物'),
    ('fire',       'object', '自然物'),
    ('snow',       'object', '自然物'),
    ('ice',        'object', '自然物'),
    ('book',       'object', '日用品'),
    ('bottle',     'object', '日用品'),
    ('cup',        'object', '日用品'),
    ('bag',        'object', '日用品'),
    ('clothing',   'object', '日用品'),
    ('furniture',  'object', '日用品'),
    ('screen',     'object', '日用品'),
    ('ball',       'object', 'スポーツ'),
    ('sport',      'object', 'スポーツ'),
-- シードデータ (scene タグ)
    ('outdoor',      'scene', '屋内外'),
    ('indoor',       'scene', '屋内外'),
    ('nature',       'scene', '自然'),
    ('forest',       'scene', '自然'),
    ('beach',        'scene', '自然'),
    ('mountain',     'scene', '自然'),
    ('desert',       'scene', '自然'),
    ('field',        'scene', '自然'),
    ('park',         'scene', '自然'),
    ('garden',       'scene', '自然'),
    ('ocean',        'scene', '水域'),
    ('lake',         'scene', '水域'),
    ('river',        'scene', '水域'),
    ('waterfall',    'scene', '水域'),
    ('pool',         'scene', '水域'),
    ('urban',        'scene', '都市'),
    ('street',       'scene', '都市'),
    ('road',         'scene', '都市'),
    ('cityscape',    'scene', '都市'),
    ('architecture', 'scene', '都市'),
    ('sky',          'scene', '空・気象'),
    ('sunset',       'scene', '空・気象'),
    ('sunrise',      'scene', '空・気象'),
    ('night',        'scene', '空・気象'),
    ('fog',          'scene', '空・気象'),
    ('rain',         'scene', '空・気象'),
    ('spring',       'scene', '季節'),
    ('summer',       'scene', '季節'),
    ('autumn',       'scene', '季節'),
    ('winter',       'scene', '季節'),
    ('home',         'scene', '場所'),
    ('restaurant',   'scene', '場所'),
    ('stadium',      'scene', '場所'),
    ('airport',      'scene', '場所'),
    ('station',      'scene', '場所');
```

語彙を追加・削除する場合は、このシードデータに対するマイグレーションを作成する。
コード側の許可リストは `tag_vocabulary` テーブルを参照するため、マイグレーション適用のみで反映される。

### `face_clusters` テーブル

```sql
CREATE TABLE face_clusters (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT,                 -- ユーザーが付けた名前 (省略可)
    representative_media_id UUID REFERENCES media_items(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### `backup_cursors` テーブル

```sql
-- Macアプリのバックアップ進捗管理
CREATE TABLE backup_cursors (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    device_id       TEXT NOT NULL UNIQUE,  -- MacのUUID
    last_imported_at TIMESTAMPTZ NOT NULL, -- 最後にバックアップした取り込み日時
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

## AIタグ設計

### タグ生成元

`object_tags` / `scene_tags` は macOS アプリの `VisionAnalyzer.swift` が
`VNClassifyImageRequest` (Apple Vision framework) を実行し、以下の処理で生成する:

1. 信頼スコア 0.5 以上の `VNClassificationObservation` を抽出
2. `tag_vocabulary` テーブルに存在するラベルのみを採用 (起動時にテーブルを取得してキャッシュ)
3. `tag_type` に応じて `object_tags` / `scene_tags` に振り分け
4. タグ名配列を `*_tags TEXT[]` に、`{タグ名: スコア}` マップを `*_tag_scores JSONB` に格納

### 語彙の正（ソース・オブ・トゥルース）

`tag_vocabulary` テーブルが唯一の正。VisionAnalyzer の許可リスト・Text2SQL のプロンプト語彙は
どちらもこのテーブルから動的に取得する。語彙変更はマイグレーションのみで完結する。

### タグは英語固定

タグ値は Apple Vision framework のラベル (英語小文字) をそのまま使用する。
Text2SQL プロンプトで「日本語 → 英語タグ名」のマッピングを Claude に伝える
（例: 「犬」→ `dog`、「海」→ `ocean` または `beach`）。

## マイグレーション管理

```
db/
├── migrations/
│   ├── 001_initial_schema.sql
│   ├── 002_add_face_clusters.sql
│   ├── 003_add_backup_cursors.sql
│   └── 004_add_tag_vocabulary_and_scores.sql
├── seeds/
│   └── test_data.sql
└── supabase/
    └── config.toml
```

## Row Level Security (RLS)

```sql
-- メディアアイテムは認証済みユーザーのみ読み取り可
ALTER TABLE media_items ENABLE ROW LEVEL SECURITY;
CREATE POLICY "authenticated read" ON media_items
    FOR SELECT USING (auth.role() = 'authenticated');

-- 書き込みはサービスロール (APIサーバー) のみ
CREATE POLICY "service write" ON media_items
    FOR ALL USING (auth.role() = 'service_role');

-- タグ語彙は認証済みユーザーが読み取り可 (書き込みはサービスロールのみ)
ALTER TABLE tag_vocabulary ENABLE ROW LEVEL SECURITY;
CREATE POLICY "authenticated read" ON tag_vocabulary
    FOR SELECT USING (auth.role() = 'authenticated');
CREATE POLICY "service write" ON tag_vocabulary
    FOR ALL USING (auth.role() = 'service_role');
```
