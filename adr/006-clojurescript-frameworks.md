# ADR-006: ClojureScript フレームワーク評価

> ユーザー問い合わせ「clj のフレームワークないの？どこが強い」
> Date: 2026-06-30

---

## ClojureScript フロントエンド・フレームワーク 早見表

| フレームワーク | Reactベース | 状態管理 | 学習曲線 | 成熟度 | 適性 |
|---|---|---|---|---|---|
| **Reagent** | ✅ | atom（単純） | ★★★★ | ★★★★★ | 小〜中規模 |
| **Re-frame** | ✅ | event-driven | ★★★ | ★★★★★ | **中〜大規模** |
| **Rum** | ✅ | atom/mixin | ★★★★ | ★★★★ | 軽量志向 |
| **Helix** | ✅ | hooks | ★★★ | ★★★ | React経験者向け |
| **Fulcro** | ✅(独自) | 正規化DB | ★★ | ★★★ | フルスタック |
| **Electric** | 独自 | signals/DB | ★ | ★★ | リアルタイム |

---

## 本命: Re-frame（★★★★★）

Re-frame は ClojureScript で最も使われているフレームワーク。
Redux に似たイベント駆動だが、Clojure の不変データ構造で洗練されている。

### アーキテクチャ

```
View (Reagent)       ←─ 購読（subscribe）
    │                      │
    │ dispatch event       │
    ▼                      │
Event Handler ──→ App DB (ratom) ──→ Subscription ──→ View
    │                 ↑
    │ side effect     │
    └──→ API/DB ──────┘
```

### Rainbow Station のコード見本

```clojure
(ns rainbow-station.events
  (:require [re-frame.core :as rf]
            [ajax.core :as ajax]))

;; --- App DB（全状態を1つの atom に） ---
(rf/reg-event-db
  :init
  (fn [_ _]
    {:routes []           ;; 7路線の一覧
     :current-route nil   ;; 選択中の路線
     :photos []           ;; 現在の路線の写真
     :uploading? false    ;; アップロード中？
     :device-token nil})) ;; 端末識別

;; --- API から路線一覧を取得 ---
(rf/reg-event-fx
  :fetch-routes
  (fn [{:keys [db]} _]
    {:http-xhrio {:method          :get
                  :uri             "/api/v1/routes"
                  :response-format (ajax/json-response-format {:keywords? true})
                  :on-success      [:routes-loaded]
                  :on-failure      [:api-error]}
     :db db}))

;; --- 取得成功 → DB に格納 ---
(rf/reg-event-db
  :routes-loaded
  (fn [db [_ routes]]
    (assoc db :routes routes)))

;; --- 写真アップロード ---
(rf/reg-event-fx
  :upload-photo
  (fn [{:keys [db]} [_ file route-id lat lng]]
    {:db (assoc db :uploading? true)
     :http-xhrio {:method  :post
                  :uri     "/api/v1/photos"
                  :body    (doto (js/FormData.)
                             (.append "image" file)
                             (.append "route_id" route-id)
                             (.append "lat" lat)
                             (.append "lng" lng)
                             (.append "device_token" (:device-token db)))
                  :on-success [:photo-uploaded]
                  :on-failure [:upload-failed]}}))
```

### View (Reagent + Hiccup)

```clojure
(ns rainbow-station.views
  (:require [re-frame.core :as rf]))

(defn route-card [route]
  [:div.route-card {:style {:border-left (str "4px solid " (:color route))}}
   [:span.emoji (:emoji route)]
   [:h3 (:name route)]
   [:p.color-name (:color_name route)]
   [:span.count (str (:photo_count route) "枚")]])

(defn routes-page []
  (let [routes (rf/subscribe [:routes])]
    (fn []
      [:div.routes-grid
       (for [route @routes]
         ^{:key (:id route)} [route-card route])])))
```

---

## なぜ Re-frame が強いか

### 1. 状態が一箇所 (App DB)

```clojure
;; 全画面の状態が 1つの map
{:routes        [{:id 1 :name "赤ルート" :photos 5} ...]
 :current-route {:id 2 :name "橙ルート" :photos [...]}
 :uploading?    true
 :camera        {:stream nil :facing :environment}
 :map           {:center [35.69 139.75] :zoom 14}}
```

Elm の TEA と同じく **「全状態が1つのデータ構造」**。
しかし Elm と違い、型注釈なしで REPL から対話的に操作できる。

### 2. イベントがデータ

```clojure
;; イベントも単なるベクター。REPL から手動発火してデバッグ可能
(rf/dispatch [:fetch-routes])
(rf/dispatch [:upload-photo file 3 35.69 139.75])
(rf/dispatch [:select-route 1])
```

### 3. 副作用が分離されている

```clojure
;; reg-event-db: 純粋な状態遷移のみ（副作用なし）
(rf/reg-event-db :routes-loaded (fn [db [_ routes]] (assoc db :routes routes)))

;; reg-event-fx: 副作用あり（HTTP, ファイル, GPS）
(rf/reg-event-fx :fetch-routes (fn [_ _] {:http-xhrio {...}}))
```

### 4. REPL 駆動開発

これが ClojureScript 最大の武器。ブラウザを開いたまま:

```clojure
;; ブラウザ内で直接評価
(rf/dispatch [:fetch-routes])
;; → API 呼び出し → 画面が即更新

;; DB の中身を確認
@(rf/subscribe [:routes])
;; → [{:id 1 :name "赤ルート" ...} ...]

;; アップロード状態を偽装して UI 確認
(rf/dispatch [:routes-loaded [{:id 1 :name "テスト" :photo_count 3}]])
;; → 画面が即座に変わる。API不要。
```

---

## 比較: ClojureScript vs Elm vs TypeScript

| 特性 | Re-frame | Elm | TS+fps |
|---|---|---|---|
| 状態管理 | event→DB | TEA | zustand |
| 型安全性 | なし(実行時) | **コンパイラ強制** | コンパイラ(弱) |
| REPL開発 | **★★★★★** | ★★ | ★★★ |
| APIデコード | spec/malli 検証 | **Decoder** | zod 検証 |
| Leaflet地図 | js 相互運用 | Ports | **直接import** |
| バンドルサイズ | ~200KB | **~30KB** | ~50KB |
| 学習曲線 | Lisp + フレームワーク | 1言語 | 慣れてる |
| 既存スキル | ✅ world-model | ❌ | ✅ Stray Dog |

---

## どこが ClojureScript の「勝ち筋」か

### REPL 駆動開発が真価を発揮する場面

- **UI の試行錯誤が多い** → 7路線カードのレイアウト、写真グリッド、アップロードプログレス
- **API が頻繁に変わる** → Rails のレスポンス変更に REPL で即適応
- **状態遷移が複雑** → `dispatch` で全コンビネーションを手動テスト可能

### Elm の勝ち筋

- API の型が固まっていて安定している
- バグゼロ保証が最重要（支払い、個人情報）
- チームが Haskell 経験者

### TypeScript の勝ち筋

- 今日中に動かしたい
- 既存コード（Stray Dog）を流用できる

---

## 推奨（Rainbow Station）

```
1位: Re-frame (ClojureScript)
     REPL駆動 × 既存スキル × API試行錯誤に最強

2位: Elm
     型安全性 × 長期保守性。ただし学習曲線あり

3位: TypeScript + Vite
     納期最優先なら
```

### Re-frame を推す理由

Rainbow Station は **7色 × 写真アップロード × デバイス管理** という、
「状態の組み合わせ爆発」が起きやすいアプリ。

Re-frame の REPL 駆動開発なら:

```clojure
;; 「青ルートに写真を3枚アップロードした状態」を1行で再現
(rf/dispatch :routes-loaded
  [{:id 6 :name "青" :photo_count 3
    :photos [{:image_url "..."} {:image_url "..."} {:image_url "..."}]}])

;; → 即座に UI が更新。手動で3枚撮影する必要なし。
```

この「状態の即時再現」は Elm でも TypeScript でもできない ClojureScript の独自能力。

---

## 構成案

```
Frontend:   ClojureScript + Re-frame + Shadow-cljs
Backend:    Rails API (変更不要)
Map:        Leaflet.js (js/ 相互運用)
Camera:     getUserMedia (js/ 相互運用)
Build:      shadow-cljs → 静的ファイル → surge.sh
```

---

*ADR-006 / 2026-06-30*
