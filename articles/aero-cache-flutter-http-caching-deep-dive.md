---
dev_article_id: 2682786
title: "Flutter で本格的な HTTP キャッシュライブラリ「AeroCache」を作ってみた"
emoji: "☁️"
type: "idea"
topics: ["flutter", "dart", "http", "cache"]
published: true
---

# はじめに

Flutter は既存のキャッシュライブラリに [HTTP のプライベートキャッシュ](https://developer.mozilla.org/ja/docs/Web/HTTP/Guides/Caching) に準拠したものが少なく、想定と異なる挙動により意図した形でキャッシュが更新されない状況に遭遇することが多々あります。例えば、[Vary](https://developer.mozilla.org/ja/docs/Web/HTTP/Reference/Headers/Vary)に未対応であったり、キャッシュ再検証周りのロジックが独自実装だったりが該当します。

そこで、[RFC 9111](https://datatracker.ietf.org/doc/html/rfc9111) に準拠したキャッシュライブラリ **[AeroCache](https://pub.dev/packages/aero_cache)** を開発しました。この記事では、その実装過程で学んだ HTTP のプライベートキャッシュに関する知識を簡潔に解説しつつ AeroCache を紹介します。

https://nikaera.com/aero_cache/

# HTTP のプライベートキャッシュの仕組み

主にパフォーマンス向上や帯域節約、細やかなキャッシュ管理を実現するための仕組みとして、活用されるヘッダは以下の通りです。

| ヘッダー | 説明 |
|---|---|
| Cache-Control | レスポンスキャッシュの有効期限や再検証ルールを指示するディレクティブ群、`max-age` や `no-cache` などが代表的 |
| ETag | レスポンスごとに発行される一意の識別子、クライアントは `ETag` を使って「内容が変わったか」を効率的に判定可能 |
| Last-Modified | レスポンスの最終更新日時、`If-Modified-Since` ヘッダーと組み合わせて、キャッシュの更新判定に利用される |
| Vary | `Accept-Language` や `User-Agent` などを指定することで、端末やリクエストごとに異なるキャッシュを保持可能 |

以降では、各要素の具体的な動作について詳しく解説します。

## 主要な `Cache-Control` ディレクティブの対応

HTTP レスポンスヘッダーの `Cache-Control` は、キャッシュ動作を制御する重要な仕組みです。
プライベートキャッシュの `Cache-Control` で用いられる主要なディレクティブは以下です。

| ディレクティブ | 説明 |
|---|---|
| `max-age=N` | N秒間キャッシュを有効とみなす |
| `min-fresh=N` | 有効期限よりもN秒以上最新のデータのみ許容 |
| `max-stale[=N]` | 有効期限切れN秒以内なら古いキャッシュ許容（N省略時は無制限） |
| `stale-while-revalidate=N` | 有効期限切れN秒以内なら古いキャッシュ返却、裏側で更新処理 |
| `stale-if-error=N` | エラー時はN秒間古いキャッシュで代替 |
| `must-revalidate` | 期限切れ時は必ずキャッシュの再検証 |
| `no-cache` | 使用前に必ずキャッシュの再検証 |
| `no-store` | キャッシュしない（毎回ダウンロード） |
| `only-if-cached` | ネットワークアクセスせずキャッシュのみ利用 |

```http
Cache-Control: max-age=600, stale-while-revalidate=3600
```

上記の設定では、以下のような動作になります。
1. 最初の600秒間はキャッシュを新鮮とみなして即座に返す
1. 600秒経過後、3600秒までは古いデータを返しつつバックグラウンドで更新
1. ユーザーは常に高速なレスポンスを得られる

特に `stale-while-revalidate` は、ユーザー体験とパフォーマンスの両立を実現する強力なキャッシュ戦略です。新しいデータの取得は裏側で自動的に行われるため、ユーザーは操作を中断することなく最新のデータが利用可能になります。

**AeroCache はブラウザの挙動を強く意識しており、上記ディレクティブは全てサポートしています。ユーザーや開発者がキャッシュに関する挙動に悩まされないよう、独自ロジックなども極力排除しています。**

## ETag や Last-Modified による効率的な再検証

```http
# 初回レスポンス
HTTP/1.1 200 OK
ETag: "33a64df551425fcc55e4d42a148795d9f25f89d4"
Cache-Control: max-age=3600

# 再検証リクエスト
GET /api/data HTTP/1.1
If-None-Match: "33a64df551425fcc55e4d42a148795d9f25f89d4"

# 変更なしの場合
HTTP/1.1 304 Not Modified
```

ETag や Last-Modified の利点は以下の通りです。

1. **キャッシュが有効な場合**  
    - クライアントはキャッシュをそのまま返却
1. **キャッシュ期限切れの場合**  
    - クライアントは ETag/Last-Modified を付与して再検証リクエストを送信
    - データに変更無ければ `304 Not Modified` を返し、キャッシュを再利用
    - データに変更があれば 新しいデータを返却し、キャッシュを更新

この仕組みにより、無駄なデータ転送を防ぎつつ常に最新データを効率的に取得できます。

**再検証周りのロジックが独自実装のライブラリは、再起動しない限りキャッシュが更新されなかったりします。AeroCache ではそのような挙動は発生しません。**

## Vary による条件付きキャッシュ

レスポンスがリクエストヘッダーに依存する場合の重要な仕組みです。

```http
HTTP/1.1 200 OK
Vary: Accept-Language, User-Agent
Content-Language: ja
Cache-Control: max-age=3600
```

この例では、`Accept-Language` と `User-Agent` の値が異なるリクエストは、別々にキャッシュされます。

Vary ヘッダーはキャッシュの粒度を柔軟に制御できる仕組みです。例えば、言語やデバイスごとに異なるレスポンスを返す API では、Vary を使うことで下記のように「リクエストヘッダーごとに別々のキャッシュ」を作成できます。

- 多言語対応: `Accept-Language` を指定すれば、ユーザーの言語ごとに最適なキャッシュを保持
- デバイス最適化: `User-Agent` を使えば、デバイス間で異なるレスポンスをキャッシュ可能
- パーソナライズ: `Cookie` や `Authorization` など、個別ユーザー向けキャッシュも実現可能

**AeroCache では Vary を考慮した設計となっており Vary でキャッシュコントロールが可能です。**

# おわりに

実業務で悩まされてきた問題に終止符を打つべく開発した AeroCache について紹介しました。

標準仕様に沿ったキャッシュ管理で予期せぬ不具合や運用負荷を減らし、安心して利用できる HTTP キャッシュライブラリを目指しました。今後も実運用で得られた知見やフィードバックをもとに、より使いやすく信頼性の高いライブラリへと改善を続けていきます。

また、現状は利便性向上のための追加実装として下記を検討しています。

- [cached_network_image](https://pub.dev/packages/cached_network_image) の AeroCache 版
- HTTP クライアントの Interceptor 開発 (dio, http, etc.)
- コンテンツに影響しない URL パラメータのブラックリスト対応
- コンテンツ毎の可逆圧縮有無の設定, etc.

# 参考リンク

- [nikaera/aero\_cache: A high\-performance HTTP caching library for Dart/Flutter with zstd compression, ETag/Last\-Modified revalidation, and full Cache\-Control directive support\. ☁️](https://github.com/nikaera/aero_cache)
- [HTTP キャッシュ \- HTTP \| MDN](https://developer.mozilla.org/ja/docs/Web/HTTP/Guides/Caching)
- [RFC 9111 \- HTTP Caching \(RFC 9111\) 日本語訳](https://tex2e.github.io/rfc-translater/html/rfc9111.html)
