---
title: "TypeScriptで理解するTemplate Method パターン"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["TypeScript", "DesignPattern"]
published: false
---

## Template Methodパターンとは

Template Methodパターンとは、汎用的なアルゴリズムの骨格部分のみを親クラスに定義し、
各処理の具体的な箇所をサブクラスに実装することで、
同一の手順やフローを維持しつつ、異なる振る舞いを持たせることができるデザインパターンです。

## Template Method パターンの登場人物

- **AbstractClass(抽象クラス)**
AbstractClassは、汎用的なアルゴリズムの骨格を実装するテンプレートメソッドと、
その中で使用される抽象メソッドを定義するための抽象クラスです。
汎用的な処理は、テンプレートメソッドとして実装され、具体的な処理は、サブクラスに実装を任せます。

- **ConcreteClass(具象クラス)**
ConcreteClassは、AbstractClassを継承し、抽象メソッドを実装することで、
テンプレートメソッドを具体的に実装するための具像クラスです。
共通の処理を維持しつつ、異なる振る舞いを持たせたい場合には、ConcreteClassを複数実装します。

## Template Methodパターンのクラス図とサンプルコード

下記はTemplate Methodパターンを表したクラス図とサンプルコードです。

![](/images/template_method_pattern/class_diagram.png)

```TypeScript
abstract class AbstractClass {
    templateMethod() {
        this.method1();
        this.method2();
        this.method3();
    }

    protected abstract method1() {}

    protected abstract method2() {}

    protected abstract method3() {}
}

class ConcreteClass extends AbstractClass {
    protected method1() {
        console.log('method1');
    }

    protected method2() {
        console.log('method2');
    }

    protected method3() {
        console.log('method3');
    }
}
```

templateMethod()はテンプレートメソッドと呼ばれ、ここに汎用的な処理を実装します。
今回のサンプルコードでは、method1(), method2(), method3()を呼び出しています。

ConcreteClassでは、method1(), method2(), method3()の具体処理を実装することで、
サブクラス内でテンプレートメソッドの個別処理を具体的に実装しています。

汎用的な処理の骨格を維持しつつ、個別処理の振る舞いを変えたい場合は、
AbstractClassを継承したConcreteClassを新しく作成し、各種抽象メソッドを実装します。

```TypeScript
class ConcreteClass2 extends AbstractClass {
    protected method1() {
        console.log('method1 in ConcreteClass2');
    }

    protected method2() {
        console.log('method2 in ConcreteClass2');
    }

    protected method3() {
        console.log('method3 in ConcreteClass2');
    }
}
```

## Template Methodパターンの使用例

今度はより具体的な使用例を見てみましょう。
下記のようなデータインポート処理を持つクラスを実装するとします。
1. constructorでファイルパスを受け取る
2. ファイルを開く
3. ファイルの形式に応じてファイルのバリデーションを行う
4. ファイルの形式に応じてファイルのパースを行う
5. ファイルの形式に応じてデータの挿入を行う
6. ファイルを閉じる

対応したいファイルの形式はJSONとCSVの2種類とします。
形式によってバリデーションやパース、データの挿入の処理が異なるため、
それぞれの処理は別々に実装する必要があります。

実際にファイルやDBを用意して処理を実装するのは大変なので、
下記のような出力結果を返すだけのクラスを実装してみます。
```TypeScript
const JsonImporter = new JsonImporter('file.json');
JsonImporter.execute();
// expected output:
//
// Open file: file.json
// Validate json file
// Parse json file
// Insert json data
// Close file: file.json

const CsvImporter = new CsvImporter('file.csv');
CsvImporter.execute();
// expected output:
//
// Open file: file.csv
// Validate csv file
// Parse csv file
// Insert csv data
// Close file: file.csv
```

まずは、Template Methodパターンを使わない場合を考えてみます。
```TypeScript
class JsonImporter {
    constructor(
        protected filePath: string
    ) {}

    execute() {
        this.openFile(this.filePath);
        this.validateFile();
        this.parseFile();
        this.insertData();
        this.closeFile(this.filePath);
    }

    openFile(filePath: string) {
        console.log(`Open file: ${filePath}`);
    }

    closeFile(filePath: string) {
        console.log(`Close file: ${filePath}`);
    }

    protected validateFile() {
        console.log('Validate json file');
    }

    protected parseFile() {
        console.log('Parse json file');
    }

    protected insertData() {
        console.log('Insert json data');
    }
}

class CsvImporter {
    constructor(
        protected filePath: string
    ) {}

    execute() {
        this.openFile(this.filePath);
        this.validateFile();
        this.parseFile();
        this.insertData();
        this.closeFile(this.filePath);
    }

    openFile(filePath: string) {
        console.log(`Open file: ${filePath}`);
    }

    closeFile(filePath: string) {
        console.log(`Close file: ${filePath}`);
    }

    protected validateFile() {
        console.log('Validate csv file');
    }

    protected parseFile() {
        console.log('Parse csv file');
    }

    protected insertData() {
        console.log('Insert csv data');
    }
}
```
execute(), openFile(), closeFile()はどちらのクラスでも同じ処理を行っています。
そのため、これらの処理を変更する場合には両方のクラスを変更する必要があります。

今は二つの形式しか対応していないので、それほど問題にはなりませんが、
たとえばXMLやYAMLなど、より多数の形式に対応するクラスが増えていくと、
実装や修正に手間がかかりますし、対応漏れも発生しやすくなります。

そこで、今度はTemplate Methodパターンを使って実装してみましょう。
```TypeScript
abstract class DataImporter {
    constructor(
        protected filePath: string
    ) {}

    execute() {
        this.openFile(this.filePath);
        this.validateFile();
        this.parseFile();
        this.insertData();
        this.closeFile(this.filePath);
    }

    openFile(filePath: string) {
        console.log(`Open file: ${filePath}`);
    }

    closeFile(filePath: string) {
        console.log(`Close file: ${filePath}`);
    }

    protected abstract validateFile(): void;
    protected abstract parseFile(): void;
    protected abstract insertData(): void;
}

class JsonImporter extends DataImporter {
    protected validateFile() {
        console.log('Validate json file');
    }

    protected parseFile() {
        console.log('Parse json file');
    }

    protected insertData() {
        console.log('Insert json data');
    }
}

class CsvImporter extends DataImporter {
    protected validateFile() {
        console.log('Validate csv file');
    }

    protected parseFile() {
        console.log('Parse csv file');
    }

    protected insertData() {
        console.log('Insert csv data');
    }
}
```
DataImporterクラスはAbstractClassに相当し、
JsonImporterクラスやCsvImporterクラスはConcreteClassに相当します。
また、execute()がテンプレートメソッドであり、
validateFile(), parseFile(), insertData()が抽象メソッドとなっています。

このケースでは先ほどとは異なり、execute(), openFile(), closeFile()はDataImporterクラスに実装されています。

これらの処理は、JsonImporterクラスやCsvImporterクラスにおいては変更する必要がないため、
親クラスに実装しておくことで、サブクラスの実装を簡略化することができます。

また、validateFile(), parseFile(), insertData()は抽象メソッドとして定義されており、
形式によって処理が異なる部分に関しては、サブクラスに実装を任せています。

Template Methodパターンを使うことで、
execute()の処理を変更する場合には、DataImporterクラスのみを変更するだけで済みます。
さらに、新しい形式に対応したい場合には、DataImporterクラスを継承した新しい具像クラスを作成し、
validateFile(), parseFile(), insertData()を実装するだけで済むようになります。

## Template Methodパターンが有効な場合/有効でない場合

上記で見てきたように、Template Methodパターンは以下の場合に有効です。
- 汎用的な処理を持ち、各ステップで異なる振る舞いを持つクラスを複数作成したい場合

逆に、以下の場合ではあまり有効ではありません。
- 共通する処理が少ない場合
- 異なる振る舞いを持つクラスが少ない場合

これらのケースではAbstractClassを実装するコストの方が高くつき、
コードが不必要に複雑になる可能性があります。

## 参考

- [Java言語で学ぶデザインパターン入門 第3版 (結城 浩)](https://www.amazon.co.jp/Java%E8%A8%80%E8%AA%9E%E3%81%A7%E5%AD%A6%E3%81%B6%E3%83%87%E3%82%B6%E3%82%A4%E3%83%B3%E3%83%91%E3%82%BF%E3%83%BC%E3%83%B3%E5%85%A5%E9%96%80%E7%AC%AC3%E7%89%88-%E7%B5%90%E5%9F%8E-%E6%B5%A9/dp/4815609802?language=ja_JP)





