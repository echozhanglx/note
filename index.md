## Zabbix构建博客第一篇

## 之如何构建可以大规模监视的Zabbix

### 目录
1. 前言
2. infrastructure的构造简介
3. 搭建zabbix AP服务器
4. 搭建zabbix proxy服务器
5. 监视设计


### 前言

历时半年的项目主要是为客户搭建一个可以大规模监视zabbix。最后的交货的时候监视的个数如下：
- 监视对象（hosts）：4000台
- 监视项目（items）：40万
在搭建的过程中，经历过无数次监视延迟，因为数据太多导致zabbix不工作。所以，为了不让自己忘记当时是如何整理的，也算是复习笔记，撑着项目之间的空白期整理一下。

### infrastructure的构造简介

![infrastructure](/img/infra.PNG) 

如上图所示，此次infrastructure的设计有以下特点
- zabbix的AP服务器和所使用的数据库，使用Azure的服务
- 并且使用Azure Site Recovery实现DR(灾害对应）
- 因为客户的环境有很多服务器和Network机器在本地，所以在本地搭建了一个zabbix proxy服务器，一是为了减少和Azure之间的通信次数，二是为了给Zabbix AP减少负担
- 为了保证监视服务器的高可用性，zabbix的AP和Proxy全部使用Pacemaker和corosync搭建成HA构成（Azure和本地的failover方式不同，在后面会介绍）


