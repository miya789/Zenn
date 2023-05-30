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

:::

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
Permission の関数だけでも複数あるので、各 Permission クラス定数がどんな情報を持っているのか把握するのが大変でした……
同じものを、以下にも上げておきます。
https://github.com/miya789/Zenn/blob/main/appendix/jenkins-user-permission/Permissions.java

<!-- csvは以下に上げます。 -->

[^jenkins-matrix-based-security-with]: [midweekmidmorning: Jenkins Matrix Based Security with Groovy Scripts](http://midweekmidmorning.blogspot.com/2016/06/jenkins-matrix-based-security-with.html)

| group id               | enabled | Permission Class Constant                                                 | id                                                                  | toString                                                                              | impliedBy                                                     |
| ---------------------- | ------- | ------------------------------------------------------------------------- | ------------------------------------------------------------------- | ------------------------------------------------------------------------------------- | ------------------------------------------------------------- |
| **Overall**            | `true`  | `jenkins.model.Jenkins.ADMINISTER`                                        | hudson.model.Hudson.Administer                                      | Permission[class hudson.model.Hudson,Administer]                                      | null                                                          |
| **Overall**            | `false` | `jenkins.model.Jenkins.MANAGE`                                            | hudson.model.Hudson.Manage                                          | Permission[class hudson.model.Hudson,Manage]                                          | Permission[class hudson.model.Hudson,Administer]              |
| **Overall**            | `false` | `jenkins.model.Jenkins.SYSTEM_READ`                                       | hudson.model.Hudson.SystemRead                                      | Permission[class hudson.model.Hudson,SystemRead]                                      | Permission[class hudson.model.Hudson,Administer]              |
| **Overall**            | `true`  | `jenkins.model.Jenkins.READ`                                              | hudson.model.Hudson.Read                                            | Permission[class hudson.model.Hudson,Read]                                            | Permission[class hudson.security.Permission,GenericRead]      |
| **Overall**            | `true`  | `jenkins.model.Jenkins.RUN_SCRIPTS`                                       | hudson.model.Hudson.RunScripts                                      | Permission[class hudson.model.Hudson,RunScripts]                                      | Permission[class hudson.model.Hudson,Administer]              |
| **Credentials**        | `false` | `com.cloudbees.plugins.credentials.CredentialsProvider.USE_OWN`           | com.cloudbees.plugins.credentials.CredentialsProvider.UseOwn        | Permission[class com.cloudbees.plugins.credentials.CredentialsProvider,UseOwn]        | Permission[interface hudson.model.Item,Build]                 |
| **Credentials**        | `false` | `com.cloudbees.plugins.credentials.CredentialsProvider.USE_ITEM`          | com.cloudbees.plugins.credentials.CredentialsProvider.UseItem       | Permission[class com.cloudbees.plugins.credentials.CredentialsProvider,UseItem]       | Permission[interface hudson.model.Item,Configure]             |
| **Credentials**        | `true`  | `com.cloudbees.plugins.credentials.CredentialsProvider.CREATE`            | com.cloudbees.plugins.credentials.CredentialsProvider.Create        | Permission[class com.cloudbees.plugins.credentials.CredentialsProvider,Create]        | Permission[class hudson.security.Permission,GenericCreate]    |
| **Credentials**        | `true`  | `com.cloudbees.plugins.credentials.CredentialsProvider.UPDATE`            | com.cloudbees.plugins.credentials.CredentialsProvider.Update        | Permission[class com.cloudbees.plugins.credentials.CredentialsProvider,Update]        | Permission[class hudson.security.Permission,GenericUpdate]    |
| **Credentials**        | `true`  | `com.cloudbees.plugins.credentials.CredentialsProvider.VIEW`              | com.cloudbees.plugins.credentials.CredentialsProvider.View          | Permission[class com.cloudbees.plugins.credentials.CredentialsProvider,View]          | Permission[class hudson.security.Permission,GenericRead]      |
| **Credentials**        | `true`  | `com.cloudbees.plugins.credentials.CredentialsProvider.DELETE`            | com.cloudbees.plugins.credentials.CredentialsProvider.Delete        | Permission[class com.cloudbees.plugins.credentials.CredentialsProvider,Delete]        | Permission[class hudson.security.Permission,GenericDelete]    |
| **Credentials**        | `true`  | `com.cloudbees.plugins.credentials.CredentialsProvider.MANAGE_DOMAINS`    | com.cloudbees.plugins.credentials.CredentialsProvider.ManageDomains | Permission[class com.cloudbees.plugins.credentials.CredentialsProvider,ManageDomains] | Permission[class hudson.security.Permission,GenericConfigure] |
| **Agent**              | `true`  | `hudson.model.Computer.CONFIGURE`                                         | hudson.model.Computer.Configure                                     | Permission[class hudson.model.Computer,Configure]                                     | Permission[class hudson.security.Permission,GenericConfigure] |
| **Agent**              | `false` | `hudson.model.Computer.EXTENDED_READ`                                     | hudson.model.Computer.ExtendedRead                                  | Permission[class hudson.model.Computer,ExtendedRead]                                  | Permission[class hudson.model.Computer,Configure]             |
| **Agent**              | `true`  | `hudson.model.Computer.DELETE`                                            | hudson.model.Computer.Delete                                        | Permission[class hudson.model.Computer,Delete]                                        | Permission[class hudson.security.Permission,GenericDelete]    |
| **Agent**              | `true`  | `hudson.model.Computer.CREATE`                                            | hudson.model.Computer.Create                                        | Permission[class hudson.model.Computer,Create]                                        | Permission[class hudson.security.Permission,GenericCreate]    |
| **Agent**              | `true`  | `hudson.model.Computer.DISCONNECT`                                        | hudson.model.Computer.Disconnect                                    | Permission[class hudson.model.Computer,Disconnect]                                    | Permission[class hudson.model.Hudson,Administer]              |
| **Agent**              | `true`  | `hudson.model.Computer.CONNECT`                                           | hudson.model.Computer.Connect                                       | Permission[class hudson.model.Computer,Connect]                                       | Permission[class hudson.model.Computer,Disconnect]            |
| **Agent**              | `true`  | `hudson.model.Computer.BUILD`                                             | hudson.model.Computer.Build                                         | Permission[class hudson.model.Computer,Build]                                         | Permission[class hudson.security.Permission,GenericWrite]     |
| **Job**                | `true`  | `hudson.model.Item.CREATE`                                                | hudson.model.Item.Create                                            | Permission[interface hudson.model.Item,Create]                                        | Permission[class hudson.security.Permission,GenericCreate]    |
| **Job**                | `true`  | `hudson.model.Item.DELETE`                                                | hudson.model.Item.Delete                                            | Permission[interface hudson.model.Item,Delete]                                        | Permission[class hudson.security.Permission,GenericDelete]    |
| **Job**                | `true`  | `hudson.model.Item.CONFIGURE`                                             | hudson.model.Item.Configure                                         | Permission[interface hudson.model.Item,Configure]                                     | Permission[class hudson.security.Permission,GenericConfigure] |
| **Job**                | `true`  | `hudson.model.Item.READ`                                                  | hudson.model.Item.Read                                              | Permission[interface hudson.model.Item,Read]                                          | Permission[class hudson.security.Permission,GenericRead]      |
| **Job**                | `true`  | `hudson.model.Item.DISCOVER`                                              | hudson.model.Item.Discover                                          | Permission[interface hudson.model.Item,Discover]                                      | Permission[interface hudson.model.Item,Read]                  |
| **Job**                | `false` | `hudson.model.Item.EXTENDED_READ`                                         | hudson.model.Item.ExtendedRead                                      | Permission[interface hudson.model.Item,ExtendedRead]                                  | Permission[interface hudson.model.Item,Configure]             |
| **Job**                | `true`  | `hudson.model.Item.BUILD`                                                 | hudson.model.Item.Build                                             | Permission[interface hudson.model.Item,Build]                                         | Permission[class hudson.security.Permission,GenericUpdate]    |
| **Job**                | `true`  | `hudson.model.Item.WORKSPACE`                                             | hudson.model.Item.Workspace                                         | Permission[interface hudson.model.Item,Workspace]                                     | Permission[class hudson.security.Permission,GenericRead]      |
| **Job**                | `false` | `hudson.model.Item.WIPEOUT`                                               | hudson.model.Item.WipeOut                                           | Permission[interface hudson.model.Item,WipeOut]                                       | null                                                          |
| **Job**                | `true`  | `hudson.model.Item.CANCEL`                                                | hudson.model.Item.Cancel                                            | Permission[interface hudson.model.Item,Cancel]                                        | Permission[class hudson.security.Permission,GenericUpdate]    |
| **Job**                | `true`  | `com.cloudbees.hudson.plugins.folder.relocate.RelocationAction.RELOCATE`  | hudson.model.Item.Move                                              | Permission[interface hudson.model.Item,Move]                                          | Permission[class hudson.security.Permission,GenericCreate]    |
| **Run**                | `true`  | `hudson.model.Run.DELETE`                                                 | hudson.model.Run.Delete                                             | Permission[class hudson.model.Run,Delete]                                             | Permission[class hudson.security.Permission,GenericDelete]    |
| **Run**                | `true`  | `hudson.model.Run.UPDATE`                                                 | hudson.model.Run.Update                                             | Permission[class hudson.model.Run,Update]                                             | Permission[class hudson.security.Permission,GenericUpdate]    |
| **Run**                | `false` | `hudson.model.Run.ARTIFACTS`                                              | hudson.model.Run.Artifacts                                          | Permission[class hudson.model.Run,Artifacts]                                          | null                                                          |
| **Run**                | `true`  | `org.jenkinsci.plugins.workflow.cps.replay.ReplayAction.REPLAY`           | hudson.model.Run.Replay                                             | Permission[class hudson.model.Run,Replay]                                             | Permission[interface hudson.model.Item,Configure]             |
| **View**               | `true`  | `hudson.model.View.CREATE`                                                | hudson.model.View.Create                                            | Permission[class hudson.model.View,Create]                                            | Permission[class hudson.security.Permission,GenericCreate]    |
| **View**               | `true`  | `hudson.model.View.DELETE`                                                | hudson.model.View.Delete                                            | Permission[class hudson.model.View,Delete]                                            | Permission[class hudson.security.Permission,GenericDelete]    |
| **View**               | `true`  | `hudson.model.View.CONFIGURE`                                             | hudson.model.View.Configure                                         | Permission[class hudson.model.View,Configure]                                         | Permission[class hudson.security.Permission,GenericConfigure] |
| **View**               | `true`  | `hudson.model.View.READ`                                                  | hudson.model.View.Read                                              | Permission[class hudson.model.View,Read]                                              | Permission[class hudson.security.Permission,GenericRead]      |
| **Job Config History** | `true`  | `hudson.plugins.jobConfigHistory.JobConfigHistory.DELETEENTRY_PERMISSION` | hudson.plugins.jobConfigHistory.JobConfigHistory.DeleteEntry        | Permission[class hudson.plugins.jobConfigHistory.JobConfigHistory,DeleteEntry]        | Permission[class hudson.model.Hudson,Administer]              |
| **SCM**                | `true`  | `hudson.scm.SCM.TAG`                                                      | hudson.scm.SCM.Tag                                                  | Permission[class hudson.scm.SCM,Tag]                                                  | Permission[class hudson.security.Permission,GenericCreate]    |
| **Overall**            | `true`  | `hudson.security.Permission.HUDSON_ADMINISTER`                            | hudson.model.Hudson.Administer                                      | Permission[class hudson.model.Hudson,Administer]                                      | null                                                          |
| **N/A**                | `true`  | `hudson.security.Permission.FULL_CONTROL`                                 | hudson.security.Permission.FullControl                              | Permission[class hudson.security.Permission,FullControl]                              | Permission[class hudson.model.Hudson,Administer]              |
| **N/A**                | `true`  | `hudson.security.Permission.READ`                                         | hudson.security.Permission.GenericRead                              | Permission[class hudson.security.Permission,GenericRead]                              | Permission[class hudson.model.Hudson,Administer]              |
| **N/A**                | `true`  | `hudson.security.Permission.WRITE`                                        | hudson.security.Permission.GenericWrite                             | Permission[class hudson.security.Permission,GenericWrite]                             | Permission[class hudson.model.Hudson,Administer]              |
| **N/A**                | `true`  | `hudson.security.Permission.CREATE`                                       | hudson.security.Permission.GenericCreate                            | Permission[class hudson.security.Permission,GenericCreate]                            | Permission[class hudson.security.Permission,GenericWrite]     |
| **N/A**                | `true`  | `hudson.security.Permission.UPDATE`                                       | hudson.security.Permission.GenericUpdate                            | Permission[class hudson.security.Permission,GenericUpdate]                            | Permission[class hudson.security.Permission,GenericWrite]     |
| **N/A**                | `true`  | `hudson.security.Permission.DELETE`                                       | hudson.security.Permission.GenericDelete                            | Permission[class hudson.security.Permission,GenericDelete]                            | Permission[class hudson.security.Permission,GenericWrite]     |
| **N/A**                | `true`  | `hudson.security.Permission.CONFIGURE`                                    | hudson.security.Permission.GenericConfigure                         | Permission[class hudson.security.Permission,GenericConfigure]                         | Permission[class hudson.security.Permission,GenericUpdate]    |

:::details 上の表生成に用いた Groovy スクリプト

```groovy
def permissions = [
  jenkins.model.Jenkins.ADMINISTER,
  jenkins.model.Jenkins.MANAGE,
  jenkins.model.Jenkins.SYSTEM_READ,
  jenkins.model.Jenkins.READ,
  jenkins.model.Jenkins.RUN_SCRIPTS,
  com.cloudbees.plugins.credentials.CredentialsProvider.USE_OWN,
  com.cloudbees.plugins.credentials.CredentialsProvider.USE_ITEM,
  com.cloudbees.plugins.credentials.CredentialsProvider.CREATE,
  com.cloudbees.plugins.credentials.CredentialsProvider.UPDATE,
  com.cloudbees.plugins.credentials.CredentialsProvider.VIEW,
  com.cloudbees.plugins.credentials.CredentialsProvider.DELETE,
  com.cloudbees.plugins.credentials.CredentialsProvider.MANAGE_DOMAINS,
  hudson.model.Computer.CONFIGURE,
  hudson.model.Computer.EXTENDED_READ,
  hudson.model.Computer.DELETE,
  hudson.model.Computer.CREATE,
  hudson.model.Computer.DISCONNECT,
  hudson.model.Computer.CONNECT,
  hudson.model.Computer.BUILD,
  hudson.model.Item.CREATE,
  hudson.model.Item.DELETE,
  hudson.model.Item.CONFIGURE,
  hudson.model.Item.READ,
  hudson.model.Item.DISCOVER,
  hudson.model.Item.EXTENDED_READ,
  hudson.model.Item.BUILD,
  hudson.model.Item.WORKSPACE,
  hudson.model.Item.WIPEOUT,
  hudson.model.Item.CANCEL,
  com.cloudbees.hudson.plugins.folder.relocate.RelocationAction.RELOCATE,
  hudson.model.Run.DELETE,
  hudson.model.Run.UPDATE,
  hudson.model.Run.ARTIFACTS,
  org.jenkinsci.plugins.workflow.cps.replay.ReplayAction.REPLAY,
  hudson.model.View.CREATE,
  hudson.model.View.DELETE,
  hudson.model.View.CONFIGURE,
  hudson.model.View.READ,
  hudson.plugins.jobConfigHistory.JobConfigHistory.DELETEENTRY_PERMISSION,
  hudson.scm.SCM.TAG,
  hudson.security.Permission.HUDSON_ADMINISTER,
  hudson.security.Permission.FULL_CONTROL,
  hudson.security.Permission.READ,
  hudson.security.Permission.WRITE,
  hudson.security.Permission.CREATE,
  hudson.security.Permission.UPDATE,
  hudson.security.Permission.DELETE,
  hudson.security.Permission.CONFIGURE,
]

for (p in permissions) {
  printf("|**%s**|`%b`||%s|%s|%s|\n", p.group.id, p.enabled, p.id, p.toString(), p.impliedBy)
}
```

:::

## 🤔Q-06. Permission クラス定数の一覧取得方法は?

以下の部分で、各 `Permission` は、`hudson.security.Permission.ALL` に登録されているので、
これを呼べば一覧で取得できます。

https://github.com/jenkinsci/jenkins/blob/master/core/src/main/java/hudson/security/Permission.java#L140-L156

が、本文中でも呼んでいる以下の方だと順番が纏まっているのでお薦めです。

```groovy
def groups = hudson.security.PermissionGroup.getAll()
groups.stream().map{it.getPermissions()}.collect().flatten()
```

## 🤔Q-07. `role-strategy` では、どのように権限の一覧を取得している?

ここでは、タイプに応じて種類からグループを絞っています。
https://github.com/jenkinsci/role-strategy-plugin/blob/master/src/main/java/com/michelin/cio/hudson/plugins/rolestrategy/RoleBasedAuthorizationStrategy.java#L969-L1005

## 🤔Q-08. スクリプトで初回追加する場合の注意は?

グローバルセキュリティと違って、自動で `admin` を作ってくれないので、ログインできなくなる場合に注意しましょう。

## 🤔Q-09. 内部の実装は?

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

## 🤔Q-10. Groovy Hook Script の実行順序は?

辞書順らしいです[^groovy-hook-scripts]。
[^groovy-hook-scripts]: [Groovy Hook Scripts](https://www.jenkins.io/doc/book/managing/groovy-hook-scripts/)

## 🤔Q-11. `jenkins.model.Jenkins.RUN_SCRIPTS` があれば管理者権限無しでスクリプト実行できる?

残念ながらできません。
古いドキュメント[^jenkins-matrix-based-security]を見る限りだと以前はできたようですが、権限昇格される虞から、無理になったようです[^jep-223]。
少し見た限り、 `jenkins.model.Jenkins.ADMINISTER` になってそうですね。

https://github.com/jenkinsci/jenkins/blob/master/core/src/main/java/jenkins/model/Jenkins.java#L5645-L5647

[^jenkins-matrix-based-security]: [Jenkins : Matrix-based security](https://wiki.jenkins.io/JENKINS/Matrix-based-security.html)
[^jep-223]: https://github.com/jenkinsci/jep/blob/master/jep/223/README.adoc

## 🤔Q-12. 権限の関係は?

memaid.js でクラス図を書きました。

![Permission クラス定数の関係](/images/jenkins-user-permission/permissions.png)
_Permission クラス定数の関係_

各 `PermissionGroup` 毎のクラス図は以下を参考にしてください。

- [hudson.model.Hudson](https://github.com/miya789/Zenn/blob/main/images/jenkins-user-permission/hudson.model.Hudson.svg)

  - :::details mermaid.js のソースコード

    ```
    classDiagram
    direction BT

    class Permission HUDSON_ADMINISTER
    class Permission READ
    class ADMINISTER
    class MANAGE {
      group:        PERMISSIONS;
      name:         "Manage";
      description:  Messages._Jenkins_Manage_Description();
      impliedBy:    ADMINISTER;
      boolean:      SystemProperties.getBoolean("jenkins.security.ManagePermission");
      scope:        PermissionScope.JENKINS;
    }
    class SYSTEM_READ {
      group:        PERMISSIONS;
      name:         "SystemRead";
      description:  Messages._Jenkins_SystemRead_Description();
      impliedBy:    ADMINISTER;
      boolean:      SystemProperties.getBoolean("jenkins.security.SystemReadPermission");
      scope:        PermissionScope.JENKINS;
    }
    class READ {
      group:        PERMISSIONS;
      name:         "Read";
      description:  Messages._Hudson_ReadPermission_Description();
      impliedBy:    Permission.READ;
      scope:        PermissionScope.JENKINS;
    }
    class RUN_SCRIPTS {
      group:        PERMISSIONS;
      name:         "RunScripts";
      description:  Messages._Hudson_RunScriptsPermission_Description();
      impliedBy:    ADMINISTER;
      scope:        PermissionScope.JENKINS;
    }

    ADMINISTER --> Permission HUDSON_ADMINISTER
    MANAGE --> ADMINISTER
    SYSTEM_READ --> ADMINISTER
    READ --> Permission READ
    RUN_SCRIPTS --> ADMINISTER
    ```

    :::

- [com.cloudbees.plugins.credentials.CredentialsProvider]()

  - :::details mermaid.js のソースコード

    ```

    ```

    :::

- [hudson.model.Computer](https://github.com/miya789/Zenn/blob/main/images/jenkins-user-permission/hudson.model.Computer.svg)

  - :::details mermaid.js のソースコード

    ```
    classDiagram
    direction BT

    class CONFIGURE {
      group:        PERMISSIONS;
      name:         "Configure";
      description:  Messages._Computer_ConfigurePermission_Description();
      impliedBy:    Permission.CONFIGURE;
      scope:        PermissionScope.COMPUTER;
    }
    class EXTENDED_READ {
      group:        PERMISSIONS;
      name:         "ExtendedRead";
      description:  Messages._Computer_ExtendedReadPermission_Description();
      impliedBy:    CONFIGURE;
      boolean:      SystemProperties.getBoolean("hudson.security.ExtendedReadPermission");
      scope:        PermissionScope.COMPUTER;
    }
    class DELETE {
      group:        PERMISSIONS;
      name:         "Delete";
      description:  Messages._Computer_DeletePermission_Description();
      impliedBy:    Permission.DELETE;
      scope:        PermissionScope.COMPUTER;
    }
    class CREATE {
      group:        PERMISSIONS;
      name:         "Create";
      description:  Messages._Computer_CretePermission_Description(),;
      impliedBy:    Permission.CREATE;
      scope:        PermissionScope.JENKINS;
    }
    class DISCONNECT {
      group:        PERMISSIONS;
      name:         "Disconnect";
      description:  Messages._Computer_DisconnectPermission_Description();
      impliedBy:    Jenkins.ADMINISTER;
      scope:        PermissionScope.COMPUTER;
    }
    class CONNECT {
      group:        PERMISSIONS;
      name:         "Connect";
      description:  Messages._Computer_ConnectPermission_Description();
      impliedBy:    DISCONNECT;
      scope:        PermissionScope.COMPUTER;
    }
    class BUILD {
      group:        PERMISSIONS;
      name:         "Build";
      description:  Messages._Computer_BuildPermission_Description();
      impliedBy:    Permission.WRITE;
      scope:        PermissionScope.COMPUTER;
    }

    CONFIGURE --> Permission CONFIGURE
    EXTENDED_READ --> CONFIGURE
    DELETE --> Permission DELETE
    CREATE --> Permission CREATE
    DISCONNECT --> Jenkins ADMINISTER
    CONNECT --> DISCONNECT
    BUILD --> Permission WRITE
    ```

    :::

- [hudson.model.Item](https://github.com/miya789/Zenn/blob/main/images/jenkins-user-permission/hudson.model.Item.svg)

  - :::details mermaid.js のソースコード

    ```
    classDiagram
    direction BT

    class CREATE {
      group:        PERMISSIONS;
      name:         "Create";
      description:  Messages._Item_CREATE_description();
      impliedBy:    Permission.CREATE;
      scope:        PermissionScope.ITEM_GROUP;
    }
    class DELETE {
      group:        PERMISSIONS;
      name:         "Delete";
      description:  Messages._Item_DELETE_description();
      impliedBy:    Permission.DELETE;
      scope:        PermissionScope.ITEM;
    }
    class CONFIGURE {
      group:        PERMISSIONS;
      name:         "Configure";
      description:  Messages._Item_CONFIGURE_description();
      impliedBy:    Permission.CONFIGURE;
      scope:        PermissionScope.ITEM;
    }
    class READ {
      group:        PERMISSIONS;
      name:         "Read";
      description:  Messages._Item_READ_description();
      impliedBy:    Permission.READ;
      scope:        PermissionScope.ITEM;
    }
    class DISCOVER {
      group:        PERMISSIONS;
      name:         "Discover";
      description:  Messages._AbstractProject_DiscoverPermission_Description();
      impliedBy:    READ;
      scope:        PermissionScope.ITEM;
    }
    class EXTENDED_READ {
      group:        PERMISSIONS;
      name:         "ExtendedRead";
      description:  Messages._AbstractProject_ExtendedReadPermission_Description();
      impliedBy:    CONFIGURE;
      enable:       SystemProperties.getBoolean("hudson.security.ExtendedReadPermission");
      scope:        PermissionScope.ITEM;
    }
    class BUILD {
      group:        PERMISSIONS;
      name:         "Build";
      description:  Messages._AbstractProject_BuildPermission_Description();
      impliedBy:    Permission.UPDATE;
      scope:        PermissionScope.ITEM;
    }
    class WORKSPACE {
      group:        PERMISSIONS;
      name:         "Workspace";
      description:  Messages._AbstractProject_WorkspacePermission_Description();
      impliedBy:    Permission.READ;
      scope:        PermissionScope.ITEM;
    }
    class WIPEOUT {
      group:        PERMISSIONS;
      name:         "WipeOut";
      description:  Messages._AbstractProject_WipeOutPermission_Description();
      impliedBy:    null;
      enable:       Functions.isWipeOutPermissionEnabled();
      scope:        PermissionScope.ITEM;
    }
    class CANCEL {
      group:        PERMISSIONS;
      name:         "Cancel";
      description:  Messages._AbstractProject_CancelPermission_Description();
      impliedBy:    Permission.UPDATE;
      scope:        PermissionScope.ITEM;
    }

    CREATE --> Permission CREATE
    DELETE --> Permission DELETE
    CONFIGURE --> Permission CONFIGURE
    READ --> Permission READ
    DISCOVER --> READ
    EXTENDED_READ --> CONFIGURE
    BUILD --> Permission UPDATE
    WORKSPACE --> Permission READ
    CANCEL --> Permission UPDATE
    ```

    :::

- [hudson.model.Run](https://github.com/miya789/Zenn/blob/main/images/jenkins-user-permission/hudson.model.Run.svg)

  - :::details mermaid.js のソースコード

    ```

    ```

    :::

- [hudson.model.View](https://github.com/miya789/Zenn/blob/main/images/jenkins-user-permission/hudson.model.View.svg)

  - :::details mermaid.js のソースコード

    ```
    classDiagram
    direction BT

    class CREATE {
      group:        PERMISSIONS;
      name:         "Create";
      description:  Messages._View_CreatePermission_Description();
      impliedBy:    Permission.CREATE;
      scope:        PermissionScope.ITEM_GROUP;
    }
    class DELETE {
      group:        PERMISSIONS;
      name:         "Delete";
      description:  Messages._View_DeletePermission_Description();
      impliedBy:    Permission.DELETE;
      scope:        PermissionScope.ITEM_GROUP;
    }
    class CONFIGURE {
      group:        PERMISSIONS;
      name:         "Configure";
      description:  Messages._View_ConfigurePermission_Description();
      impliedBy:    Permission.CONFIGURE;
      scope:        PermissionScope.ITEM_GROUP;
    }
    class READ {
      group:        PERMISSIONS;
      name:         "Read";
      description:  Messages._View_ReadPermission_Description();
      impliedBy:    Permission.READ;
      scope:        PermissionScope.ITEM_GROUP;
    }

    CREATE --> Permission CREATE
    DELETE --> Permission DELETE
    CONFIGURE --> Permission CONFIGURE
    READ --> Permission READ
    ```

    :::

- hudson.plugins.jobConfigHistory.JobConfigHistory: 一つしか無いので省略

- hudson.scm.SCM: 一つしか無いので省略

- [hudson.security.Permission](https://github.com/miya789/Zenn/blob/main/images/jenkins-user-permission/hudson.security.Permission.svg)

  - :::details mermaid.js のソースコード

    ```
    classDiagram
    direction BT

    class FULL_CONTROL {
      HUDSON_ADMINISTER e;
      FULL_CONTROL(this.e);

      group:        GROUP;
      name:         "FullControl";
      description:  null;
      impliedBy:    HUDSON_ADMINISTER;
    }
    class READ {
      group:        GROUP;
      name:         "GenericRead";
      description:  null;
      impliedBy:    HUDSON_ADMINISTER;
    }
    class WRITE {
      group:        GROUP;
      name:         "GenericWrite";
      description:  null;
      impliedBy:    HUDSON_ADMINISTER;
    }
    class CREATE {
      group:        GROUP;
      name:         "GenericCreate";
      description:  null;
      impliedBy:    WRITE;
    }
    class UPDATE {
      group:        GROUP;
      name:         "GenericUpdate";
      description:  null;
      impliedBy:    WRITE;
    }
    class DELETE {
      group:        GROUP;
      name:         "GenericDelete";
      description:  null;
      impliedBy:    WRITE;
    }
    class CONFIGURE {
      group:        GROUP;
      name:         "GenericConfigure";
      description:  null;
      impliedBy:    UPDATE;
    }
    class HUDSON_ADMINISTER {
      group:        HUDSON_PERMISSIONS;
      name:         "Administer";
      description:  hudson.model.Messages._Hudson_AdministerPermission_Description();
      impliedBy:    null;
    }

    FULL_CONTROL --> HUDSON_ADMINISTER
    READ --> HUDSON_ADMINISTER
    WRITE --> HUDSON_ADMINISTER
    CREATE --> WRITE
    UPDATE --> WRITE
    DELETE --> WRITE
    CONFIGURE --> UPDATE
    ```

    :::

## 🤔Q-13. やり方は妥当?

以下が参考になります。

https://github.com/jenkinsci/matrix-auth-plugin/blob/master/src/test/java/hudson/security/ProjectMatrixAuthorizationStrategyTest.java#L31-L64

初めに Jenkins ユーザーを作って、realm に追加してから、matrix 式の権限を付加してます。

## 🤔Q-14. Groovy スクリプトの内容をシステムログとして残すには?

以下で可能です。

```groovy
// 4. ログにも残せるので推奨
LOGGER = java.util.logging.Logger.getLogger(hudson.security.GlobalMatrixAuthorizationStrategy.class.getName());
LOGGER.log(java.util.logging.Level.INFO, "Update \"{0}\" by adding \"{1}\"", strategy, user_name);
```

## 🤔Q-15. 公式リポジトリのビルド方法は?

以下で開発コンテナとして可能

```json:/workspaces/jenkins/.devcontainer/devcontainer.json
// For format details, see https://aka.ms/vscode-remote/devcontainer.json or this file's README at:
// https://github.com/microsoft/vscode-dev-containers/tree/v0.195.0/containers/java
{
	"name": "Java",
	"build": {
		"dockerfile": "Dockerfile",
		"args": {
			// Update the VARIANT arg to pick a Java version: 8, 11, 17
			// Append -bullseye or -buster to pin to an OS version.
			// Use the -bullseye variants on local arm64/Apple Silicon.
			"VARIANT": "11-bullseye",
			// Options
			"INSTALL_MAVEN": "true",
			"MAVEN_VERSION": "3.8.6",
			"INSTALL_GRADLE": "false",
			"NODE_VERSION": "lts/*"
		}
	},
	// Configure tool-specific properties.
	"customizations": {
		// Configure properties specific to VS Code.
		"vscode": {
			// Set *default* container specific settings.json values on container create.
			"settings": {
				"maven.executable.path": "/usr/local/sdkman/candidates/maven/current/bin/mvn"
			},
			// Add the IDs of extensions you want installed when the container is created.
			"extensions": [
				"vscjava.vscode-java-pack"
			]
		}
	},
	// Use 'forwardPorts' to make a list of ports inside the container available locally.
	// "forwardPorts": [],
	// Use 'postCreateCommand' to run commands after the container is created.
	// "postCreateCommand": "java -version",
	// Uncomment to connect as a non-root user. See https://aka.ms/vscode-remote/containers/non-root.
	"remoteUser": "vscode"
}
```

```json:/workspaces/jenkins/.vscode/launch.json
{
  // IntelliSense を使用して利用可能な属性を学べます。
  // 既存の属性の説明をホバーして表示します。
  // 詳細情報は次を確認してください: https://go.microsoft.com/fwlink/?linkid=830387
  "version": "0.2.0",
  "configurations": [
    {
      "type": "java",
      "name": "Launch Current File",
      "request": "launch",
      "mainClass": "${file}"
    },
    {
      "type": "java",
      "name": "Launch CLI",
      "request": "launch",
      "mainClass": "hudson.cli.CLI",
      "projectName": "cli"
    },
    {
      "type": "java",
      "name": "Launch Main",
      "request": "launch",
      "mainClass": "hudson.Main",
      "projectName": "jenkins-core"
    },
    {
      "type": "java",
      "name": "Launch DCOMSandbox",
      "request": "launch",
      "mainClass": "hudson.os.DCOMSandbox",
      "projectName": "jenkins-core"
    },
    {
      "type": "java",
      "name": "Launch SUTester",
      "request": "launch",
      "mainClass": "hudson.os.SUTester",
      "projectName": "jenkins-core"
    },
    {
      "type": "java",
      "name": "Launch RunIdMigrator",
      "request": "launch",
      "mainClass": "jenkins.model.RunIdMigrator",
      "projectName": "jenkins-core"
    },
    {
      "type": "java",
      "name": "Launch DomainValidatorTest",
      "request": "launch",
      "mainClass": "jenkins.org.apache.commons.validator.routines.DomainValidatorTest",
      "projectName": "jenkins-core"
    },
    {
      "type": "java",
      "name": "Launch UrlValidatorTest",
      "request": "launch",
      "mainClass": "jenkins.org.apache.commons.validator.routines.UrlValidatorTest",
      "projectName": "jenkins-core"
    },
    // {
    //   "type": "java",
    //   "name": "Launch Main(1)",
    //   "request": "launch",
    //   "mainClass": "executable.Main",
    //   "projectName": "jenkins-war",
    // },
    {
      "type": "java",
      "name": "Launch Main(1)",
      "request": "launch",
      // "mainClass": "executable.Main",
      "projectName": "war",
      "args": "-Xmx1100m -classpath /usr/local/sdkman/candidates/maven/current/boot/plexus-classworlds-2.6.0.jar -Dclassworlds.conf=/usr/local/sdkman/candidates/maven/current/bin/m2.conf -Dmaven.home=/usr/local/sdkman/candidates/maven/current -Dlibrary.jansi.path=/usr/local/sdkman/candidates/maven/current/lib/jansi-native -Dmaven.multiModuleProjectDirectory=workspaces/jenkins org.codehaus.plexus.classworlds.launcher.Launcher -pl war"
    }
  ]
}
```

最新の tag に checkout する。

:::message
かなりメモリを食うので注意。
:::

自前でインストールするなら以下を参照

- https://www.linuxmania.jp/apt-install-java.html

- https://maven.apache.org/install.html
- https://maven3.kengo-toda.jp/module/option
- https://kazuhira-r.hatenablog.com/entry/20150407/1428419372

:::message
バージョンは以下で行いました。

```bash
$ javac -version
Picked up _JAVA_OPTIONS: -Dawt.useSystemAAFontSettings=on -Dswing.aatext=true
javac 11.0.16

$ java -version
Picked up _JAVA_OPTIONS: -Dawt.useSystemAAFontSettings=on -Dswing.aatext=true
openjdk version "11.0.16" 2022-07-19
OpenJDK Runtime Environment (build 11.0.16+8-post-Debian-1)
OpenJDK 64-Bit Server VM (build 11.0.16+8-post-Debian-1, mixed mode, sharing)

$ mvn -v
Picked up _JAVA_OPTIONS: -Dawt.useSystemAAFontSettings=on -Dswing.aatext=true
Apache Maven 3.8.6 (84538c9988a25aec085021c365c560670ad80f63)
Maven home: /usr/share/maven
Java version: 11.0.16, vendor: Debian, runtime: /usr/lib/jvm/java-11-openjdk-amd64
Default locale: en_US, platform encoding: UTF-8
OS name:        "linux", version: "5.16.0-kali7-amd64", arch: "amd64", family: "unix"

$ node -v
v18.7.0
```

:::

## 🤔Q-16. プラグインのデバッグは?

以下を参考に
https://jenkinsci.github.io/maven-hpi-plugin/run-mojo.html

## 🤔Q-17. `PermissionScope` って?

`PermissionGroup` との違いは把握できてない状況ですが、少なくとも以下が確認されました。

```java:PermissionScope
public static final PermissionScope JENKINS     = new PermissionScope(Jenkins.class);
public static final PermissionScope ITEM_GROUP  = new PermissionScope(ItemGroup.class,JENKINS);
public static final PermissionScope ITEM        = new PermissionScope(Item.class,ITEM_GROUP);
public static final PermissionScope RUN         = new PermissionScope(Run.class,ITEM);
public static final PermissionScope COMPUTER    = new PermissionScope(Computer.class,JENKINS);
```

## 🗒️C-1. 取得系コードのサンプル

```groovy:
hudson.security.SecurityRealm.all()
// [hudson.security.HudsonPrivateSecurityRealm$DescriptorImpl@5454d763, hudson.security.LegacySecurityRealm$DescriptorImpl@687047d7, hudson.security.SecurityRealm$None$DescriptorImpl@1807eee9]

def instance = Jenkins.getInstance()
def strategy = instance.getAuthorizationStrategy()
println instance.securityRealm.getAllUsers()
println strategy.getGrantedPermissionEntries()
```

# 補足

「打ち上げ花火、下から見るか？横から見るか？」自体はドラマでやっていたもののアニメだったらしいですね。

どこかの口コミにもありましたが、ドラマ版と年齢設定を変えておきながら登場人物の台詞が幼稚な部分があり、自分は少々苦手でした。
ジェネリック新海誠を期待して観に行った節がありましたが、自然の風景と漠然とした主人公の抱く不安を搔き混ぜた上で主人公が心情を吐露する語りも無く、何を考えているのかあまり分からない感触がありましたね。

ただ個人的には、アニメだからこそできる列車の演出やラストの演出など、情景としては印象に残るもので好きでした。
新海誠作品に関して、印象を強調した景色を描くので印象派の絵に近いと大学の講義中に言っている先生もいましたが、アニメだからこそこうした景色の世界や果ては幻想的な世界が創造できると考えており、点に関しては良いものを見れたと思ってます。

自分の中では、花火が変な形に咲く世界に来てしまっていたのであれは幻。と思っていたのですが、どうなんでしょうね?
