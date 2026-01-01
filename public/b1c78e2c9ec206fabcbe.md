---
title: VSCodeのExtensionをバージョンアップした話 with Clineに機能追加を依頼したら1分で実装が終わった話
tags:
  - Java
  - JUnit
  - VSCode
  - 個人開発
  - VSCode-Extension
private: false
updated_at: '2025-04-25T10:30:41+09:00'
id: b1c78e2c9ec206fabcbe
organization_url_name: works-hi
slide: false
ignorePublish: false
---
# VSCodeのExtensionをバージョンアップした話

## 更新したExtensionについて

https://marketplace.visualstudio.com/items/?itemName=gengen0719.java-testingpair-opener

Javaクラスを開いた状態で Ctrl + 9 を押すとテスティングペアのJavaクラスを開く（なければ作る）という超単純なExtensionです。

作った時の記事はこちら。

https://qiita.com/gengen0719/items/c3ef391d04fe42598828

ソースコードはこちら

https://github.com/gengen0719/java-testingpair-opener

## 機能追加した内容
- 複数のテスティングペアをサポートする機能を追加しました
私が保守しているプロダクトがそうなのですが（だから作った）Hoge.javaに対してHogeTest.javaとHogeIntTest.javaが存在するようなプロジェクトでも利用できるようになりました。
Ctrl + 9 もしくは Cmd + 9 を押すと、Hoge.java、HogeTest.java、HogeIntTest.java（以降繰り返し）の順に切り替わるようになっています。

- テスティングペアの関係性をカスタマイズできる設定オプションを追加しました
テスティングペアのSuffixやフォルダ構成を任意に設定できる機能を追加しました。
こんな感じでsettings.jsonに設定できます。
ペアを増やしたり減らしたりすることもできます。
```
"java-testingpair-opener.testingPairs": [
    {
        "name": "Production",
        "pattern": "src/main/java",
        "suffix": "",
        "position": 0
    },
    {
        "name": "Unit Test",
        "pattern": "src/test/java",
        "suffix": "Test",
        "position": 1
    },
    {
        "name": "Integration Test",
        "pattern": "src/int-test/java",
        "suffix": "IntTest",
        "position": 2
    }
]
```
- 自動削除機能を追加しました
テスティングペアを切り替える際に、デフォルトの内容から変更されていないテストファイルは自動的に削除されるようになりました。（プロダクションコードのファイルは消えません）

## 機能追加時に注意が必要だったポイント
### 設定の取り扱い
今回 `package.json` にデフォルトの設定を持たせてリリースしましたが、いくつか疑問が湧いたためどういう挙動なのか調べてみました。

- 設定値の挙動
    - `package.json` の `configuration` `properties` 以下に設定した名前で設定値が判別される（今回だと [java-testingpair-opener.testingPairs](https://github.com/gengen0719/java-testingpair-opener/blob/main/package.json#L31) ）
    - settings.json に設定が存在しない場合は、package.json に記載されたデフォルトの設定値が使用される
    - ユーザーが設定を変更しない限り、`settings.json` にはデフォルトの設定値は書き込まれない
    - 一度settings.jsonに書き込まれた設定は、それ以降package.jsonのデフォルト設定を読み込むことはなく、アップデートされない
    - 設定値を強制的に更新したい場合は `activate` 関数内に更新処理を書き込む

既存ユーザーの動作を変えたくない場合は注意が必要かなと思います。

### Market Placeに表示されるファイルの更新を忘れない
色々と忘れて、何回かアップしなおしたりしたので備忘として。

- `package.json` のversionの更新
MarketPlaceにVersionとして表示される値になります（大事）
- CHANGELOG.mdの更新
MarketPlaceのVersion Historyはこの内容をもとに作成されるようです。（忘れがち）

# Clineに機能追加を依頼したら実装は1分で終わった話

## 今回の機能追加で使ったポイント
今回追加した機能のメインである `複数のテスティングペアのサポート` についてはClineに以下の依頼をして実装してもらいました。

```
任意の数のテスティングペアを設定できるように機能追加をしてください
- Hoge.java の単体テストのTestingPairとしてHogeTest.java、IntegrationテストのTestingPairとしてHogeIntTest.javaとなるような設定ができる
- Ctrl + 9 を押すたびに、プロダクションコード、単体テスト、Integrationテスト（以降繰り返し）のような切り替えとファイル生成が行われる
- TestingPairは2つだけではなく任意の数を追加できる。設定画面で設定することができる。
```

## 良かった点
以下が開発体験として良かったです。

- リポジトリに対するコンテキストはREADME.mdを読み込んで補完してくれた
- 1分程度で実装が終わった
- 一発で意図通りの動作をする実装が出てきた
- コードの修正と合わせてREADME.mdも修正してくれた（超うれしい）

## 改善ポイント

注意が必要だったポイントに記載した以下は `.clinerules` に明記するなどしてコンテキストを与えた方が良かったなと思いました。
- package.jsonのversion更新作業の明記
- CHANGELOG.mdの更新作業の明記

また実装の楽さに対してテストが面倒に感じたので

- 自動テストの整備
- 実装後テスト実行して動作確認をするインストラクション

をやっておくとよりスムーズに進められそうだなと思いました。

## 大きなリポジトリで実装してもらった際との対比
仕事で保守をしている大きなリポジトリでも、主にテストコードを書く作業をやってみてもらっているのですが、そちらでは大量のインストラクションが必要で、それでもなかなか意図したコードを書いてくれず、その体験との対比が印象的でした。
よりいっそう小さくシンプルなものを組み合わせて作っていく重要性を感じています。

おわり。
