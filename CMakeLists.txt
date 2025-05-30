cmake_minimum_required(VERSION 3.19)
project(inflate64 C CXX)

set(CMAKE_C_STANDARD 11)
set(CMAKE_CXX_STANDARD 17)

# TARGET PYTHON version
set(PY_VERSION 3.13)
set(Python_FIND_IMPLEMENTATIONS CPython)
#set(Python_FIND_IMPLEMENTATIONS PyPy)
set(VENV_PATH "${CMAKE_BINARY_DIR}/venv")
set(DEBUG_BUILD OFF CACHE BOOL "Enable debug build?")
set(DEBUG_LOG_LEVEL "1" CACHE STRING "Debug log level?")

if (DEBUG_BUILD)
else()
  set(COMPILE_WARNING_AS_ERROR ON)
endif()
# ##################################################################################################
#
set(_sources
    src/ext/_inflate64module.c
    src/lib/deflate.c
    src/lib/inflate.c
    src/lib/inflate_tree.c
    src/lib/deflate_tree.c
    src/lib/util.c
    src/lib/static_tables.c
)
# ##################################################################################################
# Configuration for python-ext
set(Python_FIND_STRATEGY VERSION)
find_package(Python ${PY_VERSION}.0...${PY_VERSION}.99 COMPONENTS Interpreter Development)

set(PY_PACKAGE inflate64)
set(PY_EXT_FILE _inflate64)

execute_process(
    COMMAND ${Python_EXECUTABLE} -c "import sysconfig\nprint(sysconfig.get_config_var('EXT_SUFFIX'))\n"
    OUTPUT_VARIABLE PY_EXT_EXT
    OUTPUT_STRIP_TRAILING_WHITESPACE)
execute_process(
    COMMAND ${Python_EXECUTABLE} -c "import sysconfig\nprint(sysconfig.get_platform())\n"
    OUTPUT_VARIABLE PY_PLATFORM
    OUTPUT_STRIP_TRAILING_WHITESPACE)
execute_process(
        COMMAND ${Python_EXECUTABLE} -c "import sysconfig\nprint(sysconfig.get_python_version())\n"
        OUTPUT_VARIABLE PY_VERSION
        OUTPUT_STRIP_TRAILING_WHITESPACE)
message(STATUS "Python SOABI is ${Python_SOABI}")
set(PY_SRC_DIR src/${PY_PACKAGE})
set(EXT_DIR ${CMAKE_BINARY_DIR})
set(PY_EXT_DIR build/lib.${PY_PLATFORM}-${PY_VERSION}/${PY_PACKAGE})
set(PY_EXT ${PY_EXT_DIR}/${PY_EXT_FILE}${PY_EXT_EXT})
set(EXT ${EXT_DIR}/${PY_EXT_FILE}.${PY_EXT_EXT})
set(PY_EXT_INLINE ${CMAKE_SOURCE_DIR}/src/${PY_PACKAGE}/${PY_EXT_FILE}${PY_EXT_EXT})

# ##################################################################################################
# bulid ext by setup.py
if (WIN32)
  if(DEBUG_BUILD)
    set(BUILD_EXT_PYTHON ${VENV_PATH}/Scripts/python_d.exe)
  else()
    set(BUILD_EXT_PYTHON ${VENV_PATH}/Scripts/python.exe)
  endif()
  set(BUILD_EXT_OPTION)
else()
  set(BUILD_EXT_PYTHON ${VENV_PATH}/bin/python)
  set(BUILD_EXT_OPTION --warning-as-error)
endif()
add_custom_target(
  build_ext_by_setup
  BYPRODUCTS ${PY_EXT}
  COMMAND ${BUILD_EXT_PYTHON} setup.py build_ext ${BUILD_EXT_OPTION}
  WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
  DEPENDS venv.stamp
  SOURCES ${_sources})

# ##################################################################################################
# build ext by cmake
include_directories(src/lib)
Python_add_library(_inflate64 MODULE WITH_SOABI ${_sources})
if(MSVC)
  target_compile_options(_inflate64 PRIVATE "/wd4996")
endif()
if(DEBUG_BUILD)
  target_compile_definitions(_inflate64 PRIVATE -DZLIB_DEBUG=${DEBUG_LOG_LEVEL})
endif()
add_custom_target(build_ext
                  BYPRODUCTS ${PY_EXT_INLINE}
                  COMMAND ${CMAKE_COMMAND} -E copy_if_different $<TARGET_FILE:_inflate64> "${PY_SRC_DIR}"
                  DEPENDS _inflate64 venv.stamp
                  )
# ##################################################################################################
# create virtualenv
file(
        WRITE ${CMAKE_CURRENT_BINARY_DIR}/requirements.txt
        "
coverage[toml]>=5.2
hypothesis
pytest>=6.0
pytest-benchmark
pytest-cov
pytest-timeout
cffi
")
if (WIN32)
  set(PIP_COMMAND ${VENV_PATH}/Scripts/pip.exe)
else()
  set(PIP_COMMAND ${VENV_PATH}/bin/pip)
endif()
add_custom_target(
        venv.stamp
        BYPRODUCTS venv.stamp
        COMMAND ${Python_EXECUTABLE} -m venv ${VENV_PATH}
        COMMAND ${PIP_COMMAND} install -r ${CMAKE_BINARY_DIR}/requirements.txt
        COMMAND ${CMAKE_COMMAND} -E touch venv.stamp)
set(SRC_PATH "${CMAKE_SOURCE_DIR}/src")
set(VPKG_PATH_A "${VENV_PATH}/lib/python${Python_VERSION_MAJOR}.${Python_VERSION_MINOR}/site-packages/")
set(VPKG_PATH_B "${VENV_PATH}/Lib/site-packages/")
set(VPKG_PATH_C "${CMAKE_BINARY_DIR}")

# ##################################################################################################
# For pytest
file(
        WRITE ${CMAKE_CURRENT_BINARY_DIR}/pytest_runner.cpp
        "
#include <string>
#include <filesystem>
#include <unistd.h>
int main(int argc, char **argv) {
    std::string args;
    if ( argc > 1) {
        args.append(\"[\");
        for (int i = 1; i < argc; i++) {
            if (i > 2)
                args.append(\",\");
            args.append(\"\\\"\");
            args.append(argv[i]);
            args.append(\"\\\"\");
        }
        args.append(\"]\");
    }
    std::filesystem::path src_path = \"${SRC_PATH}\";
    std::filesystem::path vsite_path_a = \"${VPKG_PATH_A}\";
    std::filesystem::path vsite_path_b = \"${VPKG_PATH_B}\";
    std::filesystem::path vsite_path_c = \"${VPKG_PATH_C}\";
    std::string pycode =
        \"import sys\\n\"
        \"sys.path.append('\" + src_path.string() + \"')\\n\"
        \"sys.path.append('\" + vsite_path_a.string() + \"')\\n\"
        \"sys.path.append('\" + vsite_path_b.string() + \"')\\n\"
        \"import pytest\\n\"
        \"pytest.main(\" + args + \")\\n\";
    execl(\"${Python_EXECUTABLE}\", \"${Python_EXECUTABLE}\", \"-c\", pycode.c_str(), (char*)0);
    return 0;
}")
add_executable(pytest_runner ${CMAKE_CURRENT_BINARY_DIR}/pytest_runner.cpp)
if ("${Python_INTERPRETER_ID}" STREQUAL "PyPy")
  add_dependencies(pytest_runner generate_cffi)
else()
  add_dependencies(pytest_runner build_ext)
endif()
target_include_directories(pytest_runner PRIVATE ${Python_INCLUDE_DIRS})
target_link_libraries(pytest_runner PRIVATE ${Python_LIBRARIES})
add_dependencies(pytest_runner venv.stamp build_ext)

# ##################################################################################################
# make table utility
add_executable(makefixed9
               src/lib/static_tables.c
               src/lib/inflate.c
               src/lib/inflate_tree.c
               src/lib/deflate_tree.c
               src/lib/util.c
        )
target_compile_definitions(makefixed9 PRIVATE -DGEN_TREES_H)
# ##################################################################################################
add_custom_target(run_tox
        COMMAND  ${CMAKE_COMMAND} -E env VIRTUAL_ENV=${VENV_PATH} ${VENV_PATH}/bin/python -m tox
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
        DEPENDS venv.stamp
)
