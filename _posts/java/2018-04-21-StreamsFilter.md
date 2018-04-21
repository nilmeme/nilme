---
layout: single
title: Streams filter
date: 2018-03-06 15:03:02
tags: ['java']
category: java
toc: true
---

### 前言

Lambda是Java 8的主要主题，这是一个非常酷的，期待已久的Java平台的补充。 但是，如果我们没有任何方法将lambda应用于收藏，单靠lambda就没有价值。 将接口迁移到能够在集合中使用lambdas的问题可以通过默认方法来解决，这些方法也被称为defender方法。 在这篇博文中，我们将深入研究Java集合的批量数据操作，将向你展示一些java 8个例子来演示的流filter()，collect()使用，findany()和orelse()。 如果您对Java Streams使用的最佳实践感兴趣，之前还有一篇关于如何使用Java Streams 的最佳实践。 除此之外，请查看[这篇文章](http://blog.nilme.me/java/2018/03/09/java/Java8-Streams.html)，尤其是如果您希望梳理一般关于Java最佳实践的知识。

<!--more-->

### Streams filter() and collect()

java8 之前我们过滤一个List可能如下所示

```java
public class BeforeJava8 {
    public static void main(String[] args) {
        List<String> skills = Arrays.asList("spring", "node", "python");
        List<String> result = getFilterOutput(skills, "python");
        for (String item : result) {
            System.out.println(item);    //output : spring, node
        }
    }
    private static List<String> getFilterOutput(List<String> skills, String filter) {
        List<String> result = new ArrayList<>();
        for (String skill : skills {
            if (!filter.equals(skill)) { // 过滤 python
                result.add(skill);
            }
        }
        return result;
    }
}
```

java8之后，我们使用stream.filter()去过滤一个List,再用collect()去转化一个List

```java
public class NowJava8 {
    public static void main(String[] args) {
        List<String> skills = Arrays.asList("spring", "node", "python");
        List<String> result = skills.stream()               // convert list to stream
                .filter(skill -> !"python".equals(skill))   // we dont like mkyong
                .collect(Collectors.toList());              // collect the output and convert streams to a List
        result.forEach(System.out::println);                //output : spring, node
    }
}
```

### Streams filter(), findAny() 和 orElse()

```java
public class Person {
    private String name;
    private int age;
    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }
    //gettersm setters, toString
}
```

java8之前我们的在列表中查找一个元素

```java
public class BeforeJava8 {
    public static void main(String[] args) {
        List<Person> persons = Arrays.asList(
                new Person("mkyong", 30),
                new Person("jack", 20),
                new Person("lawrence", 40)
        );
        Person result = getStudentByName(persons, "jack");
        System.out.println(result);
    }
    private static Person getStudentByName(List<Person> persons, String name) {
        Person result = null;
        for (Person temp : persons) {
            if (name.equals(temp.getName())) {
                result = temp;
            }
        }
        return result;
    }
}
```

java8之后

```java
public class NowJava8 {
    public static void main(String[] args) {
        List<Person> persons = Arrays.asList(
                new Person("mkyong", 30),
                new Person("jack", 20),
                new Person("lawrence", 40)
        );
        Person result1 = persons.stream()                        // Convert to steam
                .filter(x -> "jack".equals(x.getName()))        // we want "jack" only
                .findAny()                                      // If 'findAny' then return found
                .orElse(null);                                  // If not found, return null
        System.out.println(result1);
        Person result2 = persons.stream()
                .filter(x -> "ahmook".equals(x.getName()))
                .findAny()
                .orElse(null);
        System.out.println(result2);
    }
}
```

如果是多条件查找的话，可以是这样

```java
public class NowJava8 {
    public static void main(String[] args) {
        List<Person> persons = Arrays.asList(
                new Person("mkyong", 30),
                new Person("jack", 20),
                new Person("lawrence", 40)
        );
        Person result1 = persons.stream()
                .filter((p) -> "jack".equals(p.getName()) && 20 == p.getAge())
                .findAny()
                .orElse(null);
        System.out.println("result 1 :" + result1);
        //or like this
        Person result2 = persons.stream()
                .filter(p -> {
                    if ("jack".equals(p.getName()) && 20 == p.getAge()) {
                        return true;
                    }
                    return false;
                }).findAny()
                .orElse(null);
        System.out.println("result 2 :" + result2);
    }
}
```

### Streams filter() and map()

```java
public class NowJava8 {
    public static void main(String[] args) {
        List<Person> persons = Arrays.asList(
                new Person("mkyong", 30),
                new Person("jack", 20),
                new Person("lawrence", 40)
        );
        String name = persons.stream()
                .filter(x -> "jack".equals(x.getName()))
                .map(Person::getName)                        //convert stream to String
                .findAny()
                .orElse("");
        System.out.println("name : " + name);
        List<String> collect = persons.stream()
                .map(Person::getName)
                .collect(Collectors.toList());
        collect.forEach(System.out::println);
    }  
}
```
