---
title: LBS-查找附近的人-mongodb-spring实现
date: 2018-04-08 21:30:46
category: LBS
tags:
  - LBS
toc: true
excerpt: "LBS应用中查找附近的人, 使用mongodb的高效解决方案, 自带过期设置, 快速集成。系类文章有mysql、redis和mongodb三种解决方案，方便读者自由选择。"
header:
  teaser: /assets/images/lbs/lbs-img10.jpg
  overlay_image: /assets/images/lbs/lbs-img11.jpg
  overlay_filter: 0.6
---

前面介绍了地理坐标基础知识，mysql、redis、mongodb基础知识的解决方案

* [LBS-查找附近的人-地理坐标定位详解](http://nilme.me/lbs/LBS-查找附近的人-地理坐标定位详解.html)
* [LBS-查找附近的人-MySQL实现](http://nilme.me/lbs/LBS-查找附近的人-MySQL实现.html)
* [LBS-查找附近的人-redis命令实现](http://nilme.me/lbs/LBS-查找附近的人-redis命令实现.html)
* [LBS-查找附近的人-redis-spring实现](http://nilme.me/lbs/LBS-查找附近的人-redis-spring实现.html)
* [LBS-查找附近的人-mongodb实现-基础知识](http://nilme.me/lbs/LBS-查找附近的人-mongodb实现-基础知识.html)

接下来我们开始介绍mongodb结合spring-data的解决方案。

通过上一篇的学习，我们可以知道使用 **GeoJSON对象** 来保存用户的位置数据比较合适。

在开始前我们先初始化一些测试数据,为了后面的测试我们一开始就通过for循环创建600w个随机数据。

### 创建数据

创建集合-插入数据-建立索引

```sql
#创建集合
db.createCollection("mongoGeoUser")
#插入数据
db.mongoGeoUser.insert({
   _id: 1,
   name: "user1",
   createdAt: new Date(),
   location: { type: "Point", coordinates: [ 120.1333737373, 30.2535809303 ] },
} );
#在location字段上创建索引
db.mongoGeoUser.createIndex({ location: "2dsphere" })
```

spring-data实现

```java
@Document(collection = "mongoGeoUser")  //指定文档名称
public class MongoGeoUser {
    @Id
    private Long id;
    private String name;
    private GeoJsonPoint location;
    private Date createdAt;

    public MongoGeoUser(Long id, String name, GeoJsonPoint location, Date createdAt) {
        this.id = id;
        this.name = name;
        this.location = location;
        this.createdAt = createdAt;
    }
    //geter and seter
}
```

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class MongoDBTests {
    @Autowired
    private MongoTemplate mongoTemplate;
    /**
     * 更新或插入对象
     */
    @Test
    public void createMongoGeoUsers() {//批量添加,600w条数据比redis添加慢了很多很多，要等一小会的，建议泡杯茶，回头复习一下之前的文章
        for (int i = 0; i < 6000000; i++) {
            BigDecimal lng = new BigDecimal(Math.random() * (130 - 100) + 100).setScale(10, BigDecimal.ROUND_HALF_UP);
            BigDecimal lat = new BigDecimal(Math.random() * 20 + 20).setScale(10, BigDecimal.ROUND_HALF_UP);

            MongoGeoUser mongoGeoUser = new MongoGeoUser(new Long(i), "user1", new GeoJsonPoint(lng.doubleValue(), lat.doubleValue()), new Date());
            mongoTemplate.save(mongoGeoUser);
        }
    }
}
```

创建好600w条数据后，开始我们的测试，切记，不要忘记建立索引

```sql
> db.mongoGeoUser.count();
6000000
```

### 在球面中查询附近1km~5km的点

```java
@Test
public void nearSphere() {
    Query query = new Query(Criteria.where("location")
            .nearSphere(new GeoJsonPoint(120.666666666, 30.888888888))
            .maxDistance(5000).minDistance(1000));
    List<MongoGeoUser> venues = mongoTemplate.find(query, MongoGeoUser.class);

    for (MongoGeoUser venue : venues) {
        System.out.println(venue);
    }
}
```

MongoDB支持查询数据库中的地理位置，并同时计算与给定原点的距离。通过地理近似查询，可以表达如下查询：“查找周围1km~5km内附近的人”，并返回距离信息。使用上面的 **Query** 是无法得到距离信息的

```java
@Test
public void nearQuery() {
    long startTime = System.currentTimeMillis();
    NearQuery near = NearQuery
            .near(new GeoJsonPoint(120.666666666, 30.888888888))
            .spherical(true)
            .maxDistance(5, Metrics.KILOMETERS)
            .minDistance(1, Metrics.KILOMETERS)
            .num(10);
    GeoResults<MongoGeoUser> results = mongoTemplate.geoNear(near, MongoGeoUser.class);

    long endTime = System.currentTimeMillis();
    System.out.println("程序运行时间：" + (endTime - startTime) + "ms");

    for (GeoResult<MongoGeoUser> result : results) {
        System.out.println(result);
    }
}
```

上面两个方法就是我们解决查找附近的人的主要解决方案，网上其他的方案都住要使用 **传统坐标对** 的方式存储位置信息，本文采用的是 **GeoJSON对象** , 实际上两个存储方案没有很大的差别，官方建议再球面上的坐标使用 **GeoJSON对象** 来存储，**传统坐标对** 存储的数据再计算球面数据的时候也可以指定 **spherical(true)** 来转化成球面的计算值，再距离不是很大的情况下具体采用什么存储方案没有很大的差别。

最后还是到了我们关心的运行效率，上面两个方案的效率大约都是 **300ms** 左右的时间。对比上一篇的redis方案，稍稍慢了一点点，可能一个是内存数据库，一个不是吧，但是不管怎么样都还是符合生产环境的使用的。

上面的 **NearQuery** 和 **Query** 还有很多其他的参数和查询方法，比如 矩形内的点，还有平面内的坐标计算等。需要分页的应用也是可以分页操作的。

其他具体的参数可以参考 [spring-data-mongo文档](https://docs.spring.io/spring-data/data-mongo/docs/1.10.0.M1/reference/html/#mongodb-template-query) 和 [mongodb官方文档](https://docs.mongodb.com/manual/geospatial-queries/#index-feature-geospatial)

### 设置自动过期

MongoDB也可以像redis那样为key设置一个过期时间，但是redis-geo是采用zset实现的，zset无法对member设置过期时间。MongoDB对过期时间字段设置索引的方式来处理自动过期。

上面的 **createdAt** 字段就是用来设置过期时间的。

创建索引

```sql
db.log_events.createIndex({ "createdAt": 1 }, { expireAfterSeconds: 3600 })
```

该记录会在 createdAt 时间的 expireAfterSeconds 秒后失效。

也可以设置一个特殊的过期时间点

```sql
db.log_events.insert( {
   "expireAt": new Date('July 22, 2013 14:00:00'),
   "logEvent": 2,
   "logMessage": "Success!"
} )
```

设置索引

```sql
db.log_events.createIndex( { "expireAt": 1 }, { expireAfterSeconds: 0 } )
```

### 结束语

到此我们的LBS-查找附近的人的所有解决方案就已经结束了，各位读者可以根据自己项目的情况需求来选择合适的方案，不管是mysql、redis还是mongodb都是在300w甚至600w数据的下测试过，各个方案都有自己的优点和缺点。

不同类型的APP对LBS系统的读写压力完全不同。例如，对于附近商家、美甲等O2O类的APP，其更新和获取LBS数据的频率很低，但是对于打车类APP，因为需要频繁的更新地理位置数据，LBS后台需要承担的读写压力要比一般的APP压力大。在快的公开的资料中LBS系统每秒的读写次数比居然达到4:1。

> 本系列文章都是基于静态的数据进行测试，当时没有写入的操作，如果写压力过大也是会导致查询的效率下降的。当然如果写入的压力过大通过主从和读写分离也还是可以处理的，如果是数据量过大mongodb还有分片的架构可以处理，总之现在给出来的几个方案都是被大量的实践所验证过的，大胆放心的使用就行。

系统不要过度架构，如果可以遇见数据的增长量的话，选择一些简单的方案可以在后面节约不少时间。
