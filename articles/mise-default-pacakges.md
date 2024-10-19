---
title: "mise install した際に、一緒にパッケージをインストールする"
emoji: "🤹"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["mise", "versionManager", "dotfiles"]
published: false
---

Node や Ruby, Go などのバージョン管理ツールはたくさんあります。  

Node の例で言えば、
- node_brew
- nvm
- nodenv

(まだありそう)

私が使っているのは `mise` というバージョンマネージャです

https://mise.jdx.dev/

これかなりよくて、平たくいうと、  
たくさんの言語のバージョン管理が一個でできるものです。  

## Version 切り替えの課題

Node や Rust などで、グローバルにインストールして扱うツールがたくさんあります。

ぼくがよく使うものであれば、

https://github.com/antfu-collective/ni

`npm` や `yarn` の打ち間違えを消し去る package です。

このような便利な Node.js のパッケージは、ひとたびインストールしてしまえば、自由に使うことができます。  

ですが、それはそのインストールした時の Node.js のバージョンの時のもの。  
```shell
$ where ni
/Users/yanskun/.local/share/mise/installs/node/20.15.1/bin/ni
```

別のプロダクトに行って、 `.node-version` や `.tool-version` によって、バージョンが切り替われば  
（それを目的としているので、期待通りではあるが）  
再度インストールしなければなりません。  


何をインストールしたのか？  
なんてそんなものいちいち覚えてられません。

そもそも覚えれる人は `npm` と `yarn` を打ち間違えない。

## Default xxx packages

Mise の Document を眺めていたら素敵なことが書かれていました。

> mise-node can automatically install a default set of npm packages right after installing a node version. To enable this feature, provide a $HOME/.default-npm-packages file that lists one package per line, for example:

https://mise.jdx.dev/lang/node.html#default-node-packages

新しいバージョンをインストールする際に、あらかじめ用意したリストに基づいて、  
パッケージもインストールしてくれるよ。  
というものです。

便利すぎる。  

## 使ってみる

私は、この辺りの config file は全て dotfiles で管理しています。

構成としては以下のようにしました

```text
../dotfiles/mise
├── .default-cargo-crates
├── .default-gems
├── .default-go-packages
├── .default-npm-packages
└── .default-python-packages
```

そして、 dotfile の root で管理している shell に以下を追加

https://github.com/yanskun/dotfiles/blob/424b1359a65678f479a1bd7fde46cb76ec49433c/dotfile.sh#L60-L64

これで、ドキュメントの通り、 `${HOME}` 配下に配置

### 例

最後に、私の例を紹介して終わります。

```:.default-cargo-crates
ra_ap_rust-analyzer
stylua
```

```:.default-gems
ruby-lsp
```

```:.default-go-packages
golang.org/x/tools/gopls@latest
github.com/go-delve/delve/cmd/dlv@latest
github.com/cweill/gotests/...@latest
golang.org/x/tools/cmd/goimports@latest
```

```:.default-npm-packages
@antfu/ni
sql-language-server
svelte-language-server
typescript-language-server
@vue/language-server
yaml-language-server
cssmodules-language-server
vscode-langservers-extracted
vscode-json-languageserver
```

```:.default-python-packages
python-lsp-server
```

LSP 周りがかなり多いですね。  
というかほとんどそれのために欲しかったと言っても過言ではない。  

新しいバージョンをインストールするたびに、いくつかのパッケージをインストールするので  
多少時間はかかりますが、ないって言われて怒られるよりずっとマシです。  
それに LSP とかはどうせインストールするものなのでいいのです。  

かなり幸せにになりました。

https://github.com/yanskun/dotfiles/tree/main/mise
