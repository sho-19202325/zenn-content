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

## マテリアライズドビューとは

マテリアライズドビューとは、データベースにおけるクエリ結果を保存し、テーブルであるかのように扱えるビューのことです。
例えば、ユーザー情報を管理するusersテーブルに対して、
```SQL
SELECT * FROM users WHERE created_at >= CURRENT_DATE - INTERVAL '1 month';
```
というSQLを実行すると、直近一ヶ月に作成されたユーザーを取得することができると思います。
この時の結果のスナップショット、つまり、```直近一ヶ月に作成されたユーザーレコードの集合```をテーブルと似たルールで使用できるようになるのがマテリアライズドビューです。

この機能はPostgreSQLではv9.3から使用できるようになっています。


## 通常のビューとの違い

通常、ビューはクエリの内容のみを保存します。
つまり、先ほどの例における
```SQL
SELECT * FROM users WHERE created_at >= CURRENT_DATE - INTERVAL '1 month';
```
という部分のみを保存し、viewが使用される際には、保存したクエリを実行して、その結果を取り出します。

一方、マテリアライズドビューはクエリの実行結果そのものを保存します。<br/>
したがって、作成したマテリアライズドビューを使用する際は上記のクエリが実行されず、
保存した結果をそのまま参照することができます。

## マテリアライズドビューが有効なケース

マテリアライズドビューは実行のたびに元のクエリを発行しないため、

- **複雑なクエリの結果を高速に取得したい場合**
- **大量のデータを集計する場合**

の場合に効果を発揮します。

## マテリアライズドビューの注意点

ここまでみてきた通り、マテリアライズドビューは非常に強力な機能ですが、
使用する上で下記三点に注意する必要があります。

- **更新の反映**
- **ストレージの圧迫**
- **indexの設計**

### 更新の反映

マテリアライズドビューはクエリ実行時点での結果セットをそのまま保存します。
この結果セットは元テーブルのデータとは独立しているため、もし元のテーブル(上記だとusers)に新たなレコードが追加されたとしても、
マテリアライズドビューが保存しているのは古い結果のスナップショットとなっているため新たなレコードの情報は反映されません。

そのため、元テーブルのデータに対する追加/更新をビューに反映するためには、マテリアライズドビューの更新(リフレッシュ)を行う必要があります。

### ストレージの圧迫

前述の通りマテリアライズドビューは通常のビューと違って結果そのものを保存しています。
そのため、マテリアライズドビューの作成に使用するクエリの結果サイズが大きくなる場合、
その分ストレージを圧迫することになることを踏まえた上でビューの設計を行う必要があります。

たとえ今現在ストレージに余裕がある場合でも
- 参照元データはどの頻度で増加するか
- 参照元データの増加に対して、マテリアライズドビューの結果行数がどれだけ増加するか
という点は考慮しておく必要があります。

### indexの設計

マテリアライズドビューにはテーブル同様indexを貼ることができます。
ビューにindexを貼ったとしても、元データの参照/書き込み性能に影響を与えません。
この点は、集計データに対するパフォーマンスを改善する上で非常に重要なポイントになっています。

しかし、逆を言えば、元データのカラムにindexを貼っており、かつ同じカラムがビューの結果として保存されていたとしても、
マテリアライズドビューのカラムにindexを貼っていなければ、当然indexは効かないことになります。

このようにマテリアライズドビューのindexは元のテーブルのindexとは切り離して考える必要があるため、
新たにテーブルを作成する際と同様に十分に注意してindexの設計を行う必要があります。

## 実際に使ってみる

マテリアライズドビューは
```SQL
CREATE MATERIALIZED VIEW [ビューの名前] AS [結果を保存したいクエリの内容];
```
で作成できます。

デモをするにあたって、少し具体的なケースを考えてみましょう。
運営しているウェブサイトの訪問者の情報を格納する下記のようなテーブルがあるとします。

```
CREATE TABLE access_log (
    id SERIAL PRIMARY KEY,          -- ユニークな識別子
    user_id INT NOT NULL,           -- ユーザーID
    page_url VARCHAR(255) NOT NULL, -- アクセスされたページのURL
    created_at TIMESTAMP NOT NULL, -- アクセス時間
);
```

各ページにおける一時間ごとのユニークな訪問者数と閲覧回数を取得したいので、
下記のようなクエリを実行するとします。


```SQL
SELECT
  date_trunc('hour', created_at) AS aggregated_hour,
  page_url,
  COUNT(*) AS page_views,
  COUNT(DISTINCT user_id) AS unique_visitors
FROM
  access_logs
GROUP BY
  aggregated_hour, page_url
ORDER BY
  aggregated_hour DESC, page_url;
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
---------------------------------------------------------------------------------------------------------------------------------------------------------
 GroupAggregate  (cost=5620382.74..12445271.27 rows=8906682 width=51) (actual time=203543.971..857387.584 rows=86271 loops=1)
   Group Key: (date_trunc('hour'::text, created_at)), page_url
   ->  Gather Merge  (cost=5620382.74..11803081.26 rows=53085648 width=43) (actual time=203536.705..611137.509 rows=53085600 loops=1)
         Workers Planned: 2
         Workers Launched: 2
         ->  Sort  (cost=5619382.72..5674680.27 rows=22119020 width=43) (actual time=203186.548..285951.465 rows=17695200 loops=3)
               Sort Key: (date_trunc('hour'::text, created_at)) DESC, page_url
               Sort Method: external merge  Disk: 1004752kB
               Worker 0:  Sort Method: external merge  Disk: 1005184kB
               Worker 1:  Sort Method: external merge  Disk: 1002888kB
               ->  Parallel Seq Scan on access_logs  (cost=0.00..879733.75 rows=22119020 width=43) (actual time=54.043..83816.524 rows=17695200 loops=3)
 Planning Time: 0.157 ms
 JIT:
   Functions: 9
   Options: Inlining true, Optimization true, Expressions true, Deforming true
   Timing: Generation 1.350 ms, Inlining 90.548 ms, Optimization 40.142 ms, Emission 29.591 ms, Total 161.632 ms
 Execution Time: 857985.803 ms
(17 rows)
```
その結果、858s前後とかなり時間のかかるクエリとなっています。
Postgresには関数ベースのindexがサポートされているので、date_trunc('hour', created_at)等に対してindexを貼ればパフォーマンス向上は見込めます。
しかし、書き込みの遅延を懸念する場合、むやみにindexを貼るのは避けたいこともあります。

そこで、このクエリに対するマテリアライズドビューを作成してみます。

```SQL
CREATE MATERIALIZED VIEW hourly_access_logs_views
AS SELECT
  date_trunc('hour', created_at) AS aggregated_hour,
  page_url,
  COUNT(*) AS page_views,
  COUNT(DISTINCT user_id) AS unique_visitors
FROM
  access_logs
GROUP BY
  aggregated_hour, page_url
ORDER BY
  aggregated_hour, page_url;
```

マテリアライズドビューが作成できたら、早速SELECT文で作成してみましょう。
```SQL
SELECT * FROM hourly_access_logs_views
```
すると、先ほどまで時間がかかっていたクエリの結果が比較的速く取得できます。
参考までにこれをEXPLAIN ANALYZEした結果が下記です。

```SQL
                                                           QUERY PLAN                                                           
--------------------------------------------------------------------------------------------------------------------------------
 Seq Scan on hourly_access_logs_views  (cost=0.00..1752.71 rows=86271 width=51) (actual time=0.007..360.822 rows=86271 loops=1)
 Planning Time: 0.044 ms
 Execution Time: 712.696 ms
(3 rows)
```

実行時間は700ms前後になりました。
それもそのはず、1時間ごとにデータが丸められているため、5300万件のレコードは8万6000件程度にまとまっています。
それをSELECTしているだけなので、元のSQLより早く、結果が返ってきているというわけです。

## マテリアライズドビューに対する絞り込みやソート

ここまで、何度もマテリアライズドビューはテーブル同様に扱えるという話をしました。
わざわざ章立てして説明するまでもないかもしれませんが、当然、絞り込みや、ソートも行えます。

先程作成した例で、2023年1月1日から3月31日までの集計結果を、集計時刻の新しい順に取得したい場合は下記のようにします。

```SQL
SELECT *
FROM hourly_access_logs_views
WHERE aggregated_hour BETWEEN '2023-01-01 00:00:00' AND '2023-03-31 23:59:59'
ORDER BY aggregated_hour DESC;
```

この結果もすぐに返ってきます。
例のごとく参考のため、EXPLAIN ANALYZEの結果を添付します。

```SQL
                                                                              QUERY PLAN                                                                               
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Sort  (cost=3762.99..3817.74 rows=21901 width=51) (actual time=199.923..285.006 rows=21600 loops=1)
   Sort Key: aggregated_hour DESC
   Sort Method: quicksort  Memory: 3623kB
   ->  Seq Scan on hourly_access_logs_views  (cost=0.00..2184.07 rows=21901 width=51) (actual time=2.521..103.460 rows=21600 loops=1)
         Filter: ((aggregated_hour >= '2023-01-01 00:00:00'::timestamp without time zone) AND (aggregated_hour <= '2023-03-31 23:59:59'::timestamp without time zone))
         Rows Removed by Filter: 64671
 Planning Time: 0.150 ms
 Execution Time: 370.340 ms
(8 rows)
```

今度は、直近一ヶ月のデータを取得してみます。
また、今度は集計時刻の昇順を指定します。

```SQL
SELECT *
FROM hourly_access_logs_views
WHERE aggregated_hour >= CURRENT_DATE - INTERVAL '1 month'
ORDER BY aggregated_hour;
```

```SQL
                                                             QUERY PLAN                                                             
------------------------------------------------------------------------------------------------------------------------------------
 Sort  (cost=2792.11..2807.67 rows=6226 width=51) (actual time=77.567..102.173 rows=6250 loops=1)
   Sort Key: aggregated_hour
   Sort Method: quicksort  Memory: 1071kB
   ->  Seq Scan on hourly_access_logs_views  (cost=0.00..2399.74 rows=6226 width=51) (actual time=11.327..46.016 rows=6250 loops=1)
         Filter: (aggregated_hour >= (CURRENT_DATE - '1 mon'::interval))
         Rows Removed by Filter: 80021
 Planning Time: 0.069 ms
 Execution Time: 126.550 ms
(8 rows)
```

このように、作成したビューに対して、絞り込みやソートが自由に行えます。

## マテリアライズドビューのインデックス

先ほどの例ではaggregated_hourへの絞り込みや並び替えをindexなしで行っていました。<br/>
前述の通りマテリアライズドビューにはindexを貼ることができるので、実際に試してみましょう。
下記のSQLを実行してみます。

```SQL
CREATE INDEX index_aggregated_hour ON hourly_access_logs_views (aggregated_hour);
```

先ほどのクエリにEXPLAIN ANALYZEをつけて実行してみましょう。

### 2023年1月1日から3月31日までの集計結果(集計時刻の降順)

```SQL
EXPLAIN ANALYZE SELECT *
FROM hourly_access_logs_views
WHERE aggregated_hour BETWEEN '2023-01-01 00:00:00' AND '2023-03-31 23:59:59'
ORDER BY aggregated_hour DESC;
```

```SQL
                                                                              QUERY PLAN                                                                              
----------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Index Scan Backward using index_aggregated_hour on hourly_access_logs_views  (cost=0.29..771.31 rows=21901 width=51) (actual time=0.040..100.947 rows=21600 loops=1)
   Index Cond: ((aggregated_hour >= '2023-01-01 00:00:00'::timestamp without time zone) AND (aggregated_hour <= '2023-03-31 23:59:59'::timestamp without time zone))
 Planning Time: 0.222 ms
 Execution Time: 194.372 ms
(4 rows)
```

### 直近一ヶ月の集計結果(集計時刻の昇順)

```SQL
EXPLAIN ANALYZE SELECT *
FROM hourly_access_logs_views
WHERE aggregated_hour >= CURRENT_DATE - INTERVAL '1 month'
ORDER BY aggregated_hour;
```

```SQL
                                                                        QUERY PLAN                                                                        
----------------------------------------------------------------------------------------------------------------------------------------------------------
 Index Scan using index_aggregated_hour on hourly_access_logs_views  (cost=0.30..209.25 rows=6226 width=51) (actual time=0.118..34.087 rows=6250 loops=1)
   Index Cond: (aggregated_hour >= (CURRENT_DATE - '1 mon'::interval))
 Planning Time: 0.081 ms
 Execution Time: 65.954 ms
(4 rows)
```

これで、indexによりパフォーマンスが改善されたことが確認できました。

## マテリアライズドビューのリフレッシュ

注意点の部分で説明した通り、マテリアライズドビューに新しいデータを反映させるためにはビューのリフレッシュをする必要があります。
ビューのリフレッシュは下記で行えます。
```SQL
REFRESH MATERIALIZED VIEW [ビューの名前]
```

実際に試してみましょう。
結果がわかりやすいように下記のSQLを実行します。

```SQL
SELECT *
FROM hourly_access_logs_views
ORDER BY aggregated_hour DESC
LIMIT 10;
```

この実行結果は下記のようになります。

```SQL
   aggregated_hour   |          page_url           | page_views | unique_visitors 
---------------------+-----------------------------+------------+-----------------
 2023-07-13 00:00:00 | https://example.com/page_1  |        640 |             101
 2023-07-13 00:00:00 | https://example.com/page_10 |        595 |             101
 2023-07-13 00:00:00 | https://example.com/page_2  |        652 |              99
 2023-07-13 00:00:00 | https://example.com/page_3  |        636 |             101
 2023-07-13 00:00:00 | https://example.com/page_4  |        616 |             101
 2023-07-13 00:00:00 | https://example.com/page_5  |        599 |             101
 2023-07-13 00:00:00 | https://example.com/page_6  |        628 |              99
 2023-07-13 00:00:00 | https://example.com/page_7  |        626 |             101
 2023-07-13 00:00:00 | https://example.com/page_8  |        647 |             101
 2023-07-13 00:00:00 | https://example.com/page_9  |        636 |             101
(10 rows)
```

マテリアライズドビューの作成時点では、日付が```2023-07-13 00:00:00```のものが最新となっています。
ここで、新たなaccess_logsを作成します。

```SQL
INSERT INTO access_logs (user_id, page_url, created_at, updated_at)
VALUES (1, 'https://example.com/page_1', '2023-07-14 00:00:00', '2023-07-14 00:00:00');
```

これで```2023-07-14 00:00:00```のaccess_logsデータが作成されました。
先ほどクエリを再度実行すると、結果が変わらず、ビューが古い結果を参照し続けていることがわかります。

ここでビューのリフレッシュを実行します。
```SQL
REFRESH MATERIALIZED VIEW hourly_access_logs_views;
```
リフレッシュが完了したら、先ほどのクエリを再度実行してみましょう。

```SQL
   aggregated_hour   |          page_url           | page_views | unique_visitors 
---------------------+-----------------------------+------------+-----------------
 2023-07-14 00:00:00 | https://example.com/page_1  |          1 |               1
 2023-07-13 00:00:00 | https://example.com/page_1  |        640 |             101
 2023-07-13 00:00:00 | https://example.com/page_10 |        595 |             101
 2023-07-13 00:00:00 | https://example.com/page_2  |        652 |              99
 2023-07-13 00:00:00 | https://example.com/page_3  |        636 |             101
 2023-07-13 00:00:00 | https://example.com/page_4  |        616 |             101
 2023-07-13 00:00:00 | https://example.com/page_5  |        599 |             101
 2023-07-13 00:00:00 | https://example.com/page_6  |        628 |              99
 2023-07-13 00:00:00 | https://example.com/page_7  |        626 |             101
 2023-07-13 00:00:00 | https://example.com/page_8  |        647 |             101
```

これで、最新のデータがビューに反映されました。

## リフレッシュの注意点

マテリアライズドビューにおける更新問題の解決策としてリフレッシュを紹介しましたが、
リフレッシュ自体に関する注意点がいくつかあります。

- 更新頻度
- 更新方法
- 参照ロック

### 更新頻度

リフレッシュを使用する上で、ビューの更新頻度の問題を考える必要があります。
理想は **「元のテーブルのデータが更新されるたびに更新する」** だと思います。
元テーブルの更新頻度がそこまで高くない場合はそれでもよいのですが、
access_logsのような頻繁に作成されるテーブルのデータに対して、毎度更新を実行するのは得策ではありません。

というのも、PostgreSQLにおけるマテリアライズドビューのリフレッシュは、現状のビューを一度削除し、再作成を行います。
その為、access_logsが追加される度にビューの再作成を行なっていて、DBに余計な負荷がかかり過ぎてしまいます。

これを避けるためには、リフレッシュの頻度を適切に定めておく必要があります。
例えば、「毎日0時に実行」、や「3時間ごとに実行」、などです。
これらは元データの更新頻度と要件に合わせて設計していく必要があります。

:::message
再作成ではなく、差分だけを検知してビューの更新を行うための[pg_ivm](https://github.com/sraoss/pg_ivm)というプラグインが存在するようです。
このプラグインの開発者の方が、PostgreSQLの開発コミュニティに機能提案をされているそうで、
今後PostgreSQL単体で差分だけを検知して、リフレッシュする機能が追加されることになるかもしれません。
詳しく知りたい方は下記をご参照ください。
- [PostgreSQLのマテリアライズドビューの自動＆高速リフレッシュ機能を開発中](https://qiita.com/yugo-n/items/98f0e5e25a5717a1212f)
- [PostgreSQL のマテリアライズドビューを高速に最新化する](https://www.sraoss.co.jp/tech-blog/pgsql/postgresql_ivm/)
:::

### 更新方法

先に見た通り、ビューのリフレッシュのためには```REFRESH MATERIALIZED VIEW```コマンドの実行が必要です。
ただし、これを毎回手動で実行するというのはあまり現実的ではありません。<br/>
マテリアライズドビューの作成時に、更新頻度の設定ができれば良いのですが、残念ながら現時点でそのような機能は用意されていないようです。
:::messagemessage alert
※執筆時点での最新版はv15
:::

そのため、自動リフレッシュを実現するには、[pg_cron](https://github.com/citusdata/pg_cron)や[イベントトリガ](https://www.postgresql.jp/document/14/html/event-trigger-definition.html)などを用いて、
特定のタイミングで```REFRESH MATERIALIZED VIEW```を実行するようにしてやる必要があります。

### 参照ロック

先ほどのような```REFRESH MATERIALIZED VIEWS```コマンドの実行中は対象のビューはロックされます。
つまり、更新が終わるまで参照ができなくなる、ということです。<br/>
これを避けるためにはリフレッシュ時に[CONCURENTLY](https://www.postgresql.jp/docs/9.5/sql-refreshmaterializedview.html#:~:text=%E3%83%91%E3%83%A9%E3%83%A1%E3%83%BC%E3%82%BF-,CONCURRENTLY,-%E3%81%9D%E3%81%AE%E3%83%9E%E3%83%86%E3%83%AA%E3%82%A2)オプションを使用する必要があります。

```SQL
REFRESH MATERIALIZED VIEW CONCURRENTLY hourly_access_logs_views;
```

ただし、このオプションを使用するためには、
- マテリアライズドビューにユニークなindexが貼られている
- 上記のindexはビューの全ての行を含み、列名を使用している。(indexに式やWHERE句を使用していない)
という条件があります。

今回の例だと下記のようなindexを貼れば使用できるようになります。
```SQL
CREATE UNIQUE INDEX index_aggregated_hour_with_page_url ON hourly_access_logs_views (aggreg
ated_hour, page_url);
```

また、このCONCURRENTLYオプションを使用すると、
内部的には変更前のビューと変更中のビューが2つ作成され、
ビューのリフレッシュが終わるまで変更前のビューを参照するようになります。<br/>
そのため、リフレッシュが完了するまでビュー2つ分のストレージが使用されるので注意してください。
