cmake_minimum_required(VERSION 3.0)

project(pyboost_ipc)

find_package(Boost REQUIRED COMPONENTS python)

find_package(PythonLibs)
find_package(PythonInterp REQUIRED)

if (NOT ${PYTHONLIBS_FOUND})
    # attempt to find python if 'python.exe' is found
    message(STATUS "Trying to use the 'libs' and 'Include' directories at the python interpreter path")
    if (NOT ${PYTHONINTERP_FOUND})
        message(FATAL_ERROR "Didn't find the python libraries nor a python interpreter. You can pass the variables:
                             -DPYTHON_LIBRARY=<python_path>/libs
                             -DPYTHON_INCLUDE_DIR=<python_path>/Include
                             instead.
                             Make sure that the Python.h and the static libraries are installed (conda install libpython)
                             "
               )
    endif()

    get_filename_component(PYTHONINTERP_PATH ${PYTHON_EXECUTABLE} DIRECTORY)
    set(PYTHON_LIBRARIES ${PYTHONINTERP_PATH}/libs)
    set(PYTHON_INCLUDE_DIRS ${PYTHONINTERP_PATH}/include)
    execute_process(COMMAND ${PYTHON_EXECUTABLE} -c "from distutils.sysconfig import get_python_lib; print get_python_lib(True)" OUTPUT_VARIABLE PYTHON_SITE_PACKAGES OUTPUT_STRIP_TRAILING_WHITESPACE)
endif()

# get the 'site-packages' directory
execute_process(COMMAND ${PYTHON_EXECUTABLE} -c "from distutils.sysconfig import get_python_lib; print get_python_lib(True)" OUTPUT_VARIABLE PYTHON_SITE_PACKAGES OUTPUT_STRIP_TRAILING_WHITESPACE)

include_directories(
    ${Boost_INCLUDE_DIRS}
    ${PYTHON_INCLUDE_DIRS}
    source/c++
)

# 
set(SOURCES
    source/c++/boost_ipc/api.hpp
    source/c++/boost_ipc/api.inl.hpp
    source/c++/boost_ipc/internal.hpp
    source/c++/boost_ipc/_boost_ipc.cpp
    source/c++/boost_ipc/containers.cpp
    source/c++/boost_ipc/creation_tags.cpp
    source/c++/boost_ipc/instance_types.cpp
    source/c++/boost_ipc/managed_shared_memory.cpp
    source/c++/boost_ipc/message_queue.cpp
    source/c++/boost_ipc/offset_ptrs.cpp
    source/c++/boost_ipc/shared_memory_object.cpp
    source/c++/boost_ipc/string.cpp
)

add_definitions(
    -DBOOST_IPC_EXPORT
    -DBOOST_ALL_NO_LIB  # since we link the libraries using cmake we disable auto-linking
)

# Linking with Python for MSVC. I have to use link_directories() because the result of PYTHON_LIBRARIES is a directory.
# Also, the link_directories() command must be done before the target.
if (${MSVC})
    link_directories(${PYTHON_LIBRARIES})
endif()

add_library(pyboost_ipc SHARED ${SOURCES})

# -fPIC
set_property(TARGET pyboost_ipc PROPERTY POSITION_INDEPENDENT_CODE 1)

# suffix and prefix
if (${MSVC})
    set_target_properties(pyboost_ipc PROPERTIES SUFFIX ".pyd")
    set(CMAKE_CXX_FLAGS "/EHsc /bigobj")
endif()
set_target_properties(pyboost_ipc PROPERTIES PREFIX "")

# Linking with Python
message("pylibs: " ${PYTHON_LIBRARIES})
if (NOT ${MSVC})
    target_link_libraries(pyboost_ipc ${PYTHON_LIBRARIES})
endif()

# linking with Boost.Python
target_link_libraries(pyboost_ipc
    ${Boost_LIBRARIES}
)

message(STATUS "Will install to: ${PYTHON_SITE_PACKAGES}")
install(TARGETS pyboost_ipc
    RUNTIME DESTINATION ${PYTHON_SITE_PACKAGES}
    LIBRARY DESTINATION ${PYTHON_SITE_PACKAGES}
)