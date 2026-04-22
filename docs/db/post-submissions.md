# 投稿データ DB 設計

## 目的

投稿機能 要件定義にある最小保存要件を、そのまま実装可能な DB 仕様に落とす。
初期リリースでは投稿受付と審査待ち管理に必要な項目だけを持つ。

## 対象

- 投稿フォームから受け付けた 1 件の投稿
- 審査待ちから審査結果反映までの状態管理

## テーブル案

テーブル名は `post_submissions` とする。

| カラム名 | 型 | 必須 | デフォルト | 説明 |
| --- | --- | --- | --- | --- |
| id | uuid | 必須 | DB 生成 | 投稿 ID |
| radio_name | varchar(30) | 必須 | なし | 投稿時のラジオネーム |
| content | varchar(300) | 必須 | なし | 投稿本文 |
| status | varchar(20) | 必須 | `pending` | 審査状態 |
| moderation_note | text | 任意 | `NULL` | 却下理由や審査メモ |
| created_at | timestamptz | 必須 | `CURRENT_TIMESTAMP` | 投稿日時 |
| updated_at | timestamptz | 必須 | `CURRENT_TIMESTAMP` | 更新日時 |

## 制約

- 主キーは `id`
- `radio_name` は 1-30 文字を許可する
- `content` は 1-300 文字を許可する
- `status` は `pending` / `approved` / `rejected` のみ許可する
- `moderation_note` は初期リリースでは任意。審査結果が `rejected` のときのみ運用上入力可能とする

SQL の制約イメージ:

```sql
create table post_submissions (
  id uuid primary key,
  radio_name varchar(30) not null,
  content varchar(300) not null,
  status varchar(20) not null default 'pending',
  moderation_note text null,
  created_at timestamptz not null default current_timestamp,
  updated_at timestamptz not null default current_timestamp,
  constraint chk_post_submissions_radio_name_length
    check (char_length(radio_name) between 1 and 30),
  constraint chk_post_submissions_content_length
    check (char_length(content) between 1 and 300),
  constraint chk_post_submissions_status
    check (status in ('pending', 'approved', 'rejected'))
);
```

## インデックス

初期リリースでは以下の 2 つに限定する。

| 対象 | 目的 |
| --- | --- |
| primary key (`id`) | 1 件取得 |
| index on (`status`, `created_at`) | 審査待ち一覧や採用候補の時系列確認 |

## 更新ルール

- 投稿作成時
  - `status` は必ず `pending`
  - `moderation_note` は `NULL`
- 審査通過時
  - `status` を `approved` に更新
  - `updated_at` を更新
- 審査却下時
  - `status` を `rejected` に更新
  - 必要に応じて `moderation_note` を保存
  - `updated_at` を更新

## 実装メモ

- `radio_name` と `content` の trim はアプリケーション層で行う
- `updated_at` の自動更新は ORM または更新処理側で担保する
- 連投制限や投稿者識別子は未確定のため、このテーブルには含めない

## 未確定事項

- DB に PostgreSQL を使うか SQLite を使うか
- `id` を UUID にするか ULID にするか
- `moderation_note` を初期リリースから管理画面で編集可能にするか
- 本文検索や NG ワード判定用の補助カラムが将来必要か

## 次の最小作業

この DB 設計に対応する投稿フォーム画面設計を追加し、入力項目・エラー表示・完了状態を 1 画面分だけ具体化する。
