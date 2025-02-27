---
author: Vince Rose
pubDatetime: 2025-02-27T16:00:00.000Z
title: Test Filtering with Ruff
slug: ruff-test-filtering
featured: true
ogImage: ../../assets/images/test-sharding/og.png
tags:
  - python
  - ruff
  - pytest
description: How to implement test filtering with ruff analyze graph.
---

[ruff](https://github.com/astral-sh/ruff) has been an immensely useful tool for linting and formatting for a while now. I discovered recently that it also has a feature called `analyze graph` that can be used to build a dependency graph of all the python files in the project.

I am sure there are a variety of use cases for this, but one that I find particularly interesting is **filtering tests to get faster feedback loops** during development.

When you have a large codebase, running the entire test suite can take a long time. If you can filter the tests to only run the ones that are affected by the changes you made, you can get faster feedback.

For this example, I am going to use a [uv python project](https://github.com/vinnybod/blog-examples/tree/main/ruff-graph). It only has a few files and tests, but it should be enough to demonstrate the concept.

## Traversing the graph

First let's look at the `ruff analyze graph` command
```bash
➜  uv run ruff analyze graph --direction=dependents
warning: `ruff analyze graph` is experimental and may change without warning
{
  "scripts/graph_analyzer.py": [],
  "src/__init__.py": [],
  "src/hello.py": [],
  "src/lib_a.py": [
    "src/tests/test_lib_a.py"
  ],
  "src/lib_b.py": [
    "src/lib_a.py",
    "src/lib_c.py",
    "src/tests/test_lib_b.py"
  ],
  "src/lib_c.py": [
    "src/lib_a.py"
  ],
  "src/tests/__init__.py": [],
  "src/tests/test_lib_a.py": [],
  "src/tests/test_lib_b.py": []
}
```

It returns a json object in the form of an adjacency list where the keys are the files and the values are the files that depend on them. Note the `--direction=dependents` flag. This is important because we want to know which files depend on the file we are changing. Without it, we would get the inverse.

Now that we have this information, let's write a small script that outputs the transitive list of test files that are impacted by an input list of files.

```python
import subprocess
import json
from collections import deque
import sys

def get_downstream_dependents(dependents_map, changed_files):
    visited = set()
    queue = deque(changed_files)

    while queue:
        current = queue.popleft()
        if current not in visited:
            visited.add(current)
            for neighbor in dependents_map.get(current, []):
                queue.append(neighbor)

    return visited

def get_dependency_graph():
    res = subprocess.run(
        ["ruff", "analyze", "graph", "--direction=dependents"],
        capture_output=True,
        text=True,
    )
    return json.loads(res.stdout)


if __name__ == "__main__":
    test_dir = sys.argv[1]
    changed_files = sys.argv[2:]
    impacted_files = get_downstream_dependents(get_dependency_graph(), changed_files)
    impacted_test_files = [f for f in impacted_files if f.startswith(test_dir)]

    print(
        f"When {changed_files} changes, the downstream dependents are: {impacted_files}"
    )
    print(f"Test files that are impacted by these changes are: {impacted_test_files}")
```

Here is an example invocation:
```bash
➜  uv run scripts/graph_analyzer.py src/tests src/lib_a.py
When ['src/lib_a.py'] changes, the downstream dependents are: {'src/tests/test_lib_a.py', 'src/lib_a.py'}
Test files that are impacted by these changes are: ['src/tests/test_lib_a.py']
```

## Getting the list of changed files

We need to be able to get the list of changed files. This can be done with a git command. Let's just stick with diffing for files that have changed since the last commit.

```bash
➜  git diff --name-only --relative | cat
src/lib_a.py
```

## Putting it all together

I am going to modify the script to output only the test files and then pipe that to `xargs` to run the tests.

```python
if __name__ == "__main__":
    test_dir = sys.argv[1]
    changed_files = sys.argv[2:]
    impacted_files = get_downstream_dependents(get_dependency_graph(), changed_files)
    impacted_test_files = [f for f in impacted_files if f.startswith(test_dir)]
    print(" ".join(impacted_test_files))
```


```bash
➜ uv run scripts/graph_analyzer.py src/tests $(git diff --name-only --relative) | xargs uv run python -m unittest
foo from lib_b in ruff-graph/src/lib_b.py
foo from lib_b in ruff-graph/src/lib_b.py
foo from lib_c
foo from lib_b in ruff-graph/src/lib_b.py
.foo from lib_a
.
----------------------------------------------------------------------
Ran 2 tests in 0.000s

OK
```

**SUCCESS!** We have a command that will only run the tests that are impacted by the changes we made.

## Making it a bit more user friendly

I added a `Makefile` target to make the development experience a bit nicer.
Now you can just run `make run-tests` to run the tests that are impacted by the changes you made.

```Makefile
run-tests:
	@echo "Changed files: $(shell git diff --name-only --relative)"
	uv run scripts/graph_analyzer.py src/tests $(shell git diff --name-only --relative) | xargs uv run python -m unittest
```

## Conclusion

So now, we have a command to run when developing locally that will only run a subset of tests based on the uncommitted changes. This can save a lot of time and give faster feedback.

There are some limitations to the current implementation.
* 3rd party dependencies are not tracked ([yet](https://github.com/astral-sh/ruff/issues/13431))
* There are likely edge cases in the git diff that I haven't handled here such as when files get renamed
* If non-python files are changed, they are not tracked in the graph
* pytest fixtures are not easy to track via static analysis of imports. Special logic would be needed to look at `conftest.py` files and track which files should be run based on those changes

I'd only recommend using this for local development. In CI, running the full test suite will give you the most confidence that your changes are correct. The full example can be found [on Github](https://github.com/vinnybod/blog-examples/tree/main/ruff-graph).
