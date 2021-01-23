---
title: "Actix web で HttpOnly な Cookie を設定する"
emoji: "🍪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rust", "actixweb", "cookie"]
published: true
---

# はじめに

最近 Rust を勉強するため、[Actix web](https://github.com/actix/actix-web) で [Bloggimg](https://github.com/nikaera/bloggimg) という Web アプリケーションを作りました。その際、セッション管理のために Cookie を利用したのですが、その際の手順及び設定方法についてまとめておきます。

本記事では Rust や Actix web のインストール方法については説明しません。Mac であれば `brew install rustup` して `rustup-init` した後、`PATH` に `$HOME/.cargo/bin` を追加するだけで大丈夫なはずです。詳細なインストール手順については [公式サイト](https://www.rust-lang.org/tools/install) をご参照ください。

開発環境については [VSCode の Rust Plugin](https://marketplace.visualstudio.com/items?itemName=rust-lang.rust) がオススメです。Rustup で Rust をインストールしている場合、設定から Rustup の PATH を `$HOME/.cargo/bin/rustup` にするだけで利用可能です。設定手順の詳細は[こちら](https://takoyaking.hatenablog.com/entry/2020/01/05/180000)をご参照ください。

# 動作環境

* Mac mini (M1, 2020)
  * Rust 1.49
  * Actix web 3
  * Serde 1.0

```toml:Cargo.toml
[package]
name = "cookie_test"
version = "0.1.0"
authors = ["nikaera"]
edition = "2018"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]
actix-web = "3"
serde = { version = "1.0", features = ["derive"] }

```

# Actix web で Cookie をセットする

サーバー側で Cookie を設定するため、HTTP レスポンスヘッダーに [Set-Cookie](https://developer.mozilla.org/ja/docs/Web/HTTP/Headers/Set-Cookie) を含める形でセッション情報をクライアントへ渡します。その際、最低でも Cookie の属性に `HttpOnly` と `Secure`、`SameSite=Strict` は設定します。実際の Cookie を設定するための Actix web でのサンプルコードは下記になります。

```rust
use std::env;
use actix_web::{App, HttpServer};
use actix_web::cookie::{Cookie, SameSite};
use actix_web::{get, web, Error, HttpRequest, HttpResponse};

use serde::{Deserialize};

/// Cookie に設定するキー
/// 今回は cookie_test をキーとして使用する
/// 
const KEY: &str = "cookie_test";

/// 存在していれば、HTTP Request ヘッダーから Cookie 文字列を取得する関数
/// 
/// # Arguments
/// * `req` - actix_web::HttpRequest
/// 
/// # Return value
/// * Option<String> - key=value; key1=value1;~ のような Cookie の文字列
/// 
fn get_cookie_string_from_header(req: HttpRequest) -> Option<String> {
    let cookie_header = req.headers().get("cookie");
    if let Some(v) = cookie_header {
        let cookie_string = v.to_str().unwrap();
        return Some(String::from(cookie_string));
    }
    return None;
}

/// 存在していれば、特定のキーで Cookie に設定された値を取得するための関数
/// 
/// # Arguments
/// * `key` - Cookie から取り出したい値のキー
/// * `cookie_string` - get_cookie_string_from_header 関数で取得した Cookie の文字列
/// 
/// # Return value
/// * Option<String> - Cookie に設定されている値を取得する
/// 
fn get_cookie_value(key: &str, cookie_string: String) -> Option<String> {
    // 取得した Cookie 文字列を ; で分割してループで回す
    let kv: Vec<&str> = cookie_string.split(';').collect();
    for c in kv {
        // Cookie 文字列をパースして key で指定した値とマッチしたキーが存在するかチェックする
        match Cookie::parse(c) {
            Ok(kv) => {
                if key == kv.name() {
                    // key で指定した値とマッチしたキーが存在していたら、その値を取得する
                    return Some(String::from(kv.value()));
                }
            }
            Err(e) => {
                println!("cookie parse error. -> {}", e);
            }
        }
    }
    return None;
}

/// 特定のキーで環境変数から値を取得するための関数
/// 
/// # Arguments
/// * `key` - 環境変数から取り出したい値のキー
/// 
/// # Return value
/// * String - 環境変数の値を文字列として取得する
/// 
fn get_env(key: &str) -> String {
    match env::var(key) {
        Ok(value) => return value,
        Err(e) => println!("ENV: ERR {:?}", e),
    }
    return String::new();
}

/// 環境変数に設定された HTTPS の値が 1 か判定する
/// Cookie の属性に Secure を付与するか判定するのに使用する
/// 
/// # Return value
/// * bool - Secure 属性を付与するか判定するための真偽値
/// 
fn is_https() -> bool {
    return get_env("HTTPS") == "1";
}

/// Cookie に設定する値を扱う HTTP Query の定義
#[derive(Deserialize)]
pub struct CookieQuery {
    pub value: String,
}

/// Cookie を設定するために用意したルート
/// 
/// # Example
/// 
/// 例えば GET /cookie?value=test にアクセスした場合、
/// Cookie に cookie_test=test が設定されるようになる
/// 
#[get("/cookie")]
async fn set_cookie(query: web::Query<CookieQuery>) -> Result<HttpResponse, Error> {
    // 設定したい Cookie を作成する
    // その際に Secure, HttpOnly, SameSite=Strict 属性を付与する
    let cookie = Cookie::build(KEY, &query.value)
            .secure(is_https())
            .http_only(true)
            .same_site(SameSite::Strict)
            .finish();

    // 作成した Cookie を HTTP Response の Set-Cookie ヘッダーに含めることで、
    // HTTP Response を受け取ったクライアントに Cookie をセットさせる
    return Ok(HttpResponse::Ok()
        .header("Set-Cookie", cookie.to_string())
        .body(""));
}

/// KEY で指定した Cookie が存在すれば、その値を返却する
/// KEY で指定した Cookie が存在しなければ、空の文字列を返却する
#[get("/")]
async fn index(req: HttpRequest) -> Result<HttpResponse, Error> {
    let cookie_string = get_cookie_string_from_header(req);
    if let Some(s) = cookie_string {
        if let Some(v) = get_cookie_value(KEY, s) {
            return Ok(HttpResponse::Ok().body(v));
        }
    }
    return Ok(HttpResponse::Ok().body(""));
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .service(set_cookie)
            .service(index)
    })
    .bind("0.0.0.0:8080")?
    .run()
    .await
}

```

ザッとインラインコメントで説明していますが、  
最も重要な `set_cookie` 関数について簡単に説明します。

Actix web には [`Cookie` クラス](https://docs.rs/actix-web/3.3.2/actix_web/http/struct.Cookie.html)が存在します。この `Cookie` クラスは Cookie 文字列を生成したり、パースしたりするのに役立ちます。`set_cookie` 関数では、Cookie を生成するための関数 [`Cookie::build`](https://docs.rs/actix-web/3.3.2/actix_web/http/struct.Cookie.html#method.build) を利用しています。

`Cookie::build` 関数を利用することで、メソッドチェインで Cookie の値や属性を設定できます。**作成した Cookie は `to_string` 関数を使用することで文字列として出力できます。出力した Cookie 文字列を HTTP レスポンスヘッダーに `Set-Cookie` として設定すれば Cookie を設定できます。**

# 動作検証

今回用意した Actix web のサンプルコードには 2つのエンドポイントを用意しました。

| URI | 説明 |
| ---- | ---- |
| `GET /cookie` | `value` クエリで HttpOnly な Cookie を設定する |
| `GET /` | `GET /cookie` で設定した Cookie を確認する |

`cargo run` で Actix web のサンプルを起動した後に、ブラウザで `http://localhost:8080/cookie?value=sample` にアクセスしてみます。またその際に HTTP レスポンスヘッダーを確認したいため、[開発者ツール](https://developer.mozilla.org/ja/docs/Learn/Common_questions/What_are_browser_developer_tools)を開いておきます。

![スクリーンショット 2021-01-23 13.12.27.png](https://i.gyazo.com/9a1cb0cf73aa001b9ad3fdf7d8ae9966.png =450x)
**HTTP レスポンスヘッダーに Set-Cookie が含まれていることを確認する**

`Set-Cookie` が含まれていることが確認できたら正常に Cookie が設定されているか確認します。

![スクリーンショット 2021-01-23 13.26.52.png](https://i.gyazo.com/a6963e1d7ec82c57b3ae8d4978e1c116.png =450x)
**HTTP リクエストヘッダーの Cookie に `cookie_test=sample` が存在していることを確認する**

![スクリーンショット 2021-01-23 13.32.00.png](https://i.gyazo.com/282473c4c235d92233929f95c25ca899.png)
**実際にブラウザーにも Cookie が正しく設定されているか、開発者ツールで確認する**

正常に Cookie がセットされていることが確認できれば作業完了です。Cookie の属性に `Secure` を設定した場合の動作検証は、環境変数に `HTTPS=1` をセットして `cargo run` で可能です。

# おわりに

Actix web で割と汎用的に使えそうな知識として Cookie の設定方法について、メモ的な記事を書いてみました。引き続き、Rust への理解を深めるために [Bloggimg](https://github.com/nikaera/bloggimg) の開発を進めながら学習を進めていきます 🧑‍🎓

:::message alert
本記事の内容がセキュリティの観点から適切でない場合等はコメントでご指摘いただけますと幸いです。
:::

# 参考リンク

* [Install Rust \- Rust Programming Language](https://www.rust-lang.org/tools/install)
* [Rust \- Visual Studio Marketplace](https://marketplace.visualstudio.com/items?itemName=rust-lang.rust)
* [VSCodeでRustインストールしたのに「Rustup not available」が出るとき \(備忘録\) \- TAKOYAKING’s blog](https://takoyaking.hatenablog.com/entry/2020/01/05/180000)
* [Set\-Cookie \- HTTP \| MDN](https://developer.mozilla.org/ja/docs/Web/HTTP/Headers/Set-Cookie)
* [actix/actix\-web: Actix Web is a powerful, pragmatic, and extremely fast web framework for Rust\.](https://github.com/actix/actix-web)
* [actix\_web::http::Cookie \- Rust](https://docs.rs/actix-web/3.3.2/actix_web/http/struct.Cookie.html)
* [ブラウザー開発者ツールとは？ \- ウェブ開発を学ぶ \| MDN](https://developer.mozilla.org/ja/docs/Learn/Common_questions/What_are_browser_developer_tools)