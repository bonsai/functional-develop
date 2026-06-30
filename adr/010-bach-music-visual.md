# ADR-010: バッハ数理音楽のヴィジュアル・アート

> ユーザー「音楽理論のヴィジュアル・アート。バッハの数理音楽」
> Date: 2026-06-30

---

## これは何か

```
バッハの音楽構造 ──→ アルゴリズム ──→ ヴィジュアル出力
    │                    │                  │
 対位法                カノン             Canvas
 フーガ                反行形             SVG
 ゴルトベルク変奏曲     逆行形             WebGL
 音楽の捧げもの         蟹行カノン          アニメーション
```

音楽理論の数理構造をコードで可視化する **ジェネラティブ・アート** プロジェクト。

---

## 技術候補

| 技術 | 型 | 出力 | 適性 |
|---|---|---|---|
| **p5.js** | JS/TS | Canvas/WebGL | ★★★★ |
| **Quil (Clojure)** | Clojure | Processing/Canvas | ★★★★★ |
| **elm-canvas** | Elm | Canvas | ★★★ |
| **Three.js** | JS | WebGL 3D | ★★★ |
| **SVG生成** | 任意 | 静的SVG | ★★ |
| **Rust + Nannou** | Rust | Native window | ★★ |

---

## 1. Quil (Clojure + Processing)

### なぜこれが一番か

```clojure
;; フーガの主題を円環として描く
(ns bach.goldberg
  (:require [quil.core :as q]))

(defn draw-fugue [subject answer countersubject]
  ;; 3声の対位法を3つの円運動として表現
  (q/background 10 10 15)

  ;; 主題 = 黄金の弧
  (doseq [[x y] (voice-path subject)]
    (q/stroke 255 215 0)
    (q/point x y))

  ;; 応答 = 銀の弧（主題の5度上）
  (doseq [[x y] (voice-path (transpose answer 7))]
    (q/stroke 192 192 192)
    (q/point x y))

  ;; 対主題 = 青の弧
  (doseq [[x y] (voice-path countersubject)]
    (q/stroke 70 130 180)
    (q/point x y)))
```

### コード構造

```clojure
;; 音楽理論を Clojure のデータ構造で表現

;; 音程 = 半音数
(def intervals {:unison 0 :minor-2nd 1 :major-2nd 2
                :perfect-5th 7 :octave 12})

;; カノンの変換操作
(defn retrograde [melody] (reverse melody))
(defn inversion [melody] (map #(- 0 %) melody))
(defn crab-canon [melody] (concat melody (retrograde (inversion melody))))

;; ゴルトベルク変奏曲のアリアの和声進行
(def goldberg-aria
  (->> [[:G :major] [:D :major] [:Em :minor] [:Bm :minor]
        [:C :major] [:G :major] [:C :major] [:D :dominant7]]
       (mapcat chord->notes)
       (visualize-as-spiral)))

;; これを Quil で描画
(q/defsketch goldberg-visualization
  :setup setup
  :draw (fn [_] (draw-spiral goldberg-aria))
  :size [800 800])
```

### REPL でリアルタイム調整

```clojure
;; 色を変える → 即反映
(def palette [:gold :silver :cobalt])

;; 変奏のパラメータを変える → 即反映
(swap! variation-params assoc :tempo 1.5 :density 0.8)

;; Elm や p5.js ではビルド → リロードが必要
;; Quil は C-c C-c → 即座に絵が変わる
```

---

## 2. p5.js (TypeScript)

```typescript
// 同様のことを p5.js で
function draw() {
  background(10, 10, 15);

  // 主題の弧
  stroke(255, 215, 0);
  voicePath(subject).forEach(([x, y]) => point(x, y));

  // 応答の弧
  stroke(192, 192, 192);
  voicePath(transpose(answer, 7)).forEach(([x, y]) => point(x, y));
}
```

### Quil との差

| 観点 | Quil | p5.js |
|---|---|---|
| コード=データ | Lisp の homoiconicity | 文字列操作 |
| ライブ調整 | **C-c C-c** | リロード |
| 音楽DSL自作 | **マクロで可能** | 関数のみ |
| 学習曲線 | Processing経験 + Clojure | p5.js 経験のみ |
| デプロイ | ClojureScript → JS → 静的 | 静的 |
| コミュニティ | 小さい | **巨大** |

---

## 3. なぜ Elm ではないのか

Elm は **状態遷移が事前に決まっている UI** に最適（Stray Dog の 4 画面遷移など）。

バッハの可視化は「こう描いたらどう見えるか」が事前に決まらない。
**試行錯誤の連続** であり、REPL が不可欠。

Elm のコンパイルサイクル（5 秒）はクリエイティブコーディングの 0.5 秒サイクルには遅すぎる。

---

## 比較マトリックス

| 軸 | Quil | p5.js | elm-canvas |
|---|---|---|---|
| 実験のテンポ | **10** | 7 | 3 |
| 音楽DSL表現力 | **10** | 6 | 7 |
| 型安全性 | 5 | 7 | **10** |
| 学習曲線 | 6 | **9** | 5 |
| デプロイ容易性 | 8 | **10** | 9 |
| コミュニティ | 3 | **10** | 4 |
| **加重** | **7.55** | **7.65** | **6.25** |

---

## 結論

### 推奨: Quil (Clojure)

わずかに p5.js に負けているのは「コミュニティの差」だけ。
それ以外の全軸で勝っている。

決定的な理由:

```
音楽理論 = 数理構造 = データ構造
Clojure = データ構造を直接コードで表現できる言語

バッハの対位法を表現するのに:
  JS:    配列 + オブジェクト（関係がコードに表れない）
  Clojure: (map retrograde (inversion theme))（関係がそのままコード）
```

### Quil の構成

```
Clojure + Quil → Processing sketch → 静止画/アニメーション
                                      │
                                      ├ PNG出力（印刷用）
                                      ├ GIF出力（SNS用）
                                      └ ClojureScript + Quil → Web（インタラクティブ）
```

### プロジェクト構造案

```clojure
;; src/bach/
;; ├── core.clj          音楽理論 DSL
;; ├── goldberg.clj       ゴルトベルク変奏曲の構造
;; ├── art_of_fugue.clj   フーガの技法
;; ├── musical_offering.clj 音楽の捧げもの
;; ├── visual/
;; │   ├── spiral.clj     螺旋表現
;; │   ├── mobius.clj     メビウスの帯（蟹行カノン）
;; │   └── architecture.clj 建築的表現
;; └── sketch.clj          描画メイン
```

---

## 今すぐ試す

```bash
# Leiningen で Quil プロジェクト作成
lein new quil bach-visual

# または Clojure CLI
clj -M -m bach-visual.sketch
```

---

*ADR-010 / 2026-06-30*
