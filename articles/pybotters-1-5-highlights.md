---
title: "pybotters 1.5 ハイライト"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["python", "pybotters", "oss"]
published: false
---

# TL;DR

pybotters 1.5 をリリースしました 🎉

https://github.com/pybotters/pybotters/releases/tag/v1.5.0

今回の更新には以下の取引所サポートが追加されました。

* OKJ (旧 OKCoinJapan) の API (REST/WebSocket) 認証サポート
* BitTrade (旧 Huobi Japan) の API (REST/WebSocket) 認証サポート

この記事では上記二つの取引所で利用可能になった Private API 呼び出しのサンプルコードを簡単に紹介します。

:::message
サンプルコードでは API 認証情報を `os.getenv()` にて環境変数から取得を試みます。 実行する場合は、該当の環境変数を設定するか `default` 引数の箇所に直書きしてください。
:::

:::message
サンプルコードは [Inline script metadata](https://packaging.python.org/ja/latest/specifications/inline-script-metadata/) を定義してあります！ [`uv run`](https://docs.astral.sh/uv/guides/scripts/) や [`pipx run`](https://pipx.pypa.io/stable/examples/#pipx-run-examples) などのツールを利用すると pybotters を含む依存関係を自動でインストールしてサンプルコードを実行してくれます。 例:
```sh-session
$ uv run okj_http.py # or
$ pipx run okj_http.py
```
これらのツールを利用しない場合はファイル上部の `dependencies` に書いてあるライブラリを `pip install` インストールしてください。
:::

# OKJ

API Docs: [https://dev.okcoin.jp/en/]()

OKJ は API 認証情報としてキー、シークレットに加えてパスフレーズの三つが必要な取引所です。 本家 OKX と同じ仕様ですね。

## REST API

残高を取得するサンプルです。 今回の更新によって pybotters によってリクエストの署名が自動で処理されます。
リクエストの方法的にも基本的な形ですね。 パスフレーズが必要な点以外、他の取引所と比べても大きく違う仕様はありませんね。

@[gist](https://gist.github.com/MtkN1/3af5b7fa5ae00d9999d260f83ebb680d?file=okj_http.py)

## WebSocket API

WebSocket でユーザーのスポットアカウント情報を取得するサンプルです。 接続開始時に pybotters によって自動で署名メッセージが送信されます。

今回の pybotters の更新とは直接は無関係ではありますが OKJ には [WebSocket メッセージの Deflate 圧縮仕様](https://dev.okcoin.jp/en/#spot_ws-general) があり、少しややこしいので注意が必要です:

- 受信するメッセージはバイトメッセージになるので `ws_connect()` のハンドラ引数は `hdlr_bytes` を指定する必要があります
- 展開するのに `zlib` でひと手間加える必要があります

@[gist](https://gist.github.com/MtkN1/3af5b7fa5ae00d9999d260f83ebb680d?file=okj_ws.py)

# BitTrade

API Docs: https://api-doc.bittrade.co.jp/

BitTrade の API 認証情報は従来通りよくあるキー、シークレットの二種類タイプですね。

## REST API

残高を取得するサンプルです。 同様に pybotters によってリクエストの署名が自動で処理されます。
これも pybottesr 自体とは関係なく BitTrade 側の仕様ですが、一発で残高を取得できないのは面倒ですね。

@[gist](https://gist.github.com/MtkN1/3af5b7fa5ae00d9999d260f83ebb680d?file=bittrade_http.py)

## WebSocket API

WebSocket でユーザーのアカウント情報を取得するサンプルです。 接続開始時に pybotters によって自動で署名メッセージが送信されます。 こちらは特殊な仕様はないですね。

@[gist](https://gist.github.com/MtkN1/3af5b7fa5ae00d9999d260f83ebb680d?file=bittrade_ws.py)

ただし Public WebSocket の方は [GZIP 圧縮されている](https://api-doc.bittrade.co.jp/#websocket-public) ので OKJ のようにひと手間加える必要があることに注意です。

# おわりに

上記のサンプルコードらは Gist にまとめてあります。 まとめてダウンロードしたい場合は以下を参照してください。

https://gist.github.com/MtkN1/3af5b7fa5ae00d9999d260f83ebb680d

---

DataStore については未実装です。 現状需要があるか分かりませんが、コントリビュートについては歓迎しています。

---

今回はなんとなく Zenn 記事として pybotters リリースハイライトを紹介しました 📝

これまでは [GitHub Releases](https://github.com/pybotters/pybotters/releases) で毎回熱量多めのハイライトを書いていましたが `>1.0` からはリリースノートを自動化したので Pull Request タイトルのみのあっさりした形式になっています。

pybotters の各取引所サンプルコードについてですが、過去はリリースノートにあったり、Docs の [Examples](https://pybotters.readthedocs.io/ja/stable/examples.html) にあったり、今回のように Zenn に投稿してみたりと場所が断片化しており申し訳ございません。 本来は Docs に *CHANGELOG* か *Blog* のようなセクションを作って投稿するのが OSS プロジェクトらしいやり方の気がするのですが、やり方がまとまっていないので実験的に Zenn に投稿しました。 課題として認識しており、情報を一元化しようと[検討しています](https://github.com/pybotters/pybotters/issues/287)。

---

最後まで読んで頂き誠にありがとうございます！ よろしければ以下についてもよろしくお願いします 🙏

- ⭐ [pybotters リポジトリ](https://github.com/pybotters/pybotters) を Star 付けて評価する
- 𝕏 で [@MtkN1XBt](https://x.com/MtkN1XBt) をフォローする
- 💖 [GitHub スポンサー](https://github.com/sponsors/MtkN1) になって支援する
