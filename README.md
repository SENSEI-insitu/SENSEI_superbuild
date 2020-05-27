# SENSEI_superbuild
A helper script to automate build and install.

## Overview
The following will create an install in the directory next to the clone.
By default the name will have the following pattern: `sensei-<branch>-<backend>`
where branch (defaults to develop) is set using `-DSENSEI_BRANCH` and backend
(defaults to python) is set using `-DSENSEI_BACKEND` on the CMake command line.

The available backends are:
1. python (done)
2. catalyst (todo)
3. libsim (todo)
4. ascent (todo)
5. vtk (todo)
6. vtk-m (todo)

In transit capabilities can be configured by adding `-DENABLE_ADIOS2=ON` (todo) or
`-DENABLE_HDF5=ON` (todo).

The install will be self contained and include an environment module in
`sensei-<branch>-<backend>/modulefiles`. One can access this via
```
module use /path/to/sensei-<branch>-<backend>/modulefiles`
module load sensei/<branch>-<backend>
```

This will put SENSEI and all it dependencies into the environment. This includes
Python. One may then add to the Python install using `pip3`. Note that it may
be neccessary to uncomment the line that sets `PYTHONHOME` in the environment
module but doing so will interfere with other tools/scripts that are cavalier
about not specifying specifically which python verison they are using. For example
invoking `python` results in `python2` where as our install is `python3`.

For those using CMake SENSEI's CMake exports may be found in the library dir.
For those using autotools or Makefile based builds one can use `sensei_config`
to get include dirs and flags via `--cflags` and link dependencies and paths
via `--libs`. Note that `source sensei_config` will set environment variables
with these making it easier to automate Makefile based builds.

## Installs
It should be notes that CMake remembers bad values. In the case of a failed
build it is often best to `rm -rf` the build directory and start over. Additionally
the incremental install process makes use of the libraries as they are compiled,
hence it is also often best to `rm -rf` the install directory as well.

### NERSC Cori
#### Python backend
```
module switch PrgEnv-intel/6.0.5 PrgEnv-gnu
module load cmake/3.14.4
export CRAYPE_LINK_TYPE=dynamic

git clone https://github.com/SENSEI-insitu/SENSEI_superbuild.git

cd SENSEI_superbuild
mkdir build && cd build

cmake -DENABLE_READLINE=ON -DENABLE_MPICH=OFF -DENABLE_CRAY_MPICH=ON ../

make -j32
make -j32 install
```
See `Overview` section above for notes on how to use the install..


