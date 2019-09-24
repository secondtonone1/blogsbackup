---
title: Go(04)Cobra命令行参数库的使用
date: 2019-04-29 16:33:54
categories: [golang]
tags: [golang]
---
## 下载和安装
Cobra是golang的命令行参数库，可以配置命令启动，读取参数等。
将cobra下载到 $GOPATH，用命令：
``` bash
go get -v github.com/spf13/cobra/cobra
```
然后使用 go install github.com/spf13/cobra/cobra, 安装后在 $GOBIN 下出现了cobra 可执行程序。
cobra程序只能在GOPATH之下使用，所以首先你需要进入到GOPATH的src目录之下，在该目录下，输入:
``` bash
cobra init demo
```
$GOPATH目录下生成了demo文件夹,其组织形式如下
``` bash
demo
├── cmd 
│   └── root.go
├── LICENSE
└── main.go
```
进去该文件夹，运行:
``` bash
go run main.go
```
<!--more-->
控制台输出如下
``` bash
A longer description that spans multiple lines and likely contains examples and usage of using your application. For example:

Cobra is a CLI library for Go that empowers applications.
This application is a tool to generate the needed files
to quickly create a Cobra application.
```
如果你并不想运行cobra的可执行命令生成示例代码，只想在项目使用其库代码，则上面的内容可以忽略。
## 添加子命令
在$GOPATH下执行
``` bash
cobra add test
```
完成后demo结构如下
``` bash
├── cmd
│   ├── root.go
│   └── test.go
├── LICENSE
└── main.go
```
在cmd目录下，已经生成了一个与我们命令同名的go文件，你也许已经猜测到，与该命令有关的操作也正是在此处实现。现在执行这个子命令:
``` bash
go run main.go test
```
命令行将会打印输出test called,因为在test.go的run函数里输出test called
其实添加自命令也可以通过修改对应的go文件，比如添加test.go命令下的自命令,我们建立一个文件subtest.go,将test.go内容复制给test.go，
然后做如下修改
``` golang
var subtestCmd = &cobra.Command{
	Use:   "subtest",
	Short: "A brief description of your command",
	Long: `A longer description that spans multiple lines and likely contains examples
and usage of using your command. For example:

Cobra is a CLI library for Go that empowers applications.
This application is a tool to generate the needed files
to quickly create a Cobra application.`,
	Run: func(cmd *cobra.Command, args []string) {
		fmt.Println("subtest called")
	},
}
```
将cobra.Command定义的变量改为subtestCmd,然后将Use和Run说明改下,接着修改init函数，将subtestcmd加入到testcmd中，
因为二者都属于cmd包，所以可以直接访问使用
``` golang
func init() {
    testCmd.AddCommand(subtestCmd)
}
```
现在在命令行运行:
``` bash
go run main.go test subtest
```
如果结果为subtest called则调用自命令成功
## 添加参数
在test.go的init函数中，添加如下内容:
``` golang
testCmd.PersistentFlags().String("foo", "", "A help for foo")
testCmd.Flags().String("foolocal", "", "A help for foo")
```
运行go run main.go test
会看到如下结果
``` bash
Cobra is a CLI library for Go that empowers applications.
This application is a tool to generate the needed files
to quickly create a Cobra application.

Usage:
  demo test [flags]
  demo test [command]

Available Commands:
  subtest     A brief description of your command

Flags:
      --foo string   A help for foo
  -h, --help         help for test
  -t, --toggle       Help message for toggle

Global Flags:
      --config string   config file (default is $HOME/.demo.yaml)

Use "demo test [command] --help" for more information about a command.
```
接着让我们再运行 go run main.go test subtest -h 
会看到如下结果
``` bash
Cobra is a CLI library for Go that empowers applications.
This application is a tool to generate the needed files
to quickly create a Cobra application.

Usage:
  demo test subtest [flags]

Flags:
  -h, --help   help for subtest

Global Flags:
      --config string   config file (default is $HOME/.demo.yaml)
      --foo string      A help for foo
```
仔细对比发现subtest的Flags里没有-t，在Global Flags里多了--foo 选项。说明数作为persistent flag存在时，如注释所言，在其所有的子命令之下该参数都是可见的。而local flag则只能在该命令调用时执行
## 获取参数值
修改test.go run函数，获取参数并打印
``` golang
Run: func(cmd *cobra.Command, args []string) {
         fmt.Println("test called")
         fmt.Println(cmd.Flags().GetString("foo"))
         fmt.Println(cmd.Flags().GetBool("toggle"))
     },
```
运行go run main.go test  --foo "hello" -t false
结果如下
``` bash
test called
hello <nil>
true <nil>
```
通过cobra库可以很方便的启动多个命令行程序，目前就介绍这些。
[源码下载地址](https://github.com/secondtonone1/golang-)
感谢关注我的公众号
![wxgzh.jpg](wxgzh.jpg)
