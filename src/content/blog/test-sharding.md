---
author: Vince Rose
pubDatetime: 2025-01-21T16:00:00.000Z
title: Bazel Test Sharding
slug: bazel-test-sharding
featured: true
ogImage: ../../assets/images/test-sharding/og.png
tags:
  - bazel
description: How to use test sharding to speed up your Bazel builds.
---

Bazel excels at speeding up build times. But what happens when a single task takes a long time to run? When that task is not cached, it can become a bottleneck for the build.

At $dayjob, I converted a large Java project to Bazel. The project had some integration test classes that had 25+ tests and would take 45+ minutes to fully run. Still being fairly new to Bazel, the only way I knew to speed this up was to break the test classes down to multiple smaller classes. This worked, but it was quite a bit of manual effort.

I hadn't known at the time about a feature in Bazel called **test sharding**. Test sharding splits up the test **cases** for a single test **target** into multiple shards. Each shard can then be run in parallel, reducing the overall test time.

Let's set up an example test that takes 5 minutes to run. In this example, we have 5 tests cases that each take 1 minute to run.

```java
package com.example;

import org.junit.jupiter.api.Test;

public class ShardTest {

  @Test
  public void test1() throws InterruptedException {
    Thread.sleep(60 * 1000);
  }

  @Test
  public void test2() throws InterruptedException {
    Thread.sleep(60 * 1000);
  }

  @Test
  public void test3() throws InterruptedException {
    Thread.sleep(60 * 1000);
  }

  @Test
  public void test4() throws InterruptedException {
    Thread.sleep(60 * 1000);
  }

  @Test
  public void test5() throws InterruptedException {
    Thread.sleep(60 * 1000);
  }
}
```

```python
java_junit5_test(
    name = "ShardTest",
    srcs = ["src/test/java/com/example/ShardTest.java"],
    test_class = "com.example.ShardTest",
    deps = [
        ":lib",
        "@maven//:org_junit_jupiter_junit_jupiter_api",
        "@maven//:org_junit_jupiter_junit_jupiter_engine",
        "@maven//:org_junit_platform_junit_platform_launcher",
        "@maven//:org_junit_platform_junit_platform_reporting",
    ],
)
```

```shell
➜  bazel test //test-shards-java:ShardTest
...
//test-shards:ShardTest                                                 TIMEOUT in 300.1s

Executed 1 out of 1 test: 1 fails locally.
```

Now, let's split this test into five shards. We can do this by adding the `shard_count` attribute to the `java_test` rule.
The test cases within the target will be split into the number of shards specified. In this case, I split the test into 5 shards, since there are 5 test cases and each of them takes the same amount of time to run.

```python
java_junit5_test(
    name = "ShardTest",
    srcs = ["src/test/java/com/example/ShardTest.java"],
    shard_count = 5,
    test_class = "com.example.ShardTest",
    deps = [
        "@maven//:org_junit_jupiter_junit_jupiter_api",
        "@maven//:org_junit_jupiter_junit_jupiter_engine",
        "@maven//:org_junit_platform_junit_platform_launcher",
        "@maven//:org_junit_platform_junit_platform_reporting",
    ],
)
```

```shell
➜  bazel test //test-shards-java:ShardTest
...
//test-shards-java:ShardTest                                             PASSED in 60.9s
  Stats over 5 runs: max = 60.9s, min = 60.9s, avg = 60.9s, dev = 0.0s
```

Our test now runs in 1 minute instead of 5 minutes. This is a very simplistic example, but it shows how test sharding can reduce the time it takes to run tests by increasing the parallelism.

In a real world situation there some things to consider:
* The number of shards should be equal to or less than the number of test cases
* There is overhead in running a `java_test` target
* Test cases are likely to have variance in run time, so you may not see a linear reduction in time as you increase the number of shards
* Keep an eye on the `min` time in the test results. If there are shards with a significantly lower time than the others, you may have too many shards

Because of all of these factors, there can be a bit of trial and error to find the optimal number of shards for your tests. In situations where there are a lot of test cases, I like to take a sort of binary search approach to find the optimal number.

![Test Sharding](../../assets/images/TestShard-1.gif)

**Note**: It is up to test runners to integrate with Bazel's sharding feature. In this case, the runner [implemented in rules_jvm](https://github.com/bazel-contrib/rules_jvm/blob/main/java/src/com/github/bazel_contrib/contrib_rules_jvm/junit5/TestSharding.java) is already set up to handle sharding.

The full code example can be found on my [GitHub](https://github.com/vinnybod/bazel-examples/test-sharding).
