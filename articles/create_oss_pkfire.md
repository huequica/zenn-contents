---
title: "ESLint とかのインストールだるくないすか？を解決する OSS を書いてる話"
emoji: "🔥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript", "eslint", "prettier", "nuxtjs", "nextjs"]
published: false
---

# tl;dr;

https://npmjs.com/package/pkfire

# JS(TS) 書くときに入れるものって何がある？

node のプロジェクト新しく作るとき JS とか TS の開発時に決まって(ほぼ脳死で)入れるものと言ったら何があるでしょうか？
人によって違いがあるとは思いますが、だいたい以下の通りではないかなと思います。

- ESLint
- Prettier
- Jest
- TypeScript

そして、Next.js などの各種フレームワークを使う場合は ESLint とか Jest とか TypeScript が自動セットアップされている状態、というのがプロジェクト作成時に構築されてるんじゃないかなと思います。
でもこれらのセットアップが面倒だと思ったこと、土日もコード書く開発者各位は一度は思ったことあるんじゃないかなと思います。

# 具体的に何が面倒？

node のプロジェクトであることを前提に、1 から構築していく前提で話を進めます。

## Prettier x ESLint

まずコードの綺麗さを保ちたいので Prettier をインストールします。コードの安全性も保ちたいので ESLint をインストールしますね。
そしてこれらを相互運用するとき、Prettier の設定と ESLint の設定が競合してしまうとまずいので回避するために `eslint-config-prettier` をインストールしますね。
この時点で既にパッケージが 3 つインストールされたことになります。

## TypeScript x ESLint

まず TypeScript 本体のインストールが必要ですね。
それから ESLint に対応させなければいけないので、以下の 2 つが必要なのでインストールします。

- `@typescript-eslint/eslint-plugin`
- `@typescript-eslint/parser`

TS 本体も含めるとパッケージが 3 つ増えます。

---

さて、これらをコマンドに起こすと以下のような物量になります。

```sh
$ npm i -D prettier eslint

$ npm i -D eslint-config-prettier

$ npm i -D typescript

$ npm i -D @typescript-eslint/eslint-plugin @typescript-eslint/parser
```

![](https://storage.googleapis.com/zenn-user-upload/e73e428b8e2d-20230305.png)

## 何がつらいか

- インストールする物の量が結構ある
- `@typescript-eslint/eslint-plugin` なんかは 1 個のパッケージ名がめちゃんこ長い
- あえて言及しなかったが **設定ファイルの構築の手間もある**

もうこの時点で怠惰な私としては「やる気なくした」となるわけです。

# 手間を解消したい

この手間を解消しようと考えたとき、 GitHub の TemplateRepository を使って設定ファイルなどをそのまま引き継いでプロジェクトを作るのが第一の手段として考えられます。
しかし TemplateRepository のパッケージについては手動の更新が必要なため結果的にあまりいいソリューションとはいえません。

そこで次に思いついたのは `create-nuxt-app` のような CLI でひな形を作る形でした。
こちらは CLI 自体の更新さえしておけば ESLint などは都度インストールなので毎回最新が保たれます。こちらがよさそうですね。
ということで作っている CLI が `node-jeneralize/pkfire` です。

https://npmjs.com/package/pkfire

# pkfire の責務、機能

pkfire の責務(ミッション)としては以下に示す通りです

- 開発環境の最速構築をアシストする
- 環境に依存しない、もしくは環境に最適な設定を提供する

その上で現時点でこのような機能を実装しています。

## npm, yarn 両方対応

現時点でパッケージマネージャとしては npm と yarn の派閥が存在します。
どちらかしか対応してないというのは論外なのでもちろん両方対応してます。

## 使うツールチェインに応じて必要になるパッケージを選定してインストールする

パッケージのインストールが非常に手間なのは上記に示した通りなのでこれらを CLI 側でラップしています。
これにより自前で `npm i -D` とか叩かなくていいようにしています。
ユーザーがやることは使うツールを CLI の指示に従って選ぶだけです。簡単ですね。

## ある程度使えるレベルの設定ファイルを自動で出力する

TypeScript や ESLint などの設定ファイルを `--init` コマンドで生成して設定項目を書くのも非常に手間なので最初からある程度使えるものを自動生成しています。
これにより設定を 1 から構築する手間を排除することに成功しました。

## フロントエンドのフレームワークで使う場合は適した設定を出す

現時点で Next.js, Nuxt.js の設定ファイルのどちらかを検知するとそれぞれの環境であることを検出して ESLint に適した設定を自動で当てるようになっています。
具体的には以下のパッケージ設定が自動で当てられます。

- Next.js
  - `eslint-config-next`
- Nuxt.js
  - 共通
    - `@nuxtjs/eslint-module`
    - `eslint-plugin-nuxt`
    - `eslint-plugin-vue`
    - `@babel/eslint-parser`
  - TypeScript と共用する場合
    - `@nuxtjs/eslint-config-typescript`

## GitHubActions の設定ファイルなどの CI 系設定も吐き出す

GitHubActions などの CI 環境でも ESLint を動かしたりしたいですよね？
そういうケースが自分にはよくあるので、この設定ファイルについても自動で吐き出しされるようにしました。
毎回設定ファイルを他のプロジェクトからコピペしたりとかの手間が減るのでとても楽になります。

GitHubActions 以外にも、依存パッケージの更新のための dependabot 用の設定の吐き出しにも対応しています。

## `npx pkfire` で実行できる

1 回しか使わないであろうものに対してプロジェクト直下に落とさせたりするのはおかしいよね、ということでパッケージをダウンロードするひと手間を省いて `npx` で直接実行できるようにしてあります。
より手軽に扱えるかなと思います。

# 今後について

- pkfire 用の設定ファイルを書くことで挙動を定義できるようにしたい
  - 毎回質問形式というのはつらいかもしれないので
  - この設定ファイルで ESLint の設定などに追加設定をかけるようにしたら社内のルール標準化も捗りそうかも
- GitHub Actions 以外の CI 設定をサポートしたい
  - GitLab CI, CircleCI, etc...
- ドキュメンテーション
  - 全くと言っていいくらいできてないのでしたい

その他ほしい機能などの要望があれば GitHub の Issue に起票いただければと思います。

https://github.com/node-jeneralize/pkfire

# 最後に

環境構築で消耗するのはできる限り避けたい人間なのでこのような CLI を書いてメンテナンスしています。
もし自分と同じことを考えている人がいましたらぜひ一度お試しください。
