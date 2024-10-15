---
title: "tmux の使い捨てターミナル用の popup 表示"
emoji: "🦘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["tmux"]
published: true
---

メインで開いているパネル以外の箇所で、軽くログなどを見たい時に、
tmux の popup を使って表示させることがあります。

## zshrc

元々は

```shell:zsh/.zshrc
function tmuxpopup {
  tmux popup -d '#{pane_current_path}' -h 50%
}
zle -N tmuxpopup
bindkey '^p' tmuxpopup
```

`.zshrc` の中で `bindkye` で表示するようにしていました。

しかし、裏で何かが実行中の時に使えないと言う課題がありました。

私は普段 Neovim で開発をしているので、  
Neovim の pain にフォーカスしてる時に、使いたい時、  
一度他の pain に移動する必要があります。  

また、大抵他の pain は、`npm dev` や `docker compose up` など、  
実行中になっていることがあります。

空いてる pain を探して、移動する必要がありますし、  
なんなら、空いてるからそこで実行しているので、  
**tmuxpopup を使う必要がない** 状態になっていました（悲しすぎる）

## tmux plugin

tmux の key bind に移動すれば、結構うまくいくんでは？  
と思い実行してみました。

:::message
tmux plugin の作り方は、plugin manager の公式が出しているので、解説は省略します。
https://github.com/tmux-plugins/tpm/blob/master/docs/how_to_create_plugin.md
:::

```sh:scripts/create.sh
#!/usr/bin/env bash

tmux popup -d '#{pane_current_path}' -h 50%
```

これでいい感じ。  
かと思いきや `esc` キーで終了すると、終了コード 129 が返ってきてしまうと言う事象がありました。  

`'/Users/yanskun/Projects/github.com/yanskun/tmux-popup/scripts/create.sh' returned 129`  
なんで？

0 以外の終了コードが返ってくると、面倒なので、


```sh:scripts/create.sh
#!/usr/bin/env bash

tmux popup -d '#{pane_current_path}' -h 50%
exit 0
```

と、無理やり終了させるようにしてみました。

丁寧にやるなら「終了コードが 129 ならば」みたいな分岐を入れるべきなのでしょうが、  
返ってきたからなんなんだって感じなので、一旦雑に終了させます。

完成したものがこちら。

https://github.com/yanskun/tmux-popup
