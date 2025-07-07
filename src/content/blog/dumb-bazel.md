---
author: Vince Rose
pubDatetime: 2025-07-07T16:00:00.000Z
title: "Dumb Bazel"
slug: dumb-bazel
featured: true
ogImage: ../../assets/images/test-sharding/og.png
tags:
  - bazel
description: To macro or not to macro
---

"Let's just auto-generate targets" someone says. "It'll save us from writing boilerplate". Fast forward six months and your simple macro now has 35 attributes, handles edge cases you didn't know existed, and requires a PhD to configure properly.

Recently I've been having a lot of conversations with [@Farid Zakaria](https://fzakaria.com/) about _Dumb Bazel_. For lack of a better name, its really about:

- Could Bazel be _MORE_ approchable with fewer macros?
- Does _DRY_ (Don't Repeat Yourself) really apply to Bazel declarations?
- how much abstraction is too much abstraction?

Let's start with an open source macro and see how the macro grows in complexity over time. Then we can talk about the trade-offs. This is a slimmed down example of some of the more complex macros that you might see in an enterprise setting.

We're going to use [java_test_suite](https://github.com/vinnybod/rules_jvm/blob/0bef82e8d7038a6628faad06b9a57d10e536c2c5/java/private/create_jvm_test_suite.bzl#L26-L129) macro which comes from `rules_jvm`.

At a high level, it does these things:

1. Separates the "test" classes from the "non-test" classes by naming convention.
2. Creates a `java_library` out of the "non-test" classes
3. Creates a `java_test` out of all of the "test" classes, passing the `deps` and newly created `java_library` to each of them.

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

Now let's see how this macro starts to grow in complexity.

1. We have a test that inherits other classes which are also tests themselves. But our macro sorts the tests and non-tests. Each test is not "aware" of one another.

Let's address this by adding a new attribute `additional_library_srcs`. This will tell the macro "this is a test, but you should also include it in the `java_library` for other tests".

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

2. Our developers are complaining that while they like only having to write the one test suite, they need to specify some deviated attributes for certain tests. Like maybe some of these tests need to be tagged for more cpu (`cpu:2`), or they need a different size than the default.

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

**Note:** At _$DAYJOB_ we have an automation set up that pulls JUnit `@Tag` annotations from the source code and generates the `per_test_args` dict automatically.

3. The test suite assumes all tests end in `Test.java` but it turns out for "reasons" we need to also generate some tests for classes that end in `XYZ.java`.

We can solve for this with a simple `test_suffixes` attribute.

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

I could go on with more requirements, but I think this starts to get the point across well.

**Pros:**

With the macro approach, developers don't need to add new `java_test` or `java_library` targets every time they add new test files. It should be picked up and handled by the macro in the majority of cases.

Most of the developers that I've worked with don't really care or want to learn a new build system. They want things to "just work".

**Cons:**

Build avoidance suffers. We're adding every dependency and resource required in the package to the test suite. If we wrote out each test target individually, we could strip down the dependencies more precisely.

The debugging experience gets worse. When something goes wrong with a generated target, you're now debugging both your code AND the macro that generated it. Most of the developers will just rope in the Bazel SMEs because they don't understand what the macros are doing beyond the surface level.

The IDE support suffers when target generation is hidden behind complex macros.

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
        ":example-base-test-lib", # ExampleOne tests extends ExampleBaseTest
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
- Better tooling support `unused_deps`, [buildozer recommendations](https://github.com/bazelbuild/buildtools/issues/886), and IDEs.

## Finding the Balance

I'm not advocating for throwing away all macros. Some abstractions genuinely make sense, especially when you're encoding complex domain knowledge or company-specific patterns that would be error-prone to repeat.

It might surprise you to hear that I just spent this blog post advocating for a position that _I_ am not fully sold on. I still lean towards the macro set up where developers rarely need to touch the `BUILD` files.

**Why?** As an engineer working in Developer Productivity, my customers are the engineers working in the codebases. And what I hear from them is that **they do not want to manage Bazel files.**

I am **more** sold on implementing the _Dumb Bazel_ approach when its paired with more automation such as [Gazelle](https://github.com/bazel-contrib/bazel-gazelle). But Gazelle has a few [limitations](https://github.com/bazel-contrib/rules_jvm/tree/main/java/gazelle#source-code-restrictions-and-limitations) that our codebases would need to address to use it.
