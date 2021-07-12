---
title: "JestでModuleNameMapperを使うとVSCodeくんがモジュール参照してくれない時～～"
emoji: "🔧"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vscode", "typescript", "jest"]
published: true
---

悲しい時～～～～～悲し以下略

Jestには ModuleNameMapper なる、テスト対象のオブジェクトを import してくる際にパス名を特定の prefix で代用する機能があります。

例えば、以下のようなディレクトリ構成のプロジェクトであると仮定します。

```bash
$ tree  -L 1
.
├── LICENSE
├── README.md
├── dist
├── jest.config.js
├── node_modules
├── package.json
├── src
├── tests
├── tsconfig.json
└── yarn.lock
```

テストのファイル郡は `tests/` に格納するわけですが、このファイルから `src/` のファイルを参照しようと思うと相対パスで表現する場合最低でも `../src/~~~` のように長ったらしく、意味もないのに冗長になります。


そこで、設定ファイル (`jest.config.js` と `tsconfig.json`) にこのように記述しておくと

```js:jest.config.js
/** @type {import ('ts-jest/dist/types').InitialOptionsTsJest} */
module.exports = {
  ~~~~~~~~,
  moduleNameMapper: {
    "^~/(.+)$": "<rootDir>/src/$1" // ここ
  },
};
```

```json:tsconfig.json
{
  "compilerOptions": {
    ~~~~~~~,
    "paths": {
      "~/*": ["src/*"]
    },
```

テストコードの中の import してくるモジュールの path を相対パスで書かなくても Jest くんがよしなにやってくれるわけです。

 ```ts:hoge.test.ts
import hoge from '~/hoge/fuga'; // src/hoge/fuga と同義
```

# 本題

そしてここからはTS, 及び VSCode ユーザーに限った？問題になるのですが、Jestの設定で ModuleNameMapper を使うと Jest の**実行結果に問題は無い**にもかかわらず VSCode くんが「ンなモジュールねーよ」と怒ってくるわけです。以下の画像でわかると思います。

![実際のコード 赤線が引かれている](https://storage.googleapis.com/zenn-user-upload/6b6fbed96aae834d5a7fe2ed.png)

行番号横の✔は [vscode-jest](https://marketplace.visualstudio.com/items?itemName=Orta.vscode-jest)をインストールすると出てくる、テストの結果をライブモニタリングできる機能(通れば✔が付く)です。

つまり Jest の実行段階では問題が無いが、VSCodeのTS解釈としては `tsconfig.json` の設定のコンテキストから外れたファイルが変なパス指定をしているためこのエラーが出るわけです。

# `tsconfig.json` の `include` をいじってみる

VSCode のTS解釈は `tsconfig.json` の内容に基づいて解析されているので、TSの設定をいじれば読んでくれそうです。

```json:tsconfig.json
~~~~~~~,
  },
  "include": [
    "src/**/*",
    "tests/**/*" // tests に設定の状態を適用する
  ]
```

エディタから警告消えた、ヨシ！ということで万事解決ヤッターーーー　ならいいんですが、残念でした。アウトです。 
仮にこの状態で `tsc` を叩くと、テストのファイル郡まで `include` に突っ込んだことによりビルドされて `dist/` に出てきます。

`$ rm -rf dist/tests` をさせればいんじゃね？と思いもしたのですが、ビルドした成果物が `dist/src/hoge/~` のように1階層深くなるので根本的により良い解決策とは言えません。

# 正解: 設定ファイルを2つに分けてオーバーライドさせる

`tsconfig.json` には `extends` というオプションがあり、これを使うと **別の設定ファイルの内容を別のファイルに継承する** ことができます。
そして、親と子で同じキー名のオプションが存在する場合は **親の内容を子が上書き** します。この特性を利用して解決しましょう。

## 1. `tsconfig.json` を `base.tsconfig.json` などにリネーム

VSCode はTSの設定を `tsconfig.json` 以外見に行きません。なのでプロジェクトのベース設定の方を接頭辞をつけるなどでリネームし、後で `tsc` にオプションを入れることでアプローチします。

## 2. `tsconfig.json` を新しく作り、`base` を継承

`extends` で継承元を指定することですべての設定が継承されました。さらにその下で `include` を設定することで親を上書きする形で `tests/` もコンテキスト内に入れます。

```json:tsconfig.json
{
  "extends": "./base.tsconfig.json",
  "include": [
    "src/**/*",
    "tests/**/*"
  ]
}
```

この段階でテストコードを表示しても警告が消えているのが確認できると思います。

![](https://storage.googleapis.com/zenn-user-upload/32a547ebce17507873ed5b35.png)

## 3. `tsc` の参照する設定を明示的に変更

このままでは `tsc` が見に行くファイルが `tsconfig.json` のままなのでビルド結果に `tests/` が出力されます。
そのため、 `--project` のオプションでビルド用の設定ファイルが `base.tsconfig.json` であることを明示します。これで問題はなくなりました。

```bash
$ tsc --project base.tsconfig.json

$ tree -L 1 dist/
dist/
├── extensions
├── index.d.ts
├── index.js
└── index.js.map

1 directory, 3 files
```

ビルド結果も問題ないことが確認できます。ﾖｶｯﾀﾖｶｯﾀ
