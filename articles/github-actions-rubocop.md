---
title: "GitHub Actions が RuboCop できないのはどう考えてもお前らが悪い！"
emoji: "🐶"
type: "tech"
topics: [githubactions, rubocop, reviewdog]
published: true
---

# TL;DR

- `reviewdog/action-rubocop` は差分に対しての Rubocop 結果を Pull Request 内でコメントしてくれるのでお薦め
- `on: push` でトリガーされた Actions は PR 内であっても認識しないので，これを動かす時は， `on: pull_request` 推奨 (それはそう)
- 要は，以下の様に公式準拠にすれば問題無いです

```yml:rubocop.yml
name: reviewdog
on: [pull_request]
jobs:
  rubocop:
    name: runner / rubocop
    runs-on: ubuntu-latest
    steps:
      - name: Check out code
        uses: actions/checkout@v1
      - uses: ruby/setup-ruby@v1
        with:
          ruby-version: 3.0.0
      - name: rubocop
        uses: reviewdog/action-rubocop@v1
        with:
          rubocop_version: gemfile
          rubocop_extensions: rubocop-rails:gemfile rubocop-rspec:gemfile
          github_token: ${{ secrets.github_token }}
          reporter: github-pr-review # Default is github-pr-check
```

# 目的

他の記事でも触れられている通り，冒頭の様に `reviewdog/action-rubocop` の公式ドキュメントに準拠する事に尽きます
しかし実際に利用するに当たり様々なエラーに直面しましたので，\
簡単に GitHub Actions で使える RuboCop を紹介した後に， `reviewdog/action-rubocop` で生じるエラーに関して簡単に述べます

# GitHub Actions で動かす RuboCop

少し調べると `bundle exec rubocop` を走らせてる記事が多く散見されますが， GitHub との連携も優れている方法を探してみます．
そこで取り敢えず以下の 2 つを見つけましたが，
結論としては [`reviewdog/action-rubocop`](https://github.com/reviewdog/action-rubocop) が GitHub との親和性も高く便利に思われます

| ライブラリ                                                                                          | 差分に対しての Rubocop 適用 | PR 内でコメント        |
| --------------------------------------------------------------------------------------------------- | --------------------------- | ---------------------- |
| [1. `reviewdog/action-rubocop`](#1.-reviewdog%2Faction-rubocop)                                     | ✅                          | ✅                     |
| [2. `andrewmcodes-archive/rubocop-linter-action`](#2.-andrewmcodes-archive%2Frubocop-linter-action) | ❌                          | ❌(アノテーションのみ) |

順に見てみます

https://github.com/reviewdog/action-rubocop
https://github.com/andrewmcodes-archive/rubocop-linter-action

## 1. `reviewdog/action-rubocop`

`reviewdog/action-rubocop` (以下， `action-rubocop`) は [reviewdog/reviewdog](https://github.com/reviewdog/reviewdog) の系列のものらしいです
他にも GitHub の PR にコメントしてくるツールを色々提供してくれているようで，使い勝手も良い雰囲気でしたすが，設定項目は本家の [reviewdog/reviewdog](https://github.com/reviewdog/reviewdog) を見た方が良さそうでした
やはり犬は可愛いですね (U^ω^)

[![action-rubocopを使ったPRの表示例](https://storage.googleapis.com/zenn-user-upload/1o37lz2lic5u28eeuhvk8rhp58jb)_`action-rubocop` を使った PR の表示例． github-actions という Bot が，Rubocop 違反箇所を勝手にコメントで指摘してくれます．_](https://github.com/miya789/RubocopReviewTest/pull/1)

## 2. `andrewmcodes-archive/rubocop-linter-action`

一方の `andrewmcodes-archive/rubocop-linter-action` (以下， `rubocop-linter-action` )は，
公式リポジトリがアーカイブされていて少し微妙でした

まずこれはコメントでは指摘してくれません
[![rubocop-linter-actionを使ったPRの表示例](https://storage.googleapis.com/zenn-user-upload/6k87lepan1tw4l3azw3d9ceupift)_`rubocop-linter-action` を使った PR の表示例． Rubocop で違反箇所があると GitHub Actions が落ちます．_](https://github.com/miya789/RubocopReviewTest/pull/5)

そして PR の `Files changed` では変更したファイルに限定してアノテーションを表示してくれますが，
**変更したファイルの変更箇所とは異なる部分であっても RuboCop の意に沿わない部分があると全て晒し上げられます**

:::message
公式ドキュメントで以下の様に無理と言われており，メンテ終了しているので恐らく無理です

> **3. The modified flag is not working!**
>
> If you specify the following in your config file:
>
> ```yaml
> check_scope: "modified"
> ```
>
> Please note that this will not work on commits to master. If you have an idea on how to make this work, please open an issue or PR!

:::

因みに，一応試しに `pull_request` をトリガーにしてみましたが，
やはりコメントでの指摘はしてくれないようで，それどころか今度はアノテーションが消失しました……
GitHub Actions が落ちますし，コメントやアノテーションで見るならやはり rubocop-linter-action は微妙そうです [^hai3.net/blog]
(あまり使わない線で考えていたので深追いはしません)

## まとめ

という訳で `action-rubocop` に焦点を絞って紹介していきます

:::message
少し脱線しますが， `.rubocop_todo.yml` の修正の手伝いをしてくれる
RuboCop Challenger という gem を GitHub Actions で動かす方法もあるらしいです[^rubocop-challenger]
これで， RuboCop のルールを容易にカスタマイズできますね
:::

# `action-rubocop` のエラー例

エラーメッセージのみからは類推が難しい場合が多かったので，一応纏めておきます

以下に挙げるテスト用のリポジトリで色々試してみましたので，もしよろしければこちらから確認をお願いいたします
https://github.com/miya789/RubocopReviewTest

:::message
このリポジトリは下記差分の commit から始まり，
これを RuboCop に**良い具合に指摘してもらう**ことを目的としています

```diff ruby:Gemfile
@@ -39,6 +39,7 @@ group :development, :test do
  # Adds support for Capybara system testing and selenium driver
  gem 'capybara', '~> 2.13'
  gem 'selenium-webdriver'
+ gem 'rubocop', '~> 1.12', require: false
end
```

:::

## `this is not PullRequest build`

```bash
Running rubocop with reviewdog 🐶 ...
reviewdog: this is not PullRequest build.
Broken pipe @ io_write - <STDOUT>
/opt/hostedtoolcache/Ruby/2.7.0/x64/lib/ruby/gems/2.7.0/gems/rubocop-1.14.0/lib/rubocop/formatter/clang_style_formatter.rb:18:in `write'
...
```

因みにこの場合はエラーは吐きません
これは `on: [push, pull_request]` もしくは `on: pull_request` で直りました
`on: push` でトリガーされた Actions は PR 内であっても認識しないので (それはそう)
`rubocop-linter-action` (使わない方) では `push` だったので混同しますね……

```diff yaml:.github/workflows/rubocop.yml
name: reviewdog
- on: [push]
+ on: [pull_request]
jobs:
...
```

:::message
動作例: https://github.com/miya789/RubocopReviewTest/runs/2523903323?check_suite_focus=true
:::

## `reviewdog: failed to run 'git rev-parse --show-prefix': exit status 128.`

```bash
Running rubocop with reviewdog 🐶 ...
reviewdog: PullRequest needs 'git' command: failed to run 'git rev-parse --show-prefix': exit status 128
Error: Process completed with exit code 1.
```

別の lint の Issue でも同じ内容が質問されていましたが[^action-tflint/issues/21]， `actions/checkout@v2` を忘れると起きます

```diff yaml:.github/workflows/rubocop.yml
...
jobs:
  rubocop:
    name: runner / rubocop
    runs-on: ubuntu-latest
    steps:
+    - name: Check out code # nameはお好みで無くても良い
+      uses: actions/checkout@v1
...
```

:::message
動作例: https://github.com/miya789/RubocopReviewTest/pull/3/checks?check_run_id=2523887864
:::

## `You don't have write permissions for the /var/lib/gems/2.7.0 directory.`

```bash
Installing rubocop with extensions ... https://github.com/rubocop/rubocop
ERROR:  While executing gem ... (Gem::FilePermissionError)
  You don't have write permissions for the /var/lib/gems/2.7.0 directory.
Error: Process completed with exit code 1.
```

詳細は割愛しますが，
setup が無いと正常に GitHub Actions が ruby の gem を参照できないようです[^rubocop-setup]
これも， `rubocop-linter-action` (使わない方) では不要だったので少しややこしいですね

```diff yaml:.github/workflows/rubocop.yml
...
jobs:
  rubocop:
    name: runner / rubocop
    runs-on: ubuntu-latest
    steps:
      - name: Check out code
        uses: actions/checkout@v1
+     - uses: ruby/setup-ruby@v1
+       with:
+         ruby-version: 2.7.0 # バージョンは自分のプロジェクトに合わせて
...
```

:::message
動作例: https://github.com/miya789/RubocopReviewTest/pull/2/checks?check_run_id=2523873135
:::

# 補足

[私がモテないのはどう考えてもお前らが悪い!](https://www.ganganonline.com/contents/watashiga/)(以下，わたモテ)に関して，
私は数年前にアニメで見た程度のにわかですが，
「気持ち悪さ」の描写が的確で共感性羞恥心を抉られる話で良かったです
最近は百合展開になったり，ロシアで人気になったり[^watamote-russia]で話題に事欠かず再燃の機運が高まっていますね
この記事を見てしまった方は，この機会に もこっち の勇姿を見届けてはいかがでしょうか?

[^rubocop-challenger]: [RuboCop Challenger を GitHub Actions で動かす](https://zenn.dev/yamat47/articles/219e14ebcf31a1d13ff4)
[^rubocop-setup]: [reviewdog/action-rubocop 実行時に起きた Gem::FilePermissionError の対処](https://zenn.dev/m_yamashii/articles/bf6a52a71f887d)
[^hai3.net/blog]: [GitHub Actions で rubocop の結果を PR 時に表示する](https://hai3.net/blog/github-actions-rubocop-linter-action/)
[^action-tflint/issues/21]: [reviewdog: failed to run 'git rev-parse --show-prefix': exit status 128 · Issue #21 · reviewdog/action-tflint](https://github.com/reviewdog/action-tflint/issues/21)
[^watamote-russia]:
    [Watamotist さんは Twitter を使っています 「#わたモテ https://t.co/Fvm4jOIM3f
    チャンネル登録 785 万人のロシア系 YouTuber 系ラッパー #Morgenshtern の絶対にスマッシュヒットする新曲の MV にもこっち https://t.co/bpLNTxtr5I」 / Twitter](https://twitter.com/watamotist/status/1307668661939757056)
