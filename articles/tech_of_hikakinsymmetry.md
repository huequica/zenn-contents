---
title: "ヒカキンシンメトリーbotの技術的なところの話"
emoji: "😎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nodejs", "typescript", "GCP", "クソアプリ"]
published: false
---

突発的にリリースして完全に技術の無駄遣いをしたbotのテクニカルなあれこれを一応書いておこうと思います。  
**今後お好きな言語で再実装しようと考えている方は参考にしてください。**
また、この記事をご覧になる前に[ブログのエントリ](https://huequica.hatenadiary.jp/entry/2021/05/19/232646)を参照することをお勧めします。

# ヒカキンシンメトリーbotってなに

[これ](https://twitter.com/HIKAKINsymmetry/status/1396366577604378628)を見ましょう

# 開発時に定めた要件

+ できる限り先代に寄せる
+ TypeScriptで書く
+ サーバレスを意識した技術選定を行う

# 採用したサービス

## 稼働場所

[GCPのComputeEngine](https://cloud.google.com/compute?hl=ja)を採用しています。開発中

> Cloud Runを使えばDockerコンテナを動かすことができる

という風に聞いていたのでそこに載せるつもりでしたが、デプロイの時にコンソールから「HTTPのポート開いてないぞ」と言われてそこで根本的にプロダクトの特徴を誤解していることに気づいて大人しくComputeEngineにしたという経緯です。要はこだわりはないです。

また、**Dockerイメージの中にGCPのサービスアカウントなどの認証情報を入れることを前提としている**ため[Container Registry](https://cloud.google.com/container-registry?hl=ja)でイメージを管理しています。
これは本来イメージ設計におけるアンチパターンですが、扱う情報が中々に多く環境変数に切り分けるのが現実的ではないと感じた為です。

## データベース

Youtubeの動画を処理したかどうかの進捗管理の目的で先代botではローカルのSQLite3を使用していました。
今回のケースの場合は最悪nodeのコンテナの中にSQLite3のバイナリ等を引っ張ってきて使用するなどすればいいのですが、それでは結果的にデータベースのストレージをメンテナンスする手間が増えてしまい非常に面倒なので[Firestore](https://cloud.google.com/firestore?hl=ja)を採用しました。

Firestoreを使った理由はぶっちゃけ最初は特に無く、フルマネージドで安いデータベースがないかなあという要求で探した結果これが一番近かったというだけです。
ですが、使ってみるとNoSQLのデータベースを使ったことが初めてなのもありますがサンプルのプロダクトが作りやすい非常に良いプロダクトだと感じます。
オブジェクトを適当に放り投げておくことができるのはかなりキックオフの段階でデータベース構造に悩むより遥かにアドバンテージとして大きいと思いました。

## 顔認識

このプロダクトにおけるコアの部分ですが、先代に倣って[Cloud Vision API](https://cloud.google.com/vision?hl=ja)を採用しています。
先代が悩んでいた部分でもあります。先代botのREADMEのchangelogにおいて使用APIが

OpenCV -> FaceAPI/OpenCV -> Cloud Vision API

と変遷していたことから選定には苦労されたのかなと思います。私の作るbotはその変遷とその末に出た結論を尊重することを選びました。

------------

その他Twitter APIやYoutube Data APIなどありますが、こちらについては特に語ることはないかなと思います。

# 使用しているモジュール

## 画像処理

nodejsで画像処理をしたい場合、[sharp](https://www.npmjs.com/package/sharp)や[Jimp](https://www.npmjs.com/package/jimp)を採用する例が多いと思います。(Googleで日本語記事出てたのはこの2つだけだった)
このプロジェクトでは最終的に[image-js](https://www.npmjs.com/package/image-js)を使用しています。

Sharpは`Crop`(切り取り)のメソッドが見当たらず、`resize()`をゴリゴリ頑張って使っていくしかなさそうという印象があったので選定の段階で却下としました。

ではJimpはどうだったのかというと、要件は全て満たしており正常通り画像も生成できていたのですがTwitterのAPIにアップロードする段階でかなりの確率で蹴られる謎の現象に遭遇して原因切り分けの結果**Jimpの吐き出してる画像はかなりの高確率で蹴られる**ことがわかったので移行しました。

その後ちゃんとTwitterAPIに通る画像処理ライブラリを探し、辿り着いたのがimage-jsでした。こちらはちゃんと想定通りに画像が生成できてTwitterへのアップロードも成功したので良かったです。
しかし使用する上で問題があり、何故か一部のメソッドがTS上で定義されていないと怒られる(選定でjsに書いたときは大丈夫)という状態であり、結果的に`require()`で無理やり`Any`にしてインポートして使用するという形に落ち着きました。なんでだろうね。

## Twitterまわり

結論からいうと以下の2つを使っています。
+ [twit](https://www.npmjs.com/package/twit)
+ [twitter-api-client](https://www.npmjs.com/package/twitter-api-client)

前者のtwitはStreamingを使用したツイートの取得に対応していたのでヒカキン氏の投稿の監視に採用していますが、基本的にレスポンスをCallback内で処理するタイプだったので扱いづらいというのが正直なところです。`@types`が提供されていたのは非常にありがたかったです。

後者のtwitter-api-clientは完全にTSで構築されている`Promise`ベースのモジュールで、画像とそれを内包したツイートをアップロードする部分に採用しています。本当はこちらにTwitterまわりの責務を全部ぶん投げてしまいたいのですがStreamingに完全対応しておらずその部分はtwitに任せているという状態です。

## 定期実行

TwitterのデータはStreamingさせているのですがYoutubeからのデータは定期取得させないといけないので、そこは[node-cron](https://www.npmjs.com/package/node-cron)を使っています。
~~ちなみに私は実行タイミングの設定間違えてFirestoreの読み取りをアホほど発生させたバカ野郎です。みんなは気を付けてね~~

