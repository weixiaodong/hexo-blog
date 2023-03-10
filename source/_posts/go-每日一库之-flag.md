---
title: go 每日一库之 flag
date: 2023-03-07 14:06:02
tags:
---


go语言内置的flag包实现了命令行参数的解析，flag包使得开发命令行工具更为简单。

<!--more-->

## os.Args
如果你只是简单的想要获取命令行参数，可以像下面的代码示例一样使用 os.Args 来获取命令行参数。


```golang
package main

import (
	"fmt"
	"os"
)

func main() {
	fmt.Println(os.Args)
}

```

os.Args 是一个存储命令行参数的字符串切片，它的第一个元素是执行文件的名称



## flag是怎么解析参数的

我们知道flag包是用于命令行解析的，但其内部是怎么解析的呢？下面我们来分析一下

一个命令行参数包含以下四个部分：

- 接收参数的变量
- 参数名称
- 默认值
- 参数说明

所以flag设置命令行参数的函数有四个参数，比如:

```golang
var p int
flag.IntVar(&p, "port", 3306, "数据库端口")
```

flag 内部有一个名称commandLine的变量，其类型为FlagSet,如:

```golang
var CommandLine = NewFlagSet(os.Args[0], ExitOnError)
```

FlagSet就是一个命令行参数的集合体，当我们调用诸如IntVar这类的函数时，就是将命令行的默认值、参数说明、参数名称、接收参数的变量等信息告诉flag库，而flag内部会让CommandLine来处理，用这些信息创建Flag类型的变量，将添加到这个集合体中。

```golang
flag := &Flag(name, usage, value, value.String())

```

最后，当我们调用flag.Parse函数时，实际就是调用FlagSet结构体的Parse函数将命令参数解析到变量中，flag.Parse函数代码如下:

```goalng
func Parse() {
	CommandLine.Parse(os.Args[1:])
}

```

从上面的代码我们也可以看出来，FlagSet的Parse函数最终是通过获取os.Args数组的数据来解析命令行参数的。

既然我们知道flag是通过类型为FlagSet的变量CommandLine来处理命令行参数的，那其实我们也可以自己创建一个FlagSet类型的变量来处理命令行参数，所以我们可以将上面的例子改成下面的样子:
```golang
package main

import (
	"flag"
	"fmt"
	"os"
)

func main() {
	var flagSet = flag.NewFlagSet("my flag", flag.ExitOnError)

	host := flagSet.String("host", "", "数据库地址")
	dbName := flagSet.String("db_name", "", "数据库名称")
	user := flagSet.String("user", "", "数据库用户")
	password := flagSet.String("password", "", "数据库密码")
	port := flagSet.Int("port", 3306, "数据库端口")

	flagSet.Parse(os.Args[1:])

	fmt.Printf("数据库地址: %s\n", *host)
	fmt.Printf("数据库名称: %s\n", *dbName)
	fmt.Printf("数据库用户: %s\n", *user)
	fmt.Printf("数据库密码: %s\n", *password)
	fmt.Printf("数据库端口: %d\n", *port)
}

```


## 自定义数据类型

如果flag提供的类型不能满足我们的需要，我们也可以自定义类型，自定义类型需要实现flag中的Value接口，该接口定义如下:

```golang

type value interface {
	String() string
	Set(string) error
}

```
value类型的String方法用于打印，Set方法则用于flag包将命令行参数解析到value类型中。

下面是一个自定义类型的示例程序：

```golang
package main

import (
	"flag"
	"fmt"
	"strings"
)

type users []string

func (u *users) Set(val string) error {
	*u = strings.Split(val, ",")
	return nil
}

func (u *users) String() string {
	return fmt.Sprint(*u)
}

func main() {

	var u users
	flag.Var(&u, "u", "用户列表")
	flag.Parse()
	fmt.Println(u)
}


```

从上面的示例中我们可以总结自定义类型的几个步骤:
1. 定义一个实现flag.Value接口的类型，即实现String和Set方法
2. 使用flag.Var函数将类型绑定到类型参数。
3. 调用flag.Parse()解析命令行参数。

## 短选项

我们在使用linux命令的时候，发现很多命令的参数是有分短选项和长选项的，不过flag库并不支持短选项；当然也有变通的方式，比如我们可以自己定义一个长选项和短选项，如:

```golang
var port int
flag.IntVar(&port, "p", 3306, "数据库端口")
flag.IntVar(&port, "port", 3306, "数据库端口")
flag.Parse()

```

## flag 参数类型

flag包支持的命令行参数类型有bool、 int、 int64、uint、 uint64、 float、 float64、 string、 duration



|flag参数 | 有效值 |
| ----- | --------- |
| 字符串flag	| 合法字符串 |
| 整数flag | 1234、0664、0x1234等类型，也可以是负数。|
| 浮点数flag	| 合法浮点数 |
| bool类型flag | 1, 0, t, f, T, F, true, false, TRUE, FALSE, True, False。|
| 时间段flag	| 任何合法的时间段字符串。如”300ms”、”-1.5h”、”2h45m”。合法的单位有”ns”、”us” /“µs”、”ms”、”s”、”m”、”h”。 |


# 定义命令行flag参数

有以下两种常用的定义命令行flag参数的方法

```bash
flag.Type()
```

基本格式如下：

```bash
flag.Type(flag名, 默认值, 帮助信息) *Type 
```

例如我们要定义姓名、年龄、婚否三个命令行参数，我们可以按如下方式定义：

```golang
name := flag.String("name", "张三", "姓名")
age := flag.Int("age", 18, "年龄")
married := flag.Bool("married", false, "婚否")
delay := flag.Duration("d", 0, "时间间隔")
```

需要注意的是，此时name、 age、 married、 delay均为对应类型的指针。

第二种形式
```bash
flag.TypeVar()
```

基本格式如下：

```bash
flag.TypeVar(Type指针, flag名, 默认值, 帮助信息)
```

例如我们要定义姓名、年龄、婚否三个命令行参数，我们可以按如下方式定义：

```golang
var name string
var age int
var married bool
var delay time.Duration
flag.StringVar(&name, "name", "张三", "姓名")
flag.IntVar(&age, "age", 18, "年龄")
flag.BoolVar(&married, "married", false, "婚否")
flag.DurationVar(&delay, "d", 0, "时间间隔")
```


## flag.Parse()

通过以上两种方法定义好命令行flag参数后，需要通过调用flag.Parse()来对命令行参数进行解析。

支持的命令行参数格式有以下几种：

- -flag xxx (使用空格， 一个 - 符号)
- --flag xxx (使用空格， 两个 - 符号)
- -flag=xxx (使用等号， 一个 - 符号)
- --flag=xxx (使用等号， 两个 - 符号)

其中，布尔类型的参数必须使用等号的方式指定。
