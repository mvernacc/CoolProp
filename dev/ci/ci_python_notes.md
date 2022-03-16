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

Solution: run `cibuildwheel` from repo root directory and use `package_dir` argument [[docs](https://cibuildwheel.readthedocs.io/en/stable/options/#command-line)].


### generate_constants_module import error
When `cibuildwheel` executes `python -m pip wheel /project/wrappers/Python --wheel-dir=/tmp/cibuildwheel/built_wheel --no-deps`, it fails with this error:

```
File "setup.py", line 293, in <module>
      import generate_constants_module
  ModuleNotFoundError: No module named 'generate_constants_module'
```

`generate_constants_module.py` is in `wrappers/Python/`, alongside `setup.py`. Its role is to parse the `DataStructures.h` and `Configuration.h` headers and generate Cython files (`.pxd`, `.pyx`) for them. It is used in two places:

1. Imported and called in `setup.py` (cause of the above error), to generate the cython files, before `setup.py` calls `cythonize`.
2. Called with `subprocess ` in ` prepare_pypi.py`. `prepare_pypi.py`, in turn, is invoked by cmake if the `COOLPROP_PYTHON_PYPI` flag is set. The only place this flag is mentioned is in the `python_source_slave` in the old buildbot's `master.cfg`.


#### Attempted solution with PYTHONPATH
Adding `wrappers/Python/` to the `PYTHONPATH` should enable the failing import. I tried adding the following to `pyproject.toml`:

```toml
[tool.cibuildwheel]
...
before-build = "echo $PYTHONPATH"

[tool.cibuildwheel.environment]
PYTHONPATH = "/project/wrappers/Python:$PYTHONPATH"
```

And I added the following to `setup.py`, just before the failing `import generate_constants_module`:

```python
print("PYTHONPATH: " + os.environ.get("PYTHONPATH"))
```

The import error still occurs! The inspections of `PYTHONPATH` give the following output:
```
Running before_build...

    + sh -c 'echo $PYTHONPATH'
/project/wrappers/Python:

Building wheel...
[...]
    + python -m pip wheel /project/wrappers/Python --wheel-dir=/tmp/cibuildwheel/built_wheel --no-deps
[...]
  PYTHONPATH: /tmp/pip-build-env-y0wnharz/site
[...]
  ModuleNotFoundError: No module named 'generate_constants_module'
```

It seems `cibuildwheel` runs the `pip wheel` command in a different environment, that does not have the same `PYTHONPATH` as configured by `[tool.cibuildwheel.environment]`?

#### Attempted solution: consolidating setup.py
The relative import in `setup.py` is atypical and is causing problems. Can we get rid of it?

It seems `generate_constants_module` currently separate so that it can also be called when making a source distribution for PyPI.

Miscellaneous articles on how to distribute source for cython projects:
 - https://stackoverflow.com/questions/4505747/how-should-i-structure-a-python-package-that-contains-cython-code
 - [Distributing Cython modules](http://docs.cython.org/en/latest/src/userguide/source_files_and_compilation.html#distributing-cython-modules)

"Distributing Cython modules" indicates standard practice is to do this all with options within setup.py, and not use another script (but id does not address auto-generated .pxd files).

TODO


## Trying to learn what jmarrec did

Julien Marrec submitted a [PR](https://github.com/CoolProp/CoolProp/pull/2097) to implement cibuildwheels for CoolProp. I want to learn what he did differently to avoid the pitfalls I've been having.

I needed to set the `cmake_compiler` or `cmake_bitness` variables in `setup.py` to make `USING_CMAKE` true, in order for cmake to be called. Julien did this by modifying `setup.py` to read these from environment variables. For now I'm just hardcoding them in.

### Error with `find_module`

cmake calls a python 2.7 script, `dev/generate_headers.py`, which parses some JSON files and generated headers for the C++ library. In Julien's branch, this runs properly, producing the output:
```
  /usr/bin/python2.7 /project/dev/generate_headers.py
  git version 2.34.1
  version written to file: /project/include/cpversion.h
  version written to hidden file: /project/.version for use in builders that don't use git repo
  git is accessible at the command line
  git revision is 6bec2d9ec7322ecc0c3cc75ac5fc156e4ce3bff0
  *** Generating gitrevision.h ***
  /project/include/gitrevision.h written to file
  /project/include/all_fluids_JSON.h written to file
  /project/include/all_incompressibles_JSON.h written to file
  /project/include/mixture_departure_functions_JSON.h written to file
  /project/include/mixture_binary_pairs_JSON.h written to file
  /project/include/predefined_mixtures_JSON.h written to file
  /project/include/all_cubics_JSON.h written to file
  /project/include/cubic_fluids_schema_JSON.h written to file
  /project/include/pcsaft_fluids_schema_JSON.h written to file
  /project/include/all_pcsaft_JSON.h written to file
  /project/include/mixture_binary_pairs_pcsaft_JSON.h written to file
```

On my branch, this step in cmake fails:
```
  /usr/bin/python2.7 /project/dev/generate_headers.py
  AttributeError: find_module
  gmake[2]: *** [CMakeFiles/generate_headers] Error 1
```
`find_module` is part of the `imp` [import internals](https://docs.python.org/2.7/library/imp.html#imp.find_module) module in python 2.7. It never appears in the CoolProp source.

I noticed that the pip command which triggers cmake is different on my branch vs on Julien's.

Mine:
```
Running command /opt/python/cp39-cp39/bin/python /opt/python/cp39-cp39/lib/python3.9/site-packages/pip/_vendor/pep517/in_process/_in_process.py get_requires_for_build_wheel /tmp/tmppi_nuiqu
```

Julien's:
```
Running command python setup.py egg_info
```

This is because my branch has a pyproject.toml file and Julien's does not. I removed my pyproject.toml file (and instead gave the same options for cibuildwheel via env vars). This caused the `find_module` error to not happen on my branch. I have no idea why.

Without pyproject.toml, cibuildwheels succeeds in building the wheel for cp39-manylinux_x86_64 on my branch.


#### Why does `find_module` error occur when pyproject.toml is present?

As an experiment to learn more about the error, I tried a trivial `generate_headers.py`.
On my branch, with pyproject.toml, I replaced the contents of `generate_headers.py` with `print "hello"`. I then ran `cibuildwheel --platform linux ./wrappers/Python/`. The command:
```
 Running command /opt/python/cp39-cp39/bin/python /tmp/pip-standalone-pip-b3tm1dq1/__env_pip__.zip/pip install --ignore-installed --no-user --prefix /tmp/pip-build-env-x1d6qlpl/overlay --no-warn-script-location --no-binary :none: --only-binary :none: -i https://pypi.org/simple -- setuptools cython
 ```
triggered cmake, and as before cmake failed with
```
  /usr/bin/python2.7 /project/dev/generate_headers.py
  AttributeError: find_module
  gmake[2]: *** [CMakeFiles/generate_headers] Error 1
```

This indicates that the error occurs from cmake trying to find or load `generate_headers.py`, not from executing an import statement within `generate_headers.py`.

#### Why is python 2.7 used?
In CMakeLists.txt, cmake is instructed to look for a python 2.7 interpreter, and then fall back on python3 if it cannot find python2:
```
set(Python_ADDITIONAL_VERSIONS 2.7 2.6 2.5 2.4)
find_package (PythonInterp 2.7)
if (NOT PYTHON_EXECUTABLE)
  MESSAGE(STATUS "Looking for Python")
  find_package (Python COMPONENTS Interpreter)
endif()
if (NOT PYTHON_EXECUTABLE)
  MESSAGE(STATUS "Looking for Python2")
  find_package (Python2 COMPONENTS Interpreter)
  if(Python2_Interpreter_FOUND)
    set(PYTHON_EXECUTABLE ${Python2_EXECUTABLE})
  endif()
endif()
if (NOT PYTHON_EXECUTABLE)
  MESSAGE(STATUS "Looking for Python3")
  find_package (Python3 COMPONENTS Interpreter)
  if(Python3_Interpreter_FOUND)
    set(PYTHON_EXECUTABLE ${Python3_EXECUTABLE})
  endif()
endif()
if (NOT PYTHON_EXECUTABLE)
  MESSAGE(WARNING "Could not find Python, be prepared for errors.")
endif()
```

On both my branch and Julien's branch, cmake finds the system python 2.7 of the manylinux2014 image:
```
-- Found PythonInterp: /usr/bin/python2.7 (Required is at least version "2.7")
```
