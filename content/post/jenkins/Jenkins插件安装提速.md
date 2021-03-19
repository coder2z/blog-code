+++
tags = ["jenkins"]
categories = ["jenkins"]
description = ""
menu = ""
banner = ""
images = []
title = "Jenkins插件安装提速"
date = "2020-06-23T14:46:27+08:00"
+++

# 操作步骤
配置Json其实在Jenkins的工作目录中
```cmd
cd {你的Jenkins工作目录}/updates  #进入更新配置位置
```
## 第一种方式：使用vim
```cmd
vim default.json
```
使用vim的命令，如下，替换所有插件下载的url
```cmd
:1,$s/http:\/\/updates.jenkins-ci.org\/download/https:\/\/mirrors.tuna.tsinghua.edu.cn\/jenkins/g
```
替换连接测试url
```cmd
:1,$s/http:\/\/www.google.com/https:\/\/www.baidu.com/g
```
    进入vim先输入：然后再粘贴上边的：后边的命令，注意不要写两个冒号！

修改完成保存退出 
```cmd
:wq
```
## 第二种方式：使用sed
在updates目录下
```cmd
sed -i 's/http:\/\/updates.jenkins-ci.org\/download/https:\/\/mirrors.tuna.tsinghua.edu.cn\/jenkins/g' default.json && sed -i 's/http:\/\/www.google.com/https:\/\/www.baidu.com/g' default.json
```
    这是直接修改的配置文件，如果前边Jenkins用sudo启动的话，那么这里的两个sed前均需要加上sudo

重启Jenkins，简直超速！！
