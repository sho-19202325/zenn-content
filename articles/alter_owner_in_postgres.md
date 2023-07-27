---
title: 'superuser以外のユーザーを使用してデータベース/テーブルのOWNERを変更する方法'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: [
  'PostgreSQL',
  'SQL'
]
emoji: "👌"
published: true
---

## はじめに

マネージドのPostgreSQLサーバーを使用している時に、superadmin以外のユーザーを使ってデータベース/テーブルのOWNERを変更する必要がありました。

何となく、データベースやテーブルのOWNERであれば
```SQL
ALTER DATABASE db_name OWNER TO new_user
```
を実行できると思っていたのですが、
OWNERであるだけだと、上記SQLは実行することができませんでした。
その理由を調べてみたので今回記事として残しておきます。

## 前提

- CREATEDB権限を持ったsuperuserでないユーザー(test_user)が存在する
- データベース(test_db), テーブル(test_table)のOWNERはtest_user
- このオーナーを別のユーザー(another_user)に変更したい

## 現象

OWNERであるtest_userを使って下記を実行すると、
another_userのメンバーである必要があるとエラーメッセージが表示されます。

```SQL
ALTER DATABASE test_db OWNER TO another_user
ALTER TABLE test_table OWNER TO another_user
-- => ERROR:  must be member of role another_user
```

## 解決策

エラー文の通りtest_userをanother_userのメンバーに加えることで、OWNERの変更行えます。

1. another_userでログインして下記を実行
    ```SQL
    GRANT another_user TO test_user;
    ```
1. test_userでログインして下記を実行
   ```SQL
   --データベース(test_db)のownerをanother_userに変更する場合
    ALTER DATABASE test_db OWNER TO another_user;
   --テーブル(test_table)のownerをanother_userに変更する場合
    ALTER TABLE test_table OWNER TO another_user;
   ```

## 解説

公式ドキュメント(日本語版)には以下のような記述があります。

### ALTER DATABASE
>3番目の構文は、データベースの所有者を変更します。 所有者を変更するにはデータベースを所有し、かつ、新しい所有者ロールの間接的あるいは直接的なメンバでなければなりません。
>さらに、CREATEDB権限も持たなければなりません。 （スーパーユーザはこれらの権限を自動的に持っていることに注意してください。）

※ 3番目の構文とは、```ALTER DATABASE name OWNER TO new_owner```のこと

参照元: [PostgreSQL 14.5文書 ALTER DATABASE](https://www.postgresql.jp/document/14/html/sql-alterdatabase.html)

### ALTER TABLE

>所有者を変更するには、新しい所有ロールの直接あるいは間接的なメンバでなければならず、かつ、そのロールがテーブルのスキーマにおけるCREATE権限を持たなければなりません 
>（この制限により、テーブルの削除と再作成を行ってもできないことが、所有者の変更によってもできないようにしています。 ただし、スーパーユーザはすべてのテーブルの所有者を変更することができます）。

参照元: [PostgreSQL 14.5文書 ALTER TABLE](https://www.postgresql.jp/document/14/html/sql-altertable.html)

これはつまり、OWNERの変更を行うためには
下記の1.または2の条件を満たす必要があるということです。
1. **superuserであること**
2. **データベース/テーブルのOWNERで、かつ次の条件を満たすこと**
    - **CREATEDB権限を持っている**
    - **新しいOWNERのメンバーである**

今回、test_userはOWNERであり、CREATEDB権限も持っていましたが、
another_userのメンバーではなかったために、
```ALTER DATABASE/TABLE ... OWNER TO ...```の実行権限がなかったというわけです。

## 注意点

test_userをanother_userのメンバーにしたまま放置していると、
test_userが```SET ROLE```などを用いて、another_userが持つ権限を使用可能な状態になります。
作業が完了したら下記のSQLを実行して、test_userをanother_userのメンバーから除外することを忘れないようにしましょう。

```SQL
REVOKE another_user from test_user;
```