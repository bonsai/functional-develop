# ADR-005: Rainbow Station フロントエンド選定

> Date: 2026-06-30 | 前提: Rails API は稼働中・変更不要

---

## Rainbow Station の概要

```
POST /api/v1/photos     ← 写真アップロード (Cloudinary)
GET  /api/v1/routes     ← 7路線一覧（赤橙黄緑青藍紫）
GET  /api/v1/routes/:id ← 路線詳細 + 写真一覧
DELETE /api/v1/photos/:id ← 写真削除
```

| 項目 | 内容 |
|---|---|
| 路線数 | 7色（赤/橙/黄/緑/青/藍/紫） |
| 写真 | 1路線あたり最大10枚/デバイス |
| 位置情報 | lat/lng 付き |
| ストレージ | Cloudinary（border加工済み） |
| 認証 | なし（device_token でデバイス識別） |

---

## 評価: Elm vs TypeScript+Vite

### 評価軸

| 軸 | 重み | 説明 |
|---|---|---|
| API 通信 | 20% | REST API 呼出し、JSON デコード、エラーハンドリング |
| ファイルアップロード | 15% | カメラ撮影 → FormData → Cloudinary |
| 地図 (Leaflet) | 15% | 路線位置表示、写真位置プロット |
| 状態管理 | 20% | 7路線 × 写真一覧 × アップロード状態 |
| 型安全性 | 15% | API レスポンスの型保証 |
| 開発速度 | 10% | 納期までの実装時間 |
| 保守性 | 5% | 将来の機能追加のしやすさ |

---

## Elm

### API 通信 — 圧倒的

```elm
-- API レスポンスの型定義がそのまま JSON デコーダーになる
type alias Route =
    { id : Int
    , name : String
    , colorName : String
    , emoji : String
    , lat : Maybe Float
    , lng : Maybe Float
    , photoCount : Int
    }

routeDecoder : Decoder Route
routeDecoder =
    Decode.succeed Route
        |> Decode.required "id" Decode.int
        |> Decode.required "name" Decode.string
        |> Decode.required "color_name" Decode.string
        |> Decode.required "emoji" (Decode.maybe Decode.string |> Decode.map (Maybe.withDefault ""))
        |> Decode.required "lat" (Decode.nullable Decode.float)
        |> Decode.required "lng" (Decode.nullable Decode.float)
        |> Decode.required "photo_count" Decode.int

-- Rails API との JSON キー不一致はここで吸収される
-- ランタイムエラーにならない（デコード失敗は Elm の Result 型で処理）
```

### ファイルアップロード — 組み込み

```elm
-- Http.multipart でカメラ画像を直接送信
uploadPhoto : File -> Route -> DeviceToken -> Cmd Msg
uploadPhoto file route token =
    Http.multipart
        { url = "/api/v1/photos"
        , body =
            [ Http.stringPart "device_token" token
            , Http.stringPart "route_id" (String.fromInt route.id)
            , Http.stringPart "lat" (String.fromFloat lat)
            , Http.stringPart "lng" (String.fromFloat lng)
            , Http.filePart "image" file
            ]
        , expect = Http.expectJson GotUploadResponse photoDecoder
        }
```

### Leaflet — Ports

```elm
-- Ports: Elm ⇄ JavaScript
port initMap : (Float, Float) -> Cmd msg
port addPhotoMarker : (Float, Float, String) -> Cmd msg
port mapClicked : ((Float, Float) -> msg) -> Sub msg
```

### スコア

| 軸 | 評価 | 理由 |
|---|---|---|
| API 通信 | ★★★★★ | Decoder が Rails JSON を安全に吸収 |
| アップロード | ★★★★☆ | `Http.multipart` で完結 |
| Leaflet | ★★★☆☆ | Ports 設計が必要。Stray Dog で経験済み |
| 状態管理 | ★★★★★ | 7路線 × 写真 × アップロード状態を TEA で網羅 |
| 型安全性 | ★★★★★ | API 変更時にコンパイラが全箇所を指摘 |
| 開発速度 | ★★☆☆☆ | 学習曲線あり。ただし Stray Dog の Elm 版を作れば加速 |
| 保守性 | ★★★★★ | 型がドキュメント。1年後に読める |

---

## TypeScript + Vite

### API 通信

```typescript
type Route = {
  id: number
  name: string
  color_name: string  // Rails の snake_case をそのまま
  emoji: string | null
  lat: number | null
  lng: number | null
  photo_count: number
}

// fetch + 手動キャスト
const res = await fetch('/api/v1/routes')
const routes: Route[] = await res.json()
// ↑ ランタイムまで型が合ってるか不明
```

### スコア

| 軸 | 評価 | 理由 |
|---|---|---|
| API 通信 | ★★★☆☆ | 型はあるがランタイム保証なし。`zod` でバリデーション追加可能 |
| アップロード | ★★★★★ | `FormData` + `fetch`。JavaScript の得意領域 |
| Leaflet | ★★★★★ | 直接インポート。Stray Dog で実績あり |
| 状態管理 | ★★★☆☆ | zustand で管理可能だが、7路線分の非同期状態が複雑化 |
| 型安全性 | ★★★☆☆ | コンパイルは通っても API 変更で実行時エラー |
| 開発速度 | ★★★★★ | 慣れている。Stray Dog のコードを流用可能 |
| 保守性 | ★★★☆☆ | テスト書かないと型があっても安心できない |

---

## 比較マトリックス

| 軸 | 重み | Elm | TS+Vite |
|---|---|---|---|
| API 通信 | 20% | **10** | 6 |
| アップロード | 15% | 8 | **10** |
| Leaflet 地図 | 15% | 6 | **10** |
| 状態管理 | 20% | **10** | 6 |
| 型安全性 | 15% | **10** | 5 |
| 開発速度 | 10% | 5 | **10** |
| 保守性 | 5% | **10** | 6 |
| **加重スコア** | **100%** | **8.55** | **7.45** |

---

## 決定的な差: API デコード

Rainbow Station の Rails API は `snake_case` キー（`color_name`, `line_name`, `photo_count`）。
Elm の `Json.Decode` はキー名を明示的にマッピングするため、**Rails 側の命名規則を吸収できる**。
TypeScript だと `res.json()` が `any` を返し、実行時までキーの不一致に気づかない。

> 「Rails でカラム名を `colour_name` に変えた → Elm はコンパイルエラー、TS は実行時 undefined」

---

## 推奨: A (Elm + Rails API)

理由:

1. **API 層の型安全性が決定的** — Rails の JSON 出力を Elm の Decoder が完全に検証
2. **7路線の状態管理** — TEA が複数路線 × 写真 × アップロード進行の状態を網羅
3. **Leaflet は Ports で十分** — 路線位置表示はシンプル。写真プロットも Ports で JS に任せればよい
4. **Stray Dog の Elm 版とコード共有** — `Camera.elm`, `Map.elm`, `Http.elm` を共通モジュール化できる

### Elm 学習のハードル

- Stray Dog より複雑（API 5エンドポイント）
- しかし Rails API が動いているので **フロントエンドだけ** に集中できる
- Stray Dog の Elm リライトで基礎を固めてから Rainbow Station に着手するのが効率的

---

## 推奨ロードマップ

```
Week 1 (7/1-7/5):    Stray Dog TypeScript 版を完成・イベント実施
Week 2 (7/7-7/12):   Stray Dog を Elm でリライト（学習 + 実績作り）
Week 3 (7/14-7/19):  Rainbow Station を Elm で構築
                      ├─ Day 1-2: 路線一覧 + 詳細 (Elm Http + Decoder)
                      ├─ Day 3:   カメラ + アップロード (Elm File + Http.multipart)
                      ├─ Day 4:   地図 (Ports + Leaflet)
                      └─ Day 5:   スタイリング + デプロイ
```

---

## B (TS+Vite) を選ぶなら

「今すぐ欲しい」が優先される場合のみ。
Stray Dog をそのまま拡張すれば 2 日で動く。

---

*ADR-005 / 2026-06-30*
