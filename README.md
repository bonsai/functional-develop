# functional-develop

アーキテクチャ決定記録 (ADR) — 関数型言語 × ユースケース評価

## ADR 一覧

| # | ケース | 選定 | キーワード |
|---|---|---|---|
| 001 | Stray Dog 段階的移行 | TS + fp-ts | 現実路線 |
| 002 | Stray Dog フルスクラッチ | **Elm** | TEA = 状態機械 |
| 003 | Haskell 評価 | 指針 | 型の極致、GP学習 |
| 004 | 多人数写真共有 | **Elixir + Phoenix** | 同時接続 |
| 005 | Rainbow Station Frontend | **Elm** or **Re-frame** | Frontend 選定 |
| 006 | ClojureScript フレームワーク | **Re-frame** | REPL + 状態管理 |
| 007 | 掲示板 | **Elixir + LiveView** | PubSub 1行 |
| 008 | 写真地図 3方式比較 | **Re-frame + Rails** | Rails 温存 |
| 009 | 哲学AI議論掲示板 | **Clojure** | プロンプト実験 |
| 010 | バッハ数理音楽可視化 | **Quil (Clojure)** | 音楽DSL = Lisp |
| 011 | しりとり対決 | **Elixir** | GenServer × ETS |
| 012 | ぷよぷよ | **TS + Canvas 2D** | 60fps ゲームループ |

## プロジェクト × 技術マップ

```
Soliloquy     → Elm
画像共有        → Elixir + Phoenix
掲示板          → Elixir + LiveView
写真地図        → Re-frame + Rails
哲学AI          → Clojure
数理音楽可視化   → Quil (Clojure)
リアルタイム対戦 → Elixir + OTP
リアルタイム落下 → TS + Canvas
```
