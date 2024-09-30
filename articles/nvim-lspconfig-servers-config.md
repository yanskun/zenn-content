---
title: "nvim-lspconfig の LSP の config のディレクトリ構成が綺麗にまとまった"
emoji: "🦉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["neovim", "lsp"]
published: true
---

Neovim で楽しく開発をしている。  

nvim-lspconfig を使って、LSP を扱えるようにして、快適にやろうとしている。
https://github.com/neovim/nvim-lspconfig

のだが、なかなか快適になるまでとても苦労する。

## 前提

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
│   ├── jsonls.lua
│   ├── lua_ls.lua
│   ├── pylsp.lua
│   ├── ruby_lsp.lua
│   ├── rust_analyzer.lua
│   ├── sqlls.lua
│   ├── svelte.lua
│   ├── tailwindcss.lua
│   ├── tflint.lua
│   ├── ts_ls.lua
│   ├── volar.lua
│   ├── yamlls.lua
│   └── zls.lua
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
  "jsonls",
  "lua_ls",
  "pylsp",
  "ruby_lsp",
  "rust_analyzer",
  "sqlls",
  "svelte",
  "tailwindcss",
  "tflint",
  "ts_ls",
  "volar",
  "yamlls",
  "zls",
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

-- vim config があるファイルパスを拾ってくる
local config_path = vim.fn.stdpath("config")

for _, lsp in ipairs(servers) do
  -- config_path から、lsp のファイルがあるところまでのパスを定義してあげる。
  -- 注意すべきは、拡張子が必要なところ
  local lsp_file = config_path .. "/lua/lsp/" .. lsp .. ".lua"
  if utils.file_exists(lsp_file) then
    conf_lsp(lsp)
  else
    default_config(lsp)
  end
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
  return require(fmt('lsp.%s', name))
end

-- Copilot とかに聞けばすぐに出てくる
function M.file_exists(name)
  local f = io.open(name, 'r')
  if f ~= nil then
    io.close(f)
    return true
  else
    return false
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

```lua:.config/vim/lua/lsp/_template.lua
if vim.fn.exepath('lsp_name') ~= '' then
  local lspconfig = require('lspconfig')
  local util = require('libraries._set_lsp')

  lspconfig.template.setup {
    on_attach = util.on_attach,
    capabilities = util.capabilities,
    flags = util.flags,
  }
else
  vim.notify(
    'lsp install command',
    vim.log.levels.WARN,
    { title = 'servername' }
  )
end
```
https://github.com/yanskun/dotfiles/blob/main/.config/vim/lua/lsp/_template.lua

Neovim のファイル構成に悩んでいる方に一助できれば幸いです。

### 成果物

- https://github.com/yanskun/dotfiles/blob/main/.config/vim/lua/plugins/lspconfig.lua
