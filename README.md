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
序号|功能|备注
-- | ----- | ------ 
1|支持目的端为PG|通过libpq接口Copy。
2|支持Oracle数据迁移|通过OCI接口。
3|支持达梦快速装载|通过DM FLDR的C接口。
4|支持信号处理|	
5|支持License|	

# 七、安装包下载地址
[releases版本下载地址](https://github.com/lxgczg/HappySunshine/releases)

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
9|ConnInfo|样例：'IP;数据库用户名;数据库用户密码;数据库名;数据库端口号;数据连接字符集;'<br>（1）单个IP长度限制19，数据库IP地址。<br>（2）单个用户名长度限制19，数据库用户名。<br>（3）单个用户名密码长度限制29，数据库密码。<br>（4）单个数据库名长度限制29，数据库名。<br>（5）数据库端口。<br>（6）长度限制9，数据库连接字符集，支持utf8和gbk，如果是达梦连接，需填写数字，对照表如下：<br>（1）UTF8                            1<br>（2）GBK                              2<br>（3）BIG5                             3<br>（4）ISO_8859_9                  4<br>（5）EUC_JP                          5<br>（6）EUC_KR                        6<br>（7）KOI8R                           7<br>（8）ISO_8859_1                  8<br>（9）SQL_ASCII                    9<br>（10）GB18030                    10<br>（11）ISO_8859_11             11
10|MigrationDb|库级迁移参数，单个数据库名长度限制29，需要迁移的数据库名。<br>MigrationType为1的情况下，此为PG的模式名。
11|BlackList|库级迁移参数，支持128个表，黑名单，表名，长度限制参考Db。<br>和MigrationDb一起使用可以。
12|SpecifiedTab|（1）MigrationType为0的情况下，表级迁移参数,格式:'源端查询字段;源端库名;源端表名;源端过滤条件;目的端插入字段;目的端库名;目的端表名;'<br><br>（2）MigrationType为1的情况下，表级迁移参数,格式:'源端查询字段;源端模式名;源端表名;源端过滤条件;目的端插入字段;目的端库名;目的端表名;'<br><br>（3）如果没有特定条件，可以不写，但必须有分隔符，举例如下：';czg;testtab;;;zxj;NewTab;'<br>这样相当于testtab迁移到NewTab，没有任何特殊条件。<br>这个可以有多个标签，想迁移多少张表就写几个标签。
13|[Target]|Target标签头，下面只能写Target相关参数。
14|ConnInfo|参考Source的。
15|MigrationDb|参考Source的。

# 八、安装步骤
大家也可以参考Readme.txt内容。

## 1、配置环境变量
```
/home/gbase/.bashrc中添加如下

[gbase@czg0 Exec]$ vim /home/gbase/.bashrc
export HAPPY_SUNSHINE_HOME=/home/gbase/HappySunshine
export LD_LIBRARY_PATH=$HAPPY_SUNSHINE_HOME/Libs:$LD_LIBRARY_PATH
```
## 2、生效环境变量
```
source /home/gbase/.bashrc
```
## 3、检验动态链接是否正常
```
[gbase@czg0 Exec]$ pwd
/home/gbase/HappySunshine/Exec

[gbase@czg0 Exec]$ ldd HsManager 
        linux-vdso.so.1 =>  (0x00007ffefa1e1000)
        libPublic.so => /home/gbase/HappySunshine/Libs/libPublic.so (0x00007fcd61e61000)
        libLog.so => /home/gbase/HappySunshine/Libs/libLog.so (0x00007fcd61c5c000)
        libSqQueue.so => /home/gbase/HappySunshine/Libs/libSqQueue.so (0x00007fcd61a56000)
        libPthread.so => /home/gbase/HappySunshine/Libs/libPthread.so (0x00007fcd61849000)
        libFileOperate.so => /home/gbase/HappySunshine/Libs/libFileOperate.so (0x00007fcd61643000)
        libDataConvertion.so => /home/gbase/HappySunshine/Libs/libDataConvertion.so (0x00007fcd6143f000)
        libProcess.so => /home/gbase/HappySunshine/Libs/libProcess.so (0x00007fcd61236000)
        libGbase8aDb.so => /home/gbase/HappySunshine/Libs/libGbase8aDb.so (0x00007fcd6102f000)
        libgbase.so.16 => /home/gbase/HappySunshine/Libs/libgbase.so.16 (0x00007fcd60b6f000)
        libHashTable.so => /home/gbase/HappySunshine/Libs/libHashTable.so (0x00007fcd6096b000)
        libPgDb.so => /home/gbase/HappySunshine/Libs/libPgDb.so (0x00007fcd60763000)
        libpq.so.5 => /home/gbase/HappySunshine/Libs/libpq.so.5 (0x00007fcd6050d000)
        libHsPublic.so => /home/gbase/HappySunshine/Libs/libHsPublic.so (0x00007fcd6030a000)
        libc.so.6 => /lib64/libc.so.6 (0x00007fcd5ff3c000)
        libpthread.so.0 => /lib64/libpthread.so.0 (0x00007fcd5fd20000)
        libdl.so.2 => /lib64/libdl.so.2 (0x00007fcd5fb1c000)
        libm.so.6 => /lib64/libm.so.6 (0x00007fcd5f81a000)
        /lib64/ld-linux-x86-64.so.2 (0x00007fcd62064000)

[gbase@czg0 Exec]$ ldd G8aExecutor 
        linux-vdso.so.1 =>  (0x00007fff53dab000)
        libPublic.so => /home/gbase/HappySunshine/Libs/libPublic.so (0x00007fb6e7799000)
        libLog.so => /home/gbase/HappySunshine/Libs/libLog.so (0x00007fb6e7594000)
        libSqQueue.so => /home/gbase/HappySunshine/Libs/libSqQueue.so (0x00007fb6e738e000)
        libPthread.so => /home/gbase/HappySunshine/Libs/libPthread.so (0x00007fb6e7181000)
        libFileOperate.so => /home/gbase/HappySunshine/Libs/libFileOperate.so (0x00007fb6e6f7b000)
        libDataConvertion.so => /home/gbase/HappySunshine/Libs/libDataConvertion.so (0x00007fb6e6d77000)
        libProcess.so => /home/gbase/HappySunshine/Libs/libProcess.so (0x00007fb6e6b6e000)
        libGbase8aDb.so => /home/gbase/HappySunshine/Libs/libGbase8aDb.so (0x00007fb6e6967000)
        libgbase.so.16 => /home/gbase/HappySunshine/Libs/libgbase.so.16 (0x00007fb6e64a7000)
        libMyPool.so => /home/gbase/HappySunshine/Libs/libMyPool.so (0x00007fb6e62a4000)
        libHashTable.so => /home/gbase/HappySunshine/Libs/libHashTable.so (0x00007fb6e60a0000)
        libHsPublic.so => /home/gbase/HappySunshine/Libs/libHsPublic.so (0x00007fb6e5e9d000)
        libc.so.6 => /lib64/libc.so.6 (0x00007fb6e5acf000)
        libpthread.so.0 => /lib64/libpthread.so.0 (0x00007fb6e58b3000)
        libdl.so.2 => /lib64/libdl.so.2 (0x00007fb6e56af000)
        libm.so.6 => /lib64/libm.so.6 (0x00007fb6e53ad000)
        /lib64/ld-linux-x86-64.so.2 (0x00007fb6e799c000)

[gbase@czg0 Exec]$ ldd Pg2G8aExecutor 
        linux-vdso.so.1 =>  (0x00007ffcefb8f000)
        libPublic.so => /home/gbase/HappySunshine/Libs/libPublic.so (0x00007f3ce49a4000)
        libLog.so => /home/gbase/HappySunshine/Libs/libLog.so (0x00007f3ce479f000)
        libSqQueue.so => /home/gbase/HappySunshine/Libs/libSqQueue.so (0x00007f3ce4599000)
        libPthread.so => /home/gbase/HappySunshine/Libs/libPthread.so (0x00007f3ce438c000)
        libFileOperate.so => /home/gbase/HappySunshine/Libs/libFileOperate.so (0x00007f3ce4186000)
        libDataConvertion.so => /home/gbase/HappySunshine/Libs/libDataConvertion.so (0x00007f3ce3f82000)
        libProcess.so => /home/gbase/HappySunshine/Libs/libProcess.so (0x00007f3ce3d79000)
        libGbase8aDb.so => /home/gbase/HappySunshine/Libs/libGbase8aDb.so (0x00007f3ce3b72000)
        libgbase.so.16 => /home/gbase/HappySunshine/Libs/libgbase.so.16 (0x00007f3ce36b2000)
        libMyPool.so => /home/gbase/HappySunshine/Libs/libMyPool.so (0x00007f3ce34af000)
        libPgDb.so => /home/gbase/HappySunshine/Libs/libPgDb.so (0x00007f3ce32a7000)
        libpq.so.5 => /home/gbase/HappySunshine/Libs/libpq.so.5 (0x00007f3ce3051000)
        libHashTable.so => /home/gbase/HappySunshine/Libs/libHashTable.so (0x00007f3ce2e4d000)
        libHsPublic.so => /home/gbase/HappySunshine/Libs/libHsPublic.so (0x00007f3ce2c4a000)
        libc.so.6 => /lib64/libc.so.6 (0x00007f3ce287c000)
        libpthread.so.0 => /lib64/libpthread.so.0 (0x00007f3ce2660000)
        libdl.so.2 => /lib64/libdl.so.2 (0x00007f3ce245c000)
        libm.so.6 => /lib64/libm.so.6 (0x00007f3ce215a000)
        /lib64/ld-linux-x86-64.so.2 (0x00007f3ce4ba7000)
```
如果有动态库没有找到，就要看看环境变量是否配置正确或是否生效。

## 4、修改配置文件MigrationConfig.txt
具体的内容我都在配置文件中写好，大家按照规则来就行。

### （1）Gbase8a -> Gbase8a
```
//*代表不可以空
//单引号包围参数，分号分割参数项
//每行开头不可以有空格，不然会跳过此参数检查。
//SpecifiedTab参数可以有多个。
 
[Tool]                                                              //工具信息。
ProcessNums      : '2;'                                             //*程序迁移时开的进程数。
Level            : '1;'	                                            //*迁移级别。
                                                                    //2:库级迁移，BlackList生效。
                                                                    //1:表级迁移，SpecifiedTab生效。
OsInfo           : '192.168.142.12;gbase;gbase;'                    //*Gbase8a -> Gbase8a LOAD使用。工具所在操作系统IP;操作系统用户;操作系统用户密码;长度同下方的数据库IP;数据库用户名;数据库用户密码;
OneBatchNums     : '80000;'                                         //*一个批次插入的数据条数。（INSERT方式）
SwitchNums       : '2000000;'                                       //*此数以上使用LOAD，以下使用INSERT。
MigrationType    : '0;'                                             //*迁移类型，支持0、1。
                                                                    //0 : Gbase8a     -> Gbase8a
                                                                    //1 : PostgreSql  -> Gbase8a

[Source]                                                            //源端信息。
ConnInfo         : '192.168.142.12;root;;czg;5258;utf8;'            //'IP;数据库用户名;数据库用户密码;数据库名;数据库端口号;数据连接字符集;'
                                                                    //*单个IP长度限制19，数据库IP地址。
                                                                    //*单个用户名长度限制19，数据库用户名。
                                                                    //*单个用户名密码长度限制29，数据库密码。
                                                                    //*单个数据库名长度限制29，数据库名。
                                                                    //*数据库端口。
                                                                    //*长度限制9，数据库连接字符集，支持utf8和gbk。
MigrationDb      : 'czg;'                                           //库级迁移参数，单个数据库名长度限制29，需要迁移的数据库名。
                                                                    //MigrationType为1的情况下，此为PG的模式名。
BlackList        : 'testtab;'                                       //库级迁移参数，支持50个表，黑名单，表名，长度限制参考Db。和MigrationDb一起使用可以。        
SpecifiedTab     : 'a,b,c;czg;testtab;Limit 10;x,y,z;zxj;NewTab;'            
                                                                    //表级迁移参数,格式:'源端查询字段;源端库名;源端表名;源端过滤条件;目的端插入字段;目的端库名;目的端表名;'
                                                                    //如果没有特定条件，可以不写，但必须有分隔符，举例如下：
                                                                    //';czg;testtab;;;zxj;NewTab;'
                                                                    //这样相当于testtab迁移到NewTab，没有任何特殊条件。
                                                                    //这个可以有多个标签，想迁移多少张表就写几个标签。
SpecifiedTab     : ';czg;testtab_copy;;;zxj;testtab_copy;'
SpecifiedTab     : ';czg;czg;;;zxj;czg;'

[Target]                                                            //目的端信息。
ConnInfo         : '192.168.142.12;czg;qwer1234;zxj;5258;utf8;'
MigrationDb      : 'zxj;'                                           //库级迁移参数，单个数据库名长度限制29，需要迁移的数据库名。
```
### （2）Pg -> Gbase8a
```
//*代表不可以空
//单引号包围参数，分号分割参数项
//每行开头不可以有空格，不然会跳过此参数检查。
//SpecifiedTab参数可以有多个。
 
[Tool]                                                           //工具信息。
ProcessNums   : '2;'                                             //*程序迁移时开的进程数。
Level         : '1;'	                                           //*迁移级别。
                                                                 //2:库级迁移，BlackList生效。
                                                                 //1:表级迁移，SpecifiedTab生效。
OsInfo        : '192.168.142.12;gbase;gbase;'                    //*Gbase8a -> Gbase8a LOAD使用。工具所在操作系统IP;操作系统用户;操作系统用户密码;长度同下方的数据库IP;数据库用户名;数据库用户密码;
OneBatchNums  : '80000;'                                         //*一个批次插入的数据条数。（INSERT方式）
SwitchNums    : '2000000;'                                       //*MigrationType为0的情况下，支持此参数，此数以上使用LOAD，以下使用INSERT。
MigrationType : '1;'                                             //*迁移类型，支持0、1。
                                                                 //0 : Gbase8a     -> Gbase8a
                                                                 //1 : PostgreSql  -> Gbase8a
			  
[Source]                                                         //源端信息。
#ConnInfo      : '192.168.142.12;root;;czg;5258;utf8;'
ConnInfo      : '192.168.142.12;postgres;postgres;czg;5432;utf8;' //'IP;数据库用户名;数据库用户密码;数据库名;数据库端口号;数据连接字符集;'
                                                                 //*单个IP长度限制19，数据库IP地址。
                                                                 //*单个用户名长度限制19，数据库用户名。
                                                                 //*单个用户名密码长度限制29，数据库密码。
                                                                 //*单个数据库名长度限制29，数据库名。
                                                                 //*数据库端口。
                                                                 //*长度限制9，数据库连接字符集，支持utf8和gbk。
MigrationDb   : 'zxj;'                                           //库级迁移参数，单个数据库名长度限制29，需要迁移的数据库名。
                                                                 //MigrationType为1的情况下，此为PG的模式名。
BlackList     : ''                                               //库级迁移参数，支持50个表，黑名单，表名，长度限制参考Db。和MigrationDb一起使用可以。        
#SpecifiedTab  : 'a,b,c;czg;testtab;Limit 10;x,y,z;zxj;NewTab;'            
                                                                 //MigrationType为0的情况下，表级迁移参数,格式:'源端查询字段;源端库名;源端表名;源端过滤条件;目的端插入字段;目的端库名;目的端表名;'
                                                                 //MigrationType为1的情况下，表级迁移参数,格式:'源端查询字段;源端模式名;源端表名;源端过滤条件;目的端插入字段;目的端库名;目的端表名;'
                                                                 //如果没有特定条件，可以不写，但必须有分隔符，举例如下：
                                                                 //';czg;testtab;;;zxj;NewTab;'
                                                                 //这样相当于testtab迁移到NewTab，没有任何特殊条件。
                                                                 //这个可以有多个标签，想迁移多少张表就写几个标签。
#SpecifiedTab  : ';czg;testtab_copy;;;zxj;testtab_copy;'
#SpecifiedTab  : ';czg;czg;;;zxj;czg;'

SpecifiedTab  : ';public;testtab;;;zxj;testtab;'
SpecifiedTab  : ';public;students;;;zxj;students;'
SpecifiedTab  : ';public;haha;;;zxj;haha;'

[Target]                                                         //目的端信息。
ConnInfo      : '192.168.142.12;czg;qwer1234;zxj;5258;utf8;'
MigrationDb   : 'zxj;'                                           //库级迁移参数，单个数据库名长度限制29，需要迁移的数据库名。
```
# 九、运行效果
## 1、Gbase8a -> Gbase8a
```
[gbase@czg0 Exec]$ ./HsManager &

[gbase@czg0 Exec]$ tail -100f HappySunshineV1.4.log
2024-08-16 10:44:56-P[50461]-T[50461]-[Info ]-HappySunshineV1.4
2024-08-16 10:44:56-P[50461]-T[50461]-[Info ]-PrintHsMngrSt      :
OsEnv              : /opt/Developer/ComputerLanguageStudy/C/DataStructureTestSrc/PublicFunction/HappySunshine
OsUser             : gbase
OsPwd              : gbase
OsIp               : 192.168.142.12
SrcMigrationDb     : czg
TgtMigrationDb     : 
OneBatchNums       : 50000
SwitchNums         : 2000000
Level              : 1
ProcessNums        : 2
ConnInfo           : [192.168.142.12,root,,czg,5258,utf8] -> [192.168.142.12,czg,qwer1234,zxj,5258,utf8]
PrintSeq           : TaskQ,BlackList,WhiteList.
2024-08-16 10:44:56-P[50461]-T[50461]-[Info ]-StrSqQ             :
Data               : []
FrontIndex         : 0
RearIndex          : 0
SqQueueLen         : 0
2024-08-16 10:44:56-P[50461]-T[50461]-[Info ]-SqQueue Data       :
Data               : [ 'testtab' ]
FrontIndex         : 0
RearIndex          : 1
SqQueueLen         : 1
SqQueueMaxLen      : 50
Flag               : STRING_TYPE_FLAG
2024-08-16 10:44:56-P[50461]-T[50461]-[Info ]-SqQueue Data       :
Data               : [ 'a,b,c;czg;testtab;Limit 10;x,y,z;zxj;NewTab;' ,';czg;testtab_copy;;;zxj;testtab_copy;' ,';czg;czg;;;zxj;czg;' ]
FrontIndex         : 0
RearIndex          : 3
SqQueueLen         : 3
SqQueueMaxLen      : 50
Flag               : STRING_TYPE_FLAG
2024-08-16 10:44:57-P[50465]-T[50475]-[Info ]-Migration Complete : czg.testtab              -> zxj.NewTab              , DataNum :         10 Row, Elapsed Time :     0 s.
2024-08-16 10:44:57-P[50465]-T[50475]-[Info ]-Migration Complete : czg.czg                  -> zxj.czg                 , DataNum :       2050 Row, Elapsed Time :     0 s.
2024-08-16 10:45:22-P[50466]-T[50473]-[Info ]-Migration Complete : czg.testtab_copy         -> zxj.testtab_copy        , DataNum :    1310727 Row, Elapsed Time :    25 s, Efficiency : 52429 Row/s.
2024-08-16 10:46:22-P[50461]-T[50461]-[Info ]-HappySunshine Quit : OK.
```
## 2、Pg -> Gbase8a
```
[gbase@czg2 Exec]$ ./HsManager &
[1] 30081
[gbase@czg2 Exec]$ tail -100f HappySunshineV1.4.log 
2024-09-18 10:45:22-P[30081]-T[30081]-[Info ]-HappySunshineV1.4
2024-09-18 10:45:22-P[30081]-T[30081]-[Info ]-PrintHsMngrSt      :
MigrationType      : 1
OsEnv              : /opt/Developer/ComputerLanguageStudy/C/DataStructureTestSrc/PublicFunction/HappySunshine
OsUser             : gbase
OsPwd              : gbase
OsIp               : 192.168.142.12
SrcDb              : zxj
DestDb             : zxj
OneBatchNums       : 80000
SwitchNums         : 2000000
Level              : 1
ProcessNums        : 2
ConnInfo           : [192.168.142.12,postgres,postgres,czg,5432,utf8] -> [192.168.142.12,czg,qwer1234,zxj,5258,utf8]
PrintSeq           : TaskQ,BlackList,WhiteList.
2024-09-18 10:45:22-P[30081]-T[30081]-[Info ]-StrSqQ             :
Data               : []
FrontIndex         : 0
RearIndex          : 0
SqQueueLen         : 0
2024-09-18 10:45:22-P[30081]-T[30081]-[Info ]-SqQueue Data       :
Data               : []
FrontIndex         : 0
RearIndex          : 0
SqQueueLen         : 0
SqQueueMaxLen      : 50
Flag               : STRING_TYPE_FLAG
2024-09-18 10:45:22-P[30081]-T[30081]-[Info ]-SqQueue Data       :
Data               : [ ';public;testtab;;;zxj;testtab;' ,';public;students;;;zxj;students;' ,';public;haha;;;zxj;haha;' ]
FrontIndex         : 0
RearIndex          : 3
SqQueueLen         : 3
SqQueueMaxLen      : 50
Flag               : STRING_TYPE_FLAG
2024-09-18 10:45:22-P[30084]-T[30111]-[Info ]-PgPkSplit2Q        : OK, Schema : public, Tab : testtab, SplitNums : 2.
2024-09-18 10:45:22-P[30083]-T[30118]-[Info ]-Migration Complete : public.students             -> zxj.students            , DataNum :          0 Row, Elapsed Time :    0.001 s.
2024-09-18 10:45:22-P[30083]-T[30118]-[Info ]-Migration Complete : public.haha                 -> zxj.haha                , DataNum :          0 Row, Elapsed Time :    0.001 s.
2024-09-18 10:46:01-P[30083]-T[30118]-[Info ]-Migration Complete : public.testtab              -> zxj.testtab             , DataNum :    1049631 Row, Elapsed Time :   38.782 s, Efficiency : 27621 Row/s.
2024-09-18 10:46:10-P[30084]-T[30111]-[Info ]-Migration Complete : public.testtab              -> zxj.testtab             , DataNum :    1048608 Row, Elapsed Time :   47.605 s, Efficiency : 22310 Row/s.
2024-09-18 10:47:10-P[30081]-T[30081]-[Info ]-HappySunshine Quit : OK.
```
我这边测试的PUBLIC.TESTTAB表包含一个整型主键，所以进行了数据切分，效率在48000 Row/s左右，因为我这边虚机资源有限，如果大家有条件的话，最好PG、GBASE8A、HappySunshine分别放在三台机器上运行，这时大家可以加大进程数，看一下单表（包含一个整型主键）的效果，应该会更快一些。

# 十、性能测试
测试将czg库下的testtab_copy迁移到zxj库下。

## 1、Gbase8a -> Gbase8a
### （1）测试数据
```
gbase> select count(*) from czg.testtab_copy;
+----------+
| count(*) |
+----------+
|  1310720 |
+----------+
1 row in set (Elapsed: 00:00:00.00)

gbase> select * from czg.testtab_copy limit 10;
+------+------+------+--------------------+---------------------+--------------------------+------------+---------------------+
| a    | b    | c    | d                  | e                   | f                        | g          | h                   |
+------+------+------+--------------------+---------------------+--------------------------+------------+---------------------+
|    1 |  1.1 | czg  | 快乐的小天使       | qwertasdsdfzxczxxv  | gregergjsfishfuieehfuiew | 1995-09-18 | 2022-08-03 09:24:00 |
|    1 |  1.1 | czg  | 快乐的小天使       | qwertasdsdfzxczxxv  | gregergjsfishfuieehfuiew | 1995-09-18 | 2022-08-03 09:24:00 |
|    1 |  1.1 | czg  | 快乐的小天使       | qwertasdsdfzxczxxv  | gregergjsfishfuieehfuiew | 1995-09-18 | 2022-08-03 09:24:00 |
|    1 |  1.1 | czg  | 快乐的小天使       | qwertasdsdfzxczxxv  | gregergjsfishfuieehfuiew | 1995-09-18 | 2022-08-03 09:24:00 |
|    1 |  1.1 | czg  | 快乐的小天使       | qwertasdsdfzxczxxv  | gregergjsfishfuieehfuiew | 1995-09-18 | 2022-08-03 09:24:00 |
|    2 |  1.1 | czg  | 快乐的小天使       | qwertasdsdfzxczxxv  | gregergjsfishfuieehfuiew | 1995-09-18 | 2022-08-03 09:24:00 |
|    2 |  4.1 | czg  | 快乐的小天使       | qwertasdsdfzxczxxv  | gregergjsfishfuieehfuiew | 1995-09-18 | 2022-08-03 09:24:00 |
|    2 |  4.1 | czg  | 快乐的小天使       | qwertasdsdfz\xczxxv | gregergjsfishfuieehfuiew | 1995-09-18 | 2022-08-03 09:24:00 |
|    2 |  4.1 | czg  | 快乐的小天使       | qwertasdsdfz.xczxxv | gregergjsfishfuieehfuiew | 1995-09-18 | 2022-08-03 09:24:00 |
|    2 |  4.1 | czg  | 快乐的小天使       | qwertasdsdfz.xczxxv | gregergjsfishfuieehfuiew | 1995-09-18 | 2022-08-03 09:24:00 |
+------+------+------+--------------------+---------------------+--------------------------+------------+---------------------+
10 rows in set (Elapsed: 00:00:00.31)

gbase> desc czg.testtab_copy;
+-------+--------------+------+-----+-------------------+-----------------------------+
| Field | Type         | Null | Key | Default           | Extra                       |
+-------+--------------+------+-----+-------------------+-----------------------------+
| a     | int(11)      | YES  |     | NULL              |                             |
| b     | double       | YES  |     | NULL              |                             |
| c     | varchar(100) | YES  |     | NULL              |                             |
| d     | text         | YES  |     | NULL              |                             |
| e     | blob         | YES  |     | NULL              |                             |
| f     | longblob     | YES  |     | NULL              |                             |
| g     | date         | YES  |     | NULL              |                             |
| h     | timestamp    | NO   |     | CURRENT_TIMESTAMP | on update CURRENT_TIMESTAMP |
+-------+--------------+------+-----+-------------------+-----------------------------+
8 rows in set (Elapsed: 00:00:00.00)
```
### （2）性能数据表格
工具名|每秒条数|方式|环境
--|--|--|--
HappySunshine|48545|INSERT|本机Linux虚拟机
| |87381|LOAD|

### （3）HappySunshine截图
#### INSERT

![INSERT](https://github.com/lxgczg/HappySunshine/blob/main/Photo/INSERT.png)

#### LOAD

![LOAD](https://github.com/lxgczg/HappySunshine/blob/main/Photo/LOAD.png)
​
