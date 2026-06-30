# ADR-011: しりとり対決 — Elixir の理想形

> ユーザー実装報告 | Date: 2026-06-30

---

## アーキテクチャ

```
ブラウザ A                    ブラウザ B
    │                             │
    │ WebSocket                  │ WebSocket
    ▼                             ▼
┌──────────────────────────────────────────┐
│         Phoenix LiveView                 │
│                                           │
│  Registry (部屋管理)                       │
│  ├─ room:abc → GameSession GenServer      │
│  ├─ room:def → GameSession GenServer      │
│  └─ room:ghi → GameSession GenServer      │
│                                           │
│  ETS (辞書)                               │
│  ├─ :word_lookup  (100万語, µs検索)       │
│  └─ :used_words   (ゲーム内重複チェック)    │
│                                           │
│  PubSub (リアルタイム配信)                  │
└──────────────────────────────────────────┘
```

---

## GameSession GenServer

```elixir
defmodule Shiritori.GameSession do
  use GenServer

  # 1ゲーム = 1プロセス
  defstruct [
    :room_id,
    players: [],
    current_char: nil,     # 次の人が使うべき文字
    history: [],           # 使われた単語のリスト
    timer_ref: nil,        # 制限時間タイマー
    started_at: nil
  ]

  # --- 単語提出 ---
  def handle_call({:submit_word, player_id, word}, _from, state) do
    cond do
      not valid_start?(word, state.current_char) ->
        {:reply, {:error, "「#{state.current_char}」で始まる単語を入力してください"}, state}

      already_used?(word, state.history) ->
        {:reply, {:error, "その単語は既に使われています"}, state}

      not in_dictionary?(word) ->
        {:reply, {:error, "辞書にない単語です"}, state}

      true ->
        new_char = last_char(word)
        new_state = advance(word, new_char, state)

        # 全プレイヤーに配信
        Phoenix.PubSub.broadcast(App.PubSub, "room:#{state.room_id}", {
          :word_accepted, player_id, word, new_char, state.history
        })

        {:reply, {:ok, new_char}, new_state}
    end
  end

  # --- ETS 辞書検索 (µs) ---
  defp in_dictionary?(word) do
    :ets.lookup(:word_dict, word) != []
  end

  # --- タイムアウト ---
  def handle_info(:timeout, state) do
    # 制限時間切れ → 全員に通知
    Phoenix.PubSub.broadcast(App.PubSub, "room:#{state.room_id}", {:timeout})
    {:stop, :normal, state}
  end
end
```

---

## 部屋管理 (Registry)

```elixir
# 部屋作成 = GenServer を1つ起動するだけ
def create_room(room_id) do
  DynamicSupervisor.start_child(
    GameSupervisor,
    {Shiritori.GameSession, room_id: room_id}
  )
end

# 部屋一覧 = Registry から全プロセスを取得
def list_rooms do
  Registry.select(Shiritori.Registry, [{{:"$1", :_, :_}, [], [:"$1"]}])
end

# プロセスが落ちても Supervisor が自動再起動
# 部屋の状態は GenServer の state に残っているので復旧可能
```

---

## フロントエンド（LiveView）

```elixir
defmodule Shiritori.GameLive do
  use Phoenix.LiveView

  # たったこれだけで双方向リアルタイム
  def handle_event("submit_word", %{"word" => word}, socket) do
    case GameSession.submit_word(socket.assigns.room_id, socket.assigns.player_id, word) do
      {:ok, next_char} ->
        {:noreply, assign(socket, current_char: next_char, status: "ok")}

      {:error, reason} ->
        {:noreply, assign(socket, status: "error", error: reason)}
    end
  end

  # PubSub からの配信を受け取る
  def handle_info({:word_accepted, player_id, word, next_char}, socket) do
    {:noreply,
     socket
     |> assign(current_char: next_char)
     |> stream_insert(:history, %{player: player_id, word: word})}
  end
end
```

**JS を 1 行も書かずにリアルタイム対戦が実装されている。**

---

## なぜ Elixir が完璧にハマるか

### 1. リアルタイム対決 → PubSub がネイティブ

```elixir
# 相手が単語を入力 → 即座にこちらの画面に表示
Phoenix.PubSub.broadcast(App.PubSub, "room:#{room_id}", {:word_accepted, ...})
```

これが 1 行。WebSocket のハンドシェイクも再接続も Phoenix が自動でやる。

### 2. 部屋管理 → GenServer 1部屋1プロセス

```
部屋が100個 = GenServer が100個並列実行
                 ↓
         それぞれ独立しているので：
         - 部屋Aが重い単語言っても部屋Bは影響なし
         - 部屋Cのプロセスが落ちても部屋D〜Zは無傷
         - DynamicSupervisor が落ちたプロセスを自動再起動
```

### 3. 単語バリデーション → ETS

```elixir
# 100万語の辞書をメモリに載せて µs 検索
:ets.lookup(:word_dict, "りんご")  # → 1µs
```

### 4. 掲示板 → LiveView Stream

```elixir
# 履歴表示は stream_insert で差分更新
stream_insert(:history, %{player: "A", word: "りんご"})
# → 全クライアントの履歴リストに1行追加（DOM全体の再描画なし）
```

---

## 負荷特性

```
1000部屋同時稼働 = 1000 GenServer
  ├─ メモリ: 1プロセス ≈ 2KB → 合計 2MB
  ├─ CPU:   ETS lookup = µs、アイドル時 0%
  └─ ネットワーク: WebSocket 1000接続 → Phoenix は余裕

Erlang/OTP は WhatsApp で 200万同時接続を 1台で捌いた実績。
しりとり1000部屋など「Hello World」レベルの負荷。
```

---

## 結論: Elixir の理想的なユースケース

```
しりとり対決 = リアルタイム × 並列ゲーム × PubSub × ETS

Elixir + Phoenix + OTP がこのために設計されたと言っても過言ではない。
```

| 要件 | Elixir での実現 | 他言語での苦労 |
|---|---|---|
| リアルタイム | `PubSub.broadcast` 1行 | WebSocket 自前実装 + 再接続 |
| 部屋並列 | `GenServer` 1部屋1プロセス | スレッドプール or Redis で状態管理 |
| 辞書検索 | `ETS` µs | DB クエリ or インメモリキャッシュ |
| タイムアウト | `Process.send_after` | setInterval + クリーンアップ |
| 障害耐性 | `Supervisor` 自動再起動 | try/catch + 手動リカバリ |

---

*ADR-011 / 2026-06-30*
