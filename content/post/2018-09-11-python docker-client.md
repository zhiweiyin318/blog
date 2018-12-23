---
author: "Zhiwei Yin"
date: 2018-09-11
lastmod: 2018-09-11
draft: false
title: "Python docker-client"
description: ""
tags: [
    "python",
    "code",
    "docker"
]
categories: [
    "python"
]
---
简单记录哈python的docker client的使用。  
> 官方文档：  
> https://docker-py.readthedocs.io/en/stable/client.html    
> https://docs.docker.com/develop/sdk/examples/  
> demo地址：   
> https://github.com/zhiweiyin318/yzw.python.demo/tree/master/dockerclient

# 安装
```
$ pip install docker
```
# 使用  
## client 初始化
需要先创建一个DockerClient类的对象，相当于本地的docker client端。有三种方式：
```
import docker

# 方式1 base_url 可以是socket或者tcp方式，还有version，timeout等参数
dc = docker.APIClient(base_url='unix://var/run/docker.sock',timeout=5)
# 方式2
docker_kwargs = docker.utils.kwargs_from_env()
dc = docker.APIClient(version='auto', **docker_kwargs)
# 方式3 这是上面2个的子集
dc = docker.from_env()
```

## client 的方法
client对象的有一些方法：
```
dc.configs
dc.containers
dc.images
dc.networks
dc.nodes
dc.plugins
dc.volumes
dc.services
dc.events
...
```
### 运行，查询，停止，删除 一个容器
```
>>> import docker
>>> dc = docker.from_env()
>>> dc.containers.run("busybox",'echo hello world')
'hello world\n'
>>> 
```
```
import docker
dc = docker.from_env()
container = dc.containers.run("busybox", name='test', command='echo hello world', detach=True, tty=True,
                                stdin_open=True)
print container.logs()
print "containers: {}".format(dc.containers.list())
print "container {} status is {}".format(container.name, container.status)
container.stop()
container.remove()
print "container {} status is {}".format(container.name, container.status)
print "containers: {}".format(dc.containers.list())
```

### list/pull image
```
import docker
dc = docker.from_env()
for i in dc.images.list():
    print "image ID:{}, tag:{}".format(i.short_id, i.tags)
    for t in i.tags:
        if t == "busybox:latest":
            print "remove image {}".format(t)
            dc.images.remove('busybox:latest')

# if no tag is specified all tags from the repo will be pulled.
image = dc.images.pull('busybox:latest')
print "pull busybox image, id:{},tag:{}".format(image.short_id, image.tags)
```

## Low-level API
The main object-orientated API is built on top of APIClient. Each method on APIClient maps one-to-one with a REST API endpoint, and returns the response that the API responds with.

It’s possible to use APIClient directly. Some basic things (e.g. running a container) consist of several API calls and are complex to do with the low-level API, but it’s useful if you need extra flexibility and power.
```
import docker
dc = docker.APIClient(base_url='unix://var/run/docker.sock')
    print dc.version()
dc.containers()
```
等同于：
```
import docker
dc = docker.from_env()
dc.api.containers()

```
输出结果是json结构的map
```
[{u'Status': u'Up Less than a second', u'Created': 1536673589, u'Image': u'busybox', u'Labels': {}, u'NetworkSettings': {u'Networks': {u'bridge': {u'NetworkID': u'754e02dfae9aa61b96e78ca6dfa743f121177c2985cd3179641e8650032180b4', u'MacAddress': u'02:42:ac:11:00:03', u'GlobalIPv6PrefixLen': 0, u'Links': None, u'GlobalIPv6Address': u'', u'IPv6Gateway': u'', u'DriverOpts': None, u'IPAMConfig': None, u'EndpointID': u'06443f233bfac60c83c9835172c9d076a9df406bd84a350e1e713fd274caf5d3', u'IPPrefixLen': 16, u'IPAddress': u'172.17.0.3', u'Gateway': u'172.17.0.1', u'Aliases': None}}}, u'HostConfig': {u'NetworkMode': u'default'}, u'ImageID': u'sha256:e1ddd7948a1c31709a23cc5b7dfe96e55fc364f90e1cebcde0773a1b5a30dcda', u'State': u'running', u'Command': u'/bin/sh', u'Names': [u'/cranky_lichterman'], u'Mounts': [], u'Id': u'ededc6515049f2d13b8ee79df0729bf9f8f4d92940e6012c4f1e23ccc055c0f7', u'Ports': []}, {u'Status': u'Up About a minute', u'Created': 1536673521, u'Image': u'busybox', u'Labels': {}, u'NetworkSettings': {u'Networks': {u'bridge': {u'NetworkID': u'754e02dfae9aa61b96e78ca6dfa743f121177c2985cd3179641e8650032180b4', u'MacAddress': u'02:42:ac:11:00:02', u'GlobalIPv6PrefixLen': 0, u'Links': None, u'GlobalIPv6Address': u'', u'IPv6Gateway': u'', u'DriverOpts': None, u'IPAMConfig': None, u'EndpointID': u'2a3d90402db5b9c26583a5bb4bad5c4561fb263cd2858d0e01774ab7fa5cc336', u'IPPrefixLen': 16, u'IPAddress': u'172.17.0.2', u'Gateway': u'172.17.0.1', u'Aliases': None}}}, u'HostConfig': {u'NetworkMode': u'default'}, u'ImageID': u'sha256:e1ddd7948a1c31709a23cc5b7dfe96e55fc364f90e1cebcde0773a1b5a30dcda', u'State': u'running', u'Command': u'/bin/sh', u'Names': [u'/loving_wright'], u'Mounts': [], u'Id': u'6721f996af1e9d04237fef0a1cb3f50e9e564173e8ff25056cdc475e8a985de5', u'Ports': []}]

```