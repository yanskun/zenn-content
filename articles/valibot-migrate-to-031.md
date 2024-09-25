---
title: "Valibot の v0.31.0 への Migration 対応をした。"
emoji: "🔁"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["valibot", "TypeScript"]
published: true
---

まだまだ Zod を採用しているプロダクトが多い。

![](/images/valibot-migrate-to-031/GitHubStarHistory.png)

が、  
[Finswer](https://www.finswer.jp/Finswer-Inc-f9k-Inc-f0c30c239f11462f81477246bf3875d8) では、Schema Validation ライブラリに Zod ではなく、Valibot を採用している。

https://github.com/fabian-hiller/valibot

:::message
本記事で話すこと
- Migration Command で対応してくれるところ
- 手動で対応しなければならないところ
:::

:::message alert
話さないこと
- Valibot の説明
- Valibot の採用理由
:::


## Migration to 0.31.0

https://valibot.dev/guides/migrate-to-v0.31.0/


> Migrating Valibot from an older version to v0.31.0 isn't complicated. Except for the new pipe method, most things remain the same. The following guide will help you to migrate automatically or manually step by step and also point out important differences.

Migration コマンドを実行すればすぐ終わるよ。  
と記載があったがそうではなかったため、備忘録として記載しておきます。  
（実際は、今後私が v0.31.0 への対応をすることはないのですが。）


### Automatic Migration

Guide にもある通り

```shell
npx codemod valibot/migrate-to-v0.31.0
```

こちらのコマンドで以下の変更については、対応してくれます。

- `Input/Output` → `InferInput/InferOutput`
- `v.string([v.uuid()])` → `v.pipe(v.string(), v.uuid())`

この変更はさっくりと CLI で対応してもらえたので、  
あまり問題がなかったです。

#### `Input/Output` → `InferInput/InferOutput`

命名が変更されただけなので、parser も書きやす立ったのだろう。対応してくれた。  

##### 一部発生した問題（理由は謎）

```ts
const FormSchema = v.object({
    name: v.string(),
    age: v.number(),
})
type FormInput = Input<typeof FormSchema>
```

を Migration した際に、一部

```diff ts
const FormSchema = v.object({
    name: v.string(),
    age: v.number(),
})
- type FormInput = Input<typeof FormSchema>
+ type InferInput = InferInput<typeof FormSchema>
```

となってしまったケースが何件かあった。  

詳細に記憶してなくて申し訳ないが、  
おそらく `Input` suffix が悪さをしていたんじゃないかと思われます。

#### `v.string([v.uuid()])` → `v.pipe(v.string(), v.uuid())`

他には

```diff ts
- v.number([minValue(10, '10以上にして'), '数字にして'])
+ v.pipe(v.number('数字にして'), v.minValue('10以上にして'))
```
 
この変更は、ErrorMessage がどこにあたっているのか？というのが直感的になったように感じます。

### Manual Migration

以降は、 CLI での Migration ができなく、  
かつ実際に遭遇したもののみを紹介します。

#### `transform` の syntax 変更

これはかなり直感的になったので嬉しかったです。
が、かなり使っている箇所が多く、 Migration がかなり辛かったです。  

Migration ガイドにもある通り、

```diff ts
- const TransformedSchema = v.transform(v.string(), (input) => input.length);
+ const TransformedSchema = v.pipe(
+   v.string(),
+   v.transform((input) => input.length)
+ );
```

これもかなりよみやすく、書きやすくった印象です。  
`v.string()` もそれ単体で使うのではなく、他の Validation のルールと併用することが多いため、  
「`v.pipe()` をいちいちつけなければならないのか。」というネガティブなイメージもなかったです。

#### `coerce` の廃止

同様にこちらも利用頻度が多く辛かったです。  
また、 `transform` へ書き直す必要がありました。  

> The coerce method has been removed because we felt it was an insecure API. In most cases, you don't want to coerce an unknown input into a specific data typed

まあわかる。

困ったのは、過去の人が書いた Schema で `coerce` が使われてるパターンである。
「なぜか String にも Number にも対応しなきゃいけない。」みたいなことになっていたりする時がある。  

Component から修正をしても良いのだが、  
それをすると、それをすると PR のコンテキスト的にやや違和感がある。  

そこでは、以下のように対応した  

```diff ts
- v.coerce(
-   v.number(),
-   (input) => Number(input)
- );
+ v.pipe(
+   v.union([v.number(), v.string()]),
+   transfer((input) => Number(input))
+ );
```

確かに、入力される型がわからないなんてことは基本的にないなあ。と思いながら対応した。  

## 所管

すでに膨れた Project で transform などをたくさん使っていたりすると、正直疲れはするが、
かなり読みやすくなった気がする。  

読みやすくなったのは、構文的に読みやすくなったのもあるが、  
真剣にドキュメントを読んだから単純に理解して読めるようになっただけかもしれない。  

結局ドキュメントを読んで手を動かすのが最も早く理解できるのだなと感じた対応でした。
