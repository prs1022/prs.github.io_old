# datalink简介
datalink是一个数据自助查询平台，可以通过托拉拽操作进行简单的数据查询，并预览、下载数据
[操作手册](https://thoughts.teambition.com/share/5eba5777d9ba2e001a34dedc)

# 开发部署
测试和上线使用阿里云云效流水线部署

本地开发
后端 python server.py 启动后端主进程
celery -A task worker --loglevel=DEBUG -P eventlet 启动worker异步任务进程 
> windows下需要加上-P eventlet
前端启用nginx

python版本 3.6 
> windows下开发需要强调用3.6版本,需要安装ldap.whl

# 查询流程
自助查询和SQL查询同理
1. 前端拖拽维度、度量后发起查询 /query
返回 task_id ,查询记录加到任务列表
2. 后端开启celery task查询，查询结果更新到任务列表
3. 前端轮询 任务列表，获取查询结果


# 导出数据
导出数据到csv，直接从ftp下载excel。ftp上的excel文件为查询结束后上传。

# 上传建表
存在用户自己会将excel上传作为临时表的情况，将上传的文件通过pandas读取为df，再转为二维表结构，
循环写入到odps表当中。上传文件较大时等待时间较长，所以整个过程也是异步。

# 查询、上传进度
由于查询和上传都是异步行为，有时候不知道任务的执行进度到哪里了，所以加上了进度。
进度的实现依赖于读写redis，由于datalink主进程和celery进程分别运行在两个容器中，而上传的文件需要从datalink
主进程传递到celery进程，通过参数方式需要序列化，速度较慢，于是通过docker映射外部文件的方式，在dl和celery的两个容器
分别将文件存储目录映射到宿主机的/tmp/datalink目录下。（生产就一个节点）

过程较慢的地方在于循环读取数据到二维表，所以这个给了虚拟进度10%，建表速度快就给5%，通过redis add实现。
剩余的 85%按写入odps表每隔5000条的方式算一次百分比分配。每次add的结果都会在前端轮询任务列表的时候读取到。


# 问题
- 发起查询后，任务提交到odps，会出现一直排队得不到执行

- 导出文件中含有多位数字，例如商户号十几位的数字，末尾几位都是0，精度丢失的问题

# odps查询失败的几种情况
> 本周上线了添加logview 和 子任务状态判断的代码，重点关注线上失败的任务类型

  截止目前罗列如下几种:

- 执行时间过长，超过半小时，实际一小时+能查询出来


- Semantic analysis exception - column acctdate cannot be resolved;诸如此类语法错误

- sub_task:AnonymousSQLTask,status:FAILED 子任务执行失败， 通过logview 查看具体原因 physical plan generation failed: java.lang.RuntimeException: Table(hyq_prd,ods_supay_supay_ord_log) is full scan with all partitions, please specify partition predicates. 好多都是这种情况，属于没有加分区就查询的SQL。对应的SQL如下
```
select * from hyq_prd.ods_supay_supay_ord_log
where member_id in ('3100146111773144')
and trans_type='2001'
and trans_stat='S';

```





