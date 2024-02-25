---
title: "[WezTerm] macSKKとskkeletonを使用して同じキーバインドで入力切り替えをしたい!"
emoji: "💀"
type: "tech"
topics:
  - "wezterm"
  - "karabiner"
  - "vim"
  - "skk"
published: true
published_at: "2024-02-26 00:00"
publication_name: "vim_jp"
---

# [WezTerm] macSKKとskkeletonを使用して同じキーバインドで入力切り替えをしたい!

## デモ

[![Image from Gyazo](https://i.gyazo.com/8e6d2610a75643f1490a842e3b4261b3.gif)](https://gyazo.com/8e6d2610a75643f1490a842e3b4261b3)

## 前提・モチベーション

- OS: macOS
- 端末(WezTerm)のVimではOSのIMEとは別に[skkeleton](https://github.com/vim-skk/skkeleton/)を使っている
    - IMEが有効な状態だとVimのノーマルモードとの相性が悪い
    - 端末ではIMEを無効にして、Vimの中でインサートモードの必要な時にIMEを有効にしたい
- 端末 / GUIで同じキーを使ってIMEの入力切り替えをしたい
    - macOSでIMEとして[macSKK](https://github.com/mtgto/macSKK)を使用している
    - Vimではskkeletonを使用している

## 解決方法

- WezTermで `use_ime = false` を設定しIMEを無効にする
    - <https://wezfurlong.org/wezterm/config/lua/config/use_ime.html>
- Karabiner-Elementsを使って端末内の入力切り替えを行う。以下のルールを追加する
    - <https://karabiner-elements.pqrs.org/docs/manual/configuration/configure-complex-modifications/>
- Skkeleton(Vim)の設定でトグルキーを設定(今回は `<C-\>`)

### 追加するルール

```json
{
    "description": "Skkeleton",
    "manipulators": [
        {
            "conditions": [
                {
                    "file_paths": [
                        "/Applications/WezTerm.app/Contents/MacOS/wezterm-gui",
                        "/opt/homebrew/bin/wezterm-gui"
                    ],
                    "type": "frontmost_application_if"
                }
            ],
            "from": {
                "key_code": "spacebar",
                "modifiers": {
                    "mandatory": [
                        "left_control"
                    ],
                    "optional": [
                        "right_control"
                    ]
                }
            },
            "to": [
                {
                    "key_code": "backslash",
                    "modifiers": [
                        "left_control"
                    ],
                    "repeat": true
                }
            ],
            "type": "basic"
        }
    ]
}
```
