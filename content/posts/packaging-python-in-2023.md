---
title: "Packaging Python Projects in 2023"
date: 2023-09-04T16:32:41+02:00
---

Clean package management with Pyhon is still a tricky subject even though there are a lot of
resources available online. This is also because a lot of the resources found online
still make recommendations which are becoming slowly obsolete, for example by still using
`setup.py`. I think there is also a lack of resources which cover all topics which might be relevant
for setting up a new package. In this post, I will show how to set up a new package in Python from
scratch with all features I consider useful and important for a good Python package. I really like
the packaging blogpost [Bastian Venthur](https://venthur.de/2022-12-18-python-packaging.html) about
the current best practices of Python packaging, whch is worth a read as well.

The package will have following properties, which can be adapted based on preference and use-case:

- Uses `pyproject.toml`, which is now the recommended standard by PyPA, and makes integration with
  other tools inside the Pyton ecosystem a lot easier.
- Uses [`setuptools`](https://setuptools.pypa.io/en/latest/setuptools.html) as the distibution
  and build tool.
- Uses [`Sphinx`](https://www.sphinx-doc.org/en/master/), [`reStructuredText`](https://docutils.sourceforge.io/rst.html)
  and the [`sphinx-rtd-theme`](https://pypi.org/project/sphinx-rtd-theme/) for writing, building
  and rendering the documentation.
- Has a single-sourced version inside the `pyproject.toml` file.
- Has a `CHANGELOG` to list all changes between versions.
- Has a unittest folder with tests which can be executed with `pytest` or any other test framework.
- Has examples inside the documentation which can also be automatically tested using `doctest`.
- Has a `.flake8` configuration to be used with the [`flake8`](https://flake8.pycqa.org/en/latest/) linter.
- Has a working GitHub CI/CD configuration.
- Uses a uniform line width of 100 for both [`black`](https://github.com/psf/black) (my preferred
  auto-formatter) and `flake8`.

All the shell instructions were written for an Ubuntu system, so those might need adaptions
if your are using Windows or another OS.

## Writing the source code

We will write a `Catlifier` class first which is then packaged. The `Catlifier` uses the `crcmod`
package because I would also like to showcase cross-referencing other libraries in the
documentation at a later stage.

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


def get_catlifier_crc_calculator() -> PredefinedCrc:
    return PredefinedCrc("crc-ccitt-false")


class Catlifier:
    def __init__(self, base_text: str):
        self.base_text = base_text
        self.crc_calculator = PredefinedCrc("crc-ccitt-false")

    def catlify(self) -> str:
        """"Catlify a given string. Also updates internal CRC calculator with catlified data."""
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
        instance.crc_calculator.update(catlified_text.encode())
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

[tool.black]
line-length = 100
```

There are various ways of single-sourcing the Python version, and the most common ways
were listed [here](https://packaging.python.org/en/latest/guides/single-sourcing-package-version/).

I really like the variant 5, which puts the version information into the package configuration
and uses the new [`import.metadata`](https://docs.python.org/3/library/importlib.metadata.html) API
to retrieve the version if this becomes necessary. This means I don't have to make changes inside
the source code for version bumps anymore.

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
inside a `venv` folder to keep the system python clean:

```sh
python3 -m venv venv
. venv/bin/activate
pip install build
python3 -m build .
```

All following shell commands will assume an active virtual environment.

I still add a `requirements.txt` file to the package, but I simply forward the requirements
to `pyproject.toml` because this is a pure library. If your are working on a project with a binary
where exact pinning of dependency versions is important, for example for a deployment, you should
adapt the `requirements.txt` for your needs.

The content of the `requirements.txt` file for my case is simple:

```sh
.
```

I usually also add the [GitHub Python `.gitignore`](https://github.com/github/gitignore/blob/main/Python.gitignore)
to my Python projects, which contains everything that should not be part of version control.

All of my projects also have a `.flake8` linter configuraion file to my projects. Ideally, I would
like this configuration to be part of the `pyproject.toml`, similarly to how `black` is configured
there as well. However, `.flake8` still does not support specifying configuration there. However,
this might change in the future. You can track
[corrensponding GitHub issue](https://github.com/PyCQA/flake8/issues/234) for the state of
`pyproject.toml` support.

`.flake8`:

```txt
[flake8]
max-line-length = 100
ignore = D203, W503
per-file-ignores =
    */__init__.py: F401
exclude =
    .git,
    __pycache__,
    docs/conf.py,
    old,
    build,
    dist,
    venv
max-complexity = 10
extend-ignore =
    # See https://github.com/PyCQA/pycodestyle/issues/373
    E203,
```

## Adding unittests

Next, we add some tests for out catlifier module. These will also be automatically executed
by the CI at a later stage. We will keep our tests outside the source code.
The [pytest documentation](https://docs.pytest.org/en/stable/explanation/goodpractices.html) provides
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

## Adding Sphinx documentation

Next, we set up Sphinx to generate documentation for out Catlifier from the source code
automatically. This can be done using the
[`autodoc`](https://www.sphinx-doc.org/en/master/usage/extensions/autodoc.html) extension. We also
want to use the [`intersphinx`](https://www.sphinx-doc.org/en/master/usage/extensions/intersphinx.html)
extension to provide cross-referencing to external packages like `crcmod` and the
[`doctest`](https://www.sphinx-doc.org/en/master/usage/extensions/doctest.html) extension to
automatically test code examples inside the documentation which we marked specifically.

Finally, I also added the [`shinx_rtd_theme`](https://sphinx-rtd-theme.readthedocs.io/en/stable/)
which is a bit cleaner and and more readable than the default [`Alabaster`](https://alabaster.readthedocs.io/en/latest/)
theme provided by Sphinx by default in my opinion.

We create a documentation folder first and install all required packages.

```sh
pip install sphinx-rtd-theme
mkdir docs
cd docs
sphinx-quickstart --no-sep -p "Catlifier" -a "Robin Mueller" -r "0.1.0" -l en
```

This gives us a good starting point with a `conf.py` lookling like this:

```py
# Configuration file for the Sphinx documentation builder.
#
# For the full list of built-in configuration values, see the documentation:
# https://www.sphinx-doc.org/en/master/usage/configuration.html

# -- Project information -----------------------------------------------------
# https://www.sphinx-doc.org/en/master/usage/configuration.html#project-information
from importlib.metadata import version

project = 'Catlifier'
copyright = '2023, Robin Mueller'
author = 'Robin Mueller'
release = "0.1.0"

# -- General configuration ---------------------------------------------------
# https://www.sphinx-doc.org/en/master/usage/configuration.html#general-configuration

extensions = []

templates_path = ['_templates']
exclude_patterns = ['_build', 'Thumbs.db', '.DS_Store']

language = 'en'

# -- Options for HTML output -------------------------------------------------
# https://www.sphinx-doc.org/en/master/usage/configuration.html#options-for-html-output

html_theme = 'alabaster'
html_static_path = ['_static']
```

Next, we make all the necessary adaptions to the configuration file:

```py
# Configuration file for the Sphinx documentation builder.
#
# For the full list of built-in configuration values, see the documentation:
# https://www.sphinx-doc.org/en/master/usage/configuration.html

# -- Project information -----------------------------------------------------
# https://www.sphinx-doc.org/en/master/usage/configuration.html#project-information
from importlib.metadata import version

project = 'Catlifier'
copyright = '2023, Robin Mueller'
author = 'Robin Mueller'
# Use importlib.metadata API to extract version automatically
version = release = version("catlifier")

# -- General configuration ---------------------------------------------------
# https://www.sphinx-doc.org/en/master/usage/configuration.html#general-configuration

extensions = [
    "sphinx.ext.autodoc",
    "sphinx.ext.intersphinx",
    "sphinx.ext.doctest",
    "sphinx_rtd_theme",
]

# Disable the doctests of the full package because those would require the explicit specification
# of imports. The doctests inside the source code are covered by pytest, using the --doctest-modules
# configuration option.
doctest_test_doctest_blocks = ""

templates_path = ['_templates']
exclude_patterns = ['_build', 'Thumbs.db', '.DS_Store']

# Mapping for external packages
intersphinx_mapping = {
    "python": ("https://docs.python.org/3", None),
    "crcmod": ("https://crcmod.sourceforge.net/", None)
}

language = 'en'

# -- Options for HTML output -------------------------------------------------
# https://www.sphinx-doc.org/en/master/usage/configuration.html#options-for-html-output

html_theme = "sphinx_rtd_theme"
html_static_path = ['_static']
```

Next, we add example code, which will be automatically tested by the `doctest` extension.
We create a new `examples.rst` inside the `docs` folder with the following content:

```rst
Examples
===========

Example usage

.. testcode:: catlifier

    from catlifier import Catlifier
    
    test_string = "hello world" 
    catlifier = Catlifier(test_string)
    print(catlifier.catlify())
    
Output:

.. testoutput:: catlifier

    hello world🐈

```

`doctest` will automatically `testcode` sections and verify their output against the
`testoutput` section. You can also add [doctests](https://docs.python.org/3/library/doctest.html)
inside the documentation blocks of your source code, but for those it might be better to test
them with another tool like `pytest`.

We would also like to document our API, and generate that documentation automatically
from the source code. For this, we create a new `api.rst` with the following content:

```rst
API
====

.. automodule:: catlifier
 :members:
 :undoc-members:
 :show-inheritance:
```

Then, we create a new documentation table of content inside the `index.rst` file:

```rst
Welcome to Catlifer's documentation!
====================================

.. toctree::
   :maxdepth: 2
   :caption: Contents:

.. toctree::
   :maxdepth: 3

   examples
   api

Indices and tables
==================

* :ref:`genindex`
* :ref:`modindex`
* :ref:`search`
```

We can now build the documentation using `make html` inside the `docs` folder.

```sh
cd docs
make html 
firefox _build/html/index.html
```

If you check the API documentation, you should also see that the [`PredefinedCrc`](https://crcmod.sourceforge.net/crcmod.predefined.html#crcmod.predefined.PredefinedCrc)
cross-reference is working properly.

You can test the examples using

```sh
cd docs
make doctest
```

The best thing about this is that this can be checked in the CI, so you can catch your examples
becoming out of data, for example when the API changes.

This is a good starting point for providing useful documentation for users 🎉. If you work on
an open-source project, you should also consider a service like [readthedocs](https://docs.readthedocs.io/en/stable/)
where you can host the documentation of your package for free.

As a final step, I also like to add a `docs` specific `requirements.txt` file which only includes
the dependencies for building the documentation with the following content.

`docs/requirements.txt`:

```txt
sphinx-rtd-theme==1.2.0
```

## Testing the upload of the package

Most of the following steps are based on the official [packaging tutorial](https://packaging.python.org/en/latest/tutorials/packaging-projects/).
We have the most important components of our package and would like to upload it to PyPI now.
We build the package first like already shown.

```sh
python3 -m build .
```

After that, you can create an account on [Test PyPI](https://test.pypi.org) to test your package
upload without affecting the normal package index.

With everything in place, you can upload with

```sh
python3 -m twine upload --repository testpypi dist/*
```

If everything goes well, you should see your package on the Test PyPI.

Before uploading any package and doing releases in general, I really like to add a CHANGELOG to
a project so it becomes easier for users to figure out what changed between versions. I usually use
the CHANGELOG format proposed by [Keep A Changelog](https://keepachangelog.com/en/1.1.0/).

## Adding GitHub CI

Finally, assuming that your project is hosted on GitHub, it is relatively easy to add a CI
configuration for your Python project.

Our CI configuration will install the package, run the tests, build the documentation and
lint the code with `flake8`. Add a `.github/workflows/package.yml` file to your project:

```yml
name: package

on: [push]

jobs:
  build:

    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest]
        python-version: ['3.8', '3.9', '3.10', '3.11']

    steps:
    - uses: actions/checkout@v3

    - name: Set up Python ${{ matrix.python-version }}
      uses: actions/setup-python@v4
      with:
        python-version: ${{ matrix.python-version }}

    - name: Install dependencies
      run: |
        python3 -m pip install --upgrade pip setuptools wheel
        pip install flake8
        pip install -r docs/requirements.txt
        pip install .
 
    - name: Build documentation and examples
      run: |
        sphinx-build -b html docs docs/_build
        sphinx-build -b doctest docs docs/_build

    - name: Lint with flake8
      run: |
        # stop the build if there are Python syntax errors or undefined names
        flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics
        # exit-zero treats all errors as warnings.
        flake8 . --count --exit-zero --max-complexity=10 --max-line-length=100 --statistics

    - name: Run tests and generate coverage data
      run: |
        python3 -m pip install coverage pytest
        coverage run -m pytest
```

If you push your package to GitHub now, The GitHub code actions CI should trigger automatically.

## Conclusion

We have written a complete small package contaning all the features I consider important
for a good Python package. I hope that this mini-workshop can help some people who are considering
publishing their project to PyPI or are looking for a general guide on how to set up a Python
package.

You can find the full resulting source code on [GitHub](https://github.com/robamu/catlifier-py)
as well.
