---
title: "npmのパッケージをリリースしてCI/CDさせるまでやってみた"
emoji: "📁"
type: "tech"
topics: ["npm", "typescript", "circleci"]
published: true
---

以前ヒカキンシンメトリーbotを書き直したという話をしたのですが一応まだメンテナンスを続けており、それなりに改善を続けています。
今回はそのbotの新機能を入れるにあたりコードの分割の目的で自作パッケージをリリースした話をします。

# 内容

+ 既存の `Array` クラスに対し、最後の要素と引数が等しいかどうかを返す
  + `['hoge', fuga].lastItemIs('fuga')` みたいに使いたい

これだけ。故に超軽量超簡単です。
まあ今後なにか追加したいものを思いついたら追加していく感じの運用を考えているので、パッケージの名前としては `@huequica/native_extensions` という形で決めました。

# 成果物

https://www.npmjs.com/package/@huequica/native_extensions

こちらになります。

# 本体コード

現在は `Array` クラスに対しての拡張メソッド、それも1個しか作ってないのでこれで終わってます。

```ts:src/extensions/array.ts
export {};

declare global {
	interface Array<T> {
		/** Return equality what reciever's last item and param
		 * @param {any} checkItem checking item
		 * @returns {boolean}
		 */
		lastItemIs(checkItem: any): boolean;
	}
}

Array.prototype.lastItemIs = function (checkItem: any): boolean {
	const lastItem = this[this.length - 1];
	return checkItem === lastItem;
};
```

先に既存 Interface を拡張しておいて、あとからプロトタイプでメソッドの挙動を実装してるだけなのでこれだけです。

あとは今後他のクラスへの拡張を考えて `index.ts` で import してやり `export default {}` で使う側に渡してやるだけです。

# テストを書く

おおよその仕様は最初から決まっていたのであとは実装に問題がないかテストで検査をします。
と言ってもテストの正しいあり方を私は知らないのでお気持ちで書いてます。TS使ってるしこんなもんでいいでしょ(良くない)

```ts:tests/array.ts
import '~/extensions/array';
const testCase = ['hoge', 'fuga', 'piyo', 'mage', 'hage'];

describe('🧐 Array#lastItemIs', (): void => {
	test('🙅‍♂️ does not match param', (): void => {
		expect(testCase.lastItemIs('fuga')).toEqual(false);
	});

	test('🙆‍♀️ throw match param', (): void => {
		expect(testCase.lastItemIs('hage')).toEqual(true);
	});
});
```

テストスイートに絵文字使うとけっこう見やすくなるのでおすすめです。

# パッケージの公開

npmにパッケージを出す場合、パッケージの情報を `package.json` に結構ガッツリと書いたりいろいろやる感じになります。ここからは自分も知らないエリアなので調べながら頑張ってました。以下に知っておくべきことを書いていきます。

## バージョン番号管理

バージョン番号については`Major.Minor.Patch`のような値で管理するようになっていて、それぞれ `npm version [{major,minor,patch}]` で自動インクリメントしてくれます。
具体的には `package.json` の `version` のインクリメント + コミット + タグ付けまで全部やってくれるので、コードとテストを全部通してからやるのがおすすめです。

## TSのコンパイル、パッケージ利用に不要なファイルを無視する

TSで記述された npm package は JS での利用も想定されるので公開する前にJSにトランスパイルする必要があるのですが、それらは `npm publish` する際自動でやってくれないので自前で挙動を定義する必要があります。

とは言っても `scripts.prepublish` に入れるだけです。今回のケースの場合、以下のようになります。

```json:package.json
"scripts": {
    "prepublish": "rm -rf dist && npm run build",
    ~~~~~
```

また、パッケージの中にテスト用のスクリプトや `.eslintrc` などツールの設定ファイルがあっても仕方ないのでそれらは `.npmignore` というファイルに記述することで公開物のリストから除外できます。

何が公開されるのか確認することもできます。Initial publish のときは確認しておくといいでしょう。
```bash
$ npm pack --dry-run
npm notice
npm notice 📦  @huequica/native_extensions@0.0.2
npm notice === Tarball Contents ===
npm notice 1.1kB LICENSE
npm notice 832B  README.md
npm notice 240B  dist/extensions/array.d.ts
npm notice 244B  dist/extensions/array.js
npm notice 275B  dist/extensions/array.js.map
npm notice 82B   dist/index.d.ts
npm notice 163B  dist/index.js
npm notice 137B  dist/index.js.map
npm notice 981B  package.json
npm notice === Tarball Details ===
npm notice name:          @huequica/native_extensions
npm notice version:       0.0.2
npm notice filename:      @huequica/native_extensions-0.0.2.tgz
npm notice package size:  2.4 kB
npm notice unpacked size: 4.0 kB
npm notice shasum:        aaa2abce34af2cb33abd5f2c6c28e302fe5fd4d8
npm notice integrity:     sha512-v15K8/4hDzspc[...]lgvwLAWJdW0ew==
npm notice total files:   9
npm notice
huequica-native_extensions-0.0.2.tgz
```

## Initial Publish する

パッケージを Publish するときのコマンドは `npm publish` ですが、 Initial の場合は `--access public` で公開パッケージであることを明示する必要があります。これが無い状態で叩くと

> プライベートパッケージは有料会員限定機能だよ

みたいなエラー吐きます。たしかそんなんだったと思う。
最初の公開が通ればあとはオプションで明示しなくても通ります。

## CI/CD で安全に

今回のプロジェクトでは手作業でやると誤爆が発生しそうで怖いので CircleCI を使って Lint, Test, Publish まで完全に自動化しています。

具体的には

1. ブランチ切ってコードを書き、push
2. PRを作成すると CircleCI が反応して Lint, Test を実行
3. master に Merge されると `npm publish` を実行

の流れになります。以下 `.circleci/config.yml` の一部を抜粋したものです。

```yml:.circleci/config.yml
defaults: &defaults
  working_directory: ~/nativeExtensions
  docker:
    - image: cimg/node:14.16.1

~~~~~~~~~~~~~~~~~~~~~~~~

  publish:
    <<: *defaults
    steps:
      - attach_workspace:
          at: .
      - run:
          name: publishable checker install
          command: sudo npm i -g can-npm-publish # sudo 権限じゃないと入らない
      - run:
          name: Authentication via accesstoken
          command: echo "//registry.npmjs.org/:_authToken=$NPM_AUTH_TOKEN" > ./.npmrc
      - run:
          name: Publishing # バージョン番号など、不整合がないかチェックしたのち publish
          command: |
            if can-npm-publish; then
              npm publish
            else
              echo "Now module is not publishable."
            fi

workflows:
  main:
    jobs:
      - setup:
          filters:
            branches:
              ignore:
                - master
      - build:
          requires:
            - setup
            
      - lint:
          requires:
            - build
      - test:
          requires:
            - build

  deploy:
    jobs:
      - setup:
          filters:
            branches:
              only: master

      - publish:
          requires:
            - setup
          filters:
            tags:
              only: /^v.*/
```

`main` のワークフローがテストやLintのタスク、`deploy` が publish のタスクになります。
このタスクは master ブランチで発火しなくていいので setup のジョブごと `ignore: ['master']` を適用させることで無視させています。

`deploy` のなかでは `setup` のジョブでモジュールのインストールをし、その後 npm の Auth 情報をファイルに書き込んで公開可能状態であれば `npm publish` を叩いて公開、の流れにしています。

あんまり本質に関係ないですが一番上に書いている `defaults: &defaults` を使うと yml 内部の重複する情報を省略できるのを調べていて初めて知りました。

# おわりに

まさかパッケージ作る機会が来るとは思ってませんでしたが、1歩開発者として前進した気がします(適当)
一応ちゃんとテストを通るように作っていきますが、いかんせんお気持ちパッケージなので **正常な動作は保証しません。** 需要も無視します。READMEにすら書きました。

> **THIS PACKAGE IS NOT COMMING MAJOR RELEASE.**
> Working is not support.
