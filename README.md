<!---
  Licensed to the Apache Software Foundation (ASF) under one
  or more contributor license agreements.  See the NOTICE file
  distributed with this work for additional information
  regarding copyright ownership.  The ASF licenses this file
  to you under the Apache License, Version 2.0 (the
  "License"); you may not use this file except in compliance
  with the License.  You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing,
  software distributed under the License is distributed on an
  "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
  KIND, either express or implied.  See the License for the
  specific language governing permissions and limitations
  under the License.
-->

# DataFusion in Python

[![Python test](https://github.com/apache/arrow-datafusion-python/actions/workflows/test.yaml/badge.svg)](https://github.com/apache/arrow-datafusion-python/actions/workflows/test.yaml)
[![Python Release Build](https://github.com/apache/arrow-datafusion-python/actions/workflows/build.yml/badge.svg)](https://github.com/apache/arrow-datafusion-python/actions/workflows/build.yml)

This is a Python library that binds to [Apache Arrow](https://arrow.apache.org/) in-memory query engine [DataFusion](https://github.com/apache/arrow-datafusion).

Like pyspark, it allows you to build a plan through SQL or a DataFrame API against in-memory data, parquet or CSV files, run it in a multi-threaded environment, and obtain the result back in Python.

It also allows you to use UDFs and UDAFs for complex operations.

The major advantage of this library over other execution engines is that this library achieves zero-copy between Python and its execution engine: there is no cost in using UDFs, UDAFs, and collecting the results to Python apart from having to lock the GIL when running those operations.

Its query engine, DataFusion, is written in [Rust](https://www.rust-lang.org/), which makes strong assumptions about thread safety and lack of memory leaks.

Technically, zero-copy is achieved via the [c data interface](https://arrow.apache.org/docs/format/CDataInterface.html).

## How to use it

Simple usage:

```python
import datafusion
from datafusion import functions as f
from datafusion import col
import pyarrow

# create a context
ctx = datafusion.SessionContext()

# create a RecordBatch and a new DataFrame from it
batch = pyarrow.RecordBatch.from_arrays(
    [pyarrow.array([1, 2, 3]), pyarrow.array([4, 5, 6])],
    names=["a", "b"],
)
df = ctx.create_dataframe([[batch]])

# create a new statement
df = df.select(
    col("a") + col("b"),
    col("a") - col("b"),
)

# execute and collect the first (and only) batch
result = df.collect()[0]

assert result.column(0) == pyarrow.array([5, 7, 9])
assert result.column(1) == pyarrow.array([-3, -3, -3])
```

### UDFs

```python
from datafusion import udf

def is_null(array: pyarrow.Array) -> pyarrow.Array:
    return array.is_null()

is_null_arr = udf(is_null, [pyarrow.int64()], pyarrow.bool_(), 'stable')

df = df.select(is_null_arr(col("a")))

result = df.collect()

assert result.column(0) == pyarrow.array([False] * 3)
```

### UDAF

```python
import pyarrow
import pyarrow.compute
from datafusion import udaf, Accumulator


class MyAccumulator(Accumulator):
    """
    Interface of a user-defined accumulation.
    """
    def __init__(self):
        self._sum = pyarrow.scalar(0.0)

    def update(self, values: pyarrow.Array) -> None:
        # not nice since pyarrow scalars can't be summed yet. This breaks on `None`
        self._sum = pyarrow.scalar(self._sum.as_py() + pyarrow.compute.sum(values).as_py())

    def merge(self, states: pyarrow.Array) -> None:
        # not nice since pyarrow scalars can't be summed yet. This breaks on `None`
        self._sum = pyarrow.scalar(self._sum.as_py() + pyarrow.compute.sum(states).as_py())

    def state(self) -> pyarrow.Array:
        return pyarrow.array([self._sum.as_py()])

    def evaluate(self) -> pyarrow.Scalar:
        return self._sum


df = ctx.create_dataframe([[batch]])

my_udaf = udaf(MyAccumulator, pyarrow.float64(), pyarrow.float64(), [pyarrow.float64()], 'stable')

df = df.aggregate(
    [],
    [my_udaf(col("a"))]
)

result = df.collect()[0]

assert result.column(0) == pyarrow.array([6.0])
```

## How to install (from pip)

```bash
pip install datafusion
# or
python -m pip install datafusion
```

You can verify the installation by running:

```python
>>> import datafusion
>>> datafusion.__version__
'0.6.0'
```

## How to develop

This assumes that you have rust and cargo installed. We use the workflow recommended by [pyo3](https://github.com/PyO3/pyo3) and [maturin](https://github.com/PyO3/maturin).

Bootstrap:

```bash
# fetch this repo
git clone git@github.com:datafusion-contrib/datafusion-python.git
# prepare development environment (used to build wheel / install in development)
python3 -m venv venv
# activate the venv
source venv/bin/activate
# update pip itself if necessary
python -m pip install -U pip
# install dependencies (for Python 3.8+)
python -m pip install -r requirements-310.txt
```

Whenever rust code changes (your changes or via `git pull`):

```bash
# make sure you activate the venv using "source venv/bin/activate" first
maturin develop
python -m pytest
```

## How to update dependencies

To change test dependencies, change the `requirements.in` and run

```bash
# install pip-tools (this can be done only once), also consider running in venv
python -m pip install pip-tools
python -m piptools compile --generate-hashes -o requirements-310.txt
```

To update dependencies, run with `-U`

```bash
python -m piptools compile -U --generate-hashes -o requirements-310.txt
```

More details [here](https://github.com/jazzband/pip-tools)
