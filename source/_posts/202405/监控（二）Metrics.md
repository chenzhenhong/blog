---
title: 监控（二）Metrics
date: 2024-05-03 14:26:23
category:
- 实践
tags: 
- 监控
- Spring Acutator
- Micrometer
- metrics
---

> [Spring Actuator Metrics官方文档](https://docs.spring.io/spring-boot/reference/actuator/metrics.html#actuator.metrics.registering-custom)

> [Micrometer官方文档](https://docs.micrometer.io/micrometer/reference/overview.html)

Spring Boot Actuator提供Micrometer的依赖管理和自动配置，用于指标的收集；市面上有许多监控后端，如Prometheus、influx等等，Micrometer提供统一的指标记录门面，用于对接不同的监控后端。

我们先从一个仅包含Micrometer的新项目开始，后续再学习其与Spring Boot Actuator的搭配使用。


## 一、Micrometer
### 1.安装
只需要引入micrometer-core

推荐导入bom，再添加依赖，因为后续对接其他监控后端时，可以做统一的版本管理。
```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>io.micrometer</groupId>
            <artifactId>micrometer-bom</artifactId>
            <version>1.15.1</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-core</artifactId>
    </dependency>
</dependencies>
```

### 2.概念
#### （1）监控后端
- 维度

比如记录/login接口的HTTP请求成功和失败的数量，支持维度的监控后端，将url和请求结果作为标签：

http_request_count{url="/login",result=success} 10

http_request_count{url="/login",result=fail} 1

不支持维度的监控后端，只能将指标名称和标签扁平化处理：

http_request_count_login_success: 10

http_request_count_login_fail: 1

- 速率聚合(Rate Aggregation)

什么是速率聚合，还是记录/login接口的HTTP请求成功和失败的数量，有俩台服务，A服务已经运行了10天，B服务运行1天；A服务失败数量100，B服务失败数量10，单从这俩个数量我们无法断定A服务是否问题，因为他们的时间不同，没有对比性。

因此我们一般将计数与时间对比，通过比率可以确定服务A、B都是每天10个失败数量。这一步比率计算有些监控后端会自己计算，有些需要Micrometer帮忙计算。

- 发布方式

有些监控后端等待客户端推送，有些则是自己定时拉取。

#### （2）Registry
`MeterRegistry`记录一个监控后端接口，每种监控后端都有对应的实现，因为还没有降到Prometheus, 所以下文使用`SimpleMeterRegistry`演示。

#### （3）Naming Meters
不同监控后端支持不同的指标名称，有的是下划线分割，有的是点分割，Micrometer允许你自行定义。
```java
registry.config().namingConvention(myCustomNamingConvention);
```

#### （4）Meter Filters
- 拒绝或接受指标注册
```java
new MeterFilter() {
    @Override
    public MeterFilterReply accept(Meter.Id id) {
       if(id.getName().contains("test")) {
          return MeterFilterReply.DENY;
       }
       return MeterFilterReply.NEUTRAL;
    }
}
```

- 转换指标
```java
new MeterFilter() {
    @Override
    public Meter.Id map(Meter.Id id) {
       if(id.getName().startsWith("test")) {
          return id.withName("extra." + id.getName()).withTag("extra.tag", "value");
       }
       return id;
    }
}
```

- 配置分布统计
```java
new MeterFilter() {
    @Override
    public DistributionStatisticConfig configure(Meter.Id id, DistributionStatisticConfig config) {
        if (id.getName().startsWith("prefix")) {
            return DistributionStatisticConfig.builder()
                    .percentiles(0.9, 0.95)
                    .build()
                    .merge(config);
        }
        return config;
    }
};
```

### 3.指标类型
#### （1）Counter
counter是一个递增的计数器，值必须是整数，适合用来统计HTTP请求数量、JVM GC次数等。

#### （2）Gauges
gauge是一个可增可减的"计数器"，把它想象成我们汽车的油表，根据油箱的油量可高可低；类似的gauge一般持有一个集合对象的弱引用，然后统计其集合大小变化。或者持有`ThreadPoolExecutor`对象，获取线程数量、队列人物数量等。

#### （3）Timers
timer记录俩个维度，任务总数和任务总用时，比如有俩个HTTP请求，一个用时10s返回响应，一个用时5s返回响应，那么timer会记录总数为2，总用时为15s。

timer额外维护时间窗口内最大值和直方图统计。

#### （4）Distribution Summaries
distribution summarie与timer类似，不过将任务总时间替换为自定义维度。比如总计上传文件的大小，总共接收到俩个请求，一个上传了5M，一个上传了1M，则记录总数为2，总文件大小为6M。

同理也额外维护时间窗口内最大值和直方图统计。

#### （5）Long Task Timers
timer是在任务结束后统计任务的用时，而long task timer是在任务进行时统计，会记录如下指标：
- 活动任务计数
- 活动任务的总持续时间
- 活动任务的最长持续时间
- 直方图统计

### 4.代码演示
先新建一个`LoggingMeterRegistry`，配置5s打印一次、配置指标为空也打印（logInactive）。
```java
LoggingRegistryConfig loggingRegistryConfig = new LoggingRegistryConfig() {
    @Override
    public String get(String key) {
        return null;
    }

    @Override
    public Duration step() {
        return getDuration(this, "step").orElse(Duration.ofSeconds(5));
    }

    @Override
    public boolean logInactive() {
        return true;
    }
};
LoggingMeterRegistry loggingMeterRegistry = new LoggingMeterRegistry(loggingRegistryConfig, Clock.SYSTEM);
Metrics.addRegistry(loggingMeterRegistry);

// 统计通用tags
loggingMeterRegistry.config().commonTags("application", "micrometer-dem
```
#### （1） counter
```java
Counter myCounter = Metrics.counter("my.counter");
myCounter.increment();
myCounter.increment();
```
输出如下：

```
22:38:32.455 my.counter{application=micrometer-demo} delta_count=2 throughput=0.4/s
22:38:37.453 my.counter{application=micrometer-demo} delta_count=0 throughput=0/s
```
可以到我们设置的指标名称`my.counter`，标签application=micrometer-demo都如实打印了，
delta_count表示计数的次数，throughput是比率 2 / 5s = 0.4/s。

可能有人好奇为什么22:38:37.453输出的delta_count变成了0，其实这就是前面说的速率聚合(Rate Aggregation)是在Micrometer处理，还是监控后端处理。

这里便是在Micrometer处理的，我们定义`myCounter`其实是`StepCounter`类型，`StepCounter`的计数字段是`StepValue`类型，逻辑就藏在这里面。

```java
// StepValue的主要字段

private final long stepMillis;  // 时间间隔，比如在我们的示例代码为5s

private final AtomicLong lastInitPos;  // 上次处理的时间边界, 初始值为当前时间戳 / stepMillis

private volatile V previous;  // 上次记录的值，初始为0

private V current; // 当前值

private void rollCount(long now) {
    final long stepTime = now / stepMillis;
    final long lastInit = lastInitPos.get();
    if (lastInit < stepTime && lastInitPos.compareAndSet(lastInit, stepTime)) {
        final V v = valueSupplier().get();
        // Need to check if there was any activity during the previous step interval.
        // If there was then the init position will move forward by 1, otherwise it
        // will be older.
        // No activity means the previous interval should be set to the `init` value.
        previous = (lastInit == stepTime - 1) ? v : noValue();
    }
}
```
`StepValue`使用`previous` + `current`计数，每次新增写入`current`，每次查询先检查是否到达时间边界，是的话将`current`赋值给`previous`，清空`current`然后返回`previous`。

![StepCounter](/images/202405/counter.png)

#### （2）gauge
```java
List<String> list = new ArrayList<>();
Metrics.gauge("my.gauge", list, List::size);
list.add("A");
list.add("B");
list.add("C");
TimeUnit.SECONDS.sleep(7);
list.remove("A");
TimeUnit.SECONDS.sleep(12);
list = null;
System.gc();
```
输出如下：
```
23:58:42.526 my.gauge{application=micrometer-demo} value=3
23:58:47.524 my.gauge{application=micrometer-demo} value=2
23:58:52.524 my.gauge{application=micrometer-demo} value=2
23:58:57.524 my.gauge{application=micrometer-demo} value=NaN
```
gauge通常用法就是持有一个对象的弱引用，然后根据提供的方法从该对象获取统计的数值。当前gauge的实现为`DefaultGauge`，源码比较简单，如下：
```java
public class DefaultGauge<T> extends AbstractMeter implements Gauge {

    private static final WarnThenDebugLogger logger = new WarnThenDebugLogger(DefaultGauge.class);

    private final WeakReference<T> ref;

    private final ToDoubleFunction<T> value;

    public DefaultGauge(Meter.Id id, @Nullable T obj, ToDoubleFunction<T> value) {
        super(id);
        this.ref = new WeakReference<>(obj);
        this.value = value;
    }

    @Override
    public double value() {
        T obj = ref.get();
        if (obj != null) {
            try {
                return value.applyAsDouble(obj);
            }
            catch (Throwable ex) {
                logger.log(() -> "Failed to apply the value function for the gauge '" + getId().getName() + "'.", ex);
            }
        }
        return Double.NaN;
    }

}
```

#### （3）timer
```java
Timer timer = Metrics.timer("my.timer");
Runnable runnable1 = timer.wrap(() -> {
    try {
        TimeUnit.SECONDS.sleep(3);
    } catch (InterruptedException e) {
        throw new RuntimeException(e);
    }
});
new Thread(runnable1).start();

Runnable runnable2 = timer.wrap(() -> {
    try {
        TimeUnit.SECONDS.sleep(8);
    } catch (InterruptedException e) {
        throw new RuntimeException(e);
    }
});
new Thread(runnable2).start();
```
输出如下：
```
00:45:07.685 my.timer{application=micrometer-demo} delta_count=1 throughput=0.2/s mean=3.000478867s max=3.000478867s
00:45:12.681 my.timer{application=micrometer-demo} delta_count=1 throughput=0.2/s mean=8.000086505s max=8.000086505s
00:45:17.681 my.timer{application=micrometer-demo} delta_count=0 throughput=0/s mean= max=8.000086505s
00:45:22.681 my.timer{application=micrometer-demo} delta_count=0 throughput=0/s mean= max=
00:45:27.681 my.timer{application=micrometer-demo} delta_count=0 throughput=0/s mean= max=
00:45:32.681 my.timer{application=micrometer-demo} delta_count=0 throughput=0/s mean= max=
```
timer输出包括计数delta_count、比率throughput、平均用时mean、最大用时max；可以看到不同时间间隔内delta_count又清空了，没错这里timer的实现是`StepTimer`。

```java
// StepTimer

private final LongAdder count = new LongAdder();  // 计数

private final LongAdder total = new LongAdder();  // 总用时

private final StepTuple2<Long, Long> countTotal;

private final TimeWindowMax max;

@Override
protected void recordNonNegative(final long amount, final TimeUnit unit) {
    final long nanoAmount = unit.toNanos(amount);
    count.add(1L);
    total.add(nanoAmount);
    max.record((double) nanoAmount);
}

@Override
public double totalTime(final TimeUnit unit) {
    return TimeUtils.nanosToUnit(countTotal.poll2(), unit);
}

@Override
public double max(final TimeUnit unit) {
    return TimeUtils.nanosToUnit(max.poll(), unit);
}
```
可以看到`StepTimer`的主要逻辑是在`StepTuple2`和`TimeWindowMax`。

- `StepTuple2`就相当于俩个`StepValue`，`StepValue`每次跨过时间间隔处理一个值，`StepTuple2`则要同时处理俩个值。

- `TimeWindowMax`的设计下，记录、取值时都会判断是否跨过了时间间隔，是的话当前值置为0，索引往前。


  ![TimeWindowMax](/images/202405/TimeWindowMax.png)

  得益于每次记录的时候更新全部值，所以图中刚进入第二时间间隔的时候获取值，依然可以返回最大值25，不会说突然从25降到0。


#### （4）summary
summary和timer类似，不再展开

#### （5）longTaskTimer
```java
LongTaskTimer longTaskTimer = Metrics.more().longTaskTimer("my.long.task.timer");
new Thread(() -> longTaskTimer.record(() -> {
    try {
        TimeUnit.SECONDS.sleep(7);
    } catch (InterruptedException e) {
        throw new RuntimeException(e);
    }
})).start();

new Thread(() -> longTaskTimer.record(() -> {
    try {
        TimeUnit.SECONDS.sleep(100);
    } catch (InterruptedException e) {
        throw new RuntimeException(e);
    }
})).start();
```
输出如下
```
01:07:11.933 my.long.task.timer{application=micrometer-demo} active=2 duration=9.137689851s mean=4.568601899s max=4.56865963s
01:07:16.932 my.long.task.timer{application=micrometer-demo} active=1 duration=9.567361698s mean=9.567285548s max=9.567300976s
01:07:21.932 my.long.task.timer{application=micrometer-demo} active=1 duration=14.567513576s mean=14.567453406s max=14.567465618s
01:07:26.932 my.long.task.timer{application=micrometer-demo} active=1 duration=19.567244574s mean=19.567186087s max=19.567197808s
01:07:31.932 my.long.task.timer{application=micrometer-demo} active=1 duration=24.56748797s mean=24.567429272s max=24.567441014s
```
实现类是`DefaultLongTaskTimer`，将类中有一个`NavigableSet<SampleImpl> activeTasks`变量用于存在存活的任务。


### 5.MeterBinder
`MeterBinder`用于一组关联指标的注册，Micrometer内置了许多指标，如`JvmMemoryMetrics`、`JvmGcMetrics`等等。
```java
JvmGcMetrics jvmGcMetrics = new JvmGcMetrics();
jvmGcMetrics.bindTo(loggingMeterRegistry);
```

输出如下：
```
jvm.gc.memory.allocated{application=micrometer-demo} delta_count=56 MiB throughput=11.2 MiB/s
jvm.gc.memory.promoted{application=micrometer-demo} delta_count=941.5 KiB throughput=188.3 KiB/s
jvm.gc.live.data.size{application=micrometer-demo} value=0 B
jvm.gc.max.data.size{application=micrometer-demo} value=7.652344 GiB
jvm.gc.pause{action=end of minor GC,application=micrometer-demo,cause=G1 Evacuation Pause,gc=G1 Young Generation} delta_count=1 throughput=0.2/s mean=0.005s max=0.005s
```