# ADR-002 補遺: Haskell 評価

> ADR-002 への追加 | ユーザー指名「ハスケル」

---

## Haskell 系フロントエンド

| フレームワーク | 方式 | 出力 | 成熟度 |
|---|---|---|---|
| **Miso** | TEA (Elm 互換アーキテクチャ) | GHCJS → JS | ★★★★ |
| **Reflex-DOM** | FRP (関数リアクティブ) | GHCJS → JS | ★★★ |
| **Obelisk** | Reflex + フルスタック | GHCJS + サーバー | ★★★ |
| **Asterius** | GHC → WASM | WASM | ★★ |

このプロジェクトに最も適合するのは **Miso**（The Elm Architecture を Haskell で書く）。

---

## Miso の評価

### コード見本: パズル状態機械

```haskell
{-# LANGUAGE OverloadedStrings #-}
import Miso
import Miso.String (MisoString, ms)

-- 純粋な型
data GameState
  = Intro [MisoString]      -- 表示済みテキスト行
  | Puzzle4x4 PuzzleModel   -- 4×4 パズル中
  | MapView MapModel        -- 地図表示中
  | Complete                -- クリア

data PuzzleModel = PuzzleModel
  { tiles    :: [(Int, Int)]   -- (現在位置, 正解位置)
  , selected :: Maybe Int
  , moves    :: Int
  }

data Action
  = TypeNextLine
  | TapTile Int
  | GpsUpdate Double Double
  | ...

-- update は純粋関数
updateModel :: Action -> GameState -> Effect Action GameState
updateModel TypeNextLine (Intro (_:[])) =
  -- 全文表示完了 → パズルへ
  noEff (Puzzle4x4 (shufflePuzzle initTiles))

updateModel (TapTile idx) (Puzzle4x4 m) =
  case selected m of
    Nothing  -> noEff (Puzzle4x4 (m { selected = Just idx }))
    Just prev
      | prev == idx -> noEff (Puzzle4x4 (m { selected = Nothing }))
      | otherwise   ->
          let m' = swapTiles prev idx m
          in if allCorrect (tiles m')
             then saveProgress *> noEff (MapView initMap)
             else noEff (Puzzle4x4 m')

-- Leaflet/GPS は Sub で購読
subs :: GameState -> [Sub Action]
subs _ = [gpsSub GpsUpdate]  -- JS 側から Action を受け取る
```

### Elm との比較

| 特性 | Elm | Miso (Haskell) |
|---|---|---|
| 型システム | 単相型 + 型推論 | 多相型 + **型クラス** + GADT + 型族 |
| パターンマッチ | 網羅性強制 | 網羅性強制 + **ViewPatterns** |
| 副作用管理 | Cmd, Sub, Ports | Effect, Sub（ほぼ同じ） |
| 最適化 | コンパイラ内蔵 | **GHC レベルの最適化** |
| DOM 差分更新 | 仮想DOM | 仮想DOM |
| JS 相互運用 | Ports（制限あり） | **FFI で直接呼出し可能** |
| 学習曲線 | 1-2週間 | **1-3ヶ月** |
| ビルド時間 | 数秒 | 数十秒〜分（GHCJS のコンパイル） |
| バンドルサイズ | ~30KB | **~200-500KB**（GHC ランタイム + Prelude） |
| ツールチェーン | `elm` CLI 一つ | Stack/Cabal + GHCJS |
| 日本語コミュニティ | 少数 | **ほぼ皆無** |

---

## ハスケルを選ぶ理由（もし選ぶなら）

1. **表現力の限界がない** — 型クラスでパズルDSLを自作できる。例:
   ```haskell
   class Puzzle a where
     shuffle  :: a -> IO a
     validate :: a -> Answer -> Bool
     hint     :: a -> Text
   ```

2. **不変データ構造が標準** — `Data.Map`, `Data.Sequence`, `Vector` が標準ライブラリ。

3. **GHC の最適化** — 型消去、インライン化、融合変換が強力。

4. **後々サーバーサイド** — 同じHaskellコードをサーバーでも実行可能（Servant, Yesod）。

---

## ハスケルを選ばない理由

1. **ビルド時間が長い** — GHCJS のコンパイルは Elm の 10-50 倍。開発のテンポが落ちる。

2. **学習曲線が急峻** — モナド変換子、レンズ、GADT、型族。1ファイルを書くのに3つの言語拡張が必要。

3. **バンドルが重い** — 500KB gzipped はモバイル回線で数秒。Stray Dog は軽量さが武器だった。

4. **エラーメッセージ** — GHC の型エラーは Elm より圧倒的に読みにくい。HKT のエラーは悪夢。

5. **チーム不在** — 現状 TypeScript 1人。Haskell を書ける開発者を追加するのは非現実的。

---

## 結論: 「憧れ」と「実用」の差

```
Elm      = 十分な型安全性 + 軽量 + 2週間で習得
Haskell  = 究極の型安全性 + 表現力 + 重い + 3ヶ月必要
```

**Stray Dog の規模（680行）に対して Haskell はオーバーキル。**

データサイエンティストが Excel で十分な集計に Spark クラスタを立てるようなもの。
表現力の代償（ビルド時間・バンドルサイズ・学習曲線）が、得られる利益を上回る。

### ただし

もし Haskell を「趣味」や「学習」として書くなら止めない。
Miso + GHCJS は技術的に健全で、書いていて楽しい。
**プロダクトの納期（7/3）を優先するなら TypeScript → 7月中旬以降に Miso 実験。**

---

## 最終スコア比較

| | TypeScript+fp-ts | Elm | Elixir/LiveView | **Haskell/Miso** |
|---|---|---|---|---|
| 段階的 | 10 | 2 | 2 | 2 |
| フルスクラッチ | 6 | **10** | 7 | 5 |
| 学習曲線 | 9 | 6 | 6 | **2** |
| 開発速度 | 9 | 7 | 7 | **3** |
| バンドル | 9 | 10 | N/A | **4** |
| 型の表現力 | 5 | 7 | 4 | **10** |
| 実用スコア | **8.0** | **7.8** | 6.0 | **4.2** |

---

*Haskell は「聖杯」だが、今この聖杯を求める旅に出る余裕はない。*
*まず犬を見つけよう。* 🐕
