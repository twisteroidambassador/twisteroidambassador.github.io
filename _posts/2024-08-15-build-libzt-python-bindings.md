---
layout: post
title: "Building the Zerotier SDK (libzt) Python bindings"
---

The [Zerotier SDK, a.k.a. `libzt`](https://github.com/zerotier/libzt/) connects to a Zerotier network in userspace,
without creating a virtual network interface.
It provides bindings for quite a few languages, and I'm interested in the Python binding.

As of today, [there are only pre-built wheels for Python 3.5 - 3.9 on PyPI](https://pypi.org/project/libzt/),
and they are pretty old.

So here's how I compiled `libzt` Python bindings for Python 3.11, both on Linux amd64 and `termux`.
These notes should be helpful for building on other Python versions as well.

## Prerequisites

Clone `libzt`'s git repository.

Install `cmake` and `swig` via distro's package manager.

Make a new Python 3 venv, and `pip install poetry`.
This is simple on regular Linux,
but for `termux`,
the installation will fail because it can't build `cryptography`,
a dependency of a dependency of a dependency,
from source.
[Termux Wiki explains this.](https://wiki.termux.com/wiki/Python)

The solution:
```shell
# install precompiled cryptography from package manager
pkg install python-cryptography
# create venv that uses system packages
python -m venv --system-site-packages ./venv
# and then install poetry in this venv
. ./venv/bin/activate
pip install poetry
```

## Fixing the build process

`libzt` [adapted `poetry` for its Python bindings build process a while ago](https://github.com/zerotier/libzt/pull/167),
but hasn't changed its build scripts to match.
Therefore, simply running `build.sh host-python release` does not work.

However, when using a modern version of `poetry`,
`cd`-ing into `pkg\pypi` and running `poetry build` does not work either,
producing only empty packages that does not contain any code.
This is because [`poetry` still does not have a stable, documented way of building native code extensions](https://github.com/python-poetry/poetry/issues/2740),
and the [current method no longer works due to a change in defaults](https://github.com/python-poetry/poetry/issues/7470).

Solution:
```diff
diff --git a/pkg/pypi/pyproject.toml b/pkg/pypi/pyproject.toml
index 84e63a7..a79f122 100644
--- a/pkg/pypi/pyproject.toml
+++ b/pkg/pypi/pyproject.toml
@@ -44,7 +44,10 @@ classifiers = [
 packages = [
     { include = "libzt" },
 ]
-build = "build.py"
+
+[tool.poetry.build]
+script = "build.py"
+generate-setup-file = true

 [tool.poetry.dependencies]
 python = "^3.6"
```

Then, **remember to delete the existing `setup.py`**, so that `poetry` can regenerate it.

Now running `poetry build` will actually attempt to build something!
Just try it, and fix any problems that come up.


## Code fixes

If there are errors about `prometheus` includes not being found:

```diff
diff --git a/pkg/pypi/build.py b/pkg/pypi/build.py
index 557a648..b58d6cb 100644
--- a/pkg/pypi/build.py
+++ b/pkg/pypi/build.py
@@ -27,6 +27,8 @@ INCLUDE_DIRS = [
     os.path.join(ROOT_DIR, "ext/ZeroTierOne/include"),
     os.path.join(ROOT_DIR, "ext/ZeroTierOne"),
     os.path.join(ROOT_DIR, "ext/ZeroTierOne/ext"),
+    os.path.join(ROOT_DIR, "ext/ZeroTierOne/ext/prometheus-cpp-lite-1.0/simpleapi/include"),
+    os.path.join(ROOT_DIR, "ext/ZeroTierOne/ext/prometheus-cpp-lite-1.0/core/include"),
     os.path.join(ROOT_DIR, "ext/ZeroTierOne/node"),
     os.path.join(ROOT_DIR, "ext/ZeroTierOne/service"),
     os.path.join(ROOT_DIR, "ext/ZeroTierOne/osdep"),
```

If the compiler complains about `_PyTime_AsTimeval_noraise` nont existing:

```diff
diff --git a/src/bindings/python/PythonSockets.cxx b/src/bindings/python/PythonSockets.cxx
index f078284..54b2cdb 100644
--- a/src/bindings/python/PythonSockets.cxx
+++ b/src/bindings/python/PythonSockets.cxx
@@ -479,7 +479,7 @@ PyObject* zts_py_select(PyObject* module, PyObject* rlist, PyObject* wlist, PyOb
                 n = 0;
                 break;
             }
-            _PyTime_AsTimeval_noraise(timeout, &tv, _PyTime_ROUND_CEILING);
+            _PyTime_AsTimeval_clamp(timeout, &tv, _PyTime_ROUND_CEILING);
             /* retry select() with the recomputed timeout */
         }
     } while (1);
```

(I have to assume this is a private method because of the leading underscore.
[It was also changed quite a while ago.](https://github.com/python/cpython/pull/28629))

## Done!
There should be a wheel in `pkg/pypi/dist`.
Install it.

`cd` somewhere else to make sure there is not a directory called `libzt` in your current working directory,
so that Python does not get confused when you `import libzt`.

It should work now.