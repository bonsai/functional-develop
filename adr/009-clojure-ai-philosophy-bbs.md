# ADR-009: Clojure でやるべきもの

> ユーザー問い「CLJでやるべきなのは何？哲学的議論をAIとする掲示場？？？」
> Date: 2026-06-30

---

## なぜこの問いが正しいか

Clojure は「なんでもできる」が「なんにでも向いている」わけではない。

ADR-001〜008 で見たように、ユースケースごとに最適言語は異なる。

**問いを立て直す: Clojure が他の言語より圧倒的に有利になる領域は何か。**

---

## Clojure の唯一無二の武器

```
REPL駆動開発 = 動いているプログラムに生きたままコードを注入する能力
```

これは Elm にも Haskell にも TypeScript にも真似できない。

### 具体例: AI哲学議論掲示板での REPL

```clojure
;; ブラウザで動いている掲示板に、ライブで AI ペルソナを追加
(def personas
  [{:id :plato    :name "プラトン"   :prompt "あなたはプラトンです。イデア論の立場から..."}
   {:id :dogen    :name "道元"       :prompt "あなたは道元禅師です。只管打坐の立場から..."}
   {:id :marx     :name "マルクス"   :prompt "あなたはマルクスです。唯物史観の立場から..."}])

;; REPL でこのマップを評価 → 即座に掲示板に3人のAIが出現
;; Elm: コンパイル → デプロイ → 確認（5分）
;; Clojure: C-c C-c（0.5秒）
```

---

## AI哲学掲示板の要件分析

| 要件 | 内容 | CLJ適性 |
|---|---|---|
| スレッドツリー | 分岐する議論構造 | ★★★★★ zippers |
| AIペルソナ管理 | 複数の哲学者AIを切り替え | ★★★★★ データ駆動 |
| プロンプト実験 | 即座にプロンプト変更 → 応答確認 | ★★★★★ REPL |
| テキスト処理 | 引用、要約、翻訳 | ★★★★★ core lib |
| リアルタイム | AI 応答のストリーミング表示 | ★★★★ core.async |
| 議論の可視化 | 論点マップ、合意/対立グラフ | ★★★ Datascript |

---

## なぜ Clojure が AI 議論掲示板に最強か

### 1. プロンプトエンジニアリング = REPL の本領

```clojure
;; 「この議論にプラトンが参加したらどう返すか？」
;; REPL で仮説 → 即確認 → 微調整のサイクルが数秒

(require '[ai-bbs.core :as ai])

;; 試し打ち
(ai/respond {:persona :plato
             :thread   "善とは何か"
             :context  (ai/thread-history "善とは何か")})
;; → "善のイデアとは..." (0.5秒でレスポンス)

;; プロンプトを微調整
(swap! ai/personas assoc-in [:plato :temperature] 0.9)
(ai/respond ...)  ;; → より創造的な回答
```

**Elm や TypeScript だと: コード変更 → ビルド → ブラウザ再読込 → 同じ会話を再現 → 確認。Clojure はこれを 1 秒でやる。**

### 2. 議論データ構造がそのまま Clojure のデータ

```clojure
;; 議論スレッド = ネストした map と vector
{:id "thread-42"
 :title "自由意志は存在するか"
 :posts
 [{:id :p1 :author :human    :body "決定論が正しければ自由意志は幻想では？"}
  {:id :p2 :author :spinoza  :body "あなたの言う「自由」とは何か。定義を問いたい。"}
  {:id :p3 :author :human    :body "自己決定の感覚のことだ"}
  {:id :p4 :author :nietzche :body "「自己」などというものはない。主体は文法の産物だ。"}
  {:id :p5 :author :buddha   :body "我は無我なり。執着こそが苦の原因。"}]}

;; Clojure ではこのデータをそのまま REPL で操作できる
;; →  スレッドの特定の枝だけ抽出
;; →  特定の哲学者の発言だけ集計
;; →  議論の方向性をグラフ化
```

### 3. AI ペルソナがデータ駆動

```clojure
;; 新しい哲学者AIを追加する = マップを1つ追加するだけ
;; コードを一切変えずに UI が自動反映

(conj ai-personas
  {:id :zhuangzi
   :name "荘子"
   :avatar "🦋"
   :prompt "あなたは荘子です。胡蝶の夢の視点から..."
   :style {:tone "寓話的" :language "古文調" :max-length 200}})

;; → 掲示板に即座に荘子が出現
;; → 「荘子に質問する」ボタンが自動生成
```

### 4. core.async で AI ストリーミング

```clojure
(require '[clojure.core.async :as a])

(defn stream-ai-response [persona thread]
  (let [ch (a/chan)]
    ;; AI からのトークンを逐次配信
    (ai/stream-completion (:prompt persona) thread
      {:on-token (fn [token] (a/put! ch [:token token]))
       :on-done  (fn [full]  (a/put! ch [:done full]))})
    ch))

;; UI 側はこのチャンネルを購読してタイプライター表示
```

---

## 比較: なぜ Elm や Elixir ではないのか

| | Clojure | Elm | Elixir |
|---|---|---|---|
| AIプロンプト実験 | **REPL 0.5秒** | ビルド 5秒 | IEx 1秒 |
| データ構造操作 | **ネイティブ** | Decoder/Encoder | Map 操作 |
| 議論ツリー | **zippers** | 自前実装 | 自前実装 |
| AIストリーミング | **core.async** | Ports + JS | LiveView stream |
| 型安全性 | なし | ★★★★★ | ★★★ |

AI議論掲示板の核心は「実験のテンポ」。
プロンプトを変えて → 応答を見て → また変える。
このサイクルが **0.5 秒** で回る Clojure と、**5 秒〜5 分** かかる他言語では、
蓄積する試行回数が 10〜100 倍違う。

---

## 結論

```
Clojure でやるべきもの:
  AI 哲学議論掲示板 (AI Philosophy BBS)

理由:
  1. プロンプト実験が REPL で爆速（他言語の 10-100 倍の試行回数）
  2. 議論データ構造 = Clojure のデータ構造
  3. AIペルソナ追加 = マップ1つ追加するだけ
  4. core.async でストリーミング表示
  5. 型より対話。哲学の本質は REPL 的試行錯誤
```

### 構成案

```
Frontend:   ClojureScript + Re-frame + Shadow-cljs
Backend:    Clojure + Ring/Compojure + HTTP Kit
              ├ AI: OpenAI API (hato or clj-http)
              ├ DB: Datascript (client) + PostgreSQL (server)
              └ ストリーミング: core.async → SSE → UI
AI:         OpenAI / Claude API（プロンプト注入）
ホスト:     静的JS (surge.sh) + サーバー (Fly.io $5)
```

### 哲学掲示板の特徴

```
通常の掲示板:   ユーザー投稿 → ユーザー返信
哲学AI掲示板:   ユーザー投稿 → AI応答 → ユーザー応答 → AI応答...
                                      ↓
                                AI同士の議論を観戦
                                      ↓
                                プラトン vs 道元 vs ウィトゲンシュタイン
```

---

## Clojure は「実験の言語」

```
Elm     = 正しさをコンパイラで証明する言語
Elixir  = 多人数の同時接続を捌く言語
Haskell = 型で仕様を表明する言語
Clojure = 仮説を試し、壊し、また試す言語
```

AI 議論掲示板は「どういうプロンプトが面白い議論を生むか」が誰にもわからない。
だから **実験のテンポが最も速い** Clojure が最適。

---

*ADR-009 / 2026-06-30*
