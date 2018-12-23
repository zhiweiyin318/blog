---
author: "Zhiwei Yin"
date: 2018-09-12
lastmod: 2018-09-12
draft: false
title: "《深入剖析Kubernetes》-学习笔记-exec-volume"
description: ""
tags: [
    "docker",
    "kubernetes",
    "volume",
    "exec"
]
categories: [
    "kubernetes"
]
---
# docker exec 是怎么进入容器的？  

容器进程python2，docker exec又重新拉起来了进程跟python2进程都是docker-containerd-shim的子进程。docker exec又是怎么进入 python2进程的Namespace的呢?  

```
root@yzw-vm:/home/yzw/docker# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
9e4bdd819dc0        python:2.7-slim     "python2"           3 seconds ago       Up 1 second                             flamboyant_darwin
root@yzw-vm:/home/yzw/docker# ps auxf
...
root      1488  0.3  1.9 1405040 78288 ?       Ssl  11:53   2:20 /usr/bin/dockerd -H fd://
root      1743  0.2  0.8 1319652 35928 ?       Ssl  11:53   1:30  \_ docker-containerd --config /var/run/docker/containerd/containerd.toml
root     11223  0.0  0.1   7500  4080 ?        Sl   23:07   0:00      \_ docker-containerd-shim -namespace moby -workdir /var/lib/docker/containerd/daemon/io.containerd.runtime.v1.linux/moby/9e4bdd819dc0b462eb
root     11243  0.0  0.1  30084  7764 pts/0    Ss+  23:07   0:00          \_ python2

root@yzw-vm:/home/yzw/docker# docker exec -it 9e4bdd819dc0 /bin/bash
root@9e4bdd819dc0:/# 

root@yzw-vm:/home/yzw/docker# ps auxf
...
root      1488  0.3  1.9 1405040 78208 ?       Ssl  11:53   2:21 /usr/bin/dockerd -H fd://
root      1743  0.2  0.8 1319652 35928 ?       Ssl  11:53   1:31  \_ docker-containerd --config /var/run/docker/containerd/containerd.toml
root     11223  0.0  0.0   7500  4020 ?        Sl   23:07   0:00      \_ docker-containerd-shim -namespace moby -workdir /var/lib/docker/containerd/daemon/io.containerd.runtime.v1.linux/moby/9e4bdd819dc0b462eb
root     11243  0.0  0.1  30084  7764 pts/0    Ss+  23:07   0:00          \_ python2
root     11367  0.0  0.0  19952  3444 pts/1    Ss+  23:12   0:00          \_ /bin/bash
```
Linux Namespace 创建了一个隔离的空间，但是一个进程的Namespace在宿主机上是以文件形式存在着的，进程的每一个namespace都链接到一个真是的文件。  

一个进程，可以加入到某个进程已有的Namespace当中，达到进入这个进程所在容器的目的，就是exec的实现原理。  

```
root@yzw-vm:/home/yzw/docker# docker inspect --format ‘{{.State.Pid}}’ 9e4bdd819dc0
‘11243’
root@yzw-vm:/home/yzw/docker# ls -l /proc/11243/ns
total 0
lrwxrwxrwx 1 root root 0 Sep 12 23:25 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 root root 0 Sep 12 23:12 ipc -> 'ipc:[4026532239]'
lrwxrwxrwx 1 root root 0 Sep 12 23:12 mnt -> 'mnt:[4026532237]'
lrwxrwxrwx 1 root root 0 Sep 12 23:07 net -> 'net:[4026532242]'
lrwxrwxrwx 1 root root 0 Sep 12 23:12 pid -> 'pid:[4026532240]'
lrwxrwxrwx 1 root root 0 Sep 12 23:25 pid_for_children -> 'pid:[4026532240]'
lrwxrwxrwx 1 root root 0 Sep 12 23:25 user -> 'user:[4026531837]'
lrwxrwxrwx 1 root root 0 Sep 12 23:12 uts -> 'uts:[4026532238]'

```
这个操作依赖一个setns()的系统调用。将需要进入的Namespace的文件描述符fd交给setns()，就可以将当前进程加入到这个Namespace里面了。下面小程序传入2个参数，一个是Namespace的文件名，第二个是执行的程序。

```
#define _GNU_SOURCE
#include <fcntl.h>
#include <sched.h>
#include <unistd.h>
#include <stdlib.h>
#include <stdio.h>

#define errExit(msg) do { perror(msg);exit(EXIT_FAILURE);} while(0)

int main(int argc, char *argv[]){
    int fd;
    fd =open(argv[1],O_RDONLY);
    if (setns(fd,0) == -1){
        errExit("setns");
    }
    execvp(argv[2],&argv[2]);
    errExit("execvp");
}
```
编译后，传入mnt的namespace文件。
```
root@yzw-vm:/home/yzw/docker# gcc -o setnc setns.c 

root@yzw-vm:/home/yzw/docker# docker exec -it 9e4bdd819dc0 /bin/bash
root@9e4bdd819dc0:/# cd /home/
root@9e4bdd819dc0:/home# ls
root@9e4bdd819dc0:/home# touch abc
root@9e4bdd819dc0:/home# ls
abc


root@yzw-vm:/home/yzw/docker# ./setnc /proc/11243/ns/mnt /bin/bash
root@yzw-vm:/# ls
bin  boot  dev	etc  home  lib	lib64  media  mnt  opt	proc  root  run  sbin  srv  sys  tmp  usr  var
root@yzw-vm:/# ls /home/
abc
root@yzw-vm:/# 

```

查下新启的bin/bash的进程的PID的mnt Namespace，和python2进程的一样。  

```
root@yzw-vm:/home/yzw/docker# ps axuf | grep /bin/bash
root     11863  0.0  0.0  18204  3156 pts/4    S+   23:57   0:00  |                       \_ /bin/bash
root     11866  0.0  0.0  21536  1084 pts/5    S+   23:57   0:00                          \_ grep --color=auto /bin/bash
root@yzw-vm:/home/yzw/docker# ls -l /proc/11863/ns/mnt 
lrwxrwxrwx 1 root root 0 Sep 12 23:57 /proc/11863/ns/mnt -> 'mnt:[4026532237]'

```
# docker volume 是怎么实现的？

docker volume又2种声明方式,比如把宿主机目录挂进容器/test目录：  

```
# 在宿主机/var/lib/docker/volumes/xxx/_data 挂到容器/test目录
$ docker run -v /test ...
# 把宿主机/home目录挂到容器/test目录
$ docker run -v /home:/test ...
```
docker怎么把宿主机上的目录挂到容器里面的？  

前面namespace总结中，当容器进程（dockerinit 容器初始化进程）被创建之后，尽管开启了Mount Namespace，但是它执行chroot或者pivot_root之前，容器进程一直可以看到宿主机上整个文件系统。宿主机上的容器镜像的各个层，在容器进程启动后，就会被联合挂载到/var/lib/docker/overlay2/xxx/merged目录中，这样容器的rootfs就准备好了。  

> 容器启动进程dockerinit，而不是应用进程ENTRYPOINT+CMD,dockerinit负责完成根目录准备，挂载设备目录配置hostname一系列初始化工作，然后通过execv()，让应用程序取代自己，成为PID=1的进程。  

只需要在rootfs准备好之后，chroot之前，把volume指定的目录，挂载到容器指定目录在宿主机上对应的目录（/var/lib/docker/overlay2/读写层/test），就可以了。这时候Mount Namespace已经开启，挂载后，容器里面是可见的，宿主机上看不到容器里面的挂载点，容器的隔离性不好被volume打破。  

这里用到的挂载技术，就是Linux的绑定挂载 bind mount 机制，主要作用就是将一个目录/文件，而不是一个设备，挂载到一个指定目录上。并且，这时候你在该挂载点上进行的任何操作，发生在被挂载的目录或者文件上，而原挂载点的内容则会被隐藏起来，不受影响。  

绑定挂载实际上是一个inode替换的过程，indoe可以理解为存放文件内容的“对象”，dentry，目录项，就是访问inode所使用的“指针”。 bind mount相当于/test的dentry，重定向到/home的inode。当修改/test目录，实际是修改/home目录的inode。一旦umount后，/test目录原先的内容会恢复。  

这个/test目录的内容，是挂载在容器rootfs的可读写层，是不会被docker commit提交的，以为docker commint是发生在宿主机空间，由于mount namespace的隔离，不知道绑定挂载的存储，所有/test目录会被打包，出现在新的镜像里面的，但是里面是空的。