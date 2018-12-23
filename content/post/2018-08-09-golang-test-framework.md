---
author: "Zhiwei Yin"
date: 2018-08-09
lastmod: 2018-08-09
draft: false
title: "golang的测试框架"
description: ""
tags: [
    "golang",
    "code",
]
categories: [
    "golang",
]
---

golang 测试框架
===============

本文主要介绍golang 测试的集中常见的框架。

go test
-------

* 文件名称位xx_test.go
* 测试函数Testxxx(t *testing.T)

```command
go test -v
```

GoConvey
-------

可以管理和运行测试用例，同时提供了丰富的断言函数，并支持很多 Web 界面特性，集成go test。

Write behavioral tests in your editor. Get live results in your browser.

* 项目：`https://github.com/smartystreets/goconvey`
* 官网：`http://goconvey.co/`
* 介绍：`https://www.jianshu.com/p/e3b2b1194830`

#### 安装

```command
go get github.com/smartystreets/goconvey 
```

1. 在$GOPATH/src目录下新增了github.com子目录，该子目录里包含了GoConvey框架的库代码
2. 在$GOPATH/bin目录下新增了GoConvey框架的可执行程序goconvey

#### 使用

* web显示结果，在测试目录下执行goconvoy就可以

```code
import (
    . "github.com/smartystreets/goconvey/convey"
    "testing"
)

func Test_ServerRun(t *testing.T) {
    Convey("Test ServerRun ", t, func() {
        Convey("Case 1 port > 10000:", func(){
            port := 100000
            So(ServerRun(port), ShouldBeError)
        })
        Convey("Case 2 port <= 10000:", func() {
            port := 1000
            So(ServerRun(port), ShouldBeNil)
        })
    })
}
```

GoStub
------

主要用来给全局变量打桩，也可以给函数打桩，无法给方法接口打桩。
* 项目：`https://github.com/prashantv/gostub`
* 介绍：`https://www.jianshu.com/p/70a93a9ed186`

#### 安装

```command
go get github.com/prashantv/gostub
```

#### 使用

```code
import (
    "testing"
    . "github.com/prashantv/gostub"
)
func Test_ServerRun_Case3(t *testing.T){
    Convey("Test ServerRun ",t,func(){
        Convey("Case 1 IP is not local IP:",func(){
            stubs :=Stub(&localIP,"192.168.1.1")
            defer stubs.Reset()
            So(ServerRun(8080), ShouldBeError)
        })
    })
}
```

Monkey
-------

可以给函数，方法打桩

* 项目：`https://github.com/bouk/monkey`
* 官网：`https://bou.ke/`
* 介绍：`https://www.jianshu.com/p/2f675d5e334e`

#### 安装

```command
go get github.com/bouk/monkey
```

#### 使用

* inline 函数打桩无效
* 方法的首字母大小写，只能给首大写字母的方法打桩

```code
import (
    "github.com/bouk/monkey"
    . "github.com/smartystreets/goconvey/convey"
    "reflect"
    "testing"
    "errors"
)
func Test_ServerRun_Monkey(t *testing.T) {
    Convey("Test ServerRun ", t, func() {
        Convey("Case 1 Listen failed:", func() {
            defer monkey.UnpatchAll()
            monkey.Patch(Listen, func(s *server) error {
                return errors.New("fake return fail")
            })
            So(ServerRun(8080), ShouldBeError)
        })

        Convey("Case 2 s.GetIP failed:", func() {
            var s *server
            defer monkey.UnpatchAll()
            monkey.PatchInstanceMethod(reflect.TypeOf(s), "GetIP", func(_ *server) string {
                return "192.168.1.1"
            })
            So(ServerRun(8080),ShouldBeError)
        })
    })
}
```

GoMock
-------

给接口打桩

* 项目：`https://github.com/golang/mock`
* 文档：`https://godoc.org/github.com/golang/mock/gomock`
* 参考：`https://www.jianshu.com/p/f4e773a1b11f`

#### 安装

```command
mkdir $GOPATH/src/golang.org/x/
cd $GOPATH/src/golang.org/x/
git clone https://github.com/golang/net.git net 
go install net
go get github.com/golang/mock/gomock
go install github.com/golang/mock/mockgen
```

#### 使用

* 生成接口的mock文件
* 输出目录需要提起建好
* gomock的代码需放在$GOPATH/src下，mockgen运行时要在这个路径下访问gomock

```command
./mockgen -source=/home/yzw/go/src/examples/pkg/common/common.go > /home/yzw/go/src/examples/test/common/mock_common.go
```

* 测试代码

```code
import (
    . "github.com/golang/mock/gomock"
    . "github.com/smartystreets/goconvey/convey"
    "testing"

    "examples/test/common"
    "errors"
    "github.com/bouk/monkey"
    "examples/pkg/common"
    "fmt"
)

func Test_ClientRun_GoMock(t *testing.T) {
    Convey("Test ClientRun", t, func() {
        ctrl := NewController(t)
        defer ctrl.Finish()
        mockOpter := mock_common.NewMockOpter(ctrl)
        mockOpter.EXPECT().Get(Any()).Return("mock get data", errors.New("mock get error"))

        defer monkey.UnpatchAll()
        monkey.Patch(NewClient, func() common.Opter {
            fmt.Println("fake newclient")
            return mockOpter
        })
        So(ClientRun("ip","port","opt"),ShouldBeError)
    })
}
```

* 如果mock的接口被调用多次，需要用Times

```code
mockOpter.EXPECT().Get(Any()).Return("mock get data", errors.New("mock get error")).Times(5)
```

* mock的接口有先后顺序的时候，需要同After，InOder来保序

```code
getCall := mockOpter.EXPECT().Get(Any()).Return("mock get data", errors.New("mock get error"))
mockOpter.EXPECT().Post(Any()).Return(errors.New("mock post error")).After(getCall)
InOder(
    mockOpter.EXPECT().Get(Any()).Return("mock get data", errors.New("mock get error"))
    mockOpter.EXPECT().Post(Any()).Return(errors.New("mock post error"))
)
```