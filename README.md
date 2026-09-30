# 一、环境信息
名称 | 值
---- | -----
CPU	| X86
操作系统	| CentOS Linux release 8.5.2111
内存	| 5G
逻辑核数	| 6
HappySunshine版本|V1.10
Kettle版本|pdi-ce-9.5.0.1-261
Gbase8a版本|8.6.2-R43.34.27468a27
Pg版本|PostgreSQL 14.5、14.24
DM版本|V8、V9

# 二、简述
![HS Logo](https://github.com/lxgczg/HappySunshine/blob/main/Photo/HappySunshine.png)
<br>HappySunshine数据库迁移工具是由C语言编写的多进程多线程程序，支持多种数据库之间的高效数据同步、数据离线（数据库宕机）抽取等，安装简便、简单配置即可使用，功能还在逐步完善中（其实是还在陆续补充新知识），有什么好的建议，大家可以在评论或私信告知。

# 三、架构图
## 1、离线（数据库宕机）抽取
![HS](https://github.com/lxgczg/HappySunshine/blob/main/Photo/HsPgUnload1.png)
PG数据离线抽取功能是一个多线程程序，解析流程如下：

1、读取 pg_filenode.map

2、解析 pg_type

3、解析 pg_class

4、解析 pg_namespace

5、解析 pg_attribute

6、解析 pg_enum

7、解析 pg_attrdef

8、解析 pg_sequence

9、并行解析用户表

10、数据落地成 CSV文件，并生成序列定义、表定义、COPY语句

## 2、在线迁移
![HS](https://i-blog.csdnimg.cn/blog_migrate/69fb8c4716412dde87e34b76eadb2bb8.png)

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

12、消费者线程将任务包解开，将数据清洗，并放入自己的缓冲区中，清洗完再把INSERT命令发给数据库。

13、消费者线程执行任务失败，将任务状态置为错误，待生产者线程发现任务失败后，终止任务下发给线程池队列，重新到共享存储中的任务队列拿取任务。

14、消费者线程执行任务成功，就继续执行。

15、管理者进程将任务发送完成之后，将任务结束标记放入共享存储中的任务队列中。

16、生产者线程从共享存储中的任务队列中拿到任务结束标记后。等待屏障，也就是等待消费者线程完成所拿到的任务。

17、执行者进程等待所有的消费者、生产者结束后，回收线程池资源，释放自身所占用资源，结束退出。

18、管理者进程等待所有执行者进程结束后，回收进程资源，释放自身所占用资源，结束退出。

# 四、升级点
序号|名称|备注
-- | ----- | ------ 
1	|支持表数据非易失默认值。	                            |PG数据离线抽取功能。
2	|支持pg_attrdef字典表解析。	                            |PG数据离线抽取功能。
3	|单库、模式级别，部分场景下表漏扫BUG修复。	            |PG数据离线抽取功能。
4	|支持pg_sequence字典表解析。	                        |PG数据离线抽取功能。
5	|支持序列定义的语句生成。	                            |PG数据离线抽取功能。
6	|字符串数据包含字段分隔符时，数据加载报错BUG修复。	    |PG数据离线抽取功能。
7	|支持 glibc 2.22 及以上版本的 x86_64 Linux 操作系统。	|PG数据离线抽取功能。
8	|支持序列当前值设置语句生成。	                        |PG数据离线抽取功能。
9	|插入大字段到达梦时，内存泄漏BUG修复。	                |在线迁移功能。
10	|PG主键切分字段名解析错误BUG修复	                    |在线迁移功能。
11	|PG->DM映射类型添加。	                                |在线迁移功能。

# 五、支持功能
## 1、支持功能
| 序号 | 类别 | 功能模块 | 说明/备注 |
|------|------|----------|-----------|
| 1 | 在线迁移 | Gbase8a 到 8a 数据迁移 | INSERT、LOAD 方式迁移，表定义及其他暂不支持。 |
| 2 |          | PG 到 Gbase8a 数据迁移 | INSERT 方式迁移，表定义及其他暂不支持。 |
| 3 |          | PG 到 DM 数据迁移 | INSERT 方式迁移，表定义及其他暂不支持。 |
| 4 |          | 多线程并发加载单表数据 | |
| 5 |          | 多进程并发加载单表数据 | 源端库为 PG，迁移表包含单一整型主键，进程数设置大于 1 时，支持此项。 |
| 6 | 离线抽取 | PG 数据离线抽取支持表、模式、单库级 | 抽取底层数据文件数据落地为 COPY 文本。 |
| 7  |         | PG 47种数据类型离线抽取             | 47种数据类型、非易失默认值。具体类型见下文的支持列表。             |
| 8  |         | PG 普通表定义抽取                   | 47种数据类型、非空、非易失默认值。     |
| 9  |         | PG 普通表单表并行解析               |                                        |
| 10 |         | PG COPY 语句自动生成                |                                        |
| 11 |         | PG 支持版本 14                      | 验证版本为14.5、14.24，PG大版本一致情况下，小版本间底层存储无改动。 |

## 2、离线抽取支持类型
### （1）数字类型
名字|是否支持
------------------------------------------ | -----
smallint	| √
integer	| √
bigint	| √
decimal	| √
numeric	| √
real	| √
double precision	| √
smallserial	| √
serial	| √
bigserial	| √
### （2）货币类型
名字|是否支持
------------------------------------------ | -----
money	| √
### （3）字符类型
名字|是否支持
------------------------------------------ | -----
character varying(n), varchar(n)	| √
character(n), char(n)	| √
text	| √
### （4）特殊字符类型
名字|是否支持
------------------------------------------ | -----
"char"	| √
name	| √
### （5）二进制数据类型
名字|是否支持
------------------------------------------ | -----
bytea	| √
### （6）日期/时间类型
名字|是否支持
------------------------------------------ | -----
timestamp [ (p) ] [ without time zone ]	| √
timestamp [ (p) ] with time zone	| √
date	| √
time [ (p) ] [ without time zone ]	| √
time [ (p) ] with time zone	| √
interval [ fields ] [ (p) ]	| √
### （7）布尔数据类型
名字|是否支持
------------------------------------------ | -----
boolean	| √
### （8）枚举类型
名字|是否支持
------------------------------------------ | -----
enum	| √
### （9）几何类型
名字|是否支持
------------------------------------------ | -----
point	| √
line	| √
lseg	| √
box	| √
path（封闭路径）	| √
path（开放路径）	| √
polygon	| √
circle	| √
### （10）网络地址类型
名字|是否支持
------------------------------------------ | -----
cidr	| √
inet	| √
macaddr	| √
macaddr8	| √
### （11）位串类型
名字|是否支持
------------------------------------------ | -----
bit	| √
bit varying	| √
### （12）文本搜索类型
名字|是否支持
------------------------------------------ | -----
tsvector	
tsquery	
### （13）UUID类型
名字|是否支持
------------------------------------------ | -----
UUID	| √
### （14）XML类型
名字|是否支持
------------------------------------------ | -----
XML	| √
### （15）JSON类型
名字|是否支持
------------------------------------------ | -----
JSON	| √
JSONB	| √
JSONPATH	| √
### （16）数组类型
内部实现支持N维，只验证了以下基础类型，其他基础类型也是支持的，只是未测试。

名字|是否支持
------------------------------------------ | -----
INT[]	| √
VARCHAR[]	| √
### （17）组合类型
名字|是否支持
------------------------------------------ | -----
|
### （18）范围类型
名字|是否支持
------------------------------------------ | -----
|
### （19） 域类型
名字|是否支持
------------------------------------------ | -----
|
### （20） 对象标识符类型
名字|是否支持
------------------------------------------ | -----
OID	| √
XID	| √
### （21） pg_lsn 类型
名字|是否支持
------------------------------------------ | -----
pg_lsn	

# 六、后续计划支持功能
展望是挺多的，奈何时间不多呀。
序号|功能|备注
-- | ----- | ------ 
1|支持PG单表并行离线抽取	
2|支持Oracle数据迁移	
3|支持达梦快速装载	
4|支持信号处理	
5|支持DM 切分数据	
6|支持PG 切分数据	
7|支持目的端为PG	

# 七、安装包下载地址
[GITHUB-HappySunshine-release版下载地址](https://github.com/lxgczg/HappySunshine/releases)

# 八、配置参数介绍
## 1、离线抽取
序号|参数|备注
-- | ----- | ------ 
1	|PG数据库目录	|/opt/Pg14-5/Data/base/13892/
2	|数据落地目录	|/home/czg/TestPgData/
3	|PG数据块大小	|8192
4	|模式名	        |指定模式名，'*'代表所有模式。
5	|表名	        |指定表名，'*'代表所有表。
6	|线程数	        |例如3，支持1 - N。
7	|日志级别	    |例如2，建议为2，支持0 - 4。

## 2、在线迁移
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
vim /home/lzl/.bashrc 添加如下内容：
```
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
```
### （2）G8aExecutor 
```
[lzl@czg0 ~]$ ldd HappySunshine/Exec/G8aExecutor 
```
### （3）Pg2G8aExecutor
```
[lzl@czg0 ~]$ ldd HappySunshine/Exec/Pg2G8aExecutor 
```
### （4）Pg2DmExecutor 
```
[lzl@czg0 ~]$ ldd HappySunshine/Exec/Pg2DmExecutor 
```
### （5）HsPgUnload
```
[lzl@czg0 ~]$ ldd HappySunshine/Exec/HsPgUnload 
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

# 十、在线迁移性能对比测试
和开源ETL工具Kettle进行对比测试。<br>由于本人测试条件有限（所有的数据库和工具都部署在一个虚机里），如果大家有条件可以在性能更好的环境下测试，迁移效率肯定会比下面的结果更好。
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
测试步骤大家可以参考之前写的博客《[Kettle-学习-03-PostgreSql迁移至达梦DM](https://blog.csdn.net/qq_45111959/article/details/144443435?spm=1001.2014.3001.5501)》
### （2）PostgreSql -> Gbase8a
![INSERT](https://i-blog.csdnimg.cn/direct/2e3415071d364de8af997cdbcf642fee.png)
测试步骤大家可以参考之前写的博客《[Kettle-学习-02-PostgreSql迁移至Gbase8a](https://blog.csdn.net/qq_45111959/article/details/144442657?spm=1001.2014.3001.5501)》
### （3）Gbase8a-> Gbase8a
![INSERT](https://i-blog.csdnimg.cn/direct/194d48037d15427f99f9b481b94164c1.png)
测试步骤大家可以参考之前写的博客《[Kettle-学习-01-Gbase8a迁移至Gbase8a](https://blog.csdn.net/qq_45111959/article/details/144428396?spm=1001.2014.3001.5502)》

# 十一、离线抽取功能介绍
## 1、功能展示
### （1）单库级抽取示例
```
[root@dw01:/opt/Developer/ComputerLanguageStudy/C/DataStructureTestSrc/PublicFunction/PgReadData/Exec]# ./HsPgUnload /opt/Pg14-5/Data/base/13892/ /home/czg/TestPgData/ 8192 '*' '*' 3 2
```

### （2）模式级抽取示例
```
[root@dw01:/opt/Developer/ComputerLanguageStudy/C/DataStructureTestSrc/PublicFunction/PgReadData/Exec]# ./HsPgUnload /opt/Pg14-5/Data/base/13892/ /home/czg/TestPgData/ 8192 'public' '*' 3 2
```

### （3）表级抽取示例
```
[root@dw01:/opt/Developer/ComputerLanguageStudy/C/DataStructureTestSrc/PublicFunction/PgReadData/Exec]# ./HsPgUnload /opt/Pg14-5/Data/base/13892/ /home/czg/TestPgData/ 8192 'public' 'blue' 3 2
```

### （4）恢复数据示例
```
psql -d sun -f /home/sun/TestPgData/PG_SEQ_DDL.txt
psql -d sun -f /home/sun/TestPgData/PG_TAB_DDL.txt
psql -d sun -f /home/sun/TestPgData/PG_COPY.txt
```

### （5）COPY语句展示
```
[root@dw01:/opt/Developer/ComputerLanguageStudy/C/DataStructureTestSrc/PublicFunction/PgReadData/Exec]# head /home/czg/TestPgData/PG_COPY.txt 

COPY public.actor
    FROM '/home/czg/TestPgData//public_actor/public_actor.txt_0'
    WITH(
        FORMAT    CSV,
        DELIMITER '|',
        NULL      '',
        QUOTE     '"',
        ESCAPE    '\');
```

### （6）生成的表定义展示
```

[root@dw01:/opt/Developer/ComputerLanguageStudy/C/DataStructureTestSrc/PublicFunction/PgReadData/Exec]# cat /home/czg/TestPgData/PG_DDL.txt
CREATE TABLE public.pgbench_accounts 
(
"aid" INT NOT NULL,
"bid" INT,
"abalance" INT,
"filler" CHAR(84)
);
```

### （7）生成的数据展示
```
[root@localhost ~]# tail -n 10 /home/sun/TestPgData/PgUserData/public_pgbench_accounts/public_pgbench_accounts.txt_0 | sort -t'|' -k2,2n
95560|1|3374|                                                                                    
252675|3|-3984|                                                                                    
411875|5|-2409|                                                                                    
485119|5|-2401|                                                                                    
719962|8|-855|                                                                                    
765817|8|3318|                                                                                    
789713|8|895|                                                                                    
867879|9|1560|                                                                                    
870042|9|-4308|                                                                                    
970012|10|3886| 
```

### （8）库内查询展示对比
```
postgres=# select * from pgbench_accounts where aid in (970012,867879,252675,485119,411875,789713,95560,719962,765817,870042);
  aid   | bid | abalance |                                        filler                                        
--------+-----+----------+--------------------------------------------------------------------------------------
  95560 |   1 |     3374 |                                                                                     
 252675 |   3 |    -3984 |                                                                                     
 411875 |   5 |    -2409 |                                                                                     
 485119 |   5 |    -2401 |                                                                                     
 719962 |   8 |     -855 |                                                                                     
 765817 |   8 |     3318 |                                                                                     
 789713 |   8 |      895 |                                                                                     
 867879 |   9 |     1560 |                                                                                     
 870042 |   9 |    -4308 |                                                                                     
 970012 |  10 |     3886 |                                                                                     
(10 rows)
```

### （9）生成的序列展示
```
[root@localhost Exec]# head -25 /home/sun/TestPgData/PG_SEQ_DDL.txt 
CREATE SEQUENCE "public"."actor_actor_id_seq" 
AS BIGINT
INCREMENT BY 1
MINVALUE 1
MAXVALUE 9223372036854775807
START WITH 1
CACHE 1
NO CYCLE;

SELECT SETVAL('public.actor_actor_id_seq', 200, 'T');

CREATE SEQUENCE "public"."address_address_id_seq" 
AS BIGINT
INCREMENT BY 1
MINVALUE 1
MAXVALUE 9223372036854775807
START WITH 1
CACHE 1
NO CYCLE;

SELECT SETVAL('public.address_address_id_seq', 10, 'T');
```

## 2、性能展示
### （1）表结构
```
postgres=# \d pgbench_accounts
              Table "public.pgbench_accounts"
  Column  |     Type      | Collation | Nullable | Default 
----------+---------------+-----------+----------+---------
 aid      | integer       |           | not null | 
 bid      | integer       |           |          | 
 abalance | integer       |           |          | 
 filler   | character(84) |           |          | 
Indexes:
    "pgbench_accounts_pkey" PRIMARY KEY, btree (aid)
```

### （2）数据量
```
postgres=# SELECT COUNT(*) FROM pgbench_accounts;
  count  
---------
 1000000
(1 row)
```

### （3）单线程离线抽取
```
[root@dw01:/opt/Developer/ComputerLanguageStudy/C/DataStructureTestSrc/PublicFunction/PgReadData/Exec]# ./HsPgUnload /opt/Pg14-5/Data/base/13892/ /home/czg/TestPgData/ 8192 'public' 'pgbench_accounts' 1 2
2026-06-23 20:58:06.112844-P[64814]-T[64814]-[Info ]-HsLogo             : 
 █░  █░  ███░  ████░  ████░  █░    █░  ███░  █░  █░ █░  █░  ███░  █░  █░  █░ █░  █░ █████░
 █░  █░ █░  █░ █░  █░ █░  █░  █░  █░  █░  █░ █░  █░ ██░ █░ █░  █░ █░  █░  █░ ██░ █░ █░    
 █░  █░ █░  █░ █░  █░ █░  █░   █░█░   █░     █░  █░ █░█░█░ █░     █░  █░  █░ █░█░█░ █░    
 █████░ █████░ ████░  ████░     █░     ███░  █░  █░ █░ ██░  ███░  █████░  █░ █░ ██░ ████░ 
 █░  █░ █░  █░ █░     █░        █░        █░ █░  █░ █░  █░     █░ █░  █░  █░ █░  █░ █░    
 █░  █░ █░  █░ █░     █░        █░    █░  █░ █░  █░ █░  █░ █░  █░ █░  █░  █░ █░  █░ █░    
 █░  █░ █░  █░ █░     █░        █░     ███░   ███░  █░  █░  ███░  █░  █░  █░ █░  █░ █████░
 
Contact Information:
     1.QQ     : 2263143197 
     2.WeChat : Ldqczgsun 
     3.Email  : 2263143197@qq.com 
     4.Github : https://github.com/lxgczg/HappySunshine 
     5.CSDN   : https://blog.csdn.net/qq_45111959?type=lately 

2026-06-23 20:58:06.114343-P[64814]-T[64814]-[Info ]-HsLicCheck         : OK, Version : 'HappySunshineV1.7', Flag : 'Pro', ExpireTime : '2027-05-17'.
2026-06-23 20:58:06.114373-P[64814]-T[64814]-[Info ]-main               : OK, DataDir : '/opt/Pg14-5/Data/base/13892/', UnloadDir : '/home/czg/TestPgData/', PageSize : 8192, Sch : 'public', Tab : 'pgbench_accounts', WkrNums : 1, LOG_LEVEL : 'Info '.
2026-06-23 20:58:06.114863-P[64814]-T[64814]-[Info ]-PgGlbEnvInit       : OK, WritePages : 2, WriteBlockSize : 8192, PgSchMark : BT, PgClassMark : BT, PgAttrMark : BT, PgEnumMark : BT, PgTypeMark : BT.
2026-06-23 20:58:06.140936-P[64814]-T[64814]-[Info ]-PgFileNodeMap      : OK, FilePath : '/opt/Pg14-5/Data/base/13892/pg_filenode.map', num_mappings : 17, PgClassOid : 171368, PgAttrOid : 170960, PgTypeOid : 171261.
2026-06-23 20:58:06.145296-P[64814]-T[64814]-[Info ]-PgTypeDecode       : OK, Sch : pg_catalog     , Tab : pg_type             , Time :     0.000 (s.ms).
2026-06-23 20:58:06.146163-P[64814]-T[64814]-[Info ]-PgClassDecode      : OK, Sch : pg_catalog     , Tab : pg_class            , Time :     0.000 (s.ms).
2026-06-23 20:58:06.146242-P[64814]-T[64814]-[Info ]-PgSchDecode        : OK, Sch : pg_catalog     , Tab : pg_namespace        , Time :     0.000 (s.ms).
2026-06-23 20:58:06.152375-P[64814]-T[64814]-[Info ]-PgAttrDecode       : OK, Sch : pg_catalog     , Tab : pg_attribute        , Time :     0.006 (s.ms).
2026-06-23 20:58:06.152603-P[64814]-T[64814]-[Info ]-PgEnumDecode       : OK, Sch : pg_catalog     , Tab : pg_enum             , Time :     0.000 (s.ms).
2026-06-23 20:58:06.152658-P[64814]-T[64814]-[Info ]-CllPrint           : 
[ ('public' ,'pgbench_accounts' ) ]
2026-06-23 20:58:06.152741-P[64814]-T[64814]-[Info ]-PgTabDef           : OK, Sch : public         , Tab : pgbench_accounts    , Time :     0.000 (s.ms).
2026-06-23 20:58:07.426099-P[64814]-T[64814]-[Info ]-PgTabDecode        : OK, Sch : public         , Tab : pgbench_accounts    , Time :     1.273 (s.ms).
2026-06-23 20:58:08.126278-P[64814]-T[64814]-[Info ]-main               : OK, Task completed, Time :     2.013 (s.ms).
```
解析需1.273 (s.ms)。

### （4）多线程离线抽取
```
[root@dw01:/opt/Developer/ComputerLanguageStudy/C/DataStructureTestSrc/PublicFunction/PgReadData/Exec]# ./HsPgUnload /opt/Pg14-5/Data/base/13892/ /home/czg/TestPgData/ 8192 'public' 'pgbench_accounts' 4 2
2026-06-23 20:59:54.347337-P[65092]-T[65092]-[Info ]-HsLogo             : 
 █░  █░  ███░  ████░  ████░  █░    █░  ███░  █░  █░ █░  █░  ███░  █░  █░  █░ █░  █░ █████░
 █░  █░ █░  █░ █░  █░ █░  █░  █░  █░  █░  █░ █░  █░ ██░ █░ █░  █░ █░  █░  █░ ██░ █░ █░    
 █░  █░ █░  █░ █░  █░ █░  █░   █░█░   █░     █░  █░ █░█░█░ █░     █░  █░  █░ █░█░█░ █░    
 █████░ █████░ ████░  ████░     █░     ███░  █░  █░ █░ ██░  ███░  █████░  █░ █░ ██░ ████░ 
 █░  █░ █░  █░ █░     █░        █░        █░ █░  █░ █░  █░     █░ █░  █░  █░ █░  █░ █░    
 █░  █░ █░  █░ █░     █░        █░    █░  █░ █░  █░ █░  █░ █░  █░ █░  █░  █░ █░  █░ █░    
 █░  █░ █░  █░ █░     █░        █░     ███░   ███░  █░  █░  ███░  █░  █░  █░ █░  █░ █████░
 
Contact Information:
     1.QQ     : 2263143197 
     2.WeChat : Ldqczgsun 
     3.Email  : 2263143197@qq.com 
     4.Github : https://github.com/lxgczg/HappySunshine 
     5.CSDN   : https://blog.csdn.net/qq_45111959?type=lately 

2026-06-23 20:59:54.348102-P[65092]-T[65092]-[Info ]-HsLicCheck         : OK, Version : 'HappySunshineV1.7', Flag : 'Pro', ExpireTime : '2027-05-17'.
2026-06-23 20:59:54.348126-P[65092]-T[65092]-[Info ]-main               : OK, DataDir : '/opt/Pg14-5/Data/base/13892/', UnloadDir : '/home/czg/TestPgData/', PageSize : 8192, Sch : 'public', Tab : 'pgbench_accounts', WkrNums : 4, LOG_LEVEL : 'Info '.
2026-06-23 20:59:54.349238-P[65092]-T[65092]-[Info ]-PgGlbEnvInit       : OK, WritePages : 2, WriteBlockSize : 8192, PgSchMark : BT, PgClassMark : BT, PgAttrMark : BT, PgEnumMark : BT, PgTypeMark : BT.
2026-06-23 20:59:54.373733-P[65092]-T[65092]-[Info ]-PgFileNodeMap      : OK, FilePath : '/opt/Pg14-5/Data/base/13892/pg_filenode.map', num_mappings : 17, PgClassOid : 171368, PgAttrOid : 170960, PgTypeOid : 171261.
2026-06-23 20:59:54.377481-P[65092]-T[65092]-[Info ]-PgTypeDecode       : OK, Sch : pg_catalog     , Tab : pg_type             , Time :     0.000 (s.ms).
2026-06-23 20:59:54.378638-P[65092]-T[65092]-[Info ]-PgClassDecode      : OK, Sch : pg_catalog     , Tab : pg_class            , Time :     0.001 (s.ms).
2026-06-23 20:59:54.378743-P[65092]-T[65092]-[Info ]-PgSchDecode        : OK, Sch : pg_catalog     , Tab : pg_namespace        , Time :     0.000 (s.ms).
2026-06-23 20:59:54.383320-P[65092]-T[65092]-[Info ]-PgAttrDecode       : OK, Sch : pg_catalog     , Tab : pg_attribute        , Time :     0.004 (s.ms).
2026-06-23 20:59:54.383667-P[65092]-T[65092]-[Info ]-PgEnumDecode       : OK, Sch : pg_catalog     , Tab : pg_enum             , Time :     0.000 (s.ms).
2026-06-23 20:59:54.383737-P[65092]-T[65092]-[Info ]-CllPrint           : 
[ ('public' ,'pgbench_accounts' ) ]
2026-06-23 20:59:54.383857-P[65092]-T[65092]-[Info ]-PgTabDef           : OK, Sch : public         , Tab : pgbench_accounts    , Time :     0.000 (s.ms).
2026-06-23 20:59:54.732528-P[65092]-T[65092]-[Info ]-PgTabDecode        : OK, Sch : public         , Tab : pgbench_accounts    , Time :     0.348 (s.ms).
2026-06-23 20:59:56.741239-P[65092]-T[65092]-[Info ]-main               : OK, Task completed, Time :     2.393 (s.ms).
```
解析需0.348 (s.ms)。

### （5）包含大字段、枚举表性能对比
线程/项|包含大字段、枚举(s.ms)	|提升倍数
--- | --- | ---
单线程	        |4.472	|2.4
多线程（3线程）	|1.830  |2.4

### （6）不包含大字段、枚举表性能对比
线程/项|不包含大字段、枚举(s.ms)	|提升倍数
--- | --- | ---
单线程	        |1.273	|3.6
多线程（3线程）	|0.348  |3.6

# 十二、许可证
版本|限制
--- | ---
Free	|1、单表数据文件大小小于100MB。<br>2、线程数最多支持2。<br>3、免费使用一年，续期请联系作者。
Pro	    |功能上无限制，需相应许可请联系作者。

# 十三、需求&Bug
需提供如下内容：

序号|名称|必要性
--- | --- | ---
1	|操作系统版本	    |必要
2	|CPU型号	        |必要
3	|HappySunshine版本	|必要
4	|数据库版本	        |必要
5	|问题 \| 需求	    |必要
6	|Bug复现方法	    |Bug必要
7	|报错截图	        |Bug必要
8	|core文件	        |Bug必要

# 十四、联系方式
名称 | 描述
---- | -----
WeChat     | Ldqczgsun
Email      | 2263143197@qq.com
CSDN Blog  | [阳光九叶草LZL](https://blog.csdn.net/qq_45111959?type=blog)
QQ         | 2263143197 
