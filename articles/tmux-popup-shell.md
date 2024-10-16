---
title: "tmux の使い捨てターミナル用の popup 表示"
emoji: "🦘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["tmux"]
published: true
---

![](/images/tmux-popup-shell/popup.png)

メインで開いているパネル以外の箇所で、軽くログなどを見たい時に、
tmux の popup を使って表示させることがあります。

`.zshrc` で関数を定義していたのですが、  やや不便だったので、
tmux.conf で読み込んだところ、かなり満足のいく状態になりました。

## zshrc での課題

:::message
pain が実行中の時に popup が表示できない
:::

```shell:zsh/.zshrc
function tmuxpopup {
  tmux popup -d '#{pane_current_path}' -h 50%
}
zle -N tmuxpopup
bindkey '^p' tmuxpopup
```

`.zshrc` の中で `bindkye` で表示するようにしていました。

ターミナルのコマンドになるので、  
コンソールが返ってきてない時は実行できないという課題がありました。

私は普段 Neovim で開発をしているので、  
Neovim の pain にフォーカスしてる時に、使いたい時、  
一度他の pain に移動する必要があります。  

また、大抵他の pain は、`npm dev` や `docker compose up` など、  
実行中になっていることがあります。

空いてる pain を探して、移動する必要がありますし、  
なんなら、空いてるからそこで実行しているので、  
**tmuxpopup を使う必要がない** 状態になっていました（悲しすぎる）

## Tmux run-shell

tmux の key bind に移動すれば、結構うまくいくんでは？  
と思い実行してみました。

```:tmux.conf
bind-key P run-shell "tmux popup -d '#{pane_current_path}' -h 50%"
```

これでいい感じ。  
かと思いきや `esc` キーで終了すると、終了コード 129 が返ってきてしまうと言う事象がありました。  

`'tmux popup -d '/Users/yanskun/Projects/github.com/yanskun/dotfiles' -h 50%' returned 129`  
なんで？

0 以外の終了コードが返ってくると、面倒なので、

```:tmux.conf
bind-key P run-shell "tmux popup -d '#{pane_current_path}' -h 50% | exit 0"
```

と、無理やり終了させるようにしてみました。

完成したものがこちら。

https://github.com/yanskun/dotfiles/blob/main/tmux/tmux.conf
