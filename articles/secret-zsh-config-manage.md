---
title: "非公開にしたい zsh config を dotfiles と共存させる"
emoji: "🔑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["zsh", "zshrc", "dotfiles"]
published: true
---

## 前提

- zshrc を GitHub で public に管理している
- 仕事でのみ使うコマンドを alias で登録したい 

## 元々やっていた対策

**別ファイルに置いて、import して .gitignore しておく**

これがかなりシンプルな解決策かなと思います。

```zshrc:.zshrc
. $ZDOTDIR/.zshrc_secret
```

```zshrc:.zshrc_secret
alias access-server=xxxxxxx

DAIJI_KEY=XXXXXXX
```

### 課題

複数案件や、副業、個人開発でも同様のこと発生した。
そういう時に

```zshrc:.zshrc_secret
# 本業の会社
alias access-server=xxxxxxx

DAIJI_KEY=XXXXXXX

# 副業の会社

~~ 略 ~~

# 個人開発の見せちゃいけんもの

~~ 略 ~~

```

みたいな感じか、

ファイルを増やして、

```zshrc:.zshrc
. $ZDOTDIR/.zshrc_hongyo

. $ZDOTDIR/.zshrc_fukugyo

. $ZDOTDIR/.zshrc_kojin
```

と import を増やすかになります。

かなり埒が開かないです。

## 綺麗な対策

```zshrc:.zshrc
if [ -d "$ZDOTDIR/config.d" ]; then
  for conf in "$ZDOTDIR/config.d/"*.zsh; do
    [ -e "$conf" ] || break
    source "${conf}"
  done
fi
```

再起的に見つけて、import するようにしてみました。
これで、ファイルがあればそれを読み込む。
なくても怒られない。状態になりました。

```
zsh
├── .gitignore
├── .zshenv
├── .zshrc
└── config.d
    ├── .gitignore
    ├── README.md
    └── copilot.zsh
```

[dotfiles](https://github.com/yanskun/dotfiles/tree/b25d934a110839939affa1dc2dbb60e13bb4bad7/zsh)

今のフォルダ構成はこんな感じです。

copilot の alias は、かなり長く、かつ発行されたものなので、僕が管理しないということもあり
同様に config.d のなかに入れています。
しかし、これは見せてもいいものなので、

```:zsh/config.d/.gitignore
*.zsh

!copilot.zsh
```

上記のような形で、管理するようにしました。
