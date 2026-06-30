# ADR-012: ぷよぷよ実装 — リアルタイムゲームの言語選定

> ユーザー分析 | Date: 2026-06-30

---

## 評価サマリー

| 方式 | 適性 | 一言 |
|---|---|---|
| **TS + Canvas 2D** | **◎** | 0依存。フレームループ直書き。最速 |
| Phaser.js | △ | ミニゲームに 1MB は重い |
| React/Vue | ✗ | DOM はリアルタイム落下に向かない |
| Elm | ○→△ | TEA は完璧だが「キー押しっぱなし」が苦手 |

---

## Elm がリアルタイムゲームに向かない理由

```elm
-- Elm のキー入力
subscriptions : Model -> Sub Msg
subscriptions _ =
    Sub.batch
        [ Browser.Events.onKeyDown keyDecoder   -- 押した瞬間だけ
        , Browser.Events.onKeyUp keyDecoder     -- 離した瞬間だけ
        ]

-- 「押し続ける = 左に移動し続ける」が Elm では表現しにくい
-- onKeyDown は最初の 1回しか発火しない（OSのキーリピート頼み）
-- ゲームループ内で「現在押されているキー」をポーリングできない
```

Puyo Puyo に必要なもの:
- キーを押し続ける → 左に移動し続ける
- 連鎖中は入力をブロック
- 落下速度の変化（レベルに応じて）
- 60fps のゲームループ

→ **Canvas 2D + requestAnimationFrame + 生の keyboard event**

---

## アーキテクチャ

```
src/game/puyo/
├── types.ts    # 型定義 (Grid, Puyo, GameState, Action)
├── logic.ts    # 純粋関数（テスト可能、将来Elm移植可）
│   ├── drop()      # 落下判定
│   ├── rotate()    # 回転
│   ├── erase()     # 4連結判定 → 消去
│   ├── chain()     # 連鎖落下 → 再判定
│   └── gameOver()  # 天井到達
├── view.ts     # 副作用（Canvas 描画のみ）
│   ├── renderGrid()
│   ├── renderPuyo()
│   └── renderParticles()
└── main.ts     # ゲームループ + キー入力
    ├── gameLoop (requestAnimationFrame)
    ├── keyState (Set<string> ← 現在押されているキー)
    └── update → logic.* → view.*
```

---

## logic.ts（純粋関数）

```typescript
// 完全に純粋。Canvas も DOM も触らない
// → Jest/Vitest でテスト可能
// → 後で Elm に移植するときもそのまま port

type Grid = (PuyoColor | null)[][]  // 12x6

function eraseConnected(grid: Grid): { grid: Grid; erased: number } {
  // 4連結の同色を探索 → 消去 → 再帰
}

function applyGravity(grid: Grid): Grid {
  // 消去後の落下処理
}

function checkChain(grid: Grid): ChainResult {
  // 連鎖判定 → 連鎖ありなら再帰
}
```

---

## view.ts（副作用層）

```typescript
// Canvas 描画のみ。ロジックを一切含まない
function renderGrid(ctx: CanvasRenderingContext2D, grid: Grid): void { ... }
function renderActivePuyo(ctx: CanvasRenderingContext2D, puyo: ActivePuyo): void { ... }
function renderParticles(ctx: CanvasRenderingContext2D, particles: Particle[]): void { ... }
```

---

## main.ts（ゲームループ）

```typescript
const keyState = new Set<string>()

window.addEventListener('keydown', e => keyState.add(e.key))
window.addEventListener('keyup',   e => keyState.delete(e.key))

function gameLoop(timestamp: number) {
  const delta = timestamp - lastTimestamp

  // 入力 → ロジック
  const action = inputToAction(keyState)
  gameState = update(gameState, action, delta)

  // 描画
  render(ctx, gameState)

  lastTimestamp = timestamp
  requestAnimationFrame(gameLoop)
}
```

---

## 結論

```
リアルタイムゲーム = TypeScript + Canvas 2D

理由:
  1. requestAnimationFrame で 60fps を直接制御
  2. keydown/keyup の Set で「押しっぱなし」を自然に扱える
  3. logic.ts は純粋関数 → テスト可能 → 将来 Elm 移植も容易
  4. 依存ゼロ。バンドル 10KB 以下
  5. Stray Dog の Vite 環境をそのまま使える
```

---

*ADR-012 / 2026-06-30*
