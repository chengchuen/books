# MongoDB指南

MongoDB 是通用、基于文档的分布式数据库。支持完整的 ACID 事务，具有强大的查询语言等。属于NoSQL数据库。

## 环境搭建

```
 docker run -d --net=host --name mongo mongo:5.0.2-focal
```

这样就可以在本地启动一个MongoDB服务，默认开启mongos端口是`27017`。

## 基本概念

| SQL术语/概念 | MongoDB术语/概念 | 解释/说明                           |
| :----------- | :--------------- | :---------------------------------- |
| database     | database         | 数据库                              |
| table        | collection       | 数据库表/集合                       |
| row          | document         | 数据记录行/文档                     |
| column       | field            | 数据字段/域                         |
| index        | index            | 索引                                |
| table joins  |                  | 表连接,MongoDB不支持                |
| primary key  | primary key      | 主键,MongoDB自动将_id字段设置为主键 |

![](images/mongdb实例结构图.png)

## CRUD操作

使用`mongosh`命令即可进入Cli工具，如果是有密码的，可以使用如下命令:

```
mongosh mongodb://root:root@localhost:27017/truman_test?retryWrites=false
```

### 数据库DDL

```
// 创建/切换到指定数据库
>use truman_test
// 查询所有的数据库
truman_test> show dbs
// 删除数据
truman_test> use sns
truman_test> db.dropDatabase()
```

### 集合DDL

```
// 创建collection
truman_test> db.createCollection("all_raw_data")
// 查询所有的collection
truman_test> show collections
// 创建索引
truman_test> db.all_raw_data.createIndex({"status": 1})
// 设置ttl为30d
truman_test> db.all_raw_data.createIndex( { "inTime": 1 }, { expireAfterSeconds: 43200 } )
// 删除集合
truman_test> db.all_raw_data.drop()
```

### 文档DDL

```
// 插入文档
truman_test>  db.all_raw_data.insertOne({name:'truman',age:18,status:'success'})
// 修改文档
truman_test> db.all_raw_data.updateOne({name:'truman'},{$set:{age:19}})
// 删除文档
truman_test> db.all_raw_data.deleteOne( { status: "success" } )
```

### 查询

```
truman_test> db.all_raw_data.find().pretty()
// 查询age大于21，或者name=truman
truman_test> db.all_raw_data.find({$or:[{age:{$gt:21}},{name:'truman'}]}).pretty()

```

查询语法：

![](images/mongodb查询语法.png)

[更多文档详见官方文档](https://docs.mongodb.com/manual/tutorial/query-documents/)

## 适用场景



## 问题与经验

### document能存储最大数据？

MongoDB默认的BSON最大支持16M,对于超过16M的场景，可以考虑使用MongoDB GridFS。

## Spring Data MongoDB使用

推荐两个官网文档：[Accessing Data with MongoDB](https://spring.io/guides/gs/accessing-data-mongodb/) 、[Spring Data MongoDB](https://docs.spring.io/spring-data/mongodb/docs/3.2.5/reference/html/#introduction)

首先在项目中引入依赖：

```
<dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-mongodb</artifactId>
</dependency>
```

### 配置参数与定义实体

首先在配置文件增加mongo配置信息

```
spring.data.mongodb.uri=mongodb://localhost:27017/truman_test?retryWrites=false
```

然后新增实体

```
@Document("all_raw_data")
public class RawData {
    @Id
    private String id;
    private String name;
    private int age;
    /**
     * wait,success,fail,canceled
     */
    private String status;
    private Date inTime;
   // 省略set与get
}
```

### 编写Repository

定义`RawDataRepository`继承`MongoRepository`，这样就可以不用写任何代码，实现数据的增删改查，基本用法和Spring JPA类似。

```
public interface RawDataRepository extends MongoRepository<NotificationRawData, String> {
    /**
     * 根据状态分页查询数据
     *
     * @param status
     * @param pageable
     * @return
     */
    public Page<RawData> findAllByStatusEquals(String status, Pageable pageable);
}
```

#### 公共CRUD

```
    <S extends T> S save(S var1);

    <S extends T> Iterable<S> saveAll(Iterable<S> var1);

    Optional<T> findById(ID var1);

    boolean existsById(ID var1);

    Iterable<T> findAll();

    Iterable<T> findAllById(Iterable<ID> var1);

    long count();

    void deleteById(ID var1);

    void delete(T var1);

    void deleteAll(Iterable<? extends T> var1);

    void deleteAll();
```

#### 自定义查询

```
//根据状态统计数量
RawData rawData = new RawData();
rawData.setStatus("wait");
mongoDbRepository.count(Example.of(rawData));
// 在Repository自定义查询，通过固定语法例如findAllByXXXOrder....
public Page<RawData> findAllByStatusEquals(String status, Pageable pageable);
```



#### 分页查询

```
// 从下标0开始读取，读取1条记录
Pageable pageable = PageRequest.of(0, 1, Sort.by(Sort.Order.asc(Constant.MessageHeaderConstant.IN_TIME)));
Page<NotificationRawData> page = mongoDBRepository.findAllByStatusEquals("wait", pageable);
// 根据数据执行相应的业务逻辑
execute(page.getContent());
while (page.hasNext()) {
      page = mongoDBRepository.findAllByStatusEquals("wait", page.nextPageable());
      execute(page.getContent());
}
```

## 参考

1. [MongoDB 教程](https://www.runoob.com/mongodb/mongodb-tutorial.html)
2. [MongoDB 基础浅谈](https://mp.weixin.qq.com/s/CWjPigYjnREPXTiIRXI6MA)
3. [The MongoDB 5.0 Manual](https://docs.mongodb.com/manual/)

