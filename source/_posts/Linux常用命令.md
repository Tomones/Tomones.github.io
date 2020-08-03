---
title: Linux常用命令
date: 2020-05-30 23:17:11
tags: Linux
categories: 
- 运维
---

![image-20200720231737484](http://cdn.andiycc.com/blog_img/image-20200720231737484.png)

### keytool操作JDK密钥库

```shell
#导入证书
keytool -import -v -trustcacerts -alias dandian -file /application/IDP/dandian.cer  -storepass  changeit  -keystore %JAVA_HOME%/jre/lib/security/cacerts

#查询证书库
keytool -list -keystore cacerts 

#删除证书
keytool -delete -alias dandian -keystore cacerts 
```

### 服务器远程复制文件

```shell
#scp 当前要复制的文件名 远程主机上的用户名@远程主机IP：要复制到远程主机上哪个文件夹中

scp adserver.cer root@10.213.70.53:/application/IDP/apache-tomcat-8.0.50

# -r 递归整个待复制的目录
scp -r /application/IDP root@10.213.70.53:/application/IDP/apache-tomcat-8.0.50
```

### 卸载Linux自带OpenJDk
```shell
# 查看java版本
[root@CFDB2 ~]# java -version
openjdk version "1.8.0_171"
OpenJDK Runtime Environment (build 1.8.0_171-b10)
OpenJDK 64-Bit Server VM (build 25.171-b10, mixed mode)

# 查看java安装软件
[root@CFDB2 ~]# rpm -qa|grep java
tzdata-java-2018e-3.el7.noarch
java-1.8.0-openjdk-1.8.0.171-8.b10.el7_5.x86_64
java-1.7.0-openjdk-headless-1.7.0.181-2.6.14.8.el7_5.x86_64
java-1.7.0-openjdk-1.7.0.181-2.6.14.8.el7_5.x86_64
javapackages-tools-3.4.1-11.el7.noarch
python-javapackages-3.4.1-11.el7.noarch
java-1.8.0-openjdk-headless-1.8.0.171-8.b10.el7_5.x86_64

# 卸载openjdk
rpm -e --nodeps java-1.8.0-openjdk-headless-1.8.0.101-3.b13.el7_2.x86_64
```

### 配置JDK8环境变量

```shell
# vi /etc/profile
export JAVA_HOME=/usr/share/jdk1.6.0_14
export PATH=$JAVA_HOME/bin:$PATH
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
# source /etc/profile
```

### 查询Linux服务器出网IP
```shell
# ip_addr 当前服务器的出网ip
[root@idp10-server1-local webapps]# curl ifconfig.me/all.xml
<info>
    <ip_addr>xxx.xx.xx.xx</ip_addr>
    <remote_host>unavailable</remote_host>
    <user_agent>curl/7.29.0</user_agent>
    <port>27058</port>
    <language></language>
    <referer></referer>
    <connection></connection>
    <keep_alive></keep_alive>
    <method>GET</method>
    <encoding></encoding>
    <mime>*/*</mime>
    <charset></charset>
    <via>1.1 google</via>
    <forwarded>xxx.xx.xx.xx, xxx.xxx.xx.xx</forwarded>
```
