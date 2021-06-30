# 1、Hive
[Hive Wiki](https://cwiki.apache.org/confluence/display/Hive/Home#Home-UserDocumentation)
## 1.1、Plan
![](https://cdn.nlark.com/yuque/0/2021/png/1544597/1625057255314-fed8ff87-4c79-4cfc-87d6-ec7cf7ee7b58.png#height=382&id=piJHn&originHeight=728&originWidth=1192&originalType=binary&ratio=1&size=0&status=done&style=none&width=626)<br />​

**大致流程：**<br />1、客户端连接到HS2(HiveServer2，目前大多数通过beeline形式连接，Hive Cli模式相对较重，且直接略过授权访问元数据),建立会话<br />2、提交sql，通过Driver进行编译、解析、优化逻辑计划，生成物理计划<br />3、对物理计划进行优化，并提交到执行引擎进行计算<br />4、返回结果<br />**细节流程：**<br />1、客户端和HiveServer2建立连接，创建会话<br />2、提交查询或者DDL，转交到Driver进行处理<br />3、Driver内部会通过Parser对语句进行解析，校验语法是否正确    <br />4、然后通过Compiler编译器对语句进行编译，生成AST Tree<br />5、SemanticAnalyzer会遍历AST Tree，进一步进行语义分析，这个时候会和Hive MetaStore进行通信获取Schema信息,抽象成QueryBlock，逻辑计划生成器会遍历QueryBlock，翻译成Operator（计算抽象出来的算子）生成OperatorTree,这个时候是未优化的逻辑计划<br />6、Optimizer会对逻辑计划进行优化，如进行谓词下推、常量值替换、列裁剪等操作，得到优化后的逻辑计划。<br />7、SemanticAnalyzer会对逻辑计划进行处理，通过TaskCompiler生成物理执行计划TaskTree。<br />8、TaskCompiler会对物理计划进行优化，然后根据底层不同的引擎进行提交执行。<br />![](https://cdn.nlark.com/yuque/0/2021/png/1544597/1625057255335-6648bac3-f9af-4a0c-93b2-a427255cacb0.png#height=661&id=kXx26&originHeight=726&originWidth=490&originalType=binary&ratio=1&size=0&status=done&style=none&width=446)
### 1.1.1、Analyze Sql

<br />**语法：**<br />

```sql
EXPLAIN [EXTENDED|CBO|AST|DEPENDENCY|AUTHORIZATION|LOCKS|VECTORIZATION|ANALYZE] query
```

<br />**版本支持：**<br />Hive0.14.0支持AUTHORIZATION；[HIVE-5961]<br />Hive2.3.0支持VECTORIZATION;[HIVE-11394]<br />Hive3.2.0支持LOCKS;[HIVE-17683]<br />
<br />Explain结果总共分为三个部分：<br />1、对应查询的抽象语法树 AST<br />2、每个计划阶段Stage之间的依赖关系<br />3、每个计划阶段的描述（可能是map/reduce,也可能是操作元数据或者文件操作）<br />**聚合操作分析示例:**<br />

```sql
EXPLAIN FROM src INSERT OVERWRITE TABLE dest_g1 SELECT src.key, sum(substr(src.value,4)) GROUP BY src.key;
```

<br />**聚合操作分析输出信息：**<br />

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

<br />**依赖分析示例:**<br />

```sql
EXPLAIN DEPENDENCY SELECT key, count(1) FROM srcpart WHERE ds IS NOT NULL GROUP BY key
```

<br />**依赖分析结果：**<br />

```sql
{"input_partitions":[{"partitionName":"default<at:var at:name="srcpart" />ds=2008-04-08/hr=11"},{"partitionName":"default<at:var at:name="srcpart" />ds=2008-04-08/hr=12"},{"partitionName":"default<at:var at:name="srcpart" />ds=2008-04-09/hr=11"},{"partitionName":"default<at:var at:name="srcpart" />ds=2008-04-09/hr=12"}],"input_tables":[{"tablename":"default@srcpart","tabletype":"MANAGED_TABLE"}]}
```

<br />**分析实际数据量示例：**<br />

```sql
explain analyze select t1.user_id,t2.visit_url from wedw_tmp.tmp_url_info t1 full join wedw_tmp.tmp_url_info t2 on t1.user_id = t2.user_id;
```

<br />**分析实际数量结果：**<br />

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
![](https://cdn.nlark.com/yuque/0/2021/png/1544597/1625057255327-9bfa9524-02ea-421f-ac93-485101e9ecec.png#height=463&id=luMd5&originHeight=463&originWidth=1238&originalType=binary&ratio=1&size=0&status=done&style=none&width=1238)<br />

### 1.1.2、Stage division
**Stage理解：**<br />结合对前面讲到的Hive对查询的一系列执行流程的理解，那么在一个查询任务中会有一个或者多个Stage.每个Stage之间可能存在依赖关系。没有依赖关系的Stage可以并行执行。<br />Stage是Hive执行任务中的某一个阶段，那么这个阶段可能是一个MR任务，也可能是一个抽取任务，也可能是一个Map Reduce Local ,也可能是一个Limit。<br />**何时划分Stage:**<br />那么Stage划分的时机其实是发生在逻辑计划转化OperatorTree转化成物理计划的阶段TaskTree，按照深度优先遍历OperatorTree,再结合具体执行引擎的Compiler(MR/Tez/Spark)应用规则生成对应的Task。<br />Stage划分的界限决定于ReduceSinkOperator，在遇到ReduceSinkOperator之前的Operator都划分到Map阶段，同时也标识这Map阶段的结束。该ReduceSinkOperator到下一个ReduceSinkOperator阶段中间的部分划分为Reduce阶段。一个MR任务代表一个Stage（当然也包括其他非MR，如FetchTask、MoveTask、CopyTask）。<br />**划分规则(按照MR为例子)：**<br />R1: TS% ---->生成MapRedTask对象，确定MapWork<br />R2:TS%.*RS --->遇到第一个ReduceSinkOperator，划分Map阶段，确定ReduceWork<br />R3:RS%.*RS% ---->生成新的MapRedTask，切分MapRedTask。这个时候已经生成一个Job<br />R4:FS% ----> 连接MapRedTask和MoveTask。<br />R5:UNION% ---->如果所有的子查询都是map-only，则把所有的MapWork进行合并连接。<br />R6:UNION%.*RS% --->遇到ReduceSinkOpeartor，则合并Stage，<br />R7:MAPJOIN%<br />![](https://cdn.nlark.com/yuque/0/2021/png/1544597/1625057255337-a72f4358-80ae-4d80-a29f-656b5e2fb0b0.png#height=369&id=juPIG&originHeight=504&originWidth=1093&originalType=binary&ratio=1&size=0&status=done&style=none&width=800)<br />**demo**<br />

```sql
insert ovewrite table test
select
  distinct url
from tmp.test
where date_id='2021-06-08' and length(url)>0 and url is not null
distribute by rand()
limit 10000
```

<br />![](https://cdn.nlark.com/yuque/0/2021/png/1544597/1625057255314-9a40fc9b-c339-4cd1-8469-cf84e51b46e7.png#height=240&id=iG2bY&originHeight=266&originWidth=1160&originalType=binary&ratio=1&size=0&status=done&style=none&width=1045)<br />第一个Job发生的Map阶段：<br />![](https://cdn.nlark.com/yuque/0/2021/png/1544597/1625057255324-c0829e88-472a-47bc-9ae3-257a9767a836.png#height=372&id=vhK6B&originHeight=424&originWidth=1156&originalType=binary&ratio=1&size=0&status=done&style=none&width=1015)<br />
<br />第一个Job发生的Reduce阶段:<br />![](https://cdn.nlark.com/yuque/0/2021/png/1544597/1625057255318-3af29676-5987-439d-87ca-10b6dc783392.png#height=476&id=amp42&originHeight=545&originWidth=1152&originalType=binary&ratio=1&size=0&status=done&style=none&width=1006)<br />
<br />第二个Job发生的Map阶段<br />![](https://cdn.nlark.com/yuque/0/2021/png/1544597/1625057255309-0305e8be-ffe0-4dd1-95b5-db2553d6d107.png#height=290&id=UISjZ&originHeight=347&originWidth=1182&originalType=binary&ratio=1&size=0&status=done&style=none&width=988)<br />
<br />第二个Job发生的Reduce阶段<br />![](https://cdn.nlark.com/yuque/0/2021/png/1544597/1625057255316-6e4cf388-09ab-46aa-84d9-52f4f752ed98.png#height=474&id=S2OmQ&originHeight=474&originWidth=1148&originalType=binary&ratio=1&size=0&status=done&style=none&width=1148)<br />
<br />第三个Job发生的Map阶段<br />![](https://cdn.nlark.com/yuque/0/2021/png/1544597/1625057255322-5cae3ee7-432b-43d6-8879-39ae621670cc.png#height=923&id=KpfYi&originHeight=1304&originWidth=1246&originalType=binary&ratio=1&size=0&status=done&style=none&width=882)<br />
<br />第三个Job发生的Reduce阶段<br />![](https://cdn.nlark.com/yuque/0/2021/png/1544597/1625057255335-dc5654d5-9e4f-4026-9105-7f3f4ec4daff.png#height=536&id=Yz87K&originHeight=536&originWidth=696&originalType=binary&ratio=1&size=0&status=done&style=none&width=696)<br />**从sql查看具体的生成的job**<br />

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

<br />![](https://cdn.nlark.com/yuque/0/2021/png/1544597/1625057255360-a6dd6812-509f-4723-b652-96acc8536c34.png#height=546&id=T828x&originHeight=546&originWidth=1180&originalType=binary&ratio=1&size=0&status=done&style=none&width=1180)
### 1.1.3、Statistics Job
从OperatorTree生成Job的过程:<br />1、对输出表生成MoveTask<br />2、从OperatorTree中的一个根节点向下深度优先遍历<br />3、ReduceSinkOperator标识Map/Reduce界限，多个Job间的界限<br />4、遍历其他根节点，遇到JoinOpeartor合并MapReduceTask<br />5、生成StatTask更新元数据    <br />6、剪断Map和Reduce之间的Operator关系<br />代码层面：
```java
Utilities.getMRTasks(plan.getRootTasks()).size()
        + Utilities.getTezTasks(plan.getRootTasks()).size()
        + Utilities.getSparkTasks(plan.getRootTasks()).size();
```

<br />计算生成MR Job的逻辑：<br />从执行计划根节点任务开始遍历，当该task属于ExecDriver实例，那么Job数+1<br />
<br />计算TezTask生成的Job数逻辑：<br />也是从执行计划根节点开始遍历，如果该task属于TezTask，那么Job数+1<br />
<br />计算Spark Job数的逻辑：<br />也是从执行计划根节点开始遍历，如果该task属于SparkTask，那么Job数+1<br />
<br />分析层面：<br />从explain可以看出一个sql语句会被划分多少个Stage,其实Stage总数就是Job数。但是有些Stage并不是MR或者Spark/Tez任务。所以要根据Operator类型来确定大概有多少个Job.<br />

### 1.1.4、Only Map?

<br />总结：即只要不发生shuffle,只是做简单的处理，如简单查或者文件移动/删除就会只有map阶段。<br />1、Creaet table As Select field from table [where condition]<br />![](https://cdn.nlark.com/yuque/0/2021/png/1544597/1625057255323-8c1e8890-83ca-4c66-9485-1dc0a28f2e10.png#height=204&id=T1FIb&originHeight=276&originWidth=1184&originalType=binary&ratio=1&size=0&status=done&style=none&width=875)<br />2、select field from table [where condition]   --需要设置**hive.fetch.task.conversion=none**<br />![](https://cdn.nlark.com/yuque/0/2021/png/1544597/1625057255345-398129c0-d7c1-4e4c-897b-ebbb13a82bbd.png#height=623&id=r3LM8&originHeight=838&originWidth=1182&originalType=binary&ratio=1&size=0&status=done&style=none&width=879)<br />3、insert into/overwrite target_table select * from table [where condition]<br />![](https://cdn.nlark.com/yuque/0/2021/png/1544597/1625057255356-bd5967d5-f9a8-4d7b-b8b1-0161359018b2.png#height=185&id=z3BsF&originHeight=252&originWidth=1178&originalType=binary&ratio=1&size=0&status=done&style=none&width=866)<br />4、select /_+MAPJOIN(...)_/ ....<br />![](https://cdn.nlark.com/yuque/0/2021/png/1544597/1625057255384-89e9f0e9-18b3-41d2-acd8-08f9690e19ef.png#height=553&id=rkGbe&originHeight=760&originWidth=1180&originalType=binary&ratio=1&size=0&status=done&style=none&width=858)<br />5、使用transform，只调用map script;<br />具体使用可参考:[https://github.com/rgordon/hive-transform-example](https://github.com/rgordon/hive-transform-example)<br />

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
![](https://cdn.nlark.com/yuque/0/2021/png/1544597/1625057255386-394aabbe-05c6-4a54-88da-842d07a12e97.png#height=784&id=QwLXD&originHeight=1238&originWidth=1298&originalType=binary&ratio=1&size=0&status=done&style=none&width=822)
### 1.1.5、Only Reduce?

<br />常规上思考MapReduce任务肯定是要有map阶段的，reduce阶段输入数据是通过map端输出数据拿来的。所以MapReduce不可能存在只有reduce的场景。<br />同样在hive的源码中也可以发现，ExecDriver在配置Job的时候是绑定了ExecMapper和ExecReducer，那么底层引擎是依赖于MapReduce的。<br />![](https://cdn.nlark.com/yuque/0/2021/png/1544597/1625057255353-3d4b91f7-8f92-49b0-807b-89955ba892ae.png#height=666&id=uAb4K&originHeight=878&originWidth=1166&originalType=binary&ratio=1&size=0&status=done&style=none&width=884)<br />这么看来Hive也是不可能支持的。<br />但是在Hive的源码中存在另外一个MR框架。是hive自己实现的。<br />![](https://cdn.nlark.com/yuque/0/2021/png/1544597/1625057255400-7fca1220-6304-4e2f-871b-fb56b2e94473.png#height=589&id=AZlLq&originHeight=1110&originWidth=1174&originalType=binary&ratio=1&size=0&status=done&style=none&width=623)<br />该框架内的MapReduce和hadoop底层的MR组件不一样，hive自身的MR框架，Map和Reduce是独立开的，每个阶段都是直接使用输入输出流，两者之间并没有依赖关系。<br />所以hive自身框架内的MR是可以支持只有reduce阶段的。可以使用transform函数来实现。<br />

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

<br />![](https://cdn.nlark.com/yuque/0/2021/png/1544597/1625057255379-d15db6a4-b974-452d-a369-e1e1f239b738.png#height=924&id=twT1g&originHeight=924&originWidth=656&originalType=binary&ratio=1&size=0&status=done&style=none&width=656)
## 1.2、Filter PushDown Cases And Outer Join Behavior


```sql
前提:关闭优化器
set hive.auto.convert.join=false;
set hive.cbo.enable=false;
```

<br />Inner Join:<br />1、Join On中的谓词: 左表下推、右表下推<br />2、Where谓词:左表下推、右表下推<br />

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
![](https://cdn.nlark.com/yuque/0/2021/png/1544597/1625057255399-171a778d-e3db-4b45-9eed-0ff64ba53e7a.png#height=476&id=Na7aD&originHeight=626&originWidth=1174&originalType=binary&ratio=1&size=0&status=done&style=none&width=892)<br />Left join:<br />1、Join On中的谓词: 左表不下推、右表下推(前提：关闭mapjoin优化)<br />2、Where谓词:左表下推、右表下推<br />

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
![](https://cdn.nlark.com/yuque/0/2021/png/1544597/1625057255381-6d2bca52-028b-476b-a5d2-51396851d0f4.png#height=514&id=AStP1&originHeight=514&originWidth=1182&originalType=binary&ratio=1&size=0&status=done&style=none&width=1182)<br />Right Join:<br />1、Join On中的谓词: 左表下推、右表不下推(前提：关闭mapjoin优化)<br />2、Where谓词:左表下推、右表下<br />

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

<br />Full Join:<br />1、Join On中的谓词: 左表不下推、右表不下推(前提：关闭mapjoin优化)<br />2、Where谓词:左表不下推、右表不下推<br />

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
![](https://cdn.nlark.com/yuque/0/2021/png/1544597/1625057255399-0e6c78d6-2962-4db9-a7f3-eed480f805ae.png#height=355&id=Uykip&originHeight=448&originWidth=1182&originalType=binary&ratio=1&size=0&status=done&style=none&width=936)<br />总结：<br />![](https://cdn.nlark.com/yuque/0/2021/png/1544597/1625057255386-863b71c0-a609-4b23-957c-4258a2d6bbf5.png#height=165&id=o2eeb&originHeight=210&originWidth=1192&originalType=binary&ratio=1&size=0&status=done&style=none&width=936)
