---
title: "Gatling で複数ユーザ認証した情報を元に負荷テストする"
emoji: "🔫"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["gatling", "scala", "test", "web"]
published: true
---

# はじめに

今までは [JMeter](https://jmeter.apache.org/) でしか負荷テストを行ったことなかったのですが、最近 PlayFab で CloudFunction の負荷テストを行う際に [Gatling](https://gatling.io/) を初めて利用しました。

今回の負荷テストでは、各ユーザ毎のレートリミットの制限等も考慮した実利用時を想定した形で行うことが要求されたため、単一ユーザの認証情報を使い回すことは望ましくないと考えました。そこで、複数の認証済みユーザの情報を元に [PlayFab の CloudFunction](https://docs.microsoft.com/ja-jp/gaming/playfab/features/automation/cloudscript-af/) の負荷テストを実施したのですが、若干実装に苦戦したため手順について記事として残しておくことにしました。

また、本記事では Gatling のセットアップから記載していますが、該当コードやその説明を早く見たいという方は [`複数ユーザ認証を行うテストシナリオを実装する`](#複数ユーザ認証を行うテストシナリオを実装する) 項目をご参照ください。

# 動作環境

- macOS Big Sur
- Java OpenJDK 12.0.1
  - 未インストールの方は事前に [公式サイトから](https://jdk.java.net/) OpenJDK をインストールしてください

# Gatling の環境を整える

**Gatling には 2 種類のセットアップ方法が用意されています。** スタンドアローンなツールを直接公式サイトからダウンロードするか、[Maven](https://maven.apache.org/) や [sbt](https://www.scala-sbt.org/) といったツール経由でダウンロードするか選択できます。

どちらの方法でセットアップするかについてですが、**新規でテストケースを Gatling で書いていく用途だと前者になり、既存のプロジェクトに Gatling を取り込む用途だと後者になるかと存じます。**

本記事では、前者のスタンドアローンなツールを直接公式サイトからダウンロードする方法で Gatling の環境をセットアップします。

## 公式サイトから Gatling をダウンロードする

[Gatling のトップページ](https://gatling.io/open-source/) に遷移して、ページを `2 Ways to use Gatling` の項目までスクロールした後、ダウンロードボタンをクリックします。

![スクリーンショット 2021-03-14 21.57.35.png](https://i.gyazo.com/9d98802bee6499a2fcc4432fa27553d2.png)
**`DOWNLOAD GATLING'S BUNDLE` にある `DOWNLOAD NOW` ボタンをクリックする**

ファイルダウンロード後はダウンロードした zip ファイルを適当なフォルダに展開して配置します。
早速ターミナルで展開したフォルダ内にある `./bin/gatling.sh` を実行して、正常にコマンドが実行できるか確認してみます。

::: message info

OS が Windows の場合は `./bin/gatling.bat` を実行します。

:::

```bash
⊨ ./bin/gatling.sh                   ~/D/gatling-charts-highcharts-bundle-3.5.1
GATLING_HOME is set to /Users/nika/Desktop/gatling-charts-highcharts-bundle-3.5.1
Choose a simulation number:
     [0] computerdatabase.BasicSimulation
     [1] computerdatabase.advanced.AdvancedSimulationStep01
     [2] computerdatabase.advanced.AdvancedSimulationStep02
     [3] computerdatabase.advanced.AdvancedSimulationStep03
     [4] computerdatabase.advanced.AdvancedSimulationStep04
     [5] computerdatabase.advanced.AdvancedSimulationStep05
0 # 実行したいテストの番号を入力する、今回は適当に 0 を指定
Select run description (optional)
# 実行するテストに関する説明文を入力する。何も入力する内容が無い or
# 説明文の入力が完了したら Enter を入力して、実際にテストを実行する

# 選択したテストの実行が開始する
# (0 を入力したので computerdatabase.BasicSimulation が実行される)
Simulation computerdatabase.BasicSimulation started...

#...

Simulation computerdatabase.BasicSimulation completed in 26 seconds
Parsing log file(s)...
Parsing log file(s) done
Generating reports...

# テストの実行が無事完了すると、結果が表示されレポートファイルが生成される。
# レポート生成先のファイルパスは実行結果の末尾に表示される。
# レポートファイルは HTML で生成されるため、適当なブラウザで開くことで内容を確認することが出来る。

================================================================================
---- Global Information --------------------------------------------------------
> request count                                         13 (OK=13     KO=0     )
> min response time                                    230 (OK=230    KO=-     )
> max response time                                    483 (OK=483    KO=-     )
> mean response time                                   324 (OK=324    KO=-     )
> std deviation                                         98 (OK=98     KO=-     )
> response time 50th percentile                        259 (OK=259    KO=-     )
> response time 75th percentile                        415 (OK=415    KO=-     )
> response time 95th percentile                        476 (OK=476    KO=-     )
> response time 99th percentile                        482 (OK=482    KO=-     )
> mean requests/sec                                  0.481 (OK=0.481  KO=-     )
---- Response Time Distribution ------------------------------------------------
> t < 800 ms                                            13 (100%)
> 800 ms < t < 1200 ms                                   0 (  0%)
> t > 1200 ms                                            0 (  0%)
> failed                                                 0 (  0%)
================================================================================

Reports generated in 0s.
Please open the following file: /Users/nika/Desktop/gatling-charts-highcharts-bundle-3.5.1/results/basicsimulation-20210314133324259/index.html
```

`./bin/gatling.sh` を実行した後、上記のような出力が確認できれば、問題なくスタンドアローン版の Gatling 環境のセットアップが完了しています。

また、**負荷テストのレポートを確認したい場合は、出力結果にある `Please open the following file: <レポートのファイルパス>` に記載されているファイルをブラウザで開きます。**
html 拡張子を開くアプリのデフォルトが何らかのブラウザになっているのであれば、ターミナルから `open <レポートのファイルパス>` を実行するのでも構いません。

# Gatling でテストシナリオを実装する

**Gatling のテストシナリオを書く場所は `./user-files/simulations` フォルダ内になります。** テストシナリオを Scala で書いた後、ファイルを `./user-files/simulations` フォルダに配置します。すると、`./bin/gatling.sh` を実行した際の実行するテストシナリオのリストに出てくるようになります。

## 簡単なテストシナリオを試しに書いてみる

まずは試しに私のブログに対してのアクセス負荷を計測するためのテストを実装していきます。`./user-files/simulations` フォルダ内に `nikaera.com` フォルダを作成し、`AccessSimulation.scala` という負荷テストのシナリオファイルを作成します。

```scala:./user-files/simulations/nikaera.com/AccessSimulation.scala
package com.nikaera

import scala.concurrent.duration._

import io.gatling.core.Predef._
import io.gatling.http.Predef._

import scala.collection.mutable.ListBuffer
import io.gatling.core.structure.PopulationBuilder

// 1. Simulation クラスを継承してテストシナリオを実行するクラスを定義する
class AccessSimulation extends Simulation {

  // 2. HTTP アクセスする際の設定値を入力する。
  // 今回はアクセス先のベース URL を定義するための baseUrl のみ指定
  val httpConf = http
    .baseUrl("https://nikaera.com")

  // 3. Scala の ListBuffer を用いて複数シナリオを格納する scenarios 変数を用意する
  val scenarios = new ListBuffer[PopulationBuilder]()

  // 4. httpConf で設定した情報を元に / (https://nikaera.com) 及び
  // /tech/ (https://nikaera.com/tech/) へ同時 10 アクセスするのを、
  // 5秒毎に 3回実行するシナリオを作成して、scenarios 変数に格納する
  val pollingApiScn = scenario("Polling Simulation")
    .exec(
      http("Top Page")
        .get("/")
        .check(status.is(200))
    )
    .exec(
      http("Tech Page")
        .get("/tech/")
        .check(status.is(200))
    )
  scenarios += pollingApiScn
    .inject(
      atOnceUsers(10),
      nothingFor(5 seconds),
      atOnceUsers(10),
      nothingFor(5 seconds),
      atOnceUsers(10)
    )
    .protocols(httpConf);

  // 5. 4. で定義したシナリオを実行して https://nikaera.com のアクセス負荷を計測する
  setUp(
    scenarios(0)
  )
}

```

再度 `./bin/gatling.sh` を実行して、実際に負荷テストを行ってみます。

```bash
⊨ ./bin/gatling.sh                   ~/D/gatling-charts-highcharts-bundle-3.5.1
GATLING_HOME is set to /Users/nika/Desktop/gatling-charts-highcharts-bundle-3.5.1
Choose a simulation number:
     [0] com.nikaera.AccessSimulation
     [1] computerdatabase.BasicSimulation
     [2] computerdatabase.advanced.AdvancedSimulationStep01
     [3] computerdatabase.advanced.AdvancedSimulationStep02
     [4] computerdatabase.advanced.AdvancedSimulationStep03
     [5] computerdatabase.advanced.AdvancedSimulationStep04
     [6] computerdatabase.advanced.AdvancedSimulationStep05
0 # 作成したテストシナリオである com.nikaera.AccessSimulation を選択して実行します。
Select run description (optional)

Simulation com.nikaera.AccessSimulation started...

#...

Simulation com.nikaera.AccessSimulation completed in 10 seconds
Parsing log file(s)...
Parsing log file(s) done
Generating reports...

================================================================================
---- Global Information --------------------------------------------------------
> request count                                         60 (OK=60     KO=0     )
> min response time                                     11 (OK=11     KO=-     )
> max response time                                    372 (OK=372    KO=-     )
> mean response time                                   104 (OK=104    KO=-     )
> std deviation                                         99 (OK=99     KO=-     )
> response time 50th percentile                         53 (OK=53     KO=-     )
> response time 75th percentile                        212 (OK=212    KO=-     )
> response time 95th percentile                        259 (OK=259    KO=-     )
> response time 99th percentile                        307 (OK=307    KO=-     )
> mean requests/sec                                  5.455 (OK=5.455  KO=-     )
---- Response Time Distribution ------------------------------------------------
> t < 800 ms                                            60 (100%)
> 800 ms < t < 1200 ms                                   0 (  0%)
> t > 1200 ms                                            0 (  0%)
> failed                                                 0 (  0%)
================================================================================

Reports generated in 0s.
Please open the following file: /Users/nika/Desktop/gatling-charts-highcharts-bundle-3.5.1/results/accesssimulation-20210314144205502/index.html
```

テストシナリオの実行に成功していることが確認できたら、複数ユーザ認証した情報を元に行うテストシナリオを書いていきます。

## 複数ユーザ認証を行うテストシナリオを実装する

複数ユーザ認証するテストシナリオを実装していきます。PlayFab で複数ユーザの認証情報を元に CloudFunction の負荷テストを行うことを想定しています。[^1] テストシナリオを作成するにあたり、`./user-files/simulations/` フォルダに新たに `playfab.com` フォルダを作成して、その中に `TestCloudFunctionSimulation.scala` ファイルを生成します。

[^1]: コードの `setUp` でシナリオを複数指定する箇所についてですが、より良いやり方があれば是非ご教授いただけますと幸いです...

```scala:./user-files/simulations/playfab.com/TestCloudFunctionSimulation.scala
package com.playfab

import java.io._
import java.net.{HttpURLConnection, URL}
import io.gatling.core.Predef._
import io.gatling.http.Predef._
import scala.concurrent.duration._
import scala.util.Random
import scala.util.parsing.json.JSON
import scala.collection.mutable.ListBuffer
import io.gatling.core.structure.PopulationBuilder

class TestCloudFunctionSimulation extends Simulation {
  // PlayFab に登録されたユーザの ID 群
  val playfabUsers = Array[String](
    "user1",
    "user2",
    "user3",
    "user4",
    "user5",
    "user6",
    "user7",
    "user8",
    "user9",
    "user10"
  )

  // PlayFab の TitleId 及び Secret を変数に保持しておく
  val playfabId = "XXXXX"
  val playfabSecret = "XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"
  val playfabApiUrl =
    s"https://${playfabId}.playfabapi.com"

　// PlayFab の Login With Server Custom Id を利用して、
  // ユーザの認証情報を取得するために用いる関数
  def getPlayfabAuth(serverCustomId: String): Option[Any] = {
    val url = new URL(
      s"${playfabApiUrl}/Server/LoginWithServerCustomId"
    )
    val con = url.openConnection().asInstanceOf[HttpURLConnection]
    HttpURLConnection.setFollowRedirects(false)
    con.setRequestMethod("POST")
    con.setRequestProperty("Content-Type", "application/json")
    con.setRequestProperty("X-SecretKey", playfabSecret)
    con.setDoOutput(true)

    val out = new OutputStreamWriter(con.getOutputStream(), "UTF-8")
    out.write(s"""{
	    "CreateAccount": false,
	    "ServerCustomId": "${serverCustomId}"
    }""")
    out.flush()
    out.close()
    con.connect()

    val in = con.getInputStream

    val br = new BufferedReader(new InputStreamReader(in, "UTF-8"));
    val json = br.readLine()

    in.close()
    con.disconnect()

    return JSON.parseFull(json)
  }

  val httpConf = http
    .baseUrl(playfabApiUrl)
    .header("Content-Type", "application/json")

  // CloudFunction の Request Body を作成するために利用する関数
  def cloudScriptDto(funcName: String, funcArgs: String): String = {
    return s"""{
	    "FunctionName": "${funcName}",
        "FunctionParameter": ${funcArgs}
    }"""
  }

  val scenarios: ListBuffer[PopulationBuilder] =
    new ListBuffer[PopulationBuilder]()

  playfabUsers.foreach(userId => {
    // playfabUsers で指定したユーザ ID 情報を元に、
    // PlayFab の認証情報を取得利用することで、
    val playfab = this.getPlayfabAuth(userId).get
    val map: Map[String, Option[Any]] =
      playfab.asInstanceOf[Map[String, Option[Any]]];

    val data = map.get("data").get.asInstanceOf[Map[String, Option[Any]]];
    val entityTokenInfo =
      data.get("EntityToken").get.asInstanceOf[Map[String, Option[Any]]];
    val entityToken =
      entityTokenInfo.get("EntityToken").get.asInstanceOf[String];

    val entity =
      entityTokenInfo.get("Entity").get.asInstanceOf[Map[String, Option[Any]]];
    val entityId =
      entity.get("Id").get.asInstanceOf[String];

    // Test2 API については CloudFunction 実行時のパラメタを、
    // ランダム指定したいため、Random を用いてパラメタを散らすようにする
    val rand = new Random
    val values = Array(
      "value1",
      "value2",
      "value3",
      "value4",
      "value5"
    )
    val values_length = values.length

    // アクセス負荷を調査したい CloudFunction API 群を実行する。
    val pollingApiScn = scenario(s"PollingSimulation: ${entityId}")
      .exec(
        http("Test1 Api")
          .post("/CloudScript/ExecuteFunction")
          .header("X-EntityToken", entityToken)
          .body(StringBody(cloudScriptDto("Test1", null)))
          .check(status.is(200))
      )
      .exec(
        http("Test2 Api")
          .post("/CloudScript/ExecuteFunction")
          .header("X-EntityToken", entityToken)
          .body(
            StringBody(
              cloudScriptDto(
                "Test2",
                s"""{"value": "${values(rand.nextInt(values_length))}"}"""
              )
            )
          )
          .check(status.is(200))
      );

    // 1 ユーザあたり 3秒毎に 100回ずつ API 群を実行した際の
    // 負荷テストのシナリオを scenarios 変数に格納する
    scenarios += pollingApiScn
      .inject(
        atOnceUsers(100),
        nothingFor(10 seconds),
        atOnceUsers(100),
        nothingFor(10 seconds),
        atOnceUsers(100)
      )
      .protocols(httpConf);
  });

  // scenarios 変数に格納されたテストシナリオを並列実行する
  setUp(
    scenarios(0),
    scenarios(1),
    scenarios(2),
    scenarios(3),
    scenarios(4),
    scenarios(5),
    scenarios(6),
    scenarios(7),
    scenarios(8),
    scenarios(9)
  )
}

```

ザッとインラインコメントで説明を書きましたが、重要な点についてのみ補足します。
`def getPlayfabAuth` は PlayFab 認証するための関数となっていますが、**適宜認証に用いるサービス毎で関数の内容を変更することで、他サービスで認証するための関数として利用可能です。**
また `playfabUsers.foreach` 内で各ユーザが実行するテストシナリオを生成しつつ、それらを `scenarios` 変数に保持しています。そうすることで、最後に `setUp` 関数でシナリオをまとめてセットできるようになります。

::: message info

`playfabUsers.foreach` で値を指定するのではなく [CSV でテストに与えるフィードデータを定義する](http://www.ajisaba.net/develop/gatling/test_case_csv_feeder.html) する方法もあります。認証部分も含めてテストシナリオを書きたい場合にも便利に利用できます。またアカウント情報を CSV ファイルに定義できるようにしておくと、テストユーザの情報を新規追加したい場合で管理が楽になります。

:::

上記テストシナリオのソースコードを応用することで、様々なサービスで複数ユーザ認証した情報を元に負荷テストを行うためのシナリオを作成することが可能となります。

# おわりに

今回は Gatling で複数ユーザ認証した情報を元に負荷テストする手順について書きました。

**Gatling で生成されるレポートは見やすく、改善点を洗い出してコードの改善作業をするのにとても有用でした。** また、JMeter と比べて動作が軽いため気軽に実行しやすく、テストシナリオが全てコードで管理されているためシナリオの改変も素早く行うことが出来ました。

個人的にはテストシナリオを全てコードで記述できる Gatling が気に入ったので今後も有効活用していきたいと感じました。ただ、**Gatling 以外にも Python で書ける [locust](https://github.com/locustio/locust) や JavaScript で書ける [k6](https://github.com/loadimpact/k6) など、他にも気になるツールがいくつかあったので今後試してみたいなと考えています。**

勝手に負荷テストは JMeter 一択だと思っていたのですが、負荷テストツールには色々あるようなのでプロジェクトや自分にあったものを選定していけるよう随時キャッチアップしていきたいと感じました。

# 参考リンク

- [Apache JMeter \- Apache JMeter™](https://jmeter.apache.org/)
- [Gatling Open\-Source Load Testing – For DevOps and CI/CD](https://gatling.io/)
- [Azure 関数を使用した PlayFab CloudScript \- PlayFab \| Microsoft Docs](https://docs.microsoft.com/ja-jp/gaming/playfab/features/automation/cloudscript-af/)
- [JDK Builds from Oracle](https://jdk.java.net/)
- [Maven – Welcome to Apache Maven](https://maven.apache.org/)
- [sbt \- The interactive build tool](https://www.scala-sbt.org/)
- [Open Source – Gatling Open\-Source Load Testing](https://gatling.io/open-source/)
- [負荷試験ツール・Gatling・CSV ファイルと Feeder を使ったテストケース](http://www.ajisaba.net/develop/gatling/test_case_csv_feeder.html)
- [Locust \- A modern load testing framework](https://locust.io/)
- [Load testing for engineering teams \| k6](https://k6.io/)
