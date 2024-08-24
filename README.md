# pgroll 調査メモ

## Initialization
```shell
$ pgroll init --postgres-url "postgresql://postgres:password@localhost:5440/postgres?sslmode=disable"
 SUCCESS  Initialization complete  
```

環境変数も使える
```shell
$ export PGROLL_PG_URL="postgresql://postgres:password@localhost:5440/postgres?sslmode=disable"
```

## Migrations
- 最初のマイグレーションを実行
```shell
$ pgroll start sql/01_create_users_table.json --complete 
 SUCCESS  New version of the schema available under the postgres "public_01_create_users_table" schema  
```

破壊的なマイグレーションを加えてみる
```json
{
  "name": "02_user_description_set_nullable",
  "operations": [
    {
      "alter_column": {
        "table": "users",
        "column": "description",
        "nullable": false,
        "up": "(SELECT CASE WHEN description IS NULL THEN 'description for ' || name ELSE description END)",
        "down": "description"
      }
    }
  ]
}
```

2回目は `--complete` フラグをつけないで実行
```shell
$ pgroll start 02_user_description_set_nullable.json
 SUCCESS  New version of the schema available under the postgres "public_02_user_description_set_nullable" schema  
```

users テーブルを見てみると、`_pgroll_new_description` というカラムが追加されている。
```
postgres@127:postgres> SELECT * FROM public.users ORDER BY id LIMIT 10;
+----+---------+-------------------------+-------------------------+
| id | name    | description             | _pgroll_new_description |
|----+---------+-------------------------+-------------------------|
| 1  | user_1  | description for user_1  | description for user_1  |
| 2  | user_2  | <null>                  | description for user_2  |
| 3  | user_3  | <null>                  | description for user_3  |
| 4  | user_4  | <null>                  | description for user_4  |
| 5  | user_5  | <null>                  | description for user_5  |
| 6  | user_6  | description for user_6  | description for user_6  |
| 7  | user_7  | description for user_7  | description for user_7  |
| 8  | user_8  | description for user_8  | description for user_8  |
| 9  | user_9  | <null>                  | description for user_9  |
| 10 | user_10 | description for user_10 | description for user_10 |
+----+---------+-------------------------+-------------------------+
```

現在のスキーマ一覧も確認する。
```
postgres@127:postgres> \dn;
+-----------------------------------------+-------------------+
| Name                                    | Owner             |
|-----------------------------------------+-------------------|
| pgroll                                  | postgres          |
| public                                  | pg_database_owner |
| public_01_create_users_table            | postgres          |
| public_02_user_description_set_nullable | postgres          |
+-----------------------------------------+-------------------+
```

`public.users` テーブルの構造を確認する。
```
postgres@127:postgres> DESCRIBE users;
+-------------------------+------------------------+-----------------------------------------------------------------+
| Column                  | Type                   | Modifiers                                                       |
|-------------------------+------------------------+-----------------------------------------------------------------|
| id                      | integer                |  not null default nextval('_pgroll_new_users_id_seq'::regclass) |
| name                    | character varying(255) |  not null                                                       |
| description             | text                   |                                                                 |
| _pgroll_new_description | text                   |                                                                 |
+-------------------------+------------------------+-----------------------------------------------------------------+
Indexes:
    "_pgroll_new_users_pkey" PRIMARY KEY, btree (id)
    "_pgroll_new_users_name_key" UNIQUE CONSTRAINT, btree (name)
Check constraints:
    "_pgroll_check_not_null_description" CHECK (_pgroll_new_description IS NOT NULL) NOT VALID
Triggers:
    _pgroll_trigger_users__pgroll_new_description BEFORE INSERT OR UPDATE ON users FOR EACH ROW EXECUTE FUNCTION _pgroll_trigger_users__pg>
    _pgroll_trigger_users_description BEFORE INSERT OR UPDATE ON users FOR EACH ROW EXECUTE FUNCTION _pgroll_trigger_users_description()
```

```sql
postgres@127:postgres> \sf+ _pgroll_trigger_users_description
+----------------------------------------------------------------------------------------------------------------------------------------->
| source                                                                                                                                  >
|----------------------------------------------------------------------------------------------------------------------------------------->
|         CREATE OR REPLACE FUNCTION public._pgroll_trigger_users_description()                                                           >
|          RETURNS trigger                                                                                                                >
|          LANGUAGE plpgsql                                                                                                               >
| 1       AS $function$                                                                                                                   >
| 2           DECLARE                                                                                                                     >
| 3             "description" "public"."users"."description"%TYPE := NEW."description";                                                   >
| 4             "id" "public"."users"."id"%TYPE := NEW."id";                                                                              >
| 5             "name" "public"."users"."name"%TYPE := NEW."name";                                                                        >
| 6             latest_schema text;                                                                                                       >
| 7             search_path text;                                                                                                         >
| 8           BEGIN                                                                                                                       >
| 9             SELECT 'public' || '_' || latest_version                                                                                  >
| 10              INTO latest_schema                                                                                                      >
| 11              FROM "pgroll".latest_version('public');                                                                                 >
| 12                                                                                                                                      >
| 13            SELECT current_setting                                                                                                    >
| 14              INTO search_path                                                                                                        >
| 15              FROM current_setting('search_path');                                                                                    >
| 16                                                                                                                                      >
| 17            IF search_path != latest_schema THEN                                                                                      >
| 18              NEW."_pgroll_new_description" = (SELECT CASE WHEN description IS NULL THEN 'description for ' || name ELSE description E>
| 19            END IF;                                                                                                                   >
| 20                                                                                                                                      >
| 21            RETURN NEW;                                                                                                               >
| 22          END; $function$                                                                                                             >
+----------------------------------------------------------------------------------------------------------------------------------------->
Time: 0.012s
```

古いスキーマの構造を確認する。
```
postgres@127:postgres> \d+ public_01_create_users_table.users;
+-------------+------------------------+-----------+----------+-------------+
| Column      | Type                   | Modifiers | Storage  | Description |
|-------------+------------------------+-----------+----------+-------------|
| id          | integer                |           | plain    | <null>      |
| name        | character varying(255) |           | extended | <null>      |
| description | text                   |           | extended | <null>      |
+-------------+------------------------+-----------+----------+-------------+
View definition:
 SELECT id,
    name,
    description
   FROM users; 
```

2つ目のマイグレーションのスキーマを確認。
```
postgres@127:postgres> \d+ public_02_user_description_set_nullable.users
+-------------+------------------------+-----------+----------+-------------+
| Column      | Type                   | Modifiers | Storage  | Description |
|-------------+------------------------+-----------+----------+-------------|
| id          | integer                |           | plain    | <null>      |
| name        | character varying(255) |           | extended | <null>      |
| description | text                   |           | extended | <null>      |
+-------------+------------------------+-----------+----------+-------------+
View definition:
 SELECT id,
    name,
    _pgroll_new_description AS description
   FROM users; 
```

## Completing the migration
```shell
$ pgroll complete
 SUCCESS  Migration successful!
```

古いマイグレーションで作られたスキーマが削除されている
```sql
postgres@127:postgres> \dn;
+-----------------------------------------+-------------------+
| Name                                    | Owner             |
|-----------------------------------------+-------------------|
| pgroll                                  | postgres          |
| public                                  | pg_database_owner |
| public_02_user_description_set_nullable | postgres          |
+-----------------------------------------+-------------------+
```

## その他
- 現在のマイグレーション状況も一応取得できる
```shell
$ pgroll status                                              
{
  "schema": "public",
  "version": "03_add_gender_to_users_with_raw_sql",
  "status": "In progress"
}
```
- json の `operations` にまとめて突っ込めば、マイグレーションのオペレーションを一度に実行できる
  - 何個もスキーマを作らなくて済みそう（`Raw SQL`があったら無理（後述））

## 疑問
- 進行中のマイグレーションがある状態で、新たなマイグレーションを実行できる？
  - すでに進行中のがあると、実行不可
  - https://github.com/xataio/pgroll/blob/3f792b789df411c6df8259764a8c4edba8d664f3/pkg/roll/execute.go#L36
- `Raw SQL` でマイグレーションを書いた場合はどうなる？
  - `Raw SQL` 単体ならロールバック可能で他のマイグレーションのように扱える
  - ただ他の操作と一緒に実行することはできない（`onComplete`フラグを付与することでそれも可能になる）

## 留意事項
- Postgres 14 において RLS を使用している場合、pgrollは移行ツールの選択肢としては不適切との記述あり
  - 原因は、pgrollが(security_invoker = true)オプションでビューを作成できないため
- マイグレーションするたびにスキーマが作成されるので、アプリケーションは最新のスキーマに向き先を変える必要がありそう。`pgroll` が使うスキーマは使ってはいけないとのこと。
    > 実際のスキーマ（publicなど）をクライアントが直接使用するのは安全ではないので、絶対に避けること。代わりに、クライアントはバージョン管理されたビュー（public_03_add_columnなど）を持つスキーマを使用すること。
