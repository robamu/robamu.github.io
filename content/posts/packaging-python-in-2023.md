---
title: "Packaging Python Projects in 2023"
date: 2023-09-04T16:32:41+02:00
draft: true
---

Clean package management with Pyhon is still a tricky subject even though there are a lot of
resources available online. This is also because a lot of the resources found online
still make recommendations which are becoming slowly obsolete, for example by still using
`setup.py`. In this post, I want to show how to set up a new package in Python
with all features I consider useful and important for a good Python package.
I really like the packaging blogpost by [Bastian Venthur](https://venthur.de/2022-12-18-python-packaging.html)
about the current best practices of Python packaging, whch is worth a read as well.

The package will have following properties, which can be adapted based on preference and use-case:

- Uses `pyproject.toml`, which is the recommended standard by PyPA, and makes integration with
  other tools inside the Pyton ecosystem a lot easier.
- Uses [`setuptools`](https://setuptools.pypa.io/en/latest/setuptools.html) as the distibution
  and build tool.
- Uses [`Sphinx`](https://www.sphinx-doc.org/en/master/), [`reStructuredText`](https://docutils.sourceforge.io/rst.html)
  and the [`sphinx-rtd-theme`](https://pypi.org/project/sphinx-rtd-theme/) for writing, building
  and rendering the documentation.
- Has a single-sourced version inside the `pyproject.toml` file.
- Has a `CHANGELOG` to list all changes between versions.
- Has examples inside the documentation which can also be automatically tested using `doctest`.
- Has a unittest folder with tests which can be executed with `pytest` or any other test framework.
- Has a `.flake8` configuration to be used with the `flake8` linter.
- Has a working github CI/CD configuration.

## Writing the source code

We will write a `Catlifier` class first which is then packaged. The `Catlifier` uses the `crcmod`
package to do some magic.

Let's create the project folder and some python code first:

```
mkdir catlifier-py
cd catlifier-py
touch catlifier.py
```

Here is the python code:

```py
from __future__ import annotations
from crcmod.predefined import PredefinedCrc

class Catlifier:
    def __init__(self, base_text: str):
        self.base_text = base_text
        self.crc_calculator = PredefinedCrc("crc-ccitt-false")

    def catlify(self) -> str:
        """"Catlify a gven string. Also updates internal CRC calculator with
        catlified data."""
        catlified = self.base_text + "🐈"
        self.crc_calculator.new()
        self.crc_calculator.update(catlified.encode())
        return catlified
    
    @classmethod
    def uncatlify(cls, catlified_text: str) -> Catlifier:
        """Generates a new :py:class:`Catlifier` instance from a catlified text.
        """
        stripped_text = catlified_text.rstrip("🐈")
        instance = cls(stripped_text)
        instance.crc_calculator.update(catlified_text)
        return instance
```

First, we need to convert our directory structure into a format which can be used by something
like `setuptools`.  We use the source layout. You can read more about the distinction between
source layout and flat layout [here](https://packaging.python.org/en/latest/discussions/src-layout-vs-flat-layout/). We also add a `__init__.py` to mark the directory as a package directory.

```sh
mkdir src
mv catlifier.py src
touch __init__.py
```

## Writing the package configuration file

As the next step, we create the `pyproject.toml` package configuration file.

```toml
[build-system]
requires = ["setuptools>=61.0"]
build-backend = "setuptools.build_meta"

[project]
name = "catlifier"
description = "My catlifier library"
readme = "README.md"
version = "0.1.0"
requires-python = ">=3.8"
license = {text = "Apache-2.0"}
authors = [
  {name = "Robin Mueller", email = "robin.mueller.m@gmail.com"}
]
keywords = ["cats", "purr", "packaging"]
classifiers = [
    "Development Status :: 5 - Production/Stable",
    "License :: OSI Approved :: Apache Software License",
    "Natural Language :: English",
    "Operating System :: POSIX",
    "Operating System :: Microsoft :: Windows",
    "Programming Language :: Python :: 3",
    "Programming Language :: Python :: 3.7",
    "Programming Language :: Python :: 3.8",
    "Programming Language :: Python :: 3.9",
    "Topic :: Communications",
    "Topic :: Software Development :: Libraries",
    "Topic :: Software Development :: Libraries :: Python Modules",
]
dependencies = [
    "crcmod~=1.7",
]

[project.urls]
"Homepage" = "https://github.com/robamu/catlifier"
```

There are various ways of single-sourcing the Python version, and the most common ways
were listed [here](https://packaging.python.org/en/latest/guides/single-sourcing-package-version/).

I really like the variant 5, which puts the version information into the package configuration
and uses the new `import.metadata` API to retrieve the version if this becomes necessary.
This means I don't have to make changes inside the source code for version bumps anymore.

The directory tree should look like this now:

```sh
catlifier-py
├── pyproject.toml
└── src
   └── catlifier.py
   └── __init__.py
```

This is all that is required for a package which can be re-distributed and uploaded to a package
index! You can test building your package using `build`. We also set up a virtual environment
to keep the system python clean:

```sh
python3 -m venv venv
. venv/bin/activate
pip install build
python3 -m build .
```

## Adding unittests

Next, we add some tests for out catlifier module. These will also be automatically executed
by the CI at a later stage. We will keep our tests outside the source code.
The [pytest](https://docs.pytest.org/en/stable/explanation/goodpractices.html) provides
a bit of reasoning why it makes sense to keep the tests seperated from the source code.

Please note that the `test_` prefix for the test module names is necessary for `pytest` to
find the test modules. The `__init__.py` module specifier is optional for `pytest`, but is
useful for other test frameworks like `unittest` to find all the tests.

```sh
mkdir tests
touch test_catlifier.py
touch __init__.py
```

Here is the test code for `test_catlifier.py`

```py
from unittest import TestCase
from catlifier import Catlifier

class TestCatlifier(TestCase):

    def setUp(self) -> None:
        self.test_str = "hello world"
        
    def test_catlify(self):
        catlifier = Catlifier(self.test_str)
        catlified = catlifier.catlify()
        self.assertEqual(catlifier.base_text, self.test_str)
        self.assertEqual(catlified, self.test_str + "🐈")
    
    def test_uncatlify(self):
        catlified = self.test_str + "🐈"
        catlifier = Catlifier.uncatlify(catlified)
        self.assertEqual(catlifier.base_text, self.test_str)
```

We install `pytest` first:

```sh
pip install pytest
```

You can test your package using the following command

```sh
python3 -m pytest .
```

Pytest should be able to find all the tests inside the `tests` directory as long as they
use the `test_*` naming convention.

## Add Sphinx documentation

