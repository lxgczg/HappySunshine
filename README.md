# HappySunshine
数据迁移工具（支持南大Gbase8a、PostgreSQL、达梦DM），个人兴趣，写一写，练一练。

# 本人CSDN博客
[阳光九叶草LXGZXJ](https://blog.csdn.net/qq_45111959?type=blog)

# 一、环境信息
名称 | 值
---- | -----
CPU	| Intel(R) Core(TM) i5-1035G1 CPU @ 1.00GHz
操作系统	| CentOS Linux release 7.9.2009 (Core)
内存	| 5G
逻辑核数	| 6
HappySunshine版本|V1.5
Kettle版本|pdi-ce-9.5.0.1-261
Gbase8a版本|8.6.2-R43.34.27468a27
Pg版本|PostgreSQL 16.3
DM版本|1          DM Database Server 64 V8<br>2          DB Version: 0x7000c<br>3          03134284194-20240703-234060-20108<br>4          Msg Version: 12<br>5          Gsu level(5) cnt: 0

# 二、简述
HappySunshine数据库迁移工具是由C语言编写的多进程多线程程序，支持多种数据库之间的高效数据同步，安装简便、简单配置即可使用，功能还在逐步完善中（其实是还在陆续补充新知识），有什么好的建议，大家可以在评论或私信告知。

# 三、架构图
我的画图水平感觉还不错。

![HS](https://github.com/lxgczg/HappySunshine/blob/main/Photo/HS.PNG)

画图水平感觉还不错，给自己点个赞。

HappySunshine数据库迁移工具由一个管理者进程和N个执行者进程组成，每个执行进程由一个生产者线程和N个消费者线程组成。

1、管理者进程启动。

2、管理者进程读取配置文件参数。

3、管理者进程创建共享内存。

4、管理者进程拉起N个执行者进程。

5、管理者进程通过共享存储中的任务队列，将需要迁移的表放入队列中。

6、执行者进程创建线程池。

7、执行者进程开辟一个生产者线程，N个消费者线程。

8、生产者线程从共享存储中的任务队列拿取任务，查询需要迁移表的条数，来决定迁移方式：LOAD或INSERT。如果进程数大于1，任务中包含的表有一个整型主键，会自动拆分数据为N份（进程数）。

9、如果是LOAD，由生产者线程单独完成。

10、如果是INSERT，将读取的数据行指针打包，发给线程池中的任务队列。

11、消费者线程从线程池中的任务队列拿取任务。

12、消费者线程将任务包解开，将数据清洗，并放入自己的缓冲区中，清洗完再把INSERT命令发给Gbase8a。

13、消费者线程执行任务失败，将任务状态置为错误，待生产者线程发现任务失败后，终止任务下发给线程池队列，重新到共享存储中的任务队列拿取任务。

14、消费者线程执行任务成功，就继续执行。

15、管理者进程将任务发送完成之后，将任务结束标记放入共享存储中的任务队列中。

16、生产者线程从共享存储中的任务队列中拿到任务结束标记后。等待屏障，也就是等待消费者线程完成所拿到的任务。

17、执行者进程等待所有的消费者、生产者结束后，回收线程池资源，释放自身所占用资源，结束退出。

18、管理者进程等待所有执行者进程结束后，回收进程资源，释放自身所占用资源，结束退出。

# 四、升级点
序号|名称|备注
-- | ----- | ------ 
1|PG到DM库级数据迁移|INSERT方式迁移，表定义及其他暂不支持。
2|PG到DM表级数据迁移|INSERT方式迁移，表定义及其他暂不支持。
3|修复库级迁移，PG为源端数据库时，只迁移主键表的BUG|
4|修复目标端数据库连接失败，导致子进程死等的BUG|
5|添加配置文件MigrationConfig.txt中参数值的有效性判断|
6|修复Gbase8a到8a数据迁移LOAD方式，数据包含NULL时，加载报错的BUG|
7|修复Gbase8a到8a数据迁移LOAD方式，数据包含大字段时，加载数据有误的BUG|

# 五、支持功能
序号|功能|备注
-- | ----- | ------ 
1|Gbase8a到8a数据迁移|INSERT、LOAD方式迁移，表定义及其他暂不支持。
2|PG到Gbase8a数据迁移|INSERT方式迁移，表定义及其他暂不支持。
3|PG到DM数据迁移|INSERT方式迁移，表定义及其他暂不支持。
4|多线程并发加载单表数据|	
5|多进程并发加载单表数据|源端库为PG，迁移表包含单一整型主键，进程数设置大于1时，支持此项。

# 六、后续计划支持功能
展望是挺多的，奈何时间不多呀。
序号|功能|备注
-- | ----- | ------ 
1|支持目的端为PG|通过libpq接口Copy。
2|支持Oracle数据迁移|通过OCI接口。
3|支持达梦快速装载|通过DM FLDR的C接口。
4|支持信号处理|	
5|支持License|	

# 七、安装包下载地址
[GITHUB-HappySunshine-release版下载地址](https://github.com/lxgczg/HappySunshine/releases)

# 八、配置参数介绍
序号|参数|备注
-- | ----- | ------ 
1|[Tool]|Tool标签头，下面只能写Tool相关参数。
2|ProcessNums|程序迁移时开的进程数。
3|Level|迁移级别。<br>1:表级迁移，SpecifiedTab生效。<br>2:库级迁移，MigrationDb、BlackList生效。
4|OsInfo|LOAD数据时使用。<br>样例：'工具所在操作系统IP;操作系统用户;操作系统用户密码;'长度同下方的数据库IP;数据库用户名;数据库用户密码;
5|OneBatchNums|MigrationType为0、1、2的情况下，支持此参数，一个批次插入的数据条数。（INSERT方式）
6|SwitchNums|MigrationType为0的情况下，支持此参数，此数以上使用LOAD，以下使用INSERT。
7|MigrationType|迁移类型，支持0、1、2。<br>0 : Gbase8a     -> Gbase8a<br>1 : PostgreSql  -> Gbase8a<br>2 : PostgreSql  -> Dm
8|[Source]|Source标签头，下面只能写Source相关参数。
9|ConnInfo|样例：'IP;数据库用户名;数据库用户密码;数据库名;数据库端口号;数据连接字符集;'<br>1、单个IP长度限制19，数据库IP地址。<br>2、单个用户名长度限制19，数据库用户名。<br>3、单个用户名密码长度限制29，数据库密码。<br>4、单个数据库名长度限制29，数据库名。<br>5、数据库端口。<br>6、长度限制9，数据库连接字符集，支持utf8和gbk，如果是达梦连接，需填写数字，对照表如下：<br>    （1）UTF8                            1<br>    （2）GBK                              2<br>    （3）BIG5                             3<br>    （4）ISO_8859_9                  4<br>    （5）EUC_JP                          5<br>    （6）EUC_KR                        6<br>    （7）KOI8R                           7<br>    （8）ISO_8859_1                  8<br>    （9）SQL_ASCII                    9<br>    （10）GB18030                    10<br>    （11）ISO_8859_11             11
10|MigrationDb|库级迁移参数，单个数据库名长度限制29，需要迁移的数据库名。<br>MigrationType为1的情况下，此为PG的模式名。
11|BlackList|库级迁移参数，支持128个表，黑名单，表名，长度限制参考Db。<br>和MigrationDb一起使用可以。
12|SpecifiedTab|（1）MigrationType为0的情况下，表级迁移参数,格式:'源端查询字段;源端库名;源端表名;源端过滤条件;目的端插入字段;目的端库名;目的端表名;'<br><br>（2）MigrationType为1的情况下，表级迁移参数,格式:'源端查询字段;源端模式名;源端表名;源端过滤条件;目的端插入字段;目的端库名;目的端表名;'<br><br>（3）如果没有特定条件，可以不写，但必须有分隔符，举例如下：';czg;testtab;;;zxj;NewTab;'<br>这样相当于testtab迁移到NewTab，没有任何特殊条件。<br>这个可以有多个标签，想迁移多少张表就写几个标签。
13|[Target]|Target标签头，下面只能写Target相关参数。
14|ConnInfo|参考Source的。
15|MigrationDb|参考Source的。

# 九、安装步骤
大家也可以参考Readme.txt内容。
## 1、用户创建
大家也可以不创建，这是为了不影响其他用户，还有安全性考虑。
```
[root@czg0 ~]# groupadd lzl -g 2024
[root@czg0 ~]# useradd  lzl -g 2024 -u 2024
[root@czg0 ~]# passwd   lzl
```
## 2、安装包解压
```
[lzl@czg0 ~]$ unzip HappySunshine_V1.5_X86_Centos7.9_Release_日期.zip
```
## 3、环境变量配置
```
[root@czg0 ~]# cat /home/lzl/.bashrc 
# .bashrc

# Source global definitions
if [ -f /etc/bashrc ]; then
        . /etc/bashrc
fi

# Uncomment the following line if you don't like systemctl's auto-paging feature:
# export SYSTEMD_PAGER=

# User specific aliases and functions

export HAPPY_SUNSHINE_HOME=/home/lzl/HappySunshine
export LD_LIBRARY_PATH=$HAPPY_SUNSHINE_HOME/Libs:$LD_LIBRARY_PATH
```
## 4、环境变量生效
```
[lzl@czg0 ~]$ source /home/lzl/.bashrc 
```
## 5、动态库链接检验
### （1）HsManager 
```
[lzl@czg0 ~]$ ldd HappySunshine/Exec/HsManager 
        linux-vdso.so.1 =>  (0x00007ffc3a9fa000)
        libPublic.so => /home/lzl/HappySunshine/Libs/libPublic.so (0x00007f094f383000)
        libLog.so => /home/lzl/HappySunshine/Libs/libLog.so (0x00007f094f17e000)
        libSqQueue.so => /home/lzl/HappySunshine/Libs/libSqQueue.so (0x00007f094ef78000)
        libPthread.so => /home/lzl/HappySunshine/Libs/libPthread.so (0x00007f094ed6b000)
        libFileOperate.so => /home/lzl/HappySunshine/Libs/libFileOperate.so (0x00007f094eb65000)
        libDataConvertion.so => /home/lzl/HappySunshine/Libs/libDataConvertion.so (0x00007f094e960000)
        libProcess.so => /home/lzl/HappySunshine/Libs/libProcess.so (0x00007f094e757000)
        libGbase8aDb.so => /home/lzl/HappySunshine/Libs/libGbase8aDb.so (0x00007f094e550000)
        libgbase.so.16 => /home/lzl/HappySunshine/Libs/libgbase.so.16 (0x00007f094e090000)
        libHashTable.so => /home/lzl/HappySunshine/Libs/libHashTable.so (0x00007f094de8c000)
        libPgDb.so => /home/lzl/HappySunshine/Libs/libPgDb.so (0x00007f094dc83000)
        libpq.so.5 => /home/lzl/HappySunshine/Libs/libpq.so.5 (0x00007f094da2d000)
        libHsPublic.so => /home/lzl/HappySunshine/Libs/libHsPublic.so (0x00007f094d82a000)
        libc.so.6 => /lib64/libc.so.6 (0x00007f094d45c000)
        libpthread.so.0 => /lib64/libpthread.so.0 (0x00007f094d240000)
        libdl.so.2 => /lib64/libdl.so.2 (0x00007f094d03c000)
        libm.so.6 => /lib64/libm.so.6 (0x00007f094cd3a000)
        /lib64/ld-linux-x86-64.so.2 (0x00007f094f586000)
```
### （2）G8aExecutor 
```
[lzl@czg0 ~]$ ldd HappySunshine/Exec/G8aExecutor 
        linux-vdso.so.1 =>  (0x00007ffecf9fd000)
        libPublic.so => /home/lzl/HappySunshine/Libs/libPublic.so (0x00007faca38b8000)
        libLog.so => /home/lzl/HappySunshine/Libs/libLog.so (0x00007faca36b3000)
        libSqQueue.so => /home/lzl/HappySunshine/Libs/libSqQueue.so (0x00007faca34ad000)
        libPthread.so => /home/lzl/HappySunshine/Libs/libPthread.so (0x00007faca32a0000)
        libFileOperate.so => /home/lzl/HappySunshine/Libs/libFileOperate.so (0x00007faca309a000)
        libDataConvertion.so => /home/lzl/HappySunshine/Libs/libDataConvertion.so (0x00007faca2e95000)
        libProcess.so => /home/lzl/HappySunshine/Libs/libProcess.so (0x00007faca2c8c000)
        libGbase8aDb.so => /home/lzl/HappySunshine/Libs/libGbase8aDb.so (0x00007faca2a85000)
        libgbase.so.16 => /home/lzl/HappySunshine/Libs/libgbase.so.16 (0x00007faca25c5000)
        libMyPool.so => /home/lzl/HappySunshine/Libs/libMyPool.so (0x00007faca23c2000)
        libHashTable.so => /home/lzl/HappySunshine/Libs/libHashTable.so (0x00007faca21be000)
        libHsPublic.so => /home/lzl/HappySunshine/Libs/libHsPublic.so (0x00007faca1fbb000)
        libc.so.6 => /lib64/libc.so.6 (0x00007faca1bed000)
        libpthread.so.0 => /lib64/libpthread.so.0 (0x00007faca19d1000)
        libdl.so.2 => /lib64/libdl.so.2 (0x00007faca17cd000)
        libm.so.6 => /lib64/libm.so.6 (0x00007faca14cb000)
        /lib64/ld-linux-x86-64.so.2 (0x00007faca3abb000)
```
### （3）Pg2G8aExecutor
```
[lzl@czg0 ~]$ ldd HappySunshine/Exec/Pg2G8aExecutor 
        linux-vdso.so.1 =>  (0x00007ffdea9d0000)
        libPublic.so => /home/lzl/HappySunshine/Libs/libPublic.so (0x00007f2c7a38f000)
        libLog.so => /home/lzl/HappySunshine/Libs/libLog.so (0x00007f2c7a18a000)
        libSqQueue.so => /home/lzl/HappySunshine/Libs/libSqQueue.so (0x00007f2c79f84000)
        libPthread.so => /home/lzl/HappySunshine/Libs/libPthread.so (0x00007f2c79d77000)
        libFileOperate.so => /home/lzl/HappySunshine/Libs/libFileOperate.so (0x00007f2c79b71000)
        libDataConvertion.so => /home/lzl/HappySunshine/Libs/libDataConvertion.so (0x00007f2c7996c000)
        libProcess.so => /home/lzl/HappySunshine/Libs/libProcess.so (0x00007f2c79763000)
        libGbase8aDb.so => /home/lzl/HappySunshine/Libs/libGbase8aDb.so (0x00007f2c7955c000)
        libgbase.so.16 => /home/lzl/HappySunshine/Libs/libgbase.so.16 (0x00007f2c7909c000)
        libMyPool.so => /home/lzl/HappySunshine/Libs/libMyPool.so (0x00007f2c78e99000)
        libPgDb.so => /home/lzl/HappySunshine/Libs/libPgDb.so (0x00007f2c78c90000)
        libpq.so.5 => /home/lzl/HappySunshine/Libs/libpq.so.5 (0x00007f2c78a3a000)
        libHashTable.so => /home/lzl/HappySunshine/Libs/libHashTable.so (0x00007f2c78836000)
        libHsPublic.so => /home/lzl/HappySunshine/Libs/libHsPublic.so (0x00007f2c78633000)
        libc.so.6 => /lib64/libc.so.6 (0x00007f2c78265000)
        libpthread.so.0 => /lib64/libpthread.so.0 (0x00007f2c78049000)
        libdl.so.2 => /lib64/libdl.so.2 (0x00007f2c77e45000)
        libm.so.6 => /lib64/libm.so.6 (0x00007f2c77b43000)
        /lib64/ld-linux-x86-64.so.2 (0x00007f2c7a592000)
```
### （4）Pg2DmExecutor 
```
[lzl@czg0 ~]$ ldd HappySunshine/Exec/Pg2DmExecutor 
        linux-vdso.so.1 =>  (0x00007ffd4eb65000)
        libPublic.so => /home/lzl/HappySunshine/Libs/libPublic.so (0x00007f81c49f0000)
        libLog.so => /home/lzl/HappySunshine/Libs/libLog.so (0x00007f81c47eb000)
        libSqQueue.so => /home/lzl/HappySunshine/Libs/libSqQueue.so (0x00007f81c45e5000)
        libPthread.so => /home/lzl/HappySunshine/Libs/libPthread.so (0x00007f81c43d8000)
        libFileOperate.so => /home/lzl/HappySunshine/Libs/libFileOperate.so (0x00007f81c41d2000)
        libDataConvertion.so => /home/lzl/HappySunshine/Libs/libDataConvertion.so (0x00007f81c3fcd000)
        libProcess.so => /home/lzl/HappySunshine/Libs/libProcess.so (0x00007f81c3dc4000)
        libMyPool.so => /home/lzl/HappySunshine/Libs/libMyPool.so (0x00007f81c3bc1000)
        libLinkList.so => /home/lzl/HappySunshine/Libs/libLinkList.so (0x00007f81c39bd000)
        libPgDb.so => /home/lzl/HappySunshine/Libs/libPgDb.so (0x00007f81c37b4000)
        libpq.so.5 => /home/lzl/HappySunshine/Libs/libpq.so.5 (0x00007f81c355e000)
        libDmDb.so => /home/lzl/HappySunshine/Libs/libDmDb.so (0x00007f81c333f000)
        libdmdpi.so => /home/lzl/HappySunshine/Libs/libdmdpi.so (0x00007f81c2485000)
        libdmcpt.so => /home/lzl/HappySunshine/Libs/libdmcpt.so (0x00007f81c2252000)
        libdmlogmnr_client.so => /home/lzl/HappySunshine/Libs/libdmlogmnr_client.so (0x00007f81c0595000)
        libHashTable.so => /home/lzl/HappySunshine/Libs/libHashTable.so (0x00007f81c0391000)
        libHsPublic.so => /home/lzl/HappySunshine/Libs/libHsPublic.so (0x00007f81c018e000)
        libc.so.6 => /lib64/libc.so.6 (0x00007f81bfdc0000)
        libm.so.6 => /lib64/libm.so.6 (0x00007f81bfabe000)
        libpthread.so.0 => /lib64/libpthread.so.0 (0x00007f81bf8a2000)
        librt.so.1 => /lib64/librt.so.1 (0x00007f81bf69a000)
        libdl.so.2 => /lib64/libdl.so.2 (0x00007f81bf496000)
        libstdc++.so.6 => /lib64/libstdc++.so.6 (0x00007f81bf18e000)
        libgcc_s.so.1 => /lib64/libgcc_s.so.1 (0x00007f81bef78000)
        libdmcfg.so => /home/lzl/HappySunshine/Libs/libdmcfg.so (0x00007f81beb1a000)
        libdmclientlex.so => /home/lzl/HappySunshine/Libs/libdmclientlex.so (0x00007f81be8e5000)
        libdmcomm.so => /home/lzl/HappySunshine/Libs/libdmcomm.so (0x00007f81be6b2000)
        libdmcvt.so => /home/lzl/HappySunshine/Libs/libdmcvt.so (0x00007f81bdf34000)
        libdmcpr.so => /home/lzl/HappySunshine/Libs/libdmcpr.so (0x00007f81bdd30000)
        libdmelog.so => /home/lzl/HappySunshine/Libs/libdmelog.so (0x00007f81bdac1000)
        libdmmsg.so => /home/lzl/HappySunshine/Libs/libdmmsg.so (0x00007f81bd8b1000)
        libdmcalc.so => /home/lzl/HappySunshine/Libs/libdmcalc.so (0x00007f81bd618000)
        libdmcyt.so => /home/lzl/HappySunshine/Libs/libdmcyt.so (0x00007f81bd3f3000)
        libdmos.so => /home/lzl/HappySunshine/Libs/libdmos.so (0x00007f81bd1be000)
        libdmutl.so => /home/lzl/HappySunshine/Libs/libdmutl.so (0x00007f81bcfa3000)
        libdmmem.so => /home/lzl/HappySunshine/Libs/libdmmem.so (0x00007f81bcd90000)
        libdmbtr.so => /home/lzl/HappySunshine/Libs/libdmbtr.so (0x00007f81bcb5c000)
        libdmtrx.so => /home/lzl/HappySunshine/Libs/libdmtrx.so (0x00007f81bc7b5000)
        libdmstrt.so => /home/lzl/HappySunshine/Libs/libdmstrt.so (0x00007f81bc5a6000)
        libdmknl.so => /home/lzl/HappySunshine/Libs/libdmknl.so (0x00007f81bc343000)
        libdmstg.so => /home/lzl/HappySunshine/Libs/libdmstg.so (0x00007f81bc0f4000)
        libdmblb.so => /home/lzl/HappySunshine/Libs/libdmblb.so (0x00007f81bbeb8000)
        libdmnsort.so => /home/lzl/HappySunshine/Libs/libdmnsort.so (0x00007f81bbb9b000)
        libdmllog.so => /home/lzl/HappySunshine/Libs/libdmllog.so (0x00007f81bb974000)
        libdmrs.so => /home/lzl/HappySunshine/Libs/libdmrs.so (0x00007f81bb76f000)
        libdmsys.so => /home/lzl/HappySunshine/Libs/libdmsys.so (0x00007f81bb568000)
        libdmrarch.so => /home/lzl/HappySunshine/Libs/libdmrarch.so (0x00007f81bb313000)
        libdmtrv.so => /home/lzl/HappySunshine/Libs/libdmtrv.so (0x00007f81bb0d9000)
        libdmtimer.so => /home/lzl/HappySunshine/Libs/libdmtimer.so (0x00007f81baed3000)
        libdmmout.so => /home/lzl/HappySunshine/Libs/libdmmout.so (0x00007f81babfe000)
        /lib64/ld-linux-x86-64.so.2 (0x00007f81c4bf3000)
        libdmdcrm.so => /home/lzl/HappySunshine/Libs/libdmdcrm.so (0x00007f81ba9f1000)
        libdmshm.so => /home/lzl/HappySunshine/Libs/libdmshm.so (0x00007f81ba7ec000)
        libdmshmm.so => /home/lzl/HappySunshine/Libs/libdmshmm.so (0x00007f81ba5e6000)
        libdmckpt.so => /home/lzl/HappySunshine/Libs/libdmckpt.so (0x00007f81ba3ca000)
        libdmfil.so => /home/lzl/HappySunshine/Libs/libdmfil.so (0x00007f81ba1a5000)
        libdmdta.so => /home/lzl/HappySunshine/Libs/libdmdta.so (0x00007f81b9eb8000)
        libdmrlog.so => /home/lzl/HappySunshine/Libs/libdmrlog.so (0x00007f81b9bf8000)
        libdmmal.so => /home/lzl/HappySunshine/Libs/libdmmal.so (0x00007f81b99b8000)
        libdmuthr.so => /home/lzl/HappySunshine/Libs/libdmuthr.so (0x00007f81b97a5000)
        libdmtlog.so => /home/lzl/HappySunshine/Libs/libdmtlog.so (0x00007f81b959b000)
        libdmdct.so => /home/lzl/HappySunshine/Libs/libdmdct.so (0x00007f81b8edb000)
        libdmregex.so => /home/lzl/HappySunshine/Libs/libdmregex.so (0x00007f81b8cd4000)
        libdmtbl.so => /home/lzl/HappySunshine/Libs/libdmtbl.so (0x00007f81b8ace000)
        libdmhfs.so => /home/lzl/HappySunshine/Libs/libdmhfs.so (0x00007f81b8835000)
        libdmlic.so => /home/lzl/HappySunshine/Libs/libdmlic.so (0x00007f81b8629000)
        libdmlnk.so => /home/lzl/HappySunshine/Libs/libdmlnk.so (0x00007f81b83f8000)
        libdmredo.so => /home/lzl/HappySunshine/Libs/libdmredo.so (0x00007f81b8143000)
        libdmrtree.so => /home/lzl/HappySunshine/Libs/libdmrtree.so (0x00007f81b7f31000)
        libdmenet.so => /home/lzl/HappySunshine/Libs/libdmenet.so (0x00007f81b7d0b000)
        libdmxmal.so => /home/lzl/HappySunshine/Libs/libdmxmal.so (0x00007f81b7af1000)
        libdmsbtree.so => /home/lzl/HappySunshine/Libs/libdmsbtree.so (0x00007f81b78ed000)
        libdmbcast.so => /home/lzl/HappySunshine/Libs/libdmbcast.so (0x00007f81b766f000)
        libdmscp.so => /home/lzl/HappySunshine/Libs/libdmscp.so (0x00007f81b7457000)
        libdmjson.so => /home/lzl/HappySunshine/Libs/libdmjson.so (0x00007f81b722e000)
        libdmspatial.so => /home/lzl/HappySunshine/Libs/libdmspatial.so (0x00007f81b6f55000)
        libdmvtdskm.so => /home/lzl/HappySunshine/Libs/libdmvtdskm.so (0x00007f81b6d44000)
        libdmdcr.so => /home/lzl/HappySunshine/Libs/libdmdcr.so (0x00007f81b6b3c000)
        libdmasmapi.so => /home/lzl/HappySunshine/Libs/libdmasmapi.so (0x00007f81b6915000)
        libdmasmapim.so => /home/lzl/HappySunshine/Libs/libdmasmapim.so (0x00007f81b66c0000)
        libdmdfi.so => /home/lzl/HappySunshine/Libs/libdmdfi.so (0x00007f81b64aa000)
        libdmvtdsk.so => /home/lzl/HappySunshine/Libs/libdmvtdsk.so (0x00007f81b62a3000)
        libdmasm.so => /home/lzl/HappySunshine/Libs/libdmasm.so (0x00007f81b6074000)
        libdmasmm.so => /home/lzl/HappySunshine/Libs/libdmasmm.so (0x00007f81b5db8000)
        libdmdfs.so => /home/lzl/HappySunshine/Libs/libdmdfs.so (0x00007f81b5b92000)
```
如果有动态库没有找到，就要看看环境变量是否配置正确或是否生效。

如果是安装包中缺少动态库，可以留言告知。

## 6、操作系统限制修改
### （1）/etc/security/limits.conf
添加如下内容
```
lzl soft nofile 1048576
lzl hard nofile 1048576
lzl soft nproc  131072
lzl hard nproc  131072
lzl soft stack  1048576
lzl hard stack  1048576
lzl soft core   unlimited
lzl hard core   unlimited
```
### （2）验证
记得重登操作系统用户，再执行如下命令
```
[lzl@czg0 ~]$ ulimit -a
core file size          (blocks, -c) unlimited
data seg size           (kbytes, -d) unlimited
scheduling priority             (-e) 0
file size               (blocks, -f) unlimited
pending signals                 (-i) 15593
max locked memory       (kbytes, -l) 64
max memory size         (kbytes, -m) unlimited
open files                      (-n) 1048576
pipe size            (512 bytes, -p) 8
POSIX message queues     (bytes, -q) 819200
real-time priority              (-r) 0
stack size              (kbytes, -s) 1048576
cpu time               (seconds, -t) unlimited
max user processes              (-u) 131072
virtual memory          (kbytes, -v) unlimited
file locks                      (-x) unlimited
```
## 7、修改配置文件MigrationConfig.txt
具体的内容我都在配置文件中写好，大家按照规则来就行。
### （1）Gbase8a -> Gbase8a
```
//*代表不可以空
//单引号包围参数，分号分割参数项
//每行开头不可以有空格，不然会跳过此参数检查。
//SpecifiedTab参数可以有多个。
 
[Tool]                                                           //工具信息。

ProcessNums   : '1;'                                             //*程序迁移时开的进程数。

Level         : '1;'                                             //*迁移级别。
                                                                 //2:库级迁移，BlackList生效。
                                                                 //1:表级迁移，SpecifiedTab生效。
OsInfo        : '192.168.142.12;gbase;gbase;'                    //*Gbase8a -> Gbase8a LOAD使用。工具所在操作系统IP;操作系统用户;操作系统用户密码;长度同下方的数据库IP;数据库用户名;数据库用户密码;

OneBatchNums  : '50000;'                                         //*MigrationType为0、1、2的情况下，支持此参数，一个批次插入的数据条数。（INSERT方式）

SwitchNums    : '3000000;'                                       //*MigrationType为0的情况下，支持此参数，此数以上使用LOAD，以下使用INSERT。

MigrationType : '0;'                                             //*迁移类型，支持0、1、2。
                                                                 //0 : Gbase8a     -> Gbase8a
                                                                 //1 : PostgreSql  -> Gbase8a
                                                                 //2 : PostgreSql  -> Dm
                          
[Source]                                                         //源端信息。

ConnInfo      : '192.168.142.12;root;;czg;5258;utf8;'            //'IP;数据库用户名;数据库用户密码;数据库名;数据库端口号;数据连接字符集;'
                                                                 //*单个IP长度限制19，数据库IP地址。
                                                                 //*单个用户名长度限制12，数据库用户名。
                                                                 //*单个用户名密码长度限制29，数据库密码。
                                                                 //*单个数据库名长度限制29，数据库名。
                                                                 //*数据库端口。
                                                                 //*长度限制9，数据库连接字符集，支持utf8和gbk，如果是达梦连接，需填写数字，对照表如下：
                                                                 //（1）UTF8                            1
                                                                 //（2）GBK                             2
                                                                 //（3）BIG5                            3
                                                                 //（4）ISO_8859_9                      4
                                                                 //（5）EUC_JP                          5
                                                                 //（6）EUC_KR                          6
                                                                 //（7）KOI8R                           7
                                                                 //（8）ISO_8859_1                      8
                                                                 //（9）SQL_ASCII                       9
                                                                 //（10）GB18030                        10
                                                                 //（11）ISO_8859_11                    11

MigrationDb   : 'public;'                                        //库级迁移参数，单个数据库名长度限制29，需要迁移的数据库名。
                                                                 //MigrationType为1的情况下，此为PG的模式名。

BlackList     : ''                                               //库级迁移参数，支持50个表，黑名单，表名，长度限制参考Db。和MigrationDb一起使用可以。        

#SpecifiedTab  : 'a,b,c;czg;testtab;;x,y,z;zxj;NewTab;'            
                                                                 //MigrationType为0的情况下，表级迁移参数,格式:'源端查询字段;源端库名;源端表名;源端过滤条件;目的端插入字段;目的端库名;目的端表名;'
                                                                 //MigrationType为1的情况下，表级迁移参数,格式:'源端查询字段;源端模式名;源端表名;源端过滤条件;目的端插入字段;目的端库名;目的端表名;'
                                                                 //如果没有特定条件，可以不写，但必须有分隔符，举例如下：
                                                                 //';czg;testtab;;;zxj;NewTab;'
                                                                 //这样相当于testtab迁移到NewTab，没有任何特殊条件。
                                                                 //这个可以有多个标签，想迁移多少张表就写几个标签。
#SpecifiedTab  : ';czg;testtab_copy;;;zxj;testtab_copy;'
#SpecifiedTab  : ';czg;czg;;;zxj;czg;'

#SpecifiedTab  : ';public;testtab;;;zxj;testtab;'
#SpecifiedTab  : ';public;students;;;zxj;students;'
#SpecifiedTab  : ';public;haha;;;zxj;haha;'

SpecifiedTab  : ';public;testtab;;;zxj;testtab;'

[Target]                                                         //目的端信息。

ConnInfo      : '192.168.142.12;czg;qwer1234;zxj;5258;utf8;'

MigrationDb   : 'zxj;'                                           //库级迁移参数，单个数据库名长度限制29，需要迁移的数据库名。
```
### （2）PostgreSql  -> Gbase8a
```
//*代表不可以空
//单引号包围参数，分号分割参数项
//每行开头不可以有空格，不然会跳过此参数检查。
//SpecifiedTab参数可以有多个。
 
[Tool]                                                           //工具信息。

ProcessNums   : '1;'                                             //*程序迁移时开的进程数。

Level         : '1;'                                             //*迁移级别。
                                                                 //2:库级迁移，BlackList生效。
                                                                 //1:表级迁移，SpecifiedTab生效。
OsInfo        : '192.168.142.12;gbase;gbase;'                    //*Gbase8a -> Gbase8a LOAD使用。工具所在操作系统IP;操作系统用户;操作系统用户密码;长度同下方的数据库IP;数据库用户名;数据库用户密码;

OneBatchNums  : '50000;'                                         //*MigrationType为0、1、2的情况下，支持此参数，一个批次插入的数据条数。（INSERT方式）

SwitchNums    : '3000000;'                                       //*MigrationType为0的情况下，支持此参数，此数以上使用LOAD，以下使用INSERT。

MigrationType : '1;'                                             //*迁移类型，支持0、1、2。
                                                                 //0 : Gbase8a     -> Gbase8a
                                                                 //1 : PostgreSql  -> Gbase8a
                                                                 //2 : PostgreSql  -> Dm
                          
[Source]                                                         //源端信息。

ConnInfo      : '192.168.142.12;postgres;postgres;czg;5432;utf8;' //'IP;数据库用户名;数据库用户密码;数据库名;数据库端口号;数据连接字符集;'
                                                                 //*单个IP长度限制19，数据库IP地址。
                                                                 //*单个用户名长度限制12，数据库用户名。
                                                                 //*单个用户名密码长度限制29，数据库密码。
                                                                 //*单个数据库名长度限制29，数据库名。
                                                                 //*数据库端口。
                                                                 //*长度限制9，数据库连接字符集，支持utf8和gbk，如果是达梦连接，需填写数字，对照表如下：
                                                                 //（1）UTF8                            1
                                                                 //（2）GBK                             2
                                                                 //（3）BIG5                            3
                                                                 //（4）ISO_8859_9                      4
                                                                 //（5）EUC_JP                          5
                                                                 //（6）EUC_KR                          6
                                                                 //（7）KOI8R                           7
                                                                 //（8）ISO_8859_1                      8
                                                                 //（9）SQL_ASCII                       9
                                                                 //（10）GB18030                        10
                                                                 //（11）ISO_8859_11                    11

MigrationDb   : 'public;'                                        //库级迁移参数，单个数据库名长度限制29，需要迁移的数据库名。
                                                                 //MigrationType为1的情况下，此为PG的模式名。

BlackList     : ''                                               //库级迁移参数，支持50个表，黑名单，表名，长度限制参考Db。和MigrationDb一起使用可以。        

#SpecifiedTab  : 'a,b,c;czg;testtab;;x,y,z;zxj;NewTab;'            
                                                                 //MigrationType为0的情况下，表级迁移参数,格式:'源端查询字段;源端库名;源端表名;源端过滤条件;目的端插入字段;目的端库名;目的端表名;'
                                                                 //MigrationType为1的情况下，表级迁移参数,格式:'源端查询字段;源端模式名;源端表名;源端过滤条件;目的端插入字段;目的端库名;目的端表名;'
                                                                 //如果没有特定条件，可以不写，但必须有分隔符，举例如下：
                                                                 //';czg;testtab;;;zxj;NewTab;'
                                                                 //这样相当于testtab迁移到NewTab，没有任何特殊条件。
                                                                 //这个可以有多个标签，想迁移多少张表就写几个标签。
#SpecifiedTab  : ';czg;testtab_copy;;;zxj;testtab_copy;'
#SpecifiedTab  : ';czg;czg;;;zxj;czg;'

#SpecifiedTab  : ';public;testtab;;;zxj;testtab;'
#SpecifiedTab  : ';public;students;;;zxj;students;'
#SpecifiedTab  : ';public;haha;;;zxj;haha;'

SpecifiedTab  : ';public;testtab;;;zxj;testtab;'

[Target]                                                         //目的端信息。

ConnInfo      : '192.168.142.12;czg;qwer1234;zxj;5258;utf8;'

MigrationDb   : 'zxj;'                                           //库级迁移参数，单个数据库名长度限制29，需要迁移的数据库名。
```
### （3）PostgreSql  -> Dm
```
//*代表不可以空
//单引号包围参数，分号分割参数项
//每行开头不可以有空格，不然会跳过此参数检查。
//SpecifiedTab参数可以有多个。
 
[Tool]                                                           //工具信息。

ProcessNums   : '1;'                                             //*程序迁移时开的进程数。

Level         : '1;'                                             //*迁移级别。
                                                                 //2:库级迁移，BlackList生效。
                                                                 //1:表级迁移，SpecifiedTab生效。
OsInfo        : '192.168.142.12;gbase;gbase;'                    //*Gbase8a -> Gbase8a LOAD使用。工具所在操作系统IP;操作系统用户;操作系统用户密码;长度同下方的数据库IP;数据库用户名;数据库用户密码;

OneBatchNums  : '50000;'                                         //*MigrationType为0、1、2的情况下，支持此参数，一个批次插入的数据条数。（INSERT方式）

SwitchNums    : '3000000;'                                       //*MigrationType为0的情况下，支持此参数，此数以上使用LOAD，以下使用INSERT。

MigrationType : '2;'                                             //*迁移类型，支持0、1、2。
                                                                 //0 : Gbase8a     -> Gbase8a
                                                                 //1 : PostgreSql  -> Gbase8a
                                                                 //2 : PostgreSql  -> Dm
                          
[Source]                                                         //源端信息。

ConnInfo      : '192.168.142.12;postgres;postgres;czg;5432;utf8;' //'IP;数据库用户名;数据库用户密码;数据库名;数据库端口号;数据连接字符集;'
                                                                 //*单个IP长度限制19，数据库IP地址。
                                                                 //*单个用户名长度限制12，数据库用户名。
                                                                 //*单个用户名密码长度限制29，数据库密码。
                                                                 //*单个数据库名长度限制29，数据库名。
                                                                 //*数据库端口。
                                                                 //*长度限制9，数据库连接字符集，支持utf8和gbk，如果是达梦连接，需填写数字，对照表如下：
                                                                 //（1）UTF8                            1
                                                                 //（2）GBK                             2
                                                                 //（3）BIG5                            3
                                                                 //（4）ISO_8859_9                      4
                                                                 //（5）EUC_JP                          5
                                                                 //（6）EUC_KR                          6
                                                                 //（7）KOI8R                           7
                                                                 //（8）ISO_8859_1                      8
                                                                 //（9）SQL_ASCII                       9
                                                                 //（10）GB18030                        10
                                                                 //（11）ISO_8859_11                    11

MigrationDb   : 'public;'                                        //库级迁移参数，单个数据库名长度限制29，需要迁移的数据库名。
                                                                 //MigrationType为1的情况下，此为PG的模式名。

BlackList     : ''                                               //库级迁移参数，支持50个表，黑名单，表名，长度限制参考Db。和MigrationDb一起使用可以。        

#SpecifiedTab  : 'a,b,c;czg;testtab;;x,y,z;zxj;NewTab;'            
                                                                 //MigrationType为0的情况下，表级迁移参数,格式:'源端查询字段;源端库名;源端表名;源端过滤条件;目的端插入字段;目的端库名;目的端表名;'
                                                                 //MigrationType为1的情况下，表级迁移参数,格式:'源端查询字段;源端模式名;源端表名;源端过滤条件;目的端插入字段;目的端库名;目的端表名;'
                                                                 //如果没有特定条件，可以不写，但必须有分隔符，举例如下：
                                                                 //';czg;testtab;;;zxj;NewTab;'
                                                                 //这样相当于testtab迁移到NewTab，没有任何特殊条件。
                                                                 //这个可以有多个标签，想迁移多少张表就写几个标签。
#SpecifiedTab  : ';czg;testtab_copy;;;zxj;testtab_copy;'
#SpecifiedTab  : ';czg;czg;;;zxj;czg;'

#SpecifiedTab  : ';public;testtab;;;zxj;testtab;'
#SpecifiedTab  : ';public;students;;;zxj;students;'
#SpecifiedTab  : ';public;haha;;;zxj;haha;'

SpecifiedTab  : ';public;testtab;;;zxj;testtab;'

[Target]                                                         //目的端信息。

ConnInfo      : '192.168.142.12;SYSDBA;SYSDBA;;5238;1;'

MigrationDb   : 'zxj;'                                           //库级迁移参数，单个数据库名长度限制29，需要迁移的数据库名。
```

# 十、性能对比测试
和开源ETL工具Kettle进行对比测试。
由于本人测试条件有限（所有的数据库和工具都部署在一个虚机里），如果大家有条件可以在性能更好的环境下测试，迁移效率肯定会比下面的结果更好。
## 1、性能测试对比表格
工具名\迁移项（单位：行/秒）|Gbase8a -> Gbase8a|PostgreSql -> Gbase8a|PostgreSql -> Dm
------- | ----- | ------ | ------ 
HappySunshine|43690（INSERT）<br>98304（LOAD）|43690|78643
Kettle|3215|2964|20837

## 2、测试表结构
### （1）Gbase8a
```
gbase> DESC ZXJ.TESTTAB;
+-------+---------------+------+-----+-------------------+-----------------------------+
| Field | Type          | Null | Key | Default           | Extra                       |
+-------+---------------+------+-----+-------------------+-----------------------------+
| a     | int(11)       | YES  |     | NULL              |                             |
| b     | double        | YES  |     | NULL              |                             |
| c     | varchar(100)  | YES  | MUL | NULL              |                             |
| d     | text          | YES  |     | NULL              |                             |
| e     | blob          | YES  |     | NULL              |                             |
| f     | longblob      | YES  |     | NULL              |                             |
| g     | date          | YES  |     | NULL              |                             |
| h     | timestamp     | NO   |     | CURRENT_TIMESTAMP | on update CURRENT_TIMESTAMP |
| i     | decimal(10,2) | YES  |     | NULL              |                             |
+-------+---------------+------+-----+-------------------+-----------------------------+
9 rows in set (Elapsed: 00:00:00.00)
```
### （2）DM
```
SQL> DESC ZXJ.TESTTAB;

行号     NAME TYPE$        NULLABLE
---------- ---- ------------ --------
1          A    INTEGER      Y
2          B    DOUBLE       Y
3          C    VARCHAR(100) Y
4          D    TEXT         Y
5          E    BLOB         Y
6          F    BLOB         Y
7          G    DATE         Y
8          H    DATETIME(6)  Y
9          I    DEC(10, 2)   Y

9 rows got
```
### （3）PostgreSql
```
czg=# \d public.testtab
                                      Table "public.testtab"
 Column |            Type             | Collation | Nullable |              Default               
--------+-----------------------------+-----------+----------+------------------------------------
 a      | integer                     |           | not null | nextval('testtab_a_seq'::regclass)
 b      | double precision            |           |          | 
 c      | character varying(100)      |           |          | 
 d      | text                        |           |          | 
 e      | bytea                       |           |          | 
 f      | bytea                       |           |          | 
 g      | date                        |           |          | 
 h      | timestamp without time zone |           |          | 
 i      | numeric(10,2)               |           |          | 
Indexes:
    "testtab_pkey" PRIMARY KEY, btree (a)
```
## 3、测试数据样式
```
czg=# SELECT * FROM PUBLIC.TESTTAB LIMIT 10;
    a    |  b  |      c      |    d     |       e        |             f              |     g      |          h          |    i    
---------+-----+-------------+----------+----------------+----------------------------+------------+---------------------+---------
 2399798 | 2.1 | LXG'ZXJ|CLX | HAHHAHAH | \x414141414141 | \x424242424242424242424242 | 2024-08-19 | 2024-08-19 00:00:00 |        
 2399799 | 2.1 | LXG'ZXJ|CLX | HAHHAHAH | \x414141414141 | \x424242424242424242424242 | 2024-08-19 | 2024-08-19 00:00:00 |    0.00
 2399800 | 2.1 | LXG'ZXJ|CLX | HAHHAHAH | \x414141414141 | \x424242424242424242424242 | 2024-08-19 | 2024-08-19 00:00:00 |    8.80
 2399801 | 2.1 | LXG'ZXJ|CLX | HAHHAHAH | \x414141414141 | \x424242424242424242424242 | 2024-08-19 | 2024-08-19 00:00:00 |   -8.80
 2399802 | 2.1 | LXG'ZXJ|CLX | HAHHAHAH | \x414141414141 | \x424242424242424242424242 | 2024-08-19 | 2024-08-19 00:00:00 |  348.80
 2399803 | 2.1 | LXG'ZXJ|CLX | HAHHAHAH | \x414141414141 | \x424242424242424242424242 | 2024-08-19 | 2024-08-19 00:00:00 | -348.80
 2399804 | 2.1 | LXG'ZXJ|CLX | HAHHAHAH | \x414141414141 | \x424242424242424242424242 | 2024-08-19 | 2024-08-19 00:00:00 |        
 2399805 | 2.1 | LXG'ZXJ|CLX | HAHHAHAH | \x414141414141 | \x424242424242424242424242 | 2024-08-19 | 2024-08-19 00:00:00 |    0.00
 2399806 | 2.1 | LXG'ZXJ|CLX | HAHHAHAH | \x414141414141 | \x424242424242424242424242 | 2024-08-19 | 2024-08-19 00:00:00 |    8.80
 2399807 | 2.1 | LXG'ZXJ|CLX | HAHHAHAH | \x414141414141 | \x424242424242424242424242 | 2024-08-19 | 2024-08-19 00:00:00 |   -8.80
(10 rows)
```

## 4、HappySunshine测试截图
### （1）PostgreSql -> Dm
![INSERT](https://i-blog.csdnimg.cn/direct/ae3c4ac54cba4f0baf71eac722cc2db7.png)
### （2）PostgreSql -> Gbase8a
![INSERT](https://i-blog.csdnimg.cn/direct/6eecb25f6bb84a579e6020d7c982b1e9.png)
### （3）Gbase8a-> Gbase8a
INSERT
![INSERT](https://i-blog.csdnimg.cn/direct/89d0108aca714bdf859822acf6f22286.png)
LOAD
![INSERT](https://i-blog.csdnimg.cn/direct/f49687a2ff6d451aa32c12a8deac6f3e.png)

## 5、Kettle测试截图
### （1）PostgreSql -> Dm
![INSERT](https://i-blog.csdnimg.cn/direct/93dbf074382c4ee6869aa2cf467e55a7.png)
测试步骤大家可以参考之前写的博客《》
### （2）PostgreSql -> Gbase8a
![INSERT](https://i-blog.csdnimg.cn/direct/2e3415071d364de8af997cdbcf642fee.png)
测试步骤大家可以参考之前写的博客《》
### （3）Gbase8a-> Gbase8a
![INSERT](https://i-blog.csdnimg.cn/direct/194d48037d15427f99f9b481b94164c1.png)
测试步骤大家可以参考之前写的博客《》
