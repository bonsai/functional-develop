# ADR-014: omni-survey フロントエンド選定（関数型 Elm vs Vue）

> Date: 2026-07-19 | 前提: バックエンド = Rails 8 (ADR-013)、GCP/Cloud Run

---

## 要件定義

| 要件 | 内容 |
|---|---|
| 核心機能 | JSONテンプレート(title/description/fields[]) から動的フォームを描画 |
| フィールド型 | textarea / input / select / radio / checkbox 等 |
| 送信 | `/api/submit` へ JSON POST、成功時にメッセージ表示 |
| 配信 | Firebase Hosting または Cloud Run の静的アセット |
| 管理画面 | Rails 側で生成（ADR-013）。フロントは「回答フォーム」のみ |
| スコープ | モバイルファースト、シングルページ、軽量 |

---

## 最重要要件: 「JSON スキーマ → 再帰的フォーム描画」

テンプレートの `fields[]` をループしてコンポーネントを生成する。
ここがフロントエンド評価の分岐点。

---

## 候補

1. **Elm**（関数型、ADR-005 の流れを継承）
2. **Vue 3**（SFC + リアクティブ、日本勢に情報最多）
3. （参考）Rails サーバーサイド描画（Elm/Vue なし）

---

## 評価軸と重み

| 軸 | 重み | 説明 |
|---|---|---|
| 動的フォーム生成 | 25% | fields[] を再帰的に描画する容易さ |
| 型安全性 | 20% | テンプレート構造の型検証 |
| 開発速度 | 20% | 趣味運用ですぐ公開したい |
| 軽量さ/配信 | 15% | Firebase Hosting への静的バンドル |
| 保守性 | 10% | 1年後に触って読めるか |
| GCP/JS親和性 | 10% | Fetch/BQ連携のしやすさ |

---

## Elm（関数型）評価

### 動的フォーム生成

```elm
-- テンプレートの型
type Field
    = TextArea { name : String, label : String, placeholder : String }
    | Select { name : String, label : String, options : List String }
    | Input { name : String, label : String }

-- fields を map して描画（再帰的だが単純）
viewField : Field -> Html Msg
viewField field =
    case field of
        TextArea f ->
            textarea [ onInput (SetValue f.name) ] [ text f.placeholder ]
        Select f ->
            select [] (List.map (\o -> option [] [ text o ]) f.options)
        Input f ->
            input [ type_ "text", onInput (SetValue f.name) ] []
```

- **型がテンプレート構造を保証**。未知の field 型はコンパイルエラー
- **TEA** で「入力中の値」を union 型で管理 → バグが出にくい
- **ADR-005 の資産（Decoder/Httpモジュール）を流用可能**

### 弱点

- ビルド成果物は JS だが、**Elm 自体の学習コスト**が趣味運用で重い
- 管理服务（Rails）と別言語 → 2言語運用
- 小さなフォームに ELM アーキテクチャはやや大げさ

---

## Vue 3 評価

### 動的フォーム生成

```vue
<!-- SurveyForm.vue -->
<template>
  <form @submit.prevent="submit">
    <h1>{{ tpl.title }}</h1>
    <p>{{ tpl.description }}</p>
    <component
      v-for="f in tpl.fields"
      :key="f.name"
      :is="widgetFor(f.type)"
      v-model="answers[f.name]"
      :field="f"
    />
    <button>{{ tpl.submitButton }}</button>
  </form>
</template>

<script setup>
const props = defineProps(['tpl'])
const answers = reactive({})
const widgetFor = (type) =>
  ({ textarea: TextArea, select: Select, input: InputField })[type]
</script>
```

- **`v-for` + `:is` で fields[] を1行で描画**。動的生成が Vue の得意領域
- **リアクティブ** (`reactive`) で入力状態管理が自動
- **日本の情報量最多**、困ったらすぐ回答が見つかる
- **Vite** でビルド → Firebase Hosting に静的配置が標準
- Rails API との連携は `fetch` で十分

### 弱点

- 型は `any` になりやすく、テンプレート構造のミスは実行時まで発覚せず
- 厳密さは Elm に劣る（ただし `vue-tsc` + zod で補強可）

---

## Rails サーバーサイド描画（参考）

```erb
<% @tpl[:fields].each do |f| %>
  <%= render "fields/#{f[:type]", field: f %>
<% end %>
```

- フロント言語不要、**最も簡単**
- だが SPA の操作性（インライン検証、遷移なし送信）に劣る
- ADR-013 で「Rails で管理画面 + 回答受付」に含まれているので、フロント別言語は「体験向上」オプション

---

## 比較マトリクス

| 軸 | 重み | Elm | Vue 3 | Rails SSR |
|---|---|---|---|---|
| 動的フォーム生成 | 25% | 8 | **10** | 9 |
| 型安全性 | 20% | **10** | 5 | 4 |
| 開発速度 | 20% | 5 | **10** | **10** |
| 軽量さ/配信 | 15% | 7 | **9** | 6 |
| 保守性 | 10% | **10** | 7 | 8 |
| GCP/JS親和性 | 10% | 7 | **9** | 6 |
| **加重スコア** | 100% | **7.85** | **8.70** | 7.75 |

---

## 決定

```
omni-survey フロント = Vue 3 + Vite（静的 SPA）

理由:
  1. v-for + :is で fields[] 動的描画が最も簡潔（核心要件を直接満たす）
  2. 日本の情報量・学習コストの低さ → 趣味運用ですぐ公開
  3. Vite ビルド → Firebase Hosting / Cloud Run 静的配信が標準
  4. Rails API との疎結合（fetch のみ）で責務分離がきれい
  5. 型安全性は vue-tsc + zod で「十分な」レベルに持ち上げ可能

関数型を取りたい場合の代替:
  - Elm（ADR-005 の資産流用）だが、小さなフォームには学習コストが勝らず
  - 今は Vue で速度優先、将来「厳密さが必要な別プロジェクト」で Elm を温存
```

---

## 決定木

```
Q1: 今すぐ公開したい？（趣味運用・迅速）
├─ Yes → Vue 3 + Vite
└─ No  → Q2

Q2: 型安全性を絶対に担保したい？
├─ Yes → Elm（ADR-005 資産流用）
└─ No  → Vue 3 + Vite (vue-tsc + zod)

Q3: フロント言語を増やしたくない？
└─ Yes → Rails サーバーサイド描画（SPA体験は犠牲）
```

---

## スコアサマリー

| 選択肢 | スコア | 一言 |
|---|---|---|
| **Vue 3 + Vite** | **8.70/10** | 動的フォーム生成のための設計 |
| Elm | 7.85/10 | 厳密だが小さなフォームには過剰 |
| Rails SSR | 7.75/10 | 最も簡単、だがSPA体験なし |

---

## 次アクション

1. `frontend/` に Vue 3 + Vite プロジェクト作成
2. `SurveyForm.vue` で `fields[]` を再帰描画
3. `fetch('/api/submit')` で Rails へ POST
4. `npm run build` → `dist/` を Firebase Hosting / Cloud Run 静的配信
5. 型は `vue-tsc` + `zod`（テンプレート構造バリデーション）

---

*ADR-014 / omni-survey フロントエンド選定 / 2026-07-19*
