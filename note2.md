## zabbix构建博客第二篇
### 之需要调教的parameter们


### 前言

前面预告过的，当zabbix的监视对象多的时候，如果用默认值的话，zabbix是监视不了那么多的。立马就会死掉。我在这个案件中整理一点需要调节的parameter，
有兴趣的可以参考一下
包括Zabbix AP，Proxy，MySQL

### parameter

|cloumn|parmater|默认值|推荐值|说明|
|:---:|:---:|:---:|:---:|:---:|
|MySQL|innodb_buffer_pool_size|-|32GB||
|MySQL|innodb_log_file_size|50MB|268MB||
|MySQL|character_set_server|utf8|utf8|防止日志监视时候的乱码|
|MySQL|skip_character_set_client_handshake|off|on||
|MySQL|innodb_file_per_table|off|on|防止DISK容量不足|
|MySQL|binlog_expire_logs_seconds|2592000|86400|binlogの保存時間(单位是秒)|
|Zabbix AP/Proxy|HistoryCacheSize|16M|2G|文字类型的监视数据使用的Cache，如果监视取得数据速度超过DB写入速度，那就会导致Cache溢出| 
|zabbix AP/Proxy|CacheSize|64M|2G|如上，不过这是数值类型的数据使用的Cache|
|zabbix AP/Proxy|HistoryIndexCacheSize|4M|2G|| 
|zabbix AP/Proxy|UnreachableDelay|15s|300s|无法监视的项目的重试时间，如果无法监视项目太多，重试时间过短会导致队列拥挤|
|zabbix AP/Proxy|StartPollersUnreachable|1|10|重试的进程数|
|zabbix AP|TrendCacheSize|4M|2G|Trand数据使用的Cache|
|zabbix AP|ValueCacheSize|8M|20G|全部数据使用的Cache值|
|zabbix AP|StartPreprocessors|3|10|数据进行加工处理的进程数|
|zabbix AP|StatPollers|5|20|取得监视数据的进程数|
|zabbix AP|StartPingers|1|10|ping监视的进程数|
|zabbix AP|StartDBSyncers|4|20|写入数据库的进程数|
|zabbix AP|MaxHousekeeperDelete|5000|50000|HouseKepper一次删除最大的行数（防止数据库膨大化，但是如果一次删除过多会导致数据库负担增加影响监视）|



