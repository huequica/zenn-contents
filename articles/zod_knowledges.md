---
title: "zod と2年間戦ってきて便利だった機能を晒すスレ"
emoji: "💎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript", "zod"]
published: true
---

zod と付き合い始めてもうすぐ2年が経過しようとしていますが未だに「こいつなしでは俺はもう生きていけない」という状態です。
そしてこの間で結構 zod の取り扱いの knowledge が貯まってきたので、一旦ここで放出してみようと思います。
ちなみに他の方の「これ便利やでw」も知りたいのでこのタイトルにしました。

# transform 前のモデルの型情報を取りたい

zod には [transform](https://zod.dev/?id=transform) という関数があり、任意のスキーマを変形したり拡張したりすることができます。

以下のサンプルコードの例で言うと、`hoge` というキーを入力として受取りその文字列長を `hogeLength` というキーで追加して返すという感じですね。

```ts
const TransformedModel = z.object({
  hoge: z.string()
}).transform(({ hoge }) => ({
  hoge,
  hogeLength: hoge.length(),
}));
```
で、このモデルを `z.infer` に食わせると **transform 後の型情報** が返されます。
こうなると例えば関数の引数に transform 前の値を受けとり、その中で `parse` を用いて transform したものを扱いたいケースに困るわけです。

簡単な解消方法として `z.object` を分離して別個に宣言して `z.infer` に食わせるなどがありますがもっとスマートな解消法として `z.input` があります。

これは transform 前の型を作るための utility type で、以下のように扱えます。

```ts
const TransformedModel = z.object({
  hoge: z.string()
}).transform(({ hoge }) => ({
  hoge,
  hogeLength: hoge.length(),
}));

// { hoge: string; }
type BeforeTransformType = z.input<typeof TransformedValue>;

// { hoge: string; hogeLength: number; }
type TransformedType = z.infer<typeof TransformedValue>;
```

ref: https://zod.dev/?id=zodtype-with-zodeffects

# enum の値を pick した、あるいは omit したものを作りたい

zod の enum 定義は以下のように記述することで実現できますが、これを一部だけ pick したものを使いたいケースがあると思います。

```ts
const SampleEnum = z.enum(['banana', 'apple', 'grape']);

// 'banana' | 'apple' | 'grape'
type EnumType = z.infer<typeof SampleEnum>;
```

しかし ZodEnum には `pick` だったり `omit` だったりがないわけですね、どうしようとなるわけです。
実は `pick` は `extract` 、 `omit` は `exclude` の名前で用意されていてこれを使えます。

```ts
const SampleEnum = z.enum(['banana', 'apple', 'grape']);

// 'banana' | 'grape'
const ExtractedEnum = SampleEnum.extract(['banana', 'grape']);

// 'apple' | 'grape'
const ExcludedEnum = SampleEnum.exclude(['banana']);
```

ref: https://zod.dev/?id=zod-enums

# transform 前のモデルを扱いたい

:::message
この項で説明する `innerType` と `sourceType` ですが公式ドキュメント及びコード上の jsDoc が存在しません。もしかしたら今後変更されたり消される可能性があります。
:::

冒頭で説明したような型情報を取りたい場合は `z.input` を使えば取れますがモデルそのものを使いたい場合はどうでしょう。
以下の例で言うと、 `hoge` のキーの定義を取得して別のモデルに使いたい場合ですね。

```ts
const TransformedModel = z.object({
  hoge: z.string()
}).transform(({ hoge }) => ({
  hoge,
  hogeLength: hoge.length(),
}));

const PickedModel = z.object({
  // shape がないのでエラーが出る
  hoge: TransformedModel.shape.hoge,
  fuga: z.number(),
});
```
しかし ZodEffect がかかっているモデルには `shape` が存在せず、もちろんその先の `hoge` キーも扱うことができません。どうしようとなるわけです。

このようなケースの場合は `sourceType` あるいは `innerType` を使って ZodEffect を外すことで対応可能です。

```ts
const TransformedModel = z.object({
  hoge: z.string()
}).transform(({ hoge }) => ({
  hoge,
  hogeLength: hoge.length(),
}));

const PickedModel = z.object({
  // sourceType を使ってもOK
  hoge: TransformedModel.innerType().shape.hoge,
  fuga: z.number(),
});
```

`sourceType` と `innerType` の挙動の違いですが、[実装](https://github.com/colinhacks/zod/blob/5041dfa3521ec98066e60f97681adf9604b66e52/src/types.ts#L4618-L4626)を見る感じ

- innerType
  - シンプルに1段階 Effect を外したものを返す
- sourceType
  - ZodEffect がなくなる(根底の型定義にたどり着く)まで再帰して返す

という差があるようです。今回の例の場合は1段階しか transform をしていないのでどちらを使っても差がありません。
