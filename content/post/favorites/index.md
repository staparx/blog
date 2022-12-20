---
author : "Staparx"
title: "图数据库GDB - Gremlin 的使用记录"
description: 图数据库实用小技巧
date: 2022-12-20T16:47:40+08:00
image: img.png
math: 
license: 
hidden: false
comments: true
tags : [
    "实用技巧",
    "图数据库",
    "GDB",
    "Gremlin",
]
categories : [
    "收藏夹",
]

---


# 图数据库GDB的使用 - Gremlin

公司需求开发中，产品提出想要绘画出人员关系图。关系型数据库不太适合该场景，因此引入了图数据库（[阿里云GDB](https://help.aliyun.com/product/102714.html)）。

## Gremlin
[Gremlin中文文档](http://tinkerpop-gremlin.cn/?spm=a2c4g.11186623.0.0.3eb77ca3qPNew2#traversal)


## 复杂场景实战

### 通用场景类
#### 求两个用户A和B的共同好友数
下面输出A的好友数、B的好友数、共同好友数
```bash
g.inject('any').
sideEffect(g.V('A').bothE().otherV().aggregate('v1')).
sideEffect(g.V('B').bothE().otherV().aggregate('v2')).
project('count1', 'count2', 'countMerge').
by(select('v1').unfold().count()).
by(select('v2').unfold().count()).
by(select('v1').unfold().where(within('v2')).count())
```
#### K个用户的共同好友
上面的情况进一步拓展，要求K个好友的共同好友，假定求1，3，5这三个用户的共同好友，可以这样计算。
```bash
gremlin> g.V(1, 3, 5).both().groupCount().unfold().where(select(values).is(3)).select(keys)
==>v[4]
```

如果5是超级顶点，则可以这样计算。
方法3： 如果5是超级点，1，3是一般的点
```bash
g.V(1,3).both().as("x").both().where(id().is(5)).
select("x").groupCount().unfold().where(select(values).is(2)).select(keys)
```
#### 排名前三的网络大V
社交场景中，大V往往是根据粉丝的数量来衡量，粉丝越多说明越受欢迎，加入我们要找到拥有粉丝最多的三个大V，可以这样找
```bash
g.V().project('user','degree').by().by(inE().count()).order().by(select('degree'), desc).limit(3)
```

#### 图数据分布
以巴拉巴西为代表的科学家们发现了大量满足幂律的网络结构，这种网络被称为无标度网络。人们越来越倾向于认为，幂律（Power Law）是无处不在规律，尤其在复杂网络中。


#### 找和A同公司的上三届师兄、师姐
小A刚入职，可以将本公司小A的师兄、师姐推荐给小A，让小A尽量通过熟人的帮助尽快融入工作小圈子，可以这样写。
```bash
g.V().hasLabel('person').hasId('A').
sideEffect(out('workAt').aggregate('company')).
outE('studyAt').has('classYear', '2009').inV(). // 求学于哪个学校
inE('studyAt').has('classYear', within('2009', '2008', '2007', '2006')).
outV().has(id, neq('p1024')).  //该学校06-09年毕业的其他学生
where(out('workAt').where(within('company'))) // 也再该公司的校友集合
```

#### 多级遍历 & 每级只关注特定点
这种场景是k跳遍历的一种特殊情况，再每一跳中，我们只关注top1，这里的top1可以为某个属性的权重等，这种场景再金融领域也比较场景, 比如我们再每一跳中，只取weight权重最高的这条边，并沿着这条边一直走到终点。
```bash
gremlin> g.V(1).repeat(__.flatMap(outE("knows").order().by('weight',desc).limit(1).otherV())).
until(__.not(outE("knows"))).path()
==>[v[1],v[4]]
```

#### 多层级链路探索发现(股权穿透)类请求
从某个点出发，不断的向外拓展，找出一个联通子图
```bash
gremlin> g.V(1).repeat(both().dedup()).emit()
==>v[3]
==>v[2]
==>v[4]
==>v[1]
==>v[6]
==>v[5]
```

#### 根据特定点边数量进行路径过滤

比如有个sql从start开始， PeopleA和PeopleB两个分支，用户需要根据每个分支的边的数量进行过滤，比如上图中PeopleA有两个路径，PeopleB只有一个路径，此时选择路径A，这个sql如何写？
```bash
==>All scripts will now be evaluated locally - type ':remote console' to return to remote mode for Gremlin Server - [localhost/127.0.0.1:3002]
gremlin> g.V().as("1").out().as("2").project("f", "e").by(select("1")).by(select("2"))
==>[f:v[1],e:v[3]]
==>[f:v[1],e:v[2]]
==>[f:v[1],e:v[4]]
==>[f:v[4],e:v[5]]
==>[f:v[4],e:v[3]]
==>[f:v[6],e:v[3]]

gremlin> g.V().as("1").out().as("2").project("f", "e").by(select("1")).by(select("2")).
group().by(select("f"))
==>[v[1]:[[f:v[1],e:v[3]],[f:v[1],e:v[2]],[f:v[1],e:v[4]]],
v[4]:[[f:v[4],e:v[5]],[f:v[4],e:v[3]]],
v[6]:[[f:v[6],e:v[3]]]]


gremlin> g.V().as("1").out().as("2").project("f", "e").by(select("1")).by(select("2")).
group().by(select("f")).select(values)
==>[[[f:v[1],e:v[3]],[f:v[1],e:v[2]],[f:v[1],e:v[4]]],
[[f:v[4],e:v[5]],[f:v[4],e:v[3]]],
[[f:v[6],e:v[3]]]]


gremlin> g.V().as("1").out().as("2").project("f", "e").by(select("1")).by(select("2")).
group().by(select("f")).select(values). //按照f第一列进行聚合，并选择values
unfold().where(count(local).is(gte(2))). //聚合后，去掉外层的list，并求内层的数量，过滤
unfold() // 最后再unfold，即为结果
==>[f:v[1],e:v[3]]
==>[f:v[1],e:v[2]]
==>[f:v[1],e:v[4]]
==>[f:v[4],e:v[5]]
==>[f:v[4],e:v[3]]
```

### 路径探索类
#### 两点之间最短路径
根据6度理论，再社交场景中，任何两个点的距离都在6度以内。如果小A有急事需要找著名医院的小B医生帮忙，可以A和B之间有无数路径，我们只关注最短的路径，从而提高求助效率。可以这样写。
```bash
g.V("A").repeat(both().simplePath()).
until(hasId("B").or().loops().is(gt(6L))).
hasId("B").path().limit(1)
```

#### 带权重的最短路径
有时候最短路径并不是按照hop跳数定义，而是根据边上的某些属性定义，这样稍微复杂点，比如下图，1 - 3之间有两条路径，按照边的weight属性累积求和作为路径的权重，从而求出最短路径。
```bash
gremlin> g.V("1").repeat(outE().inV().simplePath()).
until(hasId("3").or().loops().is(gt(1))).hasId("3").path().as("p").
map(unfold().coalesce(values("weight"), constant(0.0)).sum()).as("cost").
select("cost","p").order().by(select("cost"), Order.incr)
==>[cost:0.4,p:[v[1],e[9][1-created->3],v[3]]]
==>[cost:1.4,p:[v[1],e[8][1-knows->4],v[4],e[11][4-created->3],v[3]]]
```

#### 股权穿透/多级分销
这类场景再企业控股类场景，比如a持股b 50%； b持股c 50%，则a持股c 25%；或者社交产品分销场景中非常常见。
```bash
gremlin> g.withSack(1.0f).V("1").
repeat(outE().sack(mult).by("weight").inV().simplePath())
.emit().path().as("p").sack().as("cost").
select("cost","p").order().by(select("cost"),Order.desc)
==>[cost:1.0,p:[v[1],e[8][1-knows->4],v[4]]]
==>[cost:1.0,p:[v[1],e[8][1-knows->4],v[4],e[10][4-created->5],v[5]]]
==>[cost:0.5,p:[v[1],e[7][1-knows->2],v[2]]]
==>[cost:0.4,p:[v[1],e[9][1-created->3],v[3]]]
==>[cost:0.4,p:[v[1],e[8][1-knows->4],v[4],e[11][4-created->3],v[3]]]
```

#### 股权穿透保留符合特定条件的路径
比如在股权累乘时只保留 累乘结果 > 0.5的路径，过滤掉 <= 0.5的路径，可以参考：
```bash
gremlin> g.withSack(1.0f).V("1").repeat(outE().sack(mult).by("weight").where(sack().is(gte(0.5))).inV().simplePath()).emit().path().as("p").sack().as("cost").select("cost", "p")
gremlin> g.withSack(1.0f).V("1").
repeat(outE().sack(mult).by("weight").where(sack().is(gte(0.5))).inV().simplePath()).
emit().path().as("p").sack().as("cost").select("cost", "p")
==>[cost:0.5,p:[v[1],e[7][1-knows->2],v[2]]]
==>[cost:1.0,p:[v[1],e[8][1-knows->4],v[4]]]
==>[cost:1.0,p:[v[1],e[8][1-knows->4],v[4],e[10][4-created->5],v[5]]]
```

#### 路径按边进行去重
不同于neo4j的path默认是边去重。gremlin查询语言的path去重是不区分点和边，也就是按照path中的object对象进行去重。
```bash
## 如果使用下面方式，则会出现A-D-A-D循环，一般不是用户需要的
g.V("A").repeat(bothE().otherV()).until(hasId(3).or().loops().is(gte(4))).hasId(3)

## 如果使用simplePath方式，则不会出现A-D-E-D这条路径，虽然D-E之间有两条路径
g.V("A").repeat(bothE().otherV().simplePath()).until(hasId(3).or().loops().is(gte(4))).hasId(3)
```

如果需要上述中A-D-E-D这种路径，但又不希望A-D之间的边重复多次，这种场景就是典型的边去重模式，可惜目前gremlin的框架不支持，相关讨论参考gremlin-users讨论。即使如此，用gremlin的用户依然可以有两种方案来规避：
```bash
## 方案1. 使用groovy语法
gremlin> g.withSack {['~~']}{it.clone()}.V(1).repeat(bothE().as("e").filter(__.sack().as("list").select("e").where(without("list"))).sack{m,v -> m += v; m}.otherV()).times(2).path()
==>[v[1],e[9][1-created->3],v[3],e[11][4-created->3],v[4]]
==>[v[1],e[9][1-created->3],v[3],e[12][6-created->3],v[6]]
==>[v[1],e[8][1-knows->4],v[4],e[10][4-created->5],v[5]]

==>[v[1],e[8][1-knows->4],v[4],e[11][4-created->3],v[3]]
==>[v[1],e[8][1-knows->4],v[4],e[0][5-fake->4],v[5]]

## 解释
g.withSack {['~~']}{it.clone()}.V(1).
repeat(bothE().as("e").
filter(__.sack().as("list").select("e").where(without("list"))). //往外跳时，对边做过滤
sack{m,v -> m += v; m}.otherV()).   //将新的边插入到sack中，表示该边已经被过滤过
times(2).path() //循环2度并退出循环

作为对比，非边去重的结果，可以看到edge 9再path中出现两次：
gremlin> g.withSack {['~~']}{it.clone()}.V(1).repeat(bothE().as("e").sack{m,v -> m += v; m}.otherV()).times(2).path()
==>[v[1],e[9][1-created->3],v[3],e[9][1-created->3],v[1]]
==>[v[1],e[7][1-knows->2],v[2],e[7][1-knows->2],v[1]]
...


## 方案2. 先不考虑边重复问题，找到结果。并对最终每个path进行判断，判断其是否有重复的边
gremlin> g.V(1).repeat(bothE().otherV()).until(hasId(3).or().loops().is(gte(4))).hasId(3).path().as("p").flatMap(unfold().group().by(choose(hasLabel("person", "software"), __.constant("v"), __.constant("e")))).select("e").as("origin_e").dedup(local).as("dedup_e").filter(select("dedup_e", "origin_e").by(count(local)).where("dedup_e", eq("origin_e"))).select("p")
==>[v[1],e[9][1-created->3],v[3]]
==>[v[1],e[8][1-knows->4],v[4],e[11][4-created->3],v[3]]
==>[v[1],e[8][1-knows->4],v[4],e[10][4-created->5],v[5],e[0][5-fake->4],v[4],e[11][4-created->3],v[3]]
==>[v[1],e[8][1-knows->4],v[4],e[0][5-fake->4],v[5],e[10][4-created->5],v[4],e[11][4-created->3],v[3]]

gremlin> g.V(1).repeat(bothE().otherV()).until(hasId(3).or().loops().is(gte(4))).hasId(3).
path().as("p").    // 求出所有path
flatMap(unfold().group().  //path按照点、边进行聚合
by(choose(hasLabel("person", "software"), __.constant("v"), __.constant("e")))).

    select("e").as("origin_e").dedup(local).as("dedup_e"). // 对边进行去重
    filter(select("dedup_e", "origin_e").by(count(local)). // 如果去重后count一样，说明没有重复
           where("dedup_e", eq("origin_e"))).
    select("p")
```

### 联通子图
#### A点K度范围内 到 B点构建的联通子图
和路径搜索类-两点之间最短路径场景非常类似，不过用户期望A的K度邻居范围内只要有B点，则将A、B和A的K度邻居组成一个联通子图，比如：用户需要A的4度邻居与C组成的联通子图：


用gremlin的写入如下所示：
```bash
gremlin> g.V('1').aggregate('vertex').repeat(bothE().aggregate('edge').otherV().aggregate('vertex')).until(loops().is(gte(1))).tail(1).V(3).as("dst").where("dst", within("vertex")).select('vertex', 'edge').by(dedup(local)).by(dedup(local))
==>[vertex:[v[1],v[3],v[2],v[4]],edge:[e[9][1-created->3],e[7][1-knows->2],e[8][1-knows->4]]]

# 解释 以起点为1，终点为3， khop为1， 及1的1跳内连通图包含3构成的子图
gremlin> g.V('1').aggregate('vertex').
repeat(bothE().aggregate('edge').otherV().aggregate('vertex')). // 将中间的点和边进行汇聚存储
until(loops().is(gte(1))).tail(1).  //1跳，取最后一行
V(3).as("dst").where("dst", within("vertex")).  // 联通子图中的点需要用终点3进行判断
select('vertex', 'edge').by(dedup(local)).by(dedup(local)) // 点和边分开存储，便于可视化绘图
```
### 协同推荐
#### 按照权重将不是好友的好友的好友推荐给自己
```bash
gremlin> g.withSack(1.0f).V(1).aggregate("friends").
sideEffect(both().aggregate("friends")).
bothE().sack(mult).by("weight").otherV().
bothE().sack(mult).by("weight").otherV().as("last_node").where(without("friends")).dedup().
sack().as("cost").order().by(select("cost"), desc).select("cost", "last_node")
==>[cost:1.0,last_node:v[5]]
==>[cost:0.08000000000000002,last_node:v[6]]
```

#### 找出我关注的和双向关注的朋友
```bash
g.V(my_id).sideEffect(__.in().id().aggregate('subscribe')). // 关注我的
outE().order().by('createAt',desc).inV().sideEffect(__.id().aggregate('i_subscribe')). //我关注的
choose(__.id().where(within('subscribe')),__.id().aggregate('both_subscribe')). //双向关注的
cap('i_subscribe','both_subscribe') //输出
```

#### 找出我的好友的好友B，按照B和我之间的共同好友数量进行倒排，作为推荐候选
```bash
gremlin> g.V(1).both().aggregate("my_friend").  // 1度好友
both().has(id, neq(1)).as("ff").   //2度好友，但不能包含自己
flatMap(__.both().where(within("my_friend")).count()). // 和自己共同好友数
as("comm_cnt").order().by(desc). // 倒排输出2度好友和共同好友数量
select("ff", "comm_cnt")
==>[ff:v[4],comm_cnt:1]
==>[ff:v[6],comm_cnt:1]
==>[ff:v[5],comm_cnt:1]
==>[ff:v[3],comm_cnt:1]
```



#### 将teamA球队 朋友球队的球员 按照其所在球队排列，推荐给自己
图片和构图参考[gremlin-google官方讨论这里](https://stackoverflow.com/questions/71178226/tinkerpop-gremlin-how-get-child-elements-filtered-and-grouped)

```bash
gremlin> g.V().has("name", "team A").
sideEffect(__.out("owned").hasLabel("player").aggregate("my_player").limit(1)). // 本球队的球员
both("is_friends_with").hasLabel("team").as("team2").out("owned").hasLabel("player").
as("friends_player").where(without("my_player")).as("friends_player2"). //朋友球队球员
select("team2", "friends_player2").
group().by(select("friends_player2")).by(select("team2").fold()). //按球员分组
unfold().project("Player", "Teams").by(select(keys).values("name")).
by(select(values).unfold().values("name").fold()) // 投影
==>[Player:Baggio,Teams:[Eagles]]
==>[Player:Kroll,Teams:[Valladolid]]
==>[Player:Icardi,Teams:[Valladolid]]
==>[Player:Papin,Teams:[Valladolid,Eagles]
```
### 特殊需求类
#### 路径按照点、边聚合
Gremlin输出正常的路径是将Vertex、Edge放在一个列表里，比如：
```bash
gremlin> g.V(1).repeat(outE().otherV()).times(2).path()
==>[v[1],e[8][1-knows->4],v[4],e[10][4-created->5],v[5]]
==>[v[1],e[8][1-knows->4],v[4],e[11][4-created->3],v[3]]
```
如果用户想把Vertex、Edge 分开输出再两个列表里, 该怎么办呢？

由于Gremlin语法本身对Element缺少点、边判断的function进行描述，因此本质上是没法做的，但是如果用户知道点的label只有"software","person"这两类，则可以根据label来进行区分，可以这样写：
```bash
gremlin> g.V(1).repeat(outE().otherV()).times(2).path().
flatMap(unfold().
group().by(choose(hasLabel("software", "person"), __.constant("vertex"), __.constant("edge"))))
==>[edge:[e[8][1-knows->4],e[10][4-created->5]],vertex:[v[1],v[4],v[5]]]
==>[edge:[e[8][1-knows->4],e[11][4-created->3]],vertex:[v[1],v[4],v[3]]]
```

#### 所有路径按照点、边聚合
有时候用户期望把所有的路径中的点和边进行聚合，此时可以这样：
```bash
# g.V(1).repeat(outE().otherV()).times(2).path().unfold().group().by(choose(hasLabel("software", "person"), __.constant("vertex"), __.constant("edge"))).unfold().project("k", "v").by(select(keys)).by(select(values).dedup(local))
g.V(1).repeat(outE().otherV()).times(2).path().unfold().
group().by(choose(hasLabel("software", "person"), __.constant("vertex"), __.constant("edge"))).unfold().
project("k", "v").by(select(keys)).by(select(values).dedup(local))
==>[k:edge,v:[e[8][1-knows->4],e[10][4-created->5],e[11][4-created->3]]]
==>[k:vertex,v:[v[1],v[4],v[5],v[3]]]
```

#### 所有路径按点、边聚合后，返回所有的属性信息
```bash
gremlin> g.V('1').repeat(bothE('knows', "created").otherV().hasLabel('person', "software").simplePath()).until(__.hasId('3').or().loops().is(P.gte(5))).hasId('3').path().unfold().dedup().choose(__.hasLabel("person", "software"), __.valueMap(true).aggregate("vertex"), __.project('inV', 'outV','inVlabel','outVlabel', "properteis").by(inV().id()).by(outV().id()).by(inV().label()).by(outV().label()).by(valueMap(true)).aggregate("edge")).tail(1).select("vertex", "edge")

g.V('1').repeat(bothE('knows', "created").otherV().hasLabel('person', "software").simplePath()).
until(__.hasId('3').or().loops().is(P.gte(5))).hasId('3').path(). // path
unfold().dedup().  //去重
choose(__.hasLabel("person", "software"),
__.valueMap(true).aggregate("vertex"), // 点去重，返回所有属性
__.project('inV', 'outV','inVlabel','outVlabel', "properteis").
by(inV().id()).by(outV().id()).by(inV().label()).by(outV().label()).
by(valueMap(true)).aggregate("edge")). //边去重，返回所有属性并且包含in，outv信息
tail(1).
select("vertex", "edge") 聚合输出


==>[vertex:[
[id:1,label:person,name:[marko],age:[29]],
[id:3,label:software,name:[lop],lang:[java]],
[id:4,label:person,name:[josh],age:[32]]
],

edge:[
[inV:3,outV:1,inVlabel:software,outVlabel:person,properteis:[id:9,label:created,weight:0.4]],
[inV:4,outV:1,inVlabel:person,outVlabel:person,properteis:[id:8,label:knows,weight:1.0]],
[inV:3,outV:4,inVlabel:software,outVlabel:person,properteis:[id:11,label:created,weight:0.4]]]]
```

#### 类型转换
Gremlin对算子执行算数运算的约束比较死板，如果两个值类型不一致，直接报错。也没有像cypher提供类型转换的函数，但是由于兼容groovy语法，可以利用groovy来进行转换（GDB禁止掉了该语法），比如。
```bash
gremlin> g.withSack(2).V(1).sack(mult).by(values("x").map{(''+it).toInteger()}).sack()
429 ==>4
```

### 批量更新
#### 批量属性插入
属性插入有两种写入，常规的比如：
```bash
gremlin> g.V(1).property("k1", "v1").property("k2", 'v2')
==>v[1]
```

上面的写法需要把kx，vx分别用property包装起来，除非用脚本生成，一般而言可扩展性比较差，其实可以把properties作为参数，并通过step进行循环插入，比如。
```bash
gremlin> g.inject([['code':'L10', 'description':'Some other area']]).unfold().as('properties').
V(1).as('vertex').
sideEffect(__.select('properties').unfold().as('kv').
select('vertex').property(__.select('kv').by(Column.keys),__.select('kv').by(Column.values))).
valueMap()
==>[code:[L10],k1:[v1],k2:[v2],name:[marko],description:[Some other area],age:[29]]
```

#### 批量插入点、插入边
```bash
g.addV("vertex").property(id,124).as("start").
addV("vertex").property(id,125).as("end").
sideEffect(addE("end").from(select("start")).to(select("end"))).
sideEffect(addE("end2").from(select("start")).to(select("end")))
```

#### 存在就更新，不存在插入
```bash
## 点
g.inject('a').choose(__.V(11),__.V(11).property('a','c'),__.addV("11").property(id,11).property('a','b'))

## 边
g.inject("Any").coalesce(g.E(EID), addE("follow").from(V(VID1)).to(V(VID2)).property(id, EID)).
property("P1", V1).
property("P2", V2).
property("P3", V3))
```
以上复杂场景实战内容转载于[用Gremlin探索万物-实用小技巧](https://zhuanlan.zhihu.com/p/453293072)
