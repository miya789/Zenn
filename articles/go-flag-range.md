---
title: ""
emoji: "🕌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

## コマンドライン引数で範囲外の値を指定された時

内容少ないですがご容赦ください

- flag の中身を覗いて最低限必要な箇所だけ参考にこんなものを書いた
  - `map` とかで高尚なことをしていたが面倒だったのでやってないです

if で比較する部分で直接数字と比較しているのはアレですが，
元は文字列が入っていたので，変換しました
他は殆どそのままです．

```go
// 元ネタ
return false, f.failf("invalid value %q for flag -%s: %v", value, name, err)

// 今回の
err := fmt.Errorf("invalid value %q for flag -%s: %v", strconv.Itoa(foo), "foo", "parse error")
```

```go
var (
  foo int
)

flag.IntVar(&foo, "foo", 0, "0: others \t(default)\n1: hoge\n2: fuga")
flag.Parse()
if foo < 0 || foo > 2 {
  err := fmt.Errorf("invalid value %q for flag -%s: %v", strconv.Itoa(foo), "foo", "parse error")
  fmt.Fprintln(flag.CommandLine.Output(), err)
  fmt.Fprintf(flag.NewFlagSet(os.Args[0], flag.ExitOnError).Output(), "Usage of %s:\n", os.Args[0])
  flag.PrintDefaults()
  return
}
```

