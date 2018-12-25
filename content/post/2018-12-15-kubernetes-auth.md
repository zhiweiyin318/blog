---
author: "Zhiwei Yin"
title: "Kubernetes 认证: X509客户端证书"
subtitle: "Kubernetes auth: X509 client certificates"
comments: true
date: 2018-12-25T22:40:54+08:00
lastmod: 2018-12-25T22:40:54+08:00
tags: ["kubernetes","translation"]
categories: ["kubernetes","translation"]
draft: false
---

> 原文：https://brancz.com/2017/10/16/kubernetes-auth-x509-client-certificates/  
> 作者：Frederic Branczyk  
> 翻译：Zhiwei Yin

Kubernetes支持[多种认证方式](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)，比如静态Token文件，静态用密码文件，[OIDC（OpenID Connect）](https://openid.net/connect/)这些认证方式已经在文档里详细说明了。但是还有一种常见的认证方式，就是使用[X509客户端证书认证](https://en.wikipedia.org/wiki/X.509)。这篇博客我将稍微解释下X509客户端证书认证，展示下怎么在Kubernetes集群中使用这种认证方式。


跟其他认证方式不同，客户端证书认证方式使用公共密钥来代替密码或者token。使用客户端证书认证的好处就是不再需要存储验证密码或者token相关信息了。客户端证书认证是纯加密解密方式，这也是为什么我们可通过证书验证请求是否可信的原因。

> 译者解释：  
> 密码或者token认证方式，是需要有数据库或者文件来存储合法的用户名密码或者token的，客户端证书认证是不需要单独存储这些信息的，是纯加密解密的方式来验证请求是否合法的。

Kubernetes自己没有存储user信息的，因此user身份必须来自选择的认证方式提供。比如如果你选择使用[Open ID Connect](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#openid-connect-tokens)来作为认证方式，则OIDC服务提供方负责存储维护user身份和认证信息，认证请求返回不同user，Kubernetes使用这些user信息进行认证。X509客户端证书完美的解决了这个问题，因为当认证内容被Kubernetes集群的CA证书签名后，Kubernetes apiserver只需要验证签名是否合法就可以了。这意味着被写入证书文件里面的user和group信息只被签名验证的时候使用一次，不需要再存储了。

> 译者解释：  
> Kubernetes的用户user是个认证授权体系逻辑概念，没有一个叫user的API的。如果使用像OIDC这种认证方式，OIDC认证服务需要存储和维护user信息，当用户发送的请求来了后，需要通过OIDC认证这个user是否合法。正式认证方式直接讲user信息写入证书里面了，只在证书被CA根证书签名的时候使用一次，不需要额外存储维护。

X509客户端证书认证方式，Kubernetes只验证客户端证书被集群的CA证书签名是否合法就可以了。一旦Kubernetes认证客户端证书合法，就会将证书中的“Common Name”作为user的名字，“Organization”作为user的group的名字。通过[RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)可以给user或者group设定相关的权限。下面是个ClusterRole配置信息的例子，这个`ClusterRole`只有所有Pods和Namespaces的只读权限：

```
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: read-only-user
rules:
- apiGroups:
  - ""
  resources:
  - pods
  - namespaces
  verbs:
  - get
  - list
  - watch
```

创建一个`ClusterRoleBinding`给指定user绑定这些权限。

```
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: read-only-users
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: read-only-user
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: brancz
```

上面`ClusterRoleBinding`将“brancz” 这个user绑定到了“read-only-user”这个`ClusterRole`上。

很简单了。我已经写了个脚本来自动完成这个过程。脚本先生成客户端证书，再用集群的CA证书对客户端证书进行签名，最后生成一个绑定的`kubeconfig`文件。这个文件可以直接被用来访问Kubernetes集群。最终文件在`clients/$USER/`目录下。（如果需要手动创建相关目录）

```
#!/usr/bin/env bash

if [[ "$#" -ne 2 ]]; then
    echo "Usage: $0 user group"
    exit 1
fi

USER=$1
GROUP=$2
CLUSTERENDPOINT=https://<apiserver-ip>:<apiserver-port>
CLUSTERNAME=your-kubernetes-cluster-name
CACERT=cluster/tls/ca.crt
CAKEY=cluster/tls/ca.key
CLIENTCERTKEY=clients/$USER/$USER.key
CLIENTCERTCSR=clients/$USER/$USER.csr
CLIENTCERTCRT=clients/$USER/$USER.crt

mkdir -p clients/$USER

openssl genrsa -out $CLIENTCERTKEY 4096
openssl req -new -key $CLIENTCERTKEY -out $CLIENTCERTCSR \
      -subj "/O=$GROUP/CN=$USER"
openssl x509 -req -days 365 -sha256 -in $CLIENTCERTCSR -CA $CACERT -CAkey $CAKEY -set_serial 2 -out $CLIENTCERTCRT

cat <<-EOF > clients/$USER/kubeconfig
apiVersion: v1
kind: Config
preferences:
  colors: true
current-context: $CLUSTERNAME
clusters:
- name: $CLUSTERNAME
  cluster:
    server: $CLUSTERENDPOINT
    certificate-authority-data: $(cat $CACERT | base64 --wrap=0)
contexts:
- context:
    cluster: $CLUSTERNAME
    user: $USER
  name: $CLUSTERNAME
users:
- name: $USER
  user:
    client-certificate-data: $(cat $CLIENTCERTCRT | base64 --wrap=0)
    client-key-data: $(cat $CLIENTCERTKEY | base64 --wrap=0)
EOF
```

> 备注: 这个脚本是写在Fedora 26上运行的，可能会在其他平台上有小的不兼容。如果CA证书文件路径不同，确保适配正确的路径。本博客内容工作在Kubernetes 1.7.x+版本上。

有任何问题可以通过[twitter](https://twitter.com/fredbrancz)联系作者。

> 译者解释：  
> 脚本和配置在kubernete 1.13.1版本验证OK。

