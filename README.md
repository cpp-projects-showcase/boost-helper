Helper Documentation for Boost
==============================

# Overview
[Boost helper documentation](https://github.com/cpp-projects-showcase/boost-helper)
aims at supporting local, _ad hoc_, Boost installations on various platforms.

As a matter of fact, on many platforms (like _e.g._, CentOS, Fedora, Ubuntu,
Debian, MacOS), the native Boost packages often lag behind the latest
[releases of Boost](https://www.boost.org/users/history/).

Moreover, on those platforms, it often happens that Boost has been built
with outdated Python versions. For instance, on Ubuntu 20.04 LTS,
(the latest long-term support release as of November 2021), the native
Boost package comes in version 1.71.0 (lagging over two years behind
the newest Boost releases) with support for Python 3.8.10; whereas
on the same platform, Python is available as 3.8.12 (the latest stable
version of the 3.8 branch of Python as of November 2021).
Hence, when one attempts to build some software on those platforms,
they often end up with inconsistencies in Python versions.

This README therefore intends to explain how to build one's own version
of Boost, and use them to build other packages.

## References
* [Boost](https://www.boost.org)
  + [Boost releases](https://www.boost.org/users/history/)
  + [Boost source tar-balls](https://dl.bintray.com/boostorg/release/)
  + [Boost from sources on MacOS](https://blog.the-pans.com/boost-darwin/)
  + [Boost from sources on Ubuntu](https://stackoverflow.com/a/24086375/798053)
* [OpenTREP](https://github.com/trep/opentrep),
  a piece of software making extensive use of Boost,
  including its Python libraries
  (hence a good use case to make sure everything works correctly)
* Python
  + [Python virtual environment with `pyenv` and `pipenv`](http://github.com/machine-learning-helpers/induction-python/tree/master/installation/virtual-env)

# Installation

## Python

* Install, with `pyenv`, the same version as installed by the native
  package manager of your environment. For instance, as of November 2021:
  + 3.9.8 on MacOS:
```bash
user@mac$ brew install python@3.9
user@mac$ $ ls -lFh /usr/local/Cellar/python\@3.9/3.9.8/Frameworks/Python.framework/Versions/3.9/bin/python3.9
-rwxr-xr-x  1 user  staff 17K Mar 12 11:34 /usr/local/Cellar/python@3.9/3.9.8/Frameworks/Python.framework/Versions/3.9/bin/python3.9*
user@mac$ pyenv install 3.9.8
user@mac$ pyenv global 3.9.8 && pip install -U pip pipenv
user@mac$ python --version
Python 3.9.8
```
  + 3.10 on Fedora 35:
```bash
user@f31$ sudo install -y python
user@f31$ rpm -ql python|grep "bin/python3.10$"
/usr/bin/python3.10
user@f31$ python3.10 --version
Python 3.10.0
```

## Boost

* Install dependencies
  + On Ubuntu:
```bash
$ sudo apt-get update
$ sudo apt-get install build-essential g++ python-dev autotools-dev libicu-dev libbz2-dev libboost-all-dev
```

* Prepare the build and install directories:
```bash
$ BOOST_VER="1_77_0"
$ BOOST_VER_DOT="$(echo ${BOOST_VER}|sed -e s/_/./g)"
$ sudo mkdir -p /opt/boost/archives && sudo chown -R ${USER} /opt/boost
$ cd /opt/boost
$ wget "https://boostorg.jfrog.io/artifactory/main/release/${BOOST_VER_DOT}/source/boost_${BOOST_VER}.tar.bz2"
$ tar jxf "boost_${BOOST_VER}.tar.bz2" && mv "boost_${BOOST_VER}.tar.bz2" archives
```

* (Optionally, if needed, _e.g._, on versions up to 1.71,)
  Inject the new `boost/timer/progress_display.hpp`
  header file:
```bash
$ cd /opt/boost/boost_${BOOST_VER}
$ wget https://raw.githubusercontent.com/boostorg/timer/c221a60102ed618a82116f84407de3f1713aa10f/include/boost/timer/progress_display.hpp -O boost/timer/progress_display.hpp
```

* Initialize the build environment
  + For a standard installation in `/usr/local`:
```bash
$ cd /opt/boost/boost_${BOOST_VER}
$ ./bootstrap.sh
```
  + To override potentially installed Boost components in `/usr` (for instance
    for
	[Travis CI virtual machines](https://docs.travis-ci.com/user/reference/linux/)):
```bash
$ sudo apt-get remove libboost-all-dev
$ cd /opt/boost/boost_${BOOST_VER}
$ ./bootstrap.sh --prefix=/usr/
```

* If needed, specify the Python location in the `project-config.jam` file,
  which may look like the following
  + On Ubuntu:
```yaml
import python ;
if ! [ python.configured ]
{
    using python : 3.8 : "/usr" ;
}
```
  + On MacOS:
```yaml
import python ;
if ! [ python.configured ]
{
    using python
        : 3.9     # Version
        : "/usr/local/Cellar/python@3.9/3.9.8/Frameworks/Python.framework/Versions/3.9" # Python path
        : "/usr/local/Cellar/python@3.9/3.9.8/Frameworks/Python.framework/Versions/3.9/include/python3.9" # include path
        : "/usr/local/Cellar/python@3.9/3.9.8/Frameworks/Python.framework/Versions/3.9/lib/python3.9" # lib path
        ;
}
```

* Build Boost (it will take a few minutes, up to an hour,
  even on some modern computers, and it is CPU intensive):
```bash
$ ./b2 cxxflags="-std=c++14"
...
common.copy /opt/boost/boost_${BOOST_VER}/stage/lib/libboost_wave.a
common.copy /opt/boost/boost_${BOOST_VER}/stage/lib/cmake/boost_wave-1.72.0/libboost_wave-variant-static.cmake
clang-darwin.compile.c++ bin.v2/libs/graph/build/clang-darwin-11.0/release/threading-multi/visibility-hidden/graphml.o
clang-darwin.compile.c++ bin.v2/libs/graph/build/clang-darwin-11.0/release/threading-multi/visibility-hidden/read_graphviz_new.o
clang-darwin.link.dll bin.v2/libs/graph/build/clang-darwin-11.0/release/threading-multi/visibility-hidden/libboost_graph.dylib
boost-install.generate-cmake-variant- bin.v2/libs/graph/build/clang-darwin-11.0/release/threading-multi/visibility-hidden/libboost_graph-variant-shared.cmake
common.copy /opt/boost/boost_${BOOST_VER}/stage/lib/libboost_graph.dylib
common.copy /opt/boost/boost_${BOOST_VER}/stage/lib/cmake/boost_graph-${BOOST_VER_DOT}/libboost_graph-variant-shared.cmake
...updated 1681 targets...

The Boost C++ Libraries were successfully built!
```

* Install Boost:
```bash
$ sudo ./b2 cxxflags="-std=c++14" install
boost-install.generate-cmake-config- bin.v2/libs/graph/build/install/boost_graph-config.cmake
boost-install.generate-cmake-config-version- bin.v2/libs/graph/build/install/boost_graph-config-version.cmake
common.copy /usr/local/lib/cmake/boost_graph-${BOOST_VER_DOT}/boost_graph-config-version.cmake
common.copy /usr/local/lib/cmake/boost_graph-${BOOST_VER_DOT}/boost_graph-config.cmake
...updated 14520 targets...
```

* Check the installation
  + On Ubuntu:
```bash
user@mac$ grep "define BOOST_VERSION " /usr/local/include/boost/version.hpp
#define BOOST_VERSION 107700
user@mac$ ldd -r -d /usr/local/lib/libboost_python38.so | grep -v "^undefined"
	linux-vdso.so.1 (0x00007ffddacd5000)
	libstdc++.so.6 => /usr/lib/x86_64-linux-gnu/libstdc++.so.6 (0x00007f9f03dc5000)
	libgcc_s.so.1 => /lib/x86_64-linux-gnu/libgcc_s.so.1 (0x00007f9f03daa000)
	libpthread.so.0 => /lib/x86_64-linux-gnu/libpthread.so.0 (0x00007f9f03d87000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f9f03b95000)
	libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007f9f03a46000)
	/lib64/ld-linux-x86-64.so.2 (0x00007f9f03ffa000)
```
  + On MacOS:
```bash
user@mac$ grep "define BOOST_VERSION " /usr/local/include/boost/version.hpp
#define BOOST_VERSION 107700
user@mac$ otool -L /usr/local/lib/libboost_python39.dylib
/usr/local/lib/libboost_python39.dylib:
        @rpath/libboost_python39.dylib (compatibility version 0.0.0, current version 0.0.0)
        /usr/lib/libc++.1.dylib (compatibility version 1.0.0, current version 800.6.0)
        /usr/lib/libSystem.B.dylib (compatibility version 1.0.0, current version 1281.0.0)
```



