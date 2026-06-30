# ADR-008: 写真地図サイト 3方式比較

> Rainbow Station = 7色路線 × 写真アップロード × Leaflet地図
> Date: 2026-06-30

---

## 3候補

| ID | Frontend | Backend | コード |
|---|---|---|---|
| **ELX** | Phoenix LiveView | Elixir/Phoenix | Rails から全面移行 |
| **REF** | ClojureScript Re-frame | Rails API（変更不要） | Frontend のみ新規 |
| **ELM** | Elm SPA | Rails API（変更不要） | Frontend のみ新規 |

---

## 評価軸（写真地図サイトの核心）

| 軸 | 重み | なぜ重要か |
|---|---|---|
| リアルタイム写真反映 | 20% | 誰かが投稿 → 即全員の地図にピン |
| カメラ+アップロード | 20% | getUserMedia → Cloudinary |
| Leaflet 地図連携 | 15% | 路線位置 + 写真ピン + ポップアップ |
| API 型安全性 | 15% | Rails JSON の snake_case を安全に扱う |
| 開発速度 | 15% | 1人開発、イベントまでの時間 |
| 運用・デプロイ | 10% | サーバーコスト、静的ホスティング可否 |
| 既存スキル | 5% | world-model (Clojure), Stray Dog (TS) |

---

## 1. ELX — Elixir + Phoenix LiveView

### アーキテクチャ

```
ブラウザ                  サーバー (Fly.io $5/mo)
┌──────────┐  WebSocket  ┌─────────────────────┐
│ Leaflet  │◄──────────►│ Phoenix LiveView     │
│ カメラ   │  pushEvent │  ├ PubSub (リアルタイム) │
│ 地図     │            │  ├ Ecto (DB)           │
└──────────┘            │  └ Cloudinary アップロード │
                        └─────────────────────┘
```

### 写真投稿 → リアルタイム反映

```elixir
# たったこれだけ
def handle_event("photo_uploaded", photo, socket) do
  Phoenix.PubSub.broadcast(App.PubSub, "map", {:new_photo, photo})
  {:noreply, socket}
end
```

**全クライアントの地図に即座にピンが立つ。** コード 2 行。

### Leaflet 連携

```javascript
// LiveView Hook (JS側)
handleEvent("add-photo-marker", ({ lat, lng, image_url }) => {
  L.marker([lat, lng])
    .addTo(map)
    .bindPopup(`<img src="${image_url}" width="200">`)
})
```

### Elixir 移行コスト

| 項目 | 工数 |
|---|---|
| Rails → Ecto DB設計 | 1日（migration → schema） |
| API移植 | 1日（7路線 × 2エンドポイント） |
| LiveView 画面 | 2日（路線一覧 + 詳細 + 投稿） |
| 認証 | `mix phx.gen.auth` → 10分 |
| Cloudinary 連携 | 半日（`cloister` or HTTPoison） |
| デプロイ (Fly.io) | 半日 |
| **合計** | **5日** |

### スコア

| 軸 | 点 | 理由 |
|---|---|---|
| リアルタイム | **10** | PubSub。コード2行で全員に配信 |
| カメラ+アップ | **10** | LiveView Upload。プログレスバー込み |
| Leaflet | 7 | Hook 経由。JS を書く必要あり |
| API型安全 | 6 | 動的型。dialyzer 補助だが Elm に劣る |
| 開発速度 | 7 | フルスタック書き直し（Rails → Elixir） |
| 運用 | 5 | Fly.io $5/月。静的ホスティング不可 |
| 既存スキル | 5 | Elixir 未経験 |
| **加重** | **7.55** | |

---

## 2. REF — ClojureScript + Re-frame + Rails API

### アーキテクチャ

```
ブラウザ (Shadow-cljs → 静的JS)
┌──────────────────────┐   REST API   ┌──────────┐
│ Re-frame App         │◄───────────►│ Rails API│
│  ├ App DB (atom)     │             │ (変更不要)│
│  ├ Leaflet (js/相互運用)│            │          │
│  └ Camera (js/相互運用) │            └──────────┘
└──────────────────────┘
      │ surge.sh (無料)
```

### REPL 駆動開発が真価を発揮

```clojure
;; ブラウザで直接評価 → 即UI反映
(rf/dispatch [:routes-loaded
  [{:id 1 :name "赤" :color "#E6001A" :lat 35.69 :lng 139.75 :photo_count 3}
   {:id 2 :name "橙" :color "#FFA200" :lat 35.70 :lng 139.76 :photo_count 1}]])

;; → 7路線カードが即表示。API待たずにUI確認完了

;; アップロード状態のテスト
(rf/dispatch [:uploading? true])
;; → プログレスバー表示を即確認
```

### コード構造

```clojure
;; イベントは単なるデータ。追跡・リプレイ・テストが容易
(rf/reg-event-fx
  :fetch-routes
  (fn [_ _]
    {:http-xhrio {:method :get
                  :uri "/api/v1/routes"
                  :on-success [:routes-loaded]}}))

(rf/reg-event-fx
  :upload-photo
  (fn [{:keys [db]} [_ file route-id]]
    {:db (assoc db :uploading? true)
     :http-xhrio {:method :post
                  :uri "/api/v1/photos"
                  :body (form-data file route-id (:device-token db))
                  :on-success [:photo-uploaded]
                  :on-failure [:upload-failed]}}))
```

### スコア

| 軸 | 点 | 理由 |
|---|---|---|
| リアルタイム | 5 | REST API のみ。ポーリング or 手動リロード |
| カメラ+アップ | 8 | `js/` 相互運用でカメラ操作。FormData 送信 |
| Leaflet | **10** | `js/` で直接インポート。Stray Dog 同等 |
| API型安全 | 5 | 動的型。malli/spec で実行時検証可能 |
| 開発速度 | **9** | Rails 使える。REPL で試行錯誤爆速 |
| 運用 | **10** | 静的ファイル → surge.sh 無料 |
| 既存スキル | **8** | world-model (Clojure) 経験あり |
| **加重** | **7.40** | |

---

## 3. ELM — Elm SPA + Rails API

### アーキテクチャ

```
ブラウザ (Elm → 静的JS)
┌──────────────────────┐   REST API   ┌──────────┐
│ Elm App (TEA)        │◄───────────►│ Rails API│
│  ├ Model (不変)       │             │ (変更不要)│
│  ├ Decoder (型安全)   │             │          │
│  ├ Leaflet (Ports)    │             └──────────┘
│  └ Camera (Ports)     │
└──────────────────────┘
      │ surge.sh (無料)
```

### API デコード — ここだけは最強

```elm
-- Rails snake_case → Elm が型安全に吸収
routeDecoder : Decoder Route
routeDecoder =
    Decode.succeed Route
        |> Decode.required "id" Decode.int
        |> Decode.required "name" Decode.string
        |> Decode.required "color" Decode.string
        |> Decode.required "color_name" Decode.string
        |> Decode.required "photo_count" Decode.int
        -- Rails がキー名を変えてもコンパイルエラーで即発見

-- これが実行時エラーを完全に防ぐ
```

### スコア

| 軸 | 点 | 理由 |
|---|---|---|
| リアルタイム | 5 | REST のみ（REF と同じ） |
| カメラ+アップ | 7 | `Http.multipart` で可能。Ports でカメラ |
| Leaflet | 6 | Ports 経由。JS 側で全操作 |
| API型安全 | **10** | Decoder がコンパイル時に Rails JSON を保証 |
| 開発速度 | 5 | 学習曲線 + Ports 設計の手間 |
| 運用 | **10** | 静的ファイル → surge.sh 無料 |
| 既存スキル | 3 | Elm 未経験 |
| **加重** | **6.50** | |

---

## 比較マトリックス

| 軸 | 重み | ELX | REF | ELM |
|---|---|---|---|---|
| リアルタイム写真反映 | 20% | **10** | 5 | 5 |
| カメラ+アップロード | 20% | **10** | 8 | 7 |
| Leaflet 地図連携 | 15% | 7 | **10** | 6 |
| API 型安全性 | 15% | 6 | 5 | **10** |
| 開発速度 | 15% | 7 | **9** | 5 |
| 運用・デプロイ | 10% | 5 | **10** | **10** |
| 既存スキル | 5% | 5 | **8** | 3 |
| **加重スコア** | **100%** | **7.55** | **7.40** | **6.50** |

---

## 結論

### 写真地図サイト: ELX (7.55) ≧ REF (7.40) > ELM (6.50)

ELX と REF は僅差。決定的な差は:

| 差 | ELX が勝つ | REF が勝つ |
|---|---|---|
| リアルタイム | PubSub 2行 | なし（ポーリング） |
| コスト | Fly.io $5/月 | surge.sh 無料 |
| 開発体験 | LiveView（サーバー型） | REPL（対話型） |
| Rails 資産 | 破棄 | **温存** |

### 推奨

```
リアルタイム必須 → ELX (Elixir + Phoenix)
コスト最小       → REF (ClojureScript + Rails)
とりあえず動かす → REF（Rails API を変えずに Frontend だけ）
```

**Rainbow Station の当面の答: REF (Re-frame + Rails API)**

理由:
1. Rails API がすでに動いている（破棄する理由がない）
2. world-model の Clojure 経験が活きる
3. 静的ホスティング無料（surge.sh）
4. リアルタイムは必須ではない（参加者が同時に投稿することは稀）
5. REPL で 7路線カードのデザインを爆速で試せる

---

## ソロゲーム (Stray Dog)

```
推奨: Elm + 静的ホスティング

理由:
  オフライン完結 → Elm SPA が最適
  バックエンド不要 → 静的ファイルのみ
  状態機械との相性 → TEA（The Elm Architecture）
  Leaflet/GPS → Ports で逃がす
```

| ソロゲーム | 選定理由 |
|---|---|
| Elm | TEA = 状態機械。コンパイル = バグゼロ。オフライン完結 |
| TS | 今すぐ。慣れてる。Ports不要（Leaflet直接import） |

Stray Dog は Elm の最も得意な領域。Elixir の出番はない（サーバー不要）。

---

*ADR-008 / 2026-06-30*
