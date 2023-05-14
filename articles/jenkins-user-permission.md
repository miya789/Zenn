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

ロールを使わず、公式ドキュメントに載っている[^jenkins-managing-security] [`matrix-auth`](https://plugins.jenkins.io/matrix-auth/)だけで頑張る方法も一応書いておきます。

::::details 実装例
前述のロールを自作するようなもので、割と面倒です。
特に削除は、全ての権限を調べて消して回る必要があるので厄介ですね。

<!-- 削除に関してロールと比べれば恩恵が分かる -->

:::message
尚、実装としては、Jenkins インスタンスが持つ `GlobalMatrixAuthorizationStrategy` のインスタンスを呼び出して、一部を追加や削除しています。
但し、これは空の entry が残る自体を誘発してしまい、構造が従来のものと異なる事態を引き起こします。
一応はエラー対応で何とかしてますが。
:::

主に[5. 付与する権限によるロールの作成 & jenkins ユーザーへロールのアサイン](#5.-付与する権限によるロールの作成-%26-jenkins-ユーザーへロールのアサイン)が変わります。

<!-- - 権限に関しては、全て列挙して省くやり方もありますが、グローバルセキュリティ画面と齟齬があるのであまりお薦めしません。 -->

## 追加(初回)

```diff groovy:追加
  def user_name = "sample_user"
- def role_name = "sample_role"

  def instance = jenkins.model.Jenkins.instance
- def strategy = new com.michelin.cio.hudson.plugins.rolestrategy.RoleBasedAuthorizationStrategy()
- def globalRoleMap = strategy.getRoleMap(com.synopsys.arc.jenkins.plugins.rolestrategy.RoleType.Global)
+ def strategy = new hudson.security.ProjectMatrixAuthorizationStrategy()

  def _permissions = new HashSet<>()
  _permissions.add(jenkins.model.Jenkins.READ) // これは必ず必要
  _permissions.add(hudson.model.Item.BUILD)
  _permissions.add(hudson.model.Item.READ)
  _permissions.add(hudson.model.Item.CONFIGURE)
  _permissions.add(hudson.model.Item.DELETE)
  permissions = new HashSet<>(_permissions.stream().filter{ it.enabled }.collect()) // enableでないPermissionを除去

- def role = new com.michelin.cio.hudson.plugins.rolestrategy.Role(role_name, permissions)
- globalRoleMap.addRole(role)
- globalRoleMap.assignRole(role, user_name)
+ def entry = org.jenkinsci.plugins.matrixauth.PermissionEntry.user(user_name)
+ for (permission in permissions) {
+   strategy.add(permission, entry)
+ }

  // 初期構築時は、誰もログインできなくならないように実施
- if (! globalRoleMap.getRoles().stream().anyMatch{p -> p.hasPermission(jenkins.model.Jenkins.ADMINISTER)}) {
-   def adminRole = new com.michelin.cio.hudson.plugins.rolestrategy.Role("admin", new HashSet(Arrays.asList(jenkins.model.Jenkins.ADMINISTER)))
-
-   globalRoleMap.addRole(adminRole)
-   globalRoleMap.assignRole(adminRole, "admin")
+ if (!strategy.getGrantedPermissionEntries().containsKey(jenkins.model.Jenkins.ADMINISTER)) {
+   def admin_entry = org.jenkinsci.plugins.matrixauth.PermissionEntry.user("admin")
+
+   strategy.add(jenkins.model.Jenkins.ADMINISTER, admin_entry);
  }

  instance.setAuthorizationStrategy(strategy)
  instance.save() // ログにも残せるので推奨
```

## 削除

エラー処理は、よしなに頑張ってください。
これは先程の追加の場合と差分を示しておきました。
当然、削除の際は、インスタンスがある筈なので、これを新規作成しませんので、ご注意ください。
尚、ユーザーアカウントの削除と言っても、正確には権限の削除です。

<!-- 例は、GitHubのこれを参照してください。 -->
<!-- 存在するキーを周って、ユーザーアカウント削除するべきでは? -->

```diff groovy:削除
  def user_name = "sample_user"

  def instance = jenkins.model.Jenkins.instance
- def strategy = new hudson.security.ProjectMatrixAuthorizationStrategy()
+ def strategy = instance.getAuthorizationStrategy()

  def _permissions = new HashSet<>()
  _permissions.add(jenkins.model.Jenkins.READ) // これは必ず必要
  _permissions.add(hudson.model.Item.BUILD)
  _permissions.add(hudson.model.Item.READ)
  _permissions.add(hudson.model.Item.CONFIGURE)
  _permissions.add(hudson.model.Item.DELETE)
  permissions = new HashSet<>(_permissions.stream().filter{ it.enabled }.collect()) // enableでないPermissionを除去
+
+ hudson.model.User.get(user_name).delete() // 外部と連携している場合は不要

  def entry = org.jenkinsci.plugins.matrixauth.PermissionEntry.user(user_name)
  for (permission in permissions) {
-   strategy.add(permission, entry)
+   if (!strategy.getGrantedPermissionEntries().containsKey(permission)) {
+     throw new Exception("the specified permission is not used")
+   }
+   if (!strategy.getGrantedPermissionEntries().get(permission).contains(entry)) {
+     throw new Exception("the specified user does not exist")
+   }
+   def entries = entries.get(permission)
+   entries.remove(permission, entry)
  }

- // 初期構築時は、誰もログインできなくならないように実施
- if (!strategy.getGrantedPermissionEntries().containsKey(jenkins.model.Jenkins.ADMINISTER)) {
-   def admin_entry = org.jenkinsci.plugins.matrixauth.PermissionEntry.user("admin")
-
-   strategy.add(jenkins.model.Jenkins.ADMINISTER, admin_entry);
- }

- instance.setAuthorizationStrategy(strategy)
  instance.save() // ログにも残せるので推奨
```

::::

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

:::message

- 初期設定ではエラーが出るので使えない場合がありました。

::::details ログ

```bash
jenkins-container | Running from: /usr/share/jenkins/jenkins.war
jenkins-container | webroot: EnvVars.masterEnvVars.get("JENKINS_HOME")
jenkins-container | 2022-09-23 11:43:38.899+0000 [id=1] INFO    winstone.Logger#logInternal: Beginning extraction from war file
jenkins-container | 2022-09-23 11:43:39.360+0000 [id=1] WARNING o.e.j.s.handler.ContextHandler#setContextPath: Empty contextPath
jenkins-container | 2022-09-23 11:43:39.398+0000 [id=1] INFO    org.eclipse.jetty.server.Server#doStart: jetty-10.0.11; built: 2022-06-21T21:12:44.640Z; git: d988aa016e0bb2de6fba84c1659049c72eae3e32; jvm 11.0.16.1+1
jenkins-container | 2022-09-23 11:43:39.558+0000 [id=1] INFO    o.e.j.w.StandardDescriptorProcessor#visitServlet: NO JSP Support for /, did not find org.eclipse.jetty.jsp.JettyJspServlet
jenkins-container | 2022-09-23 11:43:39.591+0000 [id=1] INFO    o.e.j.s.s.DefaultSessionIdManager#doStart: Session workerName=node0
jenkins-container | 2022-09-23 11:43:39.834+0000 [id=1] INFO    hudson.WebAppMain#contextInitialized: Jenkins home directory: /var/jenkins_home found at: EnvVars.masterEnvVars.get("JENKINS_HOME")
jenkins-container | 2022-09-23 11:43:39.939+0000 [id=1] INFO    o.e.j.s.handler.ContextHandler#doStart: Started w.@4d8286c4{Jenkins v2.361.1,/,file:///var/jenkins_home/war/,AVAILABLE}{/var/jenkins_home/war}
jenkins-container | 2022-09-23 11:43:39.952+0000 [id=1] INFO    o.e.j.server.AbstractConnector#doStart: Started ServerConnector@e84a8e1{HTTP/1.1, (http/1.1)}{0.0.0.0:8080}
jenkins-container | 2022-09-23 11:43:39.965+0000 [id=1] INFO    org.eclipse.jetty.server.Server#doStart: Started Server@32c8e539{STARTING}[10.0.11,sto=0] @1386ms
jenkins-container | 2022-09-23 11:43:39.968+0000 [id=35]        INFO    winstone.Logger#logInternal: Winstone Servlet Engine running: controlPort=disabled
jenkins-container | 2022-09-23 11:43:40.136+0000 [id=42]        INFO    jenkins.InitReactorRunner$1#onAttained: Started initialization
jenkins-container | 2022-09-23 11:43:40.151+0000 [id=61]        INFO    jenkins.InitReactorRunner$1#onAttained: Listed all plugins
jenkins-container | 2022-09-23 11:43:40.608+0000 [id=70]        INFO    jenkins.InitReactorRunner$1#onAttained: Prepared all plugins
jenkins-container | 2022-09-23 11:43:40.612+0000 [id=40]        INFO    jenkins.InitReactorRunner$1#onAttained: Started all plugins
jenkins-container | 2022-09-23 11:43:40.615+0000 [id=55]        INFO    jenkins.InitReactorRunner$1#onAttained: Augmented all extensions
jenkins-container | 2022-09-23 11:43:40.748+0000 [id=57]        INFO    jenkins.InitReactorRunner$1#onAttained: System config loaded
jenkins-container | 2022-09-23 11:43:40.749+0000 [id=57]        INFO    jenkins.InitReactorRunner$1#onAttained: System config adapted
jenkins-container | 2022-09-23 11:43:40.750+0000 [id=53]        INFO    jenkins.InitReactorRunner$1#onAttained: Loaded all jobs
jenkins-container | 2022-09-23 11:43:40.752+0000 [id=47]        INFO    jenkins.InitReactorRunner$1#onAttained: Configuration for all jobs updated
jenkins-container | 2022-09-23 11:43:40.764+0000 [id=84]        INFO    hudson.model.AsyncPeriodicWork#lambda$doRun$1: Started Download metadata
jenkins-container | 2022-09-23 11:43:40.771+0000 [id=84]        INFO    hudson.util.Retrier#start: Attempt #1 to do the action check updates server
jenkins-container | WARNING: An illegal reflective access operation has occurred
jenkins-container | WARNING: Illegal reflective access by org.codehaus.groovy.vmplugin.v7.Java7$1 (file:/var/jenkins_home/war/WEB-INF/lib/groovy-all-2.4.21.jar) to constructor java.lang.invoke.MethodHandles$Lookup(java.lang.Class,int)
jenkins-container | WARNING: Please consider reporting this to the maintainers of org.codehaus.groovy.vmplugin.v7.Java7$1
jenkins-container | WARNING: Use --illegal-access=warn to enable warnings of further illegal reflective access operations
jenkins-container | WARNING: All illegal access operations will be denied in a future release
jenkins-container | 2022-09-23 11:43:40.887+0000 [id=65]        INFO    j.util.groovy.GroovyHookScript#execute: Executing /var/jenkins_home/init.groovy.d/00_plugin.groovy
jenkins-container | 2022-09-23 11:43:41.033+0000 [id=60]        INFO    jenkins.install.SetupWizard#init:
jenkins-container |
jenkins-container | *************************************************************
jenkins-container | *************************************************************
jenkins-container | *************************************************************
jenkins-container |
jenkins-container | Jenkins initial setup is required. An admin user has been created and a password generated.
jenkins-container | Please use the following password to proceed to installation:
jenkins-container |
jenkins-container | f6788e153850448ca3cecd5aa0eba046
jenkins-container |
jenkins-container | This may also be found at: /var/jenkins_home/secrets/initialAdminPassword
jenkins-container |
jenkins-container | *************************************************************
jenkins-container | *************************************************************
jenkins-container | *************************************************************
jenkins-container |
jenkins-container | 2022-09-23 11:44:01.060+0000 [id=84]        INFO    hudson.util.Retrier#start: The attempt #1 to do the action check updates server failed with an allowed exception:
jenkins-container | java.net.SocketTimeoutException: connect timed out
jenkins-container |     at java.base/java.net.PlainSocketImpl.socketConnect(Native Method)
jenkins-container |     at java.base/java.net.AbstractPlainSocketImpl.doConnect(AbstractPlainSocketImpl.java:412)
jenkins-container |     at java.base/java.net.AbstractPlainSocketImpl.connectToAddress(AbstractPlainSocketImpl.java:255)
jenkins-container |     at java.base/java.net.AbstractPlainSocketImpl.connect(AbstractPlainSocketImpl.java:237)
jenkins-container |     at java.base/java.net.SocksSocketImpl.connect(SocksSocketImpl.java:392)
jenkins-container |     at java.base/java.net.Socket.connect(Socket.java:609)
jenkins-container |     at java.base/sun.security.ssl.SSLSocketImpl.connect(SSLSocketImpl.java:305)
jenkins-container |     at java.base/sun.net.NetworkClient.doConnect(NetworkClient.java:177)
jenkins-container |     at java.base/sun.net.www.http.HttpClient.openServer(HttpClient.java:508)
jenkins-container |     at java.base/sun.net.www.http.HttpClient.openServer(HttpClient.java:603)
jenkins-container |     at java.base/sun.net.www.protocol.https.HttpsClient.<init>(HttpsClient.java:266)
jenkins-container |     at java.base/sun.net.www.protocol.https.HttpsClient.New(HttpsClient.java:373)
jenkins-container |     at java.base/sun.net.www.protocol.https.AbstractDelegateHttpsURLConnection.getNewHttpClient(AbstractDelegateHttpsURLConnection.java:207)
jenkins-container |     at java.base/sun.net.www.protocol.http.HttpURLConnection.plainConnect0(HttpURLConnection.java:1187)
jenkins-container |     at java.base/sun.net.www.protocol.http.HttpURLConnection.plainConnect(HttpURLConnection.java:1081)
jenkins-container |     at java.base/sun.net.www.protocol.https.AbstractDelegateHttpsURLConnection.connect(AbstractDelegateHttpsURLConnection.java:193)
jenkins-container |     at java.base/sun.net.www.protocol.http.HttpURLConnection.getInputStream0(HttpURLConnection.java:1592)
jenkins-container |     at java.base/sun.net.www.protocol.http.HttpURLConnection.getInputStream(HttpURLConnection.java:1520)
jenkins-container |     at java.base/sun.net.www.protocol.https.HttpsURLConnectionImpl.getInputStream(HttpsURLConnectionImpl.java:250)
jenkins-container |     at hudson.model.DownloadService.loadJSON(DownloadService.java:122)
jenkins-container |     at hudson.model.UpdateSite.updateDirectlyNow(UpdateSite.java:219)
jenkins-container |     at hudson.model.UpdateSite.updateDirectlyNow(UpdateSite.java:214)
jenkins-container |     at hudson.PluginManager.checkUpdatesServer(PluginManager.java:1951)
jenkins-container |     at hudson.util.Retrier.start(Retrier.java:62)
jenkins-container |     at hudson.PluginManager.doCheckUpdatesServer(PluginManager.java:1922)
jenkins-container |     at jenkins.DailyCheck.execute(DailyCheck.java:93)
jenkins-container |     at hudson.model.AsyncPeriodicWork.lambda$doRun$1(AsyncPeriodicWork.java:102)
jenkins-container |     at java.base/java.lang.Thread.run(Thread.java:829)
jenkins-container | 2022-09-23 11:44:01.062+0000 [id=84]        INFO    hudson.util.Retrier#start: Calling the listener of the allowed exception 'connect timed out' at the attempt #1 to do the action check updates server
jenkins-container | 2022-09-23 11:44:01.064+0000 [id=84]        INFO    hudson.util.Retrier#start: Attempted the action check updates server for 1 time(s) with no success
jenkins-container | 2022-09-23 11:44:01.066+0000 [id=84]        SEVERE  hudson.PluginManager#doCheckUpdatesServer: Error checking update sites for 1 attempt(s). Last exception was: SocketTimeoutException: connect timed out
jenkins-container | 2022-09-23 11:44:01.068+0000 [id=84]        INFO    hudson.model.AsyncPeriodicWork#lambda$doRun$1: Finished Download metadata. 20,302 ms
jenkins-container | 2022-09-23 11:44:01.184+0000 [id=60]        WARNING hudson.model.UpdateCenter#updateDefaultSite: Upgrading Jenkins. Failed to update the default Update Site 'default'. Plugin upgrades may fail.
jenkins-container | java.net.SocketTimeoutException: connect timed out
jenkins-container |     at java.base/java.net.PlainSocketImpl.socketConnect(Native Method)
jenkins-container |     at java.base/java.net.AbstractPlainSocketImpl.doConnect(AbstractPlainSocketImpl.java:412)
jenkins-container |     at java.base/java.net.AbstractPlainSocketImpl.connectToAddress(AbstractPlainSocketImpl.java:255)
jenkins-container |     at java.base/java.net.AbstractPlainSocketImpl.connect(AbstractPlainSocketImpl.java:237)
jenkins-container |     at java.base/java.net.SocksSocketImpl.connect(SocksSocketImpl.java:392)
jenkins-container |     at java.base/java.net.Socket.connect(Socket.java:609)
jenkins-container |     at java.base/sun.security.ssl.SSLSocketImpl.connect(SSLSocketImpl.java:305)
jenkins-container |     at java.base/sun.net.NetworkClient.doConnect(NetworkClient.java:177)
jenkins-container |     at java.base/sun.net.www.http.HttpClient.openServer(HttpClient.java:508)
jenkins-container |     at java.base/sun.net.www.http.HttpClient.openServer(HttpClient.java:603)
jenkins-container |     at java.base/sun.net.www.protocol.https.HttpsClient.<init>(HttpsClient.java:266)
jenkins-container |     at java.base/sun.net.www.protocol.https.HttpsClient.New(HttpsClient.java:373)
jenkins-container |     at java.base/sun.net.www.protocol.https.AbstractDelegateHttpsURLConnection.getNewHttpClient(AbstractDelegateHttpsURLConnection.java:207)
jenkins-container |     at java.base/sun.net.www.protocol.http.HttpURLConnection.plainConnect0(HttpURLConnection.java:1187)
jenkins-container |     at java.base/sun.net.www.protocol.http.HttpURLConnection.plainConnect(HttpURLConnection.java:1081)
jenkins-container |     at java.base/sun.net.www.protocol.https.AbstractDelegateHttpsURLConnection.connect(AbstractDelegateHttpsURLConnection.java:193)
jenkins-container |     at java.base/sun.net.www.protocol.http.HttpURLConnection.getInputStream0(HttpURLConnection.java:1592)
jenkins-container |     at java.base/sun.net.www.protocol.http.HttpURLConnection.getInputStream(HttpURLConnection.java:1520)
jenkins-container |     at java.base/sun.net.www.protocol.https.HttpsURLConnectionImpl.getInputStream(HttpsURLConnectionImpl.java:250)
jenkins-container |     at hudson.model.DownloadService.loadJSON(DownloadService.java:122)
jenkins-container |     at hudson.model.UpdateSite.updateDirectlyNow(UpdateSite.java:219)
jenkins-container |     at hudson.model.UpdateSite.updateDirectlyNow(UpdateSite.java:214)
jenkins-container |     at hudson.model.UpdateCenter.updateDefaultSite(UpdateCenter.java:2672)
jenkins-container |     at jenkins.install.SetupWizard.init(SetupWizard.java:209)
jenkins-container |     at jenkins.install.InstallState$InitialSecuritySetup.initializeState(InstallState.java:182)
jenkins-container |     at jenkins.model.Jenkins.setInstallState(Jenkins.java:1133)
jenkins-container |     at jenkins.install.InstallUtil.proceedToNextStateFrom(InstallUtil.java:99)
jenkins-container |     at jenkins.install.InstallState$Unknown.initializeState(InstallState.java:88)
jenkins-container |     at jenkins.model.Jenkins$15.run(Jenkins.java:3499)
jenkins-container |     at org.jvnet.hudson.reactor.TaskGraphBuilder$TaskImpl.run(TaskGraphBuilder.java:175)
jenkins-container |     at org.jvnet.hudson.reactor.Reactor.runTask(Reactor.java:305)
jenkins-container |     at jenkins.model.Jenkins$5.runTask(Jenkins.java:1160)
jenkins-container |     at org.jvnet.hudson.reactor.Reactor$2.run(Reactor.java:222)
jenkins-container |     at org.jvnet.hudson.reactor.Reactor$Node.run(Reactor.java:121)
jenkins-container |     at jenkins.security.ImpersonatingExecutorService$1.run(ImpersonatingExecutorService.java:70)
jenkins-container |     at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1128)
jenkins-container |     at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:628)
jenkins-container |     at java.base/java.lang.Thread.run(Thread.java:829)
jenkins-container | 2022-09-23 11:44:01.246+0000 [id=65]        SEVERE  jenkins.InitReactorRunner$1#onTaskFailed: Failed GroovyInitScript.init
jenkins-container | java.net.SocketTimeoutException: connect timed out
jenkins-container |     at java.base/java.net.PlainSocketImpl.socketConnect(Native Method)
jenkins-container |     at java.base/java.net.AbstractPlainSocketImpl.doConnect(AbstractPlainSocketImpl.java:412)
jenkins-container |     at java.base/java.net.AbstractPlainSocketImpl.connectToAddress(AbstractPlainSocketImpl.java:255)
jenkins-container |     at java.base/java.net.AbstractPlainSocketImpl.connect(AbstractPlainSocketImpl.java:237)
jenkins-container |     at java.base/java.net.SocksSocketImpl.connect(SocksSocketImpl.java:392)
jenkins-container |     at java.base/java.net.Socket.connect(Socket.java:609)
jenkins-container |     at java.base/sun.security.ssl.SSLSocketImpl.connect(SSLSocketImpl.java:305)
jenkins-container |     at java.base/sun.net.NetworkClient.doConnect(NetworkClient.java:177)
jenkins-container |     at java.base/sun.net.www.http.HttpClient.openServer(HttpClient.java:508)
jenkins-container |     at java.base/sun.net.www.http.HttpClient.openServer(HttpClient.java:603)
jenkins-container |     at java.base/sun.net.www.protocol.https.HttpsClient.<init>(HttpsClient.java:266)
jenkins-container |     at java.base/sun.net.www.protocol.https.HttpsClient.New(HttpsClient.java:373)
jenkins-container |     at java.base/sun.net.www.protocol.https.AbstractDelegateHttpsURLConnection.getNewHttpClient(AbstractDelegateHttpsURLConnection.java:207)
jenkins-container |     at java.base/sun.net.www.protocol.http.HttpURLConnection.plainConnect0(HttpURLConnection.java:1187)
jenkins-container |     at java.base/sun.net.www.protocol.http.HttpURLConnection.plainConnect(HttpURLConnection.java:1081)
jenkins-container |     at java.base/sun.net.www.protocol.https.AbstractDelegateHttpsURLConnection.connect(AbstractDelegateHttpsURLConnection.java:193)
jenkins-container |     at java.base/sun.net.www.protocol.http.HttpURLConnection.getInputStream0(HttpURLConnection.java:1592)
jenkins-container |     at java.base/sun.net.www.protocol.http.HttpURLConnection.getInputStream(HttpURLConnection.java:1520)
jenkins-container |     at java.base/sun.net.www.protocol.https.HttpsURLConnectionImpl.getInputStream(HttpsURLConnectionImpl.java:250)
jenkins-container |     at hudson.model.DownloadService.loadJSON(DownloadService.java:122)
jenkins-container |     at hudson.model.UpdateSite.updateDirectlyNow(UpdateSite.java:219)
jenkins-container |     at hudson.model.UpdateSite$1.call(UpdateSite.java:199)
jenkins-container |     at hudson.model.UpdateSite$1.call(UpdateSite.java:197)
jenkins-container |     at java.base/java.util.concurrent.FutureTask.run(FutureTask.java:264)
jenkins-container | Caused: java.util.concurrent.ExecutionException
jenkins-container |     at java.base/java.util.concurrent.FutureTask.report(FutureTask.java:122)
jenkins-container |     at java.base/java.util.concurrent.FutureTask.get(FutureTask.java:191)
jenkins-container |     at hudson.model.UpdateCenter.updateAllSites(UpdateCenter.java:1114)
jenkins-container |     at hudson.model.UpdateCenter$updateAllSites.call(Unknown Source)
jenkins-container |     at org.codehaus.groovy.runtime.callsite.CallSiteArray.defaultCall(CallSiteArray.java:47)
jenkins-container |     at org.codehaus.groovy.runtime.callsite.AbstractCallSite.call(AbstractCallSite.java:116)
jenkins-container |     at org.codehaus.groovy.runtime.callsite.AbstractCallSite.call(AbstractCallSite.java:120)
jenkins-container |     at 00_plugin.run(00_plugin.groovy:9)
jenkins-container |     at groovy.lang.GroovyShell.evaluate(GroovyShell.java:574)
jenkins-container |     at jenkins.util.groovy.GroovyHookScript.execute(GroovyHookScript.java:136)
jenkins-container |     at jenkins.util.groovy.GroovyHookScript.execute(GroovyHookScript.java:126)
jenkins-container |     at jenkins.util.groovy.GroovyHookScript.run(GroovyHookScript.java:109)
jenkins-container |     at hudson.init.impl.GroovyInitScript.init(GroovyInitScript.java:42)
jenkins-container | Caused: java.lang.reflect.InvocationTargetException
jenkins-container |     at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
jenkins-container |     at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
jenkins-container |     at java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
jenkins-container |     at java.base/java.lang.reflect.Method.invoke(Method.java:566)
jenkins-container |     at hudson.init.TaskMethodFinder.invoke(TaskMethodFinder.java:109)
jenkins-container | Caused: java.lang.Error
jenkins-container |     at hudson.init.TaskMethodFinder.invoke(TaskMethodFinder.java:115)
jenkins-container |     at hudson.init.TaskMethodFinder$TaskImpl.run(TaskMethodFinder.java:185)
jenkins-container |     at org.jvnet.hudson.reactor.Reactor.runTask(Reactor.java:305)
jenkins-container |     at jenkins.model.Jenkins$5.runTask(Jenkins.java:1160)
jenkins-container |     at org.jvnet.hudson.reactor.Reactor$2.run(Reactor.java:222)
jenkins-container |     at org.jvnet.hudson.reactor.Reactor$Node.run(Reactor.java:121)
jenkins-container |     at jenkins.security.ImpersonatingExecutorService$1.run(ImpersonatingExecutorService.java:70)
jenkins-container |     at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1128)
jenkins-container |     at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:628)
jenkins-container |     at java.base/java.lang.Thread.run(Thread.java:829)
jenkins-container | 2022-09-23 11:44:01.248+0000 [id=32]        SEVERE  hudson.util.BootFailure#publish: Failed to initialize Jenkins
jenkins-container | java.net.SocketTimeoutException: connect timed out
jenkins-container |     at java.base/java.net.PlainSocketImpl.socketConnect(Native Method)
jenkins-container |     at java.base/java.net.AbstractPlainSocketImpl.doConnect(AbstractPlainSocketImpl.java:412)
jenkins-container |     at java.base/java.net.AbstractPlainSocketImpl.connectToAddress(AbstractPlainSocketImpl.java:255)
jenkins-container |     at java.base/java.net.AbstractPlainSocketImpl.connect(AbstractPlainSocketImpl.java:237)
jenkins-container |     at java.base/java.net.SocksSocketImpl.connect(SocksSocketImpl.java:392)
jenkins-container |     at java.base/java.net.Socket.connect(Socket.java:609)
jenkins-container |     at java.base/sun.security.ssl.SSLSocketImpl.connect(SSLSocketImpl.java:305)
jenkins-container |     at java.base/sun.net.NetworkClient.doConnect(NetworkClient.java:177)
jenkins-container |     at java.base/sun.net.www.http.HttpClient.openServer(HttpClient.java:508)
jenkins-container |     at java.base/sun.net.www.http.HttpClient.openServer(HttpClient.java:603)
jenkins-container |     at java.base/sun.net.www.protocol.https.HttpsClient.<init>(HttpsClient.java:266)
jenkins-container |     at java.base/sun.net.www.protocol.https.HttpsClient.New(HttpsClient.java:373)
jenkins-container |     at java.base/sun.net.www.protocol.https.AbstractDelegateHttpsURLConnection.getNewHttpClient(AbstractDelegateHttpsURLConnection.java:207)
jenkins-container |     at java.base/sun.net.www.protocol.http.HttpURLConnection.plainConnect0(HttpURLConnection.java:1187)
jenkins-container |     at java.base/sun.net.www.protocol.http.HttpURLConnection.plainConnect(HttpURLConnection.java:1081)
jenkins-container |     at java.base/sun.net.www.protocol.https.AbstractDelegateHttpsURLConnection.connect(AbstractDelegateHttpsURLConnection.java:193)
jenkins-container |     at java.base/sun.net.www.protocol.http.HttpURLConnection.getInputStream0(HttpURLConnection.java:1592)
jenkins-container |     at java.base/sun.net.www.protocol.http.HttpURLConnection.getInputStream(HttpURLConnection.java:1520)
jenkins-container |     at java.base/sun.net.www.protocol.https.HttpsURLConnectionImpl.getInputStream(HttpsURLConnectionImpl.java:250)
jenkins-container |     at hudson.model.DownloadService.loadJSON(DownloadService.java:122)
jenkins-container |     at hudson.model.UpdateSite.updateDirectlyNow(UpdateSite.java:219)
jenkins-container |     at hudson.model.UpdateSite$1.call(UpdateSite.java:199)
jenkins-container |     at hudson.model.UpdateSite$1.call(UpdateSite.java:197)
jenkins-container |     at java.base/java.util.concurrent.FutureTask.run(FutureTask.java:264)
jenkins-container | Caused: java.util.concurrent.ExecutionException
jenkins-container |     at java.base/java.util.concurrent.FutureTask.report(FutureTask.java:122)
jenkins-container |     at java.base/java.util.concurrent.FutureTask.get(FutureTask.java:191)
jenkins-container |     at hudson.model.UpdateCenter.updateAllSites(UpdateCenter.java:1114)
jenkins-container |     at hudson.model.UpdateCenter$updateAllSites.call(Unknown Source)
jenkins-container |     at org.codehaus.groovy.runtime.callsite.CallSiteArray.defaultCall(CallSiteArray.java:47)
jenkins-container |     at org.codehaus.groovy.runtime.callsite.AbstractCallSite.call(AbstractCallSite.java:116)
jenkins-container |     at org.codehaus.groovy.runtime.callsite.AbstractCallSite.call(AbstractCallSite.java:120)
jenkins-container |     at 00_plugin.run(00_plugin.groovy:9)
jenkins-container |     at groovy.lang.GroovyShell.evaluate(GroovyShell.java:574)
jenkins-container |     at jenkins.util.groovy.GroovyHookScript.execute(GroovyHookScript.java:136)
jenkins-container |     at jenkins.util.groovy.GroovyHookScript.execute(GroovyHookScript.java:126)
jenkins-container |     at jenkins.util.groovy.GroovyHookScript.run(GroovyHookScript.java:109)
jenkins-container |     at hudson.init.impl.GroovyInitScript.init(GroovyInitScript.java:42)
jenkins-container | Caused: java.lang.reflect.InvocationTargetException
jenkins-container |     at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
jenkins-container |     at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
jenkins-container |     at java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
jenkins-container |     at java.base/java.lang.reflect.Method.invoke(Method.java:566)
jenkins-container |     at hudson.init.TaskMethodFinder.invoke(TaskMethodFinder.java:109)
jenkins-container | Caused: java.lang.Error
jenkins-container |     at hudson.init.TaskMethodFinder.invoke(TaskMethodFinder.java:115)
jenkins-container |     at hudson.init.TaskMethodFinder$TaskImpl.run(TaskMethodFinder.java:185)
jenkins-container |     at org.jvnet.hudson.reactor.Reactor.runTask(Reactor.java:305)
jenkins-container |     at jenkins.model.Jenkins$5.runTask(Jenkins.java:1160)
jenkins-container |     at org.jvnet.hudson.reactor.Reactor$2.run(Reactor.java:222)
jenkins-container |     at org.jvnet.hudson.reactor.Reactor$Node.run(Reactor.java:121)
jenkins-container |     at jenkins.security.ImpersonatingExecutorService$1.run(ImpersonatingExecutorService.java:70)
jenkins-container |     at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1128)
jenkins-container |     at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:628)
jenkins-container |     at java.base/java.lang.Thread.run(Thread.java:829)
jenkins-container | Caused: org.jvnet.hudson.reactor.ReactorException
jenkins-container |     at org.jvnet.hudson.reactor.Reactor.execute(Reactor.java:291)
jenkins-container |     at jenkins.InitReactorRunner.run(InitReactorRunner.java:49)
jenkins-container |     at jenkins.model.Jenkins.executeReactor(Jenkins.java:1195)
jenkins-container |     at jenkins.model.Jenkins.<init>(Jenkins.java:985)
jenkins-container |     at hudson.model.Hudson.<init>(Hudson.java:86)
jenkins-container |     at hudson.model.Hudson.<init>(Hudson.java:82)
jenkins-container |     at hudson.WebAppMain$3.run(WebAppMain.java:247)
jenkins-container | Caused: hudson.util.HudsonFailedToLoad
jenkins-container |     at hudson.WebAppMain$3.run(WebAppMain.java:264)
jenkins-container | 2022-09-23 11:44:01.254+0000 [id=32]        INFO    hudson.lifecycle.Lifecycle#onStatusUpdate: Stopping Jenkins
jenkins-container | 2022-09-23 11:44:01.257+0000 [id=32]        INFO    jenkins.model.Jenkins$16#onAttained: Started termination
jenkins-container | 2022-09-23 11:44:01.263+0000 [id=32]        INFO    jenkins.model.Jenkins$16#onAttained: Completed termination
jenkins-container | 2022-09-23 11:44:01.263+0000 [id=32]        INFO    jenkins.model.Jenkins#_cleanUpDisconnectComputers: Starting node disconnection
jenkins-container | 2022-09-23 11:44:01.284+0000 [id=92]        WARNING h.ExtensionFinder$GuiceFinder$FaultTolerantScope$1#error: Failed to instantiate Key[type=jenkins.slaves.JnlpSlaveAgentProtocol4, annotation=[none]]; skipping this component
jenkins-container | java.security.KeyStoreException: JENKINS-41987: no X509Certificate found; perhaps instance-identity plugin is not installed
jenkins-container |     at jenkins.slaves.JnlpSlaveAgentProtocol4.<init>(JnlpSlaveAgentProtocol4.java:106)
jenkins-container |     at jenkins.slaves.JnlpSlaveAgentProtocol4$$FastClassByGuice$$308450093.GUICE$TRAMPOLINE(<generated>)
jenkins-container |     at jenkins.slaves.JnlpSlaveAgentProtocol4$$FastClassByGuice$$308450093.apply(<generated>)
jenkins-container |     at com.google.inject.internal.DefaultConstructionProxyFactory$FastClassProxy.newInstance(DefaultConstructionProxyFactory.java:82)
jenkins-container |     at com.google.inject.internal.ConstructorInjector.provision(ConstructorInjector.java:114)
jenkins-container |     at com.google.inject.internal.ConstructorInjector.access$000(ConstructorInjector.java:33)
jenkins-container |     at com.google.inject.internal.ConstructorInjector$1.call(ConstructorInjector.java:98)
jenkins-container |     at com.google.inject.internal.ProvisionListenerStackCallback$Provision.provision(ProvisionListenerStackCallback.java:109)
jenkins-container |     at hudson.ExtensionFinder$GuiceFinder$SezpozModule.onProvision(ExtensionFinder.java:568)
jenkins-container |     at com.google.inject.internal.ProvisionListenerStackCallback$Provision.provision(ProvisionListenerStackCallback.java:117)
jenkins-container |     at com.google.inject.internal.ProvisionListenerStackCallback.provision(ProvisionListenerStackCallback.java:66)
jenkins-container |     at com.google.inject.internal.ConstructorInjector.construct(ConstructorInjector.java:93)
jenkins-container |     at com.google.inject.internal.ConstructorBindingImpl$Factory.get(ConstructorBindingImpl.java:296)
jenkins-container |     at com.google.inject.internal.ProviderToInternalFactoryAdapter.get(ProviderToInternalFactoryAdapter.java:40)
jenkins-container | Caused: com.google.inject.ProvisionException: Unable to provision, see the following errors:
jenkins-container |
jenkins-container | 1) [Guice/ErrorInjectingConstructor]: KeyStoreException: JENKINS-41987: no X509Certificate found; perhaps instance-identity plugin is not installed
jenkins-container |   at JnlpSlaveAgentProtocol4.<init>(JnlpSlaveAgentProtocol4.java:102)
jenkins-container |
jenkins-container | Learn more:
jenkins-container |   https://github.com/google/guice/wiki/ERROR_INJECTING_CONSTRUCTOR
jenkins-container |
jenkins-container | 1 error
jenkins-container |
jenkins-container | ======================
jenkins-container | Full classname legend:
jenkins-container | ======================
jenkins-container | JnlpSlaveAgentProtocol4: "jenkins.slaves.JnlpSlaveAgentProtocol4"
jenkins-container | KeyStoreException:       "java.security.KeyStoreException"
jenkins-container | ========================
jenkins-container | End of classname legend:
jenkins-container | ========================
jenkins-container |
jenkins-container |     at com.google.inject.internal.InternalProvisionException.toProvisionException(InternalProvisionException.java:251)
jenkins-container |     at com.google.inject.internal.ProviderToInternalFactoryAdapter.get(ProviderToInternalFactoryAdapter.java:43)
jenkins-container |     at com.google.inject.internal.SingletonScope$1.get(SingletonScope.java:169)
jenkins-container |     at hudson.ExtensionFinder$GuiceFinder$FaultTolerantScope$1.get(ExtensionFinder.java:444)
jenkins-container |     at com.google.inject.internal.InternalFactoryToProviderAdapter.get(InternalFactoryToProviderAdapter.java:45)
jenkins-container |     at com.google.inject.internal.InjectorImpl$1.get(InjectorImpl.java:1100)
jenkins-container |     at hudson.ExtensionFinder$GuiceFinder._find(ExtensionFinder.java:402)
jenkins-container |     at hudson.ExtensionFinder$GuiceFinder.find(ExtensionFinder.java:393)
jenkins-container |     at hudson.ClassicPluginStrategy.findComponents(ClassicPluginStrategy.java:359)
jenkins-container |     at hudson.ExtensionList.load(ExtensionList.java:384)
jenkins-container |     at hudson.ExtensionList.ensureLoaded(ExtensionList.java:320)
jenkins-container |     at hudson.ExtensionList.iterator(ExtensionList.java:172)
jenkins-container |     at jenkins.AgentProtocol.of(AgentProtocol.java:111)
jenkins-container |     at hudson.TcpSlaveAgentListener$ConnectionHandler.run(TcpSlaveAgentListener.java:277)
jenkins-container | 2022-09-23 11:44:01.305+0000 [id=32]        INFO    jenkins.model.Jenkins#_cleanUpShutdownPluginManager: Stopping plugin manager
jenkins-container | 2022-09-23 11:44:01.305+0000 [id=32]        INFO    jenkins.model.Jenkins#_cleanUpPersistQueue: Persisting build queue
jenkins-container | 2022-09-23 11:44:01.312+0000 [id=32]        INFO    jenkins.model.Jenkins#_cleanUpAwaitDisconnects: Waiting for node disconnection completion
jenkins-container | 2022-09-23 11:44:01.313+0000 [id=32]        INFO    hudson.lifecycle.Lifecycle#onStatusUpdate: Jenkins stopped
```

::::

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
