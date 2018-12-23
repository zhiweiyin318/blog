---
author: "Zhiwei Yin"
date: 2018-09-14
lastmod: 2018-09-14
draft: false
title: "Python jinja2"
description: ""
tags: [
    "python",
    "code",
    "jinja2"
]
categories: [
    "python"
]
---
# jinja2是啥

python的模板引擎，也是Flask的模板引擎，能根据模板字符串替换，有自己的语法规则。  
官网: <http://docs.jinkan.org/docs/jinja2/>

# 使用

## 安装

```
$ pip install jinja2
```

## 语法定义

jinja的语法很灵活，也很丰富，下面就列举几个常用到的，详细见官网文档介绍。

* {% ... %} 语句（Statements）
* {{ ... }} 打印模板输出的表达式（Expressions）
* {# ... #} 注释
* #... ## 行语句（Line Statements）

### 变量

除了普通的字符串变量，Jinja2还支持列表、字典和对象。

```
{{ mydict['key'] }}
{{ mylist[3] }}
{{ mylist[myintvar] }}
{{ myobj.somemethod() }}
```

获取一个变量的属性的两种方式：

```
{{ foo.bar }}
{{ foo['bar'] }}
```

### 控制语句

* if  

```
{% if xxx %}
   xxx
{% elif %}
    xxx
{% else %}
    xxx
{% endif %}
```
* for  

```
{% for x in xx %}
  xxx
{% endfor %}
```

### 空白控制

默认配置中，不会对空白做进一步修改，每个空白（空格，制表符，换行符等等）都会原封不动的返回，注释也会占一行。  
可以在注释，for，if或者变量表达式的开始或者结束放一个'-'，可以移除块前或者后的空白。  
标签和'-'之间不能有空格

```
{% for item in [1,2,3,4,5,6,7,8,9]] -%}
    {{ item }}
{%- endfor %}

# 输出结果123456789
```

### 程序

```
import jinja2
# 传递的参数
values = {'base_distro': 'centos',
              'base_distro_tag': '7',
              'base_image': 'centos',
              'namespace': 'test',
              'tag': 'latest',
              'maintainer': 'zhiwei yin',
              'version': 'v1.0.0',
              'image_name': 'demo',
              'build_date': datetime.datetime.fromtimestamp(time.time()).strftime('%Y%m%d')}
    
    # 模板放置目录
    path = './template'
    
    # 创建一个loader，jinja2会从这个path加载文件
    # 用loader创建一个环境，有了它才能读取模板
    env = jinja2.Environment(loader=jinja2.FileSystemLoader(path))
    
    #加载模板并返回
    template = env.get_template('Dockerfile.j2')
    
    # 完成渲染模板
    content = template.render(values)
```

### demo
demo 地址：<https://github.com/zhiweiyin318/yzw.python.demo/tree/master/jinja2>

模板：
```
{# This is a dome file #}
FROM {{ base_image }}:{{ base_distro_tag }}
LABEL maintainer="{{ maintainer }}" name="{{ image_name }}" build-date="{{ build_date }}"

LABEL version="{{ version }}"

{% if base_image in ['ubuntu', 'debian'] %}
    {% set work_dir = 'ubuntu' %}
{% else %}
    {% set work_dir = 'centos' %}
{% endif %}

WORKDIR /{{ work_dir }}


{% for file in ['a','b'] %}
COPY {{ file }} /{{ work_dir }}
{% endfor %}

RUN{% for item in ['yum','install','python', '-y'] %} {{ item }}
{%- endfor %}
```

渲染后的文件：
```

FROM centos:7
LABEL maintainer="zhiwei yin" name="demo" build-date="20180914"

LABEL version="v1.0.0"


    


WORKDIR /centos



COPY a /centos

COPY b /centos


RUN yum install python -y
```