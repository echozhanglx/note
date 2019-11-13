## Zabbix构建博客第一篇
### 之如何构建Zabbix

### 目录
1. 前言
2. infrastructure的构造简介
3. 搭建zabbix AP服务器
4. 搭建zabbix proxy服务器
5. 总结
6. 预告


### 1.前言

历时半年的项目主要是为客户搭建一个可以大规模监视zabbix。最后的交货的时候监视的个数如下：
- 监视对象（hosts）：4000台
- 监视项目（items）：40万
在搭建的过程中，经历过无数次监视延迟，因为数据太多导致zabbix不工作。所以，为了不让自己忘记当时是如何整理的，也算是复习笔记，撑着项目之间的空白期整理一下。

### 2.infrastructure的构造简介

![infrastructure](/img/infra.PNG) 

如上图所示，此次infrastructure的设计有以下特点
- zabbix的AP服务器和所使用的数据库，使用Azure的服务
- 并且使用Azure Site Recovery实现DR(灾害对应）
- 因为客户的环境有很多服务器和Network机器在本地，所以在本地搭建了一个zabbix proxy服务器，一是为了减少和Azure之间的通信次数，二是为了给Zabbix AP减少负担
- 为了保证监视服务器的高可用性，zabbix的AP和Proxy全部使用Pacemaker和corosync搭建成HA构成（Azure和本地的failover方式不同，在后面会介绍）

### 3.搭建zabbixAP服务器

系统要件如下：
- OS： Red Hat Enterprise Linux 7.6
- Zabbix: zabbix 4.0LTS
- MySQL: 5.7.21(Azure Database for MySQL支持的最高版本…)
- Apache: 2.4.39
- PHP: 7.3  

安装zabbix AP服务器其实很简单，首先要安装一些middleware，因为这次用的小红帽（red Hat），所用就祭出了yum大法了（yum大法好啊）  
1.yum install httpd (安装apache）  
2.yum install php php-gd php-bcmath php-mbstring php-mysqlnd php-xml (安装PHP）  
3.因为Database是使用azure的PaaS，所以就省略咯，因为是PaaS所以很简单，看Azure的文档就好了  
4.最后，当然要把讨厌的SELunix给无效掉  
  - vi /etc/selinux/config 
  - SELINUX=enforcing  
  
5.PS:因为后面还要构建HA，所以httpd就不用设置成自动启动了  

安装完上面的middleware之后，就要安装zabbix了，依然是yum大法好  
1.rpm -Uvh https://repo.zabbix.com/zabbix/4.0/rhel/7/x86_64/zabbix-release-4.0-1.el7.noarch.rpm  
2.yum install zabbix-server-mysql  
3.yum install zabbix-web-mysql  
zabbix安装好，就要初始database，  
1.链接DataBase服务器 ex Mysql -h(database host) -u(databse id) -p
2.新建zabbix用的数据库 create database zabbix；
3.初始化DB zcat /usr/share/doc/zabbix-server-mysql*/create.sql.gz | mysql -h -u(你的DB账号）-p zabbix
之后编辑zabbix的设定文件，在这里就先把数据库的信息写入
1.vi /etc/zabbix/zabbix_server.conf
 - DBHost=
 - DBName=
 - DBPassword=  
 
全部弄好之后呢，就可以启动咯  
 - systemctl start httpd
 - systemctl start zabbix-server  
 
启动完后，就用浏览器访问上面搭建的zabbix AP服务器，应该就能看到初始见面了，按照上的流程，各种下一步就可以了，这里就省略了。

### 4.搭建Zabbix Proxy服务器

zabbix proxy的工作原理就是，把监视对象的监视数据全部汇总后，有proxy负责发送给zabbix AP服务器。  
zabbix proxy服务器存在的意义就是，上面也说了，最重要的一点就是可以分担zabbix AP服务器的负担。因为当没有Proxy的时候，所有的监视数据的取得，判断全部都是有zabbix AP一个人来干。当数据多的时候，就不能在短时间处理完，而导致监视发生延迟。第二点就是，在一个相对隔离的环境中，为了减少频繁跟外界存在的zabbix AP服务器进行通信的时候，也可以在相同的环境中搭建一个proxy，可惜把监视数据先汇总到proxy服务器，再有proxy服务器发送给zabbix AP服务器。
系统要件如下：
- OS： Red Hat Enterprise Linux 7.6
- Zabbix: zabbix Proxy 4.0LTS
- MySQL: 7.2  

由于zabbix proxy不需要database以外的middleware，所以只安装了MySQL，而且还因为zabbix proxy的数据库，一般在把数据发送给zabbix AP以后，就会清空数据库，所以就没有设计数据库的备份啥的
- yum -y install http://dev.mysql.com/get/mysql80-community-release-el7-2.noarch.rpm
- yum install mysql-community-server
- systemctl start mysqld.service
- cat /var/log/mysqld.log|grep password （取得初始密码）
- mysql_secure_installation （MySQL设置） 

安装完后数据库，就要安装proxy了
- rpm -ivh https://repo.zabbix.com/zabbix/4.0/rhel/7/x86_64/zabbix-release-4.0-1.el7.noarch.rpm
- yum install zabbix-proxy

然后就是就是编辑设置文件了
- vi /etc/zabbix/zabbix-proxy.conf
1. 设置Proxy的模式
   - ProxyMode （推荐使用1，proxyMode=1，被动模式passive mode，也就是zabbix AP主动过来拿数据，0的话是主动模式active mode,不管ZabbixAP死活，数据送过去就完了）
2. 登录zabbix AP服务器的IP（DNS）
   - Server= （注意：当ProxyMode为被动模式的时候，没有写在这里的IP都会被静止访问，所以当zabbix AP服务器有复数IP可能的时候，请全部写上用逗号隔开）
3. 登录Proxy的名字
   - HostName = (注意：这里要和zabbix AP画面上登录的Proxy的名字要一样，这一块会在后面登录）
4. 登录Proxy使用的DataBase
   - DBHost=
   - DBName=
   - DBPassword=
  
之后就可以启动咯（因为后面要弄HA，所以不需要让他们自动启动）
- systemctl start zabbix-proxy

然后登录zabbix的UI画面，进入设置，Proxy，登录就好啦。登录的时候，记得选择passive mode，然后登录proxy服务器的IP，Proxy的名字要是和上面写入设定文件的HostName要一致，不然没法工作。

## 总结

这样的话，一个带有Proxy的Zabbix服务器就搭建好了。其实zabbix的搭建还是很简单，没什么难度。当然这只是最低限的搭建，明天我会介绍大规模监视的时候需要调教哪些parameter
如果想查看log啥的，zabbix AP的log在/var/log/zabbix/zabbix-server.log
zabbix Proxy的在 /var/log/zabbix/zabbix-proxy.log
当他们工作异常的时候，请查看日志就好。

## 预告

大规模监视的时候，需要调教那些parameter！

