# ADR-004: 多人数写真サイトの関数型言語評価

> 仮想ケース評価 | Date: 2026-06-30

---

## 要件定義（仮想）

| 要件 | 内容 |
|---|---|
| ユーザー | 認証あり（ログイン/登録） |
| 写真アップロード | 複数同時、JPEG/PNG、リサイズ |
| フィード | リアルタイム更新（他ユーザーの新着写真） |
| リアクション | 👍 💬 投稿ごと |
| アルバム | ユーザーごとに整理 |
| 同時接続 | 100-1000 ユーザー |
| モバイル対応 | レスポンシブ、プッシュ通知 |

---

## 1. 最重要要件: 同時接続 × リアルタイム

多人数写真サイトの核心は **「他ユーザーが写真を投稿したら、自分の画面にすぐ表示される」** こと。

これが言語評価を大きく変える。Stray Dog（ソロ・オフライン）では問題にならなかった要件。

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

| 要件 | Elm | Elixir/Phoenix | Haskell/Servant | Clojure(script) | TS+fp-ts |
|---|---|---|---|---|---|
| 同時接続 | ★★ | ★★★★★ | ★★★★ | ★★★ | ★★★ |
| リアルタイム更新 | ★★ | ★★★★★ | ★★★ | ★★★ | ★★★ |
| ファイルアップロード | ★★ | ★★★★★ | ★★★★ | ★★★ | ★★★★ |
| 認証 | ★★ | ★★★★ | ★★★★ | ★★★ | ★★★★ |
| DB クエリ | ★★ | ★★★★★ | ★★★★ | ★★★ | ★★★★ |
| 型安全性 | ★★★★★ | ★★★ | ★★★★★ | ★★ | ★★★ |
| フロントエンド | ★★★★★ | ★★★★ | ★★★ | ★★★★ | ★★★★ |

---

## 詳細評価

### Elixir + Phoenix LiveView ★★★★★

**なぜここで勝つのか:**

```elixir
# 写真アップロード → 全クライアントにブロードキャスト
def handle_event("upload", %{"photo" => photo}, socket) do
  {:ok, url} = PhotoStore.upload(photo, socket.assigns.user)
  Phoenix.PubSub.broadcast(App.PubSub, "feed", {:new_photo, url, user})
  {:noreply, socket}
end

# LiveView が自動的に全クライアントのフィードを更新
```

- **PubSub** が組み込み。全接続ユーザーに一斉配信が 1 行。
- **LiveView Upload** がファイルアップロード、プレビュー、プログレスバーを標準提供
- **Ecto** が DB クエリ（PostgreSQL の JSON/全文検索まで型安全）
- **OTP Supervisor** がアップロード処理の失敗を自動リトライ
- **Phoenix Channels** が WebSocket を抽象化

**弱点:**
- クライアントサイドのオフライン対応が弱い
- 型安全性は Haskell/Elm に劣る

---

### Haskell + Servant ★★★★

**型安全 API の極致:**

```haskell
-- API の型定義がそのまま実装になる
type PhotoAPI =
       "photos" :> Get '[JSON] [Photo]
  :<|> "photos" :> Capture "id" PhotoId :> Get '[JSON] Photo
  :<|> "upload"  :> AuthProtect "jwt" :> MultipartForm PhotoUpload :> Post '[JSON] Photo

-- Servant が自動生成するもの:
--   - 型安全な HTTP クライアント
--   - OpenAPI (Swagger) ドキュメント
--   - ルーティング
```

- **Servant** が API の型定義からドキュメント、クライアント、ルーティングを自動生成
- **Persistent/Esqueleto** が型安全 DB クエリ
- **STM** で同時実行制御（ロック不要）
- **JuicyPixels** で画像処理（純粋関数）

**弱点:**
- リアルタイム更新が得意でない（WebSocket は手組み）
- フロントエンドが別言語（Miso/GHCJS の重量）
- 開発速度が Elixir より遅い

---

### Elm（フロントエンドのみ）★★★

Elm 単体ではサーバーがないので評価不能。
**バックエンドに Elixir または Haskell を組み合わせる前提** で:

- Elm フロントエンド + Elixir バックエンド = **理想的な組み合わせ**
- Elm の型安全性 × Elixir の同時実行性 = 互いの弱点を補完

---

### Clojure + ClojureScript ★★★

- **同一言語フルスタック**（`.clj` + `.cljs`）
- **core.async** で CSP スタイルの同時実行
- **Sente** で WebSocket 双方向通信
- **Re-frame** でフロントエンド状態管理

**弱点:**
- 動的型 → 実行時エラーのリスク
- JVM のメモリ使用量（写真サイトには重い）

---

### TypeScript + fp-ts ★★★

- **Next.js / Remix** で SSR + アップロード
- **Prisma** で型安全 DB
- **Socket.io** でリアルタイム
- **fp-ts** で副作用管理

**弱点:**
- Node.js のシングルスレッド制約（写真処理でブロック）
- 同時接続は Cluster / PM2 頼み

---

## 最終レコメンデーション

```
多人数写真サイト = Elixir + Phoenix LiveView (★★★★★)

理由:
  1. PubSub → 全クライアントに一斉配信がネイティブ
  2. LiveView Upload → ファイルアップロードが標準装備
  3. OTP → 1000接続を1台で処理（WhatsApp は Erlang/OTP）
  4. Ecto → DBが型安全、かつマイグレーションが Rails 並に楽
  5. 開発速度 → Phoenix ジェネレータで認証・アップロードが数分
```

### 最強構成（二言語）

```
フロントエンド: Elm    (型安全性、バグゼロ)
バックエンド:   Elixir (同時接続、PubSub、アップロード)
DB:             PostgreSQL
ストレージ:     S3/Cloudflare R2
```

---

## なぜ Haskell ではないのか

Haskell は **単一リクエストの型安全性** では最強だが、
**多人数同時接続のリアルタイム性** は Elixir/Erlang の得意領域。

Erlang は電話交換機のために設計された（1986年、Ericsson）。
Elixir はその遺伝子を受け継いでいる。

> 写真サイト = 電話交換機的な問題（多数の同時接続 × メッセージ配信）
> Haskell = 証明器的な問題（1リクエストの完全な型安全性）

---

## スコアサマリー

| 言語 | スコア | 一言 |
|---|---|---|
| **Elixir + Phoenix** | **9.2/10** | この問題のために生まれた |
| Haskell + Servant | 7.8/10 | API は完璧、リアルタイムが弱い |
| Elm + Elixir | 9.5/10 | 理想だが二言語運用のコスト |
| Clojure(script) | 6.5/10 | フルスタックだが動的型が痛い |
| TS + fp-ts | 6.0/10 | できるが、Elixir の足元にも及ばず |

---

*ADR-004 / 仮想ケース評価 / 2026-06-30*
