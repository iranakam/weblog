---
title: Cobraを利用してのCLIツールの作成
date: 2019-06-21T00:00:00+09:00
tags: ["golang", "cobra"]
draft: false

---

Goで簡単なCLIツールを作る機会があり、[cobra](https://github.com/spf13/cobra)を利用したのでメモを残します。

# インストール

まずはcobraをインストールします。

```bash
$ go get -u github.com/spf13/cobra/cobra
```

# ルートコマンドの生成

cobraには`cobra`コマンドが用意されていて、アプリケーションに必要なファイルを生成してくれます。  

`cobra`には`init`というサブコマンドがあり、新しいアプリケーションの雛形を生成してくれます。  
`init`は`--pkg-name`でパッケージ名、他に`--author`や`--license`、後は`--viper`で[viper](https://github.com/spf13/viper)の利用要否を指定できます。  

今回はhogeというコマンドを生成します。

```bash
$ cobra init --pkg-name=github.com/iranakam/hoge --author=iranakam --license=mit --viper=false
```

下記のディレクトリ及びファイルが生成されます。

```bash
$ tree
.
|-- LICENSE
|-- cmd
|   `-- root.go
`-- main.go

1 directory, 3 files
```

`root.go`に`rootCmd`が定義されます。  
なお、実際に生成されるコードは多くのコメントを含みますが、今回その部分は省略します。

```go
var rootCmd = &cobra.Command{
  Use:   "hoge",
  Short: "A brief description of your application",
  Long: `A longer description that spans multiple lines and likely contains
examples and usage of using your application. For example:

Cobra is a CLI library for Go that empowers applications.
This application is a tool to generate the needed files
to quickly create a Cobra application.`,
}
```

また、`rootCmd`を実行する関数も併せて定義されます。

```go
func Execute() {
  if err := rootCmd.Execute(); err != nil {
    fmt.Println(err)
      os.Exit(1)
  }
}
```

# サブコマンドの生成

cobraには`init`の他に`add`というサブコマンドがあり、指定のコマンドに対し新しいサブコマンドの雛形を生成してくれます。  
サブコマンドの親コマンドは`--parent`で指定し(デフォルトは`rootCmd`)、その他のフラグは`init`と同様です。

```bash
$ cobra add fuga
```

`fuga.go`が生成されます。

```bash
$ tree
.
|-- LICENSE
|-- cmd
|   |-- fuga.go
|   `-- root.go
`-- main.go

1 directory, 4 files
```

`fuga.go`に`fugaCmd`が定義されます。  

```go
var fugaCmd = &cobra.Command{
	Use:   "fuga",
	Short: "A brief description of your command",
	Long: `A longer description that spans multiple lines and likely contains examples
and usage of using your command. For example:

Cobra is a CLI library for Go that empowers applications.
This application is a tool to generate the needed files
to quickly create a Cobra application.`,
	Run: func(cmd *cobra.Command, args []string) {
		fmt.Println("fuga called")
	},
}
```

`fuga.go`に、指定した`--parent`(今回の場合は`rootCmd`)に対して追加されます。  

```go
func init() {
	rootCmd.AddCommand(fugaCmd)
}
```

# フラグの設定

コマンドにフラグを設定できます。  
例えば`--message`で文字列の値を受け取る場合は下記のように設定します。

```go
func init() {
  rootCmd.PersistentFlags().String("message", "hello", "message")
}
```

受け取った値は参照できます。

```go
message := cmd.Flag("message").Value.String()
```

また、そのまま変数に代入することもできます。

```go
var message string

func init() {
  rootCmd.PersistentFlags().StringVar(&message, "message", "hello", "message")
}
```

`rootCmd`にフラグを設定すればグローバルなフラグとして設定される一方、`fugaCmd`に設定すればローカルフラグとして扱われます。

```go
var foo string

func init() {
  rootCmd.AddCommand(fugaCmd)
  fugaCmd.PersistentFlags().StringVar(&foo, "foo", "", "A help for foo")
}
```

`--foo`は必須の場合は下記のように設定します。

```go
var foo string

func init() {
  rootCmd.AddCommand(fugaCmd)
  fugaCmd.PersistentFlags().StringVar(&foo, "foo", "", "A help for foo")
  fugaCmd.MarkPersistentFlagRequired("foo")
}
```

# 最後に

さくっとシンプルなCLIツール作るにはぴったりな気が、とはいっても他はurfaveしか使ったことがないのですが...  
urfaveとの比較でいうと、サブコマンド周りで痒いところに手が届く感がありました。
