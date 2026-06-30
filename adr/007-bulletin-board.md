# ADR-007: 掲示板の関数型言語評価

> ユーザー問い合わせ「elx掲示板にはいいかな elm」
> Date: 2026-06-30

---

## 掲示板の要件

| 要件 | 内容 |
|---|---|
| スレッド投稿 | ユーザーが新規スレッド作成 |
| 返信 | スレッド内でコメントツリー |
| リアルタイム | 他ユーザーの投稿が即反映 |
| 認証 | ログイン/登録（メール or OAuth） |
| ページネーション | スレッド一覧のページ送り |
| 管理機能 | 削除、ピン留め、通報 |

---

## Elixir + Phoenix の評価

```
掲示板 = リアルタイム × 多人数 × テキスト投稿
         ↑
   Elixir の得意領域ど真ん中
```

### Phoenix PubSub — リアルタイム掲示板

```elixir
# 投稿があったら全閲覧者にブロードキャスト
def handle_event("post", %{"body" => body}, socket) do
  post = Forum.create_post!(socket.assigns.thread, socket.assigns.user, body)

  # 同じスレッドを見ている全ユーザーに配信
  Phoenix.PubSub.broadcast(
    App.PubSub,
    "thread:#{post.thread_id}",
    {:new_post, post}
  )

  {:noreply, assign(socket, posts: [post | socket.assigns.posts])}
end
```

1行で「全クライアントに新着投稿を配信」が終わる。

### スレッドツリー（Ecto）

```elixir
# 再帰CTEでスレッドツリー表示
def thread_tree(thread_id) do
  Post
  |> where(thread_id: ^thread_id)
  |> order_by(asc: :inserted_at)
  |> Repo.all()
  |> build_tree()
end
```

### Phoenix 認証

```bash
mix phx.gen.auth Accounts User users
# → ログイン/登録/パスワードリセット/メール確認 が 1コマンド
```

### Elixir スコア: ★★★★★

---

## Elm の評価

Elm はフロントエンドのみ。掲示板を作るならバックエンドが必要。

### Elm + Elixir 構成

```
Elm (SPA)  ←──REST API──→  Elixir/Phoenix
   │                            │
   │──WebSocket──→              │──PubSub──→ 全クライアント
   │                            │
   └──Ports (WebSocket)──────→  └──Ecto/DB
```

Elm 側:

```elm
type alias Thread =
    { id : Int
    , title : String
    , posts : List Post
    , updatedAt : String
    }

-- API レスポンスを型安全にデコード
threadDecoder : Decoder Thread
threadDecoder =
    Decode.succeed Thread
        |> Decode.required "id" Decode.int
        |> Decode.required "title" Decode.string
        |> Decode.required "posts" (Decode.list postDecoder)
        |> Decode.required "updated_at" Decode.string
```

### Elm 単体では掲示板は作れない

- データベースがない
- 認証ができない（client-side only）
- リアルタイム配信できない（WebSocket は Ports → 結局サーバーが必要）

### Elm スコア: ★★★☆☆（バックエンド必須）

---

## 比較

| 要件 | Elixir (単独) | Elm + Elixir | Rails |
|---|---|---|---|
| リアルタイム | ★★★★★ | ★★★★ | ★★ |
| 認証 | ★★★★★ | ★★★★ | ★★★★★ |
| スレッドツリー | ★★★★★ | ★★★★ | ★★★★★ |
| 型安全性(API) | ★★★ | ★★★★★ | ★★ |
| 開発速度 | ★★★★★ | ★★★ | ★★★★★ |
| 学習曲線 | ★★★★ | ★★ | ★★★★★ |
| 1人で完結 | ★★★★★ | ★★★ | ★★★★★ |

---

## 推奨

```
掲示板 = Elixir + Phoenix (★★★★★)

理由:
  1. PubSub → リアルタイム投稿が 1行
  2. mix phx.gen.auth → 認証が 1コマンド
  3. Ecto → スレッドツリー、ページネーションが宣言的
  4. LiveView → JS をほぼ書かずにリアルタイム UI
  5. OTP → 落ちない（GenServer 監視ツリー）
```

### Elm はどこで入れるか

Elm を入れる価値があるケース:

- **管理画面が複雑** → Elm の型安全性でバグゼロ
- **API が公開** → Elm Decoder で型保証
- **SPA としてリッチ** → 掲示板自体は LiveView で十分

### 現実的な構成

```
Elixir + Phoenix LiveView → 掲示板本体（投稿、返信、スレッド一覧）
Elm SPA                   → 管理画面（ユーザー管理、通報処理、統計）
```

---

## Elm vs LiveView: 掲示板ではどちらが適切か

| | Elm SPA | Phoenix LiveView |
|---|---|---|
| 初期ロード | 速い（静的JS） | やや遅い（WebSocket確立） |
| リアルタイム | Ports で自前実装 | **組み込み（PubSub）** |
| SEO | 弱い（SPA） | **SSR 標準** |
| オフライン | **強い** | 弱い |
| 掲示板向き | ★★★ | **★★★★★** |

掲示板に Elm SPA はオーバーエンジニアリング。
**スレッド一覧 + 投稿フォーム + リアルタイム更新** は LiveView が最短。

---

```
結論: 掲示板 = Elixir + Phoenix LiveView（単独で完結）
      Elm は管理画面 or API 公開時の型保証レイヤーとして追加検討
```

---

*ADR-007 / 2026-06-30*
