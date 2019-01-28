---
author: "Zhiwei Yin"
title: "Go modules"
subtitle: "Go mudules Tutorial"
comments: true
date: 2019-01-24
lastmod: 2019-01-28
tags: ["golang"]
categories: ["golang"]
draft: false
---

# Go modules

> **Reference:**  
> * https://blog.golang.org/modules2019   
> * https://tonybai.com/2018/07/15/hello-go-module/  
> * https://github.com/golang/go/wiki/Modules   
> * https://www.melvinvivas.com/go-version-1-11-modules/   
> * https://github.com/golang/go/wiki/Modules

Go 1.11 之前,code必须在`GOPATH/src`路径下，依赖的package也必须在这个目录下，`go get `下载的包也在这个目录下。Go 1.11开始支持 **go modules**功能，code不需要放到 `GOPATH/src` 路径下了，可以放到任何地方。  

Go modules were initially released with Go1.11. Fixes and improvements in Go1.11.2 and the to-be-released Go1.12 have and will make Go modules even better.

目前Go 1.11版本默认情况,如果根目录下没有`go.mod`文件，还是使用`GOPATH`路径下找依赖包：

```
yzw@yzw-vm:/media/sf_share/code/hello$ ls
hello.go
yzw@yzw-vm:/media/sf_share/code/hello$ go build
hello.go:5:5: cannot find package "rsc.io/quote" in any of:
	/usr/local/go/src/rsc.io/quote (from $GOROOT)
	/home/yzw/go/src/rsc.io/quote (from $GOPATH)

```

需要通过修改环境变量来改变默认build的包管理模式 `GO111MODULE=on`:

```
yzw@yzw-vm:/media/sf_share/code/hello$ export GO111MODULE=on
yzw@yzw-vm:/media/sf_share/code/hello$ go build
go: cannot find main module; see 'go help modules'
```

使用go modules功能，需要通过`go mod init`命令创建 `go.mod`文件，或者手动创建该文件：

```
yzw@yzw-vm:/media/sf_share/code/hello$ cat hello.go 
package main

import (
    "fmt"
    "rsc.io/quote"
)

func main() {
    fmt.Println(quote.Hello())
}

yzw@yzw-vm:/media/sf_share/code/hello$ go mod init hello
go: creating new go.mod: module hello
yzw@yzw-vm:/media/sf_share/code/hello$ ls
go.mod  hello.go
yzw@yzw-vm:/media/sf_share/code/hello$ cat go.mod 
module hello

yzw@yzw-vm:/media/sf_share/code/hello$ go build
yzw@yzw-vm:/media/sf_share/code/hello$ ls
go.mod  go.sum  hello  hello.go

yzw@yzw-vm:/media/sf_share/code/hello$ cat go.mod
module hello

require rsc.io/quote v1.5.2

yzw@yzw-vm:/media/sf_share/code/hello$ cat go.sum 
golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c h1:qgOY6WgZOaTkIIMiVjBQcw93ERBE4m30iBm00nkL0i8=
golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c/go.mod h1:NqM8EUOU14njkJ3fqMW+pc6Ldnwhi/IjpwHt7yyuwOQ=
rsc.io/quote v1.5.2 h1:w5fcysjrx7yqtD/aO+QwRjYZOKnaM9Uh2b40tElTs3Y=
rsc.io/quote v1.5.2/go.mod h1:LzX7hefJvL54yjefDEDHNONDjII0t9xZLPXsUe+TKr0=
rsc.io/sampler v1.3.0 h1:7uVkIFmeBqHfdjD+gZwtXXI+RODJ2Wc4O7MPEh/QiW4=
rsc.io/sampler v1.3.0/go.mod h1:T1hPZKmBbMNahiBKFy5HrXp6adAjACjK9JXDnKaTXpA=

```

`go mod init`会自动在根目录下创建`go.mod`文件，当go build的时候，会自动将依赖包的版本加到`go.mod`文件中，并生成一个`go.sum`文件，这个文件里面有些依赖和meta信息。依赖的包不会下载到code目录，会在`GOPATH/pkg`目录下：

```
yzw@yzw-vm:/media/sf_share/code/hello$ cd ~/go/pkg/
yzw@yzw-vm:~/go/pkg$ ls
linux_amd64  mod
yzw@yzw-vm:~/go/pkg$ cd mod/
yzw@yzw-vm:~/go/pkg/mod$ ls
cache  github.com  golang.org  k8s.io  rsc.io
```

因为有墙的缘故，好多第三方包都没法下载，可以通过设置`GOPROXY` 环境变量来代理，可以使用 `https://goproxy.io`下载，或者自己搭建proxy:

```
# export GOPROXY=https://goproxy.io
```

关闭`GOPROXY` 只需将该环节变量置为空即可。详细可参考：

> * https://cloud.tencent.com/developer/news/308442  
> * https://tonybai.com/?s=module

`go list -m` 命令可以查询当前构建的依赖包极版本信息：

```
yzw@yzw-vm:/media/sf_share/code/hello$ go list -m -json all
{
	"Path": "hello",
	"Main": true,
	"Dir": "/media/sf_share/code/hello",
	"GoMod": "/media/sf_share/code/hello/go.mod"
}
{
	"Path": "golang.org/x/text",
	"Version": "v0.0.0-20170915032832-14c0d48ead0c",
	"Time": "2017-09-15T03:28:32Z",
	"Indirect": true,
	"Dir": "/home/yzw/go/pkg/mod/golang.org/x/text@v0.0.0-20170915032832-14c0d48ead0c",
	"GoMod": "/home/yzw/go/pkg/mod/cache/download/golang.org/x/text/@v/v0.0.0-20170915032832-14c0d48ead0c.mod"
}
{
	"Path": "rsc.io/quote",
	"Version": "v1.5.2",
	"Time": "2018-02-14T15:44:20Z",
	"Dir": "/home/yzw/go/pkg/mod/rsc.io/quote@v1.5.2",
	"GoMod": "/home/yzw/go/pkg/mod/cache/download/rsc.io/quote/@v/v1.5.2.mod"
}
{
	"Path": "rsc.io/sampler",
	"Version": "v1.3.0",
	"Time": "2018-02-13T19:05:03Z",
	"Indirect": true,
	"Dir": "/home/yzw/go/pkg/mod/rsc.io/sampler@v1.3.0",
	"GoMod": "/home/yzw/go/pkg/mod/cache/download/rsc.io/sampler/@v/v1.3.0.mod"
}

```

`"main": true` 标识当前module为**main module**,及`go build`时当前目录所属的module，go会在当前目录，当前目录的父目录，父目录的父目录...去找`go.mod`文件，作为main module。

code目录的子目录也是可以有自己的`go.mod`的。

为了兼容老版本`vendor`这种包管理方式，可以通过`go mod vendor`进行转换:

```
yzw@yzw-vm:/media/sf_share/code/hello$ go mod vendor
yzw@yzw-vm:/media/sf_share/code/hello$ ls
go.mod  go.sum  hello  hello.go  vendor

yzw@yzw-vm:/media/sf_share/code/hello$ ls vendor/
golang.org  modules.txt  rsc.io
yzw@yzw-vm:/media/sf_share/code/hello$ 

```

Goland 2018.3 版本支持创建go modules的工程：

欢迎界面 `New Project`：

![Create Goland go module project](/images/go-goland-create-modules-project.PNG)

![Goland go module project](/images/go-goland-create-modules-project-2.PNG)

创建go文件，运行，自动会更新`go.mod`信息，和管理依赖包：

![Run Goland go module project](/images/go-goland-create-modules-project-3.PNG)

右击`go.mod`文件选`Diagrams | Show Diagram`,可以看依赖包的图表形式：

![Show Goland go module project diagram](/images/go-goland-create-modules-project-4.PNG)

详细参考：  

> * https://blog.jetbrains.com/go/2019/01/22/working-with-go-modules/