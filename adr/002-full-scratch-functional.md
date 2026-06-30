# ADR-001 補遺: フルスクラッチ移行の評価

> **ADR-001 の続き** | Date: 2026-06-30

ADR-001 では段階的移行を前提に TypeScript + fp-ts を推奨した。
本補遺では **「全コードを破棄して書き直す」** 前提で再評価する。

---

## 候補一覧（フルスクラッチ）

| # | 言語/スタック | 実行場所 | 型 | ビルドターゲット |
|---|---|---|---|---|
| A | **Elm** | ブラウザ | 静的・純粋 | JS |
| B | **Elixir + Phoenix LiveView** | サーバー | 動的 | HTML over WebSocket |
| C | **PureScript** | ブラウザ | 静的・純粋 | JS |
| D | **Rust + Yew/Leptos** | ブラウザ | 静的 | WASM |
| E | **F# + Fable** | ブラウザ | 静的 | JS |

---

## A. Elm（フルスクラッチ）

### コード見本: 状態遷移

```elm
type GameState
    = Intro TypingState
    | Puzzle4x4 PuzzleState
    | MapView MapState
    | Complete

update : Msg -> GameState -> (GameState, Cmd Msg)
update msg state =
    case (msg, state) of
        (TypingFinished, Intro _) ->
            (Puzzle4x4 initPuzzle, Cmd.none)

        (PuzzleSolved hint, Puzzle4x4 _) ->
            (MapView initMap, saveProgress hint)

        _ ->
            (state, Cmd.none)
```

### 評価

| 項目 | 評価 | 備考 |
|---|---|---|
| 型安全性 | ★★★★★ | コンパイラが全状態遷移を検証。`case` の網羅性強制 |
| 状態管理 | ★★★★★ | TEA がゲームの画面遷移と完全に一致 |
| DOM 操作 | ★★★★☆ | 宣言的 `Html msg`。Canvas も `elm-canvas` で可能 |
| Leaflet 地図 | ★★☆☆☆ | **Ports 必須**。JS 側で Leaflet 操作 → Elm に `Msg` 送信 |
| GPS | ★★☆☆☆ | 同上。`navigator.geolocation` は Elm から直接呼べない |
| テキスト演出 | ★★★★★ | タイプライター、フェードインは Elm の遅延 `Cmd` で自然 |
| デプロイ | ★★★★★ | 静的HTML。surge.sh そのまま使える |
| バンドルサイズ | ★★★★★ | ~30KB gzipped |
| 学習曲線 | ★★★☆☆ | Haskell 系構文 + Ports 設計の理解が必要 |
| エコシステム | ★★★☆☆ | 停滞気味（Evan Czaplicki 離脱後）だが安定 |

**総合:** ★★★★☆ — SPA ゲームとして理想的。Leaflet/GPS の Ports 設計が最大の課題。

---

## B. Elixir + Phoenix LiveView

### アーキテクチャ

```
ブラウザ                         サーバー (Elixir)
┌──────────────┐    WebSocket    ┌──────────────────────┐
│  Leaflet.js  │ ◄────────────► │  GenServer (game)    │
│  GPS watch   │   LiveView     │  ├─ state: GameState │
│  Canvas blur │   双向通信      │  ├─ puzzle logic     │
└──────────────┘                │  └─ OTP supervision  │
                                └──────────────────────┘
```

### コード見本: パズル状態機械 (OTP GenServer)

```elixir
defmodule Straydog.Puzzle4x4 do
  use GenServer

  # --- State ---
  defstruct tiles: [], selected: nil, moves: 0, solved: false

  # --- Client API ---
  def tap_tile(pid, idx) do
    GenServer.call(pid, {:tap, idx})
  end

  # --- Server callbacks ---
  def handle_call({:tap, idx}, _from, %{solved: true} = state) do
    {:reply, {:already_solved, state}, state}
  end

  def handle_call({:tap, idx}, _from, %{selected: nil} = state) do
    {:reply, {:selected, idx}, %{state | selected: idx}}
  end

  def handle_call({:tap, idx}, _from, %{selected: prev} = state) do
    new_state = swap_tiles(state, prev, idx)
    solved = all_correct?(new_state.tiles)
    {:reply, {:swapped, solved}, %{new_state | selected: nil, solved: solved}}
  end
end
```

### LiveView テンプレート（タイプライター演出）

```elixir
# ~H""" テンプレート
def render(assigns) do
  ~H"""
  <div id="intro" phx-hook="Typewriter" data-lines={Jason.encode!(@lines)}>
    <div id="intro-text"><%= @typed_text %></div>
    <button :if={@finished} phx-click="start-puzzle">タップして次へ</button>
  </div>
  """
end
```

### 評価

| 項目 | 評価 | 備考 |
|---|---|---|
| 型安全性 | ★★★☆☆ | dialyzer で静的解析可能だが Elm/PureScript ほど強力でない |
| 状態管理 | ★★★★★ | OTP GenServer。耐障害性、タイムアウト、再起動戦略が組み込み |
| パターンマッチ | ★★★★★ | Elixir の本領。全状態遷移を `handle_call` で網羅 |
| DOM 操作 | ★★★★☆ | LiveView が差分更新。JS フック (`phx-hook`) で Canvas/Leaflet |
| Leaflet 地図 | ★★★☆☆ | `phx-hook` で JS 側操作。Elm の Ports よりは自然だが依然分離 |
| GPS | ★★★☆☆ | 同上。クライアント JS → `pushEvent` → サーバー |
| テキスト演出 | ★★★★★ | タイマーは `Process.send_after` で自然。LiveView が差分を配信 |
| オフライン耐性 | ★☆☆☆☆ | **WebSocket 必須**。街歩きで圏外になるとゲームが止まる |
| デプロイ | ★★★☆☆ | Fly.io / Gigalixir が必要。静的ホスティング不可 |
| サーバーコスト | ★★☆☆☆ | 常時稼働サーバー必要（$5-25/月）。アクセス数に比例 |
| 学習曲線 | ★★★☆☆ | Elixir は比較的親しみやすいが OTP の理解が必要 |
| エコシステム | ★★★★☆ | Phoenix は成熟、LiveView は活発に開発中 |

**総合:** ★★★☆☆ — テキストアドベンチャー + 状態機械のサーバーサイド実装としては秀逸。
だが **街歩きゲーム** としての最大の敵は「圏外」。

---

## C. PureScript

Haskell 系の純粋関数型で JS にコンパイル。
型システムは Elm より強力（型クラス、高階型）。しかしエコシステムが Elm よりさらに小さい。
Stray Dog 規模ならオーバーキル。

**総合:** ★★☆☆☆

---

## D. Rust + Yew/Leptos

Rust の型安全性 + WASM 実行。ゲームのパフォーマンス要件は皆無なので、
WASM のオーバーヘッド（~100KB+ 追加）に見合わない。
むしろテキストアドベンチャーに Rust は牛刀。

**総合:** ★☆☆☆☆

---

## E. F# + Fable

OCaml 系。F# コミュニティは小さい。Fable で JS 出力できるが、
Leaflet バインディングが存在しない。

**総合:** ★★☆☆☆

---

## 比較マトリックス（フルスクラッチ）

| 要因 | 重み | Elm | Elixir/LiveView | PureScript | Rust/WASM | F#/Fable |
|---|---|---|---|---|---|---|
| 型安全性 | 20% | **10** | 5 | 10 | 10 | 8 |
| 状態管理適合性 | 20% | **10** | 10 | 8 | 7 | 7 |
| オフライン耐性 | 15% | **10** | 2 | 10 | 10 | 10 |
| JS相互運用 | 15% | 5 | 6 | 6 | 4 | 5 |
| デプロイ容易性 | 10% | **10** | 4 | 8 | 6 | 7 |
| 学習曲線 | 10% | 5 | 6 | 3 | 2 | 4 |
| エコシステム | 5% | 5 | **8** | 3 | 6 | 3 |
| 開発速度 | 5% | 4 | 7 | 2 | 1 | 4 |
| **加重スコア** | **100%** | **8.15** | **6.35** | **6.80** | **6.25** | **6.35** |

---

## 結論（フルスクラッチ）

### 第1位: Elm (8.15)

Elm を推す理由：

1. **ゲームの本質とアーキテクチャが一致**: Stray Dog は「状態機械 + テキスト + パズル」。The Elm Architecture はまさにこのために設計されている。

2. **オフライン完結**: 街歩きで圏外になってもゲームは止まらない。Elixir の最大の弱点を回避。

3. **静的に保証されるバグの不在**: 680行が 22関数 + 8可変状態で構成されている TypeScript 版に対し、Elm 版は「コンパイルが通れば動く」。特に `selectedIdx` が `null` かどうかの分岐漏れ、`state` と DOM の不整合が起こりえない。

4. **デプロイが静的ファイル**: surge.sh そのまま。

5. **Leaflet/GPS は Ports で十分**: 地図表示とGPS取得は Elm の外で動かし、結果だけ `Msg` で受け取ればよい。パズルロジックと画面遷移は Elm 内で純粋に保たれる。

### Elixir が良いケース

Elixir + LiveView は以下の条件なら圧倒的に有利：

- マルチプレイヤー（複数プレイヤーが同じ犬を追う）
- リアルタイム協力パズル
- サーバーサイドでの不正防止
- 運営がプレイヤーの進行をリアルタイム監視

現状の Stray Dog はソロプレイなので、これらの恩恵はない。

---

## 推奨アクション

```
今 (7月イベントまで): TypeScript のまま完成させる
                    ↓
7月中旬:             Elm でのリライトを開始
                    ├─ Leaflet Ports の設計（1日）
                    ├─ 画面遷移 + パズルロジック（3日）
                    ├─ タイプライター演出（1日）
                    └─ GPS + localStorage（1日）
                    ↓
7月下旬:             Elm 版 リリース
```

---

## Elixir は「次の次」

Stray Dog が **マルチプレイヤー化** したり **運営ダッシュボード** が必要になったら、
Elixir + Phoenix が真価を発揮する。その時点でADRを再オープンする。

---

*評価完了: 2026-06-30*
