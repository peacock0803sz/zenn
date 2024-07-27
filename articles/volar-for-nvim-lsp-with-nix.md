---
title: "Volar (Vueの言語サーバー)をnvim-lspconfigとNixで使うための設定"
emoji: "🥷"
type: "tech"
topics:
  - "neovim"
  - "vue"
  - "nix"
published: true
published_at: "2024-08-05 00:00"
publication_name: "vim_jp"
---

NOTE: この記事は[Vim駅伝](https://vim-jp.org/ekiden/)の2024-08-05の記事です

Neovim (nvim-lspconfig)環境でVueの言語サーバーをNix管理して使うための設定でハマったのでShort Tips的な備忘録です

# TL;DR

`lspconfig.tsserver.setup({})` の引数の `init_options.plugins[]` に `@vue/typescript-plugin` のパスを指定してやる必要がある

# デモ

![](https://i.gyazo.com/4290327305a2d9adb38624c2b2043c10.gif)

# 詳細手順

1. 以下のどちらかでVolar(`vue-language-server`)をインストール
    1. `nix profile install nixpkgs#vue-language-server` を実行
    1. `flake.nix` に `pkgs.vue-language-server` を追加
1. LSPの設定ファイルに以下のように書く(Hybrid or 非Hybridのどちらか)

## Hybrid Modeの場合(`@vue/language-server` は2系が必要)

```lua
local lspconfig = require("lspconfig")
lspconfig.tsserver.setup({
  filetypes = {
    "javascript",
    "javascriptreact",
    "typescript",
    "typescriptreact",
    "vue",
  },
  root_dir = lspconfig.util.root_pattern({ "package.json", "node_modules" }),
  init_options = {
    plugins = {
      {
        name = "@vue/typescript-plugin",
        location = vim.env.HOME .. "/.nix-profile/lib/node_modules/@vue/language-server",
        languages = { "javascript", "typescript", "vue" },
      },
    },
  },
})
lspconfig.volar.setup({})
```

## 非 Hybrid Modeの場合 (`@vue/language-server` の `^2.0.7` が必要)

1. 以下のどちらかでTypeScript SDK(`typescript`)をインストール
    1. `nix profile install nixpkgs#typescript` を実行
    1. `flake.nix` に `pkgs.typescript` を追加
1. 上記の `tsserver` の設定に加えて `lspconfig.volar` の設定箇所を変更する

```lua
lspconfig.volar.setup({
  root_dir = lspconfig.util.root_pattern({ "package.json" }),
  single_file_support = true,
  init_options = {
    typescript = {
      tsdk = vim.env.HOME .. "/.nix-profile/lib/node_modules/typescript/lib",
    },
    vue = {
      hybridMode = false,
    },
  },
})
```

# 解説

- Nix (flakes)でVolarをインストールすると `~/.nix-profile/lib/node_modules/@vue/language-server` にNix Storeへのリンクが作成される
    - Nixパッケージ名は `nixpkgs#vue-language-server`
    - NPMパッケージ名は `@vue/language-server`
    - GitHub(ソース)は `vuejs/language-tools` > `packages/language-server` にある
- Hybrid ModeはVolar側でTypeScriptの言語サーバーを内包しないモード
    - 有効にしている場合はTypeScriptの言語サーバー側に連携する必要がある
    - 無効にしている場合でもTypeScriptまでは抱き抱えていないので、手動でTypeScript SDKのパスを指定する必要がある

# 参考文献

- Configurations > volar (`neovim/nvim-lspconfig/doc/server_configurations.md#volar`): https://github.com/neovim/nvim-lspconfig/blob/master/doc/server_configurations.md#volar
- feat: reintroducing the complete language server (`vuejs/language-tools#4199`): https://github.com/vuejs/language-tools/pull/4119

## 手前味噌 (私のdotfiles)

- nvim-lspconfig設定(`~/.config/nvim/lua/plugins/nvim_lsp.lua#L100-L131`): https://github.com/peacock0803sz/dotfiles/blob/773f823e734e7f49b37942e16f4c283068d38db2/.config/nvim/lua/plugins/nvim_lsp.lua#L100-L131
- Nixでパッケージ指定している箇所(`~/dotfiles/flake.nix`): https://github.com/peacock0803sz/dotfiles/blob/f6f7e7defbe149f9feeb85aecf69e92336637a76/flake.nix
    - `vue-language-server`: #L71
    - `typescript`: #47

# Thanks (宣伝)

vim-jp Slackの `#tech-lsp`, `#tech-nix` などで質問ができます
