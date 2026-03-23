## 前言
在前不久（估计也就是上个星期），PostgreSQL 19devel引入了一项新的功能——[SQL Property Graph Queries (SQL/PGQ)](https://github.com/postgres/postgres/commit/2f094e7ac691abc9d2fe0f4dcf0feac4a6ce1d9c)。
我对于SQL/PGQ一直处于听过没怎么用过的状态，所以今天借着这个机会，来简单学习一下如何使用。在步入使用前，先补充一点基本概念。

## 图
在离散数学中，图（graph）是用于表示物体与物体之间存在某种关系的结构。数学抽象后的“物体”称作节点或顶点（vertex, node, point），节点间的相关关系则称作边（edges）。简要概括一下就是：

图是描述事物关系的数据结构

- 事物 = 顶点

- 关系 = 边

而现实世界中到处都是图，较为常见的有：

| 场景   | 顶点    | 边     |
|------|-------|-------|
| 社交网络 | 用户    | 好友、关注 |
| 知识图谱 | 概念    | 关联    |
| 金融   | 账户    | 转账    |
| 组织架构 | 员工    | 汇报关系  |

在社交网络中，A和B是朋友，B和C是朋友，就很有可能会向A推荐：可能认识的人—— C，反过来说想要找一下A和C的共同好友也非常简单。后面我们将以这个示例，展示一下这个新的功能。

## PostgreSQL SQL/PGQ的简单使用
这里我们只有一个要求，就是PostgreSQL的版本需要足够的新（19devel），所以有条件的朋友可以手动编译一下PostgreSQL的内核代码。接下来看一下这个属性图的创建语法。
```
postgres@zxm-VMware-Virtual-Platform:~$ psql
psql (19devel)
Type "help" for help.

postgres=# \h CREATE PROPERTY GRAPH
Command:     CREATE PROPERTY GRAPH
Description: define an SQL-property graph
Syntax:
CREATE [ TEMP | TEMPORARY ] PROPERTY GRAPH name
    [ {VERTEX|NODE} TABLES ( vertex_table_definition [, ...] ) ]
    [ {EDGE|RELATIONSHIP} TABLES ( edge_table_definition [, ...] ) ]

where vertex_table_definition is:

    vertex_table_name [ AS alias ] [ KEY ( column_name [, ...] ) ] [ element_table_label_and_properties ]

and edge_table_definition is:

    edge_table_name [ AS alias ] [ KEY ( column_name [, ...] ) ]
        SOURCE [ KEY ( column_name [, ...] ) REFERENCES ] source_table [ ( column_name [, ...] ) ]
        DESTINATION [ KEY ( column_name [, ...] ) REFERENCES ] dest_table [ ( column_name [, ...] ) ]
        [ element_table_label_and_properties ]

and element_table_label_and_properties is either:

    NO PROPERTIES | PROPERTIES ALL COLUMNS | PROPERTIES ( { expression [ AS property_name ] } [, ...] )

or:

   { { LABEL label_name | DEFAULT LABEL } [ NO PROPERTIES | PROPERTIES ALL COLUMNS | PROPERTIES ( { expression [ AS property_name ] } [, ...] ) ] } [...]
```
更多详细信息请查看[官方文档](https://www.postgresql.org/docs/devel/sql-create-property-graph.html) 。
### Graph Queries 
正确创建完属性图之后，我们还需要了解了解关于图的查询。基本语法如下：
```
SELECT ... FROM GRAPH_TABLE (图名 MATCH ... COLUMNS (...))
```
MATCH后接图模式、COLUMNS 表示输出的字段，官方示例如下：
```
SELECT customer_name FROM GRAPH_TABLE (myshop MATCH (c IS customers)-[IS customer_orders]->(o IS orders WHERE o.ordered_when = current_date) COLUMNS (c.name AS customer_name));
```

图模式的一些符号：
```
| 符号        | 含义           |
|-----------|--------------|
| ()        | 顶点模式（匹配单个顶点） |
| -[]->     | 边模式（匹配边）     |
| ()-[]->() | 顶点-边-顶点序列    |

标签匹配（使用 IS labelname）：
(IS person)                                    -- 匹配标签为 person 的顶点
(IS person)-[IS has]->(IS account)             -- person -has-> account
(IS person)-[IS has]->(IS account|creditcard)  -- 多标签（OR 语义）

边方向：
(IS person)-[IS has]->(IS account)      -- 正向
(IS account)<-[IS has]-(IS person)      -- 反向
(IS person)-[IS is_friend_of]-(IS person)  -- 无向（双向匹配）

简写语法（省略 []）：
(IS person)->(IS account)      -- 省略边的 []
(IS account)<-(IS person)      -- 反向
(IS person)-(IS person)        -- 无向
```

更多详细信息请查看[官方文档](https://www.postgresql.org/docs/devel/queries-graph.html)。

### 测试数据准备
创建测试相关的表和插入部分数据。
```
postgres=# CREATE TABLE users (
    id INT PRIMARY KEY,
    name VARCHAR(50)
);
CREATE TABLE follows (
    id INT PRIMARY KEY,
    follower INT REFERENCES users(id),
    following INT REFERENCES users(id)
);
INSERT INTO users VALUES
    (1, 'Alice'), (2, 'Bob'), (3, 'Charlie'), (4, 'David');
INSERT INTO follows VALUES
    (1, 1, 2),  -- Alice follows Bob
    (2, 1, 3),  -- Alice follows Charlie
    (3, 2, 3),  -- Bob follows Charlie
    (4, 2, 4),  -- Bob follows David
    (5, 3, 4);  -- Charlie follows David
CREATE TABLE
CREATE TABLE
INSERT 0 4
INSERT 0 5
```
### 创建属性图
```
postgres=# CREATE PROPERTY GRAPH social_graph
    VERTEX TABLES (users)
    EDGE TABLES (
        follows
            SOURCE KEY (follower) REFERENCES users(id)
            DESTINATION KEY (following) REFERENCES users(id)
    );
CREATE PROPERTY GRAPH
```
让大模型简单解释一下~
 | 行号  | 代码                                               | 含义                                 |
  |-----|--------------------------------------------------|------------------------------------|
  | 1   | CREATE PROPERTY GRAPH social_graph               | 创建一个名为 social_graph 的属性图           |
  | 2   | VERTEX TABLES (users)                            | 指定 users 表作为顶点表，表中每一行代表一个顶点        |
  | 3-5 | EDGE TABLES (...)                                | 指定 follows 表作为边表                   |
  | 4   | SOURCE KEY (follower) REFERENCES users(id)       | 边的起点：follows.follower 指向 users.id  |
  | 5   | DESTINATION KEY (following) REFERENCES users(id) | 边的终点：follows.following 指向 users.id |

### 共同关注
让我们看一下Alice和Bob的共同关注。这里我们只要注意一下方向就好了。
```
-- Alice 关注的人：Bob, Charlie
-- Bob 关注的人：Charlie, David
-- 结论：Alice和Bob的共同关注是 Charlie

postgres=# SELECT common_name
FROM GRAPH_TABLE (
    social_graph
    MATCH (a IS users)-[]->(x IS users)<-[]-(b IS users)
    WHERE a.name = 'Alice' AND b.name = 'Bob'
    COLUMNS (x.name AS common_name)
);
 common_name 
-------------
 Charlie
(1 row)
```
这里我让大模型生成了一条等价的CTE语句（实际上还有很多等价的SQL语句， 后面我们会提到）
```
postgres=# WITH
  -- Alice 关注的人
  alice_follows AS (
      SELECT f.following AS user_id
      FROM users u
      JOIN follows f ON f.follower = u.id
      WHERE u.name = 'Alice'
  ),

  -- Bob 关注的人
  bob_follows AS (
      SELECT f.following AS user_id
      FROM users u
      JOIN follows f ON f.follower = u.id
      WHERE u.name = 'Bob'
  )

  -- 交集：共同关注的人
  SELECT u.name AS common_name
  FROM alice_follows a
  JOIN bob_follows b ON a.user_id = b.user_id
  JOIN users u ON u.id = a.user_id;
 common_name 
-------------
 Charlie
(1 row)
```
### 可能认识的人
让我们找一下Alice 可能认识的人，也就是朋友的朋友。这里我们需要保证就是最终输出的不能是Alice 关注过的人。
```
-- Alice 关注的人：Bob, Charlie
-- Bob 关注的人：Charlie, David
-- Charlie 关注的人：David
-- 结论：Alice 可能认识的人是 David

postgres=# SELECT DISTINCT gt.person
FROM GRAPH_TABLE (social_graph
  MATCH (alice IS users) -[]-> (friend IS users) -[]-> (fof IS users)
  WHERE alice.name = 'Alice' AND fof.name != 'Alice'
  COLUMNS (fof.name AS person)
) AS gt
WHERE gt.person NOT IN (
  SELECT name
  FROM GRAPH_TABLE (social_graph
      MATCH (u1 IS users) -[]-> (u2 IS users)
      WHERE u1.name = 'Alice'
      COLUMNS (u2.name)
  ) AS g
);
 person 
--------
 David
(1 row)
```
这里先找了Alice朋友的朋友
```
SELECT DISTINCT gt.person
FROM GRAPH_TABLE (social_graph
  MATCH (alice IS users) -[]-> (friend IS users) -[]-> (fof IS users)
  WHERE alice.name = 'Alice' AND fof.name != 'Alice'
  COLUMNS (fof.name AS person)
)
```
这里找到Alice直接关注过的人
```
  SELECT name
  FROM GRAPH_TABLE (social_graph
      MATCH (u1 IS users) -[]-> (u2 IS users)
      WHERE u1.name = 'Alice'
      COLUMNS (u2.name)
  )
```
然后做一些过滤。

照例给出等价的CTE语句
```
postgres=# WITH
-- CTE 1: 找 Alice 直接关注的人
alice_follows AS (
  SELECT f.following AS user_id
  FROM users u
  JOIN follows f ON f.follower = u.id
  WHERE u.name = 'Alice'
),

-- CTE 2: 找朋友的朋友
two_hops AS (
  SELECT f2.following AS user_id
  FROM users alice
  JOIN follows f1 ON f1.follower = alice.id
  JOIN follows f2 ON f2.follower = f1.following
  WHERE alice.name = 'Alice'
    AND f2.following != alice.id
)

-- 最终结果
SELECT DISTINCT u.name AS person
FROM two_hops th
JOIN users u ON u.id = th.user_id
WHERE th.user_id NOT IN (SELECT user_id FROM alice_follows);
 person 
--------
 David
(1 row)
```
简单总结：SQL/PGQ 对比SQL确实是更加直观，写起来也非常的舒服。不过值得注意的是PostgreSQL SQL/PGQ其实是“换汤不换药”。
它的本质上来说依旧还是“SQL”，只不过是当输入的是SQL/PGQ时，在rewrite阶段，改写成了等价SQL的数据结构。

## pg_pgq2sql
我做了一个简单的插件——[pg_pgq2sql](https://github.com/Z-Xiao-M/pg_pgq2sql)，它提供的功能其实就是SQL/PGQ 查询被转换为等价的SQL语句。
```
postgres=# create extension pg_pgq2sql;
CREATE EXTENSION
postgres=# select pg_pgq2sql($$SELECT common_name
FROM GRAPH_TABLE (
    social_graph
    MATCH (a IS users)-[]->(x IS users)<-[]-(b IS users)
    WHERE a.name = 'Alice' AND b.name = 'Bob'
    COLUMNS (x.name AS common_name)
);$$);
                                                                                                                    pg_pgq2sql                                                                                                                    
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
  SELECT common_name                                                                                                                                                                                                                             +
    FROM LATERAL ( SELECT users_1.name AS common_name                                                                                                                                                                                            +
            FROM users,                                                                                                                                                                                                                          +
             follows,                                                                                                                                                                                                                            +
             users users_1,                                                                                                                                                                                                                      +
             follows follows_1,                                                                                                                                                                                                                  +
             users users_2                                                                                                                                                                                                                       +
           WHERE users.id = follows.follower AND users_1.id = follows.following AND users_2.id = follows_1.follower AND users_1.id = follows_1.following AND users.name::text = 'Alice'::text AND users_2.name::text = 'Bob'::text) "graph_table"
(1 row)

postgres=# SELECT common_name
   FROM LATERAL ( SELECT users_1.name AS common_name
           FROM users,
            follows,
            users users_1,
            follows follows_1,
            users users_2
          WHERE users.id = follows.follower AND users_1.id = follows.following AND users_2.id = follows_1.follower AND users_1.id = follows_1.following AND users.name::text = 'Alice'::text AND users_2.name::text = 'Bob'::text) "graph_table";
 common_name 
-------------
 Charlie
(1 row)

postgres=# select pg_pgq2sql($$SELECT DISTINCT gt.person
FROM GRAPH_TABLE (social_graph
  MATCH (alice IS users) -[]-> (friend IS users) -[]-> (fof IS users)
  WHERE alice.name = 'Alice' AND fof.name != 'Alice'
  COLUMNS (fof.name AS person)
) AS gt
WHERE gt.person NOT IN (
  SELECT name
  FROM GRAPH_TABLE (social_graph
      MATCH (u1 IS users) -[]-> (u2 IS users)
      WHERE u1.name = 'Alice'
      COLUMNS (u2.name)
  ) AS g
);$$);
                                                                                                                pg_pgq2sql                                                                                                                
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
  SELECT DISTINCT person                                                                                                                                                                                                                 +
    FROM LATERAL ( SELECT users_2.name AS person                                                                                                                                                                                         +
            FROM users,                                                                                                                                                                                                                  +
             follows,                                                                                                                                                                                                                    +
             users users_1,                                                                                                                                                                                                              +
             follows follows_1,                                                                                                                                                                                                          +
             users users_2                                                                                                                                                                                                               +
           WHERE users.id = follows.follower AND users_1.id = follows.following AND users_1.id = follows_1.follower AND users_2.id = follows_1.following AND users.name::text = 'Alice'::text AND users_2.name::text <> 'Alice'::text) gt+
   WHERE NOT (person::text IN ( SELECT g.name                                                                                                                                                                                            +
            FROM LATERAL ( SELECT users_1.name                                                                                                                                                                                           +
                    FROM users,                                                                                                                                                                                                          +
                     follows,                                                                                                                                                                                                            +
                     users users_1                                                                                                                                                                                                       +
                   WHERE users.id = follows.follower AND users_1.id = follows.following AND users.name::text = 'Alice'::text) g))
(1 row)

postgres=#  SELECT DISTINCT person
   FROM LATERAL ( SELECT users_2.name AS person
           FROM users,
            follows,
            users users_1,
            follows follows_1,
            users users_2
          WHERE users.id = follows.follower AND users_1.id = follows.following AND users_1.id = follows_1.follower AND users_2.id = follows_1.following AND users.name::text = 'Alice'::text AND users_2.name::text <> 'Alice'::text) gt
  WHERE NOT (person::text IN ( SELECT g.name
           FROM LATERAL ( SELECT users_1.name
                   FROM users,
                    follows,
                    users users_1
                  WHERE users.id = follows.follower AND users_1.id = follows.following AND users.name::text = 'Alice'::text) g));
 person 
--------
 David
(1 row)
```
我们可以看到的是，实际上转换后的SQL并没有使用CTE，也没有很多的显式JOIN，而是采用了LATERAL，原因可能是这样子构造较为方便？我没有仔细去翻阅那些个邮件，因为实在是太多了。所以如果是复杂或者多跳的场景，这个功能的执行效率可能并不会特别高，感兴趣的朋友可以自行测试着玩玩，可能和大手子写的SQL执行时间相差几个数量级。不过还是很高兴看到PostgreSQL引入了这个功能，起码解决了有没有这个问题，虽然这个功能对于部分SQL大佬而言可能就是像c++中语法糖一般的存在。

## 注意事项
如果想要和我展示的pg_pgq2sql获得的一样的效果。目前最新的内核代码还是有点问题的，需要等`https://commitfest.postgresql.org/patch/6602/`被合并到内核代码中，或者自行打一下这个patch。