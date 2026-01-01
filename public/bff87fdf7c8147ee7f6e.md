---
title: AWS Lambda SnapStartの使い方とポイントのまとめ
tags:
  - AWS
  - 初心者向け
  - lambda
  - snapstart
private: false
updated_at: '2025-07-16T00:05:03+09:00'
id: bff87fdf7c8147ee7f6e
organization_url_name: works-hi
slide: false
ignorePublish: false
---
# この記事について
AWS LambdaのSnapStartを使ってみた際に引っかかったポイントや、最初は分からなかったことがあったので書きます。初めて触る方の何かの助けになれば幸いです。

# SnapStartとは？
SnapStartとはLambda関数の実行時間を短縮するための仕組みです。

Lambda関数はサーバーレスアーキテクチャです。
関数が呼び出されたとき、Lambdaは自動的に実行環境を立ち上げ、ソースコードの準備を行い、処理を実行します。その後しばらく関数の呼び出しのない状態が続くと、自動的に実行環境を停止します。

このように必要な時だけ実行環境を立ち上げることで低料金での処理の実行を実現しているものと思います。しかしサーバーの準備ができていない状態から処理を実行（コールドスタート）する際に、実行環境の準備をする分、処理に時間がかかってしまいます。

これはランタイムの準備に時間がかかるJavaのランタイムでも3秒〜5秒程度ですが（すごい）、例えばWebアプリケーションのバックエンドにLambda関数を利用する場合など、ユーザーにリアルタイムでレスポンスを返す必要がある時にはこの時間が問題になる場合もあると思います。

SnapStartは初期化された実行環境のメモリとディスクの状態のスナップショットを取得し、そのスナップショットを暗号化してキャッシュすることで、低レイテンシのアクセスを実現する仕組みです。

これによってコールドスタート時のランタイムの準備時間を大幅に減らすことができます。

# Lambdaスナップスタートの使い方
## 使い方
ManagementConsoleから設定する場合は、
Lambda関数を選択しConfiguration -> General configurationのEditを開きます。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1126770/18c277ce-0647-48c9-bb90-fa27a75ebffb.png)

SnapStartをPublishedVersionsに設定します。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1126770/3af8516c-7c46-4a87-93be-47ea0305e788.png)

## 設定時の注意点
### SnapStartの対象となるのは公開されたVersion
設定画面の表示にもあるようにSnapStartの対象となるのは公開されたバージョンです。
バージョンを指定せずにLATESTのLambda関数を呼び出した際はSnapStartが動作しない点に注意が必要です。

### API Gatewayから呼び出す設定をする際にサジェストが出ない
API GatewayのIntegration request settingsからLambda関数を呼び出す際は、以下のような形でARNを指定します。
`arn:aws:lambda:[region]:[ID]:function:[function名]:[Alias名 or Version]`
例
`arn:aws:lambda:ap-northeast:123456789012:function:execute:execute-dev`

:::note warn
この時Lambda関数自体は関数名などを入力するとサジェストがされますが、VersionやAliasはサジェストされません。
（恥ずかしながらここでちょっとハマってしまいました）
:::

例えば `execute` と入力すると 
`arn:aws:lambda:ap-northeast:123456789012:function:execute` は出てくるが
`arn:aws:lambda:ap-northeast:123456789012:function:execute:execute-dev` は出てこない。[^1]
[^1]: おそらくVersionやAliasを含めると候補の数が増えてしまうし、この部分は自分で指定することも容易なためだと思います。

Lambda関数の関数名のあとのAliasやversionの指定部分は自分で入力する必要があります。
サジェストはされませんが入力して指定すれば動作します。


# 細かい動作やログの出方について
SnapStartを動かしながらClowdWatchLogsを見てわかったことを書きます。

## Lambda関数のClowdWatchLogsへの出方
Lambda関数ごとにLogGroupが作成される

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1126770/bdfe70ee-7f82-4ac5-b39f-7e8c4d04e6c7.png)

Log Groupの中のLog Streamは実行マシン単位で出力される

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1126770/c5f3bb7c-656f-4ef4-a831-05b9833fa1d2.png)

ここで日付の後ろの `[21]` のような部分がVersionでその後ろがおそらく実行マシンの識別子。
冒頭のLambda関数が呼び出されて処理を実行し、しばらく呼び出しのない状態が続いて実行マシンを停止するまでが1つのLog Streamになっている。

## SnapStartを使っていない場合

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1126770/1e40769b-f12a-458e-88dd-af51826a25bc.png)

Log Streamの冒頭はINIT_STARTで始まる。
ここが実行マシンを準備する処理。
Javaの場合Handlerクラスのインスタンス化が行われるため、コンストラクタに記載した処理が実行される。
このログではHikariCPを利用したRDBのコネクションの取得とDynamoDBクライアントの初期化を行っている。
その後Start RequestId以降のEventが、Lambda関数が受け取ったリクエストを処理しているログ。
リクエストの処理開始までに3秒弱かかっていることがわかる。

## SnapStartを使っている場合
### SnapShotが作成されるタイミング
SnapStartで利用されるSnapShotはLambda関数のVersionが作成された時点でリクエストをトリガーとせずに作成される。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1126770/a9d8f0cb-6342-4c79-a8fa-bd1dc1633b3d.png)

先ほどと同様の処理が書かれた関数のVersionを作成した際のLog Stream。
リクエストは受け付けておらず、INIT処理だけを実行している。
SnapShotを作っているからか、Init Durationは先ほどよりかなり長い。

### リクエストを受け付けた際の処理

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1126770/41abf774-a90f-4002-b7b9-8c5c5bf592c7.png)

SnapShotが作成された状態でリクエストを受け取るとSnapStartを利用していない場合のINIT_STARTではなくRESTORE_STARTというログが出力され、SnapShotから実行マシンが作成されます。
その後 Restore Durationが出力されており、700ms 弱で処理を開始できていることがわかります。

## ロジックを書く際の注意点
### 初期化処理に失敗するとSnapShotが作成できず、以降の関数呼び出しが失敗する
例えばRDBのコネクションを取得する処理などを書いている場合、バージョン作成時にもRDBに接続できる必要があります。

### RDBのコネクション取得など自身のメモリとディスクの状態だけでは再現できない状態を初期化処理に書く場合
例えばSnapShot作成時に取得したコネクションが切断された場合、SnapShotには切断されたコネクションのインスタンスを持ち続けることになります。
このような状態になった場合、再取得のロジックがないとエラーになります。
Hikari CPはかなり高速にコネクションのValidationを行い、問題があれば再取得を行ってくれるようですが、再取得がボトルネックになる可能性があります。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1126770/982723ec-e753-4718-9592-eaca0f8abcad.png)


# 更なる速度改善を目指す場合
Javaの場合は実行コードをNative化するという方法でも改善する可能性があるようです。
SnapStartの利用によってかなりコールドスタート時の実行速度が改善しましたが、もう一歩早くなって欲しい気持ちもあるため試してみたいです。

# 感想
サーバレスアーキテクチャを利用することで、利用者は起動停止やスケーリングなどの実行環境の管理で考えなければいけないことが減り、アプリケーションの処理の実装に集中することができますが、完全に仕組みを知る必要がなくなるというものではなく、サーバレスアーキテクチャの特性を知った上で実装を行う必要があるのだなぁと思いました（当たり前ですが）

おわり。
