---
title: "GitHub等へのHTTPS接続用に個人用アクセストークンを求めるのは間違っているだろうか (Git Credential Manager)"
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

ですが、`git clone`等で以下の画面が出てきて初めて、以前使っていたパスワード認証が使えなくなった事に気づき、焦った方もいるのではないでしょうか?

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
  後述の GCM では`'write_repository'`と`'read_repository'`のみなので、
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
恐らくこれは、個人用アクセストークンではなく、OAuth アクセストークンですが、
GitHub のみ仕様が異なるようです。
[^github-authentication]: [Creating a personal access token - GitHub Docs](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token)
[^github-caching]: https://docs.github.com/en/get-started/getting-started-with-git/caching-your-github-credentials-in-git#git-credential-manager
:::

では、**どのようにアクセストークンを発行・管理するのが望ましいでしょうか?**

# アクセストークンの管理方法

主に以下があります。

| helper 名                   | 関連リポジトリ                                                                                           | 保存先                                  |
| --------------------------- | -------------------------------------------------------------------------------------------------------- | --------------------------------------- |
| `store`                     | [Git for Windows](https://github.com/git-for-windows/git) / [Git](https://github.com/git/git/)(built-in) | テキストファイル                        |
| `cache`                     | [Git for Windows](https://github.com/git-for-windows/git) / [Git](https://github.com/git/git/)(built-in) | ソケットファイル                        |
| `wincred`/<br>`osxkeychain` | [Git for Windows](https://github.com/git-for-windows/git) / [Git](https://github.com/git/git/)(built-in) | 認証情報マネージャー /<br> キーチェーン |
| `manager`                   | [Git Credential Manager for Windows](https://github.com/microsoft/Git-Credential-Manager-for-Windows)    | 認証情報マネージャー                    |
| `manager-core`              | [Git Credential Manager (GCM)](https://github.com/GitCredentialManager/git-credential-manager)           | 認証情報マネージャー 等                 |

[^git-tools-credential-storage]: [Git - 認証情報の保存](https://git-scm.com/book/ja/v2/Git-%E3%81%AE%E3%81%95%E3%81%BE%E3%81%96%E3%81%BE%E3%81%AA%E3%83%84%E3%83%BC%E3%83%AB-%E8%AA%8D%E8%A8%BC%E6%83%85%E5%A0%B1%E3%81%AE%E4%BF%9D%E5%AD%98)

::::details 🤔helper 名とは?
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

:::details 🤔 資格情報マネージャー(Credential Manager) とは?

Windows の場合、以下の様なものがあります。
ここに各種の認証情報を保管できるようですが、暗号化の有無などは把握できておりません。
![資格情報マネージャー](/images/manager-core-for-two-factor-authentication/credential-manager.png)
_資格情報マネージャー[^credential-manager]_

因みに macOS に於ける似たものとして "キーチェーンアクセス" というものがあるらしいです。

[^credential-manager]: [資格情報マネージャーにアクセスする (microsoft.com)](https://support.microsoft.com/ja-jp/windows/%E8%B3%87%E6%A0%BC%E6%83%85%E5%A0%B1%E3%83%9E%E3%83%8D%E3%83%BC%E3%82%B8%E3%83%A3%E3%83%BC%E3%81%AB%E3%82%A2%E3%82%AF%E3%82%BB%E3%82%B9%E3%81%99%E3%82%8B-1b5c916a-6a16-889f-8581-fc16e8165ac0)

:::

まず、Git をインストールすると、built-in で入っているのが、`store`,`cache`で、
Windows / Mac だと`wincred`/`osxkeychain`も利用可能です[^git-tools-credential-storage]。
更に、`manager-core`というものも最近推奨され始めています。
では上から順に見てみましょう。

- `store`
  テキストファイルに平文で保存します。
  デフォルトの保存先は、`~/.git-credentials`です。
  :::message alert
  平文保存なので基本的に推奨されません。
  :::

- `cache`

  > cache ヘルパーは独自形式でメモリーに情報を保持します
  > （他のプロセスはこの情報にアクセスできません）[^git-tools-credential-storage]。

  このように紹介されており、前述の`store`より安全らしいです。

  :::message
  `strace`でソケット?は覗けるらしいですが、自分は解読方法が分からないです……
  独自形式で保持して、Git のプロセス以外に共有されていないから安全なんですかね?
  :::

  :::message alert
  Windows は、Unix Socket を公式にサポートしていないので、現在は非対応です。
  参考: [🤔Q-06. "`credential-cache`on Windows"は可能か?](#🤔q-06.-"credential-cacheon-windows"は可能か%3F)
  :::

- `wincred`/`osxkeychain`
  前述の 2 つと同じで、Git インストール時にデフォルトで有効です。
  対話形式で入力したアクセストークンが、システムの認証情報マネージャーへ保存され、
  以降はそれを永続的に使います。

  ::::details コマンドで削除する場合
  Windows であれば「認証情報マネージャー」の情報を直接操作可能ですが、
  `wincred`の helper を使えば、コマンドで済ます事も可能です。

  ```bash:認証情報マネージャーから認証情報を削除するコマンド
  git credential-cache erase <<EOS
  protocol=https
  host=${対象のGitホスティングサービスのホスト名}
  EOS
  ```

  ::::

- `manager`

  古い記事だとこれの事しか書いてないですが、
  既に`manager-core`に統合され、リポジトリはアーカイブ済みです。
  Mac や Linux 用のものも同様です。
  :::message alert
  `manager`はアーカイブ済みであり、公式に`manager-core`で代替するようにアナウンスされているので、これを使いましょう。
  (Windows の場合は、Git を入れるだけで済みますが)
  名前が紛らわしく、未だに`manager`を推奨しているネット記事が残っていますが、
  **`manager-core`の正式名**は、[**Git Credential Manager (GCM)**](https://github.com/GitCredentialManager/git-credential-manager)なので注意しましょう。
  :::

- `magaer-core`
  後述する最新の GitHub ドキュメントで推奨されています。
  正式名は "Git Credential Manager (GCM)" です。
  :::message
  helper 名に "core" という部分が残っていますが、
  以前の正式名が "Git Credential Manager Core" だった頃の名残です[^github-20220407]。
  :::

  初めてアクセスする Git ホスティングサービス(GitHub 等)に対して、
  `git pull`/`git push`等を行うと、
  以下の様な UI が表示され、これに従ってブラウザ経由で認証できます。
  ![GCMによって表示される認証用のUI](/images/manager-core-for-two-factor-authentication/gcm-ui.png)
  _GCM によって表示される認証用の UI[^github-20220407]_

  尚、一度 GCM によるアクセスを認可したサービスでは、
  今後アクセストークンを要求される事はありません。
  [^github-20220407]: [Git Credential Manager: authentication for everyone | The GitHub Blog](https://github.blog/2022-04-07-git-credential-manager-authentication-for-everyone/)

  ::: message
  各種の Git ホスティングサービス内の設定で GCM を無効化すれば、
  再びアクセストークン発行が必要になりますので、UI も再び表示されます。
  :::

**アクセストークン管理**の選択肢を簡単に確認してきましたが。いかがでしたでしょうか?
**アクセストークン発行**も容易になる点で、OAuth アプリケーションの GCM が有力です。
ではここで、最近推奨され始めている**GCM**に焦点を移してみましょう。

# [Git Credential Manager (GCM)](https://github.com/GitCredentialManager/git-credential-manager)

https://github.blog/2022-04-07-git-credential-manager-authentication-for-everyone/
2022/04/07 に GitHub から公式にアナウンスされ、広く推奨されるようになったばかりです。
この発表に於けるポイントとしては、以下です。

- 以前までの [GCM for Windows](https://github.com/microsoft/git-credential-manager-for-windows) and [GCM for Mac and Linux](https://github.com/microsoft/git-credential-manager-for-mac-and-linux) を統合
  - [Avalonia UI](https://avaloniaui.net/)により、.NET でも macOS や Linux に対応
  - WSL の場合は、実行パスを Windows 側の物を設定すれば、設定の共有可能
- 全てのプラットフォームに提供する思想に基づくので、
  microsoft や github ではなく GCM のプロジェクトとして独立
- Web UI の指示に従って認可すると、
  裏でアクセストークンが発行され、更にローカルへ良い感じで保存してくれる
- GitLab も対応
  - オンプレミス環境の場合は別途設定が必要[^gcm-gitlab]だが、改善予定あり
- 認証情報に関して、様々な保存方法を設定可能
  - デフォルトの認証情報マネージャー、GPG 暗号化ファイル、キャッシュ等

:::message alert
Azure Devops 関連はちょっと知らないです…
Windows のセキュリティに詳しい方は、上記リンク先によく目を通される事を推奨します。
:::

[^gcm-gitlab]: https://github.com/GitCredentialManager/git-credential-manager/blob/main/docs/gitlab.md

:::details 🤔 仕組みは?
https://github.com/GitCredentialManager/git-credential-manager/blob/main/docs/architecture.md
:::

では、使い方について確認してみましょう。
主に以下です。

1. [GCM のインストール](#gcm-のインストール)
1. [Git ホスティングサービスの新規登録](#git-ホスティングサービスの新規登録)

## GCM のインストール

基本は公式リポジトリ[^gcm]の指示に従います。
以下には、あくまで 2022/07/01 現在の方法を一部紹介します。
[^gcm]: https://github.com/GitCredentialManager/git-credential-manager

### Windows

[Git for Windows](https://github.com/git-for-windows/git)をインストールする場合は、以下の画像の様にデフォルトで設定できます。
![Git for Windows インストール時に`credential helper`を選ぶ画面](/images/manager-core-for-two-factor-authentication/install-gcm-windows.png)
_Git for Windows インストール時に`credential helper`を選ぶ画面_

尚、別でインストールしたい場合は、
公式ページからダウンロードしてインストールしてください。

:::message
WSL では、以下 2 つの選択肢があります。
認証情報を Windows 側と共有できる点で、前者をお勧めします。

1. Windows 側の GCM を読み込む
1. WSL 内に新たにインストール

参考: [(中級者向け) WSL から Windows と資格情報の共有](<#(中級者向け)-wsl-から-windows-と資格情報の共有>)

:::

### その他

詳細に、アンインストール方法も書いてあるので、
公式リポジトリ[^gcm]を参照してください。

:::message alert
`LANG=en_US.UTF-8` でないと UI 表示でエラーが出るので注意しましょう。
参考: [🤔Q-07. Linux 環境で GCM の UI が表示されないが?](#🤔q-07-linux-環境で-gcm-の-ui-が表示されないが)
:::

## Git ホスティングサービスの新規登録

あとは`git pull`/`git push`を実行する度に以下の UI が表示されて使えるようになります。
これに従って、Web ブラウザでログインして認可すれば、
以降は GCM がアクセストークンの**更新**・**管理**を行ってくれます。
![GCMによって表示される認証用のUI](/images/manager-core-for-two-factor-authentication/gcm-ui.png)
_GCM によって表示される認証用の UI[^github-20220407]_

:::message
ここで発行するアクセストークンは OAuth のものです。
GitHub は仕様が異なるようですが、https://github.com/settings/applications の"Authorized OAuth Apps"から確認できるので OAuth と思われます。
参考: [🤔Q-02. GCM のアクセストークンは OAuth として発行されている?](#🤔q-02.-gcm-のアクセストークンは-oauth-として発行されている%3F)
:::

::::message
因みに、ここでアクセストークンを発行して保存すると、
その後は**自分で無効化しない限り**認証不要になります。

GitLab の場合、実際はアクセストークン自体変わっているのですが、裏で一緒に保存されたリフレッシュトークンがアクセストークンを更新してくれています。
:::details 🤔 アクセストークンの中身は? (オンプレミス環境 GitLab)
オンプレミス環境 GitLab の場合は以下のように確認可能です。

```bash:オンプレミス環境 GitLab のデータベースにアクセスしてOAuth2アクセストークンを確認する手順
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

一般的に、**利用者が自ら無効化しない限り**リフレッシュトークンは有効らしいです。
Google の場合はログインしてないと切れるらしいですね。
https://www.cdatablog.jp/entry/gcprefreshtokengrant

:::details 🤔GitLab ではどう実装されている?
GitLab の場合は個人アクセストークンではなく、
自動でアクセストークンを更新可能な OAuth2 のアクセストークンを使います。
内部では Doorkeeper を使ってますが、リフレッシュトークンの有効期限が無いようです[^doorkeeper-refresh-expiration]。
[^doorkeeper-refresh-expiration]: https://github.com/doorkeeper-gem/doorkeeper/wiki/Customizing-Token-Expiration#refresh-token

最近になって Closed の Issue[^doorkeeper-issue-360]で、有効期限が必要では?とのコメントがあり 👍 もありますが進展が無いですね。
正直、仕様が分からないのでどうしようも無いですね……🥶
[^doorkeeper-issue-360]: https://github.com/doorkeeper-gem/doorkeeper/issues/360
:::
::::

## (中級者向け) WSL から Windows と資格情報の共有

WSL 内に直接 GCM をインストールすると、Windows の資格情報にアクセスできません。
従って資格情報を共有したい場合は、
WSL 内の設定として Windows 側の GCM の実行パスを設定しましょう。[^gcm-wsl]
[^gcm-wsl]: https://github.com/GitCredentialManager/git-credential-manager/blob/main/docs/wsl.md

```bash
git config --global credential.helper "/mnt/c/Program\ Files/Git/mingw64/bin/git-credential-manager-core.exe"
# If you intend to use Azure DevOps you must also set the following Git configuration inside of your WSL installation.
git config --global credential.https://dev.azure.com.useHttpPath true
```

## (上級者向け) 様々な管理方法のオプション

> GCM makes use of the Windows Credential Manager on Windows and the login keychain on macOS.
> ![GCMによって表示される認証用のUI](/images/manager-core-for-two-factor-authentication/gcm-store.png)
> In addition to these existing mechanisms, we also support several alternatives across supported platforms, giving you the choice of how and where you wish to store your generated credentials (such as GPG-encrypted credential files). [^github-20220407]

公式アナウンスで紹介されている通り、
前項まで使う事はできますが、更に上級者向けの設定を紹介します。

[アクセストークンの管理方法](#アクセストークンの管理方法)で触れたように様々な helper が存在しますが、
`manager-core`はこれらを組み合わせる事も可能です。
一覧は以下ですが、詳細は公式のドキュメント[^gcm-credstores]先から確認してください。

[^gcm-credstores]: https://github.com/GitCredentialManager/git-credential-manager/blob/main/docs/credstores.md

| 設定名(`credentialStore`) | 名称                                                                                         | Windows | Mac | Linux |
| ------------------------- | -------------------------------------------------------------------------------------------- | ------- | --- | ----- |
| `wincredman`              | Windows Credential Manager                                                                   | ✅      |     |       |
| `dpapi`                   | DPAPI protected files                                                                        | ✅      |     |       |
| `keychain`                | macOS Keychain                                                                               |         | ✅  |       |
| `secretservice`           | [freedesktop.org Secret Service API](https://specifications.freedesktop.org/secret-service/) |         |     | ✅    |
| `gpg`                     | GPG/[`pass`](https://www.passwordstore.org/) compatible files                                |         | ✅  | ✅    |
| `cache`                   | Git's built-in [credential cache](https://git-scm.com/docs/git-credential-cache)             |         | ✅  | ✅    |
| `plaintext`               | Plaintext files                                                                              | ✅      | ✅  | ✅    |

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

設定としては、あくまで`git config --global credential.help "manager-core"`があり、
そのオプションとして`git config --global credential.cacheOptions "--timeout 300"`があります。
従って、単に`git config --global credential.credentialStore cache && git config --global credential.helper cache --timeout 300`と設定するのとは異なります。

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

## (上級者向け) キャッシュでアクセストークンを PC に残さない

> GCM can now also use Git’s git-credential-cache helper that is commonly built and available in many Git distributions. This is a great option for cloud shells or ephemeral environments when you don’t want to persist credentials permanently to disk but still want to avoid a prompt for every git fetch or git push. [^github-20220407]

GCM で発行・管理しても、アクセストークンを認証情報マネージャーで保管してるだけで、
実質 1 要素認証のままじゃない? と思うかもしれません。
そこでキャッシュをお勧めします。

これなら無効化しなくても、発行元ぐらいしかトークンを知らないので漏れる心配はありません。
また、キャッシュが切れる度に GCM を介してブラウザ経由の 2 要素認証が求められるようになります。
因みにこれは、内蔵の Git の`credential-cache`を使用して実現しています。

:::message alert
Windows 向けの Git は、 Unix Sockets に対応していないので、キャッシュは使えません。

正確には、Git には Windows でも Unix Sockets をサポートする設定が存在していますが、無効化されて配布されています。
自分でビルドすれば可能らしいですが、コントリビューターのレビューで、対応するまではエラー終了するようになりました[^gcm-pr-729]。
公式に Git でサポートされるのを待ちましょう。
待てない場合は、自分でビルドしようとの事でした。
[^gcm-pr-729]: https://github.com/GitCredentialManager/git-credential-manager/pull/729

参考: [🤔Q-06. "`credential-cache`on Windows"は可能か?](#🤔q-06.-"credential-cacheon-windows"は可能か%3F)
:::

## 願望

本当は、
**「GCM 導入によって 2 要素認証を経由してアクセスできるので、セキュリティ的に改善する」**
と言おうと思ってましたが、無理でした。
OAuth2.0 のリフレッシュトークンでアクセストークンを入れ替えていてもタイムアウトしないので、一度認証するとそれ以降は永久に認証不要なんですよね。

従って、**毎回ログインしてから短い有効期限のアクセストークンを発行**して使えば、正直 GCM よりもセキュリティ的には良さそうでした。
但し、**発行や管理の手間を省く点では GCM が便利**そうです。

どうしてもアクセストークンを使い回したくない場合は、
定期的に自分で GCM のアクセストークンを無効化して再発行させるしかなさそうです。
::::message
オンプレミス環境 GitLab なら、以下コマンドの自動実行で無効化して、2 要素認証を強要する運用もあり得ます。
インスタンスやグループ、ユーザーのレベルで作成しても同じ形式で無効化できるようでした。

```bash:GCMに関する全てのアカウントのアクセストークンを無効化するスクリプト
gitlab-rails runner "OauthAccessToken.all.filter_map { | oauthAccessToken | oauthAccessToken.revoke if oauthAccessToken.application_id == ${GCMのID} }"
```

::::

# 🤔Q&A

より詳細に知りたい方向けに、想定されるものを簡単に纏めました。
再掲事項も含みます。

## 🤔Q-01. 個人アクセストークン(PAT)と GCM 用の OAuth アクセストークンって本当に違うもの?

GitLab に関しては恐らくそうです。
但し、GitHub では、GCM が OAuth Application と言い張っている一方で、
次項で述べる様に、仕様がそうなっていないようなので、分からないです。

## 🤔Q-02. GCM のアクセストークンは OAuth として発行されている?

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

## 🤔Q-03. GCM よりも、2 要素認証でログインしてから必ず有効期限付きで PAT 発行した方が安全では?

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

## 🤔Q-04. PC に保存したアクセストークンは覗ける?

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

## 🤔Q-05. 初回アクセス時は必ず 2 要素認証でログインしたいんだが?

案としては、以下があります。

1. 自分で GCM の認証を無効化(Revoke)
2. キャッシュにより、PC に残さない

但し、1 は手間ですし、
2 に関しては [Git for Windows](https://github.com/git-for-windows/git) が対応していないので Windows のみ無理です。

## 🤔Q-06. "`credential-cache`on Windows"は可能か?

> Full disclosure: Technically, since Git v2.34.0,`git-credential-cache.exe`[_can_ be built and run on Windows](https://github.com/git/git/commit/c2e799012b3). Unix Sockets support [has been introduce into Windows](https://devblogs.microsoft.com/commandline/af_unix-comes-to-windows/). The problem is that you need Windows 10 build 17061 or later, and Git for Windows still supports even Vista (although [it has been announced that Git for Windows will drop supporting Vista really soon now](https://github.com/git-for-windows/build-extra/commit/c7e2c3cda90f4681400852d1207f74e96d8b1ff6)). [^gcm-issue-723-comment]

[^gcm-issue-723-comment]: https://github.com/GitCredentialManager/git-credential-manager/issues/723#issuecomment-1146817218

[Git for Windows](https://github.com/git-for-windows/git) 自体は、v2.34.0 から Unix Sockets に対応しましたが、
後方互換の為に、`NO_UNIX_SOCKETS = YesPlease`として無効化されています。
Vista のサポートは 2023 年より前に打ち切るとは伺えましたが、古いビルドバージョンのサポートは残っているので、公式に対応してくれる見通しは未だ無いです。
(ドキュメントから Windows は消して、エラーメッセージも出力する事になりました[^gcm-pr-729]😩)

従って、どうしても使いたい場合は、Git や GCM をビルドして使いましょう。

:::message
ビルド方法を自分も試してみたのですが、連携が難しくできていないです。
:::

## 🤔Q-07. Linux 環境で GCM の UI が表示されないが?

「Avalonia により.NET でも Linux 対応」とあるにも拘わらず、UI が出て来なくて焦る場合もあると思います。
これは、`LANG`を`en_US.UTF-8`以外に設定していると発生する、Avalonia 由来の仕様です。

https://github.com/AvaloniaUI/Avalonia/issues/4427

## 🤔Q-08. GCM はどのように OAuth2 アプリケーションとして UI を表示して動作している?

以下の様に、ホスト名で判別しています。
なので、これに該当しないオンプレミス環境 GitLab は`generic`として認識され UI が表示されません。
https://github.com/GitCredentialManager/git-credential-manager/blob/v2.0.779/src/shared/GitLab/GitLabConstants.cs#L48

因みに、`OAuthClientId`や`OAuthClientSecret`もソースコードに埋め込まれており、
他の設定値もここに記載されています。
https://github.com/GitCredentialManager/git-credential-manager/blob/v2.0.779/src/shared/GitLab/GitLabConstants.cs#L7-L13

## 🤔Q-09. オンプレミス環境の GitLab 対応はどうする?

@[card](https://github.com/GitCredentialManager/git-credential-manager/blob/main/docs/gitlab.md)

基本的には、上記リンクの指示に従います。
一部抜粋して和訳したものを以下に記します。

> ## 別インスタンスでの使い方
>
> 別インスタンスで使う為に、例えば`https://gitlab.example.com`では以下の設定が必要です。
>
> 1. [Create an OAuth application](https://docs.gitlab.com/ee/integration/oauth_provider.html). これは、ユーザーやグループ、インスタンスのレベルで行えます。
>    まず、名前やリダイレクト先を`http://127.0.0.1/`で指定してください。
>    そして、'Confidential' オプションは選択せず、'Expire access tokens' オプションは選択してください。
>    最後に、'write_repository' や 'read_repository' のスコープを設定してください。
> 1. application ID をコピーして、`git config --global credential.https://gitlab.example.com.GitLabDevClientId <APPLICATION_ID>`で設定してください。
> 1. application secret をコピーして、`git config --global credential.https://gitlab.example.com.GitLabDevClientSecret <APPLICATION_SECRET>`で設定してください。 1.`git config --global credential.https://gitlab.example.com.gitLabAuthModes browser`のように'browser'を含めて authentication modes を設定してください。
> 1. 念の為に、`git config --global credential.https://gitlab.example.com.provider gitlab`を設定してください。これは、ドメインを GitLab インスタンスとして認識させるのに必要かもしれません。 1.`git config --global --get-urlmatch credential https://gitlab.example.com`で、設定が期待通りかどうか確認してください。

:::message
今後のアップデートで自動対応する可能性もあります。
:::

## 🤔Q-10. HTTPS 非対応のオンプレミス環境 GitLab でも使える?

無理です。エラーメッセージがそう言ってました。
諦めて自己署名証明書を用意しましょう。

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

## 🤔Q-11. `pass` コマンドが連携できるなら、1Password を保存先として連携できる?

:::message
余力や要望があれな、以下などを参考に調査します。
https://dev.classmethod.jp/articles/create_git_credential_helper_with_1password/
:::

# 補足

https://twitter.com/akatsukioffici3/status/1530537511717482498

このすば三期おめでとうございます！🎉

ところで、**ダンまち**と**このすば**と**リゼロ**、自分は混同しがちなんですが皆さんはいかがでしょう?
ソシャゲとかでも互いにコラボするので、余計混乱するんですよね。
