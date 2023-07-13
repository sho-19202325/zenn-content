---
title: "[PostgreSQL]マテリアライズドビューを利用して複雑なクエリのパフォーマンスを向上させる"
emoji: "🐡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [
  'PostgreSQL',
  'SQL'
]
published: false
---

## 目次

github連携によるZennの執筆テストです。

## マテリアライズドビューとは

マテリアライズドビューとは、データベースにおけるクエリ結果を保存する特殊なビューです。
例えば、ユーザー情報を管理するusersテーブルに対して、
```SQL
SELECT * FROM users WHERE created_at >= CURRENT_DATE - INTERVAL '1 month';
```
というSQLを実行すると、直近一ヶ月に作成されたユーザーを取得することができると思います。
この時の結果のスナップショット、つまり、```直近一ヶ月に作成されたユーザーレコードの集合```をテーブルなどと似たルールで使用できるようになるのがマテリアライズドビューです。

この機能はPostgreSQLではv9.3から使用できるようになっています。


## 通常のビューとの違い

通常、ビューはクエリの内容のみを保存します。
つまり、先ほどの例における
```SQL
SELECT * FROM users WHERE created_at >= CURRENT_DATE - INTERVAL '1 month';
```
という部分のみを保存し、viewが使用される際には、保存したクエリを実行して、その結果を取り出します。

一方、マテリアライズドビューはクエリの実行結果そのものを保存します。
したがって、作成したマテリアライズドビューを使用する際は上記のクエリが実行されず、
保存した結果をそのまま参照することができます。

## マテリアライズドビューが有効なケース

マテリアライズドビューは実行のたびに元のクエリを発行しないため、

- 複雑なクエリの結果を高速に取得したい場合
- 大量のデータを集計する場合

の場合に効果を発揮します。
元のデータが集約したデータが


## マテリアライズドビューの注意点

ビューとの違いでも触れた通りマテリアライズドビューは実行結果そのものを保存します。
つまり、マテリアライズドビューが作成された後、もし元のテーブル(上記だとxxx)に新たなレコードが追加されたとしてもマテリアライズドビューは古い結果を返し続けます。

これを避けるためには、後ほど説明するマテリアライズドビューのリフレッシュをする必要があります。
ただし、リフレッシュを頻繁に行うと、マテリアライズドビューのロックが頻繁に発生したり、DBへの負荷が上がることがあるため、
リフレッシュの頻度やタイミングは慎重に決めておく必要がります。

また、通常のビューと違って結果そのものを保存しているため、結果のレコード数が大きければ大きいほど、ストレージを圧迫することになります。

マテリアライズドビューは非常に強力な機能ではありますが、リフレッシュの方法やクエリの設計は十分に注意して行う必要があります。

## 実際に使ってみる

運営しているウェブサイトの訪問者の情報を取得するケースを考えてみます。
下記のようなテーブルがあるとします。

```
CREATE TABLE access_log (
    id SERIAL PRIMARY KEY,          -- ユニークな識別子
    user_id INT NOT NULL,           -- ユーザーID
    page_url VARCHAR(255) NOT NULL, -- アクセスされたページのURL
    created_at TIMESTAMP NOT NULL, -- アクセス時間
);
```

各ページにおける一時間ごとのユニークな訪問者数と閲覧回数を取得したい場合、
下記のようなクエリが想定されます。


```SQL
SELECT
  date_trunc('hour', created_at) AS hour,
  page_url,
  COUNT(*) AS page_views,
  COUNT(DISTINCT user_id) AS unique_visitors
FROM
  access_logs
GROUP BY
  hour, page_url
ORDER BY
  hour DESC, page_url;
```

date_truncでcreated_atを1時間ごとに丸めたものをGROUP BYに指定しています。
これにより一時間ごとのデータが集計できるようになるわけです。

ただ、ある程度訪問者が多かったり、長期間運営しているサイトであればaccess_logsは膨大な数になるでしょう。
毎回このクエリを発行するのには時間がかかります。

実際に手元に
- 100人のユーザ
- 10つのページ
を用意し、100人のユーザーが、1分ごとにランダムなページに訪れるダミーデータを約一年分(約5300万レコード)を用意し、
上記のクエリに対してEXPLAIN ANALYZEを実行しました。


```SQL
                                                                       QUERY PLAN                                                                       
--------------------------------------------------------------------------------------------------------------------------------------------------------
 GroupAggregate  (cost=5620390.91..12421224.47 rows=6982771 width=51) (actual time=47482.617..67027.503 rows=86271 loops=1)
   Group Key: (date_trunc('hour'::text, created_at)), page_url
   ->  Gather Merge  (cost=5620390.91..11803083.84 rows=53085600 width=43) (actual time=47482.552..59740.859 rows=53085600 loops=1)
         Workers Planned: 2
         Workers Launched: 2
         ->  Sort  (cost=5619390.89..5674688.39 rows=22119000 width=43) (actual time=46245.681..49251.713 rows=17695200 loops=3)
               Sort Key: (date_trunc('hour'::text, created_at)), page_url
               Sort Method: external merge  Disk: 1011128kB
               Worker 0:  Sort Method: external merge  Disk: 993688kB
               Worker 1:  Sort Method: external merge  Disk: 1008024kB
               ->  Parallel Seq Scan on access_logs  (cost=0.00..879744.50 rows=22119000 width=43) (actual time=73.390..8994.579 rows=17695200 loops=3)
 Planning Time: 0.075 ms
 JIT:
   Functions: 9
   Options: Inlining true, Optimization true, Expressions true, Deforming true
   Timing: Generation 1.359 ms, Inlining 150.742 ms, Optimization 40.624 ms, Emission 28.368 ms, Total 221.093 ms
 Execution Time: 67256.473 ms
(17 rows)
```
その結果、67000ms以上とかなり時間のかかるクエリとなっています。
Postgresには関数ベースのindexがサポートされているので、date_trunc('hour', created_at)等に対してindexを貼ればパフォーマンス向上は見込めますが、書き込みの遅延を懸念する場合は、各種データの集計に対してindexを貼るのは避けたいこともあります。

そこで、このクエリに対するマテリアライズドビューを作成してみます。
マテリアライズドビューは```CREATE MATERIALIZED VIEW [viewにつける名前] AS [結果を保存したいクエリの内容];```で作成できます。

```SQL
CREATE MATERIALIZED VIEW hourly_access_logs_views
AS SELECT
  date_trunc('hour', created_at) AS hour,
  page_url,
  COUNT(*) AS page_views,
  COUNT(DISTINCT user_id) AS unique_visitors
FROM
  access_logs
GROUP BY
  hour, page_url
ORDER BY
  hour, page_url;
```

マテリアライズドビューが作成できたら、早速SELECT文で作成してみましょう。
```SQL
SELECT * FROM hourly_access_logs_views
```
すると、先ほどまで時間がかかっていたクエリの結果が一瞬で取得できます。
ちなみにこれをEXPLAIN ANALYZEした結果が下記です。

```SQL
                                                          QUERY PLAN                                                          
------------------------------------------------------------------------------------------------------------------------------
 Seq Scan on hourly_access_logs_views  (cost=0.00..1752.71 rows=86271 width=51) (actual time=0.007..5.893 rows=86271 loops=1)
 Planning Time: 0.053 ms
 Execution Time: 8.615 ms
(3 rows)
```

9ms以下と爆速ですね。
それもそのはず、1時間ごとにデータが丸められているため、5300万件のレコードは9万件以下にまとまっています。
それをSELECTしているだけなので、迅速に結果が返ってきているというわけです。

さらに、このマテリアライズドビューに対して絞り込みも行えます。

TODO: 一ヶ月ごと、一週間ごと、など条件をつけて検索する

### ある1日のaccess_log集計データ

### ある1週間のaccess_log集計データ

### ある一ヶ月のaccess_log集計データ


## マテリアライズドビューのパフォーマンスチューニング

通常のテーブルと同じように、マテリアライズドビューにもindexを貼ることが可能です。


## マテリアライズドビューのリフレッシュ
