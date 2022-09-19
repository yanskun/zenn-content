---
title: "ぬるぬる開閉する Table Row を作った"
emoji: "🪗"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["React", "Material-UI", "mui", "style"]
published: true
---

## やりたいこと

- テーブルの行が、親子関係になっている
- 親の行をクリックしたら、子の行が隠れる

以下を作る
@[codesandbox](https://codesandbox.io/embed/mui-table-with-collapse-vzj0or?fontsize=14&hidenavigation=1&theme=dark)


## Accordion コンポーネントを使って実装する(失敗)

![](https://storage.googleapis.com/zenn-user-upload/3b1faf123bd5-20220919.gif)

これを使えばいけそう。と言うことで作ってみた

@[codesandbox](https://codesandbox.io/embed/epic-bessie-93o98o?fontsize=14&hidenavigation=1&theme=dark)

全然違う！！！！

なんか全然違う！！！

### DOM 構成をみてみてる。

なんじゃこりゃあ
![](https://storage.googleapis.com/zenn-user-upload/3a8b1106c9f6-20220919.png)

原因を理解する

`<tbody>` の直下に `<tr>` がなく、Accordion Component によって生成された `<div>` 要素がレンダーされてしまっていた。

その結果、行ずれが発生してしまっている。

そんな具合。

## Collapse コンポーネントを使って実装する(成功)

![](https://storage.googleapis.com/zenn-user-upload/8ce442a8ce21-20220919.gif)

Material UI の Transitions Utils の Collapse を使ってみる

@[codesandbox](https://codesandbox.io/embed/mui-table-with-collapse-vzj0or?fontsize=14&hidenavigation=1&theme=dark)

これが俺の求めていたものだ！

### 解説

作るに当たって、注意した点

- TableCell にくっついてる top と bottom の padding を無効にした
  - 閉じた時に余白が生まれてしまう
- `display: none;` は使わない。
  - transition がつかないので
- 閉じた後に、TableCell に `border: none;` をつける
  - 閉じた Child の Row の border が重なってしまうので

## 終わりに

Accordion ではなくて、Transition を使ってみようと言う仮説が当たったのが非常良かった。

個人的に `transition: 500ms;` くらいが一番気持ちよくぬるぬる動く感じがするので、好み
