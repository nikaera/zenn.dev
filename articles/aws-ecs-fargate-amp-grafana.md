---
dev_article_id: 918242
title: "ECS Fargate のメトリクスを Prometheus Agent 使って AMP に送って Grafana で監視する"
emoji: "🔥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["fargate", "prometheus", "grafana", "aws", "nodejs"]
published: true
---

# はじめに

:::message
この記事は [AWS Advent Calendar 2021](https://qiita.com/advent-calendar/2021/aws) の 5 日目の記事です。
:::

[Fargate](https://aws.amazon.com/jp/fargate/) で Node.js アプリのメトリクスを [Prometheus Agent](https://prometheus.io/blog/2021/11/16/agent/) をサイドカーコンテナとして動かして、[Amazon Managed Service for Prometheus (AMP)](https://aws.amazon.com/jp/prometheus/) に送信して [Grafana](https://grafana.com/) で見られるようにしてみました。

:::message alert
ちなみに Promethus Agent はまだ [実験的な機能](https://prometheus.io/blog/2021/11/16/agent/#prometheus-agent-mode) なため、**実務での利用は推奨しません。**
:::

本記事の環境構築には [AWS CDK](https://aws.amazon.com/jp/cdk/) を利用しています。

# 動作環境

* Node.js v16.13.0
* AWS CDK 2.0.0 (build 4b6ce31)
* Prometheus 2.32.0-rc.0

# 環境構築

早速環境構築を進めていきます。まだ AMP については CDK から操作できないようでしたので、ワークスペースの作成については AWS コンソールから手動で行います。(2021/12/06)

## AMP のワークスペースを作成する

まず、[AMP のコンソール画面](https://ap-northeast-1.console.aws.amazon.com/prometheus/home) に遷移してワークスペースを作成します。

![AMP のコンソール画面からワークスペースを作成する](https://i.gyazo.com/bf30611d6bc0f6b93281fd80123b611e.png)
**1. [AMP のコンソール画面](https://ap-northeast-1.console.aws.amazon.com/prometheus/home) からワークスペース作成画面に遷移する**

![2. AMP のワークスペースを作成する](https://i.gyazo.com/7199bed8693832b26f4594bcf5541c86.png)
**2. AMP のワークスペースを作成する**

![3. AMP のワークスペース作成完了と同時に設定値を控えておく](https://i.gyazo.com/e32231b46a716aaeb37c8ef88a02e434.png)
**3. AMP のワークスペース作成完了を確認すると同時に設定値を控えておく**

AMP ワークスペースの `エンドポイント - リモート書き込み URL` は、**Prometheus Agent で AMP にデータ送信する際や、Grafana でデータソースを登録する際などに必要となるため控えておきます。**

## AWS CDK で環境構築する

AMP 以外のインフラについては CDK で構築作業を進めます。まずは下記コマンドで CDK プロジェクトを作成します。使用言語は `TypeScript` を選択します。

```bash
mkdir prometheus-agent-test && cd prometheus-agent-test
cdk init --language typescript
```

まず CDK でインフラ構築を進めていく前に、メトリクス収集テスト用の Node.js アプリを準備します。

### ECS Fargate で動かす Node.js アプリを準備する

[prom-client](https://github.com/siimon/prom-client) を利用して、Node.js のメトリクスが取得できるだけの Node.js アプリを準備します。`prometheus-agent-test` フォルダで下記コマンドを実行します。

```bash
mkdir metrics-app && cd metrics-app
npm init -y
npm install --save prom-client
```

次に `metrics-app` フォルダ内に `index.js` を作成して下記の編集を行います。

```javascript:metrics-app/index.js
'use strict';

const http = require('http');
const server = http.createServer();

const client = require('prom-client');
const register = new client.Registry()

// 5秒間隔でメトリクスを取得する
client.collectDefaultMetrics({ register, timeout: 5 * 1000 });

server.on('request', async function(req, res) {
    // /metrics にアクセスしたら、Prometheus のレポートを返す
    if (req.url === '/metrics') {
        res.setHeader('Content-Type', register.contentType)
        
        const metrics = await register.metrics()
        return res.end(metrics)
    } else {
        return res.writeHead(404, {'Content-Type' : 'text/plain'});
    }
});

server.listen(8080);
```

`node index.js`　コマンドを実行して `http://localhost:8080/metrics` にアクセスしてみます。下記のように各種メトリクスが出力されている様子が確認できれば OK です。

![Prometheus のレポートが正常に出力されている様子](https://i.gyazo.com/a5feae6dba9a9f4eaecae0055dd9be9e.png)
**Prometheus のレポートが正常に出力されている様子**

今回は ECS 上で Node.js アプリを動作させるため、`Dockerfile` も作成します。

```docker:metrics-app/Dockerfile
FROM public.ecr.aws/docker/library/node:16-alpine3.12 AS builder

EXPOSE 8080
WORKDIR /usr/src/app

COPY package*.json ./
RUN npm install --max-old-space-size=4096

COPY . .

CMD [ "node", "index.js" ]
```

上記 `Dockerfile` 作成後、再び動作検証のため下記コマンドを実行してから、`http://localhost:8080/metrics` にアクセスしてみます。

```bash
docker build -t prometheus-agent-test/metrics-app . 
docker run -p 8080:8080 prometheus-agent-test/metrics-app:latest
```

先ほどと同様に `http://localhost:8080/metrics` アクセス時に各種メトリクスが出力されている様子を確認できれば OK です。

### Node.js アプリを監視する Prometheus Agent を準備する

まずは Prometheus 関連ファイルを配置するためのフォルダを作成します。`prometheus-agent-test` フォルダ内で下記コマンドを実行します。

```bash
mkdir prometheus-agent && cd prometheus-agent
```

次に Prometheus の設定テンプレートファイルを作成します。テンプレートファイルは [`sed`](https://atmarkit.itmedia.co.jp/ait/articles/1610/18/news008.html) を利用して中身の `__TASK_ID__` および `__REMOTE_WRITE_URL__` を書き換えて利用します。

```yaml:prometheus-agent/prometheus.tmpl.yml
global:
  scrape_interval: 5s
  external_labels:
    monitor: 'prometheus'

scrape_configs:
  - job_name: 'prometheus-agent-test'
    static_configs:
      - targets: ['localhost:8080']
        labels:
          # デフォルトの localhost:8080 がインスタンスとして利用されると、
          # メトリクスの判別がしづらくなるため ECS Task の ID を利用する
          instance: '__TASK_ID__'

remote_write:
  # AMP ワークスペース作成時に控えておいた、
  # `エンドポイント - リモート書き込み URL` を設定する箇所
  - url: '__REMOTE_WRITE_URL__'
    sigv4:
      region: ap-northeast-1
    queue_config:
      max_samples_per_send: 1000
      max_shards: 200
      capacity: 2500

```

設定ファイルの作成が完了したら、テンプレートファイルを利用して Prometheus の設定ファイルを作成し、Prometheus Agent を起動させるためのシェルスクリプトを作成します。

```sh:prometheus-agent/docker-entrypoint.sh
#!/bin/sh

while [ -z "$taskId" ]
do
  # ECS Fargate で起動したタスク ID を取得する
  taskId=$(curl --silent ${ECS_CONTAINER_METADATA_URI}/task | jq -r '.TaskARN | split("/") | .[-1]')

  echo "waiting..."
  sleep 1
done

echo "taskId: ${taskId}"
echo "remoteWriteUrl: ${REMOTE_WRITE_URL}"

# タスク ID `taskId` および、環境変数 `REMOTE_WRITE_URL` で、
# Prometheus のテンプレートファイル `prometheus.tmpl.yml` の内容を書き換え、
# その結果を `/etc/prometheus/prometheus.yml` に出力する
cat /etc/prometheus/prometheus.tmpl.yml | \
    sed "s/__TASK_ID__/${taskId}/g" | \
    sed "s>__REMOTE_WRITE_URL__>${REMOTE_WRITE_URL}>g" > /etc/prometheus/prometheus.yml

# --enable-feature=agent で Prometheus を Agent モードで起動する
# Prometheus のコンフィグファイルには上記で出力した `/etc/prometheus/prometheus.yml` を利用する
/usr/local/bin/prometheus \
    --enable-feature=agent \
    --config.file=/etc/prometheus/prometheus.yml \
    --web.console.libraries=/etc/prometheus/console_libraries \
    --web.console.templates=/etc/prometheus/consoles

```

これで Prometheus Agent 起動のための準備は整ったため、最後に `Dockerfile` を準備します。ちなみに Prometheus Agent は `v2.32.0` で利用可能です。

```docker:prometheus-agent/Dockerfile
FROM --platform=arm64 alpine:3.15

ADD prometheus.tmpl.yml /etc/prometheus/

RUN apk add --update --no-cache jq sed curl

# ARM64 で動作する Prometheus v2.32.0 を curl でダウンロード展開する
RUN curl -sL -O https://github.com/prometheus/prometheus/releases/download/v2.32.0-rc.0/prometheus-2.32.0-rc.0.linux-arm64.tar.gz
RUN tar -zxvf prometheus-2.32.0-rc.0.linux-arm64.tar.gz && rm prometheus-2.32.0-rc.0.linux-arm64.tar.gz

# `prometheus` コマンドを `/usr/local/bin/prometheus` に移動する
RUN mv prometheus-2.32.0-rc.0.linux-arm64/prometheus /usr/local/bin/prometheus

COPY ./docker-entrypoint.sh /
RUN chmod +x /docker-entrypoint.sh
CMD ["/docker-entrypoint.sh"]
```

ここまでで CDK でインフラ整備を進めていくための下準備は完了です。

### ECS Fargate 上で Node.js アプリおよび Prometheus Agent を動作させる

あとは CDK で ECS Fargate 上で Node.js アプリおよび Prometheus Agent、Grafana を動作させるための環境を整備していきます。

`lib/prometheus-agent-test-stack.ts` の内容を書き換えます。

```typescript:lib/prometheus-agent-test-stack.ts
import { Construct } from 'constructs';
import {
  Stack,
  StackProps,
  aws_ecs as ecs,
  aws_logs as logs,
  aws_ecs_patterns as ecs_patterns,
  aws_iam as iam,
  aws_elasticloadbalancingv2 as elbv2,
  Duration,
} from 'aws-cdk-lib';

export class PrometheusAgentTestStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    // Node.js アプリに ecs_patterns.ApplicationLoadBalancedFargateService を利用して ALB 経由でアクセス可能にする
    const projectName = 'prometheus-agent-test';
    const fargateService = new ecs_patterns.ApplicationLoadBalancedFargateService(this, `${projectName}-fargate-service`, {
      serviceName: `${projectName}-fargate-service`,
      cpu: 256,
      desiredCount: 3,
      listenerPort: 80,
      taskImageOptions: {
        family: `${projectName}-taskdef`,
        image: ecs.ContainerImage.fromAsset('metrics-app'),
        containerPort: 8080,
        logDriver: ecs.LogDrivers.awsLogs({
          streamPrefix: `/${projectName}/metrics-app`,
          logRetention: logs.RetentionDays.ONE_DAY
        })
      },
      cluster: new ecs.Cluster(this, `${projectName}-cluster`, {
        clusterName: `${projectName}-cluster`
      }),
      memoryLimitMiB: 512,
    });
    fargateService.targetGroup.configureHealthCheck({
      path: "/metrics",
      timeout: Duration.seconds(8),
      interval: Duration.seconds(10),
      healthyThresholdCount: 2,
      unhealthyThresholdCount: 4,
      healthyHttpCodes: "200",
    });

    // 本質ではないが、Gravition2 で動作させたいために RuntimePlatform のプロパティを上書きしている
    const fargateServiceTaskdef = fargateService.taskDefinition.node.defaultChild as ecs.CfnTaskDefinition
    fargateServiceTaskdef.addPropertyOverride('RuntimePlatform', {
      CpuArchitecture: "ARM64",
      OperatingSystemFamily: "LINUX"
    });

    // AMP への書き込み権限を付与する
    fargateService.taskDefinition.taskRole.addManagedPolicy(
      iam.ManagedPolicy.fromAwsManagedPolicyName('AmazonPrometheusRemoteWriteAccess')
    )

    // AMP へメトリクス情報を送信するための Prometheus Agent コンテナを追加する
    const containerName = `${projectName}-prometheus-agent`
    fargateService.taskDefinition.addContainer(containerName, {
      containerName,
      image: ecs.ContainerImage.fromAsset('prometheus-agent'),
      memoryReservationMiB: 128,
      environment: {
        REMOTE_WRITE_URL: '<AMP ワークスペースのエンドポイント - リモート書き込み URL>'
      },
      logging: new ecs.AwsLogDriver({
        streamPrefix: `/${projectName}/prometheus-agent`,
        logRetention: logs.RetentionDays.ONE_DAY,
      })
    });

    // Grafana のタスク定義を作成する
    const grafanaDashboardTaskDefinition = new ecs.FargateTaskDefinition(this, `${projectName}-grafana-taskdef`, {
      family: `${projectName}-grafana-taskdef`
    });
    // Grafana のタスクが Prometheus Query を叩けるように権限付与する
    grafanaDashboardTaskDefinition.taskRole.addManagedPolicy(
      iam.ManagedPolicy.fromAwsManagedPolicyName('AmazonPrometheusQueryAccess')
    )

    // Grafana のコンテナを追加する。パスプレフィクスには dashboard を設定する
    const grafanaDashboardContainerName = `${projectName}-grafana-dashboard`  
    grafanaDashboardTaskDefinition.addContainer(grafanaDashboardContainerName, {
      containerName: grafanaDashboardContainerName,
      image: ecs.ContainerImage.fromRegistry('public.ecr.aws/ubuntu/grafana'),
      environment: {
        AWS_SDK_LOAD_CONFIG: 'true',
        GF_AUTH_SIGV4_AUTH_ENABLED: 'true',
        GF_SERVER_SERVE_FROM_SUB_PATH: 'true',
        GF_SERVER_ROOT_URL: '%(protocol)s://%(domain)s/dashboard'
      },
      portMappings: [{ containerPort: 3000 }],
      memoryLimitMiB: 512,
      logging: new ecs.AwsLogDriver({
        streamPrefix: `/${projectName}/grafana-dashboard`,
        logRetention: logs.RetentionDays.ONE_DAY,
      })
    });

    const grafanaDashboardServiceName = `${projectName}-grafana-dashboard-service`
    const grafanaDashboardService = new ecs.FargateService(this, grafanaDashboardServiceName, {
      serviceName: grafanaDashboardServiceName,
      cluster: fargateService.cluster,
      taskDefinition: grafanaDashboardTaskDefinition,
      desiredCount: 1
    });

    // Grafana のタスクを ALB のターゲットグループに紐づける
    fargateService.listener.addTargets(`${projectName}-grafana-dashboard-target`, {
      priority: 1,
      conditions: [
        elbv2.ListenerCondition.pathPatterns(['/dashboard/*']),
      ],
      healthCheck: {
        path: '/dashboard/login',
        interval: Duration.seconds(10),
        timeout: Duration.seconds(8),
        healthyThresholdCount: 2,
        unhealthyThresholdCount: 3,
        healthyHttpCodes: "200"
      },
      port: 3000,
      protocol: elbv2.ApplicationProtocol.HTTP,
      targets: [grafanaDashboardService],
    });
  }
}

```

その後、`cdk deploy` でインフラを構築します。

![CDK によるインフラ構築が正常に実行された時の様子](https://i.gyazo.com/c7da0f6c6b5a57edee47ae20a8026f8f.png)
**CDK によるインフラ構築が正常に実行された時の様子**

デプロイが正常に完了したことを確認したら、`Outputs` に出力されている URL の末尾に `/metrics` を付与してアクセスしてみます。出力されている URL のフォーマットは `http://<識別子>.ap-northeast-1.elb.amazonaws.com` になります。

つまり、`http://<識別子>.ap-northeast-1.elb.amazonaws.com/metrics` にアクセスしてみます。

![ALB 経由で Node.js アプリにアクセス可能なことを確認する](https://i.gyazo.com/c13a2b0efc3bb96e79b4f1f5a2886a8a.png)
**ALB 経由で Node.js アプリにアクセス可能なことを確認する**

ここまでで AWS CDK でのインフラ構築作業は完了しました。最後に Grafana で AMP のメトリクスを可視化するための作業を進めていきます。

# Grafana で Prometheus (AMP) のメトリクスを可視化する

先ほどの `/metrics` パスへのアクセス同様、`Outputs` に出力されている URL の末尾に `/dashboard/login` を付与してアクセスします。**Grafana の初期ユーザおよびパスワードは `admin` となります。**

つまり、`http://<識別子>.ap-northeast-1.elb.amazonaws.com/dashboard/login` にアクセスしてみます。

![ログインを行う](https://i.gyazo.com/fe4ea6515f54dec23234e10a93023a36.png)

ログイン情報が正しければ、新しいパスワードを設定する画面に遷移するので新たなパスワードを入力してログインを終えます。ログイン後は、Prometheus (AMP) をデータソースとして追加するために下記の操作を行います。

![1. 歯車アイコンをクリックして `Data sources` をクリックする](https://i.gyazo.com/599d49e4167ccc35c5d44d5df7678036.png)
**1. 歯車アイコンをクリックして `Data sources` をクリックする**

![2. `Add data source` ボタンをクリックする](https://i.gyazo.com/e88bfc58a35deaa16a16920a5e551837.png)
**2. `Add data source` ボタンをクリックする**

![3. データソースとして Prometheus を選択する](https://i.gyazo.com/1c8031a3182f03e20194bf0b76e66e5a.png)
**3. データソースとして Prometheus を選択する**

![4. Prometheus をデータソースとして追加する](https://i.gyazo.com/1684c1fca9decb29e60bd654400fd06b.png)

![4. Prometheus をデータソースとして追加する](https://i.gyazo.com/2119361eac350044e783ed0c9280580a.png)

![4. Prometheus をデータソースとして追加する](https://i.gyazo.com/00cc7edbfc45dbd43ad3b0cccea72c7d.png)
**4. Prometheus をデータソースとして追加する**

Prometheus (AMP) に送信したメトリクスを Grafana で可視化するための準備が整ったので、実際に Grafana のダッシュボードでメトリクスを可視化してみます。手っ取り早くメトリクスを可視化するため、ダッシュボードには [NodeJS Application Dashboard](https://grafana.com/grafana/dashboards/11159) を利用します。

![1. + アイコンをクリックして、`Import` をクリックする](https://i.gyazo.com/b36c0281e273e21fc9474666786281c3.png)
**1. + アイコンをクリックして、`Import` をクリックする**

![2. `NodeJS Application Dashboard` の ID を入力して `Load` ボタンをクリックする](https://i.gyazo.com/130718dff8b86758acb415764bd37338.png)
**2. `NodeJS Application Dashboard` の ID を入力して `Load` ボタンをクリックする**

![3. 必要な情報を入力して `NodeJS Application Dashboard` のインポートを完了する](https://i.gyazo.com/2aa8903f3f64d8021699cd5d4bfa9757.png)
**3. 必要な情報を入力して `NodeJS Application Dashboard` のインポートを完了する**

![4. ダッシュボードから Prometheus のメトリクスが確認できる](https://i.gyazo.com/1378b09c281da28653917ee40494dee8.png)
**4. ダッシュボードから Node.js アプリのメトリクスが確認できる**

ここまでの手順でメトリクスの可視化は完了しましたが、負荷に応じて実際にメトリクスが変化する様子も確認してみます。[Vegeta](https://github.com/tsenart/vegeta) を利用して、実際に負荷をかけてみます。下記コマンドを実行します。

```bash
echo 'GET <ALB の /metrics URL>' | vegeta attack -duration=5s | vegeta report
```

その後、再び Grafana のダッシュボードを見にいきます。負荷をかけた時間帯のみグラフに変化があることが確認できるはずです。

![ダッシュボードの CPU 使用率のグラフに変化があったことが確認できる](https://i.gyazo.com/d019d33d3e4bd321ae4d1f4bbecc6ef8.png)
**ダッシュボードの CPU 使用率のグラフに変化があったことが確認できる**

# おわりに

今回は ECS Fargate のメトリクスを Prometheus Agent で Amazon Managed Service for Prometheus (AMP) に送信し、それを Grafana で可視化する方法について紹介しました。

ECS のサービスでタスクを実行する場合は [サービスディスカバリ](https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/developerguide/service-discovery.html) の利用が可能なため、Prometheus の [サービスディスカバリの設定](https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/developerguide/service-discovery.html) を行うことで、単一の Prometheus で全てのコンテナのメトリクスを扱うことも可能です。

また Node.js アプリを作成する際に利用した `prom-client` で [カスタムメトリクス](https://github.com/siimon/prom-client#custom-metrics) を作成することで、監視したい項目を自由に増やすことも可能です。

本記事が ECS Fargate の監視を行う際の検討材料の 1つとなれたら幸いです。

# 参考リンク

* [AWS Fargate（サーバーやクラスターの管理が不要なコンテナの使用）\| AWS](https://aws.amazon.com/jp/fargate/)
* [Introducing Prometheus Agent Mode, an Efficient and Cloud\-Native Way for Metric Forwarding \| Prometheus](https://prometheus.io/blog/2021/11/16/agent/)
* [Amazon Managed Service for Prometheus \| フルマネージド Prometheus \| Amazon Web Services](https://aws.amazon.com/jp/prometheus/)
* [Grafana: The open observability platform \| Grafana Labs](https://grafana.com/)
* [AWS クラウド開発キット – アマゾン ウェブ サービス](https://aws.amazon.com/jp/cdk/)
* [siimon/prom\-client: Prometheus client for node\.js](https://github.com/siimon/prom-client)
* [tsenart/vegeta: HTTP load testing tool and library\. It's over 9000\!](https://github.com/tsenart/vegeta)
* [NodeJS Application Dashboard dashboard for Grafana \| Grafana Labs](https://grafana.com/grafana/dashboards/11159)