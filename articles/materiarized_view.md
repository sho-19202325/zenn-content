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

- 複雑なクエリの結果を高速に取得したい場合
- 大量のデータを集計する場合

の場合に効果を発揮します。

## マテリアライズドビューの注意点

ここまでみてきた通り、マテリアライズドビューは非常に強力な機能ですが、
使用する上で下記二点に注意する必要があります。

- リフレッシュの問題
- ストレージの圧迫
- indexの設計

### リフレッシュの問題

マテリアライズドビューが作成された後、もし元のテーブル(上記だとusers)に新たなレコードが追加されたとしてもマテリアライズドビューは古い結果を返し続けます。

これを避けるためには、後ほど説明するマテリアライズドビューのリフレッシュ(手動/自動)をする必要があります。

「なんだ、自動リフレッシュ機能があるなら何も問題ないじゃん」、と思われるかもですが、
そう簡単にいかない場合があります。

頻繁に書き込みが行われるようなテーブルに対しては、書き込みが行われるたびにリフレッシュするのはDBへの負荷上昇、およびパフォーマンスの劣化につながります。
さらに、基本的にリフレッシュ中はマテリアライズドビューはロックされます。
リフレッシュ中に他の

リフレッシュの頻度やタイミングは慎重に決めておく必要がります。

::note-info
CONCURRENTLYオプションを使用すると、リフレッシュ中も、リフレッシュ前のビューが参照できるようになります。
ただし、内部的に更新前のビューと更新用のビューが二つ用意されるため、
その分ストレージ消費量は大きくなるので注意してください。
TODO: ここの部分の一次資料を探す


### ストレージの圧迫

前述の通りマテリアライズドビューは通常のビューと違って結果そのものを保存しています。<br/>
そのため、マテリアライズドビューの作成に使用するクエリの結果サイズが大きくなる場合、<br/>
その分ストレージを圧迫することになることを踏まえた上でビューの設計を行う必要があります。<br/>

たとえ、今現在ストレージに余裕がある場合でも

- 参照元データの増加に対して、マテリアライズドビューの結果行数がどれだけ増えるか
- 参照元データはどの頻度で増加するか

は気にしておく必要があります。

### indexの設計

マテリアライズドビューにはテーブル同様indexを貼ることができます。
このindexは元のテーブルのindexとは独立したものであるため、
例えばビューに貼ったindexは元のテーブルに影響を与えません。
この性質はとても便利で、ビューにindexを貼ったとしても、元データの参照/書き込み性能に影響を与えません。

ただし、逆を言えば、元データのカラムにindexを貼っていて、同じカラムがビューの結果として保存されていたとしても、<br/>
マテリアライズドビューのカラムにindexを貼っていなければ、indexは効きません。

そのため、マテリアライズドビューを使用する際には十分に考えてindexの設計する必要があります。


## 実際に使ってみる

マテリアライズドビューは```CREATE MATERIALIZED VIEW [viewにつける名前] AS [結果を保存したいクエリの内容];```で作成できます。

デモをするにあたって、少し具体的なケースを考えてみましょう。<br/>
運営しているウェブサイトの訪問者の情報を格納する下記のようなテーブルがあるとします。

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
Postgresには関数ベースのindexがサポートされているので、date_trunc('hour', created_at)等に対してindexを貼ればパフォーマンス向上は見込めます。<br/>
しかし、書き込みの遅延を懸念する場合、むやみにindexを貼るのは避けたいこともあります。

そこで、このクエリに対するマテリアライズドビューを作成してみます。<br/>

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
すると、先ほどまで時間がかかっていたクエリの結果が一瞬で取得できます。
ちなみにこれをEXPLAIN ANALYZEした結果が下記です。

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

ここまで、何度もマテリアライズドビューはテーブル同様に扱えるという話をしました。<br/>
わざわざ章立てして説明するまでもないかもしれませんが、当然、絞り込みや、ソートも行えます。

先程作成した例で、2023年1月1日から3月31日までの集計結果を、集計時刻の新しい順に取得したい場合は下記のようにします。

```SQL
SELECT *
FROM hourly_access_logs_views
WHERE aggregated_hour BETWEEN '2023-01-01 00:00:00' AND '2023-03-31 23:59:59'
ORDER BY aggregated_hour DESC;
```

この結果もすぐに返ってきます。
参考のため、EXPLAIN ANALYZEの結果を添付します。

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
