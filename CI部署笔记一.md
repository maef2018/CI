CI部署笔记一

说明：本文档为正确部署一整套CI平台完整记录，相对简洁部分，只把命令粘贴出来并加以说明；

date:20181029

部署内容：mysql、openldap(docker)、gitlab(docker)、gerrit、jenkins(docker)

部署环境：ubuntu18.04

服务器：32G内存+300G系统盘+1T数据盘

容器ip：

```
172.17.0.2      7d5d42cfabee openldap 
172.17.0.3      460b4e9b7b9c phpldapadmin
172.17.0.4      32832cabbcbd gitlab
172.17.0.6      51a98103eeee jenkins
```



# 1.mysql

## 1.1部署规划

使用源码安装mysql5.7.23   端口：3306

```
配置文件：/etc/my.cnf
数据目录:/r2/mysqldata
安装目录：/usr/local/mysql
```

## 1.2下载mysql-boost

```
wget https://cdn.mysql.com//Downloads/MySQL-5.7/mysql-boost-5.7.23.tar.gz
```

## 1.3安装工具

```
sudo apt-get install cmake -y
apt-get install gcc g++ -y
apt-get install git -y
apt-get install libncurses5 libncurses5-dev -y
sudo apt-get install -y build-essential
sudo apt-get install flex bison -y
sudo apt-get install mpi-default-dev libicu-dev python-dev libbz2-dev -y
```

## 1.4下载并安装boost

```
sudo wget https://nchc.dl.sourceforge.net/project/boost/boost/1.59.0/boost_1_59_0.tar.gz
```

### 1.4.1解压

```
tar -zxvf boost_1_59_0.tar.gz
```

### 1.4.2编译

```
cd /r2/soft/boost_1_59_0 && ./bootstrap.sh
```

### 1.4.3b2

```
cd /r2/soft/boost_1_59_0 && ./b2 -a -sHAVE_ICU=1 # the parameter means that it support icu or unicode #执行时间稍微长
```

### 1.4.4安装

```
cd /r2/soft/boost_1_59_0 &&  sudo ./b2 install
```

这时就把包给装到/usr/local/include/boost目录下了

## 1.5.创建目录

```
cd /r2 && mkdir mysqldata
cd /usr/local && mkdir mysql
```

## 1.6.预编译msyql

```
cd /r2/soft/mysql-5.7.23 && sudo cmake \
-DCMAKE_INSTALL_PREFIX=/usr/local/mysql \
-DMYSQL_DATADIR=/r2/mysqldata \
-DSYSCONFDIR=/etc \
-DWITH_MYISAM_STORAGE_ENGINE=1 \
-DWITH_INNOBASE_STORAGE_ENGINE=1 \
-DWITH_MEMORY_STORAGE_ENGINE=1 \
-DWITH_READLINE=1 \
-DMYSQL_UNIX_ADDR=/r2/mysqldata/mysql.sock \
-DMYSQL_TCP_PORT=3306 \
-DENABLED_LOCAL_INFILE=1 \
-DWITH_PARTITION_STORAGE_ENGINE=1 \
-DEXTRA_CHARSETS=all \
-DDEFAULT_CHARSET=utf8 \
-DDEFAULT_COLLATION=utf8_general_ci \
-DENABLE_DOWNLOADS=1
```

注意：如果出现错误，再次编译的时候需要删除CMakeCache.txt文件

## 1.7.编译并安装

### 1.7.1编译

```
cd /r2/soft/mysql-5.7.23 && make -j4
```

### 1.7.2安装

```
cd /r2/soft/mysql-5.7.23 && make install
```

## 1.8.新建mysql组和用户并授权

```
groupadd mysql
useradd -g mysql mysql
mkdir /usr/local/mysql/data
chown -R mysql /usr/local/mysql
chgrp -R mysql /usr/local/mysql
chown -R mysql /r2/mysqldata
chgrp -R mysql /r2/mysqldata
```

## 1.9.添加mysql的环境变量

```
vim /etc/profile
PATH=/usr/local/mysql/bin:/usr/local/mysql/lib:$PATH
export PATH
让其生效source /etc/profile
```

## 1.10.初始化

```
/usr/local/mysql/bin/mysqld --initialize --user=mysql --basedir=/usr/local/mysql --datadir=/r2/mysqldata

这里可以看到初始化密码2018-10-04T09:36:02.519395Z 1 [Note] A temporary password is generated for root@localhost: p&Lta1fdsD,j
这个就是初始化密码：p&Lta1fdsD,j
```

## 1.11.优化便捷化操作

```
cp /usr/local/mysql/support-files/mysql.server /etc/init.d/mysqld
chmod +x /etc/init.d/mysqld
update-rc.d mysqld defaults
```

## 1.12.启动mysql

```
service mysqld start
```

## 1.13.登陆并创建账号

```
mysql -uroot -p 之后输入密码p&Lta1fdsD,j

set password=password("test123456");

grant insert,update,select,delete,drop,create,index,trigger,alter on *.* to maef@'%' identified by 'test123456';
grant insert,update,select,delete,drop,create,index,trigger,alter on *.* to maef@'localhost' identified by 'test123456';
flush privileges;
```

## 1.14.安装客户端

```
apt install mysql-client-core-5.7 -y
apt install mariadb-client-core-10.1 -y
```

注意：这时默认的配置什么都没有，位置/etc/mysql/my.cnf

## 1.15.编写脚本

启动脚本start.sh

```
#!/bin/bash
/usr/local/mysql/bin/mysqld_safe --defaults-file=/etc/my.cnf --user=mysql --datadir=/r2/mysqldata &
```

停止脚本：

```
#!/bin/bash
/usr/local/mysql/bin/mysqladmin -uroot -p shutdown &
```

## 1.16.优化配置

执行停掉数据库之后vim /etc/my.cnf之后执行启动./start.sh即可正常使用

## 1.17.my.cnf附录如下

```
本配置文件主要适用于MySQL 5.7.23版本

******************************************

[client]
port    = 3306
default-character-set = utf8
socket  = /r2/mysqldata/mysql.sock
=======================================================================

MySQL客户端配置

=======================================================================
[mysql]
prompt="(\u@\h) \\R:\\m:\\s [\d]> "
no-auto-rehash
default-character-set = utf8
=======================================================================

MySQL服务器全局配置

=======================================================================
[mysqld]
user = mysql
port = 3306
server-id = 1
basedir = /usr/local/mysql
tmpdir = /r2/mysqldata
datadir = /r2/mysqldata
socket  = /r2/mysqldata/mysql.sock
log-error=/r2/mysqldata/error.log
wait_timeout = 31536000
sql_mode =                             #sql_mode 配置为空值
skip_name_resolve           #开启dns解析
lower_case_table_names = 0
character-set-server = utf8
collation-server = utf8_unicode_ci
log_timestamps = SYSTEM
init_connect= 'SET NAMES utf8'
max_allowed_packet = 128M
---------------------------- 性能参数 --------------------------------
open_files_limit = 10240
max_connections = 10000
max_user_connections=9990
max_connect_errors = 100000
table_open_cache = 1024
thread_cache_size = 64
max_heap_table_size = 32M
query_cache_type = 0
----------------global cache------------------------
key_buffer_size = 64M
query_cache_size = 0
tmp_table_size = 32M        #内存临时表
binlog_cache_size = 4M      #二进制日志缓冲
-----------------session cache--------------------------
sort_buffer_size = 8M       #排序缓冲
join_buffer_size = 4M       #表连接缓冲
read_buffer_size = 8M       #顺序读缓冲
read_rnd_buffer_size = 8M   #随机读缓冲
thread_stack = 256KB        #线程的堆栈的大小
------------------------------ binlog设置 ------------------------------
binlog_format = MIXED
log_bin = /r2/mysqldata/binlog
max_binlog_size = 1G
expire_logs_days = 3       #binlog比较占空间，注意磁盘空间
sync_binlog = 0             #重要参数必须修改为0
--------------------------------- 复制设置 --------------------------
log_slave_updates = 1

-------GTID 配置--------

gtid_mode=ON
enforce-gtid-consistency=true
------------------------- innodb--------------------
default_storage_engine = InnoDB
innodb_buffer_pool_size = 128M         #系统内存50%
innodb_open_files = 5120
innodb_flush_log_at_trx_commit = 2    #线上服务器必须配置为2
innodb_file_per_table = 1
innodb_lock_wait_timeout = 5
innodb_io_capacity = 200              #根据您的服务器IOPS能力适当调整innodb_io_capacity，配SSD盘可调整到 10000 - 20000
innodb_io_capacity_max = 20000
innodb_flush_method = O_DIRECT
innodb_log_file_size = 2G
innodb_log_files_in_group = 2
innodb_large_prefix = 0
innodb_thread_concurrency = 64
innodb_strict_mode = OFF
innodb_sort_buffer_size = 4194304
---------------------- log 设置 -------------------------
log_error = /r2/mysqldata/error.log
slow_query_log = 1
long_query_time = 10
slow_query_log_file = /r2/mysqldata/slow.log
=======================================================================

MySQL mysqldump配置

=======================================================================
[mysqldump]
quick
max_allowed_packet = 128M
=======================================================================

MySQL mysqld_safe配置

=======================================================================
[mysqld_safe]
log_error = /r2/mysqldata/error.log
pid_file = /r2/mysqldata/mysqldb.pid
```



# 2.openldap

## 2.1部署规划

### 2.1.1端口映射

端口映射：389:389(明文）636:636(密文）   本次部署使用明文，密文方式有待后续研究

### 2.1.2目录挂载

挂载目录：/ci_volume/openldap_volume

实际存放位置/r2/ci_volume/openldap_volume；通过软连接到根目录下

```
ln -s /r2/ci_volume/openldap_volume /ci_volume/openldap_volume
```

### 2.1.3ldap账号密码

```
账号:root   明文密码：wallet20181027 密文：{SSHA}fh5KyMpmQWw8C5hlgZ0M+SdjV0ILhI4M
```

### 2.1.4域名规划

域名：wallet.com

### 2.1.5用户组和id规划

```
                                        wallet.com
cn=root(uidnum=10000)      ou=group                       ou=person
                           cn=WalletTeamDev(6000)         uid=admin--wallet20181027---gid8000--1000
                           cn=WalletTeamPub(7000)         uid=jenkins--wallet123-----gid6000--1001
                           cn=WalletTeamOps(gid8000)      uid=maef---wallet123-----gid8000---1002
cn=root是超级管理员，管理ldap，其uid为10000
uid值从1000开始到5000之间
gid的值从5000开始
```

### 2.1.6用户目录存放位置

```
/ci_volume/openldap_volume/slapd/database
```



## 2.2镜像获取

```
docker pull osixia/openldap
```

## 2.3构建容器

```
docker run -dit -p 389:389 -p 636:636 -p 18080:8080 --restart always --name openldap -v /ci_volume/openldap_volume/slapd/database:/var/lib/ldap -v /ci_volume/openldap_volume/slapd/config:/etc/ldap/slapd.d -v /etc/localtime:/etc/localtime:ro osixia/openldap
```

说明：这里的端口映射18080:8080为多余的，可以去掉

-v /etc/localtime:/etc/localtime:ro 这里是为了让容器使用宿主机的时间，保持容器和宿主机时间一致

## 2.4确认member模块

```
cd /ci_volume/openldap_volume/slapd/config/cn=config && cat cn\=module\{0\}.ldif 
# AUTO-GENERATED FILE - DO NOT EDIT!! Use ldapmodify.
# CRC32 091b451a
dn: cn=module{0}
objectClass: olcModuleList
cn: module{0}
olcModulePath: /usr/lib/ldap
olcModuleLoad: {0}back_mdb
olcModuleLoad: {1}memberof  #有member模块则可直接使用，如果没有则需要手动添加
olcModuleLoad: {2}refint
structuralObjectClass: olcModuleList
entryUUID: 667491f6-6e05-1038-86ad-d1abaf869c1c
creatorsName: cn=admin,cn=config
createTimestamp: 20181027072644Z
entryCSN: 20181027072645.745770Z#000000#000#000000
modifiersName: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
modifyTimestamp: 20181027072645Z
```

说明：上面是使用docker镜像构建，如果是直接安装到宿主机上，通常需要手动配置member模块；

## 2.5目录查询

```
cd /ci_volume/openldap_volume/slapd/config/cn=config && ll
total 40
drwxr-x--- 4 999 docker 4096 Oct 27 15:26  ./
drwxr-xr-x 3 999 docker 4096 Oct 27 15:26  ../
-rw------- 1 999 docker  543 Oct 27 15:26 'cn=module{0}.ldif'
drwxr-x--- 2 999 docker 4096 Oct 27 15:26 'cn=schema'/
-rw------- 1 999 docker  396 Oct 27 15:26 'cn=schema.ldif'
-rw------- 1 999 docker  414 Oct 27 15:26 'olcBackend={0}mdb.ldif'
-rw------- 1 999 docker  657 Oct 27 15:26 'olcDatabase={-1}frontend.ldif'
-rw------- 1 999 docker  654 Oct 27 15:26 'olcDatabase={0}config.ldif'
drwxr-x--- 2 999 docker 4096 Oct 27 15:26 'olcDatabase={1}mdb'/
-rw------- 1 999 docker 1066 Oct 27 15:26 'olcDatabase={1}mdb.ldif'
```

## 2.6schema导入

schema是添加数据的基础，用到的schema需要先制作之后并导入，常用的在安装时已经存在，个别需要手动导入，无需手动制作，只有未提供的才需要手动制作

### 2.6.1理论需要导入模块

```
#docker exec -it openldap bash
#cd /etc/ldap/schema && ls -l |grep ldif  
-rw-r--r-- 1 openldap openldap  2036 May 23 12:25 collective.ldif
-rw-r--r-- 1 openldap openldap  1845 May 23 12:25 corba.ldif
-rw-r--r-- 1 openldap openldap 21196 May 23 12:25 core.ldif
-rw-r--r-- 1 openldap openldap 12006 May 23 12:25 cosine.ldif
-rw-r--r-- 1 openldap openldap  4842 May 23 12:25 duaconf.ldif
-rw-r--r-- 1 openldap openldap  3330 May 23 12:25 dyngroup.ldif
-rw-r--r-- 1 openldap openldap  3481 May 23 12:25 inetorgperson.ldif
-rw-r--r-- 1 openldap openldap  2979 May 23 12:25 java.ldif
-rw-r--r-- 1 openldap openldap  2082 May 23 12:25 misc.ldif
-rw-r--r-- 1 openldap openldap  6809 May 23 12:25 nis.ldif
-rw-r--r-- 1 openldap openldap  3308 May 23 12:25 openldap.ldif
-rw-r--r-- 1 openldap openldap  6904 May 23 12:25 pmi.ldif
-rw-r--r-- 1 openldap openldap  4570 May 23 12:25 ppolicy.ldif
```

### 2.6.2已导入schema

有一部分已经导入完成，如下所示

```
cd /ci_volume/openldap_volume/slapd/config/cn=config/cn=schema && ll
total 140
drwxr-x--- 2 999 docker  4096 Oct 27 15:26  ./
drwxr-x--- 4 999 docker  4096 Oct 27 15:26  ../
-rw------- 1 999 docker 15596 Oct 27 15:26 'cn={0}core.ldif'
-rw------- 1 999 docker  1216 Oct 27 15:26 'cn={10}quota.ldif'
-rw------- 1 999 docker 13243 Oct 27 15:26 'cn={11}radius.ldif'
-rw------- 1 999 docker 11960 Oct 27 15:26 'cn={12}samba.ldif'
-rw------- 1 999 docker 10033 Oct 27 15:26 'cn={13}zarafa.ldif'
-rw------- 1 999 docker 11381 Oct 27 15:26 'cn={1}cosine.ldif'
-rw------- 1 999 docker  6513 Oct 27 15:26 'cn={2}nis.ldif'
-rw------- 1 999 docker  2875 Oct 27 15:26 'cn={3}inetorgperson.ldif'
-rw------- 1 999 docker  3935 Oct 27 15:26 'cn={4}ppolicy.ldif'
-rw------- 1 999 docker 22773 Oct 27 15:26 'cn={5}dhcp.ldif'
-rw------- 1 999 docker  5953 Oct 27 15:26 'cn={6}dnszone.ldif'
-rw------- 1 999 docker  3530 Oct 27 15:26 'cn={7}mail.ldif'
-rw------- 1 999 docker  1304 Oct 27 15:26 'cn={8}mmc.ldif'
-rw------- 1 999 docker   834 Oct 27 15:26 'cn={9}openssh-lpk.ldif'
```

### 2.6.3实际导入schema

通过上面对比可以看出只有如下需要手动导入

```
#doceker exec -it openldap bash #需要进入到容器内部进行导入
#ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/ldap/schema/collective.ldif
#ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/ldap/schema/corba.ldif
#ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/ldap/schema/duaconf.ldif
#ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/ldap/schema/dyngroup.ldif
#ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/ldap/schema/java.ldif
#ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/ldap/schema/openldap.ldif
#ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/ldap/schema/pmi.ldif
```

### 2.6.4确认导入模块

```
cd /ci_volume/openldap_volume/slapd/config/cn=config/cn=schema && ll
total 176
drwxr-x--- 2 999 docker  4096 Oct 27 15:40  ./
drwxr-x--- 4 999 docker  4096 Oct 27 15:26  ../
-rw------- 1 999 docker 15596 Oct 27 15:26 'cn={0}core.ldif'
-rw------- 1 999 docker  1216 Oct 27 15:26 'cn={10}quota.ldif'
-rw------- 1 999 docker 13243 Oct 27 15:26 'cn={11}radius.ldif'
-rw------- 1 999 docker 11960 Oct 27 15:26 'cn={12}samba.ldif'
-rw------- 1 999 docker 10033 Oct 27 15:26 'cn={13}zarafa.ldif'
-rw------- 1 999 docker  1615 Oct 27 15:40 'cn={14}collective.ldif'
-rw------- 1 999 docker  1377 Oct 27 15:40 'cn={15}corba.ldif'
-rw------- 1 999 docker  4583 Oct 27 15:40 'cn={16}duaconf.ldif'
-rw------- 1 999 docker  1787 Oct 27 15:40 'cn={17}dyngroup.ldif'
-rw------- 1 999 docker  2683 Oct 27 15:40 'cn={18}java.ldif'
-rw------- 1 999 docker  1384 Oct 27 15:40 'cn={19}openldap.ldif'
-rw------- 1 999 docker 11381 Oct 27 15:26 'cn={1}cosine.ldif'
-rw------- 1 999 docker  6538 Oct 27 15:40 'cn={20}pmi.ldif'
-rw------- 1 999 docker  6513 Oct 27 15:26 'cn={2}nis.ldif'
-rw------- 1 999 docker  2875 Oct 27 15:26 'cn={3}inetorgperson.ldif'
-rw------- 1 999 docker  3935 Oct 27 15:26 'cn={4}ppolicy.ldif'
-rw------- 1 999 docker 22773 Oct 27 15:26 'cn={5}dhcp.ldif'
-rw------- 1 999 docker  5953 Oct 27 15:26 'cn={6}dnszone.ldif'
-rw------- 1 999 docker  3530 Oct 27 15:26 'cn={7}mail.ldif'
-rw------- 1 999 docker  1304 Oct 27 15:26 'cn={8}mmc.ldif'
-rw------- 1 999 docker   834 Oct 27 15:26 'cn={9}openssh-lpk.ldif'
```

## 2.7ldap管理员密码设置

### 2.7.1查看账号密码

原本openldap会有一个默认账号为admin和密码，有时刚部署好是没有默认密码的，即使有默认密码，我们不知道，所以我们需要重新设定账号密码

```
cd /ci_volume/openldap_volume/slapd/config/cn=config && cat olcDatabase={0}config.ldif
# AUTO-GENERATED FILE - DO NOT EDIT!! Use ldapmodify.
# CRC32 d4f5e52a
dn: olcDatabase={0}config
objectClass: olcDatabaseConfig
olcDatabase: {0}config
olcAccess: {0}to * by dn.exact=gidNumber=0+uidNumber=0,cn=peercred,cn=extern
 al,cn=auth manage by * break
olcRootDN: cn=admin,cn=config
structuralObjectClass: olcDatabaseConfig
entryUUID: 66740420-6e05-1038-86a7-d1abaf869c1c
creatorsName: cn=config
createTimestamp: 20181027072644Z
olcRootPW:: e1NTSEF9T0pTMFpGQ1RKeEpPVERrVFBWRVlRZ3JEQlkxTUIyeW0=
entryCSN: 20181027072645.637845Z#000000#000#000000
modifiersName: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
modifyTimestamp: 20181027072645Z
```

### 2.7.2密码修改

```
将root的密码修改为明文wallet20181027 密文：{SSHA}fh5KyMpmQWw8C5hlgZ0M+SdjV0ILhI4M
```

```
cat << EOF | ldapadd -Y EXTERNAL -H ldapi:///
dn: olcDatabase={0}config,cn=config #这里要根据实际填写
changetype: modify
replace: olcRootPW
olcRootPW: {SSHA}fh5KyMpmQWw8C5hlgZ0M+SdjV0ILhI4M  #这里写的就是你需要修改密码的密文
EOF
```

## 2.8ldap管理员账号设定

```
cat << EOF | ldapadd -Y EXTERNAL -H ldapi:///
dn: olcDatabase={1}mdb,cn=config
changetype: modify
replace: olcRootDN
olcRootDN: cn=root,dc=wallet,dc=com
EOF

cat << EOF | ldapadd -Y EXTERNAL -H ldapi:///
dn: olcDatabase={1}mdb,cn=config
changetype: modify
replace: olcRootPW
olcRootPW: {SSHA}fh5KyMpmQWw8C5hlgZ0M+SdjV0ILhI4M  #这里要填写为上面设定密码的密文一致
EOF
注意上面的dn数据必须和本机的配置文件名称一样olcDatabase={0}config.ldif，要根据实际情况来
因为已经存在默认的密码了，所以这里修改的时候使用replace,如果没有默认数据这里需要将replace替换为add
文件执行方法类似ldapadd -Y EXTERNAL -H ldapi:// -f createpw.ldif
```

```
经验：如果上面修改密码的部分失败，可以尝试进行如下操作：
报错内容：
root@7d5d42cfabee:/etc/ldap/schema# cat << EOF | ldapadd -Y EXTERNAL -H ldapi:///
> dn: olcDatabase={1}mdb,cn=config
> changetype: modify
> replace: olcRootPW
> olcRootPW: {SSHA}fh5KyMpmQWw8C5hlgZ0M+SdjV0ILhI4M
> EOF
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
modifying entry "olcDatabase={1}mdb,cn=config"
ldap_modify: Other (e.g., implementation specific) error (80)
        additional info: <olcRootPW> can only be set when rootdn is under suffix
解决方案：
cat << EOF | ldapadd -Y EXTERNAL -H ldapi:///
dn: olcDatabase={0}config,cn=config
changetype: modify
replace: olcRootPW
olcRootPW: {SSHA}fh5KyMpmQWw8C5hlgZ0M+SdjV0ILhI4M
EOF


cat << EOF | ldapadd -Y EXTERNAL -H ldapi:///
dn: olcDatabase={1}mdb,cn=config
changetype: modify
replace: olcRootPW
olcRootPW: {SSHA}fh5KyMpmQWw8C5hlgZ0M+SdjV0ILhI4M
EOF
```

查看账号密码

```
/ci_volume/openldap_volume/slapd/config/cn=config# cat olcDatabase={1}mdb.ldif
# AUTO-GENERATED FILE - DO NOT EDIT!! Use ldapmodify.
# CRC32 9f61e55e
dn: olcDatabase={1}mdb
objectClass: olcDatabaseConfig
objectClass: olcMdbConfig
olcDatabase: {1}mdb
olcDbDirectory: /var/lib/ldap
olcLastMod: TRUE
olcDbCheckpoint: 512 30
olcDbMaxSize: 1073741824
structuralObjectClass: olcMdbConfig
entryUUID: 6674af60-6e05-1038-86af-d1abaf869c1c
creatorsName: cn=admin,cn=config
createTimestamp: 20181027072644Z
olcAccess: {0}to attrs=userPassword,shadowLastChange by self write by dn="cn
 =admin,dc=example,dc=org" write by anonymous auth by * none
olcAccess: {1}to * by self read by dn="cn=admin,dc=example,dc=org" write by 
 * none
olcDbIndex: uid eq
olcDbIndex: mail eq
olcDbIndex: memberOf eq
olcDbIndex: entryCSN eq
olcDbIndex: entryUUID eq
olcDbIndex: objectClass eq
olcRootDN: cn=root,dc=wallet,dc=com  #账号修改好
olcSuffix: dc=wallet,dc=com 
olcRootPW:: e1NTSEF9Zmg1S3lNcG1RV3c4QzVobGdaME0rU2RqVjBJTGhJNE0= #密码修改好
```

## 2.9ldap域名设定

```
cat << EOF | ldapadd -Y EXTERNAL -H ldapi:///
dn: olcDatabase={1}mdb,cn=config
changetype: modify
replace: olcSuffix
olcSuffix: dc=wallet,dc=com
EOF

注意：上面的dn必须根据实际情况找到自己的db文件，即带有db的ldif文件；

备注：这里没有olcDatabase={2}monitor.ldif文件，所以不用对其进行修改，如果有则需要对其进行下面的操作，这个是监控文件
cat << EOF | ldapadd -Y EXTERNAL -H ldapi:///
dn: olcDatabase={2}monitor,cn=config
changetype: modify
add: olcAccess
olcAccess: {0}to * by dn.base="gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth" read by dn.base="cn=root,dc=wallet,dc=com" read by * none
EOF
```

截止到目前为止完成了设置账号root，将域名修改为wallet.com，将root的密码设置完毕

## 2.10配置修改

将容器内部/etc/ldap/ldap.conf末尾添加如下配置之后重启docker restart openldap

```
BASE    dc=wallet,dc=com
URI     ldap://192.168.32.108
```

## 2.11添加组

```
cat << EOF | ldapadd -x -D  cn=root,dc=wallet,dc=com -wwallet20181027
dn: dc=wallet,dc=com
objectClass: dcObject
objectClass: organization
dc: wallet
o: wallet.com

dn: ou=people,dc=wallet,dc=com
objectClass: organizationalUnit
objectClass: top
ou: people

dn: ou=groups,dc=wallet,dc=com
objectClass: organizationalUnit
ou: groups

dn: cn=root,dc=wallet,dc=com
objectClass: organizationalRole
cn: root

dn: cn=WalletTeamDev,ou=groups,dc=wallet,dc=com
objectClass: posixGroup
cn: WalletTeamDev
gidNumber: 6000

dn: cn=WalletTeamPub,ou=groups,dc=wallet,dc=com
objectClass: posixGroup
cn: WalletTeamPub
gidNumber: 7000

dn: cn=WalletTeamOps,ou=groups,dc=wallet,dc=com
objectClass: posixGroup
cn: WalletTeamOps
gidNumber: 8000
EOF
```

## 2.12授权用户权限

```
需要在openldap-server中做授权,让所有用户可以自己修改密码，下面需要根据实际情况进行填写
cat << EOF | ldapmodify -Y EXTERNAL -H ldapi:///
dn: olcDatabase={1}mdb,cn=config
changetype: modify
add: olcAccess
olcAccess: {0}to attrs=userPassword,shadowLastChange by
  dn="cn=root,dc=wallet,dc=com" write by anonymous auth by self write by * none
olcAccess: {1}to dn.base="" by * read
olcAccess: {2}to * by dn="cn=root,dc=wallet,dc=com" write by * read
EOF
```

## 2.13add users

添加账号的时候邮箱和用户名  uid都不能冲突

### 2.13.1add admin

wallet2018

```
cat << EOF | ldapadd -D "cn=root,dc=wallet,dc=com" -h 192.168.32.108 -x -wwallet20181027
dn: uid=admin,ou=people,dc=wallet,dc=com
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
homeDirectory: /ci_volume/openldap_volume/users/admin
loginShell: /bin/bash
cn: admin
uidNumber: 1000
gidNumber: 8000
sn: System Administrator
mail: 3118064263@qq.com
postalAddress: zs
givenName: admin
displayName: admin
gecos: system User
uid: admin
userPassword: {SSHA}wyJhFZ77yqZ90V6brODE8hhzssqkXWcr
EOF
```

### 2.13.2add jenkins

wallet123

```
cat << EOF | ldapadd -D "cn=root,dc=wallet,dc=com" -h 192.168.32.108 -x -wwallet20181027
dn: uid=jenkins,ou=people,dc=wallet,dc=com
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
homeDirectory: /ci_volume/openldap_volume/users/jenkins
loginShell: /bin/bash
cn: jenkins
uidNumber: 1001
gidNumber: 8000
sn: System Administrator
mail: 3377379238@qq.com
postalAddress: zs
givenName: jenkins
displayName: jenkins
gecos: system User
uid: jenkins
userPassword: {SSHA}puMjW99AwlS6ryPt8Ob+a0KhQyFn8xug
EOF
```

### 2.13.3add maef

maef123

```
cat << EOF | ldapadd -D "cn=root,dc=wallet,dc=com" -h 192.168.32.108 -x -wwallet20181027
dn: uid=maef,ou=people,dc=wallet,dc=com
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
homeDirectory: /ci_volume/openldap_volume/users/maef
loginShell: /bin/bash
cn: maef
uidNumber: 1002
gidNumber: 8000
sn: System Administrator
mail: 1972341225@qq.com
postalAddress: zs
givenName: maef
displayName: maef
gecos: system User
uid: maef
userPassword: {SSHA}SofqD+WmFmcPN1whDCI8g+qLcfjPLiNY
EOF
```

### 2.13.4add libai

邮箱不要一样，如果邮箱一样会认为是同一个人

密码是123456

```
cat << EOF | ldapadd -D "cn=root,dc=wallet,dc=com" -h 192.168.32.108 -x -wwallet20181027
dn: uid=libai,ou=people,dc=wallet,dc=com
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
homeDirectory: /ci_volume/openldap_volume/users/libai
loginShell: /bin/bash
cn: libai
uidNumber: 1003
gidNumber: 8000
sn: System Administrator
mail: libai@qq.com
postalAddress: zs
givenName: libai
displayName: libai
gecos: system User
uid: libai
userPassword: {SSHA}XhmX2vHfLXxiLO/Lx/ZvdB0cttRWyhzB
EOF
```



# 3.部署phpldapadmin

## 3.1 phpldapadmin run

实际执行：  如果是443端口，所以访问使用https，如果是80端口则使用http方式,下面创建到最后不要ctrl+c停止，这样就自动停止了

```
docker run -p 13080:80 \
--name phpldapadmin \
--link openldap:openldap \
--env PHPLDAPADMIN_LDAP_HOSTS=openldap \
--env PHPLDAPADMIN_HTTPS=false \
osixia/phpldapadmin:latest
```

## 3.2phpldapadmin login

```
http://192.168.32.108:13080    
cn=root,dc=wallet,dc=com
wallet20181027
```

# 4.gitlab

## 4.1部署规划

实际使用的是/home/openwallet中的

端口映射:10100:10100 -p 10443:443 -p 10022:22

```
docker run -itd --name=gitlab \
-p 10100:10100 -p 10443:443 -p 10022:22 \
--restart always  \
--volume /home/openwallet/gitlab_volume/config:/etc/gitlab \
--volume /home/openwallet/gitlab_volume/logs:/var/log/gitlab \
--volume /home/openwallet/gitlab_volume/data:/var/opt/gitlab \
-v /etc/localtime:/etc/localtime:ro \
gitlab/gitlab-ce:latest 
```



## 4.2镜像获取

```
gitlab/gitlab-ce:latest
```

## 4.3构建容器#废除

```
docker run -itd --name=gitlab \
-p 10100:10100 -p 10443:443 -p 10022:22 \
--restart always \
--volume /ci_volume/gitlab_volume/config:/etc/gitlab \
--volume /ci_volume/gitlab_volume/logs:/var/log/gitlab \
--volume /ci_volume/gitlab_volume/data:/var/opt/gitlab \
-v /etc/localtime:/etc/localtime:ro \
gitlab/gitlab-ce:latest
```

## 4.4配置修改

修改配置实现：集成openldap、添加邮箱

配置修改后如下：

```
cd /r2/ci_volume/gitlab_volume/config && egrep -vE "^#|^$" gitlab.rb 
external_url 'http://192.168.32.108:10100'  #http访问地址
 gitlab_rails['ldap_enabled'] = true  #下面这些是集成openldap所需配置
 gitlab_rails['ldap_servers'] = YAML.load <<-'EOS'
   main: # 'main' is the GitLab 'provider ID' of this LDAP server
     label: 'LDAP'
     host: '192.168.32.108'
     port: 389
     uid: 'uid'
     bind_dn: 'cn=root,dc=wallet,dc=com'
     password: 'wallet20181027'
     encryption: 'plain' # "start_tls" or "simple_tls" or "plain"
     verify_certificates: true
     active_directory: false
     base: 'ou=people,dc=wallet,dc=com'
     user_filter: ''
 EOS
 gitlab_rails['gitlab_shell_ssh_port'] = 10022 #ssh端口
 gitlab_rails['smtp_enable'] = true #下面这些是邮箱信息
 gitlab_rails['smtp_address'] = "smtp.qq.com"
 gitlab_rails['smtp_port'] = 465
 gitlab_rails['smtp_user_name'] = "3377379238@qq.com"
 gitlab_rails['smtp_password'] = "jstrwoswyjxwdbea"
 gitlab_rails['smtp_domain'] = "qq.com"
 gitlab_rails['smtp_authentication'] = "login"
 gitlab_rails['smtp_enable_starttls_auto'] = true
 gitlab_rails['smtp_tls'] =  true
 gitlab_rails['gitlab_email_enabled'] = true
 gitlab_rails['gitlab_email_from'] = '3377379238@qq.com'
 gitlab_rails['gitlab_email_display_name'] = 'Gitlab'
 user['git_user_email'] = "3377379238@qq.com"
```

## 4.5登陆验证

登陆之前先重启docker restart gitlab

```
http://192.168.32.108:10100
首次登陆账号密码root：wallet20181027   
ldap的超级管理员是admin     wallet@2018
```

## 4.6关闭注册入口

超级管理员账号登陆

![1540631777706](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540631777706.png)

!![1540631889313](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540631889313.png)

![1540631919851](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540631919851.png)

## 4.7ldap login

特殊情况：超级管理员设置密码登陆之后从其gitlab之后才会显示ldap的登陆方式

docker restart gitlab

## 4.8将admin账号设置为超级管理员

![1540632998339](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540632998339.png)

![1540633029341](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540633029341.png)

![1540633047661](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540633047661.png)

## 4.9放开gitlab远程ssh

gitlab容器系统ubuntu16.04,因为这里主机和容器之间的端口映射是10022，故只能通过10022来连接容器，同时需要放开容器的22和10022端口

无秘ssh-copy-id -p10022 root@172.17.0.4

配置修改如下:

```
root@32832cabbcbd:/etc/ssh# egrep -vE "^#|^$" sshd_config 
Port 22
Port 10022
Protocol 2
HostKey /etc/ssh/ssh_host_rsa_key
HostKey /etc/ssh/ssh_host_dsa_key
HostKey /etc/ssh/ssh_host_ecdsa_key
HostKey /etc/ssh/ssh_host_ed25519_key
UsePrivilegeSeparation yes
KeyRegenerationInterval 3600
ServerKeyBits 1024
SyslogFacility AUTH
LogLevel INFO
LoginGraceTime 120
PermitRootLogin yes #运行root登陆
StrictModes yes
RSAAuthentication yes
PubkeyAuthentication yes
AuthorizedKeysFile      ~/.ssh/authorized_keys#修改为当前容器内部实际的秘钥绝对路径
IgnoreRhosts yes
RhostsRSAAuthentication no
HostbasedAuthentication no
PermitEmptyPasswords no
ChallengeResponseAuthentication no
X11Forwarding yes
X11DisplayOffset 10
PrintMotd no
PrintLastLog yes
TCPKeepAlive yes
AcceptEnv LANG LC_*
Subsystem sftp /usr/lib/openssh/sftp-server
UsePAM yes    #修改
```

Ubuntu的ssh重启可以使用/etc/init.d/ssh restart

如果上述重启失效，则可以使用这种方法进行启动/usr/sbin/sshd -D & 

# 5.gerrit

## 5.1部署规划

使用docker容器部署到Ubuntu上面无法从gerrit中将代码同步到gitlab中，所以这里将gerrit部署到宿主机上，并使用root账号进行部署，部署到root目录下，便于后续集成

## 5.2create db

```
create database gerritdb CHARACTER SET utf8 COLLATE utf8_general_ci;
```

## 5.3java install

```
tar -zxvf jdk-8u101-linux-x64.tar.gz之后重命名到/usr/java
export JAVA_HOME=/usr/java
export JRE_HOME=${JAVA_HOME}/jre
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export PATH USER LOGNAME MAIL HOSTNAME HISTSIZE INPUTRC JAVA_HOME

source /etc/profile

```

## 5.4gerrrit install

### 5.4.1安装操作

解压压缩包gerrit.zip将里面的插件放到对应的目录

```
#cd /root/ && mkdir -p gerrit/gerrit_site/lib
#cp bcpkix-jdk15on-1.52.jar mysql-connector-java-5.1.21.jar /root/gerrit/gerrit_site/lib/
root@openwallet:~/gerrit/gerrit_site/lib# ls
bcpkix-jdk15on-1.52.jar  mysql-connector-java-5.1.21.jar
#cd /root/gerrit && java -jar gerrit-2.11.3.war init -d ~/gerrit/gerrit_site输入内容如下：
回车
mysql
回车
192.168.32.108
3306
gerritdb
wallet
wallet20180806
回车
http
回车
回车
y
smtp.qq.com
465
tls
3377379238@qq.com
jstrwoswyjxwdbea

root
回车
192.168.32.108
回车
回车
n
回车
回车
回车
连续输入六个y之后回车

java -jar gerrit-2.11.3.war reindex -d /root/gerrit/gerrit_site
cd /root/gerrit/gerrit_sitebak/bin && vim gerrit.sh添加如下
GERRIT_SITE=/root/gerrit/gerrit_site
重启cd /root/gerrit/gerrit_sitebak/bin &&  ./gerrit.sh start
Starting Gerrit Code Review: OK

```

### 5.4.2实际安装记录

```
root@openwallet:~/gerrit# java -jar gerrit-2.11.3.war init -d ~/gerrit/gerrit_site
WARNING: An illegal reflective access operation has occurred
WARNING: Illegal reflective access by com.google.inject.internal.cglib.core.$ReflectUtils$2 (file:/root/.gerritcodereview/tmp/gerrit_10669717050076689175_app/guice-4.0.jar) to method java.lang.ClassLoader.defineClass(java.lang.String,byte[],int,int,java.security.ProtectionDomain)
WARNING: Please consider reporting this to the maintainers of com.google.inject.internal.cglib.core.$ReflectUtils$2
WARNING: Use --illegal-access=warn to enable warnings of further illegal reflective access operations
WARNING: All illegal access operations will be denied in a future release
Using secure store: com.google.gerrit.server.securestore.DefaultSecureStore

*** Gerrit Code Review 2.11.3
*** 


*** Git Repositories
*** 

Location of Git repositories   [git]: 【回车】

*** SQL Database
*** 

Database server type           [h2]: mysql
Server hostname                [localhost]: 192.168.32.108
Server port                    [(mysql default)]: 3306
Database name                  [reviewdb]: gerritdb
Database username              [root]: wallet
wallet's password              : 【输入数据库密码】
              confirm password : 【输入数据库密码】

*** Index
*** 

Type                           [LUCENE/?]: 【回车】

The index must be rebuilt before starting Gerrit:
  java -jar gerrit.war reindex -d site_path

*** User Authentication
*** 

Authentication method          [OPENID/?]: http
Get username from custom HTTP header [y/N]? 【回车】
SSO logout URL                 : 

*** Review Labels
*** 

Install Verified label         [y/N]? y

*** Email Delivery
*** 

SMTP server hostname           [localhost]: smtp.qq.com
SMTP server port               [(default)]: 465
SMTP encryption                [NONE/?]: tls
SMTP username                  [root]: 3377379238@qq.com
3377379238@qq.com's password   : 【输入邮箱密码】
              confirm password : 【输入邮箱密码】

*** Container Process
*** 

Run as                         [root]: root
Java runtime                   [/usr/lib/jvm/java-11-openjdk-amd64]: 
Copy gerrit-2.11.3.war to /root/gerrit/gerrit_site/bin/gerrit.war [Y/n]? y
Copying gerrit-2.11.3.war to /root/gerrit/gerrit_site/bin/gerrit.war

*** SSH Daemon
*** 

Listen on address              [*]: 【回车】
Listen on port                 [29418]: 【回车】

Gerrit Code Review is not shipped with Bouncy Castle Crypto SSL v151
  If available, Gerrit can take advantage of features
  in the library, but will also function without it.
Download and install it now [Y/n]? 【回车】
Renaming bcpkix-jdk15on-1.52.jar to .bcpkix-jdk15on-1.52.jar.backupDownloading http://www.bouncycastle.org/download/bcpkix-jdk15on-151.jar ... !! FAIL !!


error: http://www.bouncycastle.org/download/bcpkix-jdk15on-151.jar: 302 Found
Please download:

  http://www.bouncycastle.org/download/bcpkix-jdk15on-151.jar

and save as:

  /root/gerrit/gerrit_site/lib/bcpkix-jdk15on-151.jar

Press enter to continue 
Continue without this library  [Y/n]?【回车】
Generating SSH host key ... rsa(simple)... done

*** HTTP Daemon
*** 

Behind reverse proxy           [y/N]? 【回车】
Use SSL (https://)             [y/N]? n
Listen on address              [*]: 【回车】
Listen on port                 [8080]: 【回车】
Canonical URL                  [http://localhost:8080/]: 【回车】

*** Plugins
*** 

Installing plugins.
Install plugin download-commands version v2.11.3 [y/N]? y
Install plugin reviewnotes version v2.11.3 [y/N]? y
Install plugin singleusergroup version v2.11.3 [y/N]? y
Install plugin replication version v2.11.3 [y/N]? y
Install plugin commit-message-length-validator version v2.11.3 [y/N]? y
Initializing plugins.
No plugins found with init steps.

Initialized /root/gerrit/gerrit_site

```



```
root@openwallet:~/gerrit# java -jar gerrit-2.11.3.war reindex -d /root/gerrit/gerrit_site
WARNING: An illegal reflective access operation has occurred
WARNING: Illegal reflective access by com.google.inject.internal.cglib.core.$ReflectUtils$2 (file:/root/.gerritcodereview/tmp/gerrit_15172515343403548645_app/guice-4.0.jar) to method java.lang.ClassLoader.defineClass(java.lang.String,byte[],int,int,java.security.ProtectionDomain)
WARNING: Please consider reporting this to the maintainers of com.google.inject.internal.cglib.core.$ReflectUtils$2
WARNING: Use --illegal-access=warn to enable warnings of further illegal reflective access operations
WARNING: All illegal access operations will be denied in a future release
[2018-10-29 11:18:39,010] INFO  com.google.gerrit.server.git.LocalDiskRepositoryManager : Defaulting core.streamFileThreshold to 2009m
[2018-10-29 11:18:39,474] INFO  com.google.gerrit.server.cache.h2.H2CacheFactory : Enabling disk cache /root/gerrit/gerrit_site/cache
Reindexing changes: done    
Reindexed 0 changes in 0.0s (0.0/s)
[2018-10-29 11:18:40,225] WARN  com.google.gerrit.server.cache.h2.H2CacheImpl : Cannot build BloomFilter for jdbc:h2:file:/root/gerrit/gerrit_site/cache/diff_intraline: Error opening database: "Sleep interrupted" [8000-174]
[2018-10-29 11:18:40,225] INFO  com.google.gerrit.server.cache.h2.H2CacheFactory : Finishing 4 disk cache updates

```



## 5.5修改配置如下

```
root@openwallet:~/gerrit/gerrit_site/etc# cat secure.config 
[database]
        password = wallet20180806
[auth]
        registerEmailPrivateKey = X8lZHbC+oGVoDH7d5U3qWAuf5lloBLfKnkM=
        restTokenPrivateKey = ydXsByp2l4hQqNXL2fRGWRTP148tOZPSHyU=
[sendemail]
        smtpPass = jstrwoswyjxwdbea
root@openwallet:~/gerrit/gerrit_site/etc# cat gerrit.config 
[gerrit]
        basePath = git
        canonicalWebUrl = http://192.168.32.108:8080/
[database]
        type = mysql
        hostname = 192.168.32.108
        port = 3306
        database = gerritdb
        username = wallet
[index]
        type = LUCENE
[auth]
        type = LDAP
        gitBasicAuthPolicy = LDAP
[sendemail]
        smtpServer = smtp.qq.com
        smtpServerPort = 465
        smtpEncryption = TLS
        smtpUser = 3377379238@qq.com
        from = 3377379238@qq.com
[ldap]
        server = ldap://192.168.32.108:389
        accountBase = dc=wallet,dc=com
        groupBase = dc=wallet,dc=com
        referral = follow
        accountPattern = (uid=${username})
        groupPattern = (cn=${groupname})
        accountFullName = cn
        #accountMemberField = memberOf
        accountEmailAddress = mail
        username = CN=root,DC=wallet,DC=com
        password = wallet20181027
[container]
        user = root
        javaHome = /usr/lib/jvm/java-11-openjdk-amd64
[plugins]
        allowRemoteAdmin = true
[sshd]
        listenAddress = *:29418
[httpd]
        listenUrl = http://*:8080/
[cache]
        directory = cache
```

下面这个配置是在安装的时候生成的，密码都在这里存放，无需修改

```
root@openwallet:~/gerrit/gerrit_site/etc# cat secure.config 
[database]
        password = wallet20180806
[auth]
        registerEmailPrivateKey = X8lZHbC+oGVoDH7d5U3qWAuf5lloBLfKnkM=
        restTokenPrivateKey = ydXsByp2l4hQqNXL2fRGWRTP148tOZPSHyU=
[sendemail]
        smtpPass = jstrwoswyjxwdbea
```

### 5.6重启

```
root@openwallet:~/gerrit/gerrit_site/bin# ./gerrit.sh restart
Stopping Gerrit Code Review: OK
Starting Gerrit Code Review: OK
```

### 5.7gerrit login

```
http://192.168.32.108:8080
admin wallet@2018
```

首次登陆用户默认为超级管理员

## 5.8create group

![1540790373015](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540790373015.png)

![1540790463216](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540790463216.png)

插件也安装下

```
~/gerrit/gerrit_sitebak/plugins# cp delete-project.jar events-log.jar hooks.jar importer.jar verify-status.jar /root/gerrit/gerrit_site/plugins/
```

## 5.9安装gerrit插件

```
cp delete-project.jar events-log.jar hooks.jar importer.jar verify-status.jar /root/gerrit/gerrit_site/plugins/
```



# 6.jenkins

## 6.1部署规划

端口-p 30000:8080 -p 50000:50000 

## 6.2镜像获取

导出镜像

```
docker load -i jenkins.tar
```

## 6.3构建容器

```
#chown -R 1000:1000 /ci_volume/jenkins_volume
# docker run -itd -p 30000:8080 -p 50000:50000 --name jenkins --privileged=true --restart always -v /ci_volume/jenkins_volume:/var/jenkins_home 941e705b3dc8  #这个是上面导出的镜像id,docker images获取
```

## 6.4jenking login

```
http://192.168.32.108:30000/jenkins/
root wallet20181027

```

![1540692599111](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540692599111.png)

```
jenkins@51a98103eeee:~$ cat /var/jenkins_home/secrets/initialAdminPassword
baf97eebc498485eb6a550d07a850aee

```

下一步也是遇到bug，需要将jenkins去掉，直接以ip+port的形式访问即可http://192.168.32.108:30000

之后点击左边的安装推荐软件即可进入自动安装插件

![1540695525488](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540695525488.png)

![1540692801882](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540692801882.png)

![1540692872667](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540692872667.png)

## 6.5install插件

![1540692955003](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540692955003.png)

![1540693036611](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540693036611.png)

![1540693052379](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540693052379.png)

下面上传安装gerrit插件

![1540693135330](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540693135330.png)

下面把安装完成后重启给勾选上，这样安装完成之后就自动重启了

![1540696680616](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540696680616.png)



## 6.7集成openldap

系统管理-------------全局安全配置

![1540693672081](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540693672081.png)

![1540693694213](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540693694213.png)



![1540694434189](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540694434189.png)

![1540694359921](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540694359921.png)

正常效果如下：

![1540694385428](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540694385428.png)

重新做ldap账号集成，下面的账号权限设置有问题

```
root@openwallet:/ci_volume/jenkins_volume# rm -rf secrets
docker restart jenkins

```

暂时改成登陆用户可以做任何事

![1540695927956](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540695927956.png)

## 6.8配置jenkins

### 6.8.1配置jenkins 地址

系统管理------------系统设置

![1540693603666](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540693603666.png)

![1540693569443](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540693569443.png)

### 6.8.2email配置

![1540696247042](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540696247042.png)

点击进行测试

![1540696607925](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540696607925.png)

# 7.gitlab创建组

使用超级管理员admin账号登陆gitlab后台创建如下三个组

WalletTeamDev
WalletTeamPub
WalletTeamOps

![1540696858218](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540696858218.png)

![1540696907665](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540696907665.png)



![1540696968252](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540696968252.png)

![1540697021131](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540697021131.png)

![1540697044802](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540697044802.png)

## 7.1为组添加成员

每个组暂时都先添加admin    jenkins   maef三个成员：这里有个奇怪现象，jenkins账号在这里登陆之后自动变成admin账号，所以只加两个账号maef和admin即可

![1540697619924](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540697619924.png)

这时用户maef自己的邮箱就会收到邮件

![1540697646605](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540697646605.png)

## 7.2gitlab创建测试项目test

![1540713890882](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540713890882.png)

ssh://git@192.168.32.108:10022/WalletTeamOps/test.git

添加.gitreview文件

![1540713956504](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540713956504.png)

```
[gerrit]
host=192.168.32.108 
port=29418
project=test.git
defaultbranch=master

```

![1540714465242](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540714465242.png)

![1540714486175](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540714486175.png)

# 8.gerrit集成gitlab

## 8.1生成gerrit秘钥

生成 gerrit 用户的密钥，并导入 Gitlab 管理员账户(也是 Review 组唯一能创建 Gitlab 仓库的用户）

这里gerrit部署到了宿主机上，而且是使用的root账号，所以这里要生成root账号的秘钥

```
ssh-keygen -t rsa 之后一路回车会生成秘钥/root/.ssh/id_rsa
```

将gerrit公钥/root/.ssh/id_rsa.pub内容添加到gitlab超级管理员admin账号的sshkey中

## 8.2克隆项目到gerrit

上面做好秘钥，这里使用ssh方式克隆

```
cd /root/gerrit/gerrit_site/git && rm -rf test.git && git clone --bare ssh://git@192.168.32.108:10022/WalletTeamOps/test.git
```

## 8.3gerrit配置

### 8.3.1配置无秘

```
gerrit到gitlab完成无秘ssh-copy-id -i -p 10022 root@172.17.0.4 这里ip是gitlab容器ip
```

### 8.3.2生成秘钥：

下面需要在gerrit所在宿主机执行，如果是gerrit容器，则需要在gerrit容器内执行

```
#sh -c "ssh-keyscan -t rsa 172.17.0.4 >> /root/.ssh/known_hosts"  #这里的ip是gitlab容器的ip
#sh -c "ssh-keygen -H -f /root/.ssh/known_hosts" #加密
```

### 8.3.3编写配置文件

```
cd /root/gerrit/gerrit_site/etc && cat replication.config 
[remote "test"] #项目名称
        projects = test #项目名称
        url = ssh://git@192.168.32.108:10022/WalletTeamOps/test.git #gitlab上面ssh clone地址
        push = +refs/heads/*:refs/heads/*
        push = +refs/tags/*:refs/tags/*
        push = +refs/changes/*:refs/changes/*
        threads = 3
        rescheduleDelay = 0
        StrictHostKeyChecking = no
        defaultForceUpdate = true
```

### 7.3.4编写ssh配置

在gerrit宿主机上，如果是容器则需要在容器中编写

```
#cat /root/.ssh/config
Host 192.168.32.108
        IdentityFile /root/.ssh/id_rsa
        StrictHostKeyChecking no
        UserKnownHostsFile /dev/null
Host *
    KexAlgorithms +diffie-hellman-group1-sha1
```

## 8.4重启&重载

```
cd ~/gerrit/gerrit_site/bin && ./gerrit.sh restart 重启gerrit
先把gerrit容器所在宿主机的公钥添加到gerrit的账号admin中之后在gerrit所在宿主机上执行如下重载
ssh -p 29418 admin@192.168.32.108 gerrit plugin reload replication 重载replication.config配置
ssh -p 29418 admin@192.168.32.108 replication start ***
```

## 8.5gerrit创建测试项目test

gerrit创建的项目和gitlab名称必须一致

![1540791535959](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540791535959.png)

![1540791565132](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540791565132.png)





### 8.5.1添加秘钥

上面已经把gerrit容器中生成的公钥添加到了gitlab的admin账号中，下面也添加到gerrit后台admin账号

![1540715123234](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540715123234.png)

![1540715138836](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540715138836.png)

![1540715177565](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540715177565.png)

### 8.5.2用户添加秘钥

普通用户需要将自己的秘钥分别添加到自己账号的gitlab和gerrit账号内，这里拿maef这个普通账号举例

首先生成秘钥并获取公钥

```
root@openwallet:/r2/test# cat /root/.ssh/id_rsa.pub 
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC1BPv936EDObO5uEdUvKhdWIM7yalAJ+CKtGwd+qRte18lRo1IW88sMcW6lT3HwcG974bNRlja9dtN6hf+z/lYeNi4EnDwKGqbuZX0RqQ5HcaXSo86UkMxMUc154NvYwxrEiXjLlF+hDTWyk/Sjks8PUC4mhhJRHWGOf3qPypJq5Q/XynvlPTtfRdT+EfiFv5JBIt7ZWCxvRiOKV7j9TouokI5IKXQXgGed+oqKGvdce5VGywwsCSdNSzHwpO08ga27W08c91bn5MKXsLbcZmGf5ZCkT5ddVO4dnQFtQaeJj9B91tQG5TLvlOaEHPftGGsaIV3/3jAcs4I8XqFNRfF “3118064263@qq.com”

```

gitlab添加

![1540715396295](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540715396295.png)

![1540715436353](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540715436353.png)

这时添加成功自己也会收到邮件

![1540715478741](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540715478741.png)

gerrit添加秘钥

![1540715540240](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540715540240.png)

![1540715577589](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540715577589.png)





# 9.gerrit集成jenkins

## 9.1jenkins秘钥

生成jenkins秘钥

```
jenkins@51a98103eeee:~$ ssh-keygen -t rsa  一路回车   在jenkins容器内执行
jenkins@51a98103eeee:~$ cat /var/jenkins_home/.ssh/id_rsa.pub 
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCv2Kx4ojsoRvlKbf69cXmXO7V65nSciNLkNBbWoTgDE4lmMpCNcelYbaWuKQeM5RZJKKcfyrtIAqRH/TgjvZ9RizXWU76sHaokWGDwqKZ2QlItkCtnhz4j5dcrPYCKbIW+8/h7JX+plovhUPy5q9XIghUjKNQnJxdHbfUIzrTk0M+nM03WqaWrE0qCeUhTIyME5RY42clD/o3HVGKOAj+TFO3eqwegHUQdXcF4A3edtgxyeMfcsLz5XUX36aQ3I1GXHTEr8RcILExCkzARTtW5Vr92W6+f/8pDaJJq8x+FNTV5J1XeAUw6JU9QqMiIh4+Tmw76d8jhcZTuqwlDz8Z1 jenkins@51a98103eeee

```

上面的jenkins公钥添加到gerrit中admin账号中，也添加到jenkins账号一份；把jenkins账号添加到gerrit账号All-Projects中这里

![1540718091264](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540718091264.png)

![1540718118134](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540718118134.png)



## 9.2gerrit插件配置

![1540717590804](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540717590804.png)

![1540717624642](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540717624642.png)

![1540717699082](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540717699082.png)

![1540790890683](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540790890683.png)

![1540718262342](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540718262342.png)

![1540817532144](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540817532144.png)

## 9.3jenkins创建项目

![1540718349300](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540718349300.png)

![1540718381779](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540718381779.png)

![1540718426691](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540718426691.png)

![1540718454873](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540718454873.png)

![1540718505712](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540718505712.png)

![1540718544924](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540718544924.png)

![1540718607030](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540718607030.png)

![1540718651408](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540718651408.png)

## 9.4测试gerrit&gitlab

```
在本地进行测试操作
操作命令如下：
git clone ssh://git@192.168.32.108:10022/WalletTeamOps/test.git
 cd test/
 echo "111111" >>maef.txt
 git add .
 git commit -m "add maef.txt"
 git review 

 

```

## 9.5配置gerrit&jenkins

![1540719184084](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540719184084.png)



## 

![1540789975634](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540789975634.png)

![1540790009114](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540790009114.png)

![1540790034796](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540790034796.png)

![1540790050547](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540790050547.png)

![1540790117741](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540790117741.png)

![1540790158878](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540790158878.png)

![1540790222596](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540790222596.png)

![1540790316369](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540790316369.png)



![1540790252424](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540790252424.png)





## 9.6gerrit添加秘钥

将jenkins秘钥添加到jenkins账号和admin上面，gerrit页面中

![1540790763836](C:\Users\31180\AppData\Roaming\Typora\typora-user-images\1540790763836.png)

修改jenkins配置gerrit插件

将宿主机的秘钥也要添加到gitlab和gerrit上

将笔记本上的本地秘钥分别添加到maef和admin的秘钥中

# 10.经验

## 10.1gerrit正常推送标志

```
正常推送的标志：
[2018-10-29 15:14:13,379] [e303b6e2] Replication to ssh://git@192.168.32.108:10022/WalletTeamOps/test.git started...
[2018-10-29 15:14:13,382] [e303b6e2] Push to ssh://git@192.168.32.108:10022/WalletTeamOps/test.git references: [RemoteRefUpdate[remoteName=refs/changes/03/3/1, NOT_ATTEMPTED, (null)...1d421a9583ae3b9eac7e5805287f5390ec3990c1, srcRef=refs/changes/03/3/1, forceUpdate, message=null]]
[2018-10-29 15:14:14,395] [e303b6e2] Replication to ssh://git@192.168.32.108:10022/WalletTeamOps/test.git completed in 1016 ms
[2018-10-29 15:14:16,826] [] scheduling replication test:refs/heads/master => ssh://git@192.168.32.108:10022/WalletTeamOps/test.git
[2018-10-29 15:14:16,828] [] scheduled test:refs/heads/master => [0384ca94] push ssh://git@192.168.32.108:10022/WalletTeamOps/test.git to run after 15s
```

## 10.2报错解决



```
报错解决：
root@openwallet:/r2/test/test# git review 
Errors running git rebase -p -i remotes/gerrit/master
Cannot rebase: You have unstaged changes.
Please commit or stash them.
It is likely that your change has a merge conflict. You may resolve it
in the working tree now as described above and then run 'git review'
again, or if you do not want to resolve it yet (note that the change
can not merge until the conflict is resolved) you may run 'git rebase
--abort' then 'git review -R' to upload the change without rebasing.
root@openwallet:/r2/test/test# git rebase -p -i remotes/gerrit/master
Cannot rebase: You have unstaged changes.
请提交或为它们保存进度。
root@openwallet:/r2/test/test# cat maef.txt 
1111
2222
root@openwallet:/r2/test/test# git add .
root@openwallet:/r2/test/test# git commit -m "add maef.txt"
[master 1d421a9] add maef.txt
 1 file changed, 1 insertion(+)
root@openwallet:/r2/test/test# git review
remote: 
remote: Processing changes: new: 1, refs: 1        
remote: Processing changes: new: 1, refs: 1        
remote: Processing changes: new: 1, refs: 1        
remote: Processing changes: new: 1, refs: 1, done            
remote: 
remote: New Changes:        
remote:   http://192.168.32.108:8080/3 add maef.txt        
remote: 
To ssh://admin@192.168.32.108:29418/test.git
 * [new branch]      HEAD -> refs/for/master

```

