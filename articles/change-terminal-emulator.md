---
title: "iTerm2 をやめて Alacritty デビュー"
emoji: "🪵"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["alacritty", "hammerspoon"]
published: true
---

## 話さないこと
- Alacritty について
- Hammerspoon について
- Alacritty と iTerm2 の比較

## Alacritty に興味があった。

一年前に[こんな記事](https://qiita.com/yanskun/items/65338fab35bf3d2ca59a)を書いていた。  
が、僕は以前 iTerm2 を使っていた。  

理由は以下の、iTerm2 で僕が気に入って使っていた機能が備わっていないから。  

- control key のダブルタップによる表示の切り替えができない
- アクティブなスクリーンで開くことができない
- cmd + U で背景透過の切り替えができない

一年経った今思うのは、そんな理由で辞めるのはダサいなあ。と。

## Hammerspoon でゴリゴリやる

Hammerspoon で全てを解決していこうと思う。

### 1. Control key のダブルタップでの表示切り替え

Google で hammerspoon duble tap と検索すると gist なり何なりが出るので、それをコピった

今回は[こちら](https://gist.github.com/asmagill/c38f75fff9d9ef43d1226329fc1436e4)をいただく。

ここの `module.action` という箇所を修正して、Alacritty を呼び出していく

```lua:hammerspoon/alacritty.lua
module.action = function()
  local appName = "alacritty"
  local app = hs.application.find(appName)

  if app:isFrontmost() then
    app:hide()
  else
    hs.application.launchOrFocus(appName)
  end
end
```

これで OK

が、ここで問題である。  
アクティブなスクリーンで開かずに、初回に Alacritty を開いた箇所で再度開いてしまう。  

つまり Ctrl のダブルタップで、強制的に Alacritty のあるスクリーンをアクティブにしてしまうのだ。  
これはよくない。  
なぜなら、僕がターミナルを透けさせている理由は、  
`今見てるものを見える状態でターミナルを開きたい!` だから。

### 2. アクティブなスクリーンで表示する

そのためにまず  
[spaces](https://github.com/asmagill/hs._asm.spaces) というモジュールを使っていく。
公式の通りに、Repository の root にある  `spaces-v0.x.tar.gz` をダウンロード

```bash
cd ~/.hammerspoon
tar -xzf ~/Downloads/spaces-v0.x.tar.gz
```

すると `~/.hammerspoon/hs` というディレクトリが出来上がるので、  
それを呼び出し以下のような Function を作成する。

```lua:hammerspoon/alacritty.lua
local spaces = require("hs.spaces")

function MoveActiveScreen(app)
  local window = app:focusedWindow()

  local focused = spaces.focusedSpace()

  spaces.moveWindowToSpace(window:id(), focused)
  window:focus()
end
```

これは、そのままで、  
Application をみつけ、  
アクティブなスクリーンに移動させ、  
Application にフォーカスしている。  

これを `action` で呼び出す。

```lua:hammerspoon/alacritty.lua
module.action = function()
  local appName = "alacritty"
  local app = hs.application.find(appName)

  if app == nil then
    hs.application.launchOrFocus(appName)
  elseif app:isFrontmost() then
    app:hide()
  else
    MoveActiveScreen(app)
  end
end
```

Application を見つけにいき、  
なければ起動  
一番前にいれば非表示  
それ以外ならアクティブなスクリーンで開く  

これで僕の望んだ動きはできた。

### 3. 透明度を切り替える

hammerspoon だけでは限界があるので、  
Shell Script を書く。  

これもいい記事があった。  
[Tutorial: Key-binding to toggle Alacritty background opacity](https://trunc8.github.io/2021/08/08/tut-toggle-alacritty-opacity)

若干 Alacritty の config の key が古いので、修正

```shell:.config/alacritty/bin/toggle_opacity
#!/usr/bin/env bash

## If alacritty.yml does not exist, raise an alert
[[ ! -f ~/.config/alacritty/alacritty.yml ]] && \
    notify-send "alacritty.yml does not exist" && exit 0

## Fetch opacity from alacritty.yml
opacity=$(awk '$1 == "opacity:" {print $2; exit}' \
    ~/.config/alacritty/alacritty.yml)

## Assign toggle opacity value
case $opacity in
  1)
    toggle_opacity=0.85 # ここはお好み
    ;;
  *)
    toggle_opacity=1
    ;;
esac

## Replace opacity value in alacritty.yml
sed -i -- "s/opacity: $opacity/opacity: $toggle_opacity/" \
    ~/.config/alacritty/alacritty.yml
```

ただ、僕は Symbolic link でやっていて  
なんか sed コマンドがうまく行かなかったので、  
以下のようになった。

```shell:.config/alacritty/bin/toggle_opacity
#!/usr/bin/env bash

## If alacritty.yml does not exist, raise an alert
[[ ! -f $DOTDIR/.config/alacritty/alacritty.yml ]] && \
    notify-send "alacritty.yml does not exist" && exit 0

## Fetch opacity from alacritty.yml
opacity=$(awk '$1 == "opacity:" {print $2; exit}' \
    $DOTDIR/.config/alacritty/alacritty.yml)

## Assign toggle opacity value
case $opacity in
  1)
    toggle_opacity=.85
    ;;
  *)
    toggle_opacity=1
    ;;
esac

## Replace opacity value in alacritty.yml
sed -i "" "s/opacity: $opacity/opacity: $toggle_opacity/g" \
  $DOTDIR/.config/alacritty/alacritty.yml
```

$DOTDIR は dotfile の directory です。  

この shell ファイルを  
dotfiles の `.config/alacritty/bin/toggle_opacity` に配置し  
`chmod +x .config/alacritty/bin/toggle_opacity` で実行可能に  

さらにこの実行ファイルに PATH を通していく  

```shell
export PATH=$PATH:$DOTDIR/.config/alacritty/bin
```

そしてこれを Hammerspoon を使って、実行コマンドを hotkey に登録する。  

```lua:hammerspoon/init.lua 
-- toggle Alacritty opacity
hs.hotkey.bind({ "cmd" }, "u", function()
  hs.execute("toggle_opacity", true)
end)
```

これで OK

## Goodbye iTerm2 👋

```shell
brew uninstall --cask iterm2
```

これで iTerm2 に備わっていて、気に入って使っていた機能を  
Alacritty でも使えるようになった。  

わかったことは、この一年でぐぐり力が上がったということである。  

Alacritty x Tmux x Neovim という何ともモダンな開発環境を手に入れて、  
非常に気分が良い。  

皆さんもぜひ、パクれそうな箇所があればパクって、  
俺かっけーな開発環境を作ってください。  
