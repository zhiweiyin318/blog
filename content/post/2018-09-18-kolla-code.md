---
author: "Zhiwei Yin"
date: 2018-09-18
lastmod: 2018-11-23
draft: false
title: "kolla源码走读"
description: ""
tags: [
    "kolla",
    "openstack"
]
categories: [
    "openstack"
]
---

> 代码路径：https://github.com/openstack/kolla  
> 代码版本: origin/stable/pike branch  
> 文档: https://docs.openstack.org/kolla/latest/  

# 概述  

kolla项目主要是把容器化的openstack的各个组件的容器编译出来的，实现很简单：  
1. 定义Dockerfile.j2模板  
2. 通过jinja2根据参数渲染模板生成Dockerfile  
3. 调用docker的python接口build镜像  

**kolla 项目的难点不在项目的实现上，在各个组件编译的依赖上，rpm，pip安装相关组件的依赖上。**


# 项目目录结构

```
[root@yzw kolla]# tree -L 2
.
├── bindep.txt
├── contrib
├── doc
├── docker
│...
│   ├── base
│   │   ├── aarch64-cbs.repo
│   │   ├── ...
│   │   ├── Dockerfile.j2
│...
├── etc
│   ├── kolla
│   │   └── kolla-build.conf
│   └── oslo-config-generator
├── HACKING.rst
├── kolla
│   ├── cmd
│   ├── common
│   ├── exception.py
│   ├── exception.pyc
│   ├── hacking
│   ├── image
│   ├── __init__.py
│   ├── __init__.pyc
│   ├── opts.py
│   ├── opts.pyc
│   ├── template
│   ├── tests
│   ├── version.py
│   └── version.pyc
├── kolla.egg-info
├── LICENSE
├── README.rst
├── releasenotes
├── requirements.txt
├── setup.cfg
├── setup.py
├── specs
├── test-requirements.txt
├── tests
├── tools
│   ├── build.py -> ../kolla/cmd/build.py
│   ├── cleanup-images
└── tox.ini
```

主要目录介绍：  
* docker目录  

容器目录，包含了所有容器的Dockerfile模板Dockerfile.j2文件，和其他需要打包的一堆文件。  

base目录是所有基础容器镜像目录，其他容器都是From这个镜像的。

* kolla目录  

放的源码，后面详细说。  

* tools目录  

一些工具，最主要的就是*build.py*，是*/kolla/cmd/build.py*的软链接，就是项目命令行工具的入口。

* etc目录  

放的是配置文件kolla-build.conf，项目所有配置项都在里面，有一部分是跟命令行重合的，如果命令行配置了，就覆盖掉配置文件的配置了。

# 源码

## 源码目录

```
[root@yzw kolla]# tree -L 3
.
├── cmd
│   ├── build.py
├── common
│   ├── config.py
│   ├── task.py
├── exception.py
├── hacking
├── image
│   ├── build.py
├── opts.py
├── template
│   ├── filters.py
│   ├── filters.pyc
│   ├── methods.py
│   └── methods.pyc
├── tests
├── version.py
└── version.pyc

```
目录介绍：

* cmd  

build.py是命令行入口

* common  

config.py是命令行和配置文件相关的定义和设置。所有参数都在这定义的。

* image  

build.py是build镜像的相关的流程。  

* template  

给模板渲染时增加过滤和函数（属于jinja2的机制），替换字符串之类的操作。  

* tests 

ut相关的东西。

## 用到的主要库
### oslo_config  

oslo作为OpenStack的通用组件，在每一个项目中都有用到，oslo.config主要用于命令行和配置项解析。

写了一个简单的使用介绍和domo，放到了github上：  

* 介绍：  
<https://zhiweiyin318.github.io/post/olso_config/>  
* demo:   
<https://github.com/zhiweiyin318/yzw.python.demo/tree/master/oslo_config>  

### jinja2

python的模板引擎，也是Flask的模板引擎，能根据模板字符串替换，有自己的语法规则。就是定义自己的模板，然后根据配置生成想要的内容。

写的简单的使用介绍和domo，放到了github上：  

* 介绍：  
https://zhiweiyin318.github.io/post/python-jinja2/  
* demo：  
https://github.com/zhiweiyin318/yzw.python.demo/tree/master/jinja2  

### dokcer-client

python的docker client的库，直接调用跟命令行一样使用docker。  

写的简单的使用介绍和domo，放到了github上：  

* 介绍:  
https://zhiweiyin318.github.io/post/python-docker-client/  
* demo:  
https://github.com/zhiweiyin318/yzw.python.demo/tree/master/dockerclient  

### 代码流程

```
cmd/build.py main()  ## 入口
  -> image/build.py build.run_build()
    -> common/config.py common_config.parse(...) ## 命令行，配置文件参数的定义配置注册。
    -> KollaWorker()  ## build的任务对象,把命令行，配置文件的参数拿出来给对象。
      -> docker.APIClient() ## 创建docker api client 用于调用docker 接口
    -> kolla.setup_working_dir() ## 新建个放dockerfile和构建的工作目录，把docker目录的文件拷贝过来
    -> kolla.find_dockerfiles() ## 检查下工作目录下的dokerfile.j2文件
    -> kolla.create_dockerfiles() ## 做镜像文件
      -> jinja2.Environment() ## jinja2的使用流程 见jinja2 介绍
      -> jinja2.FileSystemLoader() 
      -> template.render() ## 所有的*j2模板重新渲染，后面写入新建dockerfie文件
    -> kolla.build_queue() ## 创建多队列来执行build操作
      -> self.build_image_list() ## 根据配置生成所有image列表，已经依赖关系
      -> self.find_parents() ## 整理出image的父image
      -> self.filter_images() ## 根据配置的regex和profile生成需要编译的iamge列表
      -> queue.put(BuildTask(self.conf, image, push_queue))
    
      -> task.run()       
        -> BuildTask.run
          -> self.builder()
            -> make_an_archive ## 需要下载plugin和addition
            -> self.dc.build  ## 调用 docker api 创建image

            -> PushTask.run
              -> self.push_image(image)
                -> self.dc.push  ## 调用 docker api push image到仓库
    
```

## kolla-build.conf 配置参数说明
**DEFAULT**  

* **skip_parents：** 编译跳过父镜像（FROM的镜像），如果夫镜像没有，就报错了。  
* **skip_existing：**  编译跳过已经存在的镜像。  
* **list_dependencies：** 打印本次编译的镜像的依赖关系。  
* **save_dependency：** 保存镜像的依赖关系为Graphviz dot 格式，后面跟保存的文件名路径。  
* **profile：** 本次编译的组件组。  
* **regex：** 本次只编译的组件的正则表达式，这个参数设置后就不会编profile的镜像了。  
* **template-only：**  只生成dockerfile文件不build。  
* **base_image：** 没有配置，默认给centos。  
* **base_tag：** 没有配置，默认给latest。  
* **registry：** 配了，镜像的namespace就是 registry的地址/namespace的内容；没有配置，镜像的namespace就是 namespace的内容。  
* **namespace：** 默认为kolla。  