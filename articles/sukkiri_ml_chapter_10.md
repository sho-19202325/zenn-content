---
title: "スッキリわかるPython機械学習入門 第10章 より実践的な前処理"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

## 10.1 さまざまなデータの読み込み

### tsv, csv, json, 文字コードなど

※各種ファイル形式やエンコードの方法が語られているだけなので割愛

### 内部結合と外部結合

読み込んだ複数のデータフレーム同士を結合することができる。
SQLのJOINと一緒で
- INNER or OUTER
- 結合キーの指定

を行うことで表を連結することができる。
INNERとOUTERの違いもSQLと一緒。

Q. INNER JOINとOUTER JOINの違いはなんでしょう。SQL研修でやりましたよね(圧)

## 10.2 より高度な欠損値の処理

### 線形補完

今までは中央値や平均値を使用して欠損値を埋めてきた。
今回は時系列データなので違ったアプローチをしてみる。

※時系列データ - 一定間隔の時間ごとに観測され、前後に何かしらの関係があるデータ。

時系列データは前後の値に依存するという特徴がある。
そこで線形補完法というものを用いる。
まず、p366の図参照すると、atemp列のデータの一部が途切れているのがわかる。
indexの299のデータが欠損しているため折れ線グラフが繋がっていない状態だが、
大体前後のデータである0.28と0.36の間であることは予想できる。
そこで298と300を線で結んでみる。

※p367 図10-9参照

このように、前後の値を元に欠損値を予測する方法を線形補完という。

簡単のために、気温変化の例を考えてみる。

|日時|気温(°C)|
|--|--|
|3/24(日) 10:00|12°C|
|3/24(日) 11:00|null|
|3/24(日) 12:00|15°C|
|3/24(日) 13:00|16°C|
|3/24(日) 14:00|16°C|
|3/24(日) 15:00|16°C|
|3/24(日) 16:00|16°C|
|3/24(日) 17:00|15°C|

Q. 上記は杉並区の3/24(日)の気温である。杉並区における3/24(日) 11:00の気温は何度か?


A. 14°C

### 教師あり学習による補完

線形補完は時系列データのみでしか使えない。
このような場合でも「平均値」より賢い欠損値の予測方法はある?

=> **欠損値のある列を予測するモデルを作れば良い**

予測の流れ:

1. 欠損値のある行を削除
2. 削除前に欠損があった行を正解データとした予測モデルを作成
3. 1.で削除した行の特徴量を利用して欠損値の予測値を求める

本書では、「がく片弁長さ」列に二つの欠損がある。

- がく片弁長さ(欠損値あり)
- がく片幅
- 花弁幅

の三つの特徴量を使用して、

- アヤメの種別

を判別していた。

以前は
1. 「がく片弁の長さ」に対して平均値の穴埋めを実施。
2. 三つの特徴量を学習してモデルを作成
3. 評価

という流れであったが今回の手法を使用すると、

1. 「がく片弁の長さ」に欠損がある行を削除
2. 「がく片幅」、「花弁幅」を特徴量として利用して「がく片弁の長さ」の予測を行うモデルを作成
3. 1.で削除した行の「がく片幅」、「花弁幅」と2.で作成したモデルを用いて、「がく片弁の長さ」予測し、その値で欠損値を埋める
4. 補完後は元の流れと同様

という形になる。
つまり、機械学習の前処理に機械学習を利用すれば、
平均値より精度の高い欠損値の補填を行うことが可能。
さらに高度な手法もあるが、本書のレベルを超えるので割愛。

## 10.3 より高度な外れ値の処理

### マハラノビス距離

これまで外れ値の処理は散布図によって行なっていた。
しかし、特徴量の数が膨大になってくると全て目視で確認するのは至難。

**単純な距離を用いた方法**

外れ値とは「データの分布から非常に孤立しているデータ」のこと。
「孤立している」をどう分析的に捉えるのか?
=> **分布の中心を求めて、その中心からの距離を計算する**

中心から離れているということはそれだけ孤立している可能性が高いので、
あらかじめ決めておいた基準値以上かどうかを判定すればよい。

※基準値の定め方は触れられていない

Q. この方法だと、外れ値の特定という意味では十分でない。どんな場合だとうまくいかないか。

**マハラノビス距離**

二次元以上のデータの外れ値を判定する際には、マハラノビス距離というものを計算する。

図10-11では、外れ値以外は円状にデータが分布しているが、
このようなデータにおいても相関関係があることもある。

特徴量同士で相関関係があったりバラ付きに違いがあると、単純に中心からの距離を測るだけでは不十分。

図10-12の右上の点と、右下の点では中心からの距離が右上の方が遠いが、
肉眼では右下の方が外れ値に見える。

そこでマハラノビス距離というものを使う。
マハラノビス距離とは、ざっくりいうと相関やばらつきを考慮した距離。

※ 本書にはマハラノビス距離について説明がない

- [マハラノビス距離を徹底解説](https://qiita.com/yutera12/items/db425fafce2d87a25a1f)
- [【異常検知】マハラノビス距離を嚙み砕いて理解する (1)](https://qiita.com/kotai2003/items/297930db4466b71f06b0)

### 中央値を用いた外れ値の判定

中央値を利用して外れ値を判定する方法もある。

```四分位範囲(IQR) = 第3四分位数 - 第1四分位数```

を用いると全データの50%がこのIQRの中に収まるので、外れ値の判定に利用できる。

外れ値かどうかは判断する閾値は下記の通り
- 値が大きい側の閾値 = 第3四分位数 + 1.5 * IQR
- 値が小さい側の閾値 = 第1四分位数 - 1.5 * IQR