# ADR-013: omni-survey の Web フレームワーク / 言語選定

> Date: 2026-07-19 | Project: omni-survey (汎用アンケート生成サービス)

---

## 要件定義

| 要件 | 内容 |
|---|---|
| 本質的機能 | テンプレートJSON差し替えだけで新規アンケートを即公開 |
| フロントエンド | 動的フォーム生成（title/description/fields[] から描画） |
| バックエンド | `/api/submit` で受付、BigQuery へ全投入 |
| クラウド | GCP（Cloud Run / Cloud Functions）+ GWS 連携（スプシエクスポート） |
| 運用 | Firebase Hosting で静的配信、Functions で API |
| 多言語候補 | Java(Spring) / Rails / Gleam / OCaml / Go |
| 制約 | 個人の趣味・小規模運用。学習コストより「書きやすさ×GCP親和性」 |

---

## 最重要要件: 「JSONテンプレート → 動的フォーム」の容易さ

omni-survey の核心は **「1ファイル(JSON)追加 = 新規サーベイ誕生」** である。
これが言語/フレームワーク評価を決める。

---

## 候補とスコア

### 凡例

```
★ = 致命的に不向き
★★ = 可能だが苦しい
★★★ = 普通にできる
★★★★ = 得意
★★★★★ = このために設計された
```

| 要件 | Java/Spring | Rails | Gleam | OCaml | Go |
|---|---|---|---|---|---|
| ルーティング/テンプレート | ★★★★ | ★★★★★ | ★★★ | ★★★ | ★★★ |
| JSON→動的フォーム生成 | ★★★ | ★★★★★ | ★★★★ | ★★★ | ★★★ |
| GCP 親和性 | ★★★★★ | ★★★ | ★★ | ★★★ | ★★★★★ |
| GWS(スプシ)連携 | ★★★★ | ★★★ | ★★ | ★★ | ★★★★ |
| 型安全性 | ★★★★ | ★★ | ★★★★★ | ★★★★★ | ★★★ |
| ローカル起動の軽さ | ★★ | ★★★ | ★★★★★ | ★★★★ | ★★★★★ |
| Firebase Functions 載せやすさ | ★★ | ★★ | ★★ | ★★ | ★★★★ |
| 開発速度(趣味運用) | ★★ | ★★★★★ | ★★★ | ★★ | ★★★★ |

---

## 詳細評価

### Rails ★★★★★（本命）

**なぜ勝つのか:**

```ruby
# config/routes.rb
resources :surveys, param: :slug
# GET /s/uber  → surveys#show (uber.json を描画)
# POST /api/submit → responses#create (BQへ)

# app/controllers/surveys_controller.rb
def show
  @tpl = JSON.parse(File.read("templates/#{params[:slug]}.json"))
  # 動的フォームは @tpl[:fields] を each で描画
end
```

- **`jsonb` カラム + ActiveRecord** で「どんなフィールドでもそのまま保存」が1行
- **scaffold** で管理画面（誰でもサーベイ作成）が数分
- **Active Job + Sidekiq** で BQ エクスポートを非同期化
- **google-api-client** gem で GWS スプシ連携が既製品
- JSONテンプレート → フォーム生成のパターンが Rail の強みそのもの

**弱点:**
- Firebase Functions に載らない（Cloud Run 必須）
- 起動が重い（趣味運用の小規模なら問題なし）
- 型安全性は動的

---

### Go ★★★★（クラウド最適）

```go
// Cloud Run 単体で完結。バイナリ1つでデプロイ
func submit(w http.ResponseWriter, r *http.Request) {
    var payload map[string]interface{}
    json.NewDecoder(r.Body).Decode(&payload)
    bigquery.Insert(payload) // 型不要で jsonb 投入
}
```

- **Cloud Run との相性最高**（バイナリ単体、コールドスタート数ms）
- **GCP SDK が公式前提言語**
- Firebase のバックエンドにも関数として載る
- 静的型だが、JSONテンプレートの動的生成は reflect 任せで少し泥臭い

**弱点:**
- 動的フォーム生成の記述が Rails より冗長
- テンプレート新規作成の「楽さ」は Rails に劣る

---

### Gleam ★★★（型安全・軽量だが若い）

- BEAM 上で動き、型安全 + パターンマッチが美しい
- だが **GCP/GWS SDK が未成熟**、Web フレームワーク(Mist等)も発展途上
- JSONテンプレート描画は `gleam/json` で書けるが、全体骨組みの労力大
- 趣味の「言語探求」には最高だが、サービス安定運用には早計

**弱点:**
- GCP 連携を自前実装（HTTP直叩き）するしかない
- エコシステムが小さく、詰まった時の情報が少ない

---

### OCaml ★★★（型安全の極みだが重い）

- 型安全性は 5つ中最強、が Web フレームワーク(Opium/Dream) の情報が少ない
- GWS/BigQuery 連携ライブラリが貧弱
- コンパイルが遅く、趣味の迅速開発には不向き

**弱点:**
- エコシステムの小ささ（Rails/Go と数比較で1桁違う）
- GCP 公式 SDK なし

---

### Java / Spring ★★（やり過ぎ）

- エンタープライズ向け。omni-survey の規模に不要
- 起動重い、Firebase Functions に載らない
- 書く量が多く、JSONテンプレート差し替えの「軽さ」と逆行

**弱点:**
- オーバーエンジニアリング
- 趣味運用の迅速さを殺す

---

## 決定

```
omni-survey = Ruby on Rails 8 (Cloud Run)  +  BigQuery  +  GWSスプシ連携

理由:
  1. JSONテンプレート → 動的フォーム生成が Rails の真骨頂
  2. jsonb カラムで「どんな回答でもそのまま保存」が1行
  3. google-api-client gem で GWS スプシエクスポートが既製品
  4. scaffold で管理画面（誰でもサーベイ作成）が数分
  5. Cloud Run なら Rails の起動重さを気にせずデプロイ可能

代替(クラウド最適): Go on Cloud Run
  - Rails の「書きやすさ」より「GCP親和性・コールドスタート」を取りたい時
  - または Firebase バックエンド関数として載せる時
```

---

## 決定木

```
Q1: 新規サーベイ作成の「容易さ」を最優先？
├─ Yes → Rails (jsonb + scaffold + 動的フォーム)
└─ No  → Q2

Q2: GCP / Firebase との親和性を最優先？
├─ Yes → Go (Cloud Run / Functions、コールドスタート最速)
└─ No  → Q3

Q3: 型安全性を最優先（趣味の探求）？
├─ Yes → Gleam (若いが美しい) または OCaml (枯れてるが重い)
└─ No  → Rails に戻る

Q4: エンタープライズ規模が必要？
└─ ほぼ No → Java/Spring は除外
```

---

## スコアサマリー

| 言語/FW | スコア | 一言 |
|---|---|---|
| **Rails 8** | **9.0/10** | JSONテンプレート差し替えのための設計 |
| **Go** | **8.2/10** | クラウド最適、書きやすさは劣る |
| Gleam | 6.5/10 | 美しいがエコシステム若い |
| OCaml | 6.0/10 | 型安全だがWeb骨組みが辛い |
| Java/Spring | 4.5/10 | オーバーエンジニアリング |

---

## 次アクション

1. omni-survey を Rails 8 で再実装（Cloud Run デプロイ）
2. `templates/*.json` を `surveys` テーブルの seed に
3. `responses` テーブルは `survey_id` + `data jsonb`
4. Active Job で BigQuery 投入 + スプシエクスポート
5. Firebase Hosting は静的フロントのみに（APIはCloud Run）

---

*ADR-013 / omni-survey Web フレームワーク選定 / 2026-07-19*
