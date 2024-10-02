---
title: "nvim-lspconfig の config が綺麗にまとまった"
emoji: "🦉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["neovim", "lsp"]
published: true
---

Neovim で楽しく開発をしている。  

nvim-lspconfig を使って、LSP を扱えるようにして、快適にやろうとしている。
https://github.com/neovim/nvim-lspconfig

のだが、なかなか快適になるまでとても苦労する。

## ディレクトリ構成

ざっくり以下のような dotfiles になっている。

```
.config/vim/lua
├── libraries
│   ├── _set_lsp.lua
├── lsp
│   ├── _template.lua
│   ├── cssls.lua
│   ├── default.lua
│   ├── denols.lua
│   ├── gopls.lua
|   ...
├── plugins
│   ├── lspconfig.lua
```

思想としては以下を意識した構成になっている。

### `plugins/`

lazy で install した Neovim Plugin の config を管理している。
基本的に、ひとつの Plugin に対して、ひとつの config を用意するようにしている。  
ファイル検索でヒットしやすいように。

### `libraries/`

使い回すような関数を定義している

### `lsp/`

LSP Server 毎の config を用意している。  
plugins 同様、ひとつの LSP に対して、ひとつの config を原則としている。  
これもファイル検索でヒットしやすいように。

## LSP の Config を書くのが面倒すぎる

元の構成としては、以下のような形にしていた

```lua:.config/vim/lua/plugins/lspcconfig.lua
local servers = {
  "bufls",
  "biome",
  "cssls",
  "denols",
  "gopls",
  ...
}

for _, lsp in ipairs(servers) do
  require(fmt('lsp.%s', lsp))
end
```

使いたい LSP の全ての config ファイルを生成していた。
それが、[nvim-lspconfig の default](https://github.com/neovim/nvim-lspconfig/blob/master/doc/server_configurations.md) のものと全く同じものであっても。

また、ある程度 default からカスタマイズはしているものの、  
ほとんどは、 `on_attach` などを `libraries/` から import しただけの同じ config になっている。

これでは埒があかない。

## カスタマイズした config だけを読み込む

特定の LSP だけ、config をいじりたいが、  
定義していない場合は、default の設定で lsp を読み込む。  

という構成にした。

```lua:.config/vim/lua/plugins/lspcconfig.lua
local utils = require("libraries._set_config")
local conf_lsp = utils.conf_lsp

local default_config = require("lsp.default")


local servers = {
  -- 略
}

for _, lsp in ipairs(servers) do
  conf_lsp(lsp)
end
```

以下は、LSP の初期設定  
カスタムがまだ必要じゃない LSP が呼ぶ config になる
```lua:.config/vim/lua/lsp/default.lua
function default_config(server)
  local lspconfig = require('lspconfig')
  local util = require('libraries._set_lsp')

  lspconfig[server].setup {
    on_attach = util.on_attach,
    capabilities = util.capabilities,
    flags = util.flags,
  }
end

return default_config
```

```lua:.config/vim/lua/libraries/_set_config.lua 
local M = {}

local fmt = string.format

function M.conf_lsp(name)
  local ok, _ = pcall(require, fmt('lsp.%s', name))

  if not ok then
    default_config(name)
  end
end

return M
```

このような構成になった。

これによって、  
例えば `denols` であれば、  
`root_dir = lspconfig.util.root_pattern("deno.json")` などのカスタマイズを行い、 `ts_ls`(旧: tsserver) と共存させるための config なんかも書ける。

## やってよかった

毎回新しい言語を扱う際に、  
`lsp/` 配下に新しいコードを書いて、別の config からコピーしたものを使っていたが、  
これで、新しい言語の習得に感じていた億劫さがひとつ取り除かれた。

コピーするのが面倒すぎて

こんなものまで用意していた。これをコピーして使い回すのである。  
9割内容が同じファイルがいくつも増える状態であった。

https://github.com/yanskun/dotfiles/blob/d028251c96156dd6772819ec96c5f88a448f1de9/.config/vim/lua/lsp/_template.lua

Neovim のファイル構成に悩んでいる方に一助できれば幸いです。

### 成果物

https://github.com/yanskun/dotfiles/blob/d028251c96156dd6772819ec96c5f88a448f1de9/.config/vim/lua/plugins/lspconfig.lua
