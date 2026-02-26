# Supabase Realtime を Golang で書き直す際の問題点分析

## 1. 現行アーキテクチャの概要

Supabase Realtime は Elixir/Phoenix で構築されたリアルタイムサーバーで、以下の3つの機能を提供する:

- **Postgres Changes**: PostgreSQL の論理レプリケーション (WAL) を監視し、DB変更を WebSocket クライアントに配信
- **Broadcast**: クライアント間の低レイテンシなメッセージング
- **Presence**: クライアントの接続状態の追跡・同期

### データフロー (Postgres Changes)

```
PostgreSQL WAL → Logical Replication (pgoutput/wal2json)
    → Realtime Server (Elixir GenServer)
        → RLS フィルタリング (subscription_id ベース)
            → Phoenix PubSub
                → Phoenix Channel (WebSocket)
                    → クライアント
```

---

## 2. 問題カテゴリ別の分析

### 2.1 PostgreSQL 論理レプリケーション接続の再実装

#### 現行実装

- `Postgrex.ReplicationConnection` を使用し、PostgreSQL のストリーミングレプリケーションプロトコルを直接利用
- `pgoutput` プラグイン（新方式）と `wal2json` プラグイン（旧方式 poller）の2系統が存在
- WAL バイナリプロトコルを Elixir のパターンマッチで効率的にデコード (`lib/realtime/adapters/postgres/decoder.ex`)
- レプリケーションスロットの作成・管理、スタンバイステータスの応答

#### Golang での問題点

| 問題 | 詳細 | 深刻度 |
|------|------|--------|
| **レプリケーションプロトコルのライブラリ不足** | Go では `jackc/pglogrepl` が存在するが、Elixir の `Postgrex.ReplicationConnection` ほど成熟していない。特にエラーハンドリングやリカバリのロジックを自前で実装する必要がある | 高 |
| **WAL バイナリデコード** | PostgreSQL WAL のバイナリプロトコル (Begin, Commit, Relation, Insert, Update, Delete, Type, Origin メッセージ) を Go で正確にパースする必要がある。Elixir のパターンマッチは Go の `encoding/binary` よりも簡潔に書ける | 中 |
| **LSN 管理** | Log Sequence Number の追跡とスタンバイステータスの定期送信を正確に実装する必要がある。送信が遅れると PostgreSQL が WAL を保持し続けてディスクを圧迫する | 高 |
| **OID→型名マッピング** | Relation メッセージから列の OID を型名にマッピングするキャッシュを自前で管理する必要がある | 低 |

#### 推奨ライブラリ

- [`jackc/pglogrepl`](https://github.com/jackc/pglogrepl) - PostgreSQL 論理レプリケーションクライアント
- [`jackc/pgx`](https://github.com/jackc/pgx) - 基盤となる PostgreSQL ドライバ

---

### 2.2 Phoenix Channel プロトコルの再実装

#### 現行実装

- Phoenix Channel プロトコル v1.0.0 (JSON object) と v2.0.0 (JSON array + binary) をサポート
- イベント: `phx_join`, `phx_leave`, `heartbeat`, `access_token`, `broadcast`, `presence`, `postgres_changes`
- 25秒間隔の heartbeat メカニズム
- Fastlane 最適化（同一メッセージの複数クライアントへの効率的な配信）

#### Golang での問題点

| 問題 | 詳細 | 深刻度 |
|------|------|--------|
| **Phoenix Channel プロトコルの完全再実装** | Phoenix Channel は独自のメッセージングプロトコルであり、Go には互換ライブラリが存在しない。JSON/バイナリ両フォーマットを完全に実装する必要がある | 高 |
| **既存クライアント SDK との互換性** | `@supabase/realtime-js` 等のクライアントライブラリは Phoenix Channel プロトコルを前提としている。サーバー側で完全なプロトコル互換を維持するか、クライアント SDK も書き直す必要がある | 最高 |
| **Fastlane 最適化** | Phoenix の Fastlane は同一メッセージの複数宛先配信でシリアライズを1回に抑える最適化。Go では goroutine + channel で同等のパターンを設計する必要がある | 中 |
| **バイナリフレームフォーマット (v2.0.0)** | `USER_BROADCAST_PUSH (0x03)` / `USER_BROADCAST (0x04)` などのバイナリメッセージタイプの処理 | 中 |

#### 設計上の選択肢

1. **Phoenix Channel プロトコル完全互換** → クライアント SDK をそのまま利用可能だが実装コストが非常に高い
2. **独自 WebSocket プロトコル** → シンプルだがクライアント SDK の書き直しが必要
3. **Phoenix Channel の Go 実装をOSSとして開発** → コミュニティへの貢献にもなるが開発期間が長い

---

### 2.3 BEAM VM / OTP の利点の喪失

#### 現行実装が BEAM の恩恵を受けている箇所

| BEAM/OTP の機能 | Realtime での用途 | Go での代替 |
|----------------|-------------------|-------------|
| **軽量プロセス** | WebSocket 接続ごとに1プロセス。数万接続でもメモリ効率が良い | goroutine で代替可能だが、プロセス隔離（クラッシュ隔離）がない |
| **Supervisor ツリー** | テナントごと・接続ごとの障害分離。1接続のクラッシュが他に波及しない | `errgroup` やカスタム supervisor パターンが必要。障害伝播の制御が難しい |
| **OTP GenServer** | 状態を持つプロセス（レプリケーション接続、サブスクリプション管理）| Go の struct + mutex / channel パターンで代替 |
| **ETS テーブル** | サブスクリプション ID→ノードのマッピング、キャッシュ | `sync.Map` や外部キャッシュ (Redis) で代替 |
| **syn (グローバルレジストリ)** | テナントのプロセスをクラスタ全体で名前解決 | etcd / Consul / カスタムサービスディスカバリが必要 |
| **ホットコードリロード** | ダウンタイムなしでのコード更新 | ローリングデプロイで代替（WebSocket 接続の再接続が必要）|
| **Erlang 分散プロトコル** | ノード間のメッセージパッシング、RPC | gRPC / NATS / Redis Pub/Sub で代替 |
| **プロセスリンクとモニタ** | レプリケーション接続の監視とリカバリ | context.Context + goroutine 管理 |

#### 最も深刻な問題: 障害隔離

Elixir では、1つのテナントのレプリケーション接続がクラッシュしても Supervisor が自動再起動し、他のテナントには一切影響しない。Go ではこのレベルの障害隔離を実現するには:

- テナントごとに goroutine プールを管理
- panic recovery を各所に配置
- 再起動ロジックをカスタム実装
- 状態のリカバリメカニズムを設計

---

### 2.4 マルチテナント・マルチノードアーキテクチャ

#### 現行実装

- テナントごとに Supervisor ツリー、DB接続プール、レプリケーション接続を動的に生成
- `libcluster` + `libcluster_postgres` でクラスタ自動検出
- `gen_rpc` でノード間 RPC (Supabase 独自フォーク)
- `syn` でクラスタ全体のプロセスレジストリ
- テナントのリバランシング機構（ノード間でテナントを移動）

#### Golang での問題点

| 問題 | 詳細 | 深刻度 |
|------|------|--------|
| **動的テナント管理** | テナントの追加・削除に伴うリソース (DB接続プール、レプリケーション接続、goroutine) のライフサイクル管理が複雑 | 高 |
| **クラスタメンバーシップ** | Erlang の分散プロトコルに相当する仕組みがない。`memberlist` (HashiCorp) や etcd を使ったサービスディスカバリが必要 | 高 |
| **ノード間メッセージルーティング** | 特定のテナントがどのノードにいるかを追跡し、そのノードにメッセージを転送する仕組みが必要 | 高 |
| **テナントリバランシング** | ノード追加・削除時にテナントを安全に移動する仕組み。WebSocket 接続の graceful migration が難しい | 中 |
| **コネクションプール管理** | テナントごとに独立した DB 接続プールを動的に作成・破棄する仕組み (`pgxpool`) | 中 |

---

### 2.5 RLS (Row Level Security) フィルタリング

#### 現行実装 (2つの方式)

**方式1: 旧 CDC RLS ポーラー (`wal2json` + `realtime.list_changes()`)**
- PostgreSQL のストアドプロシージャ `realtime.list_changes()` が WAL 変更を取得しつつ RLS を適用
- サブスクリプション作成時にユーザーの JWT claims を `realtime.subscription` テーブルに保存
- `list_changes()` が各サブスクリプションの claims に基づいてフィルタリングし、subscription_id の配列を返す
- サーバー側では subscription_id に基づいてメッセージをルーティング

**方式2: 新 Broadcast 認可 (`realtime.messages` テーブル)**
- `realtime.messages` テーブルに対する RLS ポリシーで認可
- チャネル join 時にロールバックトランザクションで権限チェック
- `realtime.topic()` 関数と JWT claims でポリシー評価

#### Golang での問題点

| 問題 | 詳細 | 深刻度 |
|------|------|--------|
| **RLS フィルタリングの PostgreSQL 依存** | `realtime.list_changes()` は DB 側で RLS を適用しており、この仕組みは Go でも同様に利用可能。ただし、ストアドプロシージャの呼び出しとトランザクション管理が必要 | 低 |
| **JWT claims の設定** | `set_config('request.jwt.claims', ...)` を各クエリ前に実行する必要がある | 低 |
| **サブスクリプション管理** | ユーザーの subscribe/unsubscribe に伴う `realtime.subscription` テーブルの CRUD と、メモリ内のサブスクリプション→ノードマッピングの管理 | 中 |

> RLS 自体は PostgreSQL 側で処理されるため、Go でも同じアプローチが使える。ここは比較的移植しやすい。

---

### 2.6 Presence (CRDT ベースの状態同期)

#### 現行実装

- Phoenix.Presence を使用 (CRDT ベースの分散状態管理)
- Erlang のプロセスグループ (PG2) でノード間のステート同期
- `presence_state` (フルステート) と `presence_diff` (差分) イベント
- プロセスの生死でプレゼンスが自動更新される (Erlang プロセスモニタ)

#### Golang での問題点

| 問題 | 詳細 | 深刻度 |
|------|------|--------|
| **CRDT の再実装** | Phoenix.Presence の CRDT ロジック (フェニックストラッカー) を Go で再実装する必要がある | 高 |
| **分散ステート同期** | ノード間でプレゼンスステートを同期する仕組み。Erlang のプロセスグループに相当する機能がない | 高 |
| **接続状態の追跡** | WebSocket 接続の切断検知とプレゼンスステートの自動クリーンアップ | 中 |
| **ハートビートとの連携** | プレゼンスのタイムアウトと WebSocket ハートビートの統合 | 低 |

---

### 2.7 パフォーマンスとスケーラビリティ

| 観点 | Elixir (現行) | Go (想定) | 注意点 |
|------|--------------|-----------|--------|
| **同時接続数** | BEAM VM で数十万接続可能 | goroutine で同等以上も可能 | Go の方が有利な場合が多い |
| **レイテンシ** | GC は per-process で影響小 | Go の GC は stop-the-world あり | 大量のオブジェクトを扱う場合に注意 |
| **CPU 効率** | BEAM は CPU バウンドに弱い | Go は CPU 効率が高い | JSON シリアライズ等で Go が有利 |
| **メモリ効率** | Erlang プロセスは ~300 bytes/プロセス | goroutine は ~2-8 KB | 大量接続時は Go の方がメモリを使う |
| **バックプレッシャー** | メールボックスによる自然なバックプレッシャー | channel のバッファサイズで制御 | 設計が必要 |

---

## 3. 問題の深刻度ランキング

### 最高 (プロジェクトの成否に関わる)

1. **Phoenix Channel プロトコル互換性** — 既存のクライアント SDK (`@supabase/realtime-js`, `supabase-dart`, `supabase-swift` 等) との互換性を維持できないとエコシステム全体の書き直しが必要
2. **BEAM の障害隔離の喪失** — マルチテナント環境で1テナントの問題が全体に波及するリスク

### 高 (大きな設計・実装コスト)

3. **マルチノードクラスタリング** — Erlang 分散プロトコルの代替 (gRPC, NATS, Redis 等)
4. **Presence の CRDT 再実装** — 分散状態同期の正確な実装
5. **レプリケーション接続のライフサイクル管理** — テナントごとの接続管理、再接続、エラーリカバリ
6. **動的テナント管理** — Supervisor ツリーに相当するリソース管理

### 中 (実装は可能だが工数が必要)

7. **WAL バイナリデコード** — `pglogrepl` を活用すれば対応可能
8. **Fastlane 最適化** — メッセージシリアライズの最適化
9. **テナントリバランシング** — ノード間のテナント移動
10. **バイナリ WebSocket フレーム** — v2.0.0 プロトコルの実装

### 低 (比較的容易)

11. **RLS フィルタリング** — PostgreSQL 側の仕組みをそのまま利用可能
12. **JWT 認証** — Go のライブラリで容易に対応
13. **レートリミット** — `golang.org/x/time/rate` 等で実装可能

---

## 4. 推奨アプローチ

### 段階的移行

全面書き直しではなく、段階的に移行することを推奨する:

**Phase 1: Postgres Changes のみ (最小スコープ)**
- PostgreSQL 論理レプリケーション → Go サーバー → WebSocket クライアント
- Phoenix Channel プロトコル v1.0.0 互換の WebSocket サーバー
- 単一ノード、単一テナント構成から開始

**Phase 2: マルチテナント対応**
- テナントごとの接続プール管理
- 動的テナント追加・削除
- RLS フィルタリング統合

**Phase 3: クラスタリング**
- ノード間メッセージルーティング (gRPC or NATS)
- サービスディスカバリ (etcd / Consul)
- テナントリバランシング

**Phase 4: Broadcast + Presence**
- Broadcast のノード間転送
- Presence の CRDT 実装
- 分散状態同期

### 主要な Go ライブラリ候補

| 用途 | ライブラリ |
|------|-----------|
| PostgreSQL ドライバ | `jackc/pgx/v5` |
| 論理レプリケーション | `jackc/pglogrepl` |
| WebSocket | `coder/websocket` or `gorilla/websocket` |
| JWT 認証 | `golang-jwt/jwt/v5` |
| クラスタリング | `hashicorp/memberlist` |
| サービスディスカバリ | `etcd` or `Consul` |
| Pub/Sub (ノード間) | `NATS` or `Redis Pub/Sub` |
| 接続プール | `jackc/pgx/v5/pgxpool` |
| レートリミット | `golang.org/x/time/rate` |
| メトリクス | `prometheus/client_golang` |

---

## 5. まとめ

Golang で書き直す最大のリスクは以下の3点:

1. **Phoenix Channel プロトコルの完全互換性** — クライアントエコシステムを壊さずに移行できるかが最大の課題
2. **BEAM VM が提供する障害隔離・分散処理の再実装** — OTP の Supervisor、プロセスレジストリ、ノード間通信の代替は大きな設計・実装コスト
3. **Presence の CRDT** — 正確な分散状態同期の実装は理論的にも実装的にも難しい

一方で、Go の方が有利な点もある:
- CPU バウンドな処理 (JSON シリアライズ、暗号処理) のパフォーマンス
- デプロイの簡易さ（シングルバイナリ）
- より広い開発者プール
- 既存の Go ベースのインフラツール（Kubernetes, Prometheus 等）との親和性

**RLS フィルタリングは PostgreSQL 側で処理されるため、Go でも同じアプローチをそのまま利用できる** という点は大きな利点。データベースレベルのセキュリティは言語に依存しない。
