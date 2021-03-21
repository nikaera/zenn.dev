---
dev_article_id: 640766
title: "MediaPackage 用の CloudFront ディストリビューションを AWS SDK で作成する"
emoji: "🎥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "nodejs", "mediapackage", "cloudfront"]
published: true
---

# はじめに

とある事情で MediaPackage のエンドポイント用の CloudFront ディストリビューションを AWS SDK で作成する機会がありました。その際得た知見をソースコードを交えながら備忘録として記事に残しておきます。

本記事内容で紹介しているソースコードは [Gist](https://gist.github.com/nikaera/d9d616a089998ff2d23b210faf3a85c3) にも同じ内容でアップしてあります。

**ちなみに MediaLive + MediaPackage + CloudFront の構成でインフラ構築したい場合は、[CloudFormation が MediaPackage にも対応した](https://dev.classmethod.jp/articles/update-aws-elemental-mediapackage-cloudformation/)ので CloudFormation の利用を推奨します。**

本記事内容はあくまでも何らかの事情で、**後から CloudFront ディストリビューションを MediaPackage エンドポイントに紐づけたいケース等で参考になると思われます。**

# 実装内容

作成したソースコードの内容は下記になります。
最下部の `createDistributionForMediaPackage` が本記事タイトルに該当する関数です。

```typescript:CloudFrontClientForMediaPackage.ts
import { CloudFront } from "aws-sdk";
import * as url from "url";

import {
    CreateDistributionWithTagsResult,
    GetDistributionResult,
    UpdateDistributionResult
} from "aws-sdk/clients/cloudfront";

export class CloudFrontClientForMediaPackage {
    private cloudFront: CloudFront;

  constructor() {
    this.cloudFront = new CloudFront({
        region: "ap-northeast-1",
        apiVersion: '2020-05-31',
    });
}

/**
 * CloudFront ディストリビューションの情報を取得するために利用する
 * @param id CloudFront ディストリビューションの ID
 * @return ディストリビューションの情報を取得する
 */
  async getDistribution(id: string): Promise<GetDistributionResult> {
      const distribution = await this.cloudFront.getDistribution({
          Id: id
      }).promise()

      return distribution;
  }

  /**
   * CloudFront ディストリビューションの設定内容を取得するために利用する
   * @param id CloudFront ディストリビューションの ID
   * @return ディストリビューションの設定内容を取得する
   */
  async getDistributionConfig(id: string): Promise<CloudFront.DistributionConfig> {
      const config = await this.cloudFront.getDistributionConfig({
          Id: id
      }).promise()

      return config.DistributionConfig;
  }

  /**
   * CloudFront ディストリビューションを削除する
   * @param id 削除したい CloudFront ディストリビューションの ID
   */
  async deleteDistribution(id: string) {
    const distribution = await this.getDistribution(id);

    await this.cloudFront.deleteDistribution({
        Id: id, IfMatch: distribution.ETag
    }).promise()
  }

  /**
   * CloudFront ディストリビューションを無効化する
   * @param id 無効化したい CloudFront ディストリビューションの ID
   * @return 無効化した CloudFront ディストリビューションの情報
   */
  async disableDistribution(id: string): Promise<UpdateDistributionResult> {
      const distribution = await this.getDistribution(id);
      const config = distribution.Distribution.DistributionConfig;
      config.Enabled = false;

      return await this.cloudFront.updateDistribution({
        Id: id,
        IfMatch: distribution.ETag,
        DistributionConfig: config
      }).promise();
  }

  /**
   * MediaPackage のエンドポイント用の CloudFront ディストリビューションを作成する
   * @param id CloudFront ディストリビューションを判別するための ID
   * @param mediaPackageArn MediaPackage チャンネルの ARN
   * @param mediaPackageUrl MediaPackage エンドポイントの URL
   */
  async createDistributionForMediaPackage(
      id: string,
      mediaPackageArn: string,
      mediaPackageUrl: string
    ): Promise<CreateDistributionWithTagsResult> {

    // 1. url モジュールを用いて URL 文字列をパースする
    const mediaPackageEndpoint = url.parse(mediaPackageUrl);

    /**
    2. MediaPackage のエンドポイント URL から FQDN を取得する。
    後述する CloudFront ディストリビューションのオリジンのドメイン名としても利用する
    */
    const mediaPackageHostname = mediaPackageEndpoint.hostname;

    /**
    3. MediaPackage のエンドポイント URL のフォーマットは
    https://<AccountID>.mediapackage.<Region>.amazonaws.com/**** となっているので、
    FQDN の先頭部分を文字列分割で取り出すとアカウント ID が取得できる
    */
    const accountId = mediaPackageHostname.split('.')[0];

    // 4. 後述する CloudFront ディストリビューションのオリジン ID として、アカウント ID を利用する
    const targetOriginId = `MP-${accountId}`

    /**
    5. createDistribution ではなく、createDistributionWithTags 関数で、
    CloudFront ディストリビューションを作成する。MediaPackage との紐付けにタグを利用するため。
    */
    return await this.cloudFront.createDistributionWithTags({
        DistributionConfigWithTags: {
            Tags: {
                Items: [
                    /**
                    !!!!!重要!!!!!

                    6. CloudFront ディストリビューションに紐付けたい
                    MediaPackage エンドポイントのチャンネル ARN を
                    mediapackage:cloudfront_assoc で定義する。

                    mediapackage:cloudfront_assoc を定義することで、
                    CloudFront ディストリビューションと
                    MediaPackage チャンネルを紐付けることが可能となる。
                    */
                    {
                        Key: 'mediapackage:cloudfront_assoc',
                        Value: mediaPackageArn
                    },
                    {
                        Key: 'Id',
                        Value: id
                    },
                    {
                        Key: 'Product',
                        Value: 'product'
                    },
                    {
                        Key: 'Stage',
                        Value: 'dev'
                    }
                ]
            },
            DistributionConfig: {
                CallerReference: new Date().toISOString(),
                Comment: `Managed by MediaPackage - ${id}`,
                Enabled: true,
                /**
                7. CloudFront ディストリビューションのオリジンには 2つ設定します。
                1つが MediaPackage のエンドポイントに対するものと、
                もう 1つが MediaPacakge サービスに対するものです。

                基本的には MediaPackage のエンドポイントに対するオリジンを利用します。
                例外時に向けるオリジンが MediaPacakge サービスに対するものになります。
                */
                Origins: {
                    Quantity: 2,
                    Items: [
                        {
                            DomainName: mediaPackageHostname,
                            Id: targetOriginId,
                            CustomOriginConfig: {
                                HTTPPort: 80,
                                HTTPSPort: 443,
                                OriginProtocolPolicy: 'match-viewer'
                            }
                        },
                        {
                            DomainName: 'mediapackage.amazonaws.com',
                            Id: "TEMP_ORIGIN_ID/channel",
                            CustomOriginConfig: {
                                HTTPPort: 80,
                                HTTPSPort: 443,
                                OriginProtocolPolicy: 'match-viewer'
                            }
                        }
                    ]
                },
                /**
                8. CacheBehaviors のいずれにも当てはまらなかった場合の
                キャッシュの振る舞いを定義します。

                MediaPackage は タイムシフト表示機能を使用する際等で、クエリ文字列に start, m, end を利用しています。
                そのため、それらの文字列は WhitelistedNames に含め QueryString には true を指定しておきます。

                DefaultCacheBehavior に引っかかる挙動は例外的扱いなので、
                使用するオリジンは MediaPackage サービスのものを設定します。
                */
                DefaultCacheBehavior: {
                    ForwardedValues: {
                        Cookies: {
                            Forward: 'whitelist',
                            WhitelistedNames: {
                                Quantity: 3,
                                Items: [
                                    'end', 'm', 'start'
                                ]
                            }
                        },
                        QueryString: true,
                        Headers: {
                            Quantity: 0
                        },
                        QueryStringCacheKeys: {
                            Quantity: 0
                        }
                    },
                    MinTTL: 6,
                    TargetOriginId: "TEMP_ORIGIN_ID/channel",
                    TrustedSigners: {
                        Enabled: false,
                        Quantity: 0
                    },
                    ViewerProtocolPolicy: 'redirect-to-https',
                    AllowedMethods: {
                        Items: [
                            'GET', 'HEAD'
                        ],
                        Quantity: 2,
                    },
                    MaxTTL: 60
                },
                /**
                9. CloudFront のエラーコード全ての TTL に 1sec を設定します。
                MediaPackage のエラーのキャッシュが長時間持続してしまうと、
                その間は MediaPackage で正常に配信できているとしても、
                復旧できない状態となるからです。
                */
                CustomErrorResponses: {
                    Quantity: 10,
                    Items: [
                    {
                        ErrorCode: 400,
                        ErrorCachingMinTTL: 1
                    }, {
                        ErrorCode: 403,
                        ErrorCachingMinTTL: 1
                    }, {
                        ErrorCode: 404,
                        ErrorCachingMinTTL: 1
                    }, {
                        ErrorCode: 405,
                        ErrorCachingMinTTL: 1
                    }, {
                        ErrorCode: 414,
                        ErrorCachingMinTTL: 1
                    }, {
                        ErrorCode: 416,
                        ErrorCachingMinTTL: 1
                    }, {
                        ErrorCode: 500,
                        ErrorCachingMinTTL: 1
                    }, {
                        ErrorCode: 501,
                        ErrorCachingMinTTL: 1
                    }, {
                        ErrorCode: 502,
                        ErrorCachingMinTTL: 1
                    }, {
                        ErrorCode: 503,
                        ErrorCachingMinTTL: 1
                    }
                    ]
                },
                /**
                10. CloudFront ディストリビューションのキャッシュの振る舞いを 2つ定義します。

                それぞれの設定内容は基本的に DefaultCacheBehavior で定義したものと同様です。
                しかし、利用するオリジンは MediaPackage エンドポイントに向けたものを利用します。

                1つは Microsoft Smooth Streaming での配信時に利用する
                index.ism に対するもので Smooth Streaming を true に設定しています。

                もう 1つは上記 Microsoft Smooth Streaming 以外の
                全てに当てはまるストリーミングに適用されるものになります。
                */
                CacheBehaviors: {
                    Quantity: 2,
                    Items: [{
                        MinTTL: 6,
                        PathPattern: 'index.ism/*',
                        TargetOriginId: targetOriginId,
                        ViewerProtocolPolicy: 'redirect-to-https',
                        AllowedMethods: {
                            Items: [
                                'GET', 'HEAD'
                            ],
                            Quantity: 2,
                        },
                        ForwardedValues: {
                            Cookies: {
                                Forward: 'whitelist',
                                WhitelistedNames: {
                                    Quantity: 3,
                                    Items: [
                                        'end', 'm', 'start'
                                    ]
                                }
                            },
                            QueryString: true,
                            Headers: {
                                Quantity: 0
                            },
                            QueryStringCacheKeys: {
                                Quantity: 0
                            },
                        },
                        SmoothStreaming: true
                    }, {
                        MinTTL: 6,
                        PathPattern: '*',
                        TargetOriginId: targetOriginId,
                        ViewerProtocolPolicy: 'redirect-to-https',
                        AllowedMethods: {
                            Items: [
                                'GET', 'HEAD'
                            ],
                            Quantity: 2,
                        },
                        ForwardedValues: {
                            Cookies: {
                                Forward: 'whitelist',
                                WhitelistedNames: {
                                Quantity: 3,
                                Items: [
                                    'end', 'm', 'start'
                                ]
                                }
                            },
                            QueryString: true,
                            Headers: {
                                Quantity: 0
                            },
                            QueryStringCacheKeys: {
                                Quantity: 0
                            },
                        }
                    }]
                },
                PriceClass: 'PriceClass_All'
            }
        }
    }).promise()
  }
}
```

`createDistributionForMediaPackage` で作成したディストリビューションは、[公式ページに記載された手順](https://docs.aws.amazon.com/ja_jp/mediapackage/latest/ug/cdns-cf.html) で作成した CloudFront ディストリビューションと同等のものになります。

詳細な説明はインラインコメントにて書きましたが、一応補足説明を少し付け加えておきます。

## 随所に出てくる `Quantity` について

**`Quantity` には `Items` で指定する項目の数を入力します。** 例えば `Headers` や `QueryStringCacheKeys` には `Items` に何も指定していないため、`Quantity` に `0` を指定します。

しかし、`AllowedMethods` や `WhitelistedNames` には `Items` に指定した項目数である `2` や `3` を `Quantity` に入力しています。**`Quantity` の数と `Items` の項目数が合わないと、エラーが発生するため、注意が必要です。**

## `mediapackage:cloudfront_assoc` を定義する意味

CloudFront ディストリビューションのタグに **`mediapackage:cloudfront_assoc` で紐付ける MediaPackage のチャンネル ARN を指定することで、MediaPackage コンソールから紐付けられた CloudFront ディストリビューション情報を参照できるようになります。**

試しに紐づけられた MediaPackage のチャンネルのエンドポイント詳細ページに遷移すると、
下記のような画面が確認できるはずです。

![mediapackage:cloudfront_assoc で紐付いた CloudFront ディストリビューションが確認できる](https://i.gyazo.com/93ec5b42f4e550e38a7b06a945e0102b.png)
**`mediapackage:cloudfront_assoc` で紐付いた CloudFront ディストリビューションが確認できる**

:::message

本記事内のソースコードでは他にも `Id`, `Product`, `Stage` といったタグを定義していますが、MediaPackage とは関係無いものなので削除して問題ありません。

:::

## `updateDistribution` を実行する際の注意点

これは今回の記事内容とは直接関係ないのですが、地味にハマったので載せておきます。

**CloudFront では `createDistribution` の時に要求されるパラメータよりも `updateDistribution` で要求されるパラメータのほうが多いです。** [AWS 公式ページの比較表](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/distribution-overview-required-fields.html)にある通りです。

そのため、`updateDistribution` で設定を一部更新したいだけなのに、とても多くのパラメータを指定する必要があり非常に面倒です。例えば CloudFront ディストリビューションの Enable/Disable を切り替えるだけでも 30 個近いパラメータを指定する必要あります。

上記の入力の手間を省くのには **`getDistribution` で取得した既存のディストリビューション情報を改変する形で `updateDistribution` のパラメータを作成すると楽でした。**

今回のソースコードの内容を参照すると `disableDistribution` が該当します。

```typescript
// 1. getDistribution を実行して CloudFront ディストリビューションの情報を取得する
const distribution = await this.getDistribution(id);

// 2. CloudFront ディストリビューションの設定内容を取得する
const config = distribution.Distribution.DistributionConfig;

// 3. CloudFront ディストリビューションの Enabled/Disabled を切り替えるオプションを改変する
config.Enabled = false;

// 4. 3. で改変した内容を updateDistribution で CloudFront ディストリビューションに反映する
return await this.cloudFront
  .updateDistribution({
    Id: id,
    IfMatch: distribution.ETag,
    DistributionConfig: config,
  })
  .promise();
```

# おわりに

ニッチな内容なので、本記事内容を今後利用するかどうかは分かりませんが、一応得た知見を記事として残しておきました。同様のことを行う必要が出てきた方の参考になれれば幸いです。

# 参考リンク

- [Class: AWS.CloudFormation — AWS SDK for JavaScript](https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/CloudFormation.html)
- [Amazon CloudFront を MediaPackage に使用する - AWS Elemental MediaPackage](https://docs.aws.amazon.com/ja_jp/mediapackage/latest/ug/cdns-cf.html)
- [Required fields for creating and updating distributions - Amazon CloudFront](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/distribution-overview-required-fields.html)
- [CloudFront と AWS Media Services によるライブストリーミングビデオの配信](https://docs.aws.amazon.com/ja_jp/AmazonCloudFront/latest/DeveloperGuide/live-streaming.html)
