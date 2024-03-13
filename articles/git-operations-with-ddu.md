---
title: "ddu.vimでGitの操作を快適にする設定"
emoji: "🚧"
type: "tech"
topics:
  - "vim"
  - "fuzzyfinder"
published: true
published_at: "2024-03-13 00:00"
publication_name: "vim_jp"
---

この記事は[Vim 駅伝](https://vim-jp.org/ekiden/) 2024/03/13の記事です。

# まえがき

本記事の想定読者は以下のような人です

- Fuzzy Finderとして[ddu.vim](https://github.com/Shougo/ddu.vim/)を使用している
- 普段のGit操作をVimから行いたい

また、以下の内容については扱いません。参照先記事をリンクしています。

- ddu.vimの基本的な紹介(ui/source/filter/kindの概念含む)
    - [新世代の UI 作成プラグイン ddu.vim (@shougo)](https://zenn.dev/shougo/articles/ddu-vim-beta)
- ddu.vimを使用するための設定方法
    - [ddu.vimの基本設定概観](https://zenn.dev/vim_jp/articles/c0d75d1f3c7f33)
    - [一歩ずつ始めるddu.vim (@wagomu)](https://zenn.dev/vim_jp/articles/20231020step-by-step-ddu)

また、本記事内の設定についてはLua + TypeScriptで記述しています。

# Git操作用のddu source (+ kind)

## Status: [ddu-source-git_status](https://github.com/kuuote/ddu-source-git_status)

Git statusの結果を表示します。一番使っているSourceです。
Sourceの他にKind, Filterも提供されています。

### デモ

![](https://i.gyazo.com/f6bdf375c2b936b61d9a52e8f0c855c3.gif)

### 設定例

```lua
---@type LazySpec
return {
  "https://github.com/kuuote/ddu-source-git_status",
  dependencies = { "Shougo/ddu.vim" },
  config = function ()
    local name = "git_status"
    local function callback()
      local args = {
        resume = true,
        name = name,
        sources = { { name = name } } 
      }
      vim.fn["ddu#start"](args)
    end
    vim.keymap.set("n", "<Space>gs", callback, {
      remap = false,
      desc = "Start ddu source: " .. name
    })
  end,
}
```

commit, patchについては[gin.vim](https://github.com/lambdalisue/gin.vim)を呼び出すことで実装しています。

```typescript
type GitStatusActionData = {
  status: string;
  path: string;
  worktree: string;
};

export class Config extends BaseConfig {
  override config(args: ConfigArguments): Promise<void> {
    args.contextBuilder.patchGlobal({
      kindOptions: {
        git_status: {
          defaultAction: "open",
          actions: {
            commit: async () => {
              await args.denops.call("Gin commit");
              return ActionFlags.None;
            },
            diff: async (args: ActionArguments<Record<string, unknown>>) => {
              const action = args.items[0].action as GitStatusActionData;
              const path = stdpath.join(action.worktree, action.path);
              await args.denops.call("ddu#start", {
                name: "file:git_diff",
                sources: [
                  {
                    name: "git_diff",
                    options: {
                      path,
                    },
                    params: {
                      ...(u.maybe(args.actionParams, u.isRecord) ?? {}),
                      onlyFile: true,
                    },
                  },
                ],
              });
              return ActionFlags.None;
            },
            // fire GinPatch command to selected items
            // using https://github.com/lambdalisue/gin.vim
            patch: async (args: ActionArguments<Record<never, never>>) => {
              for (const item of args.items) {
                const action = item.action as GitStatusActionData;
                await args.denops.cmd("tabnew");
                await args.denops.cmd("tcd " + action.worktree);
                await args.denops.cmd("GinPatch ++no-head " + action.path);
              }
              return ActionFlags.None;
            },
          },
        },
      },
    });
    return Promise.resolve();
  }
}
```

## Log: [ddu-source-git_log](https://github.com/kyoh86/ddu-source-git_log)

Git logの結果を表示します。ここからcherry-pickやブランチ作成も可能です。

### デモ

![](https://i.gyazo.com/73060d0b3965fccc1ef94af5a4b22eeb.png)

### 設定例

```lua
---@type LazySpec
return {
  "https://github.com/kyoh86/ddu-source-git_log",
  dependencies = { "Shougo/ddu.vim" },
  config = function ()
    local name = "git_log"
    local function callback()
      local args = {
        resume = true,
        name = name,
        sources = { { name = name } } 
      }
      vim.fn["ddu#start"](args)
    end
    vim.keymap.set("n", "<Space>gl", callback, {
      remap = false,
      desc = "Start ddu source: " .. name
    })
  end,
}
```

## Stash: [ddu-source-git_stash](https://github.com/peacock0803sz/ddu-source-git_stash)

拙作Sourceです。stashの一覧を表示し、そこからstash pop, drop操作を提供しています。

### デモ

![](https://user-images.githubusercontent.com/33555487/257060600-74c03fed-f213-420b-89e2-8df806be103a.png)

### 設定例

```lua
---@type LazySpec
return {
  "https://github.com/peacock0803sz/ddu-source-git_stash",
  dependencies = { "Shougo/ddu.vim" },
  config = function ()
    local name = "git_stash"
    local function callback()
      local args = {
        resume = true,
        name = name,
        sources = { { name = name } } 
      }
      vim.fn["ddu#start"](args)
    end
    vim.keymap.set("n", "<Space>gS", callback, {
      remap = false,
      desc = "Start ddu source: " .. name
    })
  end,
}
```

# おわりに

私はこれらと先述のgin.vimを組みあわせて快適なGit操作ライフを実現しています。
ddu.vimを使用してのGit操作を快適にしましょうーー
