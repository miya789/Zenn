---
title: "オンプレGitは2要素認証の夢を見るか?"
emoji: "🔑"
type: "tech"
topics: [gitlab, github, git, sso]
published: true
---

# TL;DR

- Git Server と Git Client の間に **Reverse Proxy** を挟むことで、**通常の認証にプラスアルファした 2 要素認証を実現可能**
- ブラウザ経由は簡単に対応可能だが、Git コマンドも対応させるには **`git config`の`http.cookieFile`** を使えば良い
- 筆者のコントリビュートにより GCM も`http.cookieFike`を参照するようになった[^gcm-pr-1251]ので、**GCM も問題無く使える**(これが言いたかっただけ)
  [^gcm-pr-1251]: https://github.com/git-ecosystem/git-credential-manager/pull/1251

# 📚 背景

## ❓ Git サーバへのアクセスを 2 要素認証で防御したくない?

最近、様々なサービスで 2 要素認証が流行ってますよね?
GitHub や GitLab 等も 2 要素認証を有効にすれば、確かにブラウザ経由では 2 要素です。

しかし **Git コマンド経由だとどうでしょうか?**
結局、**アクセストークンだけの 1 要素**になってしまっていませんか?
更に、後述する helper 機能や OAuth 2.0 の仕組みを使うと、ユーザーが何も認証情報を渡さずに、Git コマンドが使えるようになってしまいます。これでは、実質 **0 要素認証**です。
アクセストークンが漏れるだけでソースコードにアクセスできてしまうって結構リスクありませんか?

:::message
確かに、「ローカルのリポジトリは良いのか?」みたいなご指摘もあるかもしれないですが、ローカルに置いていないものも取得される点で、やはり影響範囲が広いかと思います。
PC 自体の認証もあるにはありますが、PC に入られたら意味無いかと思います。
:::

可能であれば、**更にもう一つの認証で防御**したくありませんか?

## 🔑 再認証の必要十分条件

では、2 要素認証を具体的に考える前に、**個々の要素が満たすべき再認証の必要十分条件**を考えましょう。

例えば、認証が必要な手段が存在するとしても、回避可能な設定をされてしまうと意味がありません。
また、有効期限が無い上に、一度使った認証情報がプロセス終了時にも残ってしまうと、再認証が不要になってしまいます。

以上を考慮しますと、**再認証の条件は以下の表から構成され**、更に**図の網掛け部分が必要十分条件**に相当します。

| 項目         | 説明                                         | 例: TOTP は?    | 例: パスワードは? |
| ------------ | -------------------------------------------- | --------------- | ----------------- |
| **揮発性**   | プロセス終了による、保存済み認証情報の消失   | - / O(ブラウザ) | △ / O (ブラウザ)  |
| **有効期限** | 有効期限切れによる、アクセストークンの無効化 | O               | X                 |
| **強制力**   | 他に抜け道的な選択肢が無い                   | O               | O                 |

![再認証の必要十分条件](/images/git-2fa-authn/authz-condition.png =500x)
_再認証の必要十分条件_

例えば、パスワードは満たさないですが、TOTP は満たすので、これを組み合わせれば 2 要素認証です。
また、仮にアクセストークンが同様に満たさない場合でも、他に満たすものがあれば、2 要素認証は実現できます。

従って、まずはアクセストークンに関して、**この必要十分条件を満たすかどうか**を確認し、2 要素認証が実現できるかを確認します。

## 🎯 目的

目的としては、**オンプレ Git サーバ(特に GitLab)に関して、2 要素認証の必要十分条件を満たす方法**の調査です。尚、本項では主に GitLab に関して論じます。

# ❌ ① Client: helper(cache 以外)による 2 要素認証の検討

長くなりますが、アクセストークンの種類と課題から説明します。
尚、以前の記事[^gcm-for-2fa-authn]でも、PAT や AT&RT、そして GCM (Git Credential Manager) の話をしましたが、今回は再認証の観点から、2 要素認証として足り得るか見てみます。
[^gcm-for-2fa-authn]: [Git の HTTPS 認証に個人アクセストークンを求めるのは間違っているだろうか (Git Credential Manager のすゝめ)](https://zenn.dev/miya789/articles/manager-core-for-two-factor-authentication)

## 🔑 アクセストークンの種類

まずアクセストークンについて解説します。
以前の記事[^gcm-for-2fa-authn]でも紹介していますが、再度 2 要素認証の実現と言う観点で、整理します。

:::message
パスワードは論外です。漏れるとアクセストークンの発行でも何でもできてしまいますし。
:::

**Git コマンド使用時に必要**且つ**個人のアカウントに紐付く**ものに限ると、以下 2 種類に大別されます。

| 項目                               | 発行方法                                                                           |
| ---------------------------------- | ---------------------------------------------------------------------------------- |
| **Personal Access Token (PAT)**    | 個人の設定画面等から                                                               |
| **OAuth 2.0 Access Token (AT&RT)** | アプリケーションの GUI によるリダイレクト先から (事前のアプリケーション登録が必要) |

何れもスコープ等が適切に設定されていれば、**パスワードよりも流出時の影響を抑えられる点**で優れています。
では順に詳細を見ていきます。

### Personal Access Token (PAT)

一番簡単で、インターネットを調べて素人記事の大量に出て来るものが、この個人アクセストークン(Personal Access Token: PAT)です。
これは**個人でスコープ設定する必要**があり、本来であれば用途を理解した開発者のみが扱うものなので、 GitLab ではなく GitHub の場合は、開発者向けメニューでしか表示されません。

![GitHubの個人アクセストークンは開発者向けメニューにあります](/images/git-2fa-authn/github-pat-setting.png)
_GitHub の個人アクセストークンは開発者向けメニューにあります_

その分、スコープ次第で様々な用途に使用可能であり、例えば GitLab の`api`スコープであれば、GitLab の Web API を叩けるが、Git コマンドでは不要です。
このように権限を絞れるので、**正しくスコープ設定すれば、パスワードよりも流出時の影響を抑制可能**です。
但し、**付与するスコープは注意が必要**である上に、GitLab の場合は Ultimate プランでない限り**有効期限を無期限に設定可能**なので、注意が必要です。

:::message
GitLab 16.0 以降であれば、デフォルトで 1 年が最大の有効期限になります[^gitlab-pat]が、より短くカスタマイズできません。
[^gitlab-pat]: [Personal access tokens | GitLab](https://docs.gitlab.com/ee/user/profile/personal_access_tokens.html)
:::

### OAuth 2.0 Access Token (AT&RT)

RFC6749 で定義されている OAuth 2.0[^rfc6749]には、4 つの認可フローがあります[^qiita-oauth-flow]が、中でも GitLab で取り扱うのは、認可コードフロー(Authorization Code Flow)[^gitlab-oauth][^qiita-oauth-1][^qiita-oauth-2]です。
これを使った場合、主に **OAuth 2.0 Access Token(AT)と Refresh Token(RT)** の 2 つを使って Git サーバと通信しますが、予め認可サーバで**後述する専用アプリ(GCM)の登録が必要**です。
[^rfc6749]: https://openid-foundation-japan.github.io/rfc6749.ja.html
[^qiita-oauth-flow]: [OAuth 2.0 全フローの図解と動画 #OAuth - Qiita](https://qiita.com/TakahikoKawasaki/items/200951e5b5929f840a1f)
[^gitlab-oauth]: [OAuth 2.0 identity provider API | GitLab](https://docs.gitlab.com/ee/api/oauth2.html)
[^qiita-oauth-1]: [OAuth 2.0 の認可レスポンスとリダイレクトに関する説明 #OAuth - Qiita](https://qiita.com/TakahikoKawasaki/items/8567c80528da43c7e844)
[^qiita-oauth-2]: [OAuth 2.0 のシーケンスをまとめてみた #OAuth - Qiita](https://qiita.com/guromityan/items/4f8d8ed74276539e9fd1)

OAuth の例として、各種 SNS アカウントと任意のアプリ(Qiita や食べログ等)を連携する場合を考えましょう。連携ボタンをクリックすると、SNS の認証画面に飛びます。
そして、そこで許可するとリダイレクトされて、SNS のアカウントでアプリログインが完了するのを見た人も多いのではないでしょうか?

![Twitterを認可サーバとして食べログの認証をする場合の画面](/images/git-2fa-authn/tabelog-twitter-authz.png =400x)
_Twitter を認可サーバとして食べログの認証をする場合の画面_

これが OAuth による認証ですが、指示に従うだけで終わる点で、**PAT に比べれば簡単**です。
尚、Git の場合は、後述の GCM が必要ですが、その場合は任意のアプリがインターネット上のサービス(Qiita や食べログ等)ではなく、ローカルのアプリ(GCM)になります。認可サーバは、Git ホスティングサービスです。

しかし、課題として**RT による無期限化**があります。詳細は最後に纏めて述べます。

## 🔐 アクセストークンの管理方法

以下の記事で以前に紹介した内容ですが、簡単に復習します。
https://zenn.dev/miya789/articles/manager-core-for-two-factor-authentication

Git コマンドを扱う際に、パスワードならまだしも**アクセストークンを毎回入力するのは大変**なので、それを補助する以下の**helper ツール**が存在します。

| helper 名           | 関連リポジトリ                     | 保存先                              | AT&RT に対応可能か? |
| ------------------- | ---------------------------------- | ----------------------------------- | ------------------- |
| store               | Git for Windows / Git(built-in)    | テキストファイル                    | ❌                  |
| cache               | Git for Windows / Git(built-in)    | ソケットファイル                    | ❌                  |
| wincred/osxkeychain | Git for Windows / Git(built-in)    | 認証情報マネージャー / キーチェーン | ❌                  |
| manager(旧)         | Git Credential Manager for Windows | 認証情報マネージャー                | ✅(推測)            |
| manager             | Git Credential Manager (GCM)       | 認証情報マネージャー等              | ✅                  |

まず Git インストールで必ず入るのが`store`と`cache`で、Windows/Mac だと`wincred/osxkeychain`も利用可能です。
更にこれらに加えて Git for Windows インストール時には、`manager(GCM)`が推奨されデフォルトでインストールされます。
尚、先述した**AT&RT に対応可能なものは GCM 等のみ**です。

以上の通りで、アクセストークンの管理方法を簡単に紹介しましたが、**AT&RT も使える点で GCM が有用**です。

https://github.com/git-ecosystem/git-credential-manager

:::details OAuth 2.0 に於ける認可コードフロー (GitLab の場合)
GitLab の場合のシーケンス図は下図の通り。

![OAuth 2.0 に於ける認可コードフロー (GitLab の場合)](/images/git-2fa-authn/authorization-code.png)
_OAuth 2.0 に於ける認可コードフロー (GitLab の場合)_

GCM の場所には大抵、別の Web アプリケーションがいますが、今回はそれがローカルのアプリです。
謂わば、アプリケーション連携用アクセストークンを使って Git 通信を試みているようなものです。
今回はそのアプリケーションが、偶々 Git 通信をする用のものとなっています。
:::

## ⚠️ アクセストークンの課題

前項までで各アクセストークンの種類や管理方法を簡単に紹介しましたが、これらの課題を纏めます。
まず、GitLab で取り扱い可能な**アクセストークン(AT&RT と PAT)の比較**を行います。
そして最後に、**Git 経由の HTTPS 通信で使う認証情報の課題**を述べ、何れのアクセストークンでも解決できないことを確認します。

### アクセストークン(AT&RT と PAT)に関する長所短所の比較

GCM だけで取り扱い可能な**AT&RT**をメインに、他の helper でも取り扱い可能な**PAT**と比較します。

PAT に対する AT&RT の長所は、**発行が容易である点**と**自動ローテーションしてくれる点**です。
例えば PAT は、ブラウザでログインしてから設定ページを自分で開く必要があり、更にスコープ設定も必要でした。
しかし、**AT&RT であれば、GCM によって表示された UI に従ってブラウザで認証すれば、勝手に発行してくれるだけでなく管理もしてくれます**。
更に OAuth 2.0 の仕組みとして、有効期限の短い AT(アクセストークン)と有効期限が長い RT(リフレッシュトークン)を併用し、**2 時間経てばアクセストークンをローテーションすることで、通信経路で漏れた場合のリスクを抑えられます**。

一方で、PAT は有効期限が切れれば再認証が必要になりますが、AT&RT は**利用者が再認証する手間さえ省いてしまいます**。
これは、少なくとも GitLab で使っている doorkeeper は有効期限が無い[^doorkeeper-360]為です(RT の有効期限はプロバイダーによって異なる)。
[^doorkeeper-360]: https://github.com/doorkeeper-gem/doorkeeper/issues/360

以上から、比較結果は以下となります。
パスワードに関しては PAT に近いものの、漏洩時の影響が大きいです。

| 種類           | 初回発行時に 2 要素認証?<br>(Web UI 経由) | 発行は用意?                                        | 漏洩による影響の薄さ    | 再認証が必要になるか?                                               |
| -------------- | ----------------------------------------- | -------------------------------------------------- | ----------------------- | ------------------------------------------------------------------- |
| **パスワード** | -                                         | -                                                  | ❌ 影響範囲大           | 🤔 `helper`で保存してしまうと、2 回目以降で必要十分条件を満たさない |
| **PAT**        | ✅                                        | 🤔 自力で設定を開く必要<br>🤔 スコープも自分で設定 | 🤔 有効期限の設定が必要 | 🤔 `helper`で保存してしまうと、2 回目以降で必要十分条件を満たさない |
| **AT&RT**      | ✅                                        | ✅                                                 | ✅ 自動ローテーション   | ❌ RT を無効化しない限り、AT は自動更新                             |

以上から、**GitLab で使えるどんな種類の認証情報であっても、認証の必要十分条件すら満たさない**と分かりました。

### Git 経由の HTTPS 通信に於ける課題

**GitLab で備わっている 2FA**に関して纏めます。
これは GitLab に内蔵された機能で、例えば表示された QR コードを読み取って Google Authenticator 等に登録すれば、TIme-based One-time Password(TOTP)を 2 つ目の要素として使えます。
これにより、ブラウザでログインする際には、従来のユーザー名・パスワードだけでなく、この TOTP の PIN コードが必要になります。

一方の Git コマンド経由のアクセスでは TOTP が使えないので、パスワードとしてのアクセストークンで 1 要素認証します。
ここで確かに**事前のアクセストークン発行には 2 要素認証のブラウザ経由でアクセス**します。
従って**Git コマンドで通信を試みる度にブラウザ経由で認証できれば理想的**です。

しかし**アクセストークンは再認証の必要十分条件を満たさない**ので、**ブラウザ経由どころか認証すらしない可能性**があります。
即ち**Git コマンド経由のアクセスは認証不要の場合があり、2 要素認証だけでなく 1 要素認証ですらなくなる可能性がある**と判明しました。

ここまで登場した要素をアクセス経路に応じて纏めると、以下の通りです。

| アクセス経路         | 認証に使われる要素         |
| -------------------- | -------------------------- |
| **ブラウザ経由**     | 2 要素 (パスワード & TOTP) |
| **Git コマンド経由** | 1 要素 (アクセストークン)  |

ここで、**Git コマンド経由のアクセスを試みる度に、ブラウザ経由でアクセストークン発行**が可能であれば未だ勝算がありました。
しかし、ブラウザ経由のパスワードや TOTP は再認証の必要十分条件を満たすものの、**Git コマンド経由ではパスワードもアクセストークンもこれを満たさない**です。

| 要素                                    | 再認証の必要十分条件を満たすか?                     |
| --------------------------------------- | --------------------------------------------------- |
| **(ブラウザ経由) パスワード**           | ✅ 有効期限がある上に、ブラウザのセッションは切れる |
| **(ブラウザ経由) TOTP**                 | ✅ 有効期限が短い                                   |
| **(Git コマンド経由) パスワード**       | ❌ ① では不十分(後述の ② でも)                      |
| **(Git コマンド経由) アクセストークン** | ❌ ① では不十分(後述の ② でも)                      |

従って、**1 要素認証の Git 経由のアクセスは 0 要素認証**すら達成してしまいます。
これは、**Git コマンドで使う認証情報の helper には揮発性を損なわせる機能**があり、更には**アクセストークンが無期限の場合**もある為です。
尚、**Ultimate プランで且つ GCM による AT&RT 発行をさせない場合**は、有効期限と強制力を満たす認証方法になりますが、これも料金面で非現実的です。

以上から、特に**cache helper 以外を使った場合(①)は、以下の通りで全く必要十分条件を満たしません**。

![①では必要十分条件を満たさない](/images/git-2fa-authn/authz-condition-1.png =500x)
_① では必要十分条件を満たさない_

# ❌ ② Client: helper(cache)による 2 要素認証の検討

前項で、単に GitLab の 2FA だけだと条件を満たせないと分かりました。
そこで、**他のオプションによる揮発性の実現**を目指します。

## 🪟 Cache が使えないカス(Windows)環境

Windows 10 build 17061 以降であれば cache 対応可能なようですが、Vista サポートが続く限り非対応らしいです。

> Full disclosure: Technically, since Git v2.34.0,`git-credential-cache.exe`[_can_ be built and run on Windows](https://github.com/git/git/commit/c2e799012b3). Unix Sockets support [has been introduce into Windows](https://devblogs.microsoft.com/commandline/af_unix-comes-to-windows/). The problem is that you need Windows 10 build 17061 or later, and Git for Windows still supports even Vista (although [it has been announced that Git for Windows will drop supporting Vista really soon now](https://github.com/git-for-windows/build-extra/commit/c7e2c3cda90f4681400852d1207f74e96d8b1ff6)). [^gcm-issue-723-comment]

[^gcm-issue-723-comment]: https://github.com/GitCredentialManager/git-credential-manager/issues/723#issuecomment-1146817218

更に、GCM のメンテナーに対応していない事を指摘すると、Windows で cache を選べないようになりました。
従って、対応している環境で設定を有効にして Git をビルドしても、GCM のコードも変更してビルドしなければ cache は使えないです。
但し、GCM であろうと結局は内蔵の git-credential-cache を使うので、git-credential-cache と変わらず、更に**強制力が無い**ので、どちらにせよ**認証の条件を満たしません**。

## ⛏️ 他のオプションで対応

GCM であれば、Git に built-in されたものに限らず組み合わせることも可能ですが、cache 同様に**強制力を持ちません**。

## 🗒️ まとめ

以上から、本項の取り組み(②)は、揮発性を持つものの、**強制力を持たない**為に条件を満たしません。

![②でも、強制力を持たないが故に、必要十分条件を満たさない](/images/git-2fa-authn/authz-condition-2.png =500x)
_② でも、強制力を持たないが故に、必要十分条件を満たさない_

# 🤔 ③ Server: 定期的な無効化の検討

**前項までの案では強制力を全く持たず**、必要十分条件を満たす算段が付きませんでした。
従って、server 側で定期的に無効化することで、**強制的に全ユーザーに再認証を要求すること**を考えました。

具体的には、**期限の長い PAT や一日経った GCM 由来の AT&RT 等、望ましくないアクセストークンをスクリプトで無効化**します。
例えば、毎晩以下スクリプトを実行させることで、**定期的な GCM 由来の AT&RT の無効化**が可能です。

```bash:定期的な GCM 由来の AT&RT 無効化スクリプト
# GCM由来の場合は、token.application_id == 1
echo "OauthAccessToken.all.filter_map { |token| token.revoke if token.application_id == 1 && token.revoke? == flase }" > /var/opt/gitlab-rails/tmp/all_doorkeeper_access_tokens_revoked.rb
gitlab-rails runner /var/opt/gitlab-rails/tmp/all_doorkeeper_access_tokens_revoked.rb
```

このように、**疑似的な有効期限**を作ることで、**少なくとも毎日の初回ログインでは 2 要素認証が求められるようにできます**。
以上から、本項は必要十分条件を満たし得ると分かりました。

![③なら必要十分条件を満たす](/images/git-2fa-authn/authz-condition-3.png =500x)
_③ なら必要十分条件を満たす_

しかしやはり**有効期間内では 1,0 要素認証**になってしまう上に、一方で**間隔を狭めるとサーバに負担を掛ける可能性**があります。
また、**異なるタイムゾーンのユーザーに合わせて、無効化のタイミングをずらすと処理が複雑になる可能性**があります。
では、もう少し優れた方法を模索してみましょう。

# ✅ ④ Server: Reverse Proxy (SSO)

**③Server: 定期的に無効化(有効期限の自作)が最も現実的**と判明しました。
しかし、これもいくつかの課題を抱える為、別の案として**Reverse Proxy による SSO[^sso] の導入**を検討します。
尚、Git コマンドを対応させるには、`http.cookieFile`に対応させる必要がありました。
順に紹介します。
[^sso]: [シングルサインオン（SSO）の選び方と仕組みの解説 | アシスト](https://www.ashisuto.co.jp/product/theme/security/sso.html)

## ⛏️ Reverse Proxy (SSO)の実装

前項までで上げた案にはどれも課題がありましたが、**Client と Git Server 間に、SSO(Single Sign On)としての Reverse Proxy を挟む手**もあります。
![ClientとGit Serverの間に挟まるReverse Proxy](/images/git-2fa-authn/architecture.png =250x)
_Client と Git Server の間に挟まる Reverse Proxy_

この Reverse Proxy 認証で TOTP 等も使えれば、揮発性が無くても短い有効期限を獲得できるので、必要十分条件を満たす要素を得られ、2 要素認証が実現できます。

![④も必要十分条件を十分に満たす](/images/git-2fa-authn/authz-condition-4.png =500x)
_④ も必要十分条件を十分に満たす_

簡単に Reverse Proxy の PoC として、以下の「Form 認証を htpasswd ファイルで静的に行う方法」を参考にしてください。
https://github.com/miya789/git-via-reverse-proxy-authn

::::message
上に挙げた、「Form 認証を htpasswd ファイルで静的に行う方法」は**静的な実現方法**なので、**実用上は別の動的な方法に置き換える必要**があるかと思います。
この辺りは、**PAM(Pluggable Authenticaton Modules)** について調べてください。
気が向いたら PoC リポジトリも作りますが、あまり自分も分かってないです。

コンセプトの紹介だけに留めます。どなたか是非やってください……僕はもう疲れました。
::::

::::message
他には、Auth0 で Google 認証[^apache-auth0][^apache-google] 等と Apache を連携させる手もあるかと思います。
[^apache-auth0]: [【Auth0】Auth0+Apache(mod_auth_openidc)でシングルサインオンしてみる | DevelopersIO](https://dev.classmethod.jp/articles/auth0-apache-mod_auth_openidc/)
[^apache-google]: [認証機能のないアプリケーションで OAuth2 認証を提供する - 弥生開発者ブログ](https://tech-blog.yayoi-kk.co.jp/entry/2015/04/07/145743)

:::message alert
GCM の OAuth 2.0 と、Apache が連携する SSO の認証は別の話です。
Git Sever-Git Client 間ではなく、Reverse Proxy-Git Client 間の話です。アクセストークンとか関係無いです。
:::
::::

## ⛏️ git コマンドの Reverse Proxy 対応

但し、毎回 2 要素認証を求める認証は一般的ではなく、**Reverse Proxy 認証に git コマンドを対応させるのは難しい**です。
そこで、git コマンドの通信で、ヘッダーを指定する方法を探していると、読み込んだクッキーファイルを使って通信するオプション(`http.cookieFile`)[^git-config-cookie]を使うことで解決すると判明しました。
[^git-config-cookie]: https://git-scm.com/docs/git-config#Documentation/git-config.txt-httpcookieFile

従って、Client にはこのオプションを設定させ、**Reverse Proxy の認証で得られたセッション ID をクッキーファイルに書き込むアプリを自作**すれば、対応可能です。

:::message
実利用の際は、定期的に自動でリクエストを送り、有効期限を更新してくれると良いでしょう。
:::

## 🗒️ まとめ

Reverse Proxy 認証で再認証の必要十分条件を満たす認証が組み合わせられるので、2 要素認証が実現できます。
更に、Git コマンドはクッキーファイルの読み込み設定(`http.cookieFile`)があるので、これも対応可能です。

こうして、本稿の目的は達成できましたが、更に改善する為の手も考えてみましょう。

# ✅ ⑤ Server: Reverse Proxy (SSO) + ローカル認証情報の改善

前項の ④ でかなり 2 要素認証らしくなってきましたが、まだ改善できる点があるので、それ等に関して論じます。
主に、「ローカル認証情報の暗号化」と「アクセストークン置き換え」を検討します。

## 💭 ローカル認証情報の暗号化の検討

**パスワードをローカルに保存するのは危険**で、現在推奨されません[^remove-secret-from-local]。
[^remove-secret-from-local]: [1Password を使って、ローカルにファイル(~/.config や.env)として置かれてる生のパスワードなどを削除した | Web Scratch](https://efcl.info/2023/01/31/remove-secret-from-local/)

そこで、**ローカルの認証情報を暗号化して、追加認証を求める方法**も考えられます。
具体的には、1Password[^qiita-1password]の様な**ローカルの認証情報を暗号化して保存するアプリの自作**です。
しかし cache と同様に**クライアント側の設定は善意に基づくもの**であり、選択肢の一つに過ぎないので、強制力が無いです。
ですが、**強制力を持つ Reverse Proxy と組み合わせれば、より強力な手段**となります。
[^qiita-1password]: [1Password コマンドラインツール #command - Qiita](https://qiita.com/shimizumasaru/items/727426c8fd88542608e4)

## 💭 アクセストークン置き換えの検討

現状では**パスワードが通信で漏洩するリスク**や**git プロセスが保存された認証情報を読み込めてしまう課題**があります。
そこで、従来と同等の 2FA を実現する上では適わなかった GCM ですが、**発行や管理を容易にする点では有力**であったので、**GCM の活用**も検討すると良いでしょう。
**こちらも Reverse Proxy と組み合わせることで、威力をより発揮します**。

尚、幸いにも、今年になって筆者が ChatGPT 先生やメンテナのご協力を得て、GCM をクッキーファイルに対応させました[^gcm-pr-1251]
従って、④ の Git コマンドと同様に、Reverse Proxy を介した Git 通信であっても、GCM も対応可能です。

https://github.com/git-ecosystem/git-credential-manager/pull/1251

## 🗒️ まとめ

以上見て来た様に、**Reverse Proxy に加えて、更に別の方法も組み合わせることで更に改善可能で、全ての項目さえ満たし得るものとなります**。

![⑤であれば、全ての項目を満たし得る](/images/git-2fa-authn/authz-condition-5.png =500x)
_⑤ であれば、全ての項目を満たし得る_

# 🏃 補足

電気羊の小説、読んだのが 6 年前なのでかなり忘れてますが、気分を機械で制御しているのが衝撃的でした。
しかも暗い気分に敢えてしていた気がしますね。
とまぁこんな具合で、人間が機械によって感情も含めて管理されるような感じの世界観が印象的です。

話はかなり変わり、個人的に、人による人の統治(民主主義)は時代遅れなので、早く機械による統治を実現したいですね。
機械で且つ OSS であれば意思決定が明確になると思うんですよね。
まぁ勿論、民主主義と同じでデバッガーみたいな輩に脆弱性突かれると危険ではあるものの。

少なくとも、今生の私は、人間の自由意思は信頼できないという結論に至ったので、何とか人間から支配権を奪いたいです。
早く、根源へ至る為の次の段階へと、この螺旋を昇っていきたいですね。
皆さんも機械になら、されてみたくないですか? 管理。
それに、存外に早く訪れるかもしれませんよ。ユートピア。
