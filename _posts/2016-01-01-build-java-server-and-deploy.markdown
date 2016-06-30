---
layout: post
title:  "java服务器环境搭建及部署(centos版)"
date:   2015-11-02 23:29:25
categories: technology
author: tank
---

## 1.前期准备
先确定一下云服务器的系统

```shell
[root@Test-WindChat /]# cat /etc/redhat-release
CentOS release 6.5 (Final)
```

http://www.bkjia.com/Linuxjc/1128322.html

这里CentOS 6.5 和7及以上版本一些命令还是有区别的，比如6.5没有systemctl。。。。
用到的软件都在这里 http://10.0.1.231:11231/linux/

```shell
ssh-copy-id root@10.0.1.249
```

ssh-copy-id只是帮你创建authorized_keys,并把你的公钥放进去，就是这里的东西 ~/.ssh/id_rsa.pub

```shell
brew install ssh-copy-id

ssh-copy-id root@10.0.1.249
```

登进去之后，我们可以创建权限低一点的用户，用来之后的部署

```shell
sudo adduser deploy
passwd deploy
```

然后把deploy放入sudo组里

ubuntu:
adduser deploy sudo
(add user 'deploy' to group 'sudo')
ubuntu的话这里比较方便

centos:
[root@Test-WindChat home]# gpasswd -a deploy sudo
Adding user deploy to group sudo

visudo
visudo是跟vi /etc/sudoers一样

```shell
把
## Allow root to run any commands anywhere
root    ALL=(ALL)       ALL

改成
## Allow root to run any commands anywhere
root    ALL=(ALL)       ALL
deploy  ALL=(ALL)       ALL
```

deploy is not in the sudoers file.  This incident will be reported.
如果发现用户创建的不对可以这样撤回

```shell
sudo userdel -r deploy
sudo rm -rf /home/deploy

su deploy
```

测试加入了sudo组

```shell
sudo ls

ssh-copy-id deploy@10.0.1.249

ssh deploy@10.0.1.249
```

现在就可以这样ssh到服务器，不过要管理很多服务器的话可以用一些更方便的一键进入方法

## 2.安装jdk:
http://blog.csdn.net/sonnet123/article/details/9169741

www.cnblogs.com/always-online/p/4691507.html

```shell
ssh deploy@10.0.1.249

cd /tmp/

sudo yum -y install wget
```

debian 或者 ubuntu : sudo apt-get install wget
centos : sudo yum -y install wget

```shell
cd /tmp/

wget http://10.0.1.231:11231/linux/jdk/jdk-7u79-linux-x64.rpm
```

#### 复制到/usr/Java/路径下

```shell
sudo mkdir /usr/java/
sudo cp jdk-7u79-linux-x64.rpm /usr/java/
```

#### 添加可执行权限，并安装

```shell
cd /usr/java/
sudo chmod +x jdk-7u79-linux-x64.rpm
sudo rpm -ivh jdk-7u79-linux-x64.rpm
```

#### 配置环境变量

1.进入编辑profile文件

```shell
sudo vi /etc/profile
```

2.在profile文件最后追加入如下内容

```shell
export JAVA_HOME=/usr/java/jdk1.7.0_79
export CLASSPATH=.:$JAVA_HOME/jre/lib/rt.jar:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export PATH=$PATH:$JAVA_HOME/bin
```

3.保存并退出，执行如下

```shell
[deploy@Test-WindChat java]$ source /etc/profile
[deploy@Test-WindChat java]$ java -version
java version "1.7.0_79"
Java(TM) SE Runtime Environment (build 1.7.0_79-b15)
Java HotSpot(TM) 64-Bit Server VM (build 24.79-b02, mixed mode)
```

## 3.安装Tomcat

这里如果考虑到安全的话，可以创建无特权的tomcat用户，比如

```shell
sudo groupadd tomcat，
sudo useradd -M -s /bin/nologin -g tomcat -d /opt/tomcat tomcat
```

这样就把tomcat用户创建好了

https://www.digitalocean.com/community/tutorials/how-to-install-apache-tomcat-8-on-centos-7

#### 下载Tomcat

```shell
cd /tmp

wget http://10.0.1.231:11231/linux/tomcat/apache-tomcat-7.0.68.tar.gz

tar xzf apache-tomcat-7.0.68.tar.gz

mv apache-tomcat-7.0.68 /usr/local/tomcat7
```

我把tomcat放到/usr/local下

#### 开启Tomcat

```shell
cd /usr/local/tomcat7
./bin/startup.sh
```

如果是centos7或以上可以使用systemctl开关
这里如果要测试Tomcat是否工作，可以看http://10.0.1.231:8080，不过得配置一下云服务的安全策略，如果是linux安全策略的话，外网只有22端口是开着的，要想在外网开8080需要设置一下

http://tecadmin.net/steps-to-install-tomcat-server-on-centos-rhel/#


## 4.安装postgresql

#### 下载安装

```shell
cd /tmp

wget http://10.0.1.231:11231/linux/pgsql/pgdg-centos92-9.2-7.noarch.rpm

rpm -ivh pgdg-centos92-9.2-7.noarch.rpm

yum list postgres*
```

发现postgresql92-server.x86_64

```shell
yum install postgresql92-server
```

#### 初始化数据库环境

初始化环境

```shell
service postgresql-9.2 initdb
```

然后配置开机启动pgsql

```shell
chkconfig postgresql-9.2 on
```

开启

```shell
service postgresql-9.2 start
```

In centos 7+, 可以:

```shell
 systemctl enable postgresql
```

开启后可以在deploy用户下psql -U postgres
然后把项目需要的sql跑一跑


## 5.部署应用
这里先用比较傻瓜的gui的方式，放war包

修改tomcat conf/tomcat-users.xml，在文件里面加上下面的配置

```shell
<role rolename="manager-gui" />
<user username="manager" password="xxxxxx" roles="manager-gui" />

<role rolename="admin-gui" />
<user username="admin" password="xxxxxx" roles="manager-gui,admin-gui" />
```
重启Tomcat，就可以在gui界面里面登录上传WindChat.war文件了，有个deploy的按钮
然后查看一下这个日志 /usr/local/tomcat7/logs/catalina.out 这个跟idea控制台的日志是一样的
一开始看了带日期的日志, 看到的log很少，发现不了问题。。。

我们现在的项目是把数据库配置文件放在代码库里面，要想在本地和线上共用数据库配置一般有三种方式：
1.  spring profiles标签
2.  让项目在不同的环境加载不同的配置文件，http://www.dexcoder.com/selfly/article/3558
3.  dorker的方式

https://www.digitalocean.com/community/tutorials/how-to-install-and-use-postgresql-on-centos-6

接下来就是我在搭建数据库配合部署时遇到的问题，主要与pg_hba.conf相关

```shell
[deploy@VM_114_114_centos tmp]$ psql -U postgres
psql: FATAL:  Peer authentication failed for user "postgres"

sudo su - postgres

sudo service postgresql-9.2 stop
```

https://www.digitalocean.com/community/tutorials/how-to-install-and-use-postgresql-on-centos-7

```shell
INFO: validateJarFile(/usr/local/tomcat7/webapps/WindChat/WEB-INF/lib/javax.servlet-api-3.0.1.jar) - jar not loaded. See Servlet Spec 3.0, section 10.7.2. Offending class: javax/servlet/Servlet.class

scp jdk-7u79-linux-x64.rpm deploy@115.159.222.13:/tmp
```

```shell
[deploy@VM_114_114_centos lib]$ ls
annotations-api.jar  catalina.jar   jasper.jar       tomcat-coyote.jar   tomcat-i18n-ja.jar websocket-api.jar
catalina-ant.jar     ecj-4.4.2.jar  jsp-api.jar      tomcat-dbcp.jar   tomcat-jdbc.jar
catalina-ha.jar      el-api.jar     servlet-api.jar  tomcat-i18n-es.jar  tomcat-util.jar
catalina-tribes.jar  jasper-el.jar  tomcat-api.jar   tomcat-i18n-fr.jar  tomcat7-websocket.jar
```

```shell
[deploy@VM_114_114_centos logs]$ psql -U postgres
psql: FATAL:  Peer authentication failed for user "postgres"

find / -name pg_hba.conf

[root@VM_114_114_centos bin]# service postgresql-9.2 restart
https://www.digitalocean.com/community/tutorials/how-to-install-and-use-postgresql-on-centos-6
```

遇到问题：
com.alibaba.druid.pool.DruidDataSource.init datasource error, url: jdbc:postgresql://localhost:5432/windchat?charSet=utf-8(643)
org.postgresql.util.PSQLException: FATAL: Ident authentication failed for user "postgres"

```shell
把
# "local" is for Unix domain socket connections only
local   all             all                                     peer
# IPv4 local connections:
host    all             all             127.0.0.1/32            ident
# IPv6 local connections:
host    all             all             ::1/128                 ident
# Allow replication connections from localhost, by a user with the
# replication privilege.
#local   replication     postgres                                peer
#host    replication     postgres        127.0.0.1/32            ident
#host    replication     postgres        ::1/128                 ident

改成
# Database administrative login by Unix domain socket
local   all             postgres                                trust

# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     trust
# IPv4 local connections:
host    all             all             127.0.0.1/32            ident
# IPv6 local connections:
host    all             all             ::1/128                 ident
# Allow replication connections from localhost, by a user with the
# replication privilege.
#local   replication     postgres                                peer
#host    replication     postgres        127.0.0.1/32            ident
#host    replication     postgres        ::1/128                 ident

之后把
host    all             all             127.0.0.1/32            ident
host    all             all             ::1/128                 ident

改成
host    all             all             127.0.0.1/32            md5
host    all             all             ::1/128                 md5
```

遇到问题：
[INFO][2016-06-30 10:45:28,155]org.springframework.beans.factory.config.PropertiesFactoryBean.Loading properties file from class path resource [pgsql.properties](172)
[ERROR][2016-06-30 10:45:28,532]com.alibaba.druid.pool.DruidDataSource.init datasource error, url: jdbc:postgresql://127.0.0.1:5432/windchat?charSet=utf-8(643)
org.postgresql.util.PSQLException: The server requested password-based authentication, but no password was provided.

http://stackoverflow.com/questions/18664074/getting-error-peer-authentication-failed-for-user-postgres-when-trying-to-ge


```shell
最后是这个样子
# Database administrative login by Unix domain socket
local   all             postgres                                trust

# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     trust
# IPv4 local connections:
host    all             all             127.0.0.1/32            trust
# IPv6 local connections:
host    all             all             ::1/128                 md5
# Allow replication connections from localhost, by a user with the
# replication privilege.
#local   replication     postgres                                peer
#host    replication     postgres        127.0.0.1/32            ident
#host    replication     postgres        ::1/128                 ident
```

遇到问题
[INFO][2016-06-30 10:57:51,266]org.springframework.beans.factory.xml.XmlBeanDefinitionReader.Loading XML bean definitions from URL [jar:file:/usr/local/tomcat7/webapps/WindChat/WEB-INF/lib/WindService-1.0-SNAPSHOT.jar!/config/spring-application-service.xml](317)
Exception in thread "http-bio-8080-exec-55"
Exception: java.lang.OutOfMemoryError thrown from the UncaughtExceptionHandler in thread "http-bio-8080-exec-55"

```shell
[deploy@VM_114_114_centos bin]$ pwd
/usr/local/tomcat7/bin
[deploy@VM_114_114_centos bin]$ vi catalina.bat
```

加上这一行

```shell
set JAVA_OPTS= -Xms1024M -Xmx1024M -XX:PermSize=256M -XX:MaxNewSize=256M -XX:MaxPermSize=256M
```

解释一下各个参数：

-Xms1024M：初始化堆内存大小（注意，不加M的话单位是KB）
-Xmx1029M：最大堆内存大小
-XX:PermSize=256M：初始化类加载内存池大小
-XX:MaxPermSize=256M：最大类加载内存池大小

然后结果这样：
[INFO][2016-06-30 11:19:46,083]org.springframework.web.servlet.DispatcherServlet.FrameworkServlet 'SpringMVC': initialization completed in 2489 ms(503)

那接下来如果使用自己写的api呢？
参考http://blog.csdn.net/cangchen/article/details/44492753

```shell
vi /usr/local/tomcat7/conf/server.xml
```

在<Host>下加

```shell
<Context path="/" docBase="WindChat.war" debug="0" privileged="true" reloadable="true"/>
```

类似的api就可以跑了，可以测试
http://115.159.222.13:8080/WindChat/user/57591993eed0931dcf282c8c/friendship/list

## 6.安装nginx

## 7.自动化部署

未完待续

参考链接：

https://www.digitalocean.com/community/tutorials/how-to-add-and-delete-users-on-ubuntu-12-04-and-centos-6

https://www.digitalocean.com/community/tutorials/how-to-edit-the-sudoers-file-on-ubuntu-and-centos

http://shuyangyang.blog.51cto.com/1685768/1040127

http://blog.csdn.net/cangchen/article/details/44492753

http://blog.csdn.net/sonnet123/article/details/9169741

http://tecadmin.net/steps-to-install-tomcat-server-on-centos-rhel/#

https://www.digitalocean.com/community/tutorials/how-to-install-and-use-postgresql-on-centos-6
