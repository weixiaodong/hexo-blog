---
title: go flag包源码阅读
date: 2023-03-08 14:33:31
tags:
---


## 源码分析

<!--more-->

### 结构类型分析

![](/images/结构类型分析.png)


### 设计思路

1. 抽象命令行参数类型接口 value，每个参数对应flag类型
2. 抽象参数结合进行操作 flagSet类型

### 函数分析

#### flag.Parse():

过程很简单，命令行参数重头到尾一个个分析，是不是预定义的flag，如果是，设置相应的值，然后分析下一个命令行参数。
最后调用flag.Var() 根据传入的参数的name查看flag是否存在，存在就报错，不存在就实例化一个flag存储起来

内部调用parseOne，一个个进行flag set解析。

#### flag.Visit()系列:

遍历flag集合,对每个flag都执行指定函数


#### flag.Lookup():

查找通过flag的k,查找flag

#### flag.Set()

通过flag的k,设置flag的值(手动设置,使用场景在定义预期flag之后)

#### flag.PrintDefaults():

遍历所有的flag,打印 -flag名 值类型 usage 默认值 换行

#### flag.NFlag()

返回通过命令行参数设置的flag个数

#### flag.Arg()

返回第几个参数, 这个参数是命令行参数

#### flag.Args()

返回剩余的命令行参数


## 支持的格式

-a=b 写法 支持所有数据类型
-a 如果是bool型 默认a是true
-a b 如果不是bool型 等同于a=b, 如果a是bool型,那么b就无法识别为一个flag了

换句话说:

-a=b 最中庸的写法,是ok的
-a 写法 只适用于bool
-a b 写法 只适用于非bool型
对标准flag库来说 前缀是-和--都是同样的效果

```
官方文档上写的是:
-flag
-flag=x
-flag x  // non-boolean flags only
前缀是一个破折号或两个破折号是等价的
```


## 使用步骤:

1. 定义程序预期的flag

	- 这个定义的动作放在解析之前即可,一般推荐放在init()中
	- 定义的方式有多种,flag包提供了IntVar()系列/Int()系列
	- 除了定义规定的数据类型,还可以通过Var()定义自定义数据类型的flag

2. 解析flag

	flag.Parse()

3. 使用通过命令行参数传进来的值(这也是这个包最大的意义)

	在定义预期flag时,就指定了解析之后,值存放在哪个变量上,使用时直接用这个变量即可

## 关键部分源码

```golang
type Value interface {
	String() string
	Set(string) error
}

// A Flag represents the state of a flag.
type Flag struct {
	Name     string // name as it appears on command line
	Usage    string // help message
	Value    Value  // value as set
	DefValue string // default value (as text); for usage message
}

type FlagSet struct {
	// Usage is the function called when an error occurs while parsing flags.
	// The field is a function (not a method) that may be changed to point to
	// a custom error handler. What happens after Usage is called depends
	// on the ErrorHandling setting; for the command line, this defaults
	// to ExitOnError, which exits the program after calling Usage.
	Usage func()

	name          string
	parsed        bool
	actual        map[string]*Flag
	formal        map[string]*Flag
	args          []string // arguments after flags
	errorHandling ErrorHandling
	output        io.Writer // nil means stderr; use Output() accessor
}

func (f *FlagSet) Var(value Value, name string, usage string) {
	// Flag must not begin "-" or contain "=".
	if strings.HasPrefix(name, "-") {
		panic(f.sprintf("flag %q begins with -", name))
	} else if strings.Contains(name, "=") {
		panic(f.sprintf("flag %q contains =", name))
	}

	// Remember the default value as a string; it won't change.
	flag := &Flag{name, usage, value, value.String()}
	_, alreadythere := f.formal[name]
	if alreadythere {
		var msg string
		if f.name == "" {
			msg = f.sprintf("flag redefined: %s", name)
		} else {
			msg = f.sprintf("%s flag redefined: %s", f.name, name)
		}
		panic(msg) // Happens only if flags are declared with identical names
	}
	if f.formal == nil {
		f.formal = make(map[string]*Flag)
	}
	f.formal[name] = flag
}

// parseOne parses one flag. It reports whether a flag was seen.
func (f *FlagSet) parseOne() (bool, error) {
	if len(f.args) == 0 {
		return false, nil
	}
	s := f.args[0]
	if len(s) < 2 || s[0] != '-' {
		return false, nil
	}
	numMinuses := 1
	if s[1] == '-' {
		numMinuses++
		if len(s) == 2 { // "--" terminates the flags
			f.args = f.args[1:]
			return false, nil
		}
	}
	name := s[numMinuses:]
	if len(name) == 0 || name[0] == '-' || name[0] == '=' {
		return false, f.failf("bad flag syntax: %s", s)
	}

	// it's a flag. does it have an argument?
	f.args = f.args[1:]
	hasValue := false
	value := ""
	for i := 1; i < len(name); i++ { // equals cannot be first
		if name[i] == '=' {
			value = name[i+1:]
			hasValue = true
			name = name[0:i]
			break
		}
	}
	m := f.formal
	flag, alreadythere := m[name] // BUG
	if !alreadythere {
		if name == "help" || name == "h" { // special case for nice help message.
			f.usage()
			return false, ErrHelp
		}
		return false, f.failf("flag provided but not defined: -%s", name)
	}

	if fv, ok := flag.Value.(boolFlag); ok && fv.IsBoolFlag() { // special case: doesn't need an arg
		if hasValue {
			if err := fv.Set(value); err != nil {
				return false, f.failf("invalid boolean value %q for -%s: %v", value, name, err)
			}
		} else {
			if err := fv.Set("true"); err != nil {
				return false, f.failf("invalid boolean flag %s: %v", name, err)
			}
		}
	} else {
		// It must have a value, which might be the next argument.
		if !hasValue && len(f.args) > 0 {
			// value is the next arg
			hasValue = true
			value, f.args = f.args[0], f.args[1:]
		}
		if !hasValue {
			return false, f.failf("flag needs an argument: -%s", name)
		}
		if err := flag.Value.Set(value); err != nil {
			return false, f.failf("invalid value %q for flag -%s: %v", value, name, err)
		}
	}
	if f.actual == nil {
		f.actual = make(map[string]*Flag)
	}
	f.actual[name] = flag
	return true, nil
}

```