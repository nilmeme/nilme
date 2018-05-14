---
title: LBS-查找附近的人-redis-spring实现
date: 2018-04-06 21:12:03
category: LBS
tags:
  - LBS
excerpt: "LBS应用中查找附近的人, 使用redis的高效解决方案"
header:
  teaser: /assets/images/lbs/lbs-img8.jpg
  overlay_image: /assets/images/lbs/lbs-img9.jpg
  overlay_filter: 0.6
---

前面介绍了地理坐标定位相关的基础知识和查找附近人的MySQL版实现和redis版的6个基础命令，本则主要介绍redis+spring版的实现。

代码主要对redis-geo的6个命令做了demo的测试，在最后我们生成600w条数据测试一下redis的geo实现效率如何。没错，mysql版我们使用300w，redis当然要更高一些来体现redis方案的优越性。

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class GEOTests {
    @Autowired
    public RedisTemplate<String, String> redisTemplate;

    /**
     * 前缀
     */
    public static final String KEY_PREFIX_GEO = "geo_test_spring";

    @Test
    public void addGeo() {//单条添加
        Point point = new Point(120.1384162903, 30.2532102251);
        redisTemplate.opsForGeo().geoAdd(KEY_PREFIX_GEO, point, "user1");
    }

    @Test
    public void addGeos() {//批量添加
        HashMap<String, Point> map = new HashMap<>();
        map.put("user2", new Point(120.1389124356, 30.2532779243));
        map.put("user3", new Point(120.1384162904, 30.2532102251));
        redisTemplate.opsForGeo().geoAdd(KEY_PREFIX_GEO, map);
    }

    @Test
    public void remove() {//删除一个元素
        redisTemplate.opsForGeo().geoRemove(KEY_PREFIX_GEO, "user3");
    }

    @Test
    public void GEOHASH(){
        GeoOperations<String, String> geoOps = redisTemplate.opsForGeo();
        List<String> strings = geoOps.geoHash(KEY_PREFIX_GEO, "user1");
        for (String string : strings) {
            System.out.println(string);
        }
    }

    @Test
    public void GEOPOS(){
        GeoOperations<String, String> geoOps = redisTemplate.opsForGeo();
        List<Point> points = geoOps.geoPos(KEY_PREFIX_GEO, "user1");
        for (Point point : points) {
            System.out.println(point);
        }
    }

    @Test
    public void GEODIST(){
        GeoOperations<String, String> geoOps = redisTemplate.opsForGeo();
        Distance distance = geoOps.geoDist(KEY_PREFIX_GEO, "user1", "user2",DistanceUnit.KILOMETERS);
        System.out.println(distance);
    }

    @Test
    public void GEORADIUS() {
        long startTime = System.currentTimeMillis();    //获取开始时间

        GeoOperations<String, String> geoOps = redisTemplate.opsForGeo();
        //设置geo查询参数
        GeoRadiusCommandArgs geoRadiusArgs = GeoRadiusCommandArgs.newGeoRadiusArgs();
        geoRadiusArgs = geoRadiusArgs.includeCoordinates().includeDistance();//查询返回结果包括距离和坐标
        geoRadiusArgs.sortAscending();//按查询出的坐标距离中心坐标的距离进行排序
        geoRadiusArgs.limit(100);//限制查询数量

        GeoResults<GeoLocation<String>> radiusGeo = geoOps.geoRadius(
                KEY_PREFIX_GEO,
                new Circle(new Point(120.1384162903, 30.2532102251), new Distance(5, DistanceUnit.KILOMETERS)),
                geoRadiusArgs);

        long endTime = System.currentTimeMillis();    //获取结束时间
        System.out.println("程序运行时间：" + (endTime - startTime) + "ms");    //输出程序运行时间
//        for (GeoResult<GeoLocation<String>> geoLocationGeoResult : radiusGeo) {
//            System.out.println(geoLocationGeoResult);
//        }
    }

    @Test
    public void GEORADIUSBYMEMBER() {
        long startTime = System.currentTimeMillis();    //获取开始时间

        GeoOperations<String, String> geoOps = redisTemplate.opsForGeo();
        //设置geo查询参数
        GeoRadiusCommandArgs geoRadiusArgs = GeoRadiusCommandArgs.newGeoRadiusArgs();
        geoRadiusArgs = geoRadiusArgs.includeCoordinates().includeDistance();//查询返回结果包括距离和坐标
        geoRadiusArgs.sortAscending();//按查询出的坐标距离中心坐标的距离进行排序
        geoRadiusArgs.limit(100);//限制查询数量

        GeoResults<GeoLocation<String>> radiusGeo = geoOps.geoRadiusByMember(
                KEY_PREFIX_GEO, "user1", new Distance(10, DistanceUnit.KILOMETERS), geoRadiusArgs);

        long endTime = System.currentTimeMillis();    //获取结束时间
        System.out.println("程序运行时间：" + (endTime - startTime) + "ms");    //输出程序运行时间

//        List<GeoResult<GeoLocation<String>>> content = radiusGeo.getContent();
//        for (GeoResult<GeoLocation<String>> geoLocationGeoResult : content) {
//            System.out.println(geoLocationGeoResult.getContent());
//        }
    }

    @Test
    public void addGeosp() {//批量添加,600w条数据还是要那么一小会的，但是添加速度比mysql快多多了
        for (int i = 0; i < 6000000; i++) {
            BigDecimal lng = new BigDecimal(Math.random() * (130 - 100) + 100).setScale(10, BigDecimal.ROUND_HALF_UP);
            BigDecimal lat = new BigDecimal(Math.random() * 20 + 20).setScale(10, BigDecimal.ROUND_HALF_UP);
            redisTemplate.opsForGeo().geoAdd(KEY_PREFIX_GEO, new Point(lng.doubleValue(), lat.doubleValue()), "user" + i);
        }
    }
}
```

代码中已经添加了注释，所以不过多解释。

spring-data-redis 还有另外一种添加数据的方式，使用的是JPA的方式， [spring-data-redis文档](https://docs.spring.io/spring-data/redis/docs/2.1.0.M2/reference/html/#redis.repositories.indexes.geospatial)，有兴趣也可以试一下，本人试了一下感觉没有很强的实际意义，如果你想把其他一些需要持久化的数据也存进去的话，也可以选择。

现在我们自动生成600w条数据测试性能，结果是 **GEORADIUS** 和 **GEORADIUSBYMEMBER** 查询600w条数据并计算距离并排序耗时 **80ms** 左右，效率非常的高。唯一的不足就是不方便分页。

有时候我们的地图应用可能类是嘀嘀打车，司机的位置信息，需要实时更新，更新没有问题，但是司机下线了位置信息无法自动删除。因为redis的过期时间为key设置，没法对zset里面的member设置。勉强的做法可以用sorted set，把要过期的member和key的信息放在sorted set的member里，把过期时间放在score中。跑个任务用zrangebyscore遍历就行了。用sorted set好处是只需要遍历过期的member，不用扫描整个过期member集合。不过这个删除的操作还是交给应用程序主动调用吧，毕竟司机有 **下班** 这个操作，触发操作后主动删除对应的位置信息就可以。

到此查找附近的人redis解决方案也已经完成，请看下篇mongodb解决方案。
