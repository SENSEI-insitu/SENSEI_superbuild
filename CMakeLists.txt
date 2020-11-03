cmake_minimum_required(VERSION 3.0)

project(SENSEI_superbuild)
include(ExternalProject)

# where various resources are housed and served
set(SENSEI_URL "https://gitlab.kitware.com/sensei/sensei"
    CACHE STRING "URL to the SENSEI source code repository")

set(SENSEI_DEPS_URL "http://missmarple.lbl.gov/SENSEI/SENSEI_deps"
    CACHE STRING "URL to the dependency tarballs")

set(SENSEI_BRANCH develop CACHE STRING
    "branch/tag/sha to checkout from the SENSEI git repository")

set(SENSEI_BACKEND "python" CACHE STRING
    "The name of the back-end to build with")
set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS
    "python" "catalyst" "acsent" "libsim" "vtk" "vtkm")


# The following versions of each library are installed by this script:
set(MPICH_SOURCE mpich-3.3.1)
set(OPENMPI_SOURCE openmpi-4.0.1)
set(OPENSSL_SOURCE openssl-1.1.1d)
set(ZLIB_SOURCE zlib-1.2.11)
set(NCURSES_SOURCE ncurses-6.1)
set(READLINE_SOURCE readline-8.0)
set(PYTHON_SOURCE Python-3.7.4)
set(SETUPTOOLS_SOURCE setuptools-41.2.0)
set(SWIG_SOURCE swig-4.0.1)
set(CYTHON_SOURCE Cython-0.29.13)
set(NUMPY_SOURCE numpy-1.17.2)
set(MPI4PY_SOURCE mpi4py-3.0.2)
set(FREETYPE_SOURCE freetype-2.10.1)
set(LIBPNG_SOURCE libpng-1.6.37)
set(PYTHON_DATEUTIL_SOURCE python-dateutil-2.8.0)
set(PYPARSING_SOURCE pyparsing-2.4.2)
set(CYCLER_SOURCE cycler-0.10.0)
set(PYTZ_SOURCE pytz-2019.2)
set(KIWISOLVER_SOURCE kiwisolver-1.1.0)
set(MATPLOTLIB_SOURCE matplotlib-3.1.1)
set(VTK_SOURCE VTK-8.2.0)

if (SENSEI_BACKEND STREQUAL python)
    option(ENABLE_MPICH "builds and installs MPICH MPI" ON)
    option(ENABLE_OPENMPI "builds and installs OpenMPI MPI" OFF)
    option(ENABLE_CRAY_MPICH "Auto detect MPICH configuration for Cray systems" OFF)
    option(ENABLE_CORI_GPU "Auto detect OpenMPI configuration for Cori GPU nodes" OFF)
    option(WITHOUT_MPI "Disable all dependence on MPI" OFF)
    option(ENABLE_PYTHON "builds and installs Python group" ON)
    option(ENABLE_READLINE "builds and installs readline" OFF)
    option(ENABLE_MATPLOTLIB "builds and installs matplotlib group" ON)
    option(ENABLE_SENSEI "builds and installs SENSEI" ON)
    option(ENABLE_SENSEI_TEST "runs the SENSEI regression tests" OFF)
    option(ENABLE_OPENSSL "builds and installs OpenSSL" OFF)
    option(ENABLE_VTK "builds and installs VTK" ON)
    option(ENABLE_MEMCHECK "configures for use with valgrind" OFF)
else()
    message(FATAL_ERROR "Invalid SENSEI_BACKEND ${SENSEI_BACKEND}")
endif()

# the list of enabled dependencies
set(sensei_deps)
set(sensei_flags)

message(STATUS "C compiler is ${CMAKE_C_COMPILER} (${CMAKE_C_COMPILER_ID})")
message(STATUS "C++ compiler is ${CMAKE_CXX_COMPILER} (${CMAKE_CXX_COMPILER_ID})")

# We need to override the default value of CMAKE_INSTALL_PREFIX.
if (CMAKE_INSTALL_PREFIX STREQUAL "/usr/local")
  set(CMAKE_INSTALL_PREFIX ${PROJECT_SOURCE_DIR}/../sensei-${SENSEI_BRANCH}-${SENSEI_BACKEND}
     CACHE PATH "" FORCE)
endif()
message(STATUS "Installing libraries in ${CMAKE_INSTALL_PREFIX}")

# if we forgot to set the build type default to release
if (NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE "Release"
      CACHE STRING "options are: Debug Release RelWithDebInfo MinSizeRel."
      FORCE)
  set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS
      "Debug" "Release" "MinSizeRel" "RelWithDebInfo")
endif()
message(STATUS "Configuring a ${CMAKE_BUILD_TYPE} build")

# Sniff out the platform option and set compiler flags.
# default to native, which gives the best performance
# but is not portable. use generic for max protability.
# sandybridge should be very portible, covering processors
# released 2011 or later.
set (TARGET_PLATFORM "native" CACHE STRING
    "Select from a set of pre-defined compiler flags")
set_property(CACHE TARGET_PLATFORM PROPERTY STRINGS
    "generic" "native" "sandybridge")

if (TARGET_PLATFORM STREQUAL "generic")
  # go with the defaults specified by the build type
elseif(TARGET_PLATFORM STREQUAL "knl")
    if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
        set(CMAKE_C_FLAGS_RELEASE "-O3 -march=knl -mtune=knl -DNDEBUG"
            CACHE STRING "" FORCE)
        set(CMAKE_CXX_FLAGS_RELEASE "-O3 -march=knl -mtune=knl -DNDEBUG"
            CACHE STRING "" FORCE)
    elseif(CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
        message(SEND_ERROR "No knl support with clang")
    endif()
elseif(TARGET_PLATFORM STREQUAL "native")
    if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
        set(CMAKE_C_FLAGS_RELEASE "-O3 -march=native -mtune=native -DNDEBUG"
            CACHE STRING "" FORCE)
        set(CMAKE_CXX_FLAGS_RELEASE "-O3 -march=native -mtune=native -DNDEBUG"
            CACHE STRING "" FORCE)
    elseif(CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
        # Clang's offerings are pretty spare at the moment (maybe?).
        # go with the defaults
    endif()
elseif(TARGET_PLATFORM STREQUAL "sandybridge")
  if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
    set(CMAKE_C_FLAGS_RELEASE "-O3 -march=sandybridge -mtune=sandybridge -mavx -DNDEBUG"
        CACHE STRING "" FORCE)
    set(CMAKE_CXX_FLAGS_RELEASE "-O3 -march=sandybridge -mtune=sandybridge -mavx -DNDEBUG"
        CACHE STRING "" FORCE)
  elseif(CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
    # Clang's offerings are pretty spare at the moment (maybe?).
    # go with the defaults
  endif()
else()
    message(FATAL_ERROR "Unknown platform: ${TARGET_PLATFORM}")
endif()
message(STATUS "Configuring for ${TARGET_PLATFORM} platform")

# collect flags to pass them into autotools scripts
set(cpp_flags "$ENV{CPPFLAGS} -I${CMAKE_INSTALL_PREFIX}/include")
if (ENABLE_MEMCHECK)
    string(CONCAT cpp_flags "${cpp_flags} -DPy_USING_MEMORY_DEBUGGER")
endif()
message(STATUS "CPPFLAGS=${cpp_flags}")

set(c_flags "-I${CMAKE_INSTALL_PREFIX}/include")
set(cxx_flags "-I${CMAKE_INSTALL_PREFIX}/include")
if (CMAKE_BUILD_TYPE STREQUAL "Release")
    string(CONCAT c_flags "${c_flags} ${CMAKE_C_FLAGS} ${CMAKE_C_FLAGS_RELEASE}")
    string(CONCAT cxx_flags "${cxx_flags} ${CMAKE_CXX_FLAGS} ${CMAKE_CXX_FLAGS_RELEASE}")
elseif(CMAKE_BUILD_TYPE STREQUAL "Debug")
    string(CONCAT c_flags "${c_flags} ${CMAKE_C_FLAGS} ${CMAKE_C_FLAGS_DEBUG}")
    string(CONCAT cxx_flags "${cxx_flags} ${CMAKE_CXX_FLAGS} ${CMAKE_CXX_FLAGS_DEBUG}")
else()
    message(FATAL_ERROR "Unknown build type: ${CMAKE_BUILD_TYPE}")
endif()
message(STATUS "CFLAGS=${c_flags}")
message(STATUS "CXXFLAGS=${cxx_flags}")

set(ld_flags "-L/${CMAKE_INSTALL_PREFIX}/lib -L${CMAKE_INSTALL_PREFIX}/lib64")
message(STATUS "LDFLAGS=${ld_flags}")

# collect MPI falgs passed on the command line, usually only for
# Cray builds
set(mpi_flags)
if (NOT WITHOUT_MPI AND NOT ENABLE_MPICH AND NOT ENABLE_OPENMPI)
    if (ENABLE_CRAY_MPICH)
        # use pkg-config to detect MPI dependencies
        set(ENV{PKG_CONFIG_PATH} "$ENV{CRAY_MPICH_DIR}/lib/pkgconfig:$ENV{PKG_CONFIG_PATH}")
        find_package(PkgConfig QUIET)
        pkg_check_modules(CRAY_MPICH REQUIRED QUIET mpich)
        set(MPI_C_INCLUDE_PATH ${CRAY_MPICH_INCLUDE_DIRS} CACHE STRING "MPI include directories")
        set(MPI_C_LIBRARIES ${CRAY_MPICH_LDFLAGS} CACHE STRING "MPI link dependencies")
        set(MPIEXEC srun CACHE STRING "Platform MPI run equivalent")
        set(MPI_C_FOUND CACHE BOOL ON "status of MPI config")
    elseif (ENABLE_CORI_GPU)
        # use pkg-config to detect MPI dependencies
        set(ENV{PKG_CONFIG_PATH} "$ENV{OPENMPI_DIR}/lib/pkgconfig:$ENV{PKG_CONFIG_PATH}")
        find_package(PkgConfig QUIET)
        pkg_check_modules(CORI_GPU REQUIRED QUIET ompi)
        set(MPI_C_INCLUDE_PATH ${CORI_GPU_INCLUDE_DIRS} CACHE STRING "MPI include directories")
        set(MPI_C_LIBRARIES ${CORI_GPU_LDFLAGS} CACHE STRING "MPI link dependencies")
        set(MPIEXEC srun CACHE STRING "Platform MPI run equivalent")
        set(MPI_C_FOUND CACHE BOOL ON "status of MPI config")
    endif()
    list(APPEND mpi_flags "-DMPI_CXX_COMPILER=${MPI_CXX_COMPILER}")
    list(APPEND mpi_flags "-DMPI_C_COMPILER=${MPI_C_COMPILER}")
    list(APPEND mpi_flags "-DMPI_CXX_LIBRARIES=")
    string(REPLACE ";" " " tmp "${MPI_C_LIBRARIES}")
    list(APPEND mpi_flags "-DMPI_C_LIBRARIES=${tmp}")
    list(APPEND mpi_flags "-DMPI_INCLUDE_PATH=${MPI_INCLUDE_PATH}")
    list(APPEND mpi_flags "-DMPI_EXTRA_LIBRARY=")
    list(APPEND mpi_flags "-DMPIEXEC=${MPIEXEC}")
endif()
set(MPI_FLAGS ${mpi_flags} CACHE STRING "MPI configuration overides")
message(STATUS "MPI_FLAGS=${MPI_FLAGS}")

# flags for testing
set(CTEST_FLAGS  --output-on-failure CACHE STRING
    "flags passed to ctest during tests after install")

# set the build enviornment for autotools builds and Python builds
# this ensures that python-config works as expected and prevents
# external Python installs from mixing in. this is also the environment
# that should be used at run-time.
set(py_env)
if (ENABLE_PYTHON)
    set(py_env PYTHONHOME=${CMAKE_INSTALL_PREFIX} MPLBACKEND=Agg
        PYTHONPATH=${CMAKE_INSTALL_PREFIX}/lib:${CMAKE_INSTALL_PREFIX}/lib/python3.7/site-packages/:${CMAKE_INSTALL_PREFIX}/lib64/python3.7/site-packages)
endif()
set(env CC=${CMAKE_C_COMPILER} CXX=${CMAKE_CXX_COMPILER}
    "CPPFLAGS=${cpp_flags}" "CFLAGS=${c_flags}" "CXXFLAGS=${cxx_flags}" "LDFLAGS=${ld_flags}"
    LD_LIBRARY_PATH=${CMAKE_INSTALL_PREFIX}/lib/:${CMAKE_INSTALL_PREFIX}/lib64/:$ENV{LD_LIBRARY_PATH}
    DYLD_LIBRARY_PATH=${CMAKE_INSTALL_PREFIX}/lib/:${CMAKE_INSTALL_PREFIX}/lib64/:$ENV{DYLD_LIBRARY_PATH}
    PKG_CONFIG_PATH=${CMAKE_INSTALL_PREFIX}/lib/pkgconfig:${CMAKE_INSTALL_PREFIX}/lib64/pkgconfig:$ENV{PKG_CONFIG_PATH}
    PATH=.:${CMAKE_INSTALL_PREFIX}/bin/:$ENV{PATH} ${py_env})

# configure a bash script that will let us reproduce
# the build enviornment at run-time
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/sensei_env.sh.in
    ${CMAKE_CURRENT_BINARY_DIR}/sensei_env.sh @ONLY)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/sensei_env.sh
    DESTINATION ${CMAKE_INSTALL_PREFIX}/bin)

# configure the environment module that will let us reproduce
# the build enviornment at run-time
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/modulefiles/${SENSEI_BACKEND}.in
    ${CMAKE_CURRENT_BINARY_DIR}/modulefiles/${SENSEI_BRANCH}_${SENSEI_BACKEND} @ONLY)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/modulefiles/${SENSEI_BRANCH}_${SENSEI_BACKEND}
    DESTINATION ${CMAKE_INSTALL_PREFIX}/modulefiles/sensei)

# Build MPICH if MPI is requested.
if (NOT WITHOUT_MPI AND ENABLE_CRAY_MPICH)
    message(STATUS "MPICH .................. Disabled")
    message(STATUS "Cray MPICH ............. Enabled")
    message(STATUS "Cori GPU ............... Disabled")
    message(STATUS "OpenMPI ................ Disabled")
    list(APPEND sensei_flags "-DENABLE_CRAY_MPICH=ON")
    list(APPEND sensei_flags "-DREQUIRE_MPI=TRUE")
elseif (NOT WITHOUT_MPI AND ENABLE_CORI_GPU)
    message(STATUS "MPICH .................. Disabled")
    message(STATUS "Cray MPICH ............. Disabled")
    message(STATUS "Cori GPU ............... Enabled")
    message(STATUS "OpenMPI ................ Disabled")
    list(APPEND sensei_flags "-DENABLE_CORI_GPU=ON")
    list(APPEND sensei_flags "-DREQUIRE_MPI=TRUE")
elseif (NOT WITHOUT_MPI AND ENABLE_MPICH)
    message(STATUS "MPICH .................. Enabled")
    message(STATUS "Cray MPICH ............. Disabled")
    message(STATUS "Cori GPU ............... Disabled")
    message(STATUS "OpenMPI ................ Disabled")
    list(APPEND sensei_flags "-DREQUIRE_MPI=TRUE")
    ExternalProject_Add(mpi
        DOWNLOAD_DIR ${PROJECT_BINARY_DIR}
        URL ${SENSEI_DEPS_URL}/${MPICH_SOURCE}.tar.gz
        SOURCE_DIR ${PROJECT_BINARY_DIR}/${MPICH_SOURCE}
        CONFIGURE_COMMAND env ${env} ${PROJECT_BINARY_DIR}/${MPICH_SOURCE}/configure
            --prefix=${CMAKE_INSTALL_PREFIX} --disable-maintainer-mode --enable-threads=multiple
        BUILD_IN_SOURCE 1 LOG_DOWNLOAD 1 LOG_CONFIGURE 1 LOG_BUILD 1 LOG_INSTALL 1)
elseif (NOT WITHOUT_MPI AND ENABLE_OPENMPI)
    message(STATUS "MPICH .................. Disabled")
    message(STATUS "Cray MPICH ............. Disabled")
    message(STATUS "Cori GPU ............... Disabled")
    message(STATUS "OpenMPI ................ Enabled")
    list(APPEND sensei_flags "-DREQUIRE_MPI=TRUE")
    ExternalProject_Add(mpi
        DOWNLOAD_DIR ${PROJECT_BINARY_DIR}
        URL ${SENSEI_DEPS_URL}/${OPENMPI_SOURCE}.tar.gz
        SOURCE_DIR ${PROJECT_BINARY_DIR}/${OPENMPI_SOURCE}
        CONFIGURE_COMMAND env ${env} ${PROJECT_BINARY_DIR}/${OPENMPI_SOURCE}/configure
            --prefix=${CMAKE_INSTALL_PREFIX} --enable-mpi-thread-multiple
        BUILD_IN_SOURCE 1 LOG_DOWNLOAD 1 LOG_CONFIGURE 1 LOG_BUILD 1 LOG_INSTALL 1)
else()
    message(STATUS "MPICH .................. Disabled")
    message(STATUS "Cray MPICH ............. Disabled")
    message(STATUS "Cori GPU ............... Disabled")
    message(STATUS "OpenMPI ................ Disabled")
endif()

# zlib
if (ENABLE_PYTHON OR ENABLE_NETCDF OR ENABLE_LIBXLSXWRITER OR ENABLE_OPENSSL)
    message(STATUS "zlib ................... Enabled")
    ExternalProject_Add(zlib
        DOWNLOAD_DIR ${PROJECT_BINARY_DIR}
        URL ${SENSEI_DEPS_URL}/${ZLIB_SOURCE}.tar.gz
        SOURCE_DIR ${PROJECT_BINARY_DIR}/${ZLIB_SOURCE}
        CONFIGURE_COMMAND env ${env}
             ${PROJECT_BINARY_DIR}/${ZLIB_SOURCE}/configure
            --prefix=${CMAKE_INSTALL_PREFIX} --enable-shared
        BUILD_IN_SOURCE 1 LOG_DOWNLOAD 1 LOG_CONFIGURE 1 LOG_BUILD 1 LOG_INSTALL 1)
else()
    message(STATUS "zlib ................... Disabled")
endif()

# OpenSSL
if (ENABLE_OPENSSL)
    message(STATUS "OpenSSL ................ Enabled")
    list(APPEND sensei_deps openssl)
    list(APPEND sensei_flags "-DREQUIRE_OPENSSL=TRUE")
    ExternalProject_Add(openssl
        DEPENDS zlib
        DOWNLOAD_DIR ${PROJECT_BINARY_DIR}
        URL ${SENSEI_DEPS_URL}/${OPENSSL_SOURCE}.tar.gz
        SOURCE_DIR ${PROJECT_BINARY_DIR}/${OPENSSL_SOURCE}
        CONFIGURE_COMMAND env ${env}
             ${PROJECT_BINARY_DIR}/${OPENSSL_SOURCE}/config
            enable-ssl3 enable-ssl3-method --prefix=${CMAKE_INSTALL_PREFIX} ${c_flags}
        BUILD_IN_SOURCE 1 LOG_DOWNLOAD 1 LOG_CONFIGURE 1 LOG_BUILD 1 LOG_INSTALL 1)
else()
    message(STATUS "OpenSSL ................ Disabled")
endif()

# Python group
if (ENABLE_PYTHON)
    set(py_deps ncurses zlib)
    # ncurses
    message(STATUS "ncurses ................ Enabled")
    ExternalProject_Add(ncurses
        DOWNLOAD_DIR ${PROJECT_BINARY_DIR}
        URL ${SENSEI_DEPS_URL}/${NCURSES_SOURCE}.tar.gz
        SOURCE_DIR ${PROJECT_BINARY_DIR}/${NCURSES_SOURCE}
        CONFIGURE_COMMAND env ${env} ${PROJECT_BINARY_DIR}/${NCURSES_SOURCE}/configure
            --prefix=${CMAKE_INSTALL_PREFIX}
        BUILD_IN_SOURCE 1 LOG_DOWNLOAD 1 LOG_CONFIGURE 1 LOG_BUILD 1 LOG_INSTALL 1)

    # readline
    if (ENABLE_READLINE)
        list(APPEND py_deps readline)
        message(STATUS "readline ............... Enabled")
        ExternalProject_Add(readline
            DEPENDS ncurses
            DOWNLOAD_DIR ${PROJECT_BINARY_DIR}
            URL ${SENSEI_DEPS_URL}/${READLINE_SOURCE}.tar.gz
            SOURCE_DIR ${PROJECT_BINARY_DIR}/${READLINE_SOURCE}
            CONFIGURE_COMMAND env ${env} ${PROJECT_BINARY_DIR}/${READLINE_SOURCE}/configure
                --prefix=${CMAKE_INSTALL_PREFIX}
            BUILD_IN_SOURCE 1 LOG_DOWNLOAD 1 LOG_CONFIGURE 1 LOG_BUILD 1 LOG_INSTALL 1)
    else()
        message(STATUS "readline ............... Disabled")
    endif()

    # Build Python.
    message(STATUS "Python ................. Enabled")
    list(APPEND sensei_flags "-DREQUIRE_PYTHON=TRUE")
    ExternalProject_Add(Python
        DEPENDS ${py_deps}
        DOWNLOAD_DIR ${PROJECT_BINARY_DIR}
        URL ${SENSEI_DEPS_URL}/${PYTHON_SOURCE}.tgz
        SOURCE_DIR ${PROJECT_BINARY_DIR}/${PYTHON_SOURCE}
        CONFIGURE_COMMAND env ${env} ${PROJECT_BINARY_DIR}/${PYTHON_SOURCE}/configure
            --enable-shared --with-ensurepip=yes --prefix=${CMAKE_INSTALL_PREFIX}
        LOG_DOWNLOAD 1 LOG_CONFIGURE 1 LOG_BUILD 1 LOG_INSTALL 1)

    # Build Swig.
    message(STATUS "SWIG ................... Enabled")
    list(APPEND sensei_deps SWIG)
    ExternalProject_Add(SWIG
        DOWNLOAD_DIR ${PROJECT_BINARY_DIR}
        URL ${SENSEI_DEPS_URL}/${SWIG_SOURCE}.tar.gz
        DEPENDS Python
        SOURCE_DIR ${PROJECT_BINARY_DIR}/${SWIG_SOURCE}
        CONFIGURE_COMMAND env ${env} ${PROJECT_BINARY_DIR}/${SWIG_SOURCE}/configure
            --with-python=${CMAKE_INSTALL_PREFIX}/bin/python3 --prefix=${CMAKE_INSTALL_PREFIX}
        LOG_DOWNLOAD 1 LOG_CONFIGURE 1 LOG_BUILD 1 LOG_INSTALL 1)

    # Build Cython.
    message(STATUS "Cython ................. Enabled")
    ExternalProject_Add(Cython
        DEPENDS Python
        DOWNLOAD_DIR ${PROJECT_BINARY_DIR}
        URL ${SENSEI_DEPS_URL}/${CYTHON_SOURCE}.tar.gz
        SOURCE_DIR ${PROJECT_BINARY_DIR}/${CYTHON_SOURCE}
        CONFIGURE_COMMAND ""
        BUILD_COMMAND env ${env} ${CMAKE_INSTALL_PREFIX}/bin/python3
            setup.py build install --prefix=${CMAKE_INSTALL_PREFIX}
        INSTALL_COMMAND ""
        BUILD_IN_SOURCE 1 LOG_DOWNLOAD 1 LOG_BUILD 1)

    # Build numpy.
    message(STATUS "NumPy .................. Enabled")
    list(APPEND sensei_deps numpy)
    ExternalProject_Add(numpy
        DEPENDS Cython
        DOWNLOAD_DIR ${PROJECT_BINARY_DIR}
        URL ${SENSEI_DEPS_URL}/${NUMPY_SOURCE}.tar.gz
        SOURCE_DIR ${PROJECT_BINARY_DIR}/${NUMPY_SOURCE}
        CONFIGURE_COMMAND ""
        BUILD_COMMAND env ${env} NPY_DISTUTILS_APPEND_FLAGS=1
           ${CMAKE_INSTALL_PREFIX}/bin/python3 setup.py build install
           "--prefix=${CMAKE_INSTALL_PREFIX}"
        INSTALL_COMMAND ""
        BUILD_IN_SOURCE 1 LOG_DOWNLOAD 1 LOG_BUILD 1)

    # Build mpi4py if MPI was requested.
    set(tmp "MPICH")
    if (DEFINED ENV{NERSC_HOST})
        set(tmp $ENV{NERSC_HOST})
    endif()
    set(MPI4PY_CONFIG ${tmp} CACHE STRING
        "Select the mpi4py configuration we use.")

    if (NOT WITHOUT_MPI)
        set(mpi4py_deps numpy)
        if (ENABLE_MPICH OR ENABLE_OPENMPI)
            list(APPEND mpi4py_deps mpi)
        endif()
        if (ENABLE_CRAY_MPICH)
            # set variables needed by mpi4py
            string(REPLACE ";" " " MPI4PY_LDFLAGS "${CRAY_MPICH_LDFLAGS}")
            string(REPLACE ";" " " MPI4PY_CFLAGS "${CRAY_MPICH_CFLAGS}")
        elseif(ENABLE_CORI_GPU)
            string(REPLACE ";" " " MPI4PY_LDFLAGS "${CORI_GPU_LDFLAGS}")
            string(REPLACE ";" " " MPI4PY_CFLAGS "${CORI_GPU_CFLAGS}")
        endif()
        configure_file(mpi4py.cfg.in mpi4py.cfg)
        message(STATUS "mpi4py ................. Enabled")
        list(APPEND sensei_deps mpi4py)
        ExternalProject_Add(mpi4py
            DEPENDS ${mpi4py_deps}
            DOWNLOAD_DIR ${PROJECT_BINARY_DIR}
            URL ${SENSEI_DEPS_URL}/${MPI4PY_SOURCE}.tar.gz
            SOURCE_DIR ${PROJECT_BINARY_DIR}/${MPI4PY_SOURCE}
            CONFIGURE_COMMAND ""
            BUILD_COMMAND env ${env}
                ${CMAKE_INSTALL_PREFIX}/bin/python3 setup.py build
                --mpi=${MPI4PY_CONFIG},${PROJECT_BINARY_DIR}/mpi4py.cfg
            INSTALL_COMMAND env ${env}
                ${CMAKE_INSTALL_PREFIX}/bin/python3 setup.py
            install --prefix=${CMAKE_INSTALL_PREFIX}
            BUILD_IN_SOURCE 1 LOG_DOWNLOAD 1 LOG_BUILD 1 LOG_INSTALL 1)
    else()
        message(STATUS "mpi4py ................. Disabled")
    endif()

    if (ENABLE_MATPLOTLIB)
        message(STATUS "matplotlib ............. Enabled")
        list(APPEND sensei_deps matplotlib)
        ExternalProject_Add(matplotlib
            DEPENDS numpy
            URL ""
            DOWNLOAD_COMMAND ""
            SOURCE_DIR ${PROJECT_BINARY_DIR}
            CONFIGURE_COMMAND ""
            BUILD_COMMAND env ${env} python3 -m pip install matplotlib
            INSTALL_COMMAND ""
            LOG_BUILD 1)
    else()
        message(STATUS "matplotlib ............. Disabled")
    endif()
else()
    message(STATUS "Python ................. Disabled")
    message(STATUS "setuptools ............. Disabled")
    message(STATUS "SWIG ................... Disabled")
    message(STATUS "Cython ................. Disabled")
    message(STATUS "NumPy .................. Disabled")
    message(STATUS "mpi4py ................. Disabled")
    message(STATUS "libpng ................. Disabled")
    message(STATUS "freetype ............... Disabled")
    message(STATUS "python_dateutil ........ Disabled")
    message(STATUS "pyparsing .............. Disabled")
    message(STATUS "cycler ................. Disabled")
    message(STATUS "pytz ................... Disabled")
    message(STATUS "kiwisolver ............. Disabled")
    message(STATUS "matplotlib ............. Disabled")
endif()

# VTK
if (ENABLE_VTK)
    message(STATUS "VTK .................... Enabled")
    list(APPEND sensei_deps vtk)
    list(APPEND sensei_flags "-DREQUIRE_VTK=TRUE")
    ExternalProject_Add(vtk
        DEPENDS Python
        DOWNLOAD_DIR ${PROJECT_BINARY_DIR}
        URL ${SENSEI_DEPS_URL}/${VTK_SOURCE}.tar.gz
        SOURCE_DIR ${PROJECT_BINARY_DIR}/${VTK_SOURCE}
        CMAKE_COMMAND env ${env} ${CMAKE_COMMAND}
            -DCMAKE_C_COMPILER=${CMAKE_C_COMPILER}
            "-DCMAKE_C_FLAGS=${CMAKE_C_FLAGS}"
            "-DCMAKE_C_FLAGS_DEBUG=${CMAKE_C_FLAGS}"
            "-DCMAKE_C_FLAGS_RELEASE=${CMAKE_C_FLAGS}"
            -DCMAKE_CXX_COMPILER=${CMAKE_CXX_COMPILER}
            "-DCMAKE_CXX_FLAGS=${CMAKE_CXX_FLAGS}"
            "-DCMAKE_CXX_FLAGS_DEBUG=${CMAKE_CXX_FLAGS_DEBUG}"
            "-DCMAKE_CXX_FLAGS_RELEASE=${CMAKE_CXX_FLAGS_RELEASE}"
            -DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX}
            ${sensei_flags} ${MPI_FLAGS}
            -DVTK_PYTHON_VERSION=3 -DVTK_WRAP_PYTHON=ON
            -DVTK_Group_Rendering=OFF -DTK_Group_StandAlone=OFF
        BUILD_COMMAND env ${env} make -j
        BUILD_IN_SOURCE 0 LOG_DOWNLOAD 1 LOG_CONFIGURE 1 LOG_BUILD 1 LOG_INSTALL 1)
else()
    message(STATUS "VTK .................... Disabled")
endif()


# SENSEI proper
if (ENABLE_SENSEI)
    message(STATUS "SENSEI ................. Enabled")

    # download and build SENSEI
    ExternalProject_Add(SENSEI
        DEPENDS ${sensei_deps}
        DOWNLOAD_DIR ${PROJECT_BINARY_DIR}
        GIT_REPOSITORY ${SENSEI_URL}
        GIT_TAG ${SENSEI_BRANCH}
        UPDATE_COMMAND ${SENSEI_UPDATE}
        CMAKE_COMMAND env ${env} ${CMAKE_COMMAND}
            -DCMAKE_C_COMPILER=${CMAKE_C_COMPILER}
            "-DCMAKE_C_FLAGS=${CMAKE_C_FLAGS}"
            "-DCMAKE_C_FLAGS_DEBUG=${CMAKE_C_FLAGS}"
            "-DCMAKE_C_FLAGS_RELEASE=${CMAKE_C_FLAGS}"
            -DCMAKE_CXX_COMPILER=${CMAKE_CXX_COMPILER}
            "-DCMAKE_CXX_FLAGS=${CMAKE_CXX_FLAGS}"
            "-DCMAKE_CXX_FLAGS_DEBUG=${CMAKE_CXX_FLAGS_DEBUG}"
            "-DCMAKE_CXX_FLAGS_RELEASE=${CMAKE_CXX_FLAGS_RELEASE}"
            -DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX}
            ${sensei_flags} ${MPI_FLAGS}
            -DSENSEI_PYTHON_VERSION=3 -DENABLE_VTK_IO=ON -DENABLE_VTK_MPI=OFF
            -DENABLE_VTK_FILTERS=ON -DENABLE_PYTHON=ON -DBUILD_TESTING=ON
            -DVTK_DIR=${CMAKE_INSTALL_PREFIX}/lib64/cmake/vtk-8.2/
            ${PROJECT_BINARY_DIR}/sensei
        LOG_DOWNLOAD 1 LOG_CONFIGURE 0 LOG_BUILD 0 LOG_INSTALL 0)

        # optionally run regression tests
        if (ENABLE_SENSEI_TEST)
            ExternalProject_Add_Step(SENSEI sensei_config
                WORKING_DIRECTORY <BINARY_DIR>
                TEST_COMMAND env ${env} ctest ${CTEST_FLAGS} -D Experimental || true
                DEPENDEES build)
        endif()

        # install sensei config
        ExternalProject_Add_Step(SENSEI sensei_config
            WORKING_DIRECTORY <SOURCE_DIR>/tools/sensei_config
            COMMAND env ${env} make_install ${CMAKE_INSTALL_PREFIX}
            DEPENDEES install
            ALWAYS 1)

else()
    message(STATUS "SENSEI ................. Disabled")
endif()
