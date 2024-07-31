---
title: "Shell Plugin Manager `Sheldon` と、入れて嬉しい Zsh Plugin"
emoji: "🐚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Zsh", "Sheldon", "shell", "abbreviation"]
published: true
---

Shell に限らず、Plugin Manger は迷います。  
迷った挙句、git-submodule で Plugin を管理していました。

```shell:.zshrc
source ${ZDOTDIR}/submodules/zsh-autosuggestions/zsh-autosuggestions.zsh
```

カッコつけた結果  
意味がわからないなって個人的に思いました。 

## Sheldon

追加するたびに、submodule コマンドを実行して、 `.zshrc` にパスを追加する。  
という作業は二重管理してる感じも流石に辛いということで Plugin Manager を探すことにしました

こういう時、まずは awesome でみます。   
https://github.com/unixorn/awesome-zsh-plugins

ということで、 [sheldon](https://github.com/rossmacarthur/sheldon) にしました。

選定理由は、Rust で書かれていたから。  
早そうですよね、あとかっこいいから人に勧めやすい。

ということで早速移行

```toml:.config/sheldon/plugins.toml
shell = "zsh"

[plugins]

[plugins.zsh-autosuggestions]
github = "zsh-users/zsh-autosuggestions"
```

```diff:.zshrc
- source ${ZDOTDIR}/submodules/zsh-autosuggestions/zsh-autosuggestions.zsh
+ eval "$(sheldon source)"
```

https://github.com/yanskun/dotfiles/commit/d690df1d2b03ab92086f4f6399e0e8d2f293da57
当時のコミット

## Plugin
簡単に Plugin が入れれるようになったので、他の Plugin も入れてみたくなりました。

### zsh-autosuggestions

https://github.com/zsh-users/zsh-autosuggestions

これはすでに入れていたもの

history をベースに入力コマンドのサジェストをしてくれる


### zsh-syntax-highlighting

https://github.com/zsh-users/zsh-syntax-highlighting

入力したコマンドにシンタックスハイライトをつけてくれる

有効な時は青く光って、無効な入力時には赤くなる。

typo が事前にわかるので嬉しい

### zsh-abbr

https://github.com/olets/zsh-abbr

この記事で紹介したい目玉

alias をより綺麗に設定できる。

#### 何が嬉しい

以下のような設定をする

```
abbr dc='docker compose'
```

プロンプトに入力する時は

```shell
$ dc
```

こうなるわけだが、  
コマンド実行時には

```shell
$ docker compose
```

と Alias の中身を展開してから実行してくれる。

すると何が嬉しいかというと、  
history に元のコマンドが貯まるようになる

僕は元々、入力と実行結果を見比べて、  
結局何入力したのかわかんなくなる。という点で alias をあんまり設定していなかったのだが、  

実行時に alias の中身を展開してくれるのであれば、そこはクリアされる。

前述した [zsh-autosuggestions](#zsh-autosuggestions) と相性がとても良い。

suggest されるコマンドが、正しい形になっている。


#### history が汚れないメリット

history を使って、コマンドを実行することはザラにある  

私は、 fzf を使って history を検索したりして、  
~~よくわからない、覚える気にもなれない~~ サーバーとの接続コマンドなんかを叩いている。

https://github.com/junegunn/fzf

こんな感じ

```shell:.zshrc
function fzf-history-selection() {
    BUFFER=`history -n 1 | tac  | awk '!a[$0]++' | fzf-tmux -p --reverse --height 40%`
    CURSOR=$#BUFFER
    zle reset-prompt
}

zle -N fzf-history-selection
bindkey '^H' fzf-history-selection
```

`fzf-tmux` をすると、tmux のpopup を使って、fzf を表示することができる。  
tmux が起動してなければ、普通に fzf として走る優れもの。（地味に嬉しい）  

これで、history から実行コマンドを探すときに、  
元のコマンドで探すことができるようにもしています。  


ただし  
**alias したコマンドを history から元の実行コマンドで検索して実行することはないです。**  
だって、それしないための alias だし。

#### 気分がとても良い

zsh-abbr は(僕の例では) `dc` って入力したら `docker compose` と展開してから実行してくれます。

`dc` って入力すると、 `docker compose` と入力されるのです。  
alias と違い、プロンプトに表示されるんです。

僕はちょっとしかタイプしてないのに、たくさん入力してくれるんです。  

気分が良いですね。

## 最後に

sheldon を入れる前は、そもそも zsh plugin を入れるのが面倒くさくて、そもそも探しもしてなかったのですが、  
設定も楽だし、多分早いし、で嬉しいです。  

また、plugin を探すようになったおかげで、いろんな素晴らしい plugin と出会うことができました。  

他にも探してみたり、なければ作ってみたりもしたいので、    
たくさんターミナルを触らなきゃなと思います。

## Link

- https://github.com/yanskun/dotfiles/blob/main/.config/sheldon/plugins.toml
- https://github.com/yanskun/dotfiles/blob/main/zsh/.zshrc
- https://github.com/yanskun/dotfiles/blob/main/.config/zsh-abbr/user-abbreviations
