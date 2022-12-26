---
title: "pip installの-c/--constraintオプション使ってますか?"
emoji: "📦"
type: "tech"
topics:
  - "python"
  - "pip"
published: true
published_at: "2022-03-15 16:31"
---

# pip installの`-c`/`--constraint`オプション使ってますか?

`pip install`の`-c`/`--constraint`オプションがあるけれど、あまり普及していなさそうに見えるので啓蒙として書いてみます

## tl;dr

前提として、venvによる仮想環境作成・アクティベート済み
コマンド `pip install -c constraints.txt`, setup.py / setup.cfgのinstall_requires, extras_requireオプションを使う

## 具体例

### 初回セットアップ時

setup.pyをこのように書いて、
```python
from setuptools import setup

install_requires = [  # ここはrequirements.txtに書き出すのもアリ
    "fastapi",
    "sqlalchemy ~= 1.4.32",
    # その他必要なパッケージを列挙
]
tests_require = [
    "pytest",
    "freezegun",
    # 他、テストに必要なパッケージ
]

setup(
    # nameなど他省略
    install_requires=install_requires
    extras_require={
        "test": tests_require,
        "deploy": "aws-sam-cli",
        "research": "jupyterlab"
    }
)
```
:::message
`~=`などの細かいバージョン指定方法はこちらを参照 [Version Specifiers](https://peps.python.org/pep-0440/#version-specifiers)
:::

これを`pip install -e ."[test,deploy,research]"`として全部を一括インストール
install_requiresをrequirements.txtに書き出した場合、`-r requirements.txt`オプションも追加

`pip freeze --exclude-editable > constraints.txt`として、constraints.txtを書き出す
constraints.txtはこんな感じになるはず(主要なパッケージのみ抜き出し)

:::message
`--exclude-editable` はeditable installしたパッケージを含めないので、このconstraints.txtには自パッケージは入らない
:::

```
aws-sam-cli==1.40.1
fastapi==0.75.0
freezegun==1.2.0
ipykernel==6.9.2
ipython==8.1.1
jupyterlab==3.3.2
pandas-1.4.1
pytest==7.1.0
SQLAlchemy==1.4.32
```

### Cloneしてきて使う時

`pip install -e ."[test,deploy,research]" -c constraints.txt`で必要なパッケージと自分自身がインストールされる(`-e`: editableはお好みでどうぞ)

### 依存パッケージ更新時

venvを作り直し、再度`pip install -e ."[test,deploy,research]"`
`pip freeze --exclude-editable > constraints.txt`でconstraints.txtを更新

## `pip install -c constraints.txt`が何をしているか

`pip install --help`を見ると(抜粋)

```  -c, --constraint <file>     Constrain versions using the given constraints```

なるほどわからんとなったので、[解説記事](https://qiita.com/methane/items/11219ceedb44c0ebcc75#constraints-の使い方)を参照すると

> constraints は、 pip install の -c オプションを使います。 -r オプションと違い、これで指定しただけではライブラリはインストール されません。ただバージョンを制限するだけです。

つまり例えば`pip install -e ."[test,deploy]" -c constraints.txt`とすると、pandasやjupyterlabはインストールされない。ここが`-r`オプションとは違う。

---

今日Poetryによる依存管理がトレンドだが、一切3rd Partyなものを使わずでもここまで管理はできる。
注: venvの管理まではやってくれないのでそこは頑張ること


## 追記 on 2022/7/4
[PEP 621](https://peps.python.org/pep-0621/) が実装されたので、`setup.py`/`setup.cfg`をほとんど書かなくて良いようになった。
thx: @aodag 氏

### PEP 621を踏まえた`pyproject.toml`の例

まずsetup.pyに以下3行を記述

```python
from setuptools import setup

setup()
```

そして、pyproject.tomlをこんな感じで書く

```toml
[build-system]
requires = ["setuptools>=45", "wheel", "setuptools_scm>=6.2"]
build-backend = "setuptools.build_meta"

[tool.setuptools_scm]

[project]
name = "package_name"
authors = [{name="Yoichi Takai", email="peacock@example.com"}]

dynamic = ["version"]
requires-python = ">=3.10"  # 指定したい場合書く
dependencies = [  # ここはrequirements.txtに書き出すのもアリ
    "fastapi",
    "sqlalchemy ~= 1.4.32",
]
urls = {homepage = "https://github.com/peacock0803sz/package_name"}  # privateでもよい

[project.optional-dependencies]
test = ["pytest", "freezegun"]
deploy = ["aws-sam-cli"]
research = ["jupyterlab"]
```