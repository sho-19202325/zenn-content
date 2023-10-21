---
title: "Template Method パターン"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

## Template Methodパターンとは

Template Methodパターンとは、汎用的なフローの骨格部分のみを親クラスに定義し、各処理の具体的な箇所をサブクラスに実装することで、
同一の手順やフローを維持しつつ、異なる振る舞いを持たせることができるデザインパターンです。

## Template Methodパターンが有効な例

特定の形式のファイルを読み込んで、それをDBに保存する処理を考えてみます。
今回はcsv, json形式のデータを取り込めるとします。

まずは、Template Methodパターンを使わない場合をみてみましょう。
```TypeScript
class JSONImporter {
  importData(rawData: string): void {

  }
}
```


## Template Methodパターンが有効でない例


