---
author: Vince Rose
pubDatetime: 2025-04-28T16:00:00.000Z
title: One weird trick to make PyYAML faster
slug: make-pyyaml-faster-cloader
featured: true
ogImage: ../../assets/images/test-sharding/og.png TODO
tags:
  - python
  - pyyaml
description: How to make PyYAML faster by using the C extension.
---

In most cases, the performance of PyYAML's pure python loader is sufficient.
However, in [Empire](TODO), we load 100s of YAML files, some as large as [N MB](TODO). With my MacBook Pro, the default loader is fast enough that I didn't notice any issues. But in CI, on the free GitHub runners, tests were slow enough that I started trying to find bottlenecks.

One of the bottlenecks I found was the loading of all the YAML files. Well, it turns out that PyYAML has a C extension that can be used to speed up the loading of YAML files. The C extension is essentially(?) [libyaml](https://pyyaml.org/wiki/PyYAML), which is a C library for parsing YAML.

## What does the solution look like?

```py
try:
    from yaml import CSafeDumper as Dumper
    from yaml import CSafeLoader as Loader
except ImportError:
    from yaml import Dumper, Loader
```

That's it. Keep reading only if you care about the details.

## How does native extension loading work?

The C extension is a compiled library that is loaded into the Python interpreter. When you install PyYAML from PyPi, it downloads a wheel file that contains the C extension, assuming one is available for your platform.

```bash
> wget https://files.pythonhosted.org/packages/py3/p/pyyaml/PyYAML-6.0-cp39-cp39-macosx_10_9_x86_64.whl
> unzip -l PyYAML-6.0-cp39-cp39-macosx_10_9_x86_64.whl | grep ".so"
TODO
```

## Setting the stage

I set up a small test case to compare the performance of the pure python loader and the C extension. The test case loads a fairly large yaml file (~500kb). Let's load it 1000 times.

```python
for i in range(1000):
    with open("test.yaml", "r") as f:
        data = yaml.safe_load(f)
```

```bash

```

The test took about n seconds to load with the default, pure python loader in a github action free runner.

```bash

```

In Empire, it took the test times from n minutes to m minutes. Not bad for a few lines of code.
