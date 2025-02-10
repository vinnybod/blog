---
author: Vince Rose
pubDatetime: 2025-02-09T16:00:00.000Z
title: Bazel Test Sharding in Detail (using Python)
slug: bazel-test-sharding-detail-python
featured: true
ogImage: ../../assets/images/test-sharding/og.png
tags:
  - bazel
description: How to implement test sharding into a Bazel test runner.
---

This post is going to go into more detail on _how_ test sharding works in Bazel. If you haven't already seen the previous blog post about what test sharding _is_, start [here](https://vincerose.dev/posts/bazel-test-sharding/).

Test sharding in Bazel is implemented by the runner, or put another way the _executable_ that is called by the **\*\_test** rule. Some runners, like the `JUnit` runner in [rules_jvm](https://github.com/bazel-contrib/rules_jvm/blob/main/java/src/com/github/bazel_contrib/contrib_rules_jvm/junit5/TestSharding.java) have it implemented. Others, like the `py_test` runnner in `rules_python`, do not.

In this post, we'll go over how to implement test sharding in a Python test target that uses `pytest`.

## Bazel Environemnt

When Bazel invokes a **\*\_test** target, it sets a [few environment variables](https://bazel.build/reference/test-encyclopedia). We can use these variables to split up the test cases.

| Variable               | Description                                            | Status   |
| ---------------------- | ------------------------------------------------------ | -------- |
| TEST_SHARD_INDEX       | shard index, if sharding is used                       | optional |
| TEST_SHARD_STATUS_FILE | path to file to touch to indicate support for sharding | optional |
| TEST_TOTAL_SHARDS      | total shard count, if sharding is used                 | optional |

First thing we want to do is determine if we are running in a a sharded environment. If these variables are set, we can assume we are.

If we are in a sharded environment, first we should touch the status file. Bazel checks the updated time of this file to see if the test runner supports sharding. If the file is not updated, Bazel will assume that the runner does not, and fail the test.

In Python, this is a simple operation:

```python
import os
from pathlib import Path

Path(os.environ["TEST_SHARD_STATUS_FILE"]).touch()
```

Next we look at the shard index and the total number of shards. We can use these to determine which tests to run.
The selection of tests is arbitrary, but **must be determininstic**. For this example, let's just use a round-robin selection.

In Python, this would look something like this:

```python
import os

tests = some_function_that_gets_tests()
index = int(os.environ["TEST_SHARD_INDEX"])
total_shards = int(os.environ["TEST_TOTAL_SHARDS"])

filtered_tests = [test for i, test in enumerate(tests) if i % total_shards == index]

run_tests(filtered_tests)
```

Okay so now in our test runner, we know

1. If we are in a sharded environment
2. The shard index and total number of shards
3. How to split up the tests

With this information, we should be able to implement test sharding for any language and test runner.

## Applying what we learned

Let's apply the above to a real world example by adding test sharding to the `pytest` runner from [rules_py](https://github.com/aspect-build/rules_py/blob/main/docs/py_test.md).

The runner does quite a few things, but let's key in on the part that actually invokes pytest. Remember that that the runner is really just any executable that Bazel calls. In this case, it is a Python script that calls `pytest`.

Reduced for brevity:

```python
import pytest
exit_code = pytest.main(args)
```

Pytest has plugin hooks that we can use to modify the test collection. To add sharding, we will create a pytest plugin that uses the `pytest_collection_modifyitems` hook. I will not go into too much detail on writing the plugin. For the actual implementation, I forked [pytest-shard](https://github.com/AdamGleave/pytest-shard) and made a couple changes to it. You can see the full code [here](https://github.com/aspect-build/rules_py/blob/a23ffaa728edeb253bd50a1f3d96c1720a921b13/py/private/pytest_shard/pytest_shard.py).

The important thing to note is that the plugin adds `--shard-id` and `--num-shards` arguments to the pytest command, and it uses a round-robin selection to filter the tests.

The below diff are the [actual changes](https://github.com/aspect-build/rules_py/pull/493/files#diff-eab700c56364dc4619c882e7e89bf26cd4cd1cd9efe61284e3bd0607bfe152bc) made to the pytest runner. The changes are:

1. Import the `ShardPlugin`
2. Check for the shard environment variables
3. Add the shard arguments to the `pytest` command
4. Touch the status file
5. Pass the arguments and plugin to `pytest`

```diff
diff --git a/py/private/pytest.py.tmpl b/py/private/pytest.py.tmpl
index e8a4d6c2..fa60eb0e 100644
--- a/py/private/pytest.py.tmpl
+++ b/py/private/pytest.py.tmpl
@@ -14,10 +14,13 @@

 import sys
 import os
+from pathlib import Path
 from typing import List

 import pytest

+from aspect_rules_py.py.private.pytest_shard.pytest_shard import ShardPlugin
+
 if __name__ == "__main__":
     # Change to the directory where we need to run the test or execute a no-op
     $$CHDIR$$
@@ -40,6 +43,16 @@ if __name__ == "__main__":
         if suite_name:
             args.extend(["-o", f"junit_suite_name={suite_name}"])

+    test_shard_index = os.environ.get("TEST_SHARD_INDEX")
+    test_total_shards = os.environ.get("TEST_TOTAL_SHARDS")
+    test_shard_status_file = os.environ.get("TEST_SHARD_STATUS_FILE")
+    if all([test_shard_index, test_total_shards, test_shard_status_file]):
+        args.extend([
+            f"--shard-id={test_shard_index}",
+            f"--num-shards={test_total_shards}",
+        ])
+        Path(test_shard_status_file).touch()
+
     test_filter = os.environ.get("TESTBRIDGE_TEST_ONLY")
     if test_filter is not None:
         args.append(f"-k={test_filter}")
@@ -52,7 +65,7 @@ if __name__ == "__main__":
     if len(cli_args) > 0:
         args.extend(cli_args)

-    exit_code = pytest.main(args)
+    exit_code = pytest.main(args, plugins=[ShardPlugin()])

     if exit_code != 0:
         print("Pytest exit code: " + str(exit_code), file=sys.stderr)
```

## Conclusion

For a complete working example see [bazel-examples](https://github.com/vinnybod/bazel-examples/blob/main/test-shards-python/BUILD.bazel). If you'd like to use this in your own project, just make sure to be using `rules_py` 1.3.0 or later.

This is something that was already implemented in other rulesets such as [rules_python_pytest](https://github.com/caseyduquettesc/rules_python_pytest). However, I was already using `rules_python`, `rules_py`, and `rules_uv`. I did not want to add yet another Python ruleset to my project, and also [Gazelle](https://github.com/bazelbuild/rules_python/blob/main/gazelle/README.md) integrates nicely with `rules_py` already.

Thanks to the developers of `pytest-shard`, `rules_python_pytest`, and `rules_py` for their prior work.
