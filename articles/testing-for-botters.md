---
title: "botter のためのテスト入門"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["python", "pytest", "botter", "仮想通貨"]
published: true
---

この記事は「仮想通貨botter Advent Calendar 2024」の 14 日目の記事です。

https://qiita.com/advent-calendar/2024/botter

# テストの意義

みなさん、トレード bot の **テスト** は行っていますか？ 🤔

**バックテスト** ではありません。 **ソフトウェアのテスト** の方です。 いわゆる **ユニットテスト** などと呼ばれるものです。 バックテストは、過去のデータを使って bot の戦略を検証するものです。 一方、ソフトウェアのテストは、bot のコードが正しく動作するかを確認するものです。

バックテストは確かに重要なファクターです。 儲からない、または損をする戦略をデプロイしても意味がありません。 一方で、ソフトウェアテストはトレード bot という性質上あまり重視されていません。 それは基本的に **実弾テスト** を行うことである程度 bot の動作を確認することができるためです。

しかしみなさん、一度は bot のコードにバグがあって悩まされたことはありませんか？

- bot がエラーで落ちて肝心な時に取引ができなかった
- ポジションをオープンはするが、なぜかクローズがされない
- 条件が間違っていてレバレッジ最大まで取引をしてしまった
- ロジックを弄ったらなぜかバグった
- よくみたら実装が間違っているけどなぜか儲かっている (これは悩みではない)

先輩方々のポストでも、バグに関する内容がちらほら見かけることができます 🧐

https://x.com/i_love_profit/status/1371932550881447936

https://x.com/tomui_bitcoin/status/1766022457205653571

ソフトウェアのテストによって、これらの問題を未然に防げる可能性が高くなります。 つまり、**バックテストでは利益を最大化できますが、ソフトウェアテストでは意図しない損失を最小化する** 効果が期待できます。

さらに、**テストを書くことは AI コーディングと非常に相性が良い** です 🤖

AI によるコーディングは近年当たり前になってきていますが、AI で生成したコードが役に立たなかったことはありませんか？ テストと AI を組み合わせることで、以下のような利点があります。

- プログラム (トレード bot) の振る舞いをテストで定義することで、AI がそのテストを満たすようなプログラムを生成する可能性が高くなる
- 間違ったコードを生成した場合でも、テストがあればそれを検知することができる

テストはバグを減らすだけではなく、**AI 時代において開発速度を上げるための手段としても有用です** 🚀

しかし、テストを書くには **テストフレームワーク** や **テスト可能なデザインパターン** などの知識が必要です。 また、bot のテストは **外部 API への依存** が強いため、テストが難しいという問題もあります。 そこで私の [pybotters](https://github.com/pybotters/pybotters) という botter 向けライブラリの開発で得た経験、及び本職エンジニアからの開発経験を元に、botter 向けソフトウェアテストチュートリアルを書いてみました。

それでは「**botter のためのテスト入門**」と題して、トレード bot というドメインにおいてのテストの基本的な考え方や実践方法を紹介していきます！

- **テストの意義のまとめ**:
  - テストコードを書くと意図しない損失を防げる
  - AI コーディングとの相性がよく、慣れれば開発効率も上がる
  - テストを書くにはテストフレームワークやデザインパターンなどの知識が必要

# pytest チュートリアル

Python には `pytest` という素晴らしいテストフレームワークがあります。 `pytest` は Python の標準ライブラリに含まれていないため、別途インストールする必要があります。

```:$
pip install pytest
```

bot のテストコードがない場合は、REST API 経由などで取得された **リアルデータのみでしか bot の振る舞いを検証できません**。 テストを作成することで **意図したテストデータを元に bot の振る舞いを検証することができます** 。 `pytest` はそう言ったテストコードを手軽に書けるようにする為のフレームワークです。

まずは取引戦略とは別に、単純な計算の例でみていきましょう。

1. **あなたは引数として与えられた数値に 2 を足す関数として `inc()` を実装しました**
1. 関数 `inc()` をテストするためにテスト関数 `test_answer()` を実装しました
   - 関数 `inc()` を呼び出し、その結果が期待通りの値であるかを `assert` で検証しています
   - **`3` を与えたら `5` が返ってくることを期待しています**

```python:test_sample.py
def inc(x):
    return x + 1


def test_answer():
    assert inc(3) == 5
```

`pytest` コマンドでテストを実行します。 `pytest` はテストファイル `test_*.py` とテスト関数 `test_*()` を自動的に認識してテストを実行します。

```:$
pytest
```

```python:Output
=========================== test session starts ============================
platform linux -- Python 3.x.y, pytest-8.x.y, pluggy-1.x.y
rootdir: /home/sweet/project
collected 1 item

test_sample.py F                                                     [100%]

================================= FAILURES =================================
_______________________________ test_answer ________________________________

    def test_answer():
>       assert inc(3) == 5
E       assert 4 == 5
E        +  where 4 = inc(3)

test_sample.py:6: AssertionError
========================= short test summary info ==========================
FAILED test_sample.py::test_answer - assert 4 == 5
============================ 1 failed in 0.12s =============================
```

上記のような出力が表示されるはずです。 `5` を期待しているのに `4` が返ってきたのでエラーが発生していることがわかります。 このように、`pytest` はテストが失敗した場合にエラーを表示してくれます。

関数 `inc()` は明らかに間違っていました！🤣 それでは `inc()` を `+ 1` ではなく `+ 2` に修正してみましょう。

```diff:test_sample.py
 def inc(x):
-    return x + 1
+    return x + 2
```

再度テストを実行します。

```:$
pytest
```

```python:Output
=========================== test session starts ============================
platform linux -- Python 3.x.y, pytest-8.x.y, pluggy-1.x.y
rootdir: /home/sweet/project
collected 1 item

test_sample.py .                                                     [100%]

============================ 1 passed in 0.12s =============================
```

ターミナル画面上ではグリーンに表示され、テストがパスしたことが表示されます！ 🎉 これで関数 `inc()` に問題がない確証が得られました。

`pytest` での書き方のポイントは以下の通りです。

1. テストファイルは `test_*.py` という名前で保存します
1. テスト関数は `test_*()` という名前で定義します
1. テスト関数内で構文 `assert` を使って **入力に対して振る舞いが正しいか** テストします

以上が `pytest` の基礎的な使い方です。 詳しい使い方は公式ドキュメントを参照してください。

https://docs.pytest.org/en/stable/

また VS Code や Cursor などのエディタであれば、コマンドではなく拡張機能の UI からテストの実行ができるので便利です (画像は公式サイトからの引用です) 。

![](https://code.visualstudio.com/assets/docs/python/testing/test-explorer.png)

https://code.visualstudio.com/docs/python/testing

さて、 `pytest` のチュートリアルが終わりました。 次は bot のテストコードについて考えていきましょう。

# テスト不可能な bot

一先ず、典型的なトレード bot のコードを単純化した例で見てみましょう。

```python:main.py
def main():
    while True:  # 無限ループ
        position = fetch_position(symbol)  # ポジションを取得する関数
        candle = fetch_candle(symbol)  # ローソク足を取得する関数

        if judge_buy(position, candle):  # 買い条件を満たすか
            execute_buy(symbol)  # 買い注文
        elif judge_sell(position, candle):  # 売り条件を満たすか
            execute_sell(symbol)  # 売り注文

        time.sleep(n)  # 次の足まで待機
```

情報の取得を `fetch_position/candle()` 、売買の判断を `judge_buy/sell()` などと単純化して表現しています。 それらを実装してこの Python ファイルを実行すれば bot 自体は動作して取引をしてくれることでしょう。

**しかし私たちは実装した売買の条件などが正しいのかなど自信が持てません**。 AI に出力してもらったのなら尚更です。 では、先ほど学んだ `pytest` でこのコードをテストするとしましょう ...

```python:test_bot.py
def main():
    while True:
        ...  # 省略

def test_main():
    main()  # どうやって検証するの ?!
```

見事に壁にぶつかりました。 まず bot の `main()` 関数は **無限ループを含んでいるため、テストが終了しません**。

:::message
もしあなたが bot の稼働に AWS Lambda や Google Cloud Functions などのクラウド関数を利用している場合は、スケジューラーが定期的に bot を起動してくれるので無限ループは含んでいないかもしれません。 しかし、まだ後述の問題が残っています。
:::

無限ループ以外にも **外部 API への依存が強い** という問題点があります。 例として `fetch_position()` や `execute_buy()` のような関数の典型的な実装は以下のようになります。

```python:test_bot.py
import requests  # または ccxt, pybotters などのライブラリ

def fetch_position(symbol):
    return requests.get(f"https://api.example.com/position/{symbol}").json()

def execute_buy(symbol):
    return requests.post(f"https://api.example.com/order", json={"symbol": symbol, "size": ...}).json()
```

これには以下の問題があります。

- `fetch_position()` はテスト毎に異なる結果が返ってくる可能性がある
- `execute_buy()` はリアル取引を行ってしまう、また残高によっては成功しない場合がある

戦略をテストしたいのに、取引所やアカウントの状態によってテストが失敗する可能性があるのは困ります。

そのため、ユニットテストにおいては **外部 API への依存を排除する必要があります**。 実際の取引所の API を呼び出すのではなく、それらをモック (フェイク) に置き換える必要があるのです。 しかし上記のコードのように直接 `requests` などのライブラリを埋め込んでしまうと、置き換えることが難しくなってしまいます。

これらの問題を解決するために、**依存性の注入** と **副作用の分離** を行う必要があります。

... 難しそうな専門用語が出てきました 😅。 これらはソフトウェアのデザインパターンです。 しかし我々は bot で利益を上げたいのであって、ソフトウェアのデザインパターンを詳しく学びたいわけではありません。

次のセクションでは、このようなデザインパターンを既に折り込んだ bot のためのテストコードを紹介していきます。

:::message
取引 bot の主な振る舞いである外部 API を排除して、テストの意味があるのか？という疑問があるかもしれません。 モックでは完全に取引所 API の振る舞いを再現できないので、間違ったパラメーターを渡していて実際の API 呼び出しではエラーが発生するケースが考えられます。 しかし私の考えでは、**プログラムコードから正しい外部 API を呼び出す責任を担保するのは、その取引所のクライアント SDK の役割です**。 しかし残念ながら殆どの取引所の SDK は満足にその役割を果たしていません。 現在私はそれを解決するために [pybotters v2](https://github.com/pybotters/pybotters/issues/248) の作業に取り組んでいます 💪
:::

# テスト可能な bot

さて、物事を解決していきましょう。 上記 `main()` 関数内だけで実装していた内容を、以下のように分離しました。

```python:test_bot.py
from collections.abc import Callable
from typing import Protocol


class Strategy(Protocol):
    def fetch(self): ...
    def plan(self, data) -> list: ...
    def apply(self, orders: list): ...


class TradingBot:
    def __init__(self, strategy: Strategy, waiter: Callable[..., object]):
        self._strategy = strategy
        self._waiter = waiter

    def execute(self):
        data = self._strategy.fetch()
        orders = self._strategy.plan(data)
        self._strategy.apply(orders)

    def wait(self):
        self._waiter()


def mainloop(trading_bot: TradingBot):
    while True:
        trading_bot.execute()
        trading_bot.wait()
```

1. `Strategy` クラス
   これは取引戦略を表すクラスです。 `fetch()` で情報を取得し、 `plan()` で取引計画を立て、 `apply()` で取引を実行します。 [`typing.Protocol`](https://docs.python.org/ja/3/library/typing.html#typing.Protocol) を継承するプロトコルクラスです。 抽象的な表現のみを定義しており、実際の取引戦略はこのクラスを継承して実装します。
1. `TradingBot` クラス
   これは取引 bot を表すクラスの実装です。 `__init__()` では **取引戦略** と **待機関数** を受け取ります。 `execute()` で取引戦略を執行し、 `wait()` で待機関数を実行します。 複雑な取引所戦略を執行するにはもう少し複雑な実装が必要ですが、今回は簡単な例としてこのようにしました。
1. `mainloop()` 関数
   先ほどの `main()` 関数とは違って取引 bot を引数として受け取ります。 この関数は無限ループを行い、取引 bot の `execute()` と `wait()` を交互に実行します。


少しコードが長くなりましたが、クラスを使うことでよりテストが可能になるだけでなく、物事がより分かりやすくなっています。

## 無限ループの解決

`mainloop()` 関数はテスト不要なほど単純になっています。 しかし、一応これが動作することを確かめておきましょう。 以下のようなテストコードを追加します。

```python:test_bot.py
import pytest


class NothingStrategy(Strategy):
    def fetch(self):
        return

    def plan(self, data) -> list:
        return []

    def apply(self, orders: list):
        return


class StopTrading(Exception): ...


def stop_trading_waiter():
    raise StopTrading()


def test_mainloop():
    strategy = NothingStrategy()
    trading_bot = TradingBot(strategy, stop_trading_waiter)

    with pytest.raises(StopTrading):
        mainloop(trading_bot)
```

1. `NothingStrategy` クラス
   これは何もしないテスト用の取引戦略です。
1. `stop_trading_waiter()` 関数、 `StopTrading` 例外クラス
   これはテスト用の例外クラスです。 待機関数にて無限ループを抜けるために例外を発生させます。
1. `test_mainloop()` 関数
   これは `mainloop()` と `TradingBot` をテストする関数です。 何もしない取引戦略と、無限ループを抜ける待機関数を持った `TradingBot` を作成して、その取引 bot で `mainloop` を実行します。 `with pytest.raises` にて `StopTrading` 例外が発生することを確認します (例外が発生することが正しい)。

`pytest` コマンドを実行してみましょう！ **テストはパスするはずです**。 これで無限ループを持つ関数をテストすることができました 🎉

もしかすると、これを読んでいる方の中は上記テストコードはあまり意味がないと感じられるかもしれません。 実際にほぼ何もしないコードを実行しています。

しかしこれが意味するのは、**`Strategy` クラスを継承した取引戦略さえ実装さえすれば `TradingBot` クラスが正しくその戦略を執行することが保証される** ということです。 取引戦略自体のテストに集中することができます。

## 取引戦略の実装

次に、取引戦略を実装してみましょう。 ここでは簡単な例として「現在価格が直近ローソク足 n 本の高値安値がブレイクしたら順張りする」という、いわゆる **ドテン君** 戦略を実装してみます。 取引所 API は特定のものではなく架空のエンドポイントを指定しています。

```python:test_strategy .py
from typing import Protocol

import requests


class Strategy(Protocol):
    def fetch(self): ...
    def plan(self, data) -> list: ...
    def apply(self, data: list): ...


class ChannelBreakOutStrategy(Strategy):
    def __init__(self, symobl: str, channel_length: int, order_amount: float):
        self._symbol = symobl
        self._channel_length = channel_length
        self._order_amount = order_amount

    def fetch(self):
        """戦略に必要なデータ (ローソク足、ポジション) を取得する"""

        candle = requests.get(
            "https://api.example.com/api/candle",
            params={"symbol": self._symbol, "interval": "1h"},
        ).json()
        position = requests.get(
            "https://api.example.com/api/position",
            params={"symbol": self._symbol},
        ).json()

        return candle, position

    def plan(self, data) -> list:
        """戦略に基づいて注文を計画する"""

        orders = []

        candle, position = data
        last = candle[-1]["close"]
        high = max([c["high"] for c in candle[-self._channel_length :]])
        low = min([c["low"] for c in candle[-self._channel_length :]])

        # 高値ブレイクアウト、かつロングしてない場合
        if last > high and position["size"] <= 0:
            amount = self._order_amount + position["size"]
            orders.append(
                {
                    "symbol": self._symbol,
                    "side": "buy",
                    "type": "market",
                    "amount": amount,
                }
            )
        # 安値ブレイクアウト、かつショートしてない場合
        elif last < low and position["size"] >= 0:
            amount = self._order_amount + position["size"]
            orders.append(
                {
                    "symbol": self._symbol,
                    "side": "sell",
                    "type": "market",
                    "amount": amount,
                }
            )

        return orders

    def apply(self, orders: list):
        """計画した注文を執行する"""

        for order in orders:
            requests.post("https://api.example.com/api/order", json=order)
```

取引戦略の `Strategy` プロトコルクラスに則って `ChannelBreakOutStrategy` クラスを実装しました。

1. `__init__()` でドテン戦略固有のパラメーターを受け取ります
1. `fetch()` でドテン戦略に必要な、ローソク足とポジションを取得してデータを返します
1. `plan()` でインプットデータからドテン戦略の注文計画を立て、注文リストを返します
1. `apply()` で注文リストを元に取引所に注文を送信します

この 3 段階のメソッドを分割したデザインは、個人的に非常に優れていると感じています。 取引戦略の振る舞いを端的に表現しているし、それぞれのメソッドを単体でテストすることができるからです。

このデザインにより **外部 API への依存** は`fetch()` と `apply()` メソッドのみになりました。 **`plan()` は完全にテストが可能になりました**。 一先ず、次のセクションでは `plan()` のテストコードを書いてみましょう。

## 取引戦略のテスト

```python:test_strategy.py
def test_plan_long():
    """終値がブレイクアウトした場合、ロング注文を計画する"""

    # Arrage
    symobl = "XXXUSD"
    channel_length = 3
    order_amount = 100
    data = (
        # ローソク足
        [
            {"timestamp": 1000, "high": 1000, "low": 90, "close": 100},
            {"timestamp": 2000, "high": 120, "low": 100, "close": 110},
            {"timestamp": 3000, "high": 130, "low": 110, "close": 120},
            {"timestamp": 4000, "high": 130, "low": 120, "close": 130},
            # この "close": 140 がブレイクアウト
            {"timestamp": 5000, "high": 130, "low": 130, "close": 140},
        ],
        # ポジション
        {"size": -100},
    )

    # Act
    strategy = ChannelBreakOutStrategy(
        symobl=symobl, channel_length=channel_length, order_amount=order_amount
    )
    orders = strategy.plan(data)

    # Assert
    assert orders == [
        {"symbol": symobl, "side": "buy", "type": "market", "amount": 200}
    ]
```

1. *Arrange* コメントの箇所にて、テストに必要なデータを準備します。 取引所 API から受け取るであろうローソク足とポジションを模したデータを用意しています。
1. *Act* コメントの箇所にて、テストデータを `ChannelBreakOutStrategy.plan()` に渡して、実行結果である注文リストを取得します。
1. *Assert* コメントの箇所にて、取得した注文リストが期待通りのものであるかを `assert` で検証します。

このテストケースは「**終値 (`timestamp: 5000`) が直近 3 本のローソク足の高値 (130) をブレイクアウトした場合、ロング注文を計画する**」というシナリオをテストしています。 先ほどの無限ループのテストはあまり意味がなかったかもしれませんが、このテストは **取引戦略の振る舞いをテストしている** という点で非常に重要です。

さて、では `pytst` コマンドを実行してみましょう！ あれ？ 思いのほかテストが失敗してしまいました 😱

```python:Output
FAILED test_strategy.py::test_plan_long - AssertionError: assert [{'symbol': 'XXXUSD', 'side': 'buy', 'type': 'market', 'amount': 0}] == [{'symbol': 'XXXUSD', 'side': 'buy', 'type': 'market', 'amount': 200}]
  
  At index 0 diff: {'symbol': 'XXXUSD', 'side': 'buy', 'type': 'market', 'amount': 0} != {'symbol': 'XXXUSD', 'side': 'buy', 'type': 'market', 'amount': 200}
  
  Full diff:
    [
        {
  -         'amount': 200,
  ?                   --
  +         'amount': 0,
            'side': 'buy',
            'symbol': 'XXXUSD',
            'type': 'market',
        },
    ]
```

この出力は、計画された注文の数量が `200` を期待していたのに `0` になってしまっているというエラーです。 数量ゼロというのはあり得ません。 これは `plan()` の実装に問題があるということです。

なので、もう一度実装をよく見直してみましょう。 おそらく `amount` の計算部分が怪しいことは予想できます。 さて、頑張ってバグを見つけましょう ... 🕵️‍♂️ (もちろん、AI に質問してみてもいいでしょう) 。 ... 分かりましたか？ ここです！

```diff:test_strategy.py
         # 高値ブレイクアウト、かつロングしてない場合
         if last > high and position["size"] <= 0:
-            amount = self._order_amount + position["size"]
+            amount = self._order_amount + (-position["size"])
             orders.append(
                 {
```

**残ポジションの演算子が間違っていました**。

今回の例ではショートポジションは負の値であるデータを意図していますが、演算子が逆であるためにドテン数量の計算が間違ってしまいました。 **これがテストの重要性です**。 実際このようなコーディングミスはありえるでしょう。 テストがなく実弾テストだけしている場合、このようなバグを見つけるのは大変時間が掛かります。 今回は数量が 0 になるバグでしたが、逆に数量が意図せず大きくなってしまうというバグも考えられます。 その場合は非常に大きな損失に繋がる可能性があります。

### マトリックステスト

さて、次はショートのテストを追加してみましょう。 `plan()` の中ではロングとショートの if 条件の分岐があります。 もしかしたらまたバグがあるかもしれません。

ショート版の `test_plan_short()` 関数を追加するのもいいですが、ロング版と **インプットと期待値だけ違って他の内容はほぼ同じになるはず** です。 そこで **`pytest.mark.parametrize`** を使って、複数のテストケースを一つの関数で定義してみましょう。

```python:test_strategy.py
@pytest.mark.parametrize(
    "test_input,expected",
    [
        # Case: ロング
        (
            # test_input
            {
                "channel_length": 3,
                "order_amount": 100,
                "data": (
                    [
                        {"timestamp": 1000, "high": 180, "low": 90, "close": 100},
                        {"timestamp": 2000, "high": 120, "low": 100, "close": 110},
                        {"timestamp": 3000, "high": 130, "low": 110, "close": 120},
                        {"timestamp": 4000, "high": 130, "low": 120, "close": 130},
                        {"timestamp": 5000, "high": 130, "low": 130, "close": 140},
                    ],
                    {"size": -100},
                ),
            },
            # expected
            [{"symbol": "XXXUSD", "side": "buy", "type": "market", "amount": 200}],
        ),
        # Case: ショートケース
        (
            # test_input
            {
                "channel_length": 3,
                "order_amount": 50,
                "data": (
                    [
                        {"timestamp": 1000, "high": 180, "low": 90, "close": 100},
                        {"timestamp": 2000, "high": 120, "low": 100, "close": 110},
                        {"timestamp": 3000, "high": 130, "low": 110, "close": 120},
                        {"timestamp": 4000, "high": 130, "low": 120, "close": 130},
                        {"timestamp": 5000, "high": 130, "low": 130, "close": 80},
                    ],
                    {"size": 50},
                ),
            },
            # expected
            [{"symbol": "XXXUSD", "side": "sell", "type": "market", "amount": 100}],
        ),
        # Case: 何もしない
        (
            # test_input
            {
                "channel_length": 3,
                "order_amount": 50,
                "data": (
                    [
                        {"timestamp": 1000, "high": 180, "low": 90, "close": 100},
                        {"timestamp": 2000, "high": 120, "low": 100, "close": 110},
                        {"timestamp": 3000, "high": 130, "low": 110, "close": 120},
                        {"timestamp": 4000, "high": 170, "low": 120, "close": 130},
                        {"timestamp": 5000, "high": 130, "low": 130, "close": 130},
                    ],
                    {"size": 50},
                ),
            },
            # expected
            [],
        ),
    ],
)
def test_plan(test_input, expected):
    # Arrage
    symobl = "XXXUSD"
    channel_length = test_input["channel_length"]
    order_amount = test_input["order_amount"]
    data = test_input["data"]

    # Act
    strategy = ChannelBreakOutStrategy(
        symobl=symobl, channel_length=channel_length, order_amount=order_amount
    )
    orders = strategy.plan(data)

    # Assert
    assert orders == expected
```

ロングに加えて **ショートのケースと、さらに注文を執行しないケースを追加** しました。 デコレーター `@pytest.mark.parametrize()` の第一引数 `"test_input,expected"` テスト関数 `test_plan()` の引数名に対応する特殊な記法です。 第二引数となっているリストのデータの数だけテストケースが生成されます。 このように **マトリックステスト** を行うことで、複数のテストケースを一つの関数で定義することができます。

では `pytest` コマンドを実行してみましょう！

```:Output
test_strategy.py::test_plan[test_input0-expected0] PASSED                [ 33%]
test_strategy.py::test_plan[test_input1-expected1] PASSED                [ 66%]
test_strategy.py::test_plan[test_input2-expected2] PASSED                [100%]

============================== 3 passed in 0.27s ===============================
```

3 つのテストケースが生成され全てパスしました！ バグはないようです 🎉 これで取引戦略の `plan()` メソッドの全てが正しく実装されていることが確認できました。

`@pytest.mark.parametrize()` の詳しい機能については公式ドキュメントを参照してください。

https://docs.pytest.org/en/stable/how-to/parametrize.html

## 外部 API のモック (省略)

最後に、外部 API のモック化する方法を説明 ...する予定でしたが、この記事が長くなってしまったため **詳しい説明は省略します** 😅。 外部 API のモック化は労力がかかるわりに、やはり実利が薄いです。 上記の取引戦略のデザインであれば外部 API 以外の殆どのロジックが動作しています。 テストしてないあとの部分は通信として渡す値レベルの話になってきます。 なので上記 `plan` メソッドのみをテスト実行して、他は省略するというのは理にかなっていると言えるかもしれません。

:::message
興味がある場合のみ ... 。

典型的な方法だと `pytest-mock` などのライブラリを使ってモック化する方法があります。 しかしこれは `patch` を使ってモジュールの動作を直接変更するもので、副作用の影響がはかり知れないしパッチ作業もとても大変です。

そこで `TradingBot` クラスに `Strategy` を渡すデザインにしたように、通信関係のクラスを `Stragegy` に渡すデザインにすると、モック化が可能になります。 例えば以下のような感じです。

```python
from typing import Protocol

import requests

class HTTPClientInterface(Protocol):  # HTTP クライアントのプロトコルクラス
    def get(self, url: str, params: dict) -> dict: ...
    def post(self, url: str, json: dict) -> dict: ...

class SomeStrategy(Strategy):
    def __init__(self, http_client: HTTPClient):
        self._http_client = http_client

    def fetch(self):
        return self._http_client.get("https://api.example.com/api/candle", {"symbol": "XXXUSD"})

class FakeHTTPClient(HTTPClientInterface):  # モック化した HTTP クライアントクラス
    def get(self, url: str, params: dict) -> dict:
        return {"symbol": "XXXUSD", "close": 100}

    def post(self, url: str, json: dict) -> dict:
        return {"status": "ok"}

def test_fetch():
    strategy = SomeStrategy(FakeHTTPClient())
    data = strategy.fetch()
    assert data == {"symbol": "XXXUSD", "close": 100}  # テストの意味 (?)

def main():
    # リアルトレード、本番用の HTTP クライアントを渡す
    strategy = SomeStrategy(requests.Session())
    ...
```

`ccxt` のモックであれば `ccxt.Exchange` クラスを継承したモックを作るのがよさそうです。

```python
import ccxt

class SomeStrategy(Strategy):
    def __init__(self, exchange: ccxt.Exchange):
        self._exchange = exchange

    def fetch(self):
        return self._exchange.fetch_ticker("BTC/USD")

class FakeSomeExchange(ccxt.Exchange):  # モック化した HTTP クライアントクラス
    def fetch_ticker(self, symbol: str) -> dict:
        return {"symbol": "BTC/USD", "close": 100}
```
:::

## 本番コード

最後に、`main()` 関数に戻って本番部分を追加して bot を完成させましょう。

```python:main.py
import time

def real_waiter():
    time.sleep(60 * 60)  # 1 時間待機

def main():
    strategy = ChannelBreakOutStrategy(symobl="XXXUSD", channel_length=3, order_amount=100)
    trading_bot = TradingBot(strategy, real_waiter)

    mainloop(trading_bot)
```

1 時間待機する関数と無限ループが組み込まれているので、これはテストコードによるテストは不可能です。 しかしこれは単純な呼び出しだけであり、分岐条件や計算ロジックなどがある `ChannelBreakOutStrategy` クラスは自信をもってテストされているので、実行しても問題ないでしょう！

# リファクタリング

bot を実装したあと、書き方などが気に入らなかったり、不要なコードある、あるいは処理効率が悪い実装が見つかって修正したくなることがあるかもしれません。 このような作業を **リファクタリング** と言います。 しかし、もしテストがなかった場合、そのようなリファクタリングを行うことは非常に危険です。 なぜなら、リファクタリングを行った後に正しく動作するかどうかを確認する手段がないからです。

テストコードを書いたということは、常に対象のコードを評価が可能なのです。 これは前述したように、**AI コーディングとも相性が良いです**。 例えば、AI が自動でコードを修正した場合、その修正が正しいかどうかをテストコードで確認することができます。

さらに「**テスト駆動開発 (TDD)**」という手法では、**テストコードを先に書いてから本番コードを書くという逆転した開発手法を取ります**。 これについては「ソフトウェアの振る舞い」を明確にするために非常に有効な手法ですが、慣れが必要であり、すぐに取り入れるのは難しいかもしれません。 ただこれも AI との相性が非常に良いです。 テストコードを先に書いて「インプットデータ」と「期待される出力」を明確にすることで、AI がより正確なコードを生成することができるかもしれません 🚀

# まとめ

- テストを書くことによって、取引 bot で意図しない損失を防ぐことができます
- `pytest` を使って `test_` で始まる関数を作成することで、テストコードを書くことができます
- 今回紹介した `TradingBot` と `Strategy` クラスのデザインによって、取引戦略のテストが容易になりました
- `Strategy.plan()` のようなメソッドを定義することで、外部 API を取り除いて重要なロジックのみをテストすることができました
- `pytest.mark.parametrize` を使うことで、複数のテストケースを一つの関数で定義することができます
- 外部 API のモック化はコストが掛かるので、合理的な範囲で行うことが重要です
- リファクタリングや、AI コード生成を行う際には、テストコードがあることで動作の正しさを保証できます

最後までお読みいただき、ありがとうございました！ この記事が少しでもお役に立てれば幸いです 🙇‍♂️

よろしければ X のフォローをお願いします 🐦

https://x.com/MtkN1XBt

他の「botter のためのシリーズ」もぜひご覧ください！

https://zenn.dev/mtkn1/articles/c61e77c1d221aa

https://zenn.dev/mtkn1/articles/a455bb8732e52e

https://zenn.dev/mtkn1/books/deployment-for-botter
