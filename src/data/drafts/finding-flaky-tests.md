---
author: Vince Rose
pubDatetime: 2024-09-27T15:57:52.737Z
title: Finding Flaky Tests
slug: finding-flaky-tests
featured: false
ogImage: https://user-images.githubusercontent.com/53733092/215771435-25408246-2309-4f8b-a781-1f3d93bdf0ec.png
tags:
  - bazel
description: AstroPaper with the enhancements of Astro v2. Type-safe markdown contents, bug fixes and better dev experience etc.
---

One issue that can be difficult to catch with Bazel is flaky tests.
The reason being that since we get build avoidance, we don't always run the tests, so a flaky test might be added and not caught until much later.

```python
def this_is_a_test():
    random_number = random.randint(0, 1)
    if randum_number < 0.5:
        assert False
```

In this example, the test will fail 50% of the time, but if it's only run once in a blue moon, it might not be caught for a long time.

One way to catch these is to run the newly introduced tests multiple times in a row.

```bash
bazel test //:flaky_test --runs_per_test=10
```
