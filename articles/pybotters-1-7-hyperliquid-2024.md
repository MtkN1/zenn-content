---
title: "pybotters の Hyperliquid サポートと 2024 年まとめ"
emoji: "🐍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["python", "OSS", "pybotters"]
published: false
---

# TL;DR

Hello, botters! ホリデーシーズンの真っ只中ですが、皆さんいかがお過ごしでしょうか？

今回は pybotters の **Hyperliquid サポート** と **2024 年のまとめ** をお届けします。

# Hyperliquid

pybotters 1.7 にて _Perpetual DEX_ な取引所 **Hyperliquid** に対応しました 🎉

https://github.com/pybotters/pybotters/releases/tag/v1.7.0

Hyperliquid は高いユーザビリティを提供する Perpetual DEX として注目を集めています。 昨今の国内取引所の状況や信頼性、また海外取引所の規制や国内版移転による影響もあり、Hyperliquid のような DEX への需要が高まっているのかと思います。

そのような流れもあり pybotters としても初の DEX サポートとして Hyperliquid に対応しました！


## 他の SDK との違い

Hyperliquid の公式 SDK として [`hyperliquid-python-sdk`](https://github.com/hyperliquid-dex/hyperliquid-python-sdk) 、また有名なライブラリとして `ccxt` も Hyperliquid をサポートしています。 それらと pybotters の違いは何でしょうか？

pybotters では上記では提供されていない次のユースケースに対応できるようになりました:
- `async` / `await` を利用した非同期処理
- 公式 API の直感的な利用
- DataStore の利用

公式 SDK はリファレンス的な実装ですが、通信のライブラリが `requests` に依存している為、非同期処理が不可能です。 また、`ccxt` は非同期処理に対応していますが、API が共通化されているため Hyperliquid Docs に記載されている API を直感的に利用できません。

さらに pybotters であれば、軽量なデータハンドラである _DataStore_ を利用することで、WebSocket からのデータを高速に処理できます！

## HTTP API

このセクションでは、Hyperliquid の API について簡単にご紹介します。

Hyperliquid API のエンドポイントは現在 2 種類あり、ドキュメント上の名称としては `Info endpoint` と `Exchange endpoint` となっています。

https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/info-endpoint
https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/exchange-endpoint

`Info endpoint` は他の取引所で言う所の _Publich API_ に相当し、`Exchange endpoint` は _Private API_ に相当します。 (ただし `Info endpoint` でも [他のユーザーのポジションを取得](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/info-endpoint/perpetuals#retrieve-users-perpetuals-account-summary) など出来るのが DEX の特徴と言えるでしょう)

`Exchange endpoint` では [オーダーの発注](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/exchange-endpoint#place-an-order) などを行えますが、リクエストに認証情報として秘密鍵での「署名」(`"signature"`) などが必要です。 この署名に関しては Ethereum に関する署名方式となっており、生成方法がかなり面倒です (Python 標準ライブラリにありません)。 **pybotters ではこの署名を自動で生成を行います** 。

つまり、**Hyperliquid のドキュメント上は `"action"`, `"nonce"`, `"signature"` のパラメーターが必須となっていますが、pybotters では `"action"` のみ指定すればよくなります** 。

```python: order における Request Body の例
{
    "action": {
        "type": "order",
        "orders": [
            {
                "a": 4,  # "ETH" in Testnet
                "b": True,  # is_buy
                "p": "3300",  # limit_px
                "s": "0.1",  # size
                "r": False,  # reduce_only
                "t": {"limit": {"tif": "Gtc"}},  # order_type
            }
        ],
        "grouping": "na",
    },
}
```

これにより pybotters でサポートしてる他の取引所同様、_Private API_ を手軽に利用できるようになります 🚀

完全なサンプルコードは pybotters Docs の方にありますので、そちらをご参照ください。

API 認証情報の仕様:
https://pybotters.readthedocs.io/ja/stable/exchanges.html#hyperliquid
オーダー発注のサンプルコード:
https://pybotters.readthedocs.io/ja/stable/examples.html#hyperliquid

サンプルコードを試すには、Hyperliquid でアカウントと _API Wallet_ を作成して秘密鍵を環境変数に入力する必要があります。 まだアカウントがない方は Hyperliquid Docs の _Onboarding_ を参照してください:

https://hyperliquid.gitbook.io/hyperliquid-docs/onboarding/how-to-start-trading

## WebSocket API

Hyperliquid は WebSocket API も提供しています。

https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/websocket

Hyperliquid の WebSocket API はまず _Subscriptions_ と記載されている「データストリームを購読する」機能と、 _Post Requests_ と記載されている「HTTP API と同様の操作」を行う機能に分かれています。

pybotters 1.7 では _Subscriptions_ から、まず一部のデータストリームのみを **DataStore** として利用できるようになりました:

- `l2Book` (オーダーブック)
- `trades` (約定履歴)

以下は現状まだ利用できません:

- 他のデータストリームの DataStore
- _Post Requests_ における自動署名

(2024 年内リリースする都合上、主要なデータのみ対応しました。 未実装の機能は 2025 年のリリースをお待ちください！)

DataStore は新しく追加された `pybotters.HyperliquidDataStore` クラスから利用できます。 WebSocket でオーダーブックを購読するサンプルコードは pybotters Docs の方にありますので、そちらをご参照ください。

https://pybotters.readthedocs.io/ja/stable/examples.html#id1

`HyperliquidDataStore` のリファレンスはこちらから確認できます。

https://pybotters.readthedocs.io/ja/stable/generated/pybotters.HyperliquidDataStore.html

未対応の Issue トラッカーはこちらです:

https://github.com/pybotters/pybotters/issues/352

pybotters 1.7 の完全な変更履歴はこちらです:

https://github.com/pybotters/pybotters/releases/tag/v1.7.0

# pybotters in 2024

最後に pybotters の 2024 年を振り返って締めようと思います。

2024 年 1 月の 1.0 安定版リリースから 1 年が経ち、多くの新機能や修正がリリースされました。 以下は追加された主要な機能です。 もしまだ未チェックの機能があれば、ぜひご利用ください！

- pybotters 1.0 リリース (API の安定化、`fetch` API の追加)
- [WebSocket 手動ハートビート機能](https://pybotters.readthedocs.io/ja/stable/advanced.html#manual-websocket-heartbeat)
- [GMO コインメンテナンス後の再接続機能](https://pybotters.readthedocs.io/ja/stable/examples.html#gmocoinhelper)
- [Bybit WebSocket トレード機能](https://pybotters.readthedocs.io/ja/stable/exchanges.html#id15)
- [Bybit Demo トレード機能](https://pybotters.readthedocs.io/ja/stable/exchanges.html#id15)
- [bitbank の `ACCESS-TIME-WINDOW` に対応](https://pybotters.readthedocs.io/ja/stable/exchanges.html#id4)
- [国内取引所 OKJ のサポート](https://pybotters.readthedocs.io/ja/stable/exchanges.html#okj)
- [国内取引所 BitTrade のサポート](https://pybotters.readthedocs.io/ja/stable/exchanges.html#bittrade)
- [bitFlyer 証拠金 (Collateral) の DataStore](https://pybotters.readthedocs.io/ja/stable/generated/pybotters.bitFlyerDataStore.html#pybotters.bitFlyerDataStore.collateral)
- [Bitget V2 API の DataStore](https://pybotters.readthedocs.io/ja/stable/generated/pybotters.BitgetV2DataStore.html#pybotters.BitgetV2DataStore)
- そして Hyperliquid サポート！

2025 年はまず Hyperliquid の未対応の機能を追加していく予定です。 また、告知のみでまだ設計段階にある [pybotters v2](https://github.com/pybotters/pybotters/issues/248) の進捗もお知らせしていきたいなと思っています。

## Thanks to our Sponsors 🙏

2024 年 1 月のリリース以降、 _GitHub Sponsors_ による支援を受けています。 スター数 ~400 という小規模な OSS プロジェクトにも関わらず、多くの方に支援をいただいております。 頂いた支援は pybotters の開発継続に大変助かっております。 本当にありがとうございます 💖

https://github.com/sponsors/MtkN1

_GitHub Sponsors_ 以外にも、プロジェクトを支援する方法はいくつかあります。 是非ご検討頂けると幸いです 🙇‍♂️

- GitHub リポジトリに Star を付ける
- Discord サーバーから取引所のリファラルを利用する
- コミュニティでバク報告や機能リクエストを行う、質問に回答する
- SNS やブログで pybotters を共有する

https://github.com/pybotters/pybotters

https://discord.com/invite/CxuWSX9U69

# おわりに

今回は pybotters 1.7 のリリースハイライトと 2024 年のまとめになりました 📝

Hyperliquid のサポートにより、DEX にも対応しました。 また、2024 年は多くの新機能が追加され、多くの方に支援をいただきました。 2025 年も引き続き pybotters をよろしくお願いします！

Happy coding and happy trading! 🤖
