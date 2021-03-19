---
title: "Hexo 2 Hugo"
date: 2021-03-19T13:40:39+08:00
Description: ""
Tags: ["hugo","blog"]
Categories: ["blog"]
DisableComments: false
url: /post/blog/hexo-to-hugo/
---

# 序

很久没更新我的博客了~~ 
不是我懒，主要还是: ~~*&&(**^()&)..~~

![](https://oss.myxy99.cn/images/2021/03/20210319140004.jpg)

刚好给我朋友看到了我的博客，就被吐槽了....之前的博客的确是有点花里胡哨的，都感觉不到是一个技术类博客。23333....就准备给博客换个主题。

之前使用的是hexo进行搭建的，随着博客文章的越来越多，hexo的编译已经有一点力不从心哦，太慢了。之前使用hexo编译30篇md文件至少都需要10s时间。实在受不了...之前很久就发现了这个问题，但是一直都不想折腾，这次就趁着换主题，给操作了。把hexo换成hugo，毕竟作为一个Gopher🐭还是需要支持一下咱们golang的工具的...

![](https://oss.myxy99.cn/images/2021/03/20210319141852.jpg)

# 1.Hugo的安装以及基础使用

## 1.1 安装Hugo

我这里直接去hugo的github仓库下载对应系统的二进制文件：

[https://github.com/gohugoio/hugo/releases](https://github.com/gohugoio/hugo/releases)

然后把二进制文件放在环境变量中，由于之前配置过**GOPATH**的**bin**到环境变量中，所有这里我直接把二进制放在**bin**文件夹下面。

![](https://oss.myxy99.cn/images/2021/03/20210319142747.png)

验证安装

```shell
λ hugo version
hugo v0.81.0-59D15C97 windows/amd64 BuildDate=2021-02-19T17:07:12Z VendorInfo=gohugoio
```

## 1.2 创建项目

创建一个blog为博客项目

```shell
hugo new site blog && cd blog
```

## 1.3 配置主题

找了半天主题，这里打算使用:[https://github.com/lxndrblz/anatole/](https://github.com/lxndrblz/anatole/)

```shell
# 切换到博客目录
cd blog
git init
git submodule add https://github.com/lxndrblz/anatole.git themes/
echo 'theme = "anatole"' >> config.toml 
```

## 1.4 编写Blog

生成一篇blog模板

```shell
hugo new post/my-first-post.md
```

## 1.5 启动server

```shell
# 启动 server 预览
hugo server
```

这个时候在浏览器输入 **http://localhost:1313** 即可查看效果

## 1.6 编译静态文件

```
hugo -D
```

执行过后就会在目录的public下生成对应的静态文件，只需要部署到自己的web服务器即可。

# 2.hexo to hugo

由于之前使用的hexo写的博客，hexo与hugo对于md文件的格式要求是不一样的。

![](https://oss.myxy99.cn/images/2021/03/20210319144635.png)

可以看到，左边是hugo需要的md文件格式，右边是hexo需要的md文件格式。

所以这里就需要把hexo格式的文件批量转为hugo格式的文件。

这里我选择的方案是通过python+正则对文件进行处理。在github上搜索了一下，再开源项目的基础上进行了修改。最后的py源码如下：

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-

import os
import re
import yaml
import pytoml as toml
from datetime import datetime
import argparse
import sys
import importlib

importlib.reload(sys)                      # reload 才能调用 setdefaultencoding 方法  
# sys.setdefaultencoding('utf-8')  # 设置 'utf-8'  


content_regex = re.compile(r'\-\-\-*([\s\S]*?)\-\-\-*([\s\S]*)')
temp_regex = re.compile(r'\+\+\+*([\s\S]*?)\+\+\+*([\s\S]*)')



def write_out_file(front_data, body_text, out_file_path):
    out_lines = ['+++']
    front_string = toml.dumps(front_data)
    out_lines.extend(front_string.splitlines())
    out_lines.append('+++')
    out_lines.extend(body_text.splitlines())

    with open(out_file_path, 'w',encoding="utf-8") as f:

        f.write('\n'.join(out_lines))

def convert_file(file_path,out_dir,file_temp=None):
    filename = os.path.basename(file_path)
    temp_content =''
    m_head={}
    if file_temp:
        with open(file_temp,'r',encoding="utf-8") as f:
            temp_content = f.read()
        m_temp = temp_regex.match(temp_content)
        m_head = toml.loads(m_temp.group(1))


    content = ''
    with open(file_path,'r',encoding="utf-8") as f :
        content = f.read()

    m = content_regex.match(content)
    if not m:
        print('Error match content: %s' % file_path)
        return False

    print(m.group(1),"************",file_path)

    front_data = yaml.load(m.group(1), Loader=yaml.FullLoader)
    if not front_data:
        print('Error load yaml: %s' % file_path)
        return False

    m_head['title']= front_data.get('title','')

    if front_data.get("description",None):
        m_head['description']=front_data.get("description",None)

    if front_data.get('date',None):
        #time.strftime("%Y-%m-%dT%H:%M:%S+08:00",b)
        m_head['date']=front_data.get('date',None).strftime('%Y-%m-%dT%H:%M:%S+08:00')

    m_tags = front_data.get('tags',None)
    if m_tags:
        print(type(m_tags),m_tags)
        m_head['tags']=[]
        if isinstance(m_tags,list):
            m_head['tags'].extend(m_tags)
        else:
            m_head['tags'].append(m_tags)
        print(m_head['tags'])

    m_categories = front_data.get('categories',None)
    if m_categories:
        m_head['categories']=[]
        if isinstance(m_categories,list):
            m_head['categories'].extend(m_categories)
        else:
            m_head['categories'].append(m_categories)
        print(m_head['categories'])
    
    write_out_file(m_head,m.group(2),out_dir+'/'+filename)

    return True 

def convert_post(file_path, out_dir):
    filename = os.path.basename(file_path)

    print(filename)
    content = ''
    with open(file_path, 'r',encoding="utf-8") as f:
        content = f.read()

    m = content_regex.match(content)
    if not m:
        print('Error match content: %s' % file_path)
        return False

    front_data = yaml.load(m.group(1), Loader=yaml.FullLoader)
    if not front_data:
        print('Error load yaml: %s' % file_path)
        return False

    '''
    保留 title tags  data categories 
    '''
    print(front_data)
    data = {}
    data['title']= front_data.get('title','')
    if front_data.get('date',None):
        #time.strftime("%Y-%m-%dT%H:%M:%S+08:00",b)
        data['date']=front_data.get('date',None).strftime('%Y-%m-%dT%H:%M:%S+08:00')


    data['tags']=front_data.get('tags',None)
    if front_data.get('categories',None):
        data['categories']=front_data.get('categories',None)

    write_out_file(data,m.group(2),out_dir+'/'+filename)

    return True


def convert(src_dir, out_dir,file_temp=None):
    count = 0
    error = 0
    for root, dirs, files in os.walk(src_dir):
        for filename in files:
            try:
                if os.path.splitext(filename)[1] != '.md' or filename in ['README.md', 'LICENSE.md']:
                    continue
                file_path = os.path.join(root, filename)
                common_prefix = os.path.commonprefix([src_dir, file_path])
                rel_path = os.path.relpath(
                    os.path.dirname(file_path), common_prefix)
                real_out_dir = os.path.join(out_dir, rel_path)
                print(file_path, real_out_dir)
                if convert_file(file_path, real_out_dir,file_temp):
                    print('Converted: %s' % file_path)
                    count += 1
                else:
                    error += 1
            except Exception as e:
                error += 1
                print('Error convert: %s \nException: %s' % (file_path, e))

    print('--------\n%d file converted! %s' % (count, 'Error count: %d' % error if error > 0 else 'Congratulation!!!'))


if __name__ == '__main__':
    parser = argparse.ArgumentParser(description='Convert Hexo blog to Hugo')
    parser.add_argument('src_dir', help='hexo _posts dir')
    parser.add_argument('out_dir', help='hugo root path')
    parser.add_argument('file_temp', help='hugo theme default templete')
    
    args = parser.parse_args()
    print(os.path.abspath(args.src_dir))
    

    convert(os.path.abspath(args.src_dir),
        os.path.abspath(args.out_dir),
        os.path.abspath(args.file_temp))
```

替换模板：

```shell
vim default.md

+++
tags = []
categories = ["Notes"]
description = ""
menu = ""
banner = ""
images = []
+++
```

使用python进行批量修改：

```shell
# 安装拓展
pip install pytoml
pip install yaml 

# 生成hugo md
python Hexo2Hugo.py hexo-post-dir output-dir default.md
```

# 3.hugo 自动部署

每次写好了博客都需要手动部署，简直太麻烦。这里就打算使用Github的Action进行自动化部署。整个自动化流程如下：

![](https://oss.myxy99.cn/images/2021/03/20210319150627.png)

## 3.1 编译

### 3.1.1 脚本
创建文件:
```shell
vim .github/workflows/main.yml
```

写入脚本内容：
```yml
name: Deploy Hugo Site to Github Pages on Master Branch

on:
  push:
    branches:
      - master

jobs:
  build-deploy:
    runs-on: ubuntu-18.04
    steps:
      - uses: actions/checkout@v1  # v2 does not have submodules option now
       # with:
       #   submodules: true

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: '0.81.0'
          extended: true

      - name: Build
        run: hugo --minify

      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        with:
          deploy_key: ${{ secrets.ACTIONS_DEPLOY_KEY }} # 这里的 ACTIONS_DEPLOY_KEY 则是上面设置 Private Key的变量名
          external_repository: coder2z/coder2z.github.io # Pages 远程仓库 
          publish_dir: "./public" #传输文件目录
          keep_files: false # remove existing files
          publish_branch: master  # 分支
          commit_message: ${{ github.event.head_commit.message }} #commit message
          exclude_assets: '' #反正排除.github文件夹
```

### 3.1.2 配置密钥

* 使用git生成**ssh key**(相当于生成对密钥)
    ```shell
    ssh-keygen -t rsa -b 4096 -C "$(git config user.email)" -f gh-pages -N ""
    # You will get 2 files:
    #   gh-pages.pub (public key)
    #   gh-pages     (private key)
    ```
* 打开**blog-code**仓库的**settings**，再点击**Secrets**，然后添加刚刚生成的私钥，name为**ACTIONS_DEPLOY_KEY**
    ![](https://oss.myxy99.cn/images/2021/03/20210319151549.png)
* 同理打开**coder2z.github.io**仓库的**settings**，再点击**Deploy keys**，**Allow write access**一定要勾上，否则会无法提交
    ![](https://oss.myxy99.cn/images/2021/03/20210319151646.png)


## 3.2 部署

### 3.1.1 脚本
创建文件:
```shell
vim static/.github/workflows/main.yml
```

写入脚本内容：
```yml
name: CI

on:
  push:
    branches: [ master ]
  pull_request:
    branches: [ master ]

  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Run a multi-line script
        uses: fifsky/ssh-action@master
        with:
          command: |
            cd /www/wwwroot/myxy99.github.io    #自己服务器的项目地址
            git reset --hard
            git pull    #同步
          host: ${{ secrets.REMOTE_HOST }} #服务器ip
          user: ${{ secrets.REMOTE_USER }} #服务器user
          pass: ${{ secrets.PASSWORD }} #服务器密码
```

### 3.1.2 配置密钥
打开**coder2z.github.io**仓库的**settings**，再点击**Secrets**，然后一共添加三个私钥，name分别为**REMOTE_HOST**,**REMOTE_USER**,**PASSWORD**

![](https://oss.myxy99.cn/images/2021/03/20210319152558.png)

# 至此，本次hexo转hugo完美完事

一路上还是碰到了 好多坑，但是也还行...谁叫我这么优秀...

![](https://oss.myxy99.cn/images/2021/03/20210319152844.gif)


