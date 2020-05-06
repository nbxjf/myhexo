---
title: Linux环境配置
date: 2017-09-11 21:19:00
categories: Linux
tags:
 - 环境配置 
 - ubuntu
---

本文包括以下内容：

* jdk 1.8安装
* mysql安装及远程访问配置
* ftp配置

<!-- more -->

# Java
网上的教程多是基于下载jdk然后上传至服务器安装的，那么如何在线安装jdk呢？

* oracle官方未提供jdk镜像地址，默认apt-get是无法安装jdk8的，需要额外设置jdk镜像源

    ```
    sudo apt-get install software-properties-common
    sudo add-apt-repository ppa:webupd8team/java
    sudo apt-get update
    sudo apt-get install -y oracle-java8-installer
    sudo apt-get install -y oracle-java8-set-default
    sudo update-alternatives --config java
    
    ```
    <br/>
* 参考：
    * [添加Oracle镜像](https://askubuntu.com/questions/593433/error-sudo-add-apt-repository-command-not-found)
    * [安装](https://www.digitalocean.com/community/tutorials/how-to-install-java-on-ubuntu-with-apt-get)

# Mysql

* 在线安装

    ```
    sudo apt-get install mysql-client
    apt install mysql-client
    apt install libmysqlclient-dev
    ```
    * 期间需两次输入root密码
* 查看是否安装成功

    ```
    netstat -tap | grep mysql
    ```
* 开启远程访问
    * 注释`bind-address = 127.0.0.1`
    
        ```
        vi /etc/mysql/mysql.conf.d/mysqld.cnf
    ```
    * 进入账户

    ```
    mysql -u root -p
    grant all on *.* to XXX@'%' identified by 'XXX';
    flush privileges;
    exit;
    /etc/init.d/mysql restart
    ```
  <br/>
  
    * 其他
        * 不建议修改root账号，所以新建账号并且设置可访问的库
        * 重启后即可访问
        * 阿里云不能访问的话，需要再配置安全规则里的入规则，开启3306端口

# Ftp

* 安装ftp

    ```
    apt-get install vsftpd -y
 apt-get install ftp
    ```
* 修改配置文件

    ```
    vi /etc/vsftpd/vsftpd.conf
    ```
    修改如下内容：
    
    ```
local_enable=YES 
write_enable=YES 
chroot_local_user=YES 
    ```
* 开启

    ```
    service vsftpd start
    ```
* 连接ftp

    ```
    sftp root@ip
    ```

# Git
* 安装git

    ```
    sudo apt-get install git
    ```

# Maven
* 安装maven

    ```
    sudo apt-get install maven
    ```
* 查看版本

    ```
    mvn -version
    ```

# 打包工程

```
mvn install
```

