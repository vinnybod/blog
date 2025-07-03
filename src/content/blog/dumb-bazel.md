---
author: Vince Rose
pubDatetime: 2025-07-07T16:00:00.000Z
title: "Dumb Bazel"
slug: dumb-bazel
featured: true
ogImage: ../../assets/images/test-sharding/og.png
tags:
  - bazel
description: 
---

"Let's just auto-generate targets," someone says. "It'll save us from writing boilerplate." Fast forward six months and your simple macro now has 35 attributes, handles edge cases you didn't know existed, and requires a PhD to configure properly.

Recently I've been having a lot of conversations with [@Farid Zakaria](https://fzakaria.com/) about *Dumb Bazel*. Which really boils down to how much abstraction is too much abstraction? Does *DRY* (Don't Repeat Yourself) really apply to Bazel declarations?

Let's start with an open source macro and see how the macro grows in complexity over time. Then we can talk about the trade-offs.

We're going to use [java_test_suite](https://github.com/vinnybod/rules_jvm/blob/0bef82e8d7038a6628faad06b9a57d10e536c2c5/java/private/create_jvm_test_suite.bzl#L26-L129) macro which comes from `rules_jvm`.

```py
java_test_suite(
    name = "example-tests",
    srcs = glob(["**/*.java"]),
    resources = glob(["src/test/resources/**"]),
    deps = [
        ...
    ],
)
```

Let's review what this macro does based on the [code](https://github.com/vinnybod/rules_jvm/blob/0bef82e8d7038a6628faad06b9a57d10e536c2c5/java/private/create_jvm_test_suite.bzl#L26-L129).

1. Separate the "Test" classes from the "non-test" classes by naming convention.
2. Create a `java_library` out of the "non-test" classes
3. Create a `java_test` out of all of the "test" classes, passing the `deps` and newly created `java_library` to each of them.

Now see how this macro starts to grow in complexity. We have a test that inherits other classes which are also tests themselves. But our macro sorts the tests and non-tests. Each test is not "aware" of one another.

Let's address this by adding a new attribute `additional_library_srcs`. This will tell the macro "this is a test, but you should also include it in the `java_library` for other tests.

```py
java_test_suite(
    name = "example-tests",
    srcs = glob(["**/*.java"]),
    resources = glob(["src/test/resources/**"]),
    additional_library_srcs = [
        "ExampleBaseTest.java",
    ],
    deps = [
        ...
    ],
)
```

Our developers are complaining that while they like only having to write the one test suite, they need to specify some deviated attributes for certain tests. Like maybe some of these tests need to be tagged for more cpu (`cpu:2`), or they need a different size than the default.

Let's solve for this by providing a `dict` of attributes to apply "per test".

```py
java_test_suite(
    name = "example-tests",
    srcs = glob(["**/*.java"]),
    resources = glob(["src/test/resources/**"]),
    additional_library_srcs = [
        "ExampleBaseTest.java",
    ],
    per_test_args = {
        "ExampleOne.java": {
            "size": "large",
            "tags": "cpu:2",
        },
    },
    deps = [
        ...
    ],
)
```

**Note:** At *$DAYJOB* we have an automation set up that pulls JUnit `@Tag` annotations from the source code and generates the `per_test_args` dict automatically.

Let's add one more modification just to get the point across. The test suite assumes all tests end in `Test.java` but it turns out for "reasons" we need to also generate some tests for classes that end in `XYZ.java`. Okay no problem. Let's add a new attribute.

```py
java_test_suite(
    name = "example-tests",
    srcs = glob(["**/*.java"]),
    resources = glob(["src/test/resources/**"]),
    additional_library_srcs = [
        "ExampleBaseTest.java",
    ],
    per_test_args = {
        "ExampleOne.java": {
            "size": "large",
            "tags": "cpu:2",
        },
    },
    test_suffixes = [
        "Test.java",
        "XYZ.java",
    ],
    deps = [
        ...
    ],
)
```

---

**Pros:**

With the macro approach, developers don't need to add new `java_test` targets every time they add a new test. It should be picked up by the globbed `srcs` in the macro.

Most of the developers that I've worked with don't really care or want to learn a new build system. They want things to "just work". With the "bloated macro" setup, the majority of the time they don't need to touch the Bazel files.

**Cons:**

Build avoidance suffers. We're adding every dependency required in the package to the test suite. If we wrote out each test target individually, we could strip down the dependencies more precisely.

The debugging experience gets worse. When something goes wrong with a generated target, you're now debugging both your code AND the macro that generated it.

## The "Dumb Bazel" Alternative

What if we just... wrote the targets out explicitly?

```py
java_library(
    name = "example-tests-test-lib",
    srcs = glob(["**/*.java"], exclude=["**/*Test.java"]),
    resources = glob(["src/test/resources/**"]),
    deps = [
        ...
    ],
)

java_library(
    name = "example-base-test-lib",
    srcs = ["ExampleBaseTest.java"],
    deps = [
        ...
    ]
)

java_test(
    name = "ExampleBaseTest",
    srcs = ["ExampleBaseTest.java"],
    deps = [
        ":example-tests-test-lib",
        ...
    ]
)

java_test(
    name = "ExampleOneTest",
    srcs = ["ExampleOneTest.java"],
    size = "large",
    tags = ["cpu:2"],
    deps = [
        ":example-tests-test-lib",
        ":example-base-test-lib",
        ...
    ]
)

java_test(
    name = "ExampleTwoTest",
    srcs = ["ExampleTwoTest.java"],
    deps = [
        ":example-tests-test-lib",
        ...
    ]
)
```

Yes, it's more verbose. Yes, there's more work for the developer. But:

- It's explicit about what each test actually needs
- Build avoidance is better because dependencies are precise
- Any developer can read and understand what's happening
- Better tooling support from `unused_deps` and `buildozer` recommendations

## Finding the Balance

I'm not advocating for throwing away all macros. Some abstractions genuinely make sense - especially when you're encoding complex domain knowledge or company-specific patterns that would be error-prone to repeat.

It might surprise you to hear that I just spent this entire blog post advocating for a position that *I* am not fully sold on. In this argument, I still lean towards the macro set up where developers rarely need to touch the `BUILD` files.

**Why?** As an engineer working in Developer Productivity, my customers are the engineers working in the codebases. And what I hear from them is that they don't want to manage Bazel files. I am **more** sold on implementing the *Dumb Bazel* approach when its paired with more automation such as [Gazelle](https://github.com/bazel-contrib/bazel-gazelle).
