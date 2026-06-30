# ADR-001: 関数型言語への開発移行

> **Status**: 提案 (Proposed)
> **Date**: 2026-06-30
> **Project**: Stray Dog (`yon-straydog/`)
> **Author**: 弁慶 (Benkei Agent)

---

## 1. コンテキスト

Stray Dog は現在 **TypeScript + Vite** で構築された SPA である。

### 現在のコード特性

| 特性 | 値 |
|---|---|
| 総行数 | 680行 (main.ts) |
| 関数数 | 22 |
| 可変状態 (`let`) | 8つ（state, completed, currentSpot, map, userMarker, spotMarkers, userPos, gpsWatchId） |
| DOM 操作 | 22 `getElementById`, 18 `classList`, 23 `style` |
| 状態遷移 | 4状態 (`intro` → `puzzle4x4` → `home` → `map`) |
| 外部依存 | Leaflet.js (地図), Vite (ビルド), Surge (デプロイ) |
| 永続化 | localStorage (4キー) |

### 問題点（現状の課題）

1. **可変状態の散在** — 8つのモジュールレベル `let` 変数が非同期コールバックで更新される。GPSウォッチ、タイムアウト、アニメーションイベントが状態を非決定論的に変更。
2. **副作用とロジックの混在** — DOM操作、localStorage、GPS、アニメーションが純粋ロジックと混ざっている。
3. **テスト不可能** — DOM に強結合しているため、ロジック単位のテストが書けない。
4. **状態バグの再現困難** — `state` 変数とDOMの表示状態が同期していないケースがある（例: puzzle4 の close 後の状態不整合）。

---

## 2. 評価対象

### 2.1 Elm

**長所:**
- 純粋関数型。ランタイムエラーが実質ゼロ（`null` や `undefined` が存在しない）
- The Elm Architecture (TEA) が SPA の状態管理に完璧に適合
- コンパイラが全分岐を強制 → 状態遷移の網羅性保証
- パターンマッチングが DOM 操作の代わりに `Html msg` 型で宣言的に書ける
- バンドルサイズが小さい（この規模なら ~30KB gzipped）

**短所:**
- 言語学習曲線が急（Haskell系構文、カスタム型、ポート機構）
- Leaflet.js との相互運用に **Ports** が必要（地図操作は Elm 内で完結しない）
- 開発速度が TypeScript より遅い（コンパイル時間 + Ports 設計）
- チーム/ユーザーが Haskell 経験ない場合ハードルが高い
- JavaScript エコシステムとの相互運用が Ports 経由に制限される

**適合性:** ★★★★☆（SPA + 状態機械に最適だが Leaflet Ports がネック）

### 2.2 ReScript

**長所:**
- OCaml 系の型システム（強力な型推論、バリアント型、パターンマッチング）
- JavaScript 出力が読みやすく高速
- Vite プラグインあり（`@jihchi/vite-plugin-rescript`）→ 既存ビルドをほぼ維持
- JS 相互運用が Elm より自然（`@module` で直接インポート）
- 段階的移行が可能（`.res` と `.ts` の共存）

**短所:**
- コミュニティが Elm より小さい
- React 前提のエコシステム（`@rescript/react`）
- 純粋関数型ではない（副作用を許容）→ 規律は開発者依存
- 日本語ドキュメントがほぼ皆無

**適合性:** ★★★☆☆（JS相互運用は良いが、React前提がこのプロジェクトにオーバーキル）

### 2.3 ClojureScript

**長所:**
- ユーザーがすでに `world-model` で Clojure 使用中 → スキルセットが活きる
- REPL 駆動開発が強力（ブラウザ上でライブコーディング）
- 不変データ構造が標準（`atom` で管理された状態）
- Shadow-cljs が Vite 的な役割を果たす
- マクロによる DSL（例: パズル定義 DSL を自作できる）

**短所:**
- ランタイムが大きい（Closure Library + core.cljs → ~200KB gzipped）
- Leaflet.js との相互運用が `js/` プレフィックスで冗長
- JVM 依存（ビルド時に Java 必要）
- 静的型がない → バグは実行時まで発見されない
- コミュニティ縮小傾向（Clojure 全体のトレンド）

**適合性:** ★★★☆☆（Lisp 慣れしているが、静的保証がないのが痛い）

### 2.4 TypeScript + fp-ts / Effect-TS

**長所:**
- **段階的移行可能** — 既存コードを壊さずに純粋関数部から導入
- ビルドシステムそのまま（Vite）
- Leaflet.js の型定義がすでにある
- 学習コストが低い（TypeScript のまま、ライブラリ追加のみ）
- チームがすでに TypeScript に習熟

**短所:**
- fp-ts の型エラーメッセージが難解（HKT エミュレーションの複雑さ）
- 副作用制御が「紳士協定」レベル（コンパイラ強制ではない）
- Effect-TS はまだ v1.0 に達していない
- 純粋関数型の恩恵（全域性、網羅性）は得られない
- ボイラープレートが多い（`pipe`, `flow`, `ReaderTaskEither` など）

**適合性:** ★★★☆☆（現実的だが中途半端になるリスク）

---

## 3. 決定要因マトリックス

| 要因 | 重み | Elm | ReScript | ClojureScript | TS+fp-ts |
|---|---|---|---|---|---|
| 型安全性（コンパイラ強制） | 25% | 10 | 8 | 3 | 5 |
| JS相互運用（Leaflet, DOM） | 20% | 4 | 8 | 6 | 10 |
| 学習曲線 / 開発速度 | 20% | 4 | 5 | 6 | 9 |
| 状態管理の適合性 | 20% | 10 | 7 | 7 | 6 |
| ビルドサイズ | 5% | 10 | 8 | 4 | 8 |
| エコシステム成熟度 | 5% | 7 | 5 | 6 | 10 |
| 既存スキルセット | 5% | 2 | 2 | 8 | 10 |
| **加重スコア** | **100%** | **6.55** | **6.55** | **5.55** | **7.80** |

---

## 4. 推奨: TypeScript + fp-ts（段階的導入）

### 理由

1. **リスク最小**: 既存の 680行を一度に書き換える必要がない。純粋関数から順次抽出。
2. **期限**: YON 2周年（7/3-7/5）までに動くものを維持できる。
3. **実利優先**: Leaflet.js との相互運用が最も自然。GPS/地図を Elm Ports で再実装するより、既存資産を活かす。
4. **教育効果**: fp-ts のパターン（`Option`, `Either`, `Task`）を導入しながら、TypeScript のまま型安全性を徐々に高められる。

### 実装ロードマップ

```
Phase 0 (今すぐ):    状態を zustand ストアに集約（let → store）
Phase 1 (1-2日):    型定義を fp-ts 化（Option, Either 導入）
Phase 2 (2-4日):    パズルロジックを純粋関数に抽出（pipe, flow）
Phase 3 (4-7日):    画面遷移を TEA パターンで再実装
Phase 4 (将来):     Effect-TS で副作用を完全分離（成熟したら）
```

### Elm は第二候補

Stray Dog の本質は **状態機械 + テキストアドベンチャー** であり、
The Elm Architecture との相性は極めて高い。
7月のイベント後、時間的余裕ができたら Elm へのフルリライトを検討する価値がある。

---

## 5. 実際のコード比較

### 現在 (TypeScript 命令型)
```typescript
let selectedIdx: number | null = null
let moves = 0

function onTap(idx: number): void {
  if (goBtn.classList.contains('show')) return
  if (selectedIdx === null) { selectedIdx = idx; renderGrid(); return }
  if (selectedIdx === idx) { selectedIdx = null; renderGrid(); return }
  // swap...
}
```

### fp-ts 化後
```typescript
import { Option, none, some, match } from 'fp-ts/Option'
import { pipe } from 'fp-ts/function'

type PuzzleState = { selected: Option<number>; moves: number; tiles: Tile[] }

const onTap = (idx: number) => (s: PuzzleState): PuzzleState =>
  pipe(
    s.selected,
    match(
      () => ({ ...s, selected: some(idx) }),              // 未選択 → 選択
      (prev) => prev === idx
        ? { ...s, selected: none }                         // 同じタイル → 解除
        : swapTiles(prev, idx)(s)                          // 異なる → 交換
    )
  )
```

### Elm 化後
```elm
type alias PuzzleState = { selected : Maybe Int, moves : Int, tiles : List Tile }

onTap : Int -> PuzzleState -> PuzzleState
onTap idx state =
    case state.selected of
        Nothing -> { state | selected = Just idx }
        Just prev ->
            if prev == idx then { state | selected = Nothing }
            else swapTiles prev idx state
```

---

## 6. 結論

| 判断 | 内容 |
|---|---|
| **今すぐやること** | TypeScript のまま zustand で状態集約、副作用を関数境界に分離 |
| **次の一手** | fp-ts で `Option` / `Either` 導入、純粋関数部から段階的に |
| **長期ビジョン** | Stray Dog が拡張されるなら Elm リライトを検討（TEA との圧倒的適合性） |
| **やらないこと** | ClojureScript（ランタイム重い）、ReScript（React前提） |

---

*本 ADR は 2026-07-15 までに再評価予定（イベント後の振り返り時）*
