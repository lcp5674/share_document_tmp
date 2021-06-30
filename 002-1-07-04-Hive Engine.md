# 1、Hive
[Hive Wiki](https://cwiki.apache.org/confluence/display/Hive/Home#Home-UserDocumentation)
## 1.1、Plan
![](https://gitee.com/PeterOtter/yu-que-sync-image/raw/master/20210629010356.png#align=left&display=inline&height=728&id=piJHn&margin=%5Bobject%20Object%5D&originHeight=728&originWidth=1192&status=done&style=none&width=1192)
**大致流程：**
	1、客户端连接到HS2(HiveServer2，目前大多数通过beeline形式连接，Hive Cli模式相对较重，且直接略过授权访问元数据),建立会话
	2、提交sql，通过Driver进行编译、解析、优化逻辑计划，生成物理计划
	3、对物理计划进行优化，并提交到执行引擎进行计算
	4、返回结果
**细节流程：**
	1、客户端和HiveServer2建立连接，创建会话
	2、提交查询或者DDL，转交到Driver进行处理
	3、Driver内部会通过Parser对语句进行解析，校验语法是否正确
	4、然后通过Compiler编译器对语句进行编译，生成AST Tree
	5、SemanticAnalyzer会遍历AST Tree，进一步进行语义分析，这个时候会和Hive MetaStore进行通信获取Schema信息,抽象成QueryBlock，逻辑计划生成器会遍历QueryBlock，翻译成Operator（计算抽象出来的算子）生成OperatorTree,这个时候是未优化的逻辑计划
	6、Optimizer会对逻辑计划进行优化，如进行谓词下推、常量值替换、列裁剪等操作，得到优化后的逻辑计划。
	7、SemanticAnalyzer会对逻辑计划进行处理，通过TaskCompiler生成物理执行计划TaskTree。
	8、TaskCompiler会对物理计划进行优化，然后根据底层不同的引擎进行提交执行。
![](https://gitee.com/PeterOtter/yu-que-sync-image/raw/master/20210629010912.png#align=left&display=inline&height=726&id=kXx26&margin=%5Bobject%20Object%5D&originHeight=726&originWidth=490&status=done&style=none&width=490)

### 1.1.1、Analyze Sql


**语法：**


```sql
EXPLAIN [EXTENDED|CBO|AST|DEPENDENCY|AUTHORIZATION|LOCKS|VECTORIZATION|ANALYZE] query
```

**版本支持：**
	Hive0.14.0支持AUTHORIZATION；[HIVE-5961]
	Hive2.3.0支持VECTORIZATION;[HIVE-11394]
	Hive3.2.0支持LOCKS;[HIVE-17683]

Explain结果总共分为三个部分：
	1、对应查询的抽象语法树 AST
	2、每个计划阶段Stage之间的依赖关系
	3、每个计划阶段的描述（可能是map/reduce,也可能是操作元数据或者文件操作）
**聚合操作分析示例:**


```sql
EXPLAIN FROM src INSERT OVERWRITE TABLE dest_g1 SELECT src.key, sum(substr(src.value,4)) GROUP BY src.key;
```


**聚合操作分析输出信息：**


```sql
STAGE DEPENDENCIES: --每个Stage之间的依赖关系
  Stage-1 is a root stage --Stage1是根阶段
  Stage-2 depends on stages: Stage-1 --当Stage1执行完成后才会执行Stage2
  Stage-0 depends on stages: Stage-2 --当Stage2执行完成后才会执行Stage0
 STAGE PLANS: --具体Stage信息
  Stage: Stage-1 
    Map Reduce  --MapReduce阶段
      Alias -> Map Operator Tree: --map阶段从一个特定表或者上一个map/reduce阶段结果中读取
        src --表名
            Reduce Output Operator
              key expressions:
                    expr: key
                    type: string
              sort order: +
              Map-reduce partition columns:
                    expr: rand()
                    type: double
              tag: -1
              value expressions:
                    expr: substr(value, 4)
                    type: string
      Reduce Operator Tree: --执行部分聚合
        Group By Operator
          aggregations:
                expr: sum(UDFToDouble(VALUE.0))
          keys:
                expr: KEY.0
                type: string
          mode: partial1
          File Output Operator
            compressed: false
            table:
                input format: org.apache.hadoop.mapred.SequenceFileInputFormat
                output format: org.apache.hadoop.mapred.SequenceFileOutputFormat
                name: binary_table
 
  Stage: Stage-2
    Map Reduce --MapReduce阶段
      Alias -> Map Operator Tree:
        /tmp/hive-zshao/67494501/106593589.10001
          Reduce Output Operator
            key expressions:
                  expr: 0
                  type: string
            sort order: +  --列排序；+表示升序；-表示降序；一个+代表对一个列进行排序操作，如sort order:++ :代表按照两个列升序序排列
            Map-reduce partition columns:
                  expr: 0
                  type: string
            tag: -1
            value expressions:
                  expr: 1
                  type: double
      Reduce Operator Tree: --从Stage1中的部分聚合计算进行再次聚合
        Group By Operator --分组操作
          aggregations: --聚合函数信息
                expr: sum(VALUE.0)
          keys: --分组字段
                expr: KEY.0
                type: string
          mode: final --聚合模式:hash:随机聚合，final:最终聚合
          Select Operator
            expressions:
                  expr: 0
                  type: string
                  expr: 1
                  type: double
            Select Operator --选取操作
              expressions: --需要的字段名和类型
                    expr: UDFToInteger(0)
                    type: int
                    expr: 1
                    type: double
              File Output Operator --过滤操作
                compressed: false
                table:
                    input format: org.apache.hadoop.mapred.TextInputFormat
                    output format: org.apache.hadoop.hive.ql.io.IgnoreKeyTextOutputFormat
                    serde: org.apache.hadoop.hive.serde2.dynamic_type.DynamicSerDe
                    name: dest_g1
 
  Stage: Stage-0
    Move Operator --文件操作，将从临时目录移动到结果目录下
      tables:
            replace: true
            table:
                input format: org.apache.hadoop.mapred.TextInputFormat
                output format: org.apache.hadoop.hive.ql.io.IgnoreKeyTextOutputFormat
                serde: org.apache.hadoop.hive.serde2.dynamic_type.DynamicSerDe
                name: dest_g1
```


**依赖分析示例:**


```sql
EXPLAIN DEPENDENCY SELECT key, count(1) FROM srcpart WHERE ds IS NOT NULL GROUP BY key
```


**依赖分析结果：**


```sql
{"input_partitions":[{"partitionName":"default<at:var at:name="srcpart" />ds=2008-04-08/hr=11"},{"partitionName":"default<at:var at:name="srcpart" />ds=2008-04-08/hr=12"},{"partitionName":"default<at:var at:name="srcpart" />ds=2008-04-09/hr=11"},{"partitionName":"default<at:var at:name="srcpart" />ds=2008-04-09/hr=12"}],"input_tables":[{"tablename":"default@srcpart","tabletype":"MANAGED_TABLE"}]}
```


**分析实际数据量示例：**


```sql
explain analyze select t1.user_id,t2.visit_url from wedw_tmp.tmp_url_info t1 full join wedw_tmp.tmp_url_info t2 on t1.user_id = t2.user_id;
```


**分析实际数量结果：**


```sql
STAGE DEPENDENCIES:
  Stage-1 is a root stage
  Stage-0 depends on stages: Stage-1

STAGE PLANS:
  Stage: Stage-1
    Map Reduce
      Map Operator Tree:
          TableScan
            alias: t1
            Statistics: Num rows: 9/9 Data size: 1041 Basic stats: COMPLETE Column stats: NONE
            Select Operator
              expressions: user_id (type: string)
              outputColumnNames: _col0
              Statistics: Num rows: 9/9 Data size: 1041 Basic stats: COMPLETE Column stats: NONE
              Reduce Output Operator
                key expressions: _col0 (type: string)
                sort order: +
                Map-reduce partition columns: _col0 (type: string)
                Statistics: Num rows: 9/9 Data size: 1041 Basic stats: COMPLETE Column stats: NONE
          TableScan
            alias: t2
            Statistics: Num rows: 9/9 Data size: 1041 Basic stats: COMPLETE Column stats: NONE
            Select Operator
              expressions: user_id (type: string), visit_url (type: string)
              outputColumnNames: _col0, _col1
              Statistics: Num rows: 9/9 Data size: 1041 Basic stats: COMPLETE Column stats: NONE
              Reduce Output Operator
                key expressions: _col0 (type: string)
                sort order: +
                Map-reduce partition columns: _col0 (type: string)
                Statistics: Num rows: 9/9 Data size: 1041 Basic stats: COMPLETE Column stats: NONE
                value expressions: _col1 (type: string)
      Reduce Operator Tree:
        Join Operator
          condition map:
               Outer Join 0 to 1
          keys:
            0 _col0 (type: string)
            1 _col0 (type: string)
          outputColumnNames: _col0, _col2
          Statistics: Num rows: 9/41 Data size: 1145 Basic stats: COMPLETE Column stats: NONE
          Select Operator
            expressions: _col0 (type: string), _col2 (type: string)
            outputColumnNames: _col0, _col1
            Statistics: Num rows: 9/41 Data size: 1145 Basic stats: COMPLETE Column stats: NONE
            File Output Operator
              compressed: false
              Statistics: Num rows: 9/41 Data size: 1145 Basic stats: COMPLETE Column stats: NONE
              table:
                  input format: org.apache.hadoop.mapred.SequenceFileInputFormat
                  output format: org.apache.hadoop.hive.ql.io.HiveSequenceFileOutputFormat
                  serde: org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe

  Stage: Stage-0
    Fetch Operator
      limit: -1
      Processor Tree:
        ListSink
```
![](https://gitee.com/PeterOtter/yu-que-sync-image/raw/master/20210629004041.png#align=left&display=inline&height=463&id=luMd5&margin=%5Bobject%20Object%5D&originHeight=463&originWidth=1238&status=done&style=none&width=1238)


### 1.1.2、Stage division（不够细致，需要例子）
**Stage理解：**
	结合对前面讲到的Hive对查询的一系列执行流程的理解，那么在一个查询任务中会有一个或者多个Stage.每个Stage之间可能存在依赖关系。没有依赖关系的Stage可以并行执行。

​	Stage是Hive执行任务中的某一个阶段，那么这个阶段可能是一个MR任务，也可能是一个抽取任务，也可能是一个Map Reduce Local ,也可能是一个Limit。
**何时划分Stage:**
​	那么Stage划分的时机其实是发生在逻辑计划转化OperatorTree转化成物理计划的阶段TaskTree，按照深度优先遍历OperatorTree,再结合具体执行引擎的Compiler(MR/Tez/Spark)应用规则生成对应的Task。
​	Stage划分的界限决定于ReduceSinkOperator，在遇到ReduceSinkOperator之前的Operator都划分到Map阶段，同时也标识这Map阶段的结束。该ReduceSinkOperator到下一个ReduceSinkOperator阶段中间的部分划分为Reduce阶段。一个MR任务代表一个Stage（当然也包括其他非MR，如FetchTask、MoveTask、CopyTask）。
**划分规则(按照MR为例子)：**
​	R1: TS% ---->生成MapRedTask对象，确定MapWork
​	R2:TS%.*RS --->遇到第一个ReduceSinkOperator，划分Map阶段，确定ReduceWork
​	R3:RS%.*RS% ---->生成新的MapRedTask，切分MapRedTask。这个时候已经生成一个Job
​	R4:FS% ----> 连接MapRedTask和MoveTask。
​	R5:UNION% ---->如果所有的子查询都是map-only，则把所有的MapWork进行合并连接。
​	R6:UNION%.*RS% --->遇到ReduceSinkOpeartor，则合并Stage，
​	R7:MAPJOIN%
![](https://gitee.com/PeterOtter/yu-que-sync-image/raw/master/20210629004245.png#align=left&display=inline&height=504&id=juPIG&margin=%5Bobject%20Object%5D&originHeight=504&originWidth=1093&status=done&style=none&width=1093)
**demo**


```sql
insert ovewrite table test
select
  distinct url
from tmp.test
where date_id='2021-06-08' and length(url)>0 and url is not null
distribute by rand()
limit 10000
```


![](https://gitee.com/PeterOtter/yu-que-sync-image/raw/master/20210629004309.png#align=left&display=inline&height=266&id=iG2bY&margin=%5Bobject%20Object%5D&originHeight=266&originWidth=1160&status=done&style=none&width=1160)
第一个Job发生的Map阶段：
![](https://gitee.com/PeterOtter/yu-que-sync-image/raw/master/20210629005001.png#align=left&display=inline&height=424&id=vhK6B&margin=%5Bobject%20Object%5D&originHeight=424&originWidth=1156&status=done&style=none&width=1156)


第一个Job发生的Reduce阶段:
![](https://gitee.com/PeterOtter/yu-que-sync-image/raw/master/20210629005036.png#align=left&display=inline&height=545&id=amp42&margin=%5Bobject%20Object%5D&originHeight=545&originWidth=1152&status=done&style=none&width=1152)


第二个Job发生的Map阶段
![](https://gitee.com/PeterOtter/yu-que-sync-image/raw/master/20210629005057.png#align=left&display=inline&height=347&id=UISjZ&margin=%5Bobject%20Object%5D&originHeight=347&originWidth=1182&status=done&style=none&width=1182)


第二个Job发生的Reduce阶段
![](https://gitee.com/PeterOtter/yu-que-sync-image/raw/master/20210629005127.png#align=left&display=inline&height=474&id=S2OmQ&margin=%5Bobject%20Object%5D&originHeight=474&originWidth=1148&status=done&style=none&width=1148)


第三个Job发生的Map阶段
![](https://gitee.com/PeterOtter/yu-que-sync-image/raw/master/20210629005412.png#align=left&display=inline&height=1304&id=KpfYi&margin=%5Bobject%20Object%5D&originHeight=1304&originWidth=1246&status=done&style=none&width=1246)


第三个Job发生的Reduce阶段
![](https://gitee.com/PeterOtter/yu-que-sync-image/raw/master/20210629011248.png#align=left&display=inline&height=536&id=Yz87K&margin=%5Bobject%20Object%5D&originHeight=536&originWidth=696&status=done&style=none&width=696)
**从sql查看具体的生成的job**


```sql
create table  wedw_tmp.test as
select
 t1.user_id,count(1)
from test1 t1
left join test1  t2
on t1.user_id = t2.user_id
where t1.date_id='2021-06-08' and t2.date_id='2021-06-08'  and t1.user_id='12313' and t2.user_id='12313'
group by t1.user_id
distribute by rand()
limit 10000
```


![](https://gitee.com/PeterOtter/yu-que-sync-image/raw/master/20210629005442.png#align=left&display=inline&height=546&id=T828x&margin=%5Bobject%20Object%5D&originHeight=546&originWidth=1180&status=done&style=none&width=1180)
### 1.1.3、Statistics Job
从OperatorTree生成Job的过程:
	1、对输出表生成MoveTask
	2、从OperatorTree中的一个根节点向下深度优先遍历
	3、ReduceSinkOperator标识Map/Reduce界限，多个Job间的界限
	4、遍历其他根节点，遇到JoinOpeartor合并MapReduceTask
	5、生成StatTask更新元数据    
	6、剪断Map和Reduce之间的Operator关系
代码层面：
```java
Utilities.getMRTasks(plan.getRootTasks()).size()
        + Utilities.getTezTasks(plan.getRootTasks()).size()
        + Utilities.getSparkTasks(plan.getRootTasks()).size();
```


计算生成MR Job的逻辑：
从执行计划根节点任务开始遍历，当该task属于ExecDriver实例，那么Job数+1


计算TezTask生成的Job数逻辑：
也是从执行计划根节点开始遍历，如果该task属于TezTask，那么Job数+1


计算Spark Job数的逻辑：
也是从执行计划根节点开始遍历，如果该task属于SparkTask，那么Job数+1


分析层面：
从explain可以看出一个sql语句会被划分多少个Stage,其实Stage总数就是Job数。但是有些Stage并不是MR或者Spark/Tez任务。所以要根据Operator类型来确定大概有多少个Job.


### 1.1.4、Only Map?


总结：即只要不发生shuffle,只是做简单的处理，如简单查或者文件移动/删除就会只有map阶段。
1、Creaet table As Select field from table [where condition]
![](https://gitee.com/PeterOtter/yu-que-sync-image/raw/master/20210629011331.png#align=left&display=inline&height=276&id=T1FIb&margin=%5Bobject%20Object%5D&originHeight=276&originWidth=1184&status=done&style=none&width=1184)
2、select field from table [where condition]   --需要设置**hive.fetch.task.conversion=none**
![](https://gitee.com/PeterOtter/yu-que-sync-image/raw/master/20210629005513.png#align=left&display=inline&height=838&id=r3LM8&margin=%5Bobject%20Object%5D&originHeight=838&originWidth=1182&status=done&style=none&width=1182)
3、insert into/overwrite target_table select * from table [where condition]
![](https://gitee.com/PeterOtter/yu-que-sync-image/raw/master/20210629011416.png#align=left&display=inline&height=252&id=z3BsF&margin=%5Bobject%20Object%5D&originHeight=252&originWidth=1178&status=done&style=none&width=1178)
4、select /_+MAPJOIN(...)_/ ....
![](https://gitee.com/PeterOtter/yu-que-sync-image/raw/master/20210629011436.png#align=left&display=inline&height=760&id=rkGbe&margin=%5Bobject%20Object%5D&originHeight=760&originWidth=1180&status=done&style=none&width=1180)
5、使用transform，只调用map script;
具体使用可参考:[https://github.com/rgordon/hive-transform-example](https://github.com/rgordon/hive-transform-example)


```shell
cat>/tmp/myscript.sh
sed -r -e 's/\{(.*)\}/\1/' -e 's/"//g' -e 's/v(.)/v\100/g'
```


```sql
create table d (item map<string,string>);
create table s (item map<string,string>);
insert into s select map('k1','v1','k2','v2','k3','v3');

add file /tmp/myscript.sh;
select  
     str_to_map (result)
 from  
   (select  
      transform (item) using "myscript.sh" as result
    from s
    ) t
```
![](https://gitee.com/PeterOtter/yu-que-sync-image/raw/master/20210629011500.png#align=left&display=inline&height=1238&id=QwLXD&margin=%5Bobject%20Object%5D&originHeight=1238&originWidth=1298&status=done&style=none&width=1298)
### 1.1.5、Only Reduce?


常规上思考MapReduce任务肯定是要有map阶段的，reduce阶段输入数据是通过map端输出数据拿来的。所以MapReduce不可能存在只有reduce的场景。
同样在hive的源码中也可以发现，ExecDriver在配置Job的时候是绑定了ExecMapper和ExecReducer，那么底层引擎是依赖于MapReduce的。
![](https://gitee.com/PeterOtter/yu-que-sync-image/raw/master/20210629005616.png#align=left&display=inline&height=878&id=uAb4K&margin=%5Bobject%20Object%5D&originHeight=878&originWidth=1166&status=done&style=none&width=1166)
这么看来Hive也是不可能支持的。
但是在Hive的源码中存在另外一个MR框架。是hive自己实现的。
![](https://gitee.com/PeterOtter/yu-que-sync-image/raw/master/20210629005632.png#align=left&display=inline&height=1110&id=AZlLq&margin=%5Bobject%20Object%5D&originHeight=1110&originWidth=1174&status=done&style=none&width=1174)
该框架内的MapReduce和hadoop底层的MR组件不一样，hive自身的MR框架，Map和Reduce是独立开的，每个阶段都是直接使用输入输出流，两者之间并没有依赖关系。
所以hive自身框架内的MR是可以支持只有reduce阶段的。可以使用transform函数来实现。


```sql
FROM (
  FROM src
   MAP value, key
 USING 'java -cp hive-contrib-${system:hive.version}.jar org.apache.hadoop.hive.contrib.mr.example.IdentityMapper'
    AS k, v
 CLUSTER BY k) map_output
  REDUCE k, v
   USING 'java -cp hive-contrib-${system:hive.version}.jar org.apache.hadoop.hive.contrib.mr.example.WordCountReduce'
   AS k, v
;
```


![](https://gitee.com/PeterOtter/yu-que-sync-image/raw/master/20210629011628.png#align=left&display=inline&height=924&id=twT1g&margin=%5Bobject%20Object%5D&originHeight=924&originWidth=656&status=done&style=none&width=656)
## 1.2、Filter PushDown Cases And Outer Join Behavior


```sql
前提:关闭优化器
set hive.auto.convert.join=false;
set hive.cbo.enable=false;
```


Inner Join:
1、Join On中的谓词: 左表下推、右表下推
2、Where谓词:左表下推、右表下推


```sql
-- 第一种情况：join on 谓词
select 
 t1.user_id,
 t2.user_id
from wedw_tmp.tmp_test_1 t1
join wedw_tmp.tmp_test_2 t2
on t1.user_id = t2.user_id   and t1.user_id='111' and t2.user_id='222'

--第二种情况：where谓词
select 
 t1.user_id,
 t2.user_id
from wedw_tmp.tmp_test_1 t1
join wedw_tmp.tmp_test_2 t2
on t1.user_id = t2.user_id   
where t1.user_id='111' and t2.user_id='222'
```
![](https://gitee.com/PeterOtter/yu-que-sync-image/raw/master/20210629005809.png#align=left&display=inline&height=626&id=Na7aD&margin=%5Bobject%20Object%5D&originHeight=626&originWidth=1174&status=done&style=none&width=1174)
Left join:
1、Join On中的谓词: 左表不下推、右表下推(前提：关闭mapjoin优化)
2、Where谓词:左表下推、右表下推


```sql
-- 第一种情况：join on 谓词
select 
 t1.user_id,
 t2.user_id
from wedw_tmp.tmp_test_1 t1
left join wedw_tmp.tmp_test_2 t2
on t1.user_id = t2.user_id   and t1.user_id='111' and t2.user_id='222'

--第二种情况：where谓词
select 
 t1.user_id,
 t2.user_id
from wedw_tmp.tmp_test_1 t1
left join wedw_tmp.tmp_test_2 t2
on t1.user_id = t2.user_id   
where t1.user_id='111' and t2.user_id='222'
```
![](https://gitee.com/PeterOtter/yu-que-sync-image/raw/master/20210629005823.png#align=left&display=inline&height=514&id=AStP1&margin=%5Bobject%20Object%5D&originHeight=514&originWidth=1182&status=done&style=none&width=1182)
Right Join:
1、Join On中的谓词: 左表下推、右表不下推(前提：关闭mapjoin优化)
2、Where谓词:左表下推、右表下


```sql
-- 第一种情况：join on 谓词
select 
 t1.user_id,
 t2.user_id
from wedw_tmp.tmp_test_1 t1
right join wedw_tmp.tmp_test_2 t2
on t1.user_id = t2.user_id   and t1.user_id='111' and t2.user_id='222'

--第二种情况：where谓词
select 
 t1.user_id,
 t2.user_id
from wedw_tmp.tmp_test_1 t1
right join wedw_tmp.tmp_test_2 t2
on t1.user_id = t2.user_id   
where t1.user_id='111' and t2.user_id='222'
```


Full Join:
1、Join On中的谓词: 左表不下推、右表不下推(前提：关闭mapjoin优化)
2、Where谓词:左表不下推、右表不下推


```sql
-- 第一种情况：join on 谓词
select 
 t1.user_id,
 t2.user_id
from wedw_tmp.tmp_test_1 t1
full join wedw_tmp.tmp_test_2 t2
on t1.user_id = t2.user_id   and t1.user_id='111' and t2.user_id='222'

--第二种情况：where谓词
select 
 t1.user_id,
 t2.user_id
from wedw_tmp.tmp_test_1 t1
full join wedw_tmp.tmp_test_2 t2
on t1.user_id = t2.user_id   
where t1.user_id='111' and t2.user_id='222'
```
![](https://gitee.com/PeterOtter/yu-que-sync-image/raw/master/20210629005843.png#align=left&display=inline&height=448&id=Uykip&margin=%5Bobject%20Object%5D&originHeight=448&originWidth=1182&status=done&style=none&width=1182)
总结：
![](https://gitee.com/PeterOtter/yu-que-sync-image/raw/master/20210629011744.png#align=left&display=inline&height=210&id=o2eeb&margin=%5Bobject%20Object%5D&originHeight=210&originWidth=1192&status=done&style=none&width=1192)