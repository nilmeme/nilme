---
title: LBS-查找附近的人-mongodb实现-基础知识
date: 2018-04-07 20:56:46
category: LBS
tags:
  - LBS
toc: true
excerpt: "LBS应用中查找附近的人, 使用mongodb的高效解决方案, 自带过期设置, 快速集成"
header:
  teaser: /assets/images/lbs/lbs-img10.jpg
  overlay_image: /assets/images/lbs/lbs-img11.jpg
  overlay_filter: 0.6
---

前面介绍了地理坐标基础知识，mysql和redis的解决方案

* [LBS-查找附近的人-地理坐标定位详解](http://nilme.me/lbs/LBS-查找附近的人-地理坐标定位详解.html)
* [LBS-查找附近的人-MySQL实现](http://nilme.me/lbs/LBS-查找附近的人-MySQL实现.html)
* [LBS-查找附近的人-redis命令实现](http://nilme.me/lbs/LBS-查找附近的人-redis命令实现.html)
* [LBS-查找附近的人-redis-spring实现](http://nilme.me/lbs/LBS-查找附近的人-redis-spring实现.html)

接下来我们开始介绍mongodb的解决方案。

## 地理空间数据

在MongoDB中，可以将地理空间数据存储为 **GeoJSON对象** 或 **传统坐标对** 。

## GeoJSON对象

要计算类球体上的几何体，位置数据应存储为GeoJSON对象。

要指定GeoJSON数据，请使用嵌入式文档：

- type：指定GeoJSON对象类型
- coordinates：指定对象的坐标。

```sql
<field>: { type: <GeoJSON type> , coordinates: <coordinates> }
```

如：

```sql
location: {
      type: "Point",
      coordinates: [-73.856077, 40.848447]
}
```

GeoJSON对象在MongoDB地理空间查询在球体上计算，使用WGS84参考系统对GeoJSON对象进行地理空间查询。

## 传统坐标对

要计算欧几里得平面上的距离，位置数据应存储为传统坐标对，并使用2d索引。将数据转换为GeoJSON Point类型后，并通过2dsphere索引后传统坐标对也支持球面曲面计算。

要将数据指定为传统坐标对，也有两种数据格式可以使用，**数组** 或 **嵌入式文档** 。 官方推荐首先使用 **数组**。

数组方式（首选）：

```sql
<field>:[<longitude>, <latitude>]
```

嵌入式文档方式：

```sql
<field>:{<field1>:<longitude>, <field2>:<latitude>}
```

* 有效经度值介于-180和180。
* 有效的纬度值介于-90和90。

> 注意：为什么要说这个经纬度的有效值呢，因为mongodb不仅仅是对地球的经纬系统支持，也支持其他的平面坐标系。如果使用纬度和经度坐标必须使用 longitude（经度）在前，latitude（纬度）在后的顺序。

## 地理空间索引

在MongoDB中，地理数据相关的索引有两种 **2dsphere** 和 **2d**。地球状球体计算几何的查询应使用 **2dsphere** 索引。 **2d** 索引在二维平面上使用存储为点的数据的索引。该 **2d** 索引适用于MongoDB 2.2及更早版本中使用的传统坐标对。

当然通过将数据转换为GeoJSON Point类型，MongoDB通过2dsphere索引支持传统坐标对上的球面曲面计算。所以 **GeoJSON对象** 比 **传统坐标对** 更加强大复杂，但是 **传统坐标对** 也是支持 **2dsphere**

两种索引的方式虽然不同，不过，只要坐标跨度不太大（比如几百几千公里），这两个索引计算出的距离相差几乎可以忽略不计。

1. 2dsphere

2dsphere索引支持在地球球上计算几何的查询。

```sql
db.collection.createIndex({ <location field>: "2dsphere" })
```

2. 2d

2d索引支持在二维平面上计算几何的查询 。尽管索引可以支持 **$nearSphere** 在球体上计算的查询，但如果可能的话，请使用 **2dsphere** 索引进行球形查询。

```sql
db.collection.createIndex({ <location field> : "2d" })
```

其中 **<location field>** 的值是 **GeoJSON对象** 或 **传统坐标对** 的字段。

## 地理空间查询运算符

MongoDB提供了以下地理空间查询操作符：

|名称|描述|
|:---|:---|
|$geoIntersects|	选择与GeoJSON几何体相交的几何体，2dsphere索引支持|
|$geoWithin|	选择边界GeoJSON几何内的几何，2dsphere和2D索引支持|
|$near|	返回靠近点的地理空间对象。需要一个地理空间索引，2dsphere和2d索引支持|
|$nearSphere|	返回球体上某个点附近的地理空间物体。需要一个地理空间索引，dsphere和2d索引支持|

其中 **$geoNear**	包含 $match，$sort和$limit参数。输出文件包含一个额外的距离字段，并可包含位置标识符字段。

下表列出了每个地理空间操作使用的地理空间查询运算符：

| 方法	|坐标 | 说明 |
|:---|:---|:---|
|$near（GeoJSON质心点在这一行和下面一行，2dsphere）|	球形|	另请参阅$nearSphere运算符，它在与GeoJSON和2dsphere索引一起使用时提供相同的功能。|
|$near（传统坐标，2d）|	平面	 
|$nearSphere（GeoJSON，2dsphere）| 球形 |提供与$near使用GeoJSON和2dsphere相同的功能。对于球形查询，最好使用 $nearSphere明确指定名称中的球形查询而不是$near|
|$nearSphere（传统坐标，2d）|	球形|	改为使用GeoJSON点。|
|$geoWithin：{ $geometry：...}|	球形	 ||
|$geoWithin：{ $box：...}|	平面	 ||
|$geoWithin：{ $polygon：...}|	平面	 ||
|$geoWithin：{ $center：...}|	平面	 ||
|$geoWithin：{ $centerSphere：...}|	球形	 ||
|$geoIntersects|	球形	 ||
|$geoNear（2dsphere）|	球形	 ||
|$geoNear（2d）|	平面	 ||

## 示例

### GeoJSON对象数据

创建集合-插入数据-建立索引

```sql
#创建集合
db.createCollection("places")
#插入数据
db.places.insert({
    name: "Central Park",
   location: { type: "Point", coordinates: [ -73.97, 40.77 ] },
   category: "Parks"
} );
db.places.insert({
   name: "Sara D. Roosevelt Park",
   location: { type: "Point", coordinates: [ -73.9928, 40.7193 ] },
   category: "Parks"
} );
db.places.insert({
   name: "Polo Grounds",
   location: { type: "Point", coordinates: [ -73.9375, 40.8303 ] },
   category: "Stadiums"
} );
#在location字段上创建索引
db.places.createIndex( { location: "2dsphere" } )
```

以下查询使用$near操作返回距离指定GeoJSON至少1000米且最远5000米的数据，并按从最近到最远的顺序排序：

```sql
db.places.find(
   {
     location:
       { $near:
          {
            $geometry: { type: "Point",  coordinates: [ -73.9667, 40.78 ] },
            $minDistance: 1000,
            $maxDistance: 5000
          }
       }
   }
)
```

以下查询使用$geoNear命令查询并过滤 { category: "Parks" } 相匹配的数据，按照距离指定的GeoJSON最近的顺序排序

```sql
db.runCommand(
   {
     geoNear: "places",
     near: { type: "Point", coordinates: [ -73.9667, 40.78 ] },
     spherical: true,
     query: { category: "Parks" }
   }
)
```

### 传统坐标对数据

创建集合-插入数据-建立索引

```sql
db.createCollection("location2")
db.location2.save( {_id: "A", position: [0.1, -0.1]} )
db.location2.save( {_id: "B", position: [1.0, 1.0]} )
db.location2.save( {_id: "C", position: [0.5, 0.5]} )
db.location2.save( {_id: "D", position: [-0.5, -0.5]} )
db.location2.ensureIndex( {position: "2d"} )
```

查询point(0,0),半径0.7附近的点

```sql
db.location2.find( {position: { $near: [0,0], $maxDistance: 0.7  } } )
```

查询[0.25, 0.25], [1.0,1.0]区域附近的点

```sql
db.location2.find( {position: { $within: { $box: [ [0.25, 0.25], [1.0,1.0] ] } } } )
```

> 参考 [mongodb官方文档](https://docs.mongodb.com/manual/geospatial-queries)

到此mongodb地理数据支持的基础知识已经介绍完，请看下篇实战篇，我们还是会生成600w调数据mongodb方案进行测试。
