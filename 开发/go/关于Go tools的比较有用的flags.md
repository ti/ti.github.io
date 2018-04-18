---
tags: [go, 代码调优]
date: 2018-04-16
author: Cyeam
author_link: https://rakyll.org/go-tool-flags/
---

# 【译】关于Go tools的比较有用的flags

你刚接触Go tools吗？或者你想扩展下你的知识面？这篇文章是关于Go tools的flags，这些flags每个人都应该知道。

免责声明：这篇文件可能有一些偏见。这是我个人常用的flags集合。我周边的人很难找到这些falgs的参考文档。如果你有更好的主意，可以在[Twitter](https://twitter.com/rakyll)上私信我。

## $ go build -x

`-x`列出了go build触发的所有命令。

如果你对Go的工具链、使用跨平台编译器比较好奇，或者对传入外部编译器的flags不清楚，或者怀疑链接器有bug，那么使用`-x`来查看所有的触发。

## $ go build -x

```
WORK=/var/folders/00/1b8h8000h01000cxqpysvccm005d21/T/go-build600909754
mkdir -p $WORK/hello/perf/_obj/
mkdir -p $WORK/hello/perf/_obj/exe/
cd /Users/jbd/src/hello/perf
/Users/jbd/go/pkg/tool/darwin_amd64/compile -o $WORK/hello/perf.a -trimpath $WORK -p main -complete -buildid bbf8e880e7dd4114f42a7f57717f9ea5cc1dd18d -D _/Users/jbd/src/hello/perf -I $WORK -pack ./perf.go
cd .
/Users/jbd/go/pkg/tool/darwin_amd64/link -o $WORK/hello/perf/_obj/exe/a.out -L $WORK -extld=clang -buildmode=exe -buildid=bbf8e880e7dd4114f42a7f57717f9ea5cc1dd18d $WORK/hello/perf.a
mv $WORK/hello/perf/_obj/exe/a.out perf
```

## $go build -gcflags

用来给Go编译器传入参数。`go tool compile -help` 列出了可以被传入编译器的所有的参数列表。

比如，为了禁止编译器优化和内联，你可以使用下面的gcfalgs：

```
$ go build -gcflags="-N -l"
```

比如，为了检查代码中有哪些可以调优的选项，可以运行下面指令：

```
$ go build -gcflags="-m"
```


## $go test -v

它提供了非正式的测试输出，打印了测试的名字、状态（通过或者失败）、耗时、测试用例的日志等。

不带有`-v`flag的go test命令非常安静，我经常把`-v`开关打开。比如输出如下：

```
$ go test -v context
=== RUN   TestBackground
--- PASS: TestBackground (0.00s)
=== RUN   TestTODO
--- PASS: TestTODO (0.00s)
=== RUN   TestWithCancel
--- PASS: TestWithCancel (0.10s)
=== RUN   TestParentFinishesChild
--- PASS: TestParentFinishesChild (0.00s)
=== RUN   TestChildFinishesFirst
--- PASS: TestChildFinishesFirst (0.00s)
=== RUN   TestDeadline
--- PASS: TestDeadline (0.16s)
=== RUN   TestTimeout
--- PASS: TestTimeout (0.16s)
=== RUN   TestCanceledTimeout
--- PASS: TestCanceledTimeout (0.10s)
...
PASS
ok      context 2.426s
```

## $go test -race

[Go竞争检测工具](https://blog.golang.org/race-detector)可以通过`--race`使用。go test也支持这个flag并且报告竞争。在开发阶段使用这个flag可以检测竞争。

## $go test -run

使用`-run`flag，你可以通过正则过滤测试用例。下面的命令会只测试[test examples](https://blog.golang.org/examples)：

```
$ go test -run=Example
```

## $go test -coverprofile

你可以输出一个覆盖信息，如果你在测试一个包，然后使用go tool来在浏览器上实现可视化：

```
$ go test -coverprofile=c.out && go tool cover -html=c.out
```

上面的命令会创建一个覆盖信息，然后在浏览器上打开结果页面。可视化后的结果会类似下面的页面：

![go test -coverprofile](https://raw.githubusercontent.com/itfanr/articles-about-golang/master/2016-09/2016-09-27-1-1.png)

## $go test -exec

这是一个鲜为人知的特性，使用`-exec`这个flag，你可以用另外的程序和tools交互。这个flag允许你使用Go tool把一些工作代理到另外的程序。

使用这个flag常用的需求场景是：当你需要做更多的事情，而不是仅仅执行宿主机的程序。Go的[Android build](https://github.com/golang/go/blob/master/misc/android/go_android_exec.go)，使用了`-exec`来推送测试二进制文件到Android设备（通过使用`adb`），并收集测试结果。可以作为一个参考。

## $go get -u

如果你执行go-test命令来获取一个已经在GOPATH中的包，那么go-get不好更新包到最新版本，而`-u`会强制tool同步这个仓库的最新的版本。

如果你是一个library的作者，那么你可能喜欢写你的安装说明通过`-u`flag，比如，[golin](https://github.com/golang/lint#installation)t这样的方式：

```
$ go get -u github.com/golang/lint/golint
```

## $go get -d

如果你只想clone一个repo到GOPATH，跳过编译和安装过程，那么使用`-d`。它会下载包，然后在尝试编译和安装之前停止。

我经常使用它，作为git clone的替代命令，使用虚假的URLs，因为它会克隆这个repo到它合适的GOPATH。比如：

```
$ go get -d golang.org/x/oauth2/...
```

会克隆包到`$GOPATH/src/golang.org/x/ouath2`。给出的`golang.org/x/oauth2`是一个虚假的URL，go-get这个仓库是很有用的，而不是尝试知道知己的repo是什么（go.googlesource.com/oauth2）。

## $go get -t

如果你的包需要额外的包来测试，`-t`会允许你在go-get过程中下载它们。如果你不传入`-t`参数，go get会只下载非测试代码的依赖。

## $ go list -f

允许你下载Go包以一种自定义的格式。对写bash脚本非常有用。下面的命令会输出runtime包的依赖：

```
$ go list -f '{{.Deps}}' runtime
[runtime/internal/atomic runtime/internal/sys unsafe]
```

[英文原文](https://rakyll.org/go-tool-flags/)

> 写在后面：虽然我们开发过程中不想被语言层面的东西限制太多，但是一些偶尔熟悉下这些命令的确可以使我们少走一些弯路。
