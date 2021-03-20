---
devto_article_id: 640772
title: "GitHub 接続時の ~/.ssh/config の書き方"
emoji: "🔥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ssh", "git", "github", "初心者"]
published: true
---

# はじめに

本記事ではタイトルの内容に加えて `~/.ssh/config` に対する基本的な理解を深めるため、各種設定項目に関する説明も付随して行います。GitHub を例に挙げてはいるものの、Bitbucket や GitLab、Git サーバへ接続する際にも利用可能な `~/.ssh/config` の設定について記載しております。

# `~/.ssh/config` の書き方

まず GitHub へ接続する `~/.ssh/config` の設定を見ていきます。GitHub で認証に使用する SSH キーは登録済みで秘密鍵は `~/.ssh/github` へ配置している想定です。

:::message
SSH キーが未登録の場合は [公式サイトの手順](https://docs.github.com/ja/free-pro-team@latest/github/authenticating-to-github/connecting-to-github-with-ssh) に従って鍵の生成・登録までを行っておきます。
:::

```ssh:~/.ssh/config
Host github.com
    IdentityFile ~/.ssh/github
    User git
```

`~/.ssh/config` の各種設定項目は下記になります。
| 項目 | 説明 |
| ---- | ---- |
| Host | ホスト名を指定する |
| IdentityFile | 接続時に使用する秘密鍵を指定する |
| User | 接続時のユーザ名を指定する |

設定を行うことで `github.com` に SSH 接続する際、ユーザ名に `git` が指定され、`~/.ssh/github` に存在する秘密鍵を用いて SSH 接続を試みるようになります。

早速 `github.com` 接続時に正しく認証が通っているか確認するため、適当なプライベートリポジトリを `git clone` してみます。

![git clone の実行結果 (成功)](https://i.gyazo.com/98db386f77ab94590c162279e9e039e9.png)
*プライベートリポジトリを `git clone` した実行結果 (成功)*

無事に `git clone` できることが確認できたら成功です。

## 接続先に応じて秘密鍵を使い分けたい

GitHub で常に同一の秘密鍵を用いて認証を行う際は問題ないのですが、例えば GitHub アカウントをプライベート用と仕事用で使い分けていて、リポジトリ先に応じて秘密鍵を使い分けたいというケースはあると思います。

その場合は **`~/.ssh/config` の設定と `git` のリモートリポジトリへの接続情報を変更することで対応可能です。**

例えばプライベート用の GitHub アカウント `A` と仕事用の GitHub アカウント `B` が存在するとします。その際 `github-A` がホストの時は `A` の秘密鍵を、`github-B` がホストの時は `B` の秘密鍵を利用するための設定は下記になります。

```ssh:~/.ssh/config
# プライベート用の GitHub アカウントで利用する接続設定
Host github-A
    HostName github.com
    IdentityFile ~/.ssh/github-A
    User git

# 仕事用の GitHub アカウントで利用する接続設定
Host github-B
    HostName github.com
    IdentityFile ~/.ssh/github-B
    User git
```

新たに `HostName` という項目を設定しました。`HostName` には接続先を指定します。**`HostName` は何も指定しない場合 `Host` と同じ値が設定されます。**

今回は同一接続先に対してアカウントを切り替えたいので、`Host` にはアカウント識別可能な値 (ex: `github-A`, `github-B`, etc.) を、`HostName` に接続先である `github.com` を明示的に指定してアカウント毎に設定をわけました。

次に `git` のリモートリポジトリへの接続情報を変更します。

---

`~/.ssh/config` に `User` を指定している場合は該当 `Host` へ接続時に自動的にユーザが設定されるため、接続 URL からユーザ名である `git` は取り除いても問題ありません。

```bash
# 現在のリモートリポジトリ URL
git remote -v

origin  git@github.com:nikaera/private-repository.git (fetch)
origin  git@github.com:nikaera/private-repository.git (push)

# リモートリポジトリ URL を
# git@github.com:nikaera/private-repository.git から
# github-A:nikaera/private-repository.git に変更する
# (ユーザ名の git は変更先から取り除いている)
git remote set-url origin github-A:nikaera/private-repository.git

# 新たに設定したリモートリポジトリ URL
git remote -v

origin  github-A:nikaera/private-repository.git (fetch)
origin  github-A:nikaera/private-repository.git (push)
```

この状態で先ほど `git clone` してきたプライベートリポジトリ内で `git ls-remote origin` コマンドを実行してみて正常に結果が取得できれば正しく設定できています。

![`A` の接続設定で `git ls-remote origin` の実行に成功する](https://i.gyazo.com/0b5cbb50e44239afa000b69295f0eebe.png)
*`A` の接続設定で `git ls-remote origin` の実行に成功する*

次に `B` のアカウント情報で接続を試みます。

```bash
# 現在のリモートリポジトリ URL
git remote -v

origin  github-A:nikaera/private-repository.git (fetch)
origin  github-A:nikaera/private-repository.git (push)

# リモートリポジトリ URL を github-A:nikaera/private-repository.git から
# github-B:nikaera/private-repository.git に変更する
git remote set-url origin github-B:nikaera/private-repository.git

# 新たに設定したリモートリポジトリ URL
git remote -v

origin  github-B:nikaera/private-repository.git (fetch)
origin  github-B:nikaera/private-repository.git (push)
```

`B` には先ほどのプライベートリポジトリの読み取り権限が無い状態で、`git ls-remote origin` コマンドを実行してみると失敗するはずです。

![`B` の接続設定で `git ls-remote origin` の実行に失敗する](https://i.gyazo.com/f8fc81d537453512bda2c8f64bde8821.png)
*`B` の接続設定で `git ls-remote origin` の実行に失敗する*

これで今後は `git` のリモートリポジトリ URL を一度書き換えておくだけで、それぞれ適切な秘密鍵で GitHub 認証する設定ができました。

単に Git サーバに SSH 接続する際の設定情報としては、上記項目を抑えておけば問題無いです。しかし、他にも設定したほうが良い項目や、場合によっては設定が必要な項目も存在します。

## 基本的に設定しておいた方が良い項目

他に設定しておいたほうが良い項目には下記があります。

```ssh:~/.ssh/config
Host github.com
    IdentityFile ~/.ssh/github
    User git
    IdentitiesOnly yes # IdentityFile で指定した秘密鍵でのみ認証を試みる
    Compression yes # Git でのファイル転送時に圧縮する
```

| 項目 | 説明 | 型 |
| ---- | ---- | ---- |
| IdentitiesOnly | `IdentityFile` で指定した鍵ファイルでのみ認証を行うかどうか指定する | boolean (`yes` or `no`) |
| Compression | 圧縮転送を行うかどうか指定する | boolean (`yes` or `no`) |

`IdentityFile` で指定したファイル以外で認証が通ってしまう状況だと、意図したアカウントで適切に認証が通せていない可能性が出てきてしまいます。

そのため、**特別な理由がなければ `IdentitiesOnly` には `yes` を設定しておいたほうが良いです。**

`IdentitiesOnly` を `no` に設定しておくと [`ssh-add`](https://nxmnpg.lemoda.net/ja/1/ssh-add) で登録したキーの中からも認証を試みるようになります。

また、`Compression` についても `yes` に設定しておくことをオススメします。**プログラミングを行っているプロジェクトでは、テキストファイルの数が大半を占めるはずなので、圧縮転送を有効にしておいたほうがファイル転送の速度向上を見込めるからです。**

:::message alert
ただし `Compression` は大容量の単一のファイルがアップされているなどして、**圧縮効率の悪いファイルを含んでいるプロジェクトの場合は逆に転送速度が低下する恐れもあるので要注意です。**
:::

## 接続先によっては設定が必要な項目

接続先に応じて設定する必要がありそうな項目には下記があります。

```ssh:~/.ssh/config
Host github.com
    IdentityFile ~/.ssh/github
    User git
    Port 12345 # 接続先のポートが 22 以外の場合指定する
```

| 項目 | 説明 | 型 |
| ---- | ---- | ---- |
| Port | SSH 接続時のポート番号を設定する | number (ex: 22, 22222, etc.) |

例えば自前で用意した Git リポジトリに対して接続する場合、SSH のポートがセキュリティ上の理由等でウェルノウンポートの `22` ではない場合がありえます。その場合は `Port` を明示的に指定することで接続時のポート番号を適切な値に設定しておく必要が出てくるでしょう。

# おわりに

今回は GitHub へ SSH 接続する際の例を元に `~/.ssh/config` の設定方法について書きました。`~/.ssh/config` で `Host` の設定は行っているものの、それに応じて `git` のリモートリポジトリ URL を変更すべき場合もあることを明示している記事が少なかったので書きました。

また、本記事では Git 接続時の `~/.ssh/config` の設定にフォーカスを当てましたが、`~/.ssh/config` についてはサーバに SSH 接続して作業する際に設定しておくと良い項目等もあります。

`~/.ssh/config` の各種設定項目について理解していると頻繁に実行するコマンドのコスト削減ができて便利です。特に[こちらの記事](https://qiita.com/oohira/items/7deae31469cfbbd740c1)はとても参考になりました。

# 参考リンク

- [GitHub に SSH で接続する - GitHub Docs](https://docs.github.com/ja/free-pro-team@latest/github/authenticating-to-github/connecting-to-github-with-ssh)
- [SSH 接続をテストする - GitHub Docs](https://docs.github.com/ja/free-pro-team@latest/github/authenticating-to-github/testing-your-ssh-connection)
- [~/.ssh/config による快適 SSH 環境 - Qiita](https://qiita.com/oohira/items/7deae31469cfbbd740c1)