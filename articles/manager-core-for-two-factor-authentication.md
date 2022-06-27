---
title: "GitHub等へのHTTPS接続用に個人用アクセストークンを求めるのは間違っているだろうか"
emoji: "🔑"
type: "tech"
topics: [windows, gcm, github, gitlab, bitbucket]
published: true
published_at: 2022-07-01 08:00
---

# TL;DR

https://github.com/GitCredentialManager/git-credential-manager

- **[git-credential-manager](https://github.com/GitCredentialManager/git-credential-manager) (GCM)を使い、ブラウザ経由で 2 要素認証を突破しましょう。**
- GCM を介す事で、自分でアクセストークンを**発行**・**管理**する手間が省けます。
- Windows の場合は、[Git for Windows](https://gitforwindows.org/)インストール時に GCM も設定可能ですが、
  他のプラットフォームでも自分でインストールすれば動作するようです。

# アクセストークンの発行や管理を慎重に行ってますか?

最近、OSS への悪意あるコードの混入やアクセストークンの流出(Heroku)など、インシデントの頻度が上がっていますね。
GitHub に於いては、 2 要素認証(2FA)を義務化する動き[^github-20220513]も活発です。
そんな中、ネットワーク要件等の都合で、SSH ではなく HTTPS が推奨されている場合もあるのではないでしょうか? 実際、GitHub はより簡単な HTTPS を推奨しています[^github-set-up-git][^stackoverflow-11041729]。
[^github-set-up-git]: https://docs.github.com/en/get-started/quickstart/set-up-git#connecting-over-https-recommended

[^stackoverflow-11041729]: [git - Why does GitHub recommend HTTPS over SSH? - Stack Overflow](https://stackoverflow.com/questions/11041729/why-does-github-recommend-https-over-ssh/11041782)
[^github-20220513]: [開発者のアカウントを 2 要素認証(2FA)で保護 - GitHub ブログ](https://github.blog/jp/2022-05-13-software-security-starts-with-the-developer-securing-developer-accounts-with-2fa/)

ですが、この HTTPS 接続の場合、パスワード認証が廃止され始めています。
例えば GitHub や GitLab は、以前から 2 要素認証が有効だとパスワード認証不可で、
更に GitHub は 2021/08/13 以降[^github-20201215]、Bitbucket は 2022/03/01 以降[^bitbucket-20220217]で、廃止されました。
従って、現状でパスワード認証可能なのは**2 要素認証が有効でない GitLab アカウントのみ**となりました。
[^github-20201215]: [Token authentication requirements for Git operations | The GitHub Blog](https://github.blog/2020-12-15-token-authentication-requirements-for-git-operations/)
[^bitbucket-20220217]: [Announcement: Bitbucket Cloud account password usa... - Atlassian Community](https://community.atlassian.com/t5/Bitbucket-articles/Announcement-Bitbucket-Cloud-account-password-usage-for-Git-over/ba-p/1948231)

ですが、`git clone` 等で以下の画面が出てきて初めて、以前使っていたパスワード認証が使えなくなった事に気づき、焦った方もいるのではないでしょうか?

```bash:HTTP越しにパスワード認証でエラーが出る例(GitHub)
$ git clone https://github.com/sample-user/private-repository.git
Cloning into 'private-repository'...
error: unable to read askpass response from 'C:/Program Files/Git/mingw64/bin/git-askpass.exe'
Username for 'https://github.com': sample-user
remote: Support for password authentication was removed on August 13, 2021. Please use a personal access token instead.
remote: Please see https://github.blog/2020-12-15-token-authentication-requirements-for-git-operations/ for more information.
fatal: Authentication failed for 'https://github.com/sample-user/private-repository.git/'
```

:::details Bitbucket の場合

```bash:HTTP越しにパスワード認証でエラーが出る例(Bitbucket)
$ git clone https://sample-user@bitbucket.org/sample-user/private-repository.git
Cloning into 'private-repository'...
fatal: Invalid credentials
Password for 'https://sample-user@bitbucket.org':
remote: Bitbucket Cloud recently stopped supporting account passwords for Git authentication.
remote: See our community post for more details: https://atlassian.community/t5/x/x/ba-p/1948231
remote: App passwords are recommended for most use cases and can be created in your Personal settings:
remote: https://bitbucket.org/account/settings/app-passwords/
fatal: Authentication failed for 'https://bitbucket.org/sample-user/private-repository.git/'
```

:::

そして慌てて「GitHub HTTPS clone できない」や「GitLab 2 要素認証 clone できない」等で検索し、検索結果に出て来た怪しい記事を鵜呑みにし、深く考えずにチェックを入れ、個人用のアクセストークンを発行していないでしょうか?
きちんと**有効期限**や**スコープ**の 2 点を精査していますか?

- **スコープ**
  公式ドキュメントでは特に言及されていませんが、
  後述の GCM では `'write_repository'` と `'read_repository'` のみなので、
  全てのスコープは与えるのは止めた方が良いでしょう。
  API 用途ならまだしも、`git pull`や`git push`には不要だと思います。

  :::message
  偶に「全てのスコープにチェックを入れれば OK」と語る無責任な怪しいサイトもありますが。(最小権限の原則とは 🤔、、、)
  まぁ、保存方法とアクセス経路に対して、絶対の自信があるなら問題無いんですかね?
  :::

- **有効期限**
  入力が面倒と言って、無期限で発行してしまっていませんか?

  とは言え、後述の GCM では期限を設定できないようです……
  OAuth2 の仕様だと少なくとも rotate はするようですが、GitHub に至ってはリフレッシュできているのかすら確認できていません。

  :::details 各種サービスのアクセストークン有効期限
  GitHub だとデフォルトが 30 日間ですが、GitLab だと無期限なので、そのまま使ってしまう人もいるかもしれません。
  そこで**最大の有効期限**が重要です。

  GitHub なら 1 年間使用されなかったアクセストークンは無効化されるようです[^github-authentication-keeping]。
  [^github-authentication-keeping]: [Token expiration and revocation - GitHub Docs](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/token-expiration-and-revocation)

  しかし、GitLab では Ultimate コースでないと最大の有効期限は設定できません[^gitlab-limit-the-lifetime-of-access-tokens]。
  [^gitlab-limit-the-lifetime-of-access-tokens]: https://docs.gitlab.com/ee/user/admin_area/settings/account_and_limit_settings.html#limit-the-lifetime-of-access-tokens
  :::

アクセストークンは、流出に関して話題になっており、扱いに気を付ける必要が増しています。
最近だと、Heroku で OAuth アクセストークンが流出した事故[^heroku-incident]もあります。
しかしだからと言って、毎回発行するのも大変ですし、平文で管理しては意味がありません。
[^heroku-incident]: [Heroku の OAuth トークン流出で、やっておくといいことリスト（コメント大歓迎）](https://zenn.dev/hiroga/articles/heroku-incident-2413-checklist)

では、GitHub 公式の見解はどうでしょうか?
エラーメッセージでも表示される GitHub のドキュメントでは、
個人のアクセストークン(PAT: Personal Access Token)を発行する事が推奨されています[^github-20201215]。
従って、少なくともアクセストークンの使用は避けられなさそうです。

:::message
但し、GitHub の PAT 作成ページ[^github-authentication]では、更に後述の GCM が推奨されています[^github-caching]。
[^github-authentication]: [Creating a personal access token - GitHub Docs](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token)
[^github-caching]: https://docs.github.com/en/get-started/getting-started-with-git/caching-your-github-credentials-in-git#git-credential-manager
:::

では、**どのようにアクセストークンを発行・管理するのが望ましいでしょうか?**

# アクセストークンの管理方法

主に以下があります。

| helper 名                 | 関連リポジトリ                                                                                               | 補足                                                         |
| ------------------------- | ------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------ |
| `store`                   | [Git for Windows](https://github.com/git-for-windows/git) / [Git](https://github.com/git/git/)<br>(built-in) | 平文保存[^git-tools-credential-storage]                      |
| `cache`                   | [Git for Windows](https://github.com/git-for-windows/git) / [Git](https://github.com/git/git/)<br>(built-in) | メモリに保存 <br> Unix socket を使用するが、Windows が非対応 |
| `wincred` / `osxkeychain` | [Git for Windows](https://github.com/git-for-windows/git) / [Git](https://github.com/git/git/)<br>(built-in) | Windows の場合: <br> 「資格情報マネージャー」を直接操作可能  |
| `manager`                 | [Git Credential Manager for Windows](https://github.com/microsoft/Git-Credential-Manager-for-Windows)        | アーカイブ済み<br> Mac や Linux 用のものも同様               |
| `manager-core`            | [Git Credential Manager (GCM)](https://github.com/GitCredentialManager/git-credential-manager)               | helper 名には core が残っているが、名称からは削除済み        |

[^git-tools-credential-storage]: [Git - 認証情報の保存](https://git-scm.com/book/ja/v2/Git-%E3%81%AE%E3%81%95%E3%81%BE%E3%81%96%E3%81%BE%E3%81%AA%E3%83%84%E3%83%BC%E3%83%AB-%E8%AA%8D%E8%A8%BC%E6%83%85%E5%A0%B1%E3%81%AE%E4%BF%9D%E5%AD%98)

::::details helper 名とは?🤔
以下の設定で使う事になる名前です。

```bash:configの例(.gitconfig等)
git config --global credential.helper ${helper 名}
```

:::message
因みに、ここで設定したものが、
`git pull`等を実行した際に`git credential-${helper 名} get`の形式で呼ばれ、認証情報が使われます。

```bash:初回
$ GIT_TRACE=1 git clone https://github.com/sample-user/private-repository.git
14:44:06.946257 git.c:439               trace: built-in: git clone https://github.com/sample-user/private-repository.git
Cloning into 'private-repository'...
14:44:06.948359 run-command.c:663       trace: run_command: git-remote-https origin https://github.com/sample-user/private-repository.git
14:44:07.154153 run-command.c:663       trace: run_command: '/mnt/c/Program\ Files/Git/mingw64/bin/git-credential-manager-core.exe get'
info: please complete authentication in your browser...
14:44:14.302420 run-command.c:663       trace: run_command: '/mnt/c/Program\ Files/Git/mingw64/bin/git-credential-manager-core.exe store'
14:44:14.755413 run-command.c:663       trace: run_command: git fetch-pack --stateless-rpc --stdin --lock-pack --thin --check-self-contained-and-connected --cloning https://github.com/sample-user/private-repository.git/
14:44:14.756225 git.c:439               trace: built-in: git fetch-pack --stateless-rpc --stdin --lock-pack --thin --check-self-contained-and-connected --cloning https://github.com/sample-user/private-repository.git/
remote: Enumerating objects: 300, done.
remote: Counting objects: 100% (191/191), done.
remote: Compressing objects: 100% (122/122), done.
14:44:15.025907 run-command.c:663       trace: run_command: git index-pack --stdin -v --fix-thin '--keep=fetch-pack 357 on XXXXX' --check-self-contained-and-connected --pack_header=2,300
14:44:15.029642 git.c:439               trace: built-in: git index-pack --stdin -v --fix-thin '--keep=fetch-pack 357 on XXXXX' --check-self-contained-and-connected --pack_header=2,300
remote: Total 300 (delta 112), reused 145 (delta 69), pack-reused 109
Receiving objects: 100% (300/300), 97.78 KiB | 1.96 MiB/s, done.
Resolving deltas: 100% (176/176), done.
14:44:15.089306 run-command.c:663       trace: run_command: git rev-list --objects --stdin --not --all --quiet --alternate-refs '--progress=Checking connectivity'
14:44:15.089912 git.c:439               trace: built-in: git rev-list --objects --stdin --not --all --quiet --alternate-refs '--progress=Checking connectivity'
```

```bash:二回目以降
$ GIT_TRACE=1 git clone https://github.com/sample-user/private-repository.git
20:25:05.389305 git.c:439               trace: built-in: git clone https://github.com/sample-user/private-repository.git
Cloning into 'private-repository'...
20:25:05.582959 run-command.c:663       trace: run_command: git-remote-https origin https://github.com/sample-user/private-repository.git
20:25:05.876572 run-command.c:663       trace: run_command: '/mnt/c/Program\ Files/Git/mingw64/bin/git-credential-manager-core.exe get'
20:25:06.458265 run-command.c:663       trace: run_command: '/mnt/c/Program\ Files/Git/mingw64/bin/git-credential-manager-core.exe store'
20:25:06.778699 run-command.c:663       trace: run_command: git fetch-pack --stateless-rpc --stdin --lock-pack --thin --check-self-contained-and-connected --cloning https://github.com/sample-user/private-repository.git/
20:25:06.794027 git.c:439               trace: built-in: git fetch-pack --stateless-rpc --stdin --lock-pack --thin --check-self-contained-and-connected --cloning https://github.com/sample-user/private-repository.git/
remote: Enumerating objects: 300, done.
remote: Counting objects: 100% (191/191), done.
remote: Compressing objects: 100% (122/122), done.
20:25:07.075604 run-command.c:663       trace: run_command: git index-pack --stdin -v --fix-thin '--keep=fetch-pack 785 on XXXXX' --check-self-contained-and-connected --pack_header=2,300
20:25:07.101286 git.c:439               trace: built-in: git index-pack --stdin -v --fix-thin '--keep=fetch-pack 785 on XXXXX' --check-self-contained-and-connected --pack_header=2,300
remote: Total 300 (delta 112), reused 145 (delta 69), pack-reused 109
Receiving objects: 100% (300/300), 97.78 KiB | 4.07 MiB/s, done.
Resolving deltas: 100% (176/176), done.
20:25:07.190765 run-command.c:663       trace: run_command: git rev-list --objects --stdin --not --all --quiet --alternate-refs '--progress=Checking connectivity'
20:25:07.198244 git.c:439               trace: built-in: git rev-list --objects --stdin --not --all --quiet --alternate-refs '--progress=Checking connectivity'
```

詳しくは
https://git-scm.com/book/ja/v2/Git-%E3%81%AE%E3%81%95%E3%81%BE%E3%81%96%E3%81%BE%E3%81%AA%E3%83%84%E3%83%BC%E3%83%AB-%E8%AA%8D%E8%A8%BC%E6%83%85%E5%A0%B1%E3%81%AE%E4%BF%9D%E5%AD%98
:::
::::

まず、Git をインストールすると、built-in で入っているのが、`store`, `cache` で、
更に Windows / Mac だと `wincred` / `osxkeychain` も利用可能です[^git-tools-credential-storage]。
では上から順に見てみましょう。

- `store`
  平文で保存します。
  デフォルトの保存先は、`~/.git-credentials`です。

- `cache`

  > cache ヘルパーは独自形式でメモリーに情報を保持します
  > （他のプロセスはこの情報にアクセスできません）[^git-tools-credential-storage]。

  このように紹介されており、安全らしいです。

  :::message
  `strace` でソケット?は覗けるらしいですが、自分は解読方法が分からないです……
  独自形式で保持して、Git のプロセス以外に共有されていないから安全なんですかね?
  :::

- `manager`

  古い記事だとこれの事しか書いてないですが、
  既に `manager-core` に統合され、リポジトリはアーカイブ済みです。
  Mac や Linux 用のものも同様です。
  :::message alert
  `manager` はアーカイブ済みであり、公式に `manager-core` で代替するようにアナウンスされているので、これを使いましょう。
  (Windows の場合は、Git を入れるだけで済みますが)
  名前が紛らわしく、未だに `manager` を推奨しているネット記事が残っていますが、
  **`manager-core`の正式名**は、[**Git Credential Manager (GCM)**](https://github.com/GitCredentialManager/git-credential-manager)なので注意しましょう。
  :::

- `magaer-core`

  初めてアクセスする Git ホスティングサービス(GitHub 等)に対して、
  `git pull` / `git push` 等を行うと、
  以下の様な UI が表示され、これに従ってブラウザ経由で認証できます。
  ![GCMによって表示される認証用のUI](/images/manager-core-for-two-factor-authentication/gcm-ui.png)
  _GCM によって表示される認証用の UI[^github-20220407]_

  尚、一度 GCM によるアクセスを認可したサービスでは、今後アクセストークンを要求される事はありません。
  [^github-20220407]: [Git Credential Manager: authentication for everyone | The GitHub Blog](https://github.blog/2022-04-07-git-credential-manager-authentication-for-everyone/)

  :::message
  アクセストークンの保存先に関して、
  Windows の場合は「資格情報マネージャー」なので、削除したい場合はここから操作可能です。
  :::

  ::: message
  各種の Git ホスティングサービス内の設定で GCM を無効化すると、
  再び認可が必要になりますので、UI も再び表示されます。
  :::

**アクセストークン管理**の選択肢を簡単に確認してきましたが。いかがでしたでしょうか?
**アクセストークン発行**も容易になる点では、GCM に軍配が上がりそうです。
ではここで、最近推奨され始めている**GCM**に焦点を移してみましょう。

# Git Credential Manager (GCM)

https://github.blog/2022-04-07-git-credential-manager-authentication-for-everyone/
2022/04/07 に GitHub から公式にアナウンスされたばかりのものです。
ポイントとしては、以下です。

- 以前までの [GCM for Windows](https://github.com/microsoft/git-credential-manager-for-windows) and [GCM for Mac and Linux](https://github.com/microsoft/git-credential-manager-for-mac-and-linux) を統合
  - [Avalonia UI](https://avaloniaui.net/)により、.NET でも macOS や Linux に対応
  - WSL の場合は、実行パスを Windows 側の物を設定すれば、設定の共有可能
- 全てのプラットフォームに提供する思想に基づくので、
  microsoft や github ではなく GCM のプロジェクトとして独立
- Web UI の指示に従って認可すると、
  裏でアクセストークンが発行され、更にローカルへ良い感じで保存してくれる
- GitLab も対応
  - オンプレミス環境の場合は別途設定が必要[^gcm-gitlab]
- 認証情報に関して、様々な保存方法を設定可能(GPG 暗号化ファイル、キャッシュ)

:::message  
Azure Devops 関連はちょっと知らないです…
:::

[^gcm-gitlab]: https://github.com/GitCredentialManager/git-credential-manager/blob/main/docs/gitlab.md

では機能について確認してみましょう。

## UI からブラウザ経由で OAuth アクセストークンを発行

まず、Windows の場合は、Git をインストールする際にインストールされます。
macOS や Linux の場合は別途ダウンロードして、公式の手順[^gcm]に従ってインストールしてください。
[^gcm]: https://github.com/GitCredentialManager/git-credential-manager

あとは `git pull` / `git push` を実行する度に以下の UI が表示されて使えるようになります。
![GCMによって表示される認証用のUI](/images/manager-core-for-two-factor-authentication/gcm-ui-github.png)
_GCM によって表示される認証用の UI_

::::message
因みに、ここでアクセストークンを発行して保存すると、
その後は**自分で無効化しない限り**認証不要になります。

GitLab の場合、実際はアクセストークン自体変わっているのですが、裏で一緒に保存されたリフレッシュトークンがアクセストークンを更新してくれています。
:::details 🤔 アクセストークンの中身は?
オンプレミス版 GitLab の場合は以下のように確認可能です。

```bash:オンプレミス版 GitLab のデータベースにアクセスしてOAuth2アクセストークンを確認する手順
sudo gitlab-rails dbconsole --database main
gitlabhq_production=> select * from oauth_access_tokens;
 id | resource_owner_id | application_id |                              token                               |                          refresh_token                           | expires_in | revoked_at |  created_at                |  scopes
----+-------------------+----------------+------------------------------------------------------------------+------------------------------------------------------------------+------------+------------+----------------------------+-----------
  1 |                 2 |              1 | xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx | xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx |            |            | 2021-11-09 05:17:26.984175 | read_user
(1 row)
```

:::
::::

::::details 🤔OAuth とは?
ここで説明するには難しいので、調べてください。
因みに、OAuth と OAuth2.0、更に OpenID Connect というのがあるらしいです。

一般的に、**利用者が自ら無効化しない限り** リフレッシュトークンは有効らしいです。
Google の場合はログインしてないと切れるらしいですね。
https://www.cdatablog.jp/entry/gcprefreshtokengrant

:::details 🤔GitLab ではどう実装されている?
GitLab の場合は、個人アクセストークンではなく、自動でアクセストークン更新可能な OAuth2 のアクセストークンを使います。
内部の実装には Doorkeeper を使ってますが、リフレッシュトークンの有効期限が無いようです[^doorkeeper-refresh-expiration]。
[^doorkeeper-refresh-expiration]: https://github.com/doorkeeper-gem/doorkeeper/wiki/Customizing-Token-Expiration#refresh-token

最近になって必要では?とのコメントがあり、👍 はありますが進展が無いですね。
https://github.com/doorkeeper-gem/doorkeeper/issues/360
:::
::::

## 様々な管理方法のオプション

@[card](https://github.com/GitCredentialManager/git-credential-manager/blob/main/docs/credstores.md)
[アクセストークンの管理方法](#アクセストークンの管理方法)で触れたように様々な helper が存在しますが、
`manager-core`はこれらを組み合わせることも可能です。
一覧は以下ですが、詳細はリンク先から確認してください。

| 設定名          | 説明                                                                                         | Windows | Mac | Linux |
| --------------- | -------------------------------------------------------------------------------------------- | ------- | --- | ----- |
| `wincredman`    | Windows Credential Manager                                                                   | ✅      |     |       |
| `dpapi`         | DPAPI protected files                                                                        | ✅      |     |       |
| `keychain`      | macOS Keychain                                                                               |         | ✅  |       |
| `secretservice` | [freedesktop.org Secret Service API](https://specifications.freedesktop.org/secret-service/) |         |     | ✅    |
| `gpg`           | GPG/[`pass`](https://www.passwordstore.org/) compatible files                                |         | ✅  | ✅    |
| `cache`         | Git's built-in [credential cache](https://git-scm.com/docs/git-credential-cache)             |         | ✅  | ✅    |
| `plaintext`     | Plaintext files                                                                              | ✅      | ✅  | ✅    |

以下の形式で設定しますが、最新の方法は公式ドキュメントから確認してください。

```bash:管理方法に関するオプションの設定方法
export GCM_CREDENTIAL_STORE=xxx
# or
git config --global credential.credentialStore xxx
```

:::message alert
`cache`は Git 内蔵のものを使っていたりする一方で、
`store`は GCM 独自のパスに保存していたりしており、
**厳密には GCM 用にカスタマイズされてます。**

設定としては、あくまで `git config --global credential.help "manager-core"` があり、
そのオプションとして `git config --global credential.cacheOptions "--timeout 300"`があります。
従って、単に `git config --global credential.credentialStore cache && git config --global credential.helper cache --timeout 300` と設定するのとは異なります。

同じ事を、設定項目でも比較すると以下のようになります。

```conf:従来のcache設定
[credential]
  helper "cache --timeout 300"
```

```conf:GCM用のcache設定
[credential]
  credentialStore cache
  cacheOptions "--timeout 300"
```

:::

## WSL から Windows と資格情報の共有

@[card](https://github.com/GitCredentialManager/git-credential-manager/blob/main/docs/wsl.md)

WSL 内に直接 GCM をインストールすると、Windows の資格情報にアクセスできません。
従って、WSL 内の設定では、Windows 側の GCM の実行パスを設定して、資格情報の共有を可能にしましょう。

```bash
git config --global credential.helper "/mnt/c/Program\ Files/Git/mingw64/bin/git-credential-manager-core.exe"
# If you intend to use Azure DevOps you must also set the following Git configuration inside of your WSL installation.
git config --global credential.https://dev.azure.com.useHttpPath true
```

## キャッシュでアクセストークンを Local に残さない

GCM で発行・管理しても、アクセストークンを認証情報マネージャーで保管してるだけで、実質 1 要素認証のままじゃない? と思うかもしれません。
そこでキャッシュをお勧めします。

これなら無効化しなくても、発行元ぐらいしかトークンを知らないので漏れる心配はありません。
また、キャッシュが切れる度に GCM を介してブラウザ経由の 2 要素認証が求められるようになります。
因みにこれは、内蔵の Git の `credential-cache` を使用して実現しています。

:::message alert
Windows 向けの Git は、 Unix Sockets に対応していないので、キャッシュは使えません。

正確には、Git には Windows でも Unix Sockets をサポートする設定が存在していますが、無効化されて配布されています。
自分でビルドすれば可能らしいですが、コントリビューターのレビューで、対応するまではエラー終了するようになりました[^gcm-pr-729]。
公式に Git でサポートされるのを待ちましょう。
待てない場合は、自分でビルドしようとのことでした。
[^gcm-pr-729]: https://github.com/GitCredentialManager/git-credential-manager/pull/729
:::

## 従来方法との比較

本当は、
**「GCM 導入によって 2 要素認証を経由してアクセスできるので、セキュリティ的に改善する」**
と言おうと思ってましたが、無理でした。
OAuth2.0 のリフレッシュトークンでアクセストークンを入れ替えていてもタイムアウトしないので、一度認証するとそれ以降は永久に認証不要なんですよね。

従って、**毎回ログインしてから短い有効期限のアクセストークンを発行**して使えば、正直 GCM よりもセキュリティ的には良さそうでした。
但し、**発行や管理の手間を省く点では GCM が便利**そうです。

どうしてもアクセストークンを使い回したくない場合は、
定期的に自分で GCM のアクセストークンを無効化して再発行させるしかなさそうです。
::::message
オンプレミス版 GitLab なら、以下コマンドの自動実行で無効化して、2 要素認証を強要する運用もあり得ます。

```bash
# token.application_id == 1がGCMの場合
gitlab-rails runner "OauthAccessToken.all.filter_map { | token | token.revoke if token.application_id == 1 && token.revoked? == false }"
```

::::

# Q&A

想定されるものを簡単に纏めました。
再掲事項も含みます。

## 🤔 個人アクセストークン(PAT)と GCM 用の OAuth アクセストークンって本当に違うもの?

GitLab に関しては恐らくそうです。
但し、GitHub では、GCM が OAuth Application と言い張っている一方で、
次項で述べる様に、仕様がそうなっていないようなので、分からないです。

## 🤔 GCM のアクセストークンは OAuth として発行されている?

保存された認証情報を見る限り、GitLab や Bitbucket の場合は恐らくそうですが、GitHub では怪しかったです。

まず GitLab は、実際に保存されたアクセストークンを確認すると以下の様に 2 種類保存されています。
![認証情報マネージャーに保存された、GitLabの認証情報](/images/manager-core-for-two-factor-authentication/gcm-credential-manager-gitlab.png)
_認証情報マネージャーに保存された、GitLab の認証情報_

また、Bitbucket もきちんと別で保持しているようでしたので、恐らく GitLab と同じ仕様と思われます。
![認証情報マネージャーに保存された、Bitbucketの認証情報](/images/manager-core-for-two-factor-authentication/gcm-credential-manager-bitbucket.png)
_認証情報マネージャーに保存された、Bitbucket の認証情報_

一方で、GitHub ではリフレッシュトークンと思しきものは確認されませんでした。
違反[^github-oauth-violation]してるんですかね?
![認証情報マネージャーに保存された、GitHubの認証情報](/images/manager-core-for-two-factor-authentication/gcm-credential-manager-github.png)
_認証情報マネージャーに保存された、GitHub の認証情報_

[^github-oauth-violation]: [GitHub の OAuth 実装の仕様違反とセキュリティ上の考慮事項 - Qiita](https://qiita.com/TakahikoKawasaki/items/f1905f6a346f6ecc524f)

:::message
GitHub のドキュメント、構造が分かり難くて理解し切れていません……
自分自身、OAuth の仕様理解自体が疎かなので調査しておきます……
:::

## 🤔 GCM よりも、2 要素認証でログインしてから必ず有効期限付きで PAT 発行した方が安全では?

はい、恐らくその通りです。
OAuth2 の場合は、頻繁にアクセストークンは変わる筈なので、
アクセストークン漏洩は恐らく軽症で済みますが、リフレッシュトークン漏洩は想定されていないと思います。

::: message
GitHub は、リフレッシュトークンが使われているのかは把握できていません。
(自分で オンプレミス環境 GitLab へ GCM を登録する際は、アクセストークンの期限を 2 時間にするオプションにチェックする必要があります。)
:::

また、PAT は有効期限を詳細に決められるので、その点で PAT の方が優れていると思われます。
勿論、アクセストークン自体は、
Windows なら認証情報マネージャー(`wincred`) (mac なら、`osxkeychain`?)
に保存しておけば、安心でしょう。

## 🤔 PC に保存したアクセストークンは覗ける?

Windows の場合、資格情報マネージャーから表示できるらしいですが、自分の環境では無理でした。

一方で、Git コマンドによるアクセス時に、アクセストークンは復号?されるようでした。
例えば、以下の入力例のように入力すると、出力例のように続きが出力され、アクセストークンがパスワードとして平文で丸裸になります。
パケットキャプチャまではできていないので、通信時に平文で送信しているかまでは把握できていないですが……

```bash:入力例
$ git credential fill <<EOS
protocol=https
host=github.com
EOS
```

```bash:出力例
$ git credential fill <<EOS
protocol=https
host=github.com
EOS
protocol=https
host=github.com
username=sample-user
password=${sample-userのアクセストークン}
```

## 🤔 初回アクセス時は必ず 2 要素認証でログインしたいんだが?

案としては、以下があります。

1. 自分で GCM の認証を無効化(Revoke)
2. キャッシュにより、PC に残さない

但し、1 は手間ですし、
2 に関しては [Git for Windows](https://github.com/git-for-windows/git) が対応していないので Windows のみ無理です。

## 🤔 "`credential-cache` on Windows"は可能か?

> Full disclosure: Technically, since Git v2.34.0, `git-credential-cache.exe` [_can_ be built and run on Windows](https://github.com/git/git/commit/c2e799012b3). Unix Sockets support [has been introduce into Windows](https://devblogs.microsoft.com/commandline/af_unix-comes-to-windows/). The problem is that you need Windows 10 build 17061 or later, and Git for Windows still supports even Vista (although [it has been announced that Git for Windows will drop supporting Vista really soon now](https://github.com/git-for-windows/build-extra/commit/c7e2c3cda90f4681400852d1207f74e96d8b1ff6)). [^gcm-issue-723-comment]

[^gcm-issue-723-comment]: https://github.com/GitCredentialManager/git-credential-manager/issues/723#issuecomment-1146817218

[Git for Windows](https://github.com/git-for-windows/git) 自体は、v2.34.0 から Unix Sockets に対応しましたが、
後方互換の為に、`NO_UNIX_SOCKETS = YesPlease` として無効化されています。
Vista のサポートは 2023 年より前に打ち切るとは伺えましたが、古いビルドバージョンのサポートは残っているので、公式に対応してくれる見通しは未だ無いです。
(ドキュメントから Windows は消して、エラーメッセージも出力する事になりました[^git-credential-manager-pull-729]😩)
[^git-credential-manager-pull-729]: [Remove windows from `credential-cache` in `credstores.md` by miya789 · Pull Request #729 · GitCredentialManager/git-credential-manager](https://github.com/GitCredentialManager/git-credential-manager/pull/729)

従って、どうしても使いたい場合は、Git や GCM をビルドして使いましょう。

## 🤔 Linux 環境で GCM の UI が表示されないが?

「Avalonia により.NET でも Linux 対応」とあるにも拘わらず、UI が出て来なくて焦る場合もあると思います。
これは、`LANG`を `en_US.UTF-8`以外に設定していると発生する、Avalonia 由来の仕様です。

https://github.com/AvaloniaUI/Avalonia/issues/4427

## 🤔GCM はどのように OAuth2 アプリケーションとして UI を表示して動作している?

以下の様に、ホスト名で判別しています。
なので、これに該当しないオンプレミス環境 GitLab は `generic` として認識され UI が表示されません。
https://github.com/GitCredentialManager/git-credential-manager/blob/v2.0.779/src/shared/GitLab/GitLabConstants.cs#L48

因みに、`OAuthClientId` や `OAuthClientSecret` もソースコードに埋め込まれており、
他の設定値もここに記載されています。
https://github.com/GitCredentialManager/git-credential-manager/blob/v2.0.779/src/shared/GitLab/GitLabConstants.cs#L7-L13

## 🤔 オンプレミス環境の GitLab 対応はどうする?

@[card](https://github.com/GitCredentialManager/git-credential-manager/blob/main/docs/gitlab.md)

基本的には、上記リンクの指示に従います。
一部抜粋して和訳したものを以下に記します。

> ## 別インスタンスでの使い方
>
> 別インスタンスで使う為に、例えば `https://gitlab.example.com` では以下の設定が必要です。
>
> 1. [Create an OAuth application](https://docs.gitlab.com/ee/integration/oauth_provider.html). これは、ユーザーやグループ、インスタンスのレベルで行えます。
>    まず、名前やリダイレクト先を `http://127.0.0.1/` で指定してください。
>    そして、'Confidential' オプションは選択せず、'Expire access tokens' オプションは選択してください。
>    最後に、'write_repository' や 'read_repository' のスコープを設定してください。
> 1. application ID をコピーして、`git config --global credential.https://gitlab.example.com.GitLabDevClientId <APPLICATION_ID>` で設定してください。
> 1. application secret をコピーして、`git config --global credential.https://gitlab.example.com.GitLabDevClientSecret <APPLICATION_SECRET>` で設定してください。
> 1. `git config --global credential.https://gitlab.example.com.gitLabAuthModes browser` のように'browser'を含めて authentication modes を設定してください。
> 1. 念の為に、`git config --global credential.https://gitlab.example.com.provider gitlab` を設定してください。これは、ドメインを GitLab インスタンスとして認識させるのに必要かもしれません。
> 1. `git config --global --get-urlmatch credential https://gitlab.example.com` で、設定が期待通りかどうか確認してください。

:::message
今後のアップデートで自動対応する可能性もあります。
:::

## 🤔HTTPS 非対応のオンプレミス環境 GitLab でも使える?

無理です。エラーメッセージがそう言ってました。

```bash:HTTPSでないとGitLabとして処理できないとエラー
$ GIT_TRACE=1 git clone http://gitlab.example.com/sample-user/private-repository.git
22:28:30.042786 exec-cmd.c:237          trace: resolved executable dir: C:/Program Files/Git/mingw64/bin
22:28:30.043790 git.c:459               trace: built-in: git clone http://gitlab.example.com/sample-user/private-repository.git
Cloning into 'private-repository'...
22:28:30.053785 run-command.c:654       trace: run_command: git remote-http origin http://gitlab.example.com/sample-user/private-repository.git
22:28:30.060786 exec-cmd.c:237          trace: resolved executable dir: C:/Program Files/Git/mingw64/libexec/git-core
22:28:30.061786 git.c:748               trace: exec: git-remote-http origin http://gitlab.example.com/sample-user/private-repository.git
22:28:30.061786 run-command.c:654       trace: run_command: git-remote-http origin http://gitlab.example.com/sample-user/private-repository.git
22:28:30.069786 exec-cmd.c:237          trace: resolved executable dir: C:/Program Files/Git/mingw64/libexec/git-core
22:28:30.253464 run-command.c:654       trace: run_command: 'C:/Program\ Files/Git/mingw64/bin/git-credential-manager-core.exe get'
fatal: Unencrypted HTTP is not supported for GitLab. Ensure the repository remote URL is using HTTPS.
22:28:30.515517 run-command.c:654       trace: run_command: 'C:/Program Files/Git/mingw64/bin/git-askpass.exe' 'Username for '\''http://gitlab.example.com'\'': '
error: unable to read askpass response from 'C:/Program Files/Git/mingw64/bin/git-askpass.exe'
22:28:39.575230 run-command.c:654       trace: run_command: bash -c 'cat >/dev/tty && read -r line </dev/tty && echo "$line"'
Username for 'http://gitlab.example.com':
```

# 補足

https://twitter.com/akatsukioffici3/status/1530537511717482498?s=20&t=mMcJzi-SsGXbdeU6B4_IXw

このすば三期おめでとうございます！🎉

ところで、**ダンまち**と**このすば**と**リゼロ**、自分は混同しがちなんですが皆さんはいかがでしょう?
ソシャゲとかでも互いにコラボするので、余計混乱するんですよね。
