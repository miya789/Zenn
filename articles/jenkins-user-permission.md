---
title: "Jenkinsユーザーの一括追加、前からするか？後からするか？"
emoji: "⛏️"
type: "tech"
topics: [jenkins, groovy]
published: true
---

# TL;DR

- **Jenkins ユーザー**の情報は、以下の 2 種類から構成される
  - **`SecurityRealm`**: ユーザーアカウントの情報
  - **`AuthorizationStrategy`**: 認証方法に関する情報
- **Jenkins ユーザー**をスクリプトで一括追加する手段は、以下の 2 通りから構成される
  - **Jenkins インスタンス起動前**なら、**Groovy Hook Script** として準備
    - きちんと置いてから再起動すれば読み込まれるが、
      一度使ったら削除しないと、起動する度に戻されて厄介なので、あまり使わなそう
  - **Jenkins インスタンス起動後**なら、**スクリプトコンソール** or **jenkins-cli.jar**
    - ユーザーアカウントやロールは何度も同じものを追加しないので、こちらがメイン
    - `jenkins-cli.jar` なら、管理者権限を持つユーザーアカウントの API キーが必要
- [`role-strategy`](https://github.com/jenkinsci/role-strategy-plugin)を使って、ロールとして付与する方が楽なので推奨
  - ロールを自作するようなスクリプトを書いて、[`matrix-auth`](https://plugins.jenkins.io/matrix-auth/) だけで乗り切りるのも可能
  <!-- 具体的には? -->

# 背景: Jenkins のユーザー情報とは?

**Jenkins ユーザー**の情報は、以下の 2 種類から構成されます。

- **`SecurityRealm`**: ユーザーアカウントの情報
  - `jenkins.model.Jenkins.instance.securityRealm` で所得可能
  - 例
    - `hudson.security.HudsonPrivateSecurityRealm`
    - `hudson.security.LDAPSecurityRealm`
- **`AuthorizationStrategy`**: 認証方法に関する情報
  - `jenkins.model.Jenkins.instance.authorizationStrategy` で所得可能
  - 例
    - `hudson.security.AuthorizationStrategy`
    - `hudson.security.ProjectMatrixAuthorizationStrategy`
    - `com.michelin.cio.hudson.plugins.rolestrategy.RoleBasedAuthorizationStrategy`

殊 **`AuthorizationStrategy`** は、プラグインにる詳細な権限設定が可能です。
これらは、`config.xml` として保持され、起動時はまずこれを読み込みます。

:::message
起動中に `config.xml` を直接変更しても反映されないですが、直接変更した後に再起動すると読み込まれます。
:::

# 目的: ソースコードで一括追加したくないですか?

ユーザーを作っただけだと、権限を全員共通でしか扱えず不便なので、個別に設定します。
そこで一般的[^jenkins-managing-security]なのは、以下の様な Web UI から操作する[`matrix-auth`](https://plugins.jenkins.io/matrix-auth/) でしょう。
[^jenkins-managing-security]: [Managing Security](https://www.jenkins.io/doc/book/security/managing-security/)

![`matrix-auth`の操作画面](/images/jenkins-user-permission/configure-global-security-matrix-authorization.png)
_`matrix-auth`の操作画面_

しかし、これの操作、面倒臭くないですか?
予め「誰のアカウントを登録し権限も付与するのか」決まっているのに、ブラウザの Web UI で毎回、ユーザー名を入力して枠を作りカーソルをテーブルのチェックに入れてクリック、ユーザー名を入力して枠を作りカーソルをテーブルのチェックに入れてクリック、ユーザー名を入力して枠を作りカーソルをテーブルのチェックに入れてクリック……

可能なら、全て配列とか適当な文字列のものを流し込んで、一括でやりたくないですか?
Jenkins インスタンスに対して、**ソースコードで一括追加**したくないですか?

# 方法

## 1. Groovy スクリプトの実行方法を選択

設定方法は、**設定画面**(従来)と**Groovy スクリプト**の 2 通りですが、今回は後者の説明をします。
中でも、**Groovy スクリプト** の使い方に関しては以下の 2 通りですが、起動後が個人的にはお勧めです。

| 実行タイミング | 手段                                     | ユーザーの一括追加に使うべき?   |
| -------------- | ---------------------------------------- | ------------------------------- |
| 起動前         | Groovy Hook Script                       | ❌ 毎回読み込むものではないので |
| 起動後(起動中) | スクリプトコンソール or `jenkis-cli.jar` | ✅ 一時的なものなので           |

Groovy Hook Script に関してここでは詳細を述べませんが、
公式のコンテナイメージ[^github-jenkinsci-docker]の場合は、`/usr/share/jenkins/ref/init.groovy.d/` の様なディレクトリに置くと読み込まれました。

[^github-jenkinsci-docker]: [jenkinsci/docker: Docker official jenkins repo](https://github.com/jenkinsci/docker)

## 2. `role-strategy` プラグインのインストール

- 選択肢: 設定画面 or Groovy スクリプト

ここに関しては、プラグインのインストールで調べると出て来る記事[^995cf34afd126a627c22]の通りです。
(使わない変数があるのが気になったので省いています。)
[^995cf34afd126a627c22]:[Jenkins の構築それ全部自動でできるよ - Qiita](https://qiita.com/fuku2014/items/995cf34afd126a627c22)

```groovy:role-strategy のインストール
def plugins   = ["role-strategy"]
// def plugins   = ["matrix-auth"]

def instance  = jenkins.model.Jenkins.getInstance()
pm = instance.getPluginManager()
uc = instance.getUpdateCenter()
uc.updateAllSites()

def enablePlugin(pluginName) {
  if (! pm.getPlugin(pluginName)) {
    deployment = uc.getPlugin(pluginName).deploy(true)
    deployment.get()
  }

  def plugin = pm.getPlugin(pluginName)
  if (! plugin.isEnabled()) {
    plugin.enable()
  }

  plugin.getDependencies().each {
    enablePlugin(it.shortName)
  }
}

plugins.each {
  enablePlugin(it)
}
```

:::message
Docker の場合は、既にコンテナイメージに含まれる [Plugin Installation Manager Tool for Jenkins](https://github.com/jenkinsci/plugin-installation-manager-tool) を使ってインストールさせる事も可能です[^jenkinsci-docker-preinstalling-plugins]。

```Dockerfile:role-strategy を予めインストールしたコンテナイメージ
FROM jenkins/jenkins:lts-jdk11
RUN jenkins-plugin-cli --plugins role-strategy
```

:::

## 3. Jenkins ユーザーの作成

- 選択肢: 設定画面 or Groovy スクリプト

後述の権限付与だけも可能ですが、ユーザーが不在だとこのような画面になってしまいます。
なので一応やっておきましょう。
![権限を付与したユーザーアカウントが無いと出て来るエラー](/images/jenkins-user-permission/configure-global-security-matrix-authorization-with-error.png)
_権限を付与したユーザーアカウントが無いと出て来るエラー_

:::message
LDAP 等を使って、外部と連携している場合は不要です。
:::

以下の様に、`jenkins.model.Jenkins.instance` から `realm` を取り出して、これに対してアカウント追加メソッドを呼ぶだけです[^groovy-create-user-md]。

[^groovy-create-user-md]: [Jenkins Groovy enable security and create a user in groovy script](https://gist.github.com/hayderimran7/50cb1244cc1e856873a4)

```groovy:Jenkinsユーザーの作成
def user_name = "sample_user"
def pass = "password"

def instance = jenkins.model.Jenkins.instance
def realm = instance.securityRealm
realm.createAccount(user_name, pass)

instance.save() // ログにも残せるので推奨
```

## 4. 付与する権限の確認

- 選択肢: Permission クラス定数 指定 or Group 指定

主に `PermissionGroup` -> `Permission` の流れで、付与したい権限を特定していきます。

### `PermissionGroup` の一覧取得

以下の様に `PermissionGroup.getAll()` によって、動作中の Jenkins インスタンスで取り得る `PermissionGroup` の一覧が取得できます。

```groovy:PermissionGroup.getAll() による PermissionGroup の一覧取得
def groups = hudson.security.PermissionGroup.getAll()
// [
//   PermissionGroup[hudson.model.Hudson],
//   PermissionGroup[hudson.model.Computer],
//   PermissionGroup[hudson.model.Item],
//   PermissionGroup[hudson.model.View],
//   PermissionGroup[hudson.security.Permission]
// ]
```

::::details PermissionGroup の Permission クラス定数 指定方法

`PermissionGroup` の Permission クラス定数 は、以下の様に 2 通りの方法で絞れます。

```groovy:PermissionGroup.get(Class owner) で指定する方法
hudson.security.PermissionGroup.get(hudson.model.Hudson.class)
// PermissionGroup[hudson.model.Hudson]
hudson.security.PermissionGroup.get(hudson.model.Computer.class)
// PermissionGroup[hudson.model.Computer]
hudson.security.PermissionGroup.get(hudson.model.Item.class)
// PermissionGroup[hudson.model.Item]
hudson.security.PermissionGroup.get(hudson.model.View.class)
// PermissionGroup[hudson.model.View]
hudson.security.PermissionGroup.get(hudson.security.Permission.class)
// PermissionGroup[hudson.security.Permission]
```

```groovy:Permission クラス定数 で指定する方法
println hudson.model.Hudson.PERMISSIONS
// PermissionGroup[hudson.model.Hudson]
println hudson.model.Computer.PERMISSIONS
// PermissionGroup[hudson.model.Computer]
println hudson.model.Item.PERMISSIONS
// PermissionGroup[hudson.model.Item]
println hudson.model.View.PERMISSIONS
// PermissionGroup[hudson.model.View]
println hudson.security.Permission.GROUP
// PermissionGroup[hudson.security.Permission]
```

:::message alert
`PermissionGroup[hudson.security.Permission]` は使わないので省きましょう。
中身は以下です。

```
Permission[class hudson.security.Permission,FullControl]
Permission[class hudson.security.Permission,GenericConfigure]
Permission[class hudson.security.Permission,GenericCreate]
Permission[class hudson.security.Permission,GenericDelete]
Permission[class hudson.security.Permission,GenericRead]
Permission[class hudson.security.Permission,GenericUpdate]
Permission[class hudson.security.Permission,GenericWrite]
```

:::

:::message
`hudson.model.Hudson` に関して、
これは `jenkins.model.Jenkins` を継承しているので、実際の変数定義は `jenkins.model.Jenkins` にあります。
:::
::::

### `PermissionGroup` から `Permission` の一覧取得

最後に、各 `PermissionGroup` グループから [`getPermissions()`](https://github.com/jenkinsci/jenkins/blob/master/core/src/main/java/hudson/security/PermissionGroup.java#L107) によって、これに所属する `Permission` の一覧が取得できます。

```diff groovy:PermissionGroup から Permission を取り出す方法
  def groups = hudson.security.PermissionGroup.getAll()
+ def permissions = groups.stream().map{g -> g.getPermissions()}.collect().flatten()
  // [
  //   Permission[class hudson.model.Hudson,Administer]
  //   Permission[class hudson.model.Hudson,ConfigureUpdateCenter]
  //   Permission[class hudson.model.Hudson,Manage]
  //   Permission[class hudson.model.Hudson,Read]
  //   Permission[class hudson.model.Hudson,RunScripts]
  //   Permission[class hudson.model.Hudson,SystemRead]
  //   Permission[class hudson.model.Hudson,UploadPlugins]
  //   ...
  // ]
```

:::message

一方でこんな方法もありますが、各 Permission クラス定数が登録された順序だからなのか、汚いですね……

```groovy:Permission に登録された Permission クラス定数を一覧取得する方法
def permissions = hudson.security.Permission.ALL
// [
//   Permission[class hudson.model.Hudson,Administer]
//   Permission[class hudson.security.Permission,FullControl]
//   Permission[class hudson.security.Permission,GenericRead]
//   Permission[class hudson.security.Permission,GenericWrite]
//   ...
// ]
```

:::

### `Permission` から `id` の取得

`Permission` は一覧取得できましたが、実際にどのような `Permission` クラス定数を使っているのか特定できていないので、未だソースコードで使えません。
しかし、実は各`Permission` の `getId()` から取得可能な `id` が分かれば、クラス定数を特定する必要はありません。

```diff groovy
  def groups = hudson.security.PermissionGroup.getAll()
  def permissions = groups.stream().map{g -> g.getPermissions()}.collect().flatten()
+ def permission_ids = permissions.stream().map{p -> p.getId()}.collect()
  // [
  //   hudson.model.Hudson.Administer
  //   hudson.model.Hudson.ConfigureUpdateCenter
  //   hudson.model.Hudson.Manage
  //   hudson.model.Hudson.Read
  //   hudson.model.Hudson.RunScripts
  //   hudson.model.Hudson.SystemRead
  //   ...
  // ]
```

こうして、Java 内部で呼び出すべき `Permission` クラス定数が分からなくても、
`id` と `fromId()` を使って、以下の様に呼び出し可能になりました。

```groovy
def permission = hudson.security.Permission.fromId("hudson.model.Hudson.Administer")
// Permission[class hudson.model.Hudson,Administer]
```

:::message
実際、設定画面では、この `id` が HTML に埋め込まれており、
内部では `fromId()` で変換して使われています。
![設定画面において id がHTMLに埋め込まれて使われている様子](/images/jenkins-user-permission/configure-global-security-role-strategy-permission-class-constants.png)
_設定画面において id が HTML に埋め込まれて使われている様子_
:::

所で、ここまで `id` を熱く推奨してきましたが、
この記事の Q&A で `PermissionGroup` の Permission クラス定数の一覧は公表しているので、
ソースコードは Permission クラス定数ありで説明します。(長くて読み難いですし……)

:::message
稀に、プラグインによって突然元からいた様な顔して生えて来る権限[^where-is-hudson-model-run-replay-permission]もあり、
苦しむ事になるので、`Permission` クラス定数の特定はあまりお勧めしないです。
:::

[^where-is-hudson-model-run-replay-permission]: [jenkins - Where is hudson.model.Run.Replay permission object defined? - Stack Overflow](https://stackoverflow.com/questions/59665635/where-is-hudson-model-run-replay-permission-object-defined)

## 5. 付与する権限によるロールの作成 & Jenkins ユーザーへロールのアサイン

- 選択肢: 設定画面 or Groovy スクリプト

では、初回起動時の想定で一通り書きます。

### 各種変数の準備

まず基本的なものを用意しましょう。

```groovy:付与する権限によるロールの作成 & Jenkins ユーザーへロールのアサイン (起動前) 1/5
def user_name = "sample_user"
def role_name = "sample_role"

def instance = jenkins.model.Jenkins.instance
def strategy = new com.michelin.cio.hudson.plugins.rolestrategy.RoleBasedAuthorizationStrategy()
def globalRoleMap = strategy.getRoleMap(com.synopsys.arc.jenkins.plugins.rolestrategy.RoleType.Global)
```

### `Permission` から成る `HashSet` の作成

では次に、ロールが無い想定で作成します。
まず、使いたい `Permission` で `HashSet` を作りましょう。

```diff groovy:付与する権限によるロールの作成 & Jenkins ユーザーへロールのアサイン (起動前) 2/5
  def user_name = "sample_user"
  def role_name = "sample_role"

  def instance = jenkins.model.Jenkins.instance
  def strategy = new com.michelin.cio.hudson.plugins.rolestrategy.RoleBasedAuthorizationStrategy()
  def globalRoleMap = strategy.getRoleMap(com.synopsys.arc.jenkins.plugins.rolestrategy.RoleType.Global)
+
+ def _permissions = new HashSet<>()
+ _permissions.add(jenkins.model.Jenkins.READ) // これは必ず必要
+ _permissions.add(hudson.model.Item.BUILD)
+ _permissions.add(hudson.model.Item.READ)
+ _permissions.add(hudson.model.Item.CONFIGURE)
+ _permissions.add(hudson.model.Item.DELETE)
+ permissions = new HashSet<>(_permissions.stream().filter{ it.enabled }.collect()) // enableでないPermissionを除去
```

:::message alert
`jenkins.model.Jenkins.READ` を与え忘れると、
どんな権限があっても何も見えないので注意しましょう。
:::

::::message
`enabled` でない `Permission` も混ざる可能性があるので、一応フィルターしています。
::::

### `Role` の作成

次に、`role_name` とい名前で、ジョブに関する権限を持つロールを作って登録し、最後にアサインしましょう。
尚、今回はパターンのある場合[^rbac_example-groovy]を考えるのが面倒だったので、`RoleType` は `Global` です。

[^rbac_example-groovy]: [jenkins-scripts/RBAC_Example.groovy at master · cloudbees/jenkins-scripts](https://github.com/cloudbees/jenkins-scripts/blob/master/RBAC_Example.groovy)

```diff groovy:付与する権限によるロールの作成 & Jenkins ユーザーへロールのアサイン (起動前) 3/5
  ...

  def _permissions = new HashSet<>()
  _permissions.add(jenkins.model.Jenkins.READ) // これは必ず必要
  _permissions.add(hudson.model.Item.BUILD)
  _permissions.add(hudson.model.Item.READ)
  _permissions.add(hudson.model.Item.CONFIGURE)
  _permissions.add(hudson.model.Item.DELETE)
  permissions = new HashSet<>(_permissions.stream().filter{ it.enabled }.collect()) // enableでないPermissionを除去
+
+ def role = new com.michelin.cio.hudson.plugins.rolestrategy.Role(role_name, permissions)
+ globalRoleMap.addRole(role)
+ globalRoleMap.assignRole(role, user_name)
```

### 管理者権限を持つユーザーアカウントの作成

このまま、`instance.setAuthorizationStrategy(strategy)` したい所ですが、
管理者権限を誰かしらに与えておきましょう。

:::message alert
管理者権限を持つユーザーアカウントが皆無だと、誰もログインできないので、
起動後に流す場合も、特に注意してください。
:::

```diff groovy:付与する権限によるロールの作成 & Jenkins ユーザーへロールのアサイン (起動前) 4/5
  ...

  def _permissions = new HashSet<>()
  _permissions.add(jenkins.model.Jenkins.READ) // これは必ず必要
  _permissions.add(hudson.model.Item.BUILD)
  _permissions.add(hudson.model.Item.READ)
  _permissions.add(hudson.model.Item.CONFIGURE)
  _permissions.add(hudson.model.Item.DELETE)
  permissions = new HashSet<>(_permissions.stream().filter{ it.enabled }.collect()) // enableでないPermissionを除去

  def role = new com.michelin.cio.hudson.plugins.rolestrategy.Role(role_name, permissions)
  globalRoleMap.addRole(role)
  globalRoleMap.assignRole(role, user_name)
+
+ // 初期構築時は、誰もログインできなくならないように実施
+ if (! globalRoleMap.getRoles().stream().anyMatch{p -> p.hasPermission(jenkins.model.Jenkins.ADMINISTER)}) {
+   def adminRole = new com.michelin.cio.hudson.plugins.rolestrategy.Role("admin", new HashSet(Arrays.asList(jenkins.model.Jenkins.ADMINISTER)))
+
+   globalRoleMap.addRole(adminRole)
+   globalRoleMap.assignRole(adminRole, "admin")
+ }
```

<!-- 何かを参考に -->

### `instance` へ `strategy` の反映

これで問題無くなったので、後は `instance.setAuthorizationStrategy(strategy)` するだけです。

```diff groovy:付与する権限によるロールの作成 & Jenkins ユーザーへロールのアサイン (起動前) 5/5
  def user_name = "sample_user"
  def role_name = "sample_role"

  def instance = jenkins.model.Jenkins.instance
  def strategy = new com.michelin.cio.hudson.plugins.rolestrategy.RoleBasedAuthorizationStrategy()
  def globalRoleMap = strategy.getRoleMap(com.synopsys.arc.jenkins.plugins.rolestrategy.RoleType.Global)

  def _permissions = new HashSet<>()
  _permissions.add(jenkins.model.Jenkins.READ) // これは必ず必要
  _permissions.add(hudson.model.Item.BUILD)
  _permissions.add(hudson.model.Item.READ)
  _permissions.add(hudson.model.Item.CONFIGURE)
  _permissions.add(hudson.model.Item.DELETE)
  permissions = new HashSet<>(_permissions.stream().filter{ it.enabled }.collect()) // enableでないPermissionを除去

  def role = new com.michelin.cio.hudson.plugins.rolestrategy.Role(role_name, permissions)
  globalRoleMap.addRole(role)
  globalRoleMap.assignRole(role, user_name)

  // 初期構築時は、誰もログインできなくならないように実施
  if (! globalRoleMap.getRoles().stream().anyMatch{p -> p.hasPermission(jenkins.model.Jenkins.ADMINISTER)}) {
    def adminRole = new com.michelin.cio.hudson.plugins.rolestrategy.Role("admin", new HashSet(Arrays.asList(jenkins.model.Jenkins.ADMINISTER)))

    globalRoleMap.addRole(adminRole)
    globalRoleMap.assignRole(adminRole, "admin")
  }
+
+ instance.setAuthorizationStrategy(strategy)
+ instance.save() // ログにも残せるので推奨
```

これで完成です。

これは、[init.groovy.d/05_role_assign_role_addPermission.groovy](https://github.com/miya789/assigning-jenkins-users-via-groovy-scripts/blob/main/init.groovy.d/05_role_assign_role_addPermission.groovy)として、後述のリポジトリにも置いてあります。

<!-- 直して -->

### ⚠️ 起動後に行う場合

特に起動後で何かしらセキュリティ設定した後で、上記の設定をそのまま使うと、
既存の設定が消える虞がありますので注意してください。

```diff groovy:付与する権限によるロールの作成 & Jenkins ユーザーへロールのアサイン (起動後)
  def user_name = "sample_user"
  def role_name = "sample_role"

  def instance = jenkins.model.Jenkins.instance
- def strategy = new com.michelin.cio.hudson.plugins.rolestrategy.RoleBasedAuthorizationStrategy()
+ def strategy = instance.authorizationStrategy
  def globalRoleMap = strategy.getRoleMap(com.synopsys.arc.jenkins.plugins.rolestrategy.RoleType.Global)

  def _permissions = new HashSet<>()
  _permissions.add(jenkins.model.Jenkins.READ) // これは必ず必要
  _permissions.add(hudson.model.Item.BUILD)
  _permissions.add(hudson.model.Item.READ)
  _permissions.add(hudson.model.Item.CONFIGURE)
  _permissions.add(hudson.model.Item.DELETE)
  permissions = new HashSet<>(_permissions.stream().filter{ it.enabled }.collect()) // enableでないPermissionを除去

  def role = new com.michelin.cio.hudson.plugins.rolestrategy.Role(role_name, permissions)
  globalRoleMap.addRole(role)
  globalRoleMap.assignRole(role, user_name)

  // 初期構築時は、誰もログインできなくならないように実施
  if (! globalRoleMap.getRoles().stream().anyMatch{p -> p.hasPermission(jenkins.model.Jenkins.ADMINISTER)}) {
    def adminRole = new com.michelin.cio.hudson.plugins.rolestrategy.Role("admin", new HashSet(Arrays.asList(jenkins.model.Jenkins.ADMINISTER)))

    globalRoleMap.addRole(adminRole)
    globalRoleMap.assignRole(adminRole, "admin")
  }

- instance.setAuthorizationStrategy(strategy)
  instance.save() // ログにも残せるので推奨
```

# 結果: [`role-strategy`](https://github.com/jenkinsci/role-strategy-plugin) 使用時のスクリプト

https://github.com/miya789/assigning-jenkins-users-via-groovy-scripts

検証用のリポジトリを作成しました。
簡単な例のみ挙げていますが、他は記事を参考に組み立ててください。

:::message alert
尚、簡単に動作検証するために、本記事で非推奨とした **Groovy Hook Script** を使っており、
ソースコードも初期構築用なので、
スクリプトコンソール or `jenkis-cli.jar` でそのまま実行しないでください

<!-- 具体的には? -->

:::

# 考察

[Jenkins Configuration as Code](https://www.jenkins.io/projects/jcasc/)とかと組み合わせたらもっと楽にできるかもしれないですかね。
今回は、初期起動時の話というより、起動後の話なので少し逸れそうですが……

叉、今回はあまり触れませんでしたが、外部から実行するには `jenkins-cli.jar` を使う事になります[^setup-jenkins-users-with-cli]。
[^setup-jenkins-users-with-cli]: [deployment - Automatically setup jenkins users with CLI - Stack Overflow](https://stackoverflow.com/questions/10066536/automatically-setup-jenkins-users-with-cli)

この場合も、管理者権限を持つユーザーアカウントの API キーが必要になりますが、環境変数を別で持ってしまえば解決できるかもしれないですね。

# Q&A

## 🤔Q-01. Plugin の一括ダウンロードの方法は?

Jenkins の コンテナイメージでは、以下の様に `/opt/jenkins-plugin-manager.jar` として、
[jenkinsci/plugin-installation-manager-tool: Plugin Manager CLI tool for Jenkins](https://github.com/jenkinsci/plugin-installation-manager-tool) が用意されており、
これの使用が推奨されています[^jenkinsci-docker-preinstalling-plugins]。

[^jenkinsci-docker-preinstalling-plugins]: https://github.com/jenkinsci/docker#preinstalling-plugins

https://github.com/jenkinsci/docker/blob/master/17/debian/bullseye/hotspot/Dockerfile#L90-L92

## 🤔Q-02. Jenkin インスタンスの取得方法

Jenkin インスタンスを取得する際に、稀に古い記事で使われている `jenkins.model.Jenkins.getInstance()` は Deprecated です。
スクリプトコンソールの例では、`jenkins.model.Jenkins.instance` が使われていますが、どうしても関数で呼び出したいなら、`jenkins.model.Jenkins.get()` を使いましょう。

https://github.com/jenkinsci/jenkins/blob/master/core/src/main/java/jenkins/model/Jenkins.java#L803-L816

## 🤔Q-03. `instance.save()` とは?

エラー時にログを出力してくれる筈なので、推奨

## 🤔Q-04. 権限を与えられたアカウント一覧が欲しい

`getAllPermissionEntries()` を使いましょう。
元の姿は`Map<Permission, Set<PermissionEntry>> getGrantedPermissionEntries()` という
`Permission` をキーとする HashMap なのですが、
これを纏めてアカウント一覧( `PermissionEntry` )だけにしてくれます。

https://github.com/jenkinsci/matrix-auth-plugin/blob/master/src/main/java/org/jenkinsci/plugins/matrixauth/AuthorizationContainer.java#L240-L248

```groovy:getGrantedPermissionEntries()により返ってくるHashMap
def strategy = jenkins.model.Jenkins.instance.getAuthorizationStrategy()
strategy.getGrantedPermissionEntries()
// {
//   Permission[class hudson.model.Hudson,Read]=[PermissionEntry{type=USER, sid='test_user'}],
//   Permission[interface hudson.model.Item,Delete]=[PermissionEntry{type=USER, sid='test_user'}],
//   Permission[interface hudson.model.Item,Read]=[PermissionEntry{type=USER, sid='test_user'}],
//   Permission[class hudson.model.Hudson,Administer]=[PermissionEntry{type=GROUP, sid='authenticated'}],
//   Permission[class com.cloudbees.plugins.credentials.CredentialsProvider,Delete]=[PermissionEntry{type=USER, sid='test_user'}],
//   Permission[class com.cloudbees.plugins.credentials.CredentialsProvider,Create]=[PermissionEntry{type=USER, sid='test_user'}],
//   Permission[interface hudson.model.Item,Build]=[PermissionEntry{type=USER, sid='test_user'}],
//   Permission[interface hudson.model.Item,Configure]=[PermissionEntry{type=USER, sid='test_user'}]
// }
```

## 🤔Q-05. 代表的なプラグインを入れると権限一覧はどうなる?

これ[^jenkins-matrix-based-security-with]を参考にすると以下の通りです。
🚧 残りの権限や項目に関しては追って更新します。

[^jenkins-matrix-based-security-with]: [midweekmidmorning: Jenkins Matrix Based Security with Groovy Scripts](http://midweekmidmorning.blogspot.com/2016/06/jenkins-matrix-based-security-with.html)

| id                                                                    | Permission クラス定数               |
| --------------------------------------------------------------------- | ----------------------------------- |
| "hudson.model.Hudson.Administer"                                      | `jenkins.model.Jenkins.ADMINISTER`  |
| "hudson.security.Permission.FullControl"                              |                                     |
| "hudson.security.Permission.GenericRead"                              |                                     |
| "hudson.security.Permission.GenericWrite"                             |                                     |
| "hudson.security.Permission.GenericCreate"                            |                                     |
| "hudson.security.Permission.GenericUpdate"                            |                                     |
| "hudson.security.Permission.GenericDelete"                            |                                     |
| "hudson.security.Permission.GenericConfigure"                         |                                     |
| "hudson.model.Hudson.Manage"                                          |                                     |
| "hudson.model.Hudson.SystemRead"                                      |                                     |
| "hudson.model.Hudson.Read"                                            | `jenkins.model.Jenkins.READ`        |
| "hudson.model.Hudson.RunScripts"                                      | `jenkins.model.Jenkins.RUN_SCRIPTS` |
| "hudson.model.Hudson.UploadPlugins"                                   |                                     |
| "hudson.model.Hudson.ConfigureUpdateCenter"                           |                                     |
| "hudson.model.View.Create"                                            | `hudson.model.View.CREATE`          |
| "hudson.model.View.Delete"                                            | `hudson.model.View.DELETE`          |
| "hudson.model.View.Configure"                                         | `hudson.model.View.CONFIGURE`       |
| "hudson.model.View.Read"                                              | `hudson.model.View.READ`            |
| "hudson.model.Computer.Configure"                                     | `hudson.model.Computer.CONFIGURE`   |
| "hudson.model.Computer.ExtendedRead"                                  |                                     |
| "hudson.model.Computer.Delete"                                        | `hudson.model.Computer.DELETE`      |
| "hudson.model.Computer.Create"                                        | `hudson.model.Computer.CREATE`      |
| "hudson.model.Computer.Disconnect"                                    | `hudson.model.Computer.DISCONNECT`  |
| "hudson.model.Computer.Connect"                                       | `hudson.model.Computer.CONNECT`     |
| "hudson.model.Computer.Build"                                         | `hudson.model.Computer.BUILD`       |
| "hudson.model.Computer.Provision"                                     |                                     |
| "hudson.model.Item.Create"                                            | `hudson.model.Item.CREATE`          |
| "hudson.model.Item.Delete"                                            | `hudson.model.Item.DELETE`          |
| "hudson.model.Item.Configure"                                         | `hudson.model.Item.CONFIGURE`       |
| "hudson.model.Item.Read"                                              | `hudson.model.Item.READ`            |
| "hudson.model.Item.Discover"                                          | `hudson.model.Item.DISCOVER`        |
| "hudson.model.Item.ExtendedRead"                                      | `hudson.model.Item.EXTENDED_READ`   |
| "hudson.model.Item.Build"                                             | `hudson.model.Item.BUILD`           |
| "hudson.model.Item.Workspace"                                         | `hudson.model.Item.WORKSPACE`       |
| "hudson.scm.SCM.Tag"                                                  |                                     |
| "hudson.model.Item.WipeOut"                                           | `hudson.model.Item.WIPEOUT`         |
| "hudson.model.Item.Cancel"                                            | `hudson.model.Item.CANCEL`          |
| "hudson.model.Run.Delete"                                             |                                     |
| "hudson.model.Run.Update"                                             |                                     |
| "hudson.model.Run.Artifacts"                                          |                                     |
| "com.cloudbees.plugins.credentials.CredentialsProvider.UseOwn"        |                                     |
| "com.cloudbees.plugins.credentials.CredentialsProvider.UseItem"       |                                     |
| "com.cloudbees.plugins.credentials.CredentialsProvider.Create"        |                                     |
| "com.cloudbees.plugins.credentials.CredentialsProvider.Update"        |                                     |
| "com.cloudbees.plugins.credentials.CredentialsProvider.View"          |                                     |
| "com.cloudbees.plugins.credentials.CredentialsProvider.Delete"        |                                     |
| "com.cloudbees.plugins.credentials.CredentialsProvider.ManageDomains" |                                     |
| "hudson.plugins.jobConfigHistory.JobConfigHistory.DeleteEntry"        |                                     |
| "hudson.model.Run.Replay"                                             |                                     |

以下の部分で、各 `Permission` は、`hudson.security.Permission.ALL` に登録されているので、
これを呼べばで取得できます。

https://github.com/jenkinsci/jenkins/blob/master/core/src/main/java/hudson/security/Permission.java#L140-L156

## 🤔Q-06. `role-strategy` では、どのように権限の一覧を取得している?

ここでは、タイプに応じて種類からグループを絞っています。
https://github.com/jenkinsci/role-strategy-plugin/blob/master/src/main/java/com/michelin/cio/hudson/plugins/rolestrategy/RoleBasedAuthorizationStrategy.java#L969-L1005

## 🤔Q-07. スクリプトで初回追加する場合の注意は?

グローバルセキュリティと違って、自動で `admin` を作ってくれないので、ログインできなくなる場合に注意しましょう。

## 🤔Q-08. 内部の実装は?

何が裏で起きているのでしょうか?

1. Jenkins のセキュリティのページにアクセス
1. チェックを入れて Apply クリック

ここで、2 の段階で、裏で POST が発行されています。
こうして内部では以下のデータ変換が行われます。

1. `GlobalMatrixAuthorizationStrategy` のインスタンスを新規作成
1. JSON として Java 内で変換
1. 各ユーザーの項目(`PermissionEntry`)に対して、JSON 化した構造を精査
1. HTML に埋め込まれた `<input>` の `name` 属性として値を保持し、各項目で `boolean` の値を持つ
1. 各値に対して、`Permission`に該当するか調査し、該当すれば `Permission.fromId()`で変換
1. `GlobalMatrixAuthorizationStrategy` のインスタンスに、エントリーとこれのペアのハッシュを追加

つまり、
HTML に埋め込まれた `[hudson.model.Hudson.Read]` の形式から、
`fromId(String)` で変換して、内部では Permission オブジェクトを持っています。

## 🤔Q-09. Groovy Hook Script の実行順序は?

辞書順らしいです[^groovy-hook-scripts]。
[^groovy-hook-scripts]: [Groovy Hook Scripts](https://www.jenkins.io/doc/book/managing/groovy-hook-scripts/)

## 🤔Q-10. `jenkins.model.Jenkins.RUN_SCRIPTS` があれば管理者権限無しでスクリプト実行できる?

残念ながらできません。
古いドキュメント[^jenkins-matrix-based-security]を見る限りだと以前はできたようですが、権限昇格される虞から、無理になったようです[^jep-223]。
少し見た限り、 `jenkins.model.Jenkins.ADMINISTER` になってそうですね。

https://github.com/jenkinsci/jenkins/blob/master/core/src/main/java/jenkins/model/Jenkins.java#L5645-L5647

[^jenkins-matrix-based-security]: [Jenkins : Matrix-based security](https://wiki.jenkins.io/JENKINS/Matrix-based-security.html)
[^jep-223]: https://github.com/jenkinsci/jep/blob/master/jep/223/README.adoc

# 補足

「打ち上げ花火、下から見るか？横から見るか？」自体はドラマでやっていたもののアニメだったらしいですね。

どこかの口コミにもありましたが、ドラマ版と年齢設定を変えておきながら登場人物の台詞が幼稚な部分があり、自分は少々苦手でした。
ジェネリック新海誠を期待して観に行った節がありましたが、自然の風景と漠然とした主人公の抱く不安を搔き混ぜた上で主人公が心情を吐露する語りも無く、何を考えているのかあまり分からない感触がありましたね。

ただ個人的には、アニメだからこそできる列車の演出やラストの演出など、情景としては印象に残るもので好きでした。
新海誠作品に関して、印象を強調した景色を描くので印象派の絵に近いと大学の講義中に言っている先生もいましたが、アニメだからこそこうした景色の世界や果ては幻想的な世界が創造できると考えており、点に関しては良いものを見れたと思ってます。

自分の中では、花火が変な形に咲く世界に来てしまっていたのであれは幻。と思っていたのですが、どうなんでしょうね?
