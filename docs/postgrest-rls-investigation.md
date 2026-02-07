# PostgREST RLS Investigation

PostgREST がセッション変数を設定する内部ロジックと、Supabase の `auth.*()` 関数の仕組みを調査した結果をまとめる。
目的: PostgreSQL クライアントから直接接続する際に、API と同じ RLS ロジックを適用する方法を理解する。

---

## 1. PostgREST が各リクエストで実行する SQL

PostgREST は HTTP リクエストを受け取ると、**トランザクション内で以下の手順を実行してから**メインクエリを処理する。

### 実行順序

```
1. BEGIN (トランザクション開始)
2. SET LOCAL ROLE <role>;              -- ロールの切り替え
3. set_config('request.jwt.claims', '<JSON>', true);  -- JWT Claims の設定
4. set_config('request.method', 'GET', true);         -- HTTP メソッド
5. set_config('request.path', '/table', true);        -- リクエストパス
6. set_config('request.headers', '<JSON>', true);     -- リクエストヘッダー
7. set_config('request.cookies', '<JSON>', true);     -- クッキー
8. db-pre-request 関数の呼び出し (設定されている場合)
9. メインクエリの実行
10. COMMIT / ROLLBACK
```

### set_config() の使用理由

PostgREST は `SET LOCAL` ではなく `set_config(key, value, true)` を使用する。
- 第3引数 `true` はトランザクションスコープを意味する（`SET LOCAL` と同等）
- パラメータ化されたクエリとして実行されるため SQL インジェクションを防止
- PR #1600 で `SET LOCAL` から `set_config()` に変更された

### ロールの切り替えロジック

```
JWT の role claim → SET LOCAL ROLE <role>;
JWT なし / 無効 → SET LOCAL ROLE anon;  (PGRST_DB_ANON_ROLE の値)
```

PostgREST は JWT の `role` claim をデフォルトで参照する（`jwt-role-claim-key` で変更可能）。
Supabase では `authenticator` ロールで接続し、リクエストごとに `anon` / `authenticated` / `service_role` に切り替える。

---

## 2. 設定されるセッション変数の一覧

| 変数名 | 値の例 | 説明 |
|--------|--------|------|
| `request.jwt.claims` | `{"sub":"a7194ea3-...","role":"authenticated","exp":1638307414,...}` | JWT ペイロード全体 (JSON) |
| `request.method` | `GET`, `POST`, `PUT`, `PATCH`, `DELETE` | HTTP メソッド |
| `request.path` | `table`, `rpc/function` | リクエストのパス |
| `request.headers` | `{"user-agent":"...","authorization":"Bearer ..."}` | リクエストヘッダー (JSON) |
| `request.cookies` | `{"sessionId":"..."}` | クッキー (JSON) |

### PGRST_DB_USE_LEGACY_GUCS の影響

Supabase の docker-compose では `PGRST_DB_USE_LEGACY_GUCS: "false"` に設定されている。

- **`false` (現行)**: `request.jwt.claims` に JSON オブジェクト全体を格納
- **`true` (レガシー)**: 個別の claim を `request.jwt.claim.<name>` に分解して格納

---

## 3. Supabase の auth.*() 関数

これらの関数は **Supabase の GoTrue / supabase-postgres イメージ内で定義** されている。
PostgREST が設定したセッション変数を `current_setting()` で読み取るラッパー関数。

### auth.uid()

```sql
-- 内部実装 (概念的)
CREATE FUNCTION auth.uid() RETURNS uuid
  LANGUAGE sql STABLE
AS $$
  SELECT
    coalesce(
      nullif(current_setting('request.jwt.claim.sub', true), ''),
      (nullif(current_setting('request.jwt.claims', true), '')::jsonb ->> 'sub')
    )::uuid
$$;
```

- JWT の `sub` claim を UUID として返す
- Legacy GUC (`request.jwt.claim.sub`) と新形式 (`request.jwt.claims` → `sub`) の両方に対応
- 認証されていない場合は `NULL` を返す

### auth.role() [非推奨]

```sql
-- 内部実装 (概念的)
CREATE FUNCTION auth.role() RETURNS text
  LANGUAGE sql STABLE
AS $$
  SELECT
    coalesce(
      nullif(current_setting('request.jwt.claim.role', true), ''),
      (nullif(current_setting('request.jwt.claims', true), '')::jsonb ->> 'role')
    )::text
$$;
```

- JWT の `role` claim (`'authenticated'` / `'anon'`) を返す
- **非推奨**: RLS ポリシーの `TO` 句を使用することが推奨される

```sql
-- 非推奨
CREATE POLICY "..." ON table FOR SELECT USING (auth.role() = 'authenticated');

-- 推奨
CREATE POLICY "..." ON table FOR SELECT TO authenticated USING (true);
```

### auth.jwt()

```sql
-- 内部実装 (概念的)
CREATE FUNCTION auth.jwt() RETURNS jsonb
  LANGUAGE sql STABLE
AS $$
  SELECT
    coalesce(
      nullif(current_setting('request.jwt.claims', true), '')::jsonb,
      '{}'::jsonb
    )
$$;
```

- JWT ペイロード全体を JSONB として返す
- カスタム claim へのアクセスに使用

```sql
-- 使用例
(auth.jwt() ->> 'email') = email
(auth.jwt() ->> 'user_role')::app_role
```

### auth.email() [非推奨]

- JWT の `email` claim を返す
- **非推奨**: `(auth.jwt() ->> 'email')` を使用すること

---

## 4. PostgreSQL クライアントから RLS を適用する方法

PostgreSQL クライアントから直接接続して API と同じ RLS を適用するには、PostgREST が行うのと同じセッション変数の設定を手動で行う。

### 基本パターン

```sql
-- 1. ロールを切り替え
SET LOCAL ROLE authenticated;

-- 2. JWT claims を設定 (auth.uid(), auth.jwt() が動作するために必要)
SELECT set_config('request.jwt.claims', '{
  "sub": "USER_UUID_HERE",
  "role": "authenticated",
  "aud": "authenticated",
  "iat": 1234567890,
  "exp": 9999999999
}', true);

-- 3. (オプション) その他のセッション変数
SELECT set_config('request.method', 'POST', true);
SELECT set_config('request.path', '/rpc/my_function', true);
SELECT set_config('request.headers', '{"accept": "*/*"}', true);

-- 4. RLS が適用された状態でクエリ実行
SELECT * FROM my_table;  -- auth.uid() に基づいてフィルタされる
```

### 簡易パターン (テスト用)

```sql
-- Legacy GUC スタイル (SET LOCAL で直接設定)
SET LOCAL ROLE authenticated;
SET LOCAL request.jwt.claim.sub = 'USER_UUID_HERE';

-- これで auth.uid() が動作する
SELECT * FROM my_table;
```

### Supabase Studio の実装 (参考)

`apps/studio/lib/role-impersonation.ts:95-108` で、Studio の SQL エディタがロール偽装を行う際の実装:

```sql
SELECT set_config('role', 'authenticated', true),
       set_config('request.jwt.claims', '{"sub":"...","role":"authenticated",...}', true),
       set_config('request.method', 'POST', true),
       set_config('request.path', '/impersonation-example-request-path', true),
       set_config('request.headers', '{"accept": "*/*"}', true);
```

### 注意事項

1. **トランザクション内で実行すること**: `set_config(..., true)` はトランザクションスコープ。トランザクション外では持続しない。
2. **`authenticator` ロールで接続**: PostgREST は `authenticator` ロールで接続し、`SET LOCAL ROLE` で切り替える。直接 `anon` / `authenticated` ロールで接続するのとは異なる。
3. **Legacy vs Modern GUC**: `auth.uid()` 等の関数は両方の形式に対応しているため、`request.jwt.claims` (JSON全体) または `request.jwt.claim.sub` (個別) のどちらでも動作する。
4. **exp claim**: `auth.jwt()` を使うポリシーでは `exp` が期限切れだとクエリに影響する場合がある。十分に未来の値を設定すること。

---

## 5. PostgREST の設定 (Supabase docker-compose)

```yaml
# docker/docker-compose.yml (抜粋)
rest:
  image: postgrest/postgrest:v14.3
  environment:
    PGRST_DB_URI: postgres://authenticator:${POSTGRES_PASSWORD}@${POSTGRES_HOST}:${POSTGRES_PORT}/${POSTGRES_DB}
    PGRST_DB_SCHEMAS: ${PGRST_DB_SCHEMAS}
    PGRST_DB_ANON_ROLE: anon
    PGRST_JWT_SECRET: ${JWT_SECRET}
    PGRST_DB_USE_LEGACY_GUCS: "false"
    PGRST_APP_SETTINGS_JWT_SECRET: ${JWT_SECRET}
    PGRST_APP_SETTINGS_JWT_EXP: ${JWT_EXPIRY}
```

```sql
-- docker/volumes/db/jwt.sql
ALTER DATABASE postgres SET "app.settings.jwt_secret" TO '<JWT_SECRET>';
ALTER DATABASE postgres SET "app.settings.jwt_exp" TO '<JWT_EXP>';
```

---

## 6. 関連ファイル

| ファイル | 説明 |
|----------|------|
| `docker/docker-compose.yml:179-200` | PostgREST の設定 |
| `docker/volumes/db/jwt.sql` | JWT シークレットの DB 設定 |
| `apps/studio/lib/role-impersonation.ts` | Studio のロール偽装 SQL 生成 |
| `apps/docs/content/guides/api/securing-your-api.mdx` | API セキュリティガイド |
| `apps/docs/content/guides/local-development/testing/overview.mdx` | テストでの RLS 設定例 |
| `apps/docs/content/troubleshooting/rls-performance-and-best-practices-Z5Jjwv.mdx` | RLS パフォーマンス |
| `apps/docs/content/troubleshooting/deprecated-rls-features-Pm77Zs.mdx` | auth.role() / auth.email() の非推奨情報 |

## 7. 参考リンク

- [PostgREST Authentication Docs](https://docs.postgrest.org/en/v12/references/auth.html)
- [PostgREST Transactions](https://docs.postgrest.org/en/v12/references/transactions.html)
- [PostgREST DB Authorization](https://docs.postgrest.org/en/v12/explanations/db_authz.html)
- [PostgREST PR #1600 - set_config migration](https://github.com/PostgREST/postgrest/pull/1600)
