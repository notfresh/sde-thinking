# Builder 模式：构造复杂对象的链式写法

> cs-wiki·第 002 篇。一篇笔记，三句话：Builder 把一长串构造器参数拆成命名的链式方法，每个参数有了自己的名字，未传的走默认值——目的是让"构造对象"这件事**读起来自描述**。

## 先把名词固定

| 名字 | 是什么 |
|---|---|
| **构造器（Constructor）** | `new MyClass(a, b, c, d, ...)` 这种一次性塞一堆参数的方式 |
| **可选参数** | 不传也能走的对象属性，靠默认值兜底 |
| **Builder 模式** | 一种设计模式：用一个内部类先累积参数，最后 `build()` 一次性组装出原对象 |
| **链式调用** | `a().b().c()` 这种写法——每个方法返回 `this`，所以能一路点下去 |

一句话关系：**Builder 不改变对象本身，只改"创建对象的过程"**——让它从一长串难懂的构造器参数，变成一段"不用看文档也能猜出每个参数是干嘛的"的自描述代码。

---

## 为什么要 Builder

你可能见过这种代码——构造一个对象要传一长串参数：

```java
// 假设直接用构造器
new MybatisTestEnvironment(
    null, // AnnotationConfigApplicationContext
    null, // HikariDataSource
    "classpath:application.yml", // yaml 路径 1
    "classpath:application-dev.yml", // yaml 路径 2
    GlobalDictItemMapper.class, // mapper 类
    5, // 最大连接数
    1, // 最小空闲连接
    30000, // 连接超时
    "TestHikari", // 连接池名
    "com.shaangu.qms.**.model" // 别名包
);
```

这种写法有两个问题：

1. **看参数猜不出含义**——`5` 是最大连接数还是超时秒数？得回去看构造器签名
2. **可选参数难以处理**——只想改其中一个，其他都得照样传

---

## Builder 的解法

把构造拆成**链式调用**，每个参数有自己的方法名，**未设置的参数走默认值**：

```java
MybatisTestEnvironment env = MybatisTestEnvironment.builder()
    .yamlLocations("classpath:application.yml", "classpath:application-dev.yml")
    .mapperClass(GlobalDictItemMapper.class)
    .typeAliasesPackage("com.shaangu.qms.**.model")
    .maximumPoolSize(5)   // 名字直接告诉你它是干嘛的
    .minimumIdle(1)
    .connectionTimeoutMs(30000)
    .hikariPoolName("TestHikari")
    .build();
```

你看这段代码，**不用看 `MybatisTestEnvironment` 的构造器也能猜出每个参数是干嘛的**。这就是 Builder 的核心价值——**让调用代码可读、自描述**。

---

## 标准 Builder 结构

通常长这样（你可以打开 `MybatisTestEnvironment.java:119-378` 对照看）：

```java
public class MybatisTestEnvironment {
    // 1. 私有构造器：外部不能直接 new，只能走 Builder
    private MybatisTestEnvironment(...) {
        // 把 Builder 累积的参数拷过来
    }

    // 2. 静态工厂方法：拿到 Builder
    public static Builder builder() {
        return new Builder();
    }

    // 3. Builder 内部类：累积参数
    public static class Builder {
        private String hikariPoolName = "MybatisTestHikari"; // 默认值
        private String typeAliasesPackage = "com.shaangu.qms.**.model";
        private List<String> yamlLocations = new ArrayList<>();
        // ... 其他字段，每条对应一个配置项

        public Builder hikariPoolName(String name) {
            this.hikariPoolName = name; // 改默认值
            return this; // 返回自己，支持链式调用
        }

        public Builder typeAliasesPackage(String pkg) {
            this.typeAliasesPackage = pkg;
            return this;
        }

        // 4. build() 把累积的参数组装成最终对象
        public MybatisTestEnvironment build() {
            // 校验、转换、调私有构造器
            return new MybatisTestEnvironment(...);
        }
    }
}
```

---

## Builder 模式的关键特征

1. **私有构造器**：外部代码不能 `new MybatisTestEnvironment(...)`，只能 `builder()`
2. **链式方法返回 `this`**：每个 setter 返回自己，所以能 `.hikariPoolName(...).maximumPoolSize(...).build()` 一路链下去
3. **`build()` 收尾**：链的最后一棒是 `build()`，把累积的参数实例化成最终对象
4. **每个字段一个 setter**：参数命名和业务含义一一对应，调用代码自描述

---

**一句话总结**：Builder 是"用一个流畅的链式调用，替代一长串构造器参数"。它没有改变对象本身，只是让**创建对象的过程**变得清晰、好读、好维护。
