cmake_minimum_required(VERSION 3.8)

# Having below is very bad since it causes infinite compilation
# if (NOT APPLE)
# set(CMAKE_C_COMPILER /usr/bin/gcc)
# set(CMAKE_CXX_COMPILER /usr/bin/g++)
# endif()
project(JGSL CXX)

set(LINEAR_SOLVER "CHOLMOD" CACHE STRING "Direct linear system solver to use (options: EIGEN, CHOLMOD)")
set_property(CACHE LINEAR_SOLVER PROPERTY STRINGS "EIGEN" "CHOLMOD")

set(ENABLE_FEM_SHELL ON)

if(CMAKE_SOURCE_DIR STREQUAL CMAKE_CURRENT_SOURCE_DIR)
  set(CODIM_IPC_TOPLEVEL_PROJECT ON)
else()
  set(CODIM_IPC_TOPLEVEL_PROJECT OFF)
endif()

add_definitions(-DSTORAGE_ENABLED_OPENMP )
if (ENABLE_FEM_SHELL)
  add_definitions(-DENABLE_FEM_SHELL)
endif()



# set(CMAKE_CXX_FLAGS  "${CMAKE_CXX_FLAGS} -mfma -mbmi2 -mavx2 -fPIC -Wno-unused-variable -Wno-sign-compare -Wno-reorder")
# set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/build")
# set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/build")
list(APPEND CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/CMake)
# include_directories(${CMAKE_CURRENT_SOURCE_DIR}/Library)


find_package(PythonInterp 3 REQUIRED)
add_subdirectory(Externals)

set(PYBIND11_CPP_STANDARD -std=c++1z)
add_subdirectory(Externals/pybind11-master)
include_directories(Externals/flat_hash_map)
include_directories(Externals/meta/include)

pybind11_add_module(JGSL Library/EXPORTER.cpp
    Library/Math/Distance/EVCTCD/CTCD.cpp)
pybind11_add_module(CodimIPC SHARED Library/Math/Distance/EVCTCD/CTCD.cpp Library/Math/Distance/CCD.h)

target_include_directories(JGSL PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/Library)
target_include_directories(CodimIPC PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/Library)

target_compile_options(JGSL PRIVATE -mfma -mbmi2 -mavx2 -fPIC -Wno-unused-variable -Wno-sign-compare -Wno-reorder )
target_compile_options(CodimIPC PRIVATE -mfma -mbmi2 -mavx2 -fPIC -Wno-unused-variable -Wno-sign-compare -Wno-reorder )

set_target_properties(  CodimIPC
    PROPERTIES
    RUNTIME_OUTPUT_DIRECTORY  "${CMAKE_CURRENT_SOURCE_DIR}/build"
    LIBRARY_OUTPUT_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/build"
)

include(eigen)
find_package(Boost REQUIRED)

if(LINEAR_SOLVER STREQUAL "CHOLMOD")
  message("Linear solver: use CHOLMOD")
  include(onetbb)
  include(suitesparse)
  target_compile_definitions(JGSL PUBLIC CHOLMOD_DIRECT_SOLVER)
  target_compile_definitions(CodimIPC PUBLIC CHOLMOD_DIRECT_SOLVER)
else()
  message("Linear solver: use Eigen (default)")
endif()

if(LINEAR_SOLVER STREQUAL "CHOLMOD")
  set(JGSL_DEPS partio Kokkos::kokkos cabanacore Eigen3::Eigen amgcl TBB::tbb SuiteSparse::SuiteSparse)
else()
  set(JGSL_DEPS partio Kokkos::kokkos cabanacore Eigen3::Eigen amgcl)
endif()

target_include_directories(JGSL SYSTEM PUBLIC ${EIGEN3_INCLUDE_DIR} ${Boost_INCLUDE_DIRS})
target_link_libraries(JGSL PUBLIC ${JGSL_DEPS})

target_include_directories(CodimIPC SYSTEM PUBLIC ${EIGEN3_INCLUDE_DIR} ${Boost_INCLUDE_DIRS})
target_link_libraries(CodimIPC PUBLIC ${JGSL_DEPS})