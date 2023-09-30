---
title: "Azure Functionにバインド拡張機能を追加する方法"
emoji: "👏"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Azure"]
published: false
---

## はじめに

Azure Functionを各種リソースとバインドさせる場合、
デフォルトの機能のみだと、痒いところに手が届かない場合があります。
そこで、登場するのがバインド拡張機能です。

例えば、EventHubTriggerを使って複数メッセージを一括で取得する際、
デフォルトでは最大でも10件ずつしか受け取ることができませんが、
バインド拡張機能を使えば、最大件数の調整が行えるようになります。

バインド拡張機能の設定自体はhost.jsonで行えるのですが、その前にバインド拡張機能の追加を行う必要があります。

バインドを登録する方法は使用する言語や環境によって異なります。
今回は**Functions.2.x**以降かつ、**.NET以外(nodejs環境)**で設定する方法を紹介します。

詳しくは、下記の公式ドキュメントを参照してください。

[Azure Functions バインド拡張機能を登録する](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-bindings-register)

## Azure Function諸概念解説

バインド拡張機能の登録を解説するにあたって、
Azure Functionの諸概念を簡単に紹介しておきます。

### トリガー

関数の呼び出し方法を定義するもので、その名の通り、関数実行の引き金となります。
一つの関数には一つのトリガーを含める必要があり、それより少なくても多くてもいけません。

トリガーの例としては
- 時間で関数を呼び出す、Timer Trigger
- http(s)で関数を呼び出す、Http Trigger
- EventHubにメッセージがenqueueされた時に関数を呼び出す、EventHub Trigger

などがあります。

バンドル拡張機能にトリガーはあまり関係がないのですが、
トリガーとバインドはセットで知っていた方がわかりやすいと思うのでここで触れておきます。

### バインド

関数に別のリソースを紐づけることです。
バインドには、
- 関数へ入力を行うリソースとの紐付けを行う入力バインド
- 関数の出力先として指定するリソースとの紐付けを行う出力バインド

の2種類があります。
バインドはトリガーと異なり、指定しないことも、複数指定することも可能です。

バインド先として、指定できるリソースの例としては、
- EventHub
- ServiceBus
- CosmosDB

などがあります。

### バインド拡張機能

その名の通りバインドに対する拡張機能です。
最初に触れた通り、この機能を使用することでバインドに対してより細かい設定や制御を行えるようになります。

### 拡張機能バンドル

バインド拡張機能と字面が似ていて初見だと混乱しますよね。
拡張機能バンドルとは、互換性のあるバインド拡張機能のセットを関数アプリに追加するためのものです。
いろんなリソースのバインド拡張機能を一つにまとめてパッケージ化したものと思ってもらえればわかり易いと思います。

## バインド拡張機能の登録

nodejs環境でバインド拡張機能を使用するには
- 拡張機能バンドルを使用する方法(公式推奨)
- 明示的に拡張機能をインストールする方法

という2種類があります。

## 拡張機能バンドルを使用する方法

拡張機能バンドルを使用するには、host.jsonに"extensionBundle"を設定する必要があります。

実は、FunctionCoreTools等でFunctionプロジェクトを作成すると、
デフォルトでhost.jsonに下記のような記載があるはずです。
```json
  "extensionBundle": {
    "id": "Microsoft.Azure.Functions.ExtensionBundle",
    "version": "[4.*, 5.0.0)"
  }
```
つまり、すでに拡張機能バンドルは導入されているということです。
もし、拡張機能バンドルのバージョンを変更したい場合は"version"の項目を下記のように修正します。
- 1系が使いたい場合 => ```"version": [1.*, 2.0.0]```
- 2系が使いたい場合 => ```"version": [2.*, 3.0.0]```
- 3系が使いたい場合 => ```"version": [3.*, 4.0.0]```
- 4系が使いたい場合 => ```"version": [4.*, 5.0.0]```

バージョンを変更し、再度functionをデプロイし直すと、指定した拡張機能バンドルが使用されるようになります。

拡張機能バンドルの各バージョンごとにインストールされる拡張機能のリストは公式がgithub上に公開している、
[Azure/azure-functions-extension-bundles](https://github.com/Azure/azure-functions-extension-bundles)の[src/Microsoft.Azure.ExtensionBundle/extensions.json](https://github.com/Azure/azure-functions-extension-bundles/blob/v4.x/src/Microsoft.Azure.Functions.ExtensionBundle/extensions.json)を参照してください。

適宜、確認したいバージョンにブランチのバージョンを合わせてください。
公式ドキュメントに、[拡張機能バンドルのバージョンとリポジトリの表](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-bindings-register#extension-bundles:~:text=%E6%AC%A1%E3%81%AE%E8%A1%A8%E3%81%AB%E3%80%81%E7%8F%BE%E5%9C%A8%E4%BD%BF%E7%94%A8%E5%8F%AF%E8%83%BD%E3%81%AA%E6%97%A2%E5%AE%9A%E3%81%AE%20Microsoft.Azure.Functions.ExtensionBundle%20%E3%83%90%E3%83%B3%E3%83%89%E3%83%AB%E3%81%AE%E3%83%90%E3%83%BC%E3%82%B8%E3%83%A7%E3%83%B3%E7%AF%84%E5%9B%B2%E3%80%81%E3%81%8A%E3%82%88%E3%81%B3%E3%81%9D%E3%82%8C%E3%82%89%E3%81%AB%E5%90%AB%E3%81%BE%E3%82%8C%E3%82%8B%E6%8B%A1%E5%BC%B5%E6%A9%9F%E8%83%BD%E3%81%B8%E3%81%AE%E3%83%AA%E3%83%B3%E3%82%AF%E3%82%92%E7%A4%BA%E3%81%97%E3%81%BE%E3%81%99%E3%80%82)が記載されているのでこちらも併せてご参照ください。

こちらを見るとわかる通り、拡張機能バンドルを設定するだけで、各種バンドルで使用可能なリソースの拡張機能がインストールされるため、個別に拡張機能をインストールする必要がありません。

拡張機能バンドルの使用は公式が推奨している方法であるため、基本的にはこちらを利用するのが良いと思います。

## 明示的に拡張機能をインストールする方法

特定のバージョンの拡張機能を使いたいが、拡張機能バンドルだとそれが実現できない、
そのような場合に有効なのが、この明示的なインストールです。

ただし、明示的に拡張機能をインストールする場合、拡張機能バンドルと同時には使用できないので注意してください。

明示的に拡張機能をインストールするには[func extension install](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-core-tools-reference?tabs=v2#func-extensions-install)コマンドを実行します。
```
func extensions install --package Microsoft.Azure.WebJobs.Extensions.<EXTENSION> --version <VERSION>
```
例えば、EventHub拡張機能の5.0.1を使用したい場合、
```
func extensions install --package Microsoft.Azure.WebJobs.Extensions.EventHubs --version 5.0.1
```
を実行します。

この時、host.jsonに拡張機能バンドルの設定が記載されていると下記のような警告が出力され、インストールが実行されません。
```
No action performed. Extension bundle is configured in [host.jsonへのパス]
```

この警告が表示される場合は、host.jsonの"extensionBundle"の記述を削除してください。

手動で拡張機能をインストールする方法を取ると、個別に各拡張機能をインストールする必要があるため、バンドルよりも管理する手間が増えます。
もし、使いたい拡張機能のバージョンが拡張機能バンドルで用意されているのであれば、明示的なインストールは避けた方が良いでしょう。

## おまけ: host.jsonの設定について

最初にhost.jsonの設定用ドキュメントを確認した時、少し混乱したのでおまけとして触れておきます。
例えばEventHub Trigger Functionで使用できるhost.jsonの設定は[こちらの公式ドキュメント](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-bindings-event-hubs?tabs=in-process%2Cextensionv5&pivots=programming-language-javascript#hostjson-settings)の通りです。

こちら参照いただくとわかる通り、拡張機能のバージョンごとに使用できる設定が異なります。
具体的には、
- 拡張機能 v5.x以上
- 拡張機能 v3.x以上
- Functions v1.x

と3つのタブが存在しています。 ※2023年9月30日現在

ここで記載されている拡張機能とは、拡張機能バンドルのことではなく、バインド拡張機能のことを指しています。
つまり、拡張機能バンドルを使っている際にどの設定が有効か確認するためには、前述の公式リポジトリ上にある[extensions.json](https://github.com/Azure/azure-functions-extension-bundles/blob/v4.x/src/Microsoft.Azure.Functions.ExtensionBundle/extensions.json)を参照する必要があります。

EventHubの拡張機能は[こちら](https://github.com/Azure/azure-functions-extension-bundles/blob/v4.x/src/Microsoft.Azure.Functions.ExtensionBundle/extensions.json#L49)に記載があります。
これを参照するとv4.xのバンドルでは、v5.xの拡張機能がインストールされるので、利用できるhost.jsonの設定は拡張機能v5.x以上のものとなります。
今度はv2.xのバンドルを確認すると、v4.xの拡張機能がインストールされるので、
利用できるhost.jsonの設定は拡張機能v3.x以上のものとなります。

一度理解すれば簡単ですが、拡張機能バンドルのバージョンと取り違えると、意図した設定で関数が動作しないため、注意してください。
