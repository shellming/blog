---
title: Guava Retrying
date: 2018-12-13
categories: [development]
tags: [java,retry,重试]
language: zh
---

#### 简介
Guava Retrying 是一个灵活的 Java 重试工具，该工具基于Guava 的 predicate 机制， 提供了处理各种停止、重试、异常处理的能力。相较于 Spring Retry 只能根据抛出的异常决定是否重试，Guava Retrying 可以根据返回结果来决定是否重试，功能更加强大。[项目 Github 地址](https://github.com/rholder/guava-retrying)


#### 基本使用
官网的基本用例
``` java
Callable<Boolean> callable = new Callable<Boolean>() {

public Boolean call() throws Exception {
return true; // do something useful here
}
};

Retryer<Boolean> retryer = RetryerBuilder.<Boolean>newBuilder()
.retryIfResult(Predicates.<Boolean>isNull())
.retryIfExceptionOfType(IOException.class)
.retryIfRuntimeException()
.withStopStrategy(StopStrategies.stopAfterAttempt(3))
.build();
try {
retryer.call(callable);
} catch (RetryException e) {
e.printStackTrace();
} catch (ExecutionException e) {
e.printStackTrace();
}

```
- 重试条件：结果返回 null 或者抛出 `IOException` 或者抛出 `RuntimeException`
- 停止条件：尝试三次，失败后停止
- `RetryException`：尝试失败后抛出，包含了失败信息
- `ExecutionException`：执行过程中抛出了预料之外的异常，终止了执行

#### Fibonacci Backoff
一直重试，永不停止，每次重试失败后的间隔时间为一个斐波那契数列，最大间隔时间为2分钟
``` java
Retryer<Boolean> retryer = RetryerBuilder.<Boolean>newBuilder()
        .retryIfExceptionOfType(IOException.class)
        .retryIfRuntimeException()
        .withWaitStrategy(WaitStrategies.fibonacciWait(100, 2, TimeUnit.MINUTES))
        .withStopStrategy(StopStrategies.neverStop())
        .build();

```

#### 遇到的问题
如果最终尝试失败，如何获取最后一次返回的结果，或者最后一次抛出的异常
#### 解决方案
翻一下 `RetryException` 的源码，最近一次重试尝试由 `Attempt<?> lastFailedAttempt` 这个字段保存， `Attempt` 有两个实现， `ExceptionAttempt` 和 `ResultAttempt`，调用 `Attempt.get()`，前者抛出 `ExecutionException` 异常，后者返回结果，`ExecutionException` 包装了我们真正的异常，所以最终代码如下：

``` java
private Object getLastResult(RetryException exception) throws Throwable {
Attempt attempt = exception.getLastFailedAttempt();
try {
return attempt.get();
} catch (ExecutionException e) {
throw e.getCause();
}
}
```
