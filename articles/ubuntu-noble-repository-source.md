---
title: "Ubuntu 24.04 の標準機能で deb822 の deb-src を有効にする"
emoji: "🍦"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["linux", "ubuntu", "python", "apt"]
published: false
---

# TL;DR

deb822 スタイル形式の `deb-src` (ソースコードリポジトリ) を有効するワンライナーです。

```bash:Ubuntu 24.04
sudo apt-get update \
    && sudo DEBIAN_FRONTEND=noninteractive apt-get --no-install-recommends -y install python3-software-properties \
    && sudo /usr/bin/python3 -c "from softwareproperties.SoftwareProperties import SoftwareProperties; SoftwareProperties(deb822=True).enable_source_code_sources()" \
    && sudo apt-get update
```

Ubuntu 24.04 からはパッケージリポジトリの `deb-src` (ソースコードリポジトリ) を有効にするのに `add-apt-repository -s` コマンドが効かなくなってしまったので代替として作りました。

✅ `sed` や `nano` コマンドによる独自の編集手段を使わない
✅ `python3-software-properties` パッケージによる標準機能を使う

# deb822 スタイル形式について

Ubuntu **24.04** LTS "Noble Numbat" が先日リリースされました。

Ubuntu **24.04** ではパッケージリポジトリの [**deb822 スタイル形式** が有効になりました](https://discourse.ubuntu.com/t/spec-apt-deb822-sources-by-default/29333)。 これはユーザー影響としては割と大きい変更だと思います。

Ubuntu **22.04** LTS "Jammy Jellyfish" までは、パッケージリポジトリの定義は `/etc/apt/sources.list` ファイルにありました。

```:Ubuntu 22.04 /etc/apt/sources.list
deb http://archive.ubuntu.com/ubuntu jammy main restricted
deb http://security.ubuntu.com/ubuntu jammy-security main restricted
deb http://archive.ubuntu.com/ubuntu jammy-updates main restricted
```

それが Ubuntu **24.04** ではファイルが `/etc/apt/sources.list.d/ubuntu.sources` なり、内容も以下のように変わりました。

```:Ubuntu 24.04 /etc/apt/sources.list.d/ubuntu.sources
Types: deb
URIs: http://us.archive.ubuntu.com/ubuntu
Suites: noble noble-updates
Components: main restricted

Types: deb
URIs: http://security.ubuntu.com/ubuntu
Suites: noble-security
Components: main restricted
```

セマンティックとしては同じですが、シンタックスが大きく変わっていますね。 これが deb822 スタイル形式です。 パッと見は分かりやすくなっていますね。

deb822 スタイル形式についての日本語の情報は gihyo.jp の記事が分かりやすいです。

https://gihyo.jp/admin/serial/01/ubuntu-recipe/0677#sec4

詳細を知りたい場合は Ubuntu の Man ページを参照してください。

https://manpages.ubuntu.com/manpages/noble/en/man5/sources.list.5.html#deb822-style%20format

# 問題点

まず背景情報として、私は Ubuntu を Python によるプログラミングの用途に使っており、また様々な Python バージョンを利用したいので Ubuntu 付属のパッケージを使わず、自ら **ソースコードからビルド** してインストールしています。

Python をソースコードからビルドするには **ビルド依存関係** が必要です。 ビルド依存関係をインストールするには APT 系コマンド `apt build-dep` を利用します。

しかし `apt build-dep` を利用するには、デフォルトで無効になっているソースコードリポジトリ (`deb-src`) を有効にする必要があります。 Ubuntu **22.04** までは `add-apt-repository` コマンドで有効にすることが可能でした。

```bash:Ubuntu 22.04
$ sudo apt install software-properties-common
$ sudo add-apt-repository -s
```

ここで問題点ですが、Ubuntu **24.04** からは **こちらのコマンドが効かなくなってしまいました**。 正確には、従来のワンラインスタイル形式で定義してあるファイルにはこれまで同様に機能するのですが、deb822 スタイル形式の定義ファイルは対応していないようです。

一応 `add-apt-repository` コマンドのソースコードまで追ったのですが、読む限り deb822 非対応なのが仕様で合っているようです。

https://git.launchpad.net/software-properties/tree/add-apt-repository?id=78d9407f583d1ea0baea619ee5fc82a2df81b8da#n263

(ソースを追った中で見つけた面白い点としては、パッケージリポジトリをリストする `add-apt-repository -L` のみは deb822 が有効になっています。 これはほぼ想像ですが恐らく `add-apt-repository` のコア機能的には deb822 とまだ互換性が取れないが、リスティングのみは問題ないので有効にしている感じでしょうか 🤭)

...

兎にも角にも Python をビルドするのに必要な `deb-src` 有効化ですが、 `add-apt-repository -s` が機能しなくなって有効化できなくなってしまいました。

代替方法としては `sed` コマンドで置換したり、`nano` コマンドで手動編集することです。 置換も手動編集も非常に簡単ではあるのですが、個人的にはあまり好みません。 私は安全で再現性がありかつ機械的にセットアップできる方法を実行したいのです。

# ソフトウェアとアップデート (Software & Updates)

実はここまでは **Ubuntu Server** または **WSL Ubuntu** などの CLI 環境の文脈でこの事象を語っていました。

しかし **Ubuntu Desktop** では `ソフトウェアとアップデート (Software & Updates)` という GUI アプリで `ソースコード (Source code)` というチェックボタンがあるのをふと思い出しました。

![](https://storage.googleapis.com/zenn-user-upload/c93080df63f5-20240507.png)

Ubuntu **24.04** では `add-apt-repository` は機能しないのに、GUI のこのボタンは機能するのか？ 🤔 と思って Desktop の VM を作成して試してみたところ **GUI アプリだと deb822 の deb-src を有効にするのが機能するではないですか！**

つまり GUI が呼び出している機能を部分的に呼び出せれば、CLI では `add-apt-repository` の代替として利用できるということです。

GUI がどのパッケージに該当するのか探したところ `software-properties-gtk` というパッケージでした。 詳しく調べると `software-properties` というのが親のパッケージで、`python3-software-properties` が GUI に対するバックエンドになっているようです。

https://packages.ubuntu.com/source/noble/software-properties

(`add-apt-repository` コマンドが入っている `software-properties-common` パッケージの親でもあるので追跡が比較的容易でした)

幸いなことに全てが Python なので、気合でソースを追ったところ deb822 の `deb-src` を有効にする **該当の関数を発見しました** 🙌🙌

https://git.launchpad.net/software-properties/tree/softwareproperties/SoftwareProperties.py?id=78d9407f583d1ea0baea619ee5fc82a2df81b8da#n404

Python の言葉で話すと `softwareproperties.SoftwareProperties` モジュールの `SoftwareProperties.enable_source_code_sources()` メソッドが該当する Python API です。

REPL で呼び出すならこんな感じです。

```python
>>> from softwareproperties.SoftwareProperties import SoftwareProperties
>>> s = SoftwareProperties(deb822=True)
>>> s.enable_source_code_sources()
```

ちなみに `deb-src` を無効にする `disable_source_code_sources()` メソッドも用意されているようです。

```python
>>> s.disable_source_code_sources()
```

CLI 環境の Ubuntu に戻って `enable_source_code_sources()` メソッドを実行すると、しっかり `deb-scr` を有効にしてくれるのを確認しました。 このメソッドだけで保存までされるので `save_sourceslist()` のようなメソッドを呼び出す必要はなさそうです。 これを活用すればほぼ正式な手順で `add-apt-repository -s` の代替になってくれそうです！

# One-liner

まとめとして、冒頭で紹介した同様のワンライナーです。 必要最小限なパッケージインストールと、上記 Python API を One-line でかつ確認なしで実行します。

`sudo` を付けてあるので、インストール直後の WSL Ubuntu など向けです。

```bash
sudo apt-get update \
    && sudo DEBIAN_FRONTEND=noninteractive apt-get --no-install-recommends -y install python3-software-properties \
    && sudo /usr/bin/python3 -c "from softwareproperties.SoftwareProperties import SoftwareProperties; SoftwareProperties(deb822=True).enable_source_code_sources()" \
    && sudo apt-get update
```

こちらはおまけとして `sudo` を抜いた Docker イメージ `ubuntu:noble` などの root ユーザー向けです。

```bash
apt-get update \
    && DEBIAN_FRONTEND=noninteractive apt-get --no-install-recommends -y install python3-software-properties \
    && /usr/bin/python3 -c "from softwareproperties.SoftwareProperties import SoftwareProperties; SoftwareProperties(deb822=True).enable_source_code_sources()" \
    && apt-get update
```

# まとめ

作成したワンライナーによって、`sed` コマンドなどよりも安全な手段で `deb-src` を有効化できるようになりました。 全て確認なしで実行完了するので、CI 環境での Python ビルドや [autoinstall](https://canonical-subiquity.readthedocs-hosted.com/en/latest/intro-to-autoinstall.html) での Python 開発環境の自動化に役に立ちそうです。

今回はこの事象を解決した所感としては、Ubuntu のパッケージを調べるなどして APT 系パッケージの実装を追う勉強になりました！ GitHub や PyPI などにはホスティングされておらず、ディストロが管理するリポジトリにあることなども知りませんでした。

最後に愚痴ですが、なんで CLI の `add-apt-repository` コマンドと GUI の `software-properties-gtk` で機能が断片化してしまっているのだろう 🙄 と思いました。 ただもしかしたら `add-apt-repository` コマンドも近いうちに deb822 に完全対応するかもしれません。 そしたらこの記事のワンライナーも役目を終えるでしょう！

また今回は `deb-src` を有効にすることに焦点を当てましたが、ワンライナーの Python コマンドを変更すればミラーサーバーの変更にも活用できるかもしれません。

クリーンな環境で実行して動作確認をしていますが、もしワンライナーな機能しないなどがあればお気軽にコメントください 😊 この記事が参考になりましたら、いいねボタンを押して頂けると嬉しいです！
