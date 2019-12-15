---
layout:     post
title:      centos7 编译mysql5.1 操作记录
subtitle:   低配置vps运行mysql必备
date:       2018-10-03
author:     Belin
header-img: img/post-bg-re-vs-ng2.jpg
catalog: true
tags:
    - Mysql
    - Linux
    - 编译
---

> 正所谓前人栽树，后人乘凉。
>
> 感谢[Huxpro](https://github.com/huxpro)提供的博客模板

### 系统版本
```
uname -r
4.18.11-1.el7.elrepo.x86_644.18.11-1.el7.elrepo.x86_64
```
### 获取源码
或者这里下载
![mysql-5.1.72.zip](/files/mysql-5.1.72.zip)
```shell
wget http://ftp.jaist.ac.jp/pub/mysql/Downloads/MySQL-5.1/mysql-5.1.57.tar.gz
mkdir -p /usr/local/mysql1/data 
mkdir -p /usr/local/mysql1/tmp
tar -zxvf mysql-5.1.57.tar.gz
```
### 安装依赖
```shell
yum list|grep ncurses
yum -y install ncurses-devel

./configure --prefix=/usr/local/mysql1 --localstatedir=/usr/local/mysql1/data --with-mysqld-user=mysql --with-charset=utf8 --with-unix-socket-path=/usr/local/mysql1/tmp/mysql.sock 
make
make install
```
5.修改权限和创建配置文件 
```shell
cp support-files/my-medium.cnf /usr/local/mysql1/my.cnf
```
备注： 根据机器配置的不同选择不同的文件：
```shell
//最小配置安装，内存<=64M，数据数量最少
/user/local/mysql/share/mysql/my-small.cnf
//内存=512M 
/user/local/mysql/share/mysql/my-large.cnf 
//32M<内存<64M，或者内存有128M，但是数据库与web服务器公用内存
/user/local/mysql/share/mysql/my-medium.cnf
//1G<内存<2G
/user/local/mysql/share/mysql/my-huge.cnf
//最大配置安装，内存至少4G 
/user/local/mysql/share/mysql/my-innodb-heavy-4G.cnf  
```
编辑配置文件 my.conf
```
[mysqld] 
basedir = /usr/local/mysql1              定义mysql程序目录 
datadir = /usr/local/mysql1/data         定义数据目录 
```
并修改端口，各个mysql使用不同的端口 

### 设置权限 
```shell
chown mysql:mysql /usr/local/mysql1/my.cnf 
chown -R mysql:mysql /usr/local/mysql1 
```
初始化mysql(用户表和权限表)和启动mysql 
### 进入mysql的安装目录、初始化数据库
```shell
cd /usr/local/mysql1 

bin/mysql_install_db --basedir=/usr/local/mysql1 --datadir=/usr/local/mysql1/data --user=mysql 
```

### 启动mysql 
```shell
bin/mysqld_safe --defaults-file=/usr/local/mysql1/my.cnf & 
```
备注：&代表后台运行 
停止mysql可以使用：bin/mysqladmin shutdown -u root -p 

### 设置root用户的密码 
```shell
bin/mysqladmin -u root password '123456' 
```
备注：如果出现error: 'Can't connect to local MySQL server through socket '/tmp/mysql.sock' (2)'解决办法： 
```shell
ln -s /usr/local/mysql1/tmp/mysql.sock /tmp/mysql.sock 
chmod 755 /tmp/mysql.sock 
```

### 登录mysql 
```
bin/mysql -u root -p  
```
### 设置mysql开机自动启动 
```shell
//进入解压后的源码目录 
cd mysql-5.1.57 
//将mysql.server这个文件copy到/etc/init.d/目录下，并更名为mysql1 
cp support-files/mysql.server /etc/init.d/mysql1  
//给mysql1这个文件赋予可执行的权限 
chmod 755 /etc/init.d/mysql1 
//加入到开机自动运行 
chkconfig --add mysql1 
chkconfig --level 345 mysql1 on 
//重新启动mysql 
service mysql1 restart 
```

### 检查是否正常，程序是否已经运行，端口是否打开 
```shell
ps -ef | grep mysql 
netstat -an | grep'3306' 
```
### 配置环境变量
为了直接调用mysql，需要将mysql的bin目录加入PATH环境变量。
编辑/etc/profile文件：
```shell
sudo vim /etc/profile
//在文件最后 添加如下两行：
PATH=$PATH:/usr/local/mysql1/bin
export PATH
//关闭文件，运行下面的命令，让配置立即生效：
source /etc/profile
```
### 导入数据库
```shell
mysql -uroot -p easyblog < abc.sql
```