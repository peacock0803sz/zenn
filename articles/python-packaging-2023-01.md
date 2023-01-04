---
title: "そのPythonパッケージ管理、venv + pipでよくないですか? in 2023/1"
emoji: "📦"
type: "tech"
topics:
  - "python"
  - "pip"
published: true
---
# そのPythonパッケージ管理、venv + pipでよくないですか? in 2023/1

各所(主にPython界隈の外)で「Pythonパッケージ管理どうするのが正解なの?」とよく耳にするので2023年初頭時点での私の見解を書きます。

:::message
この記事はvenv + pip以外のツール(特に[`poetry`](https://python-poetry.org/), [`pipenv`](https://github.com/pypa/pipenv), [`pip-tools`](https://github.com/jazzband/pip-tools)へのアンチテーゼ要素を多分に含みます。
私はいわゆる「venvでよくないですか? おじさん」をよくやるのでそれのまとめだと思って頂ければと思います。
:::

## tl;dr

多くの場合のPythonパッケージ管理はvenv(virtualenv) + pipで済むのでは?

## 前提

タイトルに`venv`とついていますが、以降venv(virtualenv)については言及しません。仮想環境内で作業している前提で話を進めます。
この記事では基本的に`setup.py` / `setup.cfg`ではなく`pyproject.toml`でのパッケージ定義方法を採用します。これは[PEP 621](https://peps.python.org/pep-0621/)がここ最近で採択・実装されたためです。


なお、想定しているPythonや各ツールのバージョンは以下の通りです。

- Python >= 3.8
- pip >= 23.1
- setuptools >= 45

### 用語

- [PEP](https://peps.python.org): Python Enhancement Proposalsの略。新機能や改善案などが提出され集められたもの
- [Setuptools](https://setuptools.pypa.io/en/latest/index.html): pipが裏で使っているパッケージングライブラリ
- Absolute Import: 絶対インポートとも言われ、パッケージ名からの絶対パスでインポートする書き方

## 基本の構成の `pyproject.toml`

```toml:pyproject.toml
[build-system]
requires = ["setuptools>=45", "wheel"]

[project]
name = "my-awesome-package"
readme = "README.md"
license = {file = "LICENSE"}
authors = [{name = "Peacock", email = "peacock@example.com"}]
urls = {repository = "https://github.com/peacock0803sz/my-awesome-package"}

dynamic = ["version"]
requires-python = ">=3.8"

dependencies = [
  ""  # 依存パッケージをここに書く
]

[project.optional-dependencies]
test = ["pytest"]  # 他テストに必要なもの(freezegunとか)など
dev = ["black", "flake8", "mypy"]

# mypy / blackの設定など
```

この書き方はこの後の4.以外のパターンならばパッケージディレクトリを自動検出してくれます。

## プロジェクト構成 & `pyproject.toml`例いろいろ

`.git` があるRepository Root想定からの構造と`package_dir`オプションの書き方を紹介します。
なお、以下特筆しない限り`pip install --editable`でEditable Installし、自分自身のパッケージ名(`my_awesome_package`)をAbsolute Importができるように書いています。

### 1. `snake_case` のフォルダを直下に置くタイプ

説明不要な気もしますが、`from my_awesome_package import ...`のようにAbsolute Importできます。

```
.
├── LICENSE / README.md etc ...
├── my_awesome_package
│  ├── __init__.py
│  └── main.py
├── tests
│  ├── conftest.py
│  └── test_main.py
├── pyproject.toml
└── setup.cfg
```

### 2. `src/` を一度切ってその下に1つ以上のサブパッケージを置くタイプ

```
.
├── LICENSE / README.md etc ...
├── pyproject.toml
├── setup.cfg
└── src
   ├── core
   │  ├── __init__.py
   │  └── main.py
   ├── misc
   │  ├── __init__.py
   │  └── main.py
   └── tests
      ├── conftest.py
      └── core
         └── test_main.py
```

この場合で自分のパッケージ(`my_awesome_package`)に対してAbsolute Importを行いたい場合は`pyproject.toml`に次の記載が必要です[^1]

```toml:pyproject.toml
 ...
[tool.setuptools.package-dir]
my_awesome_package = "src"
"_" = "src/tests"  # src/tests配下を除外するためのハック
 ...
```

### 3. `snake_case` で1ファイルのみで済ませるタイプ

Webフレームワークのbottleの形です。

```
.
├── LICENSE / README.md etc ...
├── pyproject.toml
├── setup.cfg
├── my_awesome_package.py
└── tests
   ├── conftest.py
   └── test_main.py
```

### 4. `.` 区切りで名前空間を切っていきたいタイプ

一番特殊。Zope/Ploneのパッケージがこれです。
このパターンは`src/my/__init__.py`(Package Rootの`__init__.py`)で少し細工をする必要がありましたが、[PEP 420](https://peps.python.org/pep-0420/)の採択・実装により不要になりました。[^2]
(引き続きAbsolute Importのために`pyproject.toml`には手を加える必要があります。)
まずディレクトリ構成はこんな感じです。

```
.
├── LICENSE / README.md etc ...
├── pyproject.toml
├── setup.cfg
└── src
   ├── my
   │  └── awesome
   │     └── package
   │        ├── __init__.py
   │        └── main.py
   └── tests
      ├── conftest.py
      └── test_main.py
```

基本パターンから`pyproject.toml`にこれを追記します。

```toml:pyproject.toml
 ...
[tool.setuptools.packages.find]
where = ["src"]
exclude = ["tests"]  # testsをsrcの下に置く場合
namespaces = false
 ...
```

こうすることで`from my.awesome.package import ...`とするAbsolute Importができるようになります。

## あなたがやりたいと思っていること、ほぼ出来ますよ

### 開発用にだけインストールするパッケージ指定

`Pipfile`(pipenv)の`[dev-packages]`, poetry(`pyproject.toml`)の`[tool.poetry.group.*.dependencies]`に相当するものがこれです。開発時のみにインストールしたいパッケージなどを書くことができます。
pip installの`-c`オプションと合わせて使うと便利です。[^3]
最初に挙げた例にも書いていますが、このように書きます。

```toml:pyproject.toml
[project.optional-dependencies]
test = ["pytest"]  # 他テストに必要なもの(freezegunとか)など
dev = ["black", "flake8", "mypy"]
```
名前(キー)に使える文字は[PEP 685](https://peps.python.org/pep-0685/)を参照してください。

### タスクランナー(npm scripts)的なもの

`pyproject.toml`に次のように記述することで、タスクランナーのようなこともできます。[^4]
ここで末尾に`[dev]`のように記述することで必要なextras(上述の`project.optional-dependences`)を明記できます。
もちろんサードパーティーパッケージのものを指定しても問題ないです。

```toml:pyproject.toml
[project.scripts]
my_awesome_cmd = "some_package.module:func"
dev_cmd = "some_package.module:dev_func [dev]"  # extrasを指定する場合
```

## 残念ながら出来ないこと

依存パッケージをアンインストールするときに`pyproject.toml` / `constraints.txt`を自動更新したり子や孫の依存も削除するなどの機能はありません。
仕事の開発で使っていると若干つらいですが、あまり使わないので、筆者はこれを理由にpoetryを使うなどは考えませんでした。

## References (参考資料)

全て英語ですが、執筆にあたり参照した記事やページを列挙しておきます。

- [Package Discovery and Namespace Packages](https://setuptools.pypa.io/en/latest/userguide/package_discovery.html)
   - Setuptools公式ドキュメント内のパッケージ検索と名前空間についてのユーザーガイド
- [Declaring project metadata - Python Packaging User Guide](https://packaging.python.org/en/latest/specifications/declaring-project-metadata/)
   - PEP 621の採択・実装をうけて書かれたユーザー向けガイド
- [peacock0803sz/python-template](https://github.com/peacock0803sz/python-template)
   - 拙作のPythonパッケージ用テンプレートリポジトリ(パッケージ本体は書いていない)

[^1]: 残念ながらLSP(pyright / pylance)の読み込み可能モジュールには出てこないようです。もっと良い方法があったらコメント等で教えてください。
[^2]: <https://setuptools.pypa.io/en/latest/userguide/package_discovery.html#legacy-namespace-packages>
[^3]: <https://zenn.dev/peacock0803sz/articles/acd723d5a5fa0b>(拙作参考記事)
[^4]: <https://packaging.python.org/en/latest/specifications/declaring-project-metadata/#entry-points>