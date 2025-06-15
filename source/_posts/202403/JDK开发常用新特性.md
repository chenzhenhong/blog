---
title: JDK常用新特性
date: 2024-03-17 20:00:19
category:
- 学习
tags: 
- JDK
---
```JAVA
package com.example.demo.feature;

import java.io.IOException;
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.util.*;

public class Main {
    public static void main(String[] args) {
        // ========================================= JDK9 =========================================
        // 新加了一些快速创建的工具方法，返回值都是不可变的集合
        List<Integer> list = List.of(1, 2, 3);
        Set<Integer> set = Set.of(1, 2, 3, 4);
        Map<String, String> map = Map.of("key1", "hello", "key2", "world");

        // Optional新加的方法
        String name = null;
        Optional.ofNullable(name)
                .ifPresentOrElse(System.out::println, () -> System.out.println("不存在呀!"));
        Optional.ofNullable(name)
                .or(() -> Optional.of("备选")) // 返回值还是一个Optional
                .ifPresent(System.out::println);
        // ========================================= JDK9 =========================================

        // ========================================= JDK10 =========================================
        var studentName = "小陈";
        System.out.println(studentName.getClass());

        var varList = new ArrayList<>();
        varList.add(1);
        varList.add("1");
        var item1 = varList.get(0);
        var item2 = varList.get(1);
        System.out.println(item1.getClass());
        System.out.println(item2.getClass());
        System.out.println(item1 instanceof Integer); // true
        System.out.println(item2 instanceof String);  // true
        // int intItem1 = item1; // 但是不能这样转换, 因为实际返回的是Object
        // ========================================= JDK10 =========================================

        // ========================================= JDK11 =========================================
        // String加了新方法
        boolean blank = "hello world".isBlank();
        long count = "hello\nworld".lines().count();
        String repeat = "h".repeat(10);
        System.out.println(blank); // false
        System.out.println(count); // 2
        System.out.println(repeat); // hhhhhhhhhh

        // HttpClient
        HttpClient httpClient = HttpClient.newHttpClient();
        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create("https://www.baidu.com"))
                .build();
        try {
            HttpResponse<String> response = httpClient.send(request, HttpResponse.BodyHandlers.ofString());
            System.out.println(response.statusCode());
            System.out.println(response.body());
        } catch (IOException e) {
        } catch (InterruptedException e) {
        }
        // ========================================= JDK11 =========================================

        // ========================================= JDK14 =========================================
        int status = 1;
        String result = switch (status) { // -> 默认带break
            case 1 -> "one";
            case 2, 3, 4 -> "two";
            case 5 -> {
                System.out.println("执行5");
                yield "five";  // 用yield关键字返回
            }
            default -> throw new IllegalStateException("Unexpected value: " + status);
        };
        // ========================================= JDK14 =========================================

        // ========================================= JDK15 =========================================
        // 文本块
        String json = """
                {
                      "name": "Alice",
                      "age": 30
                }
                """;
        // ========================================= JDK15 =========================================

        // ========================================= JDK16 =========================================
        // Record - 能在方法内部定义
        record User(String name, int age) {
            public void print() {
                System.out.println(name + " " + age);
            }
        }
        User user = new User("小明", 10);
        System.out.println(user.name());
        user.print();
        // user.name = "换个名字"; // 不可以, name是用final修饰的
        // ========================================= JDK16 =========================================

        // ========================================= JDK17 =========================================
        Object obj = "Hello";
        if (obj instanceof String s) { // 自动完成类型转换
            System.out.println(s.toLowerCase());
        }
        String switchResult = switch (obj) {
            case Integer i -> String.format("int %d", i);
            case Long l    -> String.format("long %d", l);
            case Double d  -> String.format("double %f", d);
            case String s  -> String.format("String %s", s);
            default        -> obj.toString();
        };
        System.out.println(switchResult);

        // ========================================= JDK17 =========================================

        // ========================================= JDK21 =========================================
        Thread thread = Thread.startVirtualThread(() -> {
            System.out.println("虚拟线程");
        });
        try {
            thread.join();
        } catch (InterruptedException e) {}
        // ========================================= JDK21 =========================================
    }
}
```
