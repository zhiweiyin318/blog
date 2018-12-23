---
author: "Zhiwei Yin"
date: 2018-09-10
lastmod: 2018-09-10
draft: false
title: "OpenStack olso_config"
description: ""
tags: [
    "python",
    "code"
]
categories: [
    "python"
]
---

# oslo
oslo作为OpenStack的通用组件，在每一个项目中都有用到，oslo.config主要用于命令行和配置项解析。  \

> **参考:**  
> * https://gtcsq.readthedocs.io/en/latest/openstack/oslo_cfg.html  
> * https://blog.csdn.net/zhangyifei216/article/details/50434980  
> * https://blog.csdn.net/hobertony_7/article/details/79206297  
> * https://www.programcreek.com/python/example/106149/oslo_config.cfg.ConfigOpts  

# 使用说明
demo的代码归档：  
https://github.com/zhiweiyin318/yzw.python.demo/tree/master/oslo_config

```
#!/usr/bin/env python

# reference:
# https://gtcsq.readthedocs.io/en/latest/openstack/oslo_cfg.html
# https://blog.csdn.net/zhangyifei216/article/details/50434980
# https://blog.csdn.net/hobertony_7/article/details/79206297
# https://www.programcreek.com/python/example/106149/oslo_config.cfg.ConfigOpts


from oslo_config import cfg
from oslo_config import types
import sys

BASE_OS_DISTRO = ['centos', 'rhel', 'ubuntu', 'oraclelinux', 'debian']
# 命令行参数定义
_CLI_OPTS = [
    cfg.StrOpt('os', short='o', default='centos',
               choices=BASE_OS_DISTRO,
               help='The system OS '
                    'are ' + ', '.join(BASE_OS_DISTRO)),
    cfg.IntOpt('cpu', short='c', default=1, min=1,
               help='The number of CPU')
]

#配置参数定义
_CFG_OPTS = [
    cfg.StrOpt("host", default="0.0.0.0",
               help="the host ip addr"),
    cfg.Opt("port", default=5999,
            type=types.Integer(1000, 6000),
            help="the port number")
]

_DEBUG_OPTS = [
    cfg.BoolOpt("debug", default=False,
                help="debug config True or False")
]


def main():

    # 定义configOpts实例还可以通过  conf = cfg.ConfigOpts()
    conf = cfg.CONF
    
    # 注册 命令和配置项
    conf.register_cli_opts(_CLI_OPTS)
    conf.register_opts(_DEBUG_OPTS)

    # 配置分组定义
    # DEFAULT是一个默认组，默认情况下Opt定义的选项都是属于这个组的，cfg Module给配置选项提供了组的概念，不同的组配置选项是可以重复的。
    ip_group = cfg.OptGroup(name="IP", title="ip cfg")
    conf.register_group(ip_group)

    # 待分组的配置项
    conf.register_opts(_CFG_OPTS, ip_group)

    # 获取参数，默认配置也可以通过--config-file来指定
    conf(args=sys.argv[1:],
         default_config_files=['test.conf'])

    print "System OS is ", conf.os
    print "The number of CPU is ", conf.cpu
    print "The debug: ", conf.debug
    print "Host IP: ", conf.IP.host
    print "Port number: ", conf.IP.port


if __name__ == '__main__':
    main()

```

配置文件定义
```
[DEFAULT]
debug = true

[IP]
host = 8.8.8.8
port = 4555
```