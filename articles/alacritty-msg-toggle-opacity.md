---
title: "Alacritty の CLI Command が便利すぎた"
emoji: "🪟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['Alacritty', 'Hammerspoon']
published: false
---

```sh
$ alacritty msg config -h
Update the Alacritty configuration

Usage: alacritty msg config [OPTIONS] <CONFIG_OPTIONS>...

Arguments:
  <CONFIG_OPTIONS>...  Configuration file options [example: 'cursor.style="Beam"']

Options:
  -w, --window-id <WINDOW_ID>  Window ID for the new config [env: ALACRITTY_WINDOW_ID=4594901664]
  -r, --reset                  Clear all runtime configuration changes
  -h, --help                   Print help (see more with '--help')
```

これが便利だったという話

## きっかけ

2年ほど前から iTerm2 ではなく [Alacritty](https://alacritty.org/index.html) を使っている

https://zenn.dev/yanskun/articles/change-terminal-emulator

過去の記事でも触れているが、  
iTerm2 で透明度を切り替えるための Shell Script を書いた。

参考記事
https://trunc8.github.io/2021/08/08/tut-toggle-alacritty-opacity

## 課題

便利だったのだが、
これだと、元の config file をいじるため、  
dotfile をいじっている際に、透明度を変更すると、git に差分が生まれてしまい、  
それを避けて、commit をしたりしなければならなかった。

## alacritty command との出会い

全集中モードで開発する際に、 Zen Mode を使う。
普段 Neovim で開発しているため、以下のプラグインを使って、Zen Mode に入っている。
https://github.com/folke/zen-mode.nvim

Zen Mode に入ると、Terminal の文字サイズも変更してくれ、とても便利である。  
最近 Terminal Emulator を変更しようと思い、 zen-nvim が対応している Terminal Emulator の一覧と、  
その Plugin はどうやって実装されているのかをたまたま覗いてみていた。  

そしたら

https://github.com/folke/zen-mode.nvim/blob/29b292bdc58b76a6c8f294c961a8bf92c5a6ebd6/lua/zen-mode/plugins.lua#L51-L64

こんなことをしていた。

`alacritty msg config -w $ALACRITTY_WINDOW_ID font.size=16` でサイズを変更し  
`alacritty msg config -w $ALACRITTY_WINDOW_ID --reset` で戻す  
という感じでうまいこと切り替えを行っているのである。  

特に  
`--reset` option で元に戻すことができることが素晴らしい  
元のサイズを指定するのではなく、config で指定していたものに戻せるところが嬉しい。

とても便利

## Hammerspoon のコード

`window.opacity` を変えてあげれば、今やりたいことができる。

```lua:hammerspoon/init.lua
-- toggle Alacritty opacity
local transparent = true
hs.hotkey.bind({ "cmd" }, "u", function()
	local appName = "alacritty"
	local app = hs.application.find(appName)
	if app.isFrontmost(app) then
		if transparent then
			hs.execute("alacritty msg config window.opacity=1", true)
			transparent = false
		else
			hs.execute("alacritty msg config --reset", true)
			transparent = true
		end
	end
end)
```

こんな感じにしてみた。

私は、tmux を使っているため、Terminal を同時に複数開くことがないため、 `$ALACRITTY_WINDOW_ID` での window 指定は不要だった。  
（Hammerspoon からうまく扱えなかった）

### (小技) 特定のアプリがアクティブな時のみ実行する Hammerspoon

```lua
local appName = "alacritty"
local app = hs.application.find(appName)
if app.isFrontmost(app) then
-- 略
end
```

これで、Alacritty がアクティブな時のみ実行するようにした。

## まとめ

上記によって、  
opacity を切り替えるための実行ファイルも不要になったし、  
git diff で差分表示されるようなこともなく、より気軽に dotfiles の更新ができるようになった。  

やはり気になったらコードを読んでみることは大事である。思わぬ発見がある。  
