---
title: go-每日一库之-cobra
date: 2023-03-09 09:18:20
tags:
---


## 概览

cobra 是一个用于创建命令行工具的库（框架），可以创建出类似git或者go一样的工具，比如我们平时熟悉的git clone/pull、 go get/install 等操作。kubernetes 工具中就使用了cobra。

<!--more-->

上述这些命令行工具的--help/-h既好看又好理解，开始时我也会好奇这些工具是如何实现多级子命令支持的？ cobra的目标即使提供快速构建出此类工具的能力（cobra是框架，具体的业务逻辑当然还是要自己写）。 例如可以为我们app或者系统工具的bin文件添加--config来启动服务、添加--version来查看版本号。相比起标准库里的flag包，cobra强大了很多。

官网列出了cobra的一大堆优点:

1. 易用的subcommand模式（即嵌套命令或子命令）
2. 强大的flags支持（参考标准库flagSet）
3. 支持global、local、、cascading级别的flag设置
4. 错误时的智能提示
5. 自动生成的、美观的help信息，并默认支持--help或-h打印的help信息
6. 提供命令自动补全功能（bash、 zsh、 fish、 powershell环境）
7. 提供命令man page自动生成功能
8. 有兄弟项目viper可以快捷的实现配置文件解析和flag绑定（暂不建议直接使用，viper包装了一些配置文件解析的细节，应当在熟悉cobra之后再单独了解）

## 本文的核心目的为： 通过手写一个示例来迅速入门cobra

先来简单看下cobra框架中主要概念:

```bash
kebectl get pod|service [podName|serviceName] -n <namespace>
```

以上述kubectl get为例，cobra将kubectl称为rootcmd（即根命令）， get称作rootcmd的subcmd， pod|service则是get的subcmd，podName、serviceName是pod/service的args， -n/--namespace称作flag。同时我们还观察到-n 这个flag其实写在任意-个cmd之后都会正常生效，这说明这是一个global flag，global flag对于rootcmd及所有子命令生效。
此外，当为cmd指定了一个与subcmd同名的args时，subcmd优先生效。

## 一、 使用cobra建议的项目目录结构

cobra文档告诉我们，使用cobra时最好遵循如下的项目目录设置（即： 把命令行定义相关的部分单独写在一个名为cmd的包内）

```
▾ appName/
  ▾ cmd/
      add.go
      your.go
      commands.go
      here.go
    main.go
```

使用cobra的项目，其程序入口main.go一般极为简洁（相应的业务逻辑都在cmd里调用或实现）:

```golang
package main

import (
	"{pathToYourApp}/cmd"
)

func main() {
	cmd.Execute()
}

```

## 使用cobra库

要手动写cobra代码，您需要创建一个裸main.go文件和一个rootCmd文件，你可以选择提供您认为合适的其他命令。


## 创建rootCmd文件

cobra不需要任何特殊的构造函数，只需创建您的命令。

理想情况下，您将它放在cmd/root.go中：

```golang
var rootCmd = &cobra.Command{
  Use:   "hugo",
  Short: "Hugo is a very fast static site generator",
  Long: `A Fast and Flexible Static Site Generator built with
                love by spf13 and friends in Go.
                Complete documentation is available at https://gohugo.io/documentation/`,
  Run: func(cmd *cobra.Command, args []string) {
    // Do Stuff Here
  },
}

func Execute() {
  if err := rootCmd.Execute(); err != nil {
    fmt.Fprintln(os.Stderr, err)
    os.Exit(1)
  }
}
```

您还将在 init() 函数中定义标志和处理配置。

例如cmd/root.go：

```golang
package cmd

import (
  "fmt"
  "os"

  "github.com/spf13/cobra"
  "github.com/spf13/viper"
)

var (
  // Used for flags.
  cfgFile     string
  userLicense string

  rootCmd = &cobra.Command{
    Use:   "cobra-cli",
    Short: "A generator for Cobra based Applications",
    Long: `Cobra is a CLI library for Go that empowers applications.
This application is a tool to generate the needed files
to quickly create a Cobra application.`,
  }
)

// Execute executes the root command.
func Execute() error {
  return rootCmd.Execute()
}

func init() {
  cobra.OnInitialize(initConfig)

  rootCmd.PersistentFlags().StringVar(&cfgFile, "config", "", "config file (default is $HOME/.cobra.yaml)")
  rootCmd.PersistentFlags().StringP("author", "a", "YOUR NAME", "author name for copyright attribution")
  rootCmd.PersistentFlags().StringVarP(&userLicense, "license", "l", "", "name of license for the project")
  rootCmd.PersistentFlags().Bool("viper", true, "use Viper for configuration")
  viper.BindPFlag("author", rootCmd.PersistentFlags().Lookup("author"))
  viper.BindPFlag("useViper", rootCmd.PersistentFlags().Lookup("viper"))
  viper.SetDefault("author", "NAME HERE <EMAIL ADDRESS>")
  viper.SetDefault("license", "apache")

  rootCmd.AddCommand(addCmd)
  rootCmd.AddCommand(initCmd)
}

func initConfig() {
  if cfgFile != "" {
    // Use config file from the flag.
    viper.SetConfigFile(cfgFile)
  } else {
    // Find home directory.
    home, err := os.UserHomeDir()
    cobra.CheckErr(err)

    // Search config in home directory with name ".cobra" (without extension).
    viper.AddConfigPath(home)
    viper.SetConfigType("yaml")
    viper.SetConfigName(".cobra")
  }

  viper.AutomaticEnv()

  if err := viper.ReadInConfig(); err == nil {
    fmt.Println("Using config file:", viper.ConfigFileUsed())
  }
}
```


## 创建你的 main.go

使用 root 命令，您需要让主函数执行它。为清楚起见，应在根目录上运行 Execute，尽管它可以在任何命令上调用。

在 Cobra 应用程序中，通常 main.go 文件非常简单。它有一个目的：调用 Cobra。


```golang
package main

import (
  "{pathToYourApp}/cmd"
)

func main() {
  cmd.Execute()
}
```


## 创建附加命令

可以定义其他命令，通常每个命令在 cmd/ 目录中都有自己的文件。

如果你想创建一个版本命令，你可以创建 cmd/version.go 并用以下内容填充它：

```golang
package cmd

import (
  "fmt"

  "github.com/spf13/cobra"
)

func init() {
  rootCmd.AddCommand(versionCmd)
}

var versionCmd = &cobra.Command{
  Use:   "version",
  Short: "Print the version number of Hugo",
  Long:  `All software has versions. This is Hugo's`,
  Run: func(cmd *cobra.Command, args []string) {
    fmt.Println("Hugo Static Site Generator v0.9 -- HEAD")
  },
}
```

## 返回和处理错误

如果您希望向命令的调用者返回错误，RunE可以使用。


```golang
package cmd

import (
  "fmt"

  "github.com/spf13/cobra"
)

func init() {
  rootCmd.AddCommand(tryCmd)
}

var tryCmd = &cobra.Command{
  Use:   "try",
  Short: "Try and possibly fail at something",
  RunE: func(cmd *cobra.Command, args []string) error {
    if err := someFunc(); err != nil {
  return err
    }
    return nil
  },
}
```

然后可以在执行函数调用时捕获错误。

## 使用标志

标志提供修饰符来控制操作命令的操作方式。


### 为命令分配标志

由于标志是在不同的位置定义和使用的，因此我们需要在外部定义一个具有正确范围的变量来分配要使用的标志。

```golang
var Verbose bool
var Source string
```

## 有两种不同的方法来分配标志。

### 持久标志

标志可以是“持久的”，这意味着该标志将可用于分配给它的命令以及该命令下的每个命令。对于全局标志，将标志分配为根上的持久标志。

```golang
rootCmd.PersistentFlags().BoolVarP(&Verbose, "verbose", "v", false, "verbose output")

```

### 本地标志

也可以在本地分配标志，这将仅适用于该特定命令。

```golang
localCmd.Flags().StringVarP(&Source, "source", "s", "", "Source directory to read from")

```

### 父命令的本地标志

默认情况下，Cobra 仅解析目标命令上的本地标志，而忽略父命令上的任何本地标志。通过启用Command.TraverseChildren，Cobra 将在执行目标命令之前解析每个命令的本地标志。

```golang
command := cobra.Command{
  Use: "print [OPTIONS] [COMMANDS]",
  TraverseChildren: true,
}
```

### 使用配置绑定标志

你也可以用viper绑定你的标志：


```golang
var author string

func init() {
  rootCmd.PersistentFlags().StringVar(&author, "author", "YOUR NAME", "Author name for copyright attribution")
  viper.BindPFlag("author", rootCmd.PersistentFlags().Lookup("author"))
}
```
在此示例中，持久标志author与viper绑定. 注意：当用户提供标志--author时，变量author不会设置为配置中的值。


### 必需的标志

默认情况下，标志是可选的。相反，如果您希望您的命令在未设置标志时报告错误，请将其标记为必需：

```golang
rootCmd.Flags().StringVarP(&Region, "region", "r", "", "AWS region (required)")
rootCmd.MarkFlagRequired("region")
```

或者，对于持久性标志：

```golang
rootCmd.PersistentFlags().StringVarP(&Region, "region", "r", "", "AWS region (required)")
rootCmd.MarkPersistentFlagRequired("region")
```

### 标志组

如果你有不同的标志必须一起提供（例如，如果他们提供标志，他们也--username必须提供标志）那么 Cobra 可以强制执行该要求：--password

```golang
rootCmd.Flags().StringVarP(&u, "username", "u", "", "Username (required if password is set)")
rootCmd.Flags().StringVarP(&pw, "password", "p", "", "Password (required if username is set)")
rootCmd.MarkFlagsRequiredTogether("username", "password")
```

如果不同的标志代表相互排斥的选项，例如将输出格式指定为其中之一--json或--yaml但绝不是两者，您还可以防止一起提供不同的标志：

```golang
rootCmd.Flags().BoolVar(&ofJson, "json", false, "Output in JSON")
rootCmd.Flags().BoolVar(&ofYaml, "yaml", false, "Output in YAML")
rootCmd.MarkFlagsMutuallyExclusive("json", "yaml")
```

在这两种情况下：

本地和持久标志都可以使用
注意：该组仅对定义了每个标志的命令强制执行
一个标志可能出现在多个组中
一个组可以包含任意数量的标志


### 位置和自定义参数

可以使用 的Args字段指定位置参数的验证Command。内置了以下验证器：

参数数量：

  - NoArgs- 如果有任何位置参数则报告错误。
  - ArbitraryArgs- 接受任意数量的参数。
  - MinimumNArgs(int)- 如果提供的位置参数少于 N 个，则报告错误。
  - MaximumNArgs(int)- 如果提供了超过 N 个位置参数，则报告错误。
  - ExactArgs(int)- 如果不完全有 N 个位置参数，则报告错误。
  - RangeArgs(min, max)- 如果 args 的数量不在 和 之间，则报告min错误max。

论证内容：

  OnlyValidArgsValidArgs- 如果在字段中没有指定任何位置参数，则报告错误Command，可以选择将其设置为位置参数的有效值列表。
  如果Args是 undefined 或nil，则默认为ArbitraryArgs.
  
  此外，MatchAll(pargs ...PositionalArgs)可以将现有检查与任意其他检查相结合。ValidArgs例如，如果你想在不正好有 N 个位置参数或者有任何位置参数不在的字段中时报错，Command你可以调用MatchAlland ExactArgs，OnlyValidArgs如下所示：

```golang
var cmd = &cobra.Command{
  Short: "hello",
  Args: cobra.MatchAll(cobra.ExactArgs(2), cobra.OnlyValidArgs),
  Run: func(cmd *cobra.Command, args []string) {
    fmt.Println("Hello, World!")
  },
}
```

可以设置任何满足func(cmd *cobra.Command, args []string) error. 例如:

```golang
var cmd = &cobra.Command{
  Short: "hello",
  Args: func(cmd *cobra.Command, args []string) error {
    // Optionally run one of the validators provided by cobra
    if err := cobra.MinimumNArgs(1)(cmd, args); err != nil {
        return err
    }
    // Run the custom validation logic
    if myapp.IsValidColor(args[0]) {
      return nil
    }
    return fmt.Errorf("invalid color specified: %s", args[0])
  },
  Run: func(cmd *cobra.Command, args []string) {
    fmt.Println("Hello, World!")
  },
}

```


## 例子

在下面的示例中，我们定义了三个命令。两个在顶层，一个 (cmdTimes) 是顶级命令之一的子级。在这种情况下，root 是不可执行的，这意味着需要一个子命令。这是通过不为“rootCmd”提供“运行”来实现的。

我们只为单个命令定义了一个标志。

```golang
package main

import (
  "fmt"
  "strings"

  "github.com/spf13/cobra"
)

func main() {
  var echoTimes int

  var cmdPrint = &cobra.Command{
    Use:   "print [string to print]",
    Short: "Print anything to the screen",
    Long: `print is for printing anything back to the screen.
For many years people have printed back to the screen.`,
    Args: cobra.MinimumNArgs(1),
    Run: func(cmd *cobra.Command, args []string) {
      fmt.Println("Print: " + strings.Join(args, " "))
    },
  }

  var cmdEcho = &cobra.Command{
    Use:   "echo [string to echo]",
    Short: "Echo anything to the screen",
    Long: `echo is for echoing anything back.
Echo works a lot like print, except it has a child command.`,
    Args: cobra.MinimumNArgs(1),
    Run: func(cmd *cobra.Command, args []string) {
      fmt.Println("Echo: " + strings.Join(args, " "))
    },
  }

  var cmdTimes = &cobra.Command{
    Use:   "times [string to echo]",
    Short: "Echo anything to the screen more times",
    Long: `echo things multiple times back to the user by providing
a count and a string.`,
    Args: cobra.MinimumNArgs(1),
    Run: func(cmd *cobra.Command, args []string) {
      for i := 0; i < echoTimes; i++ {
        fmt.Println("Echo: " + strings.Join(args, " "))
      }
    },
  }

  cmdTimes.Flags().IntVarP(&echoTimes, "times", "t", 1, "times to echo the input")

  var rootCmd = &cobra.Command{Use: "app"}
  rootCmd.AddCommand(cmdPrint, cmdEcho)
  cmdEcho.AddCommand(cmdTimes)
  rootCmd.Execute()
}

```


## 帮助命令

当您有子命令时，Cobra 会自动向您的应用程序添加帮助命令。这将在用户运行“应用程序帮助”时调用。此外，帮助还将支持所有其他命令作为输入。比如说，你有一个名为“create”的命令，没有任何额外的配置；Cobra 将在调用“app help create”时工作。每个命令都会自动添加“--help”标志。

### 定义你自己的帮助

您可以为默认命令提供您自己的帮助命令或您自己的模板，以与以下功能一起使用：

```golang
cmd.SetHelpCommand(cmd *Command)
cmd.SetHelpFunc(f func(*Command, []string))
cmd.SetHelpTemplate(s string)
```

## 用法

当用户提供无效标志或无效命令时，Cobra 通过向用户显示“用法”来响应。
定义自己的用法

您可以提供自己的使用函数或模板供 Cobra 使用。像帮助一样，函数和模板可以通过公共方法覆盖：

```golang
cmd.SetUsageFunc(f func(*Command) error)
cmd.SetUsageTemplate(s string)
```

## 版本标志

如果在根命令上设置了 Version 字段，Cobra 会添加一个顶级“--version”标志。使用“--version”标志运行应用程序将使用版本模板将版本打印到标准输出。可以使用该 cmd.SetVersionTemplate(s string)功能自定义模板。


## PreRun 和 PostRun 挂钩

可以在Run命令的主要功能之前或之后运行功能. PersistentPreRun和PreRun函数将在run之前执行。PersistentPostRun 和 PostRun 将在run之后执行。 Persistent*Run将由孩子继承。 函数按以下顺序运行:

1. PersistentPreRun
2. PreRun
3. Run
4. PostRun
5. PersistentPostRun


下面是使用所有这些功能的两个命令的示例。执行子命令时，它将运行根命令PersistentPreRun但不运行根命令PersistentPostRun：

```golang
package main

import (
  "fmt"

  "github.com/spf13/cobra"
)

func main() {

  var rootCmd = &cobra.Command{
    Use:   "root [sub]",
    Short: "My root command",
    PersistentPreRun: func(cmd *cobra.Command, args []string) {
      fmt.Printf("Inside rootCmd PersistentPreRun with args: %v\n", args)
    },
    PreRun: func(cmd *cobra.Command, args []string) {
      fmt.Printf("Inside rootCmd PreRun with args: %v\n", args)
    },
    Run: func(cmd *cobra.Command, args []string) {
      fmt.Printf("Inside rootCmd Run with args: %v\n", args)
    },
    PostRun: func(cmd *cobra.Command, args []string) {
      fmt.Printf("Inside rootCmd PostRun with args: %v\n", args)
    },
    PersistentPostRun: func(cmd *cobra.Command, args []string) {
      fmt.Printf("Inside rootCmd PersistentPostRun with args: %v\n", args)
    },
  }

  var subCmd = &cobra.Command{
    Use:   "sub [no options!]",
    Short: "My subcommand",
    PreRun: func(cmd *cobra.Command, args []string) {
      fmt.Printf("Inside subCmd PreRun with args: %v\n", args)
    },
    Run: func(cmd *cobra.Command, args []string) {
      fmt.Printf("Inside subCmd Run with args: %v\n", args)
    },
    PostRun: func(cmd *cobra.Command, args []string) {
      fmt.Printf("Inside subCmd PostRun with args: %v\n", args)
    },
    PersistentPostRun: func(cmd *cobra.Command, args []string) {
      fmt.Printf("Inside subCmd PersistentPostRun with args: %v\n", args)
    },
  }

  rootCmd.AddCommand(subCmd)

  rootCmd.SetArgs([]string{""})
  rootCmd.Execute()
  fmt.Println()
  rootCmd.SetArgs([]string{"sub", "arg1", "arg2"})
  rootCmd.Execute()
}
```

## 发生“未知命令”时的建议

当“未知命令”错误发生时，Cobra 将打印自动建议。git这允许 Cobra在发生拼写错误时表现得与命令相似。例如：


## Command 结构

```golang

type Command struct{
  // 具体的指令
  Use string 
  
  // 注释简短介绍
  Short string 
  //预期的参数 type PositionalArgs func(cmd *Command, args []string) error
  Args PositionalArgs
  // 长注释，通过-h --help来展示
  Long string 
  // 对命令的使用做一个简单的示例，在-h的时候会展示
  Example string
  // PreRun方法之前执行，并且子命令一样会执行
  PersistentPreRun func(cmd *Command, args []string)
 // PreRun方法之前执行，并且子命令一样会执行，返回一个error，如果该方法重写那么PersistentPreRun不会执行
  PersistentPreRunE func(cmd *Command, args []string) error
  // Run函数执行前执行
  PreRun func(cmd *Cmmand,args []string)  
  // Run函数执行前执行,返回一个error,如果重写了这个方法，PreRun不会执行
  PreRunE func(cmd *Command, args []string) error
  // 命令真正执行的函数，大多数命令都只会实现这个方法
  Run func(cmd *Command,args []string)
  // 命令执行的函数，但是会返回一个error，如果实现了这个方法那么Run方法不会再继续执行
  RunE func(cmd *Command,args []string)error
  // 该函数执行发生在 Run的方法之后
  PostRun func(cmd *Command,args []string)
  // 该函数执行发生在 Run的方法之后，返回一个error,如果重写了这个方法那么PostRun方法将不会执行
  PostRunE func(cmd *Command,args []string)error
  // 该函数执行发生在PostRun之后，并且子命令也会执行
  PersistentPostRun(cmd *Command,args []string)
  // 该函数执行发生在PostRun之后， 并且子命令也会执行，返回一个error，如果重写了这个方法，那么PersistentPostRun不会再执行
  PersistentPostRunE func(cmd *Command, args []string) error
  // 开启建议
  SuggestionsMinimumDistance int
  // 开启建议命令
  SuggestFor []string
}

```