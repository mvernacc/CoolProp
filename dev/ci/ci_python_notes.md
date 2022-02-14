Notes on building and distributing CoolProp's python wrapper via CI
===================================================================

These are [Matt Vernacchia](https://github.com/mvernacc)'s personal notes while deciding how to upgrade CoolProp's Ci system in February 2022.


Basic intro to python wheels: https://realpython.com/python-wheels/

Explanation of `manylinux` platform tag: https://opensource.com/article/19/2/manylinux-python-wheels

cibuildwheel, a potential tool: https://github.com/pypa/cibuildwheel



## Python build system, PEP 517
CoolProp currently only uses `setup.py`. This is outdated, and cibuildwheel expects the project to have a `pyproject.toml` file.

I assume we want to keep using `setuptools` and `pip`, and not switch to poetry or something.

The new PEP 517 way of doing things with `setuptools` is to have a `pyproject.toml` file, and additionally either a `setup.cfg` or `setup.py`.
`setup.cfg` is preferred, but does not yet support C Extensions (see [this SO question](https://stackoverflow.com/questions/66157987/how-to-build-a-c-extension-in-keeping-with-pep-517-i-e-with-pyproject-toml-ins) and [setuptools issue 2220](https://github.com/pypa/setuptools/issues/2220)).

Thus, CoolProp should add a `pyproject.toml` and keep its `setup.py` file.

## Attempt to use cibuildwheel
On 2022-02-14, I am trying out cibuildwheel by attempting a local build of the cp39-manylinux_x86_64 config.

### project directory error

The command `cibuildwheel --platform linux` fails with the following error:

```
 File "setup.py", line 283, in <module>
      raise ValueError('Could not run script from this folder(' + os.path.abspath(os.path.curdir) + '). Run from wrappers/Python folder')
  ValueError: Could not run script from this folder(/project). Run from wrappers/Python folder
```

I believe this happens because `cibuildwheel` only copies the python project directory (where `pyproject.toml` is, in this case `CoolProp/wrappers/Python/`) into its Docker container. This works for the typical python directory structure, but the build for CoolProp requires stuff in parent directories: `CoolProp/CMakeLists.txt`, C++ source in `CoolProp/src/`, other C++ libraries in `CoolProp/externals`.

[cibuildwheels PR 295](https://github.com/pypa/cibuildwheel/pull/295) seems related, but I have not yet read it in detail.
