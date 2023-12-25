---
title: "WebSocket などの JSON を BigQuery に無限に溜め込むサンプル"
emoji: "🌊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ "bigquery", "websocket", "python", "botter", "仮想通貨" ]
published: false
---

本記事は 仮想通貨botter Advent Calendar 2023 25 日目の記事です。

https://qiita.com/advent-calendar/2023/botter

この記事では Google Cloud Pub/Sub BigQuery サブスクリプションを利用して **WebSocket などからの大量の JSON データをリアルタイムで BigQuery に溜め込む** 為のサンプルを紹介します。

この記事を参考にして頂くことで、様々な取引所の WebSocket で配信されるデータ (約定など) を **手っ取り早く** / **なんでも** / **雑に** / **無限に** BigQuery に投入することができます。 WebSocket と題していますが、REST API のレスポンスでも JSON であればなんでも可です。

BigQuery はコストさえ払えば無限のデータレイクとしても使えますし、データをあとで加工する為に未加工データを無料枠で一時保管しておく用途としても利用することが可能です。

## 前置き

こんにちは、まちゅけんです。 アドカレでは既に 16 日目でこちらの Zenn 本を投稿しました。

https://zenn.dev/mtkn1/books/deployment-for-botter

こちらの本の内容としてはデプロイに困っている方に向けてのソリューションとなっているのですが、自分が今年学んだ中で最も書きたい事ベースで出したので、割とニッチでお固めな内容となってしまっていました。

Merry Christmas 🎅🎄 ということで(?)、何故かやたらモチベーションがアップしていることもあり、ちょうど 25 日目の 2 ページ目が空いていました。 25 日目の記事では、botter の方々により広く有用な情報をアウトプットしてみようかと思います。

この記事では冒頭に書いたように **WebSocket などからの大量の JSON データをリアルタイムで BigQuery に溜め込む** 為のサンプルを紹介します。

既存の botter 向けデータレイクの参考記事としては、2022 年のアドカレで Yoshiso さんの AWS にてS3 + Athena を利用するものや、richwomenbtc さんのそれの GCP 版があります。

https://qiita.com/yoshiso/items/dd64adbc3d273dc0e203
https://github.com/richwomanbtc/gcpts

これらは S3 / GCS に時系列データを格安で保存する素晴らしい例です。 しかし DataFrame を作成してからアップロードしているというのもあり、WebSocket のなどように高頻度 (100ms 未満) で配信されるデータを突っ込む様な用途で使おうと思うと、DataFrame 作成の処理時間が配信頻度に対してネックになる場合があります。

その為、今回のソリューションでは **WebSocket から配信された JSON を「JSON 文字列」としてそのままデータレイクに突っ込む** というアプローチを行います。 このアプローチが優れている点としては ...

- JSON を**生のまま突っ込めるので気持ちいい** 😳
- つまり DataFrame に変換しづらい "半構造化データ" な JSON でも雑に投入することができる
- DataFrame を作成せず文字列としてそのまま投入するので、例えば主要銘柄の約定など WebSocket から高頻度で配信されるデータをそのまま投入しやすい

裏を返せば完全に未加工なデータを投入するというデメリットもありますが、今回利用する BigQuery は柔軟な JSON データのクエリなどもあります (後述) 。 ただしデータ投入・クエリの料金的には S3 / GCS のアプローチよりも基本的には高くなります。

## アーキテクチャ

1. **Cloud Pub/Sub BigQuery サブスクリプション**
    - Cloud Pub/Sub は Google Cloud におけるメッセージキューサービスです。
    - BigQuery サブスクリプションを利用することでプログラム側では「BigQuery へのインサート処理」を書く代わりに「Pub/Sub への Publish 処理」を書くだけで手軽にデータを投入することができます。
    - また BigQuery サブスクリプションは Publish したデータ以外に自動で時刻などのメタデータを付与してくれます。
2. Google Cloud **BigQuery**
    - Google Cloud におけるフルマネージド型の分析データ ウェアハウスです。 **AWS では完全な互換サービスがありません** 。
    - フルマネージドなので、コストさえ払えば無限にデータを取り込むことが可能です。
    - **JSON 型カラム** の機能があり、柔軟なデータ取り込み・クエリに対応しています。
3. Google Cloud **Compute Engine**
    - WebScoket からのデータを BigQuery サブスクリプション に Publish するプログラムを実行する為のコンピューティング環境です (AWS における EC2 と同じ)。

## 料金

今回の例を試す前に、Google Cloud の料金形態を理解しておきましょう。

1. Cloud Pub/Sub BigQuery サブスクリプションによるデータの取り込みコストは ...

https://cloud.google.com/pubsub/pricing?hl=ja

> BigQuery サブスクリプションの料金は、すべての Google Cloud リージョンにおいて、サブスクリプションからの読み取り（サブスクライブ スループット）と BigQuery への書き込みに対して TiB あたり$50 です。

つまり 1 GB あたり $0.05 になります。 こちらは無料枠はありません。 配信頻度が非常に高いと思われる Binance USDⓈ-M BTCUSDT の WebSocket 約定 (`trade` ストリーム) を取り込んでいますが、**1 日辺り 4~6 円ぐらい**になっていました。

2. BigQuery のストレージコストは ...

https://cloud.google.com/bigquery/pricing?hl=ja#storage

> アクティブ ストレージ	$0.020 per GB	毎月 10 GB まで無料。

上記 Binance USDⓈ-M BTCUSDT を一か月ほど取り込み、テーブル件数は 25,000,000 件ほどで、テーブルのストレージは 4.09 GB になっていました。 今回は無料枠の範囲内にはなってますが、増え続けた事を考えてコストを最適化したい場合はデータ加工後に削除することを検討しましょう。

3. コンピューティングコストは ...

https://cloud.google.com/free/docs/free-cloud-features?hl=ja#compute

> 1 つの非プリエンプティブル e2-micro VM インスタンス（1 か月あたり）。次の米国リージョンのいずれかで利用できます。

ほぼ最小である e2-micro インスタンスなら 1 つ無料で作成できます。 今回実行するプログラムなら e2-micro インスタンスでも問題なく実行できます。 2 つ目の e2-micro インスタンスを建てたとしても大体 1,000 円以内です。

## 手順

大まかな流れとしてはこのような手順です。

1. BigQuery でデータセット・サブスクリプションの宛先テーブルを作成する
2. Pub/Sub のトピックと BigQuery サブスクリプションを作成する
3. Compute Engine にプログラムをデプロイする

botter の方は AWS しか利用してない方も多いと思うので、Google Cloud アカウントを持っていなければ開設しましょう。 最初は無料枠に加えて 90 日間 $300 分無料トライアルが貰えるのでひとまず上で説明したコストも抑えることができます。

https://cloud.google.com/?hl=ja

### BigQuery

まずは BigQuery の画面を開いて操作を行います。

[https://console.cloud.google.com/bigquery]()

1. データセットを作成します
    ![](https://storage.googleapis.com/zenn-user-upload/1cae094066b0-20231225.png)
    データセット ID を指定します (例: `datalake`) 。
    ロケーションは任意のリージョンを指定してください。
    ![](https://storage.googleapis.com/zenn-user-upload/e92ea62edced-20231225.png)
2. テーブルを作成します
    ![](https://storage.googleapis.com/zenn-user-upload/bad0903e72a0-20231225.png)
    テーブル名を指定します (例: `datalake`) 。
    スキーマで「テキストとして編集」をオンにして以下のテキストを張り付けます。
    ```
    subscription_name:STRING,
    message_id:STRING,
    publish_time:TIMESTAMP,
    data:JSON,
    attributes:JSON
    ```
    ![](https://storage.googleapis.com/zenn-user-upload/685278bff440-20231225.png)

これで BigQuery の準備は完了です！

### Cloud Pub/Sub

次に Cloud Pub/Sub のサブスクリプション画面を開いて操作を行います。

[https://console.cloud.google.com/cloudpubsub/subscription/list]()

1. 新しいサブスクリプション作成します
    ![](https://storage.googleapis.com/zenn-user-upload/6505df0a87df-20231225.png)
2. 同時に新しいトピックを作成します
    サブスクリプション ID を指定します (例: `bigquery-subscription`) 。
    トピック ID を指定します (例: `bigquery-topic`) 。
    ![](https://storage.googleapis.com/zenn-user-upload/8d2e0c027a53-20231225.png)
3. その他のサブスクリプションの設定をします
    BigQuery への書き込みを指定します。
    先ほど作成したデータセットを指定します (例: `datalake`) 。
    先ほど作成したテーブルを指定します (例: `datalake`) 。
    黄色線部分の権限について怒られている `service-******@gcp-sa-pubsub.iam.gserviceaccount.com` のアカウント名をコピーしておいてください。
    ![](https://storage.googleapis.com/zenn-user-upload/38101194d685-20231225.png)
    メタデータを書き込むをチェックします。
    有効期限なしに設定します。
    ![](https://storage.googleapis.com/zenn-user-upload/489178d5bb80-20231225.png)
    ** **作成を押す前に、権限の部分を対応します** **。
4. 新しいタブで IAM の画面を開いて Cloud Pub/Sub が BigQuery に書き込む権限を設定します
    [https://console.cloud.google.com/iam-admin/iam]()
    ![](https://storage.googleapis.com/zenn-user-upload/1f4160692e56-20231225.png)
    新しいプリシンバルに先ほどの黄色線のアカウント名を張り付けます。
    ロールを割り当てるで `BigQuery データ編集者` を割り当てます。
    ![](https://storage.googleapis.com/zenn-user-upload/1b667a6c64e9-20231225.png)
5. Cloud Pub/Sub の画面に戻って作成を押下します
    ![](https://storage.googleapis.com/zenn-user-upload/2122a8bd0c4f-20231225.png)

作成したサブスクリプションの画面が表示されていれば完了です！

### Cloud Shell でプログラムを試す

ここで一旦、プログラムから作成した BigQuery テーブルと Cloud Pub/Sub のサブスクリプションを利用して実際にデータを取り込んでみましょう 🚀

ここでは Cloud Shell で作業してみます。 Google Cloud コンソール右上にあるこのアイコンからシェルを開いてください。

![](https://storage.googleapis.com/zenn-user-upload/55b6f5523737-20231225.png)
![](https://storage.googleapis.com/zenn-user-upload/15fd7f72af4f-20231225.png)

1. 私のサンプルコードをクローンしてディレクトリを移動します
    ```bash
    git clone https://github.com/MtkN1/bigquery-subscription.git \
      && cd bigquery-subscription
    ```
https://github.com/MtkN1/bigquery-subscription
サンプルコードのメインスクリプトは以下のようなコードになっています。 冒頭で紹介したような Binance USDⓈ-M BTCUSDT の WebSocket 約定を BigQuery に投入する処理になっています。
https://github.com/MtkN1/bigquery-subscription/blob/main/src/bigquery_subscription/\_\_main\_\_.py

2. Cloud Pub/Sub トピック ID をコピーします
    ![](https://storage.googleapis.com/zenn-user-upload/97acd7fe11ab-20231225.png)
3. 環境変数 `PUBSUB_TOPIC_ID` を指定します
    サンプルコードでは環境変数 `PUBSUB_TOPIC_ID` から Cloud Pub/Sub トピック ID を読み込むようにしています。 以下コマンドの `{YOUR_PUBSUB_TOPIC_ID}` をコピーしたご自身のトピック ID に置き換えて実行してください。 `.env` ファイルにトピック ID が書き出されます。
    ```bash
    echo PUBSUB_TOPIC_ID={YOUR_PUBSUB_TOPIC_ID} >> .env
    ```
4. サンプルコードを起動する 🚀
    サンプルコードには Dockerfile などが含まれているので、Cloud Shell に内蔵されている Docker コマンドで実行可能です。 以下のコマンドを実行すると BigQuery にデータが取り込まれます。
    ※ Google Cloud の認証ポップアップが表示されたら許可を押してください。
    ```bash
    docker compose up --build
    ```
5. BigQuery のテーブルを確認します
    このような形で `data` カラムに約定の JSON データが格納されているはずです 🎉
    (データが表示されていなければ右上の更新を押してください)
    `publish_time` などのメタデータも自動で付与されています。
    ![](https://storage.googleapis.com/zenn-user-upload/6abb41bb3594-20231225.png)
6. 確認ができたら Ctrl+C でプログラムを終了します

Cloud Shell は一時的な開発環境です。 シェルを閉じたらこのプログラムも停止します。 永続的にデータを投入し続けるには、Compute Engine インスタンスを作成してプログラムをデプロイする必要があります。

### Compute Engine

:::message
**Cloud Shell で幾つかデータが投入されていれば、こちらの手順は今すぐに試す必要はありません**。 デプロイする際の参考にしてください。

ただし Compute Engine の環境構築や接続方法は多岐にわたるので、ここでは重要ポイントのみ説明します。 多くの情報は Google Cloud ドキュメントや Web 上を検索すると見つかるのでそちらを参考にしてください。
:::

1. インスタンスへ SSH 接続する為のメタデータを設定する

https://cloud.google.com/compute/docs/connect/add-ssh-keys?hl=ja#add_ssh_keys_to_instance_metadata

2. 無料枠で VM インスタンスを作成する (任意)
    リージョンを `us-west1` (無料枠の中で日本から地理的に近い)
    マシンタイムを `e2-micro`
    ブートディスクを タイプ: `標準永続ディスク` サイズ: `30GB`
3. アクセススコープを設定する
    Compute Engine から Cloud Pub/Sub にアクセス可能にする為にアクセススコープを「すべての Cloud API に完全アクセス権を許可」に設定します。
4. インスタンスに SSH 接続する
5. プログラムをデプロイする
    上記手順で示した Docker コマンド (インスタンスに Docker のインストールが必要) や、nuhop または tmux コマンドなどでプログラムをデプロイしましょう。

デプロイ方法に関しても多岐にわたりますが、それこそ私のアドカレ 16 日目の記事が参考になるかもしれません。
https://zenn.dev/mtkn1/books/deployment-for-botter

# クエリしてみる

## SQL

取り込んだデータについては BigQuery JSON 型カラムの機能でクエリすることが可能です。

BigQuery のコンソール画面を開いてクエリを試してみましょう。 作成したテーブル (例: `datalake`) から「クエリ」を選択して SQL クエリのタブを開いてください。

![](https://storage.googleapis.com/zenn-user-upload/ac5c83a1f971-20231225.png)

以下の SQL を実行してみましょう。 単純に 1000 件の約定のデータを取得できます。

```sql
SELECT
  *
FROM
  `datalake.datalake`
LIMIT
  1000
```

では次に WHERE 条件を付けて絞り込みを行います。 この条件ではテーブルから **数量が 1.0 枚以上の約定** を絞り込むことができます。

```sql
SELECT
  *
FROM
  `datalake.datalake`
WHERE
  LAX_FLOAT64(data.q) >= 1.0
LIMIT
  1000
```

- `data.q` で JSON 型 `data` カラムの `q` キーを取得しています
- `LAX_FLOAT64` 関数で文字列になっている `q` キーの数量を FLOAT に変換して `>= 1.0` の比較演算子で条件を作っています

下のように数量 1.0 枚以上の約定を絞られているはずです。

![](https://storage.googleapis.com/zenn-user-upload/73b2e97924a9-20231225.png)

このように、プログラム側では JSON 形式の **文字列** を Pub/Sub を通して BigQuery しただけですが、BigQuery の機能で `data.q` のように **JSON を解釈してくれてクエリする** ことができました。 これにより雑に突っ込んだ JSON を柔軟にクエリできることが今回の記事で紹介したかったソリューションとなります。

さらに、以下のようにカラム名を定義することで通常のテーブルのようにデータを平坦化することが可能です。

```sql
SELECT
  JSON_VALUE(data.e) AS e,
  JSON_VALUE(data.E) AS E,
  JSON_VALUE(data.s) AS s,
  JSON_VALUE(data.t) AS t,
  JSON_VALUE(data.p) AS p,
  JSON_VALUE(data.q) AS q,
  JSON_VALUE(data.b) AS b,
  JSON_VALUE(data.a) AS a,
  JSON_VALUE(data.T) AS T,
  JSON_VALUE(data.m) AS m,
  JSON_VALUE(data.M) AS M
FROM
  `datalake.datalake`
WHERE
  LAX_FLOAT64(data.q) > 1.0
LIMIT
  1000
```

![](https://storage.googleapis.com/zenn-user-upload/08eafb35d6e2-20231225.png)

JSON 型カラムの詳しい操作方法については、Google Cloud ドキュメントをご覧ください。

https://cloud.google.com/bigquery/docs/json-data?hl=ja

今回の Binance の約定は 1 行の JSON ドキュメント (オブジェクト) でしうたが、配列になっている JSON は `JSON_QUERY_ARRAY` などを利用して解釈することができます。

## Storage Read API

SQL によるクエリではなく、テーブルのデータをそのままダウンロードして Python や Jupyter Notebook からデータを絞り込み・加工を行いたい場合は、クライアントライブラリの Storage Read API でを利用すると効率的にデータをダウンロードすることができます。 参考までに。

https://cloud.google.com/bigquery/docs/reference/storage/libraries#client-libraries-usage-python

# 参考資料

今回の記事はこちらのドキュメントを嚙み砕いたチュートリアルとなっていました。

https://cloud.google.com/pubsub/docs/bigquery?hl=ja

https://cloud.google.com/pubsub/docs/create-bigquery-subscription?hl=ja

# 複数の取引所のデータを取り込んで区別する

今回の記事ではのサンプルコードでは、1 つの WebSocket チャンネルのみデータを取り込んでいます。 サンプルコードを改修してチャンネルを増やしたり、`producer` 関数をコピーして別の取引所の WebSocket の接続を増やしたりしてみてください。

そうすると作成した 1 つのテーブルに様々なデータが投入されます。 1 つの取引所の複数チャンネルであれば何かしら区別することはできますが、完全に別の取引所のデータを入れると区別ができません。 なのでテーブルを増やして区別したいと思うかもしれませんが、テーブルを増やすと BigQuery サブスクリプションも都度作成する必要があって面倒です。

その際は `attributes` (属性) を利用することをおすすめします。 `consumer` 関数内の Publish するメッセージを作成する `PubsubMessage` クラスの生成時 `attributes` 引数を辞書で指定します。

```py
pubsub_v1.PubsubMessage(data=message.encode(), attributes={"exchange": "binance"})
```

:::message
実際は `producer` 関数内のキューに入れるところで `attributes` に指定するデータを作って渡すのが現実的です。
:::

そうすることで、BigQuery テーブルの `attributes` カラムにデータが記録されます。 これで 1 つのテーブルでどの取引所の WebSocket からのデータなのかを区別することができます。

# おわりに

最後までこちらの記事を読んで頂きありがうございました！

内容についての間違い、バグ、または質問などがあれば X アカウントにお気軽にお知らせください。
もしこの記事が気に入って頂けましたら Zeen や X アカウントのフォロー・いいねをよろしくお願いします 🙇‍♂️🙇‍♂️

https://twitter.com/MtkN1XBt

データを色々溜め込んでよき botters ライフを 👋👋
