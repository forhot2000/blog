---
categories: java spring
author: forhot2000@qq.com
date: 2024/04/10
---

# Spring AOP 拦截元注解

## 什么是元注解（meta annotation）

元注解是可以注解到注解上的注解，它的作用和目的就是给其他普通的标签进行解释说明的。

Java 默认的元注解包含：

- @Retention
- @Target
- @Document
- @Inherit
- @Repeatable

## 什么是组合注解（composed annotation）

我们定义一个注解 @OperationLog，当类或者方法上添加了 @OperationLog 注解时，我们将通过 Aspect 打印方法调用的日志。

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface OperationLog {

}
```

假设，我们经常需要给 Controller 的 handler 方法同时加 @RequestMapping 和 @OperationLog 注解。

那么，我们可以定义一个新注解 @Operation，在 @Operation 上添加 @RequestMapping 和 @OperationLog 作为的元注解，这样就可以只添加一个 @Operation 注解代替原来的两个注解了，这种方式就叫组合注解。

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@RequestMapping
@OperationLog
public @interface Operation {

    @AliasFor(annotation = RequestMapping.class)
    String[] value() default {};

    @AliasFor(annotation = RequestMapping.class)
    String[] path() default {};
}
```

## AOP 如何拦截指定注解

假设，我们有一个 Controller 类如下

```java
@RestController
public class TestController {

    @RequestMapping("/test")
    @OperationLog
    public void test() {
        System.out.println("on test");
    }
}
```

定义一个 OperationLogAspect 类用于拦截带有 @OperationLog 的类和方法

```java
@Component
@Aspect
public class OperationLogAspect {

    @Pointcut("@within(OperationLog) || @annotation(OperationLog)")
    public void log() {

    }

    @Before("log()")
    public void beforeLog(JoinPoint joinPoint) {
        System.out.println("execute operation: " + joinPoint.getSignature());
    }
}
```

当我们访问 /test 的时候，就会被拦截并进入 beforeLog 方法，打印 operation 日志如下：

```log
execute operation: void cn.forhot2000.TestController.test()
on test
```

## AOP 如何拦截注解中包含指定元注解

那么，如果我们使用组合注解 @Operation 会这么样呢？

修改 TestController 的注解

```java
@RestController
public class TestController {

    @Operation("/test")
    public String test() {
        return "test";
    }
}
```

现在，我们访问 /test 的时候，发现并没有拦截并进入 beforeLog 方法。

因为 @within 和 @annotation 都是按注解类的类名去匹配的，并不能主动地去匹配注解的元注解。

所幸还有替代方案，使用 `within(@(@OperationLog *) *)` 来匹配类注解的元注解，使用 `execution(@(OperationLog *) * *(..))` 来匹配方法注解的元注解，修改下 OperationLogAspect 类：

```java
@Component
@Aspect
public class OperationLogAspect {

    @Pointcut("within(@OperationLog *) || " +
            "within(@(OperationLog *) *) || " +
            "execution(@OperationLog * *(..)) || " +
            "execution(@(OperationLog *) * *(..))")
    public void log() {

    }

    @Before("log()")
    public void beforeLog(JoinPoint joinPoint) {
        System.out.println("execute operation: " + joinPoint.getSignature());
    }
}
```

上面的代码中 Pointcut 表达式说明：

- `within(@OperationLog *)` 匹配带有 @OperationLog 注解的类，相当于 @within(OperationLog)
- `within(@(OperationLog *) *)` 匹配注解中带有 @OperationLog 元注解的类
- `execution(@OperationLog * *(..))` 匹配带有 @OperationLog 注解的方法，相当于 @annotation(OperationLog)
- `execution(@(OperationLog *) * *(..))` 匹配注解中带有 @OperationLog 元注解的方法

现在，我们再访问 /test 的时候，发现又可以拦截进入 beforeLog 方法，正常打印出日志了。
