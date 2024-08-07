---
title: "GitHub の Organization のプロフィールが編集できるのでやってみた"
emoji: "🐙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["GitHub", "README", "Organization", "Profile"]
published: true
published_at: 2021-12-28
publication_name: "wed_engineering"
---

フロントエンジニアの安田です。

普段は Zero の開発を行なっています。

GitHub で自分のユーザーネームと同じ名前の Repository を作り、そこの `./README.md` を記載すると、プロフィールに表示されるというのはご存知でしょうか？

例えば僕であれば、
この[リポジトリ](https://github.com/yanskun)に記載した内容が、こんな感じで表示されます。

![](/images/github-org-profile/profile.png)

なんとこれが、Organization のプロフィールページでも同様のことができるようになったみたいです！  
[READMEs for organization profiles | GitHub Changelog](https://github.blog/changelog/2021-09-14-readmes-for-organization-profiles/)

なので、早速やってみました。

## やってみた

上記のブログと GitHub の Organization のプロフィールを参考に、  
wedinc 配下に `.github` という Repository を Public で作り、  
`./profile/README.md` を記載しました。

[GitHub](https://github.com/github)  

そうして、プロフィールを表示すると、、、  
ちゃんと表示されました！！

![[https://github.com/wedinc](https://github.com/wedinc)](/images/github-org-profile/org.png)

Private な情報が含まれちゃうので、シークレットモードのスクショです。  

## おわりに

これで、エンジニア採用とかで、GitHub の Organization のページを見せびらかすことができるんじゃないかなって思います。

知らない人から見たら「なんか賑やかで洒落てる」  
知ってる人から見たら「お、こいつらわかってんな」

と一目置かれるんじゃないかなって思って、作ってみました。

工数も安いし、Git 管理なので diff 追えたり、PR ベースで更新できたりと、良いこと盛り沢山なので、是非是非みなさんもやってみてください！
