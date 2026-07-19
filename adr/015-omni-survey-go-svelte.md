# ADR-015: omni-survey 再設計 — Go + Svelte + BigQuery（集計・可視化重視・軽量高速）

> Date: 2026-07-19 | Supersedes: ADR-013, ADR-014

---

## 要件の変化（なぜ見直すのか）

ADR-013/014 は「収集」を核心とした。しかし実運用で明確になった真の目的:

```
単なるアンケート収集  ≠  目的
収集後の「集計・可視化を綺麗に・素早く見せる」 ＝  目的
```

新要件:

| 要件 | 内容 |
|---|---|
| 収集 | JSONテンプレート差し替えで新規サーベイ即公開（維持） |
| **集計** | 回答をリアルタイム／準リアルタイムで集計 |
| **可視化** | 綺麗なチャート（円・棒・時系列）を即表示 |
| **軽量** | バックエンド起動数ms、フロントバンドル最小 |
| **速さ** | 開発も実行もサクッと。Rails の組む遅さ・起動重さは除外 |
| DB | BigQuery に全投入、集計は BQ クエリで直接 |

---

## 評価軸と重み（再設定）

| 軸 | 重み | 説明 |
|---|---|---|
| 集計クエリの容易さ | 25% | BigQuery への問い合わせ・集計記述 |
| コールドスタート/軽量 | 20% | Cloud Run/Functions での起動速度 |
| チャート可視化 | 20% | 綺麗な集計表示の容易さ |
| 開発速度 | 20% | 趣味運用ですぐ動かしたい |
| フロントバンドル重量 | 10% | モバイルファースト・軽量 |
| GCP SDK親和性 | 5% | BigQuery クライアントの充実度 |

---

## バックエンド候補: Go vs Rails（再評価）

### Go ★★★★★

```go
// 集計: BigQuery への直接クエリ（公式クライアント）
func aggregate(surveyID string) ([]Agg, error) {
    q := fmt.Sprintf(`
        SELECT field, COUNT(*) AS cnt
        FROM `%s.responses`
        WHERE survey_id = @sid
        GROUP BY field`, dataset, surveyID)
    // cloud.google.com/go/bigquery が公式・型安全
}

// 受付: 1バイナリ、コールドスタート数ms
func submit(w http.ResponseWriter, r *http.Request) {
    var p map[string]interface{}
    json.NewDecoder(r.Body).Decode(&p)
    bq.InsertRow(p) // jsonb 的に投入
}
```

- **BigQuery 公式 Go SDK** が最も充実（GCP 第1級言語）
- **単一バイナリ** → Cloud Run / Cloud Functions 両対応、コールドスタート数ms
- 集計クエリを `fmt.Sprintf` + パラメータで直書き、軽い
- Rails の「boot 数秒・Gemfile 地獄」が消える

### Rails ★★（棄却）

- 起動重い、組むの遅い（ユーザー要件と逆行）
- jsonb の楽さはあるが、集計は結局 BQ に逃がすので ORM 恩恵小
- 本要件ではオーバースペック

**→ BE = Go に決定**

---

## フロントエンド候補: Svelte vs Vue vs Elm vs **htmx**（再評価）

### htmx ★★★★★（軽量・ビルドレス・Go 親和）

```html
<!-- Go が集計HTMLを返す。フロント言語・ビルド不要 -->
<form hx-post="/api/submit" hx-target="#msg">
  <textarea name="content"></textarea>
  <button>送信</button>
</form>
<div id="msg"></div>

<!-- 集計: Go が BigQuery 結果を HTML で返す -->
<div hx-get="/api/agg/uber" hx-trigger="load" hx-swap="innerHTML"></div>
```

- **バンドルほぼ 0**（htmx ~14KB）。Svelte より軽い
- **Go が HTML を直接生成** → フロントエンド言語・ビルドステップが消える
- 集計ダッシュボードも「Go が集計HTML返し」で完結
- チャート描画は Chart.js 等を `<script>` で併用（サーバー側で data-JSON を埋め込む）

### Svelte ★★★★（インタラクティブ向け）

- バンドル最小級だが htmx の「0」には勝てない
- クライアント側での絞り込み・ソートは Svelte の方が楽
- ビルドステップ（Vite）が1つ増える

### Vue 3（ADR-014 案）★★★

- 情報量は多いがバンドルが Svelte/htmx より重い

### Elm ★★

- 厳密だが集計画面に過剰、学習コストが「素早く」と逆行

**→ FE = htmx を推奨（ビルドレス・最軽量）。Svelte は「インタラクティブ集計」が要る場合の代替**

---

## 追加候補: Gleam (JS target) を FE に使う場合

- Gleam で `html` ライブラリ（或いは Lustre）を使い、静的 HTML/JS を生成
- `gleam build --target javascript` → バンドルを Firebase/Cloud Run に配置
- htmx と併用も可（Gleam が生成した HTML に `hx-*` 属性を付ける）
- 小規模なら Gleam の型安全性を活かしつつ、ビルド成果物は軽量

### Go + Gleam の組合せ

- **BE: Go**（BigQuery SDK・コールドスタート速い）で API
- **FE: Gleam (JS)** で静的アセット生成、または htmx でビルドレス
- 小規模ゆえ「言語統一」より「書きやすさ」を優先しても良い

---

## 決定（小規模・柔軟方針）

```
omni-survey = Go (BE, BigQuery)  ＋  FE は以下から選択（小規模ゆえ柔軟）

  A. htmx    （推奨・ビルドレス・最軽量。Go が HTML 返し）
  B. Gleam   （型安全を活かしたい時。JS target で静的ビルド）
  C. Svelte  （インタラクティブ集計が要る時）

理由:
  - 小規模サービスなので「正解の一つ」に縛らず、実装のしやすさ優先
  - BE は Go に固定（BQ SDK・軽量・高速の要件を満たす）
  - FE は htmx を既定とし、Gleam/Svelte は好みで
  - Go+Gleam のハイブリッドも可（小規模なら言語混合のコスト微少）
```

---

## 決定木

```
Q1: 集計・可視化を重視する？
├─ Yes → BE は Go で固定
└─ No  → 収集のみなら更軽い構成でも可

Q2: FE をビルドレスにしたい？
├─ Yes → htmx（Go が HTML 返し）
└─ No  → Q3

Q3: 型安全な FE が欲しい？
├─ Yes → Gleam (JS target)
└─ No  → Svelte（インタラクティブ集計）
```

---

## スコアサマリー（新要件下）

| 構成 | スコア | 一言 |
|---|---|---|
| **Go + htmx** | **9.3/10** | ビルドレス・最軽量・Go親和 |
| Go + Gleam | 8.8/10 | 型安全・小規模に十分 |
| Go + Svelte | 8.6/10 | インタラクティブ集計向け |
| Go + Vue | 8.2/10 | 情報量重視 |
| Rails + Vue | 6.5/10 | 起動重く「サクッと」に反する |

---

## 次アクション

1. `omni-survey` を Go で再実装（Chi または標準 net/http）
2. `templates/*.json` をローカルまたは GCS から読込 → 動的フォーム
3. `POST /api/submit` → BigQuery insert
4. `GET /api/agg/:slug` → BQ 集計クエリ
5. `frontend/` を Svelte + Vite で作成（フォーム + ダッシュボード）
6. Cloud Run デプロイ（Blaze 必要）／ または Cloudflare で無料

---

*ADR-015 / omni-survey 再設計 (Go + Svelte + BigQuery) / 2026-07-19*
