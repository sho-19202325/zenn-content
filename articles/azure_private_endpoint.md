---
title: "[仮]Azure Private Endpointという名の不可解"
emoji: "👋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

NOTE: 別記事でもいいかも
## おまけ: Function側で指定するCosmosDBエンドポイントについて

おまけとして自分が少し詰まったエンドポイント指定部分の解説をする。

具体的には
- https://<cosmosdbアカウント>.document.com
- https://<cosmosdbアカウント>.privatelink.document.com

のどちらを指定するかということである。
結論、この記事で作成したものと同じように前者を指定するのが正しい。
理由は二つある。
1. .privatelinkなしの場合でもprivateDNSが正しく設定されていれば、期待するプライベートipに解決してくれるので、問題ない(リダイレクト時の挙動についてもう少し調べたい)
2. cosmosdb側には後者のドメインに対する証明書が登録されていない。
    実際、後者でアクセスしようとすると下記のようなエラーが出る。

もし、前者を指定して、パブリックアクセスからのリクエストは受け付けないという由のエラーが出た時は、
おそらく名前解決がうまく行っていないので、高度なツール->Kudu->Bashから下記を実行する。
```
nslookup https://<>.document.com
nslookup https://<>.privatelink.document.com
```

名前解決がうまく行って

- Vnet統合を行なっているか
- Vnet統合を行なったsubnetが所属するVNet内に、対象のprivate endpointが存在するか
- PrivateDNSが対象のVNetにリンクされているか

を確認してみると良い。
