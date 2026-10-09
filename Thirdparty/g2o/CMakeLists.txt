CMAKE_MINIMUM_REQUIRED(VERSION 3.16)
set(CMAKE_LEGACY_CYGWIN_WIN32 0)

project(g2o CXX) 

# Standard settings
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release CACHE STRING "Choose the type of build." FORCE)
endif()

# Output directories
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}/lib)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}/lib)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}/bin)

# Dependencies
find_package(Eigen3 REQUIRED NO_MODULE)

# Interface library for common warnings
add_library(g2o_warnings INTERFACE)
target_compile_options(g2o_warnings INTERFACE
    $<$<CXX_COMPILER_ID:GNU,Clang>:-Wall -Wextra -Wpedantic>
    $<$<CXX_COMPILER_ID:MSVC>:/W4>
)

# Workaround for MSVC90
if(MSVC90)
  add_definitions(-DEIGEN_DONT_ALIGN_STATICALLY=1)
  message(STATUS "Disabling memory alignment for MSVC8")
endif()

# Platform macro used by g2o/stuff
if(WIN32)
  add_definitions(-DWINDOWS)
else()
  add_definitions(-DUNIX)
endif()

# Configuration Header
set(G2O_CXX_COMPILER "${CMAKE_CXX_COMPILER_ID} ${CMAKE_CXX_COMPILER}")
configure_file(config.h.in ${PROJECT_SOURCE_DIR}/config.h @ONLY)

# Create shared library
set(G2O_SOURCE_FILES
        #types
        g2o/types/types_sba.cpp
        g2o/types/types_six_dof_expmap.cpp
        g2o/types/types_seven_dof_expmap.cpp
        #core
        g2o/core/hyper_graph_action.cpp
        g2o/core/hyper_graph.cpp
        g2o/core/marginal_covariance_cholesky.cpp
        g2o/core/matrix_structure.cpp
        g2o/core/batch_stats.cpp
        g2o/core/parameter.cpp
        g2o/core/cache.cpp
        g2o/core/optimizable_graph.cpp
        g2o/core/solver.cpp
        g2o/core/optimization_algorithm_factory.cpp
        g2o/core/estimate_propagator.cpp
        g2o/core/factory.cpp
        g2o/core/sparse_optimizer.cpp
        g2o/core/hyper_dijkstra.cpp
        g2o/core/parameter_container.cpp
        g2o/core/optimization_algorithm.cpp
        g2o/core/optimization_algorithm_with_hessian.cpp
        g2o/core/optimization_algorithm_levenberg.cpp
        g2o/core/optimization_algorithm_gauss_newton.cpp
        g2o/core/jacobian_workspace.cpp
        g2o/core/robust_kernel.cpp
        g2o/core/robust_kernel_factory.cpp
        g2o/core/robust_kernel_impl.cpp
        #stuff
        g2o/stuff/timeutil.cpp
        g2o/stuff/os_specific.c
        g2o/stuff/string_tools.cpp
        g2o/stuff/property.cpp
)

# Create static library (linked into libORB_SLAM3)
add_library(${PROJECT_NAME} STATIC ${G2O_SOURCE_FILES})
set_target_properties(${PROJECT_NAME} PROPERTIES POSITION_INDEPENDENT_CODE ON)

# Apply warnings
target_link_libraries(${PROJECT_NAME} PRIVATE g2o_warnings)

# Set include directories for the target
target_include_directories(${PROJECT_NAME} PUBLIC
    $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}>
    $<INSTALL_INTERFACE:include>
)

# Link dependencies
target_link_libraries(${PROJECT_NAME} PUBLIC Eigen3::Eigen)

# Install targets and headers, only when built on its own (inside ORB-SLAM3 it is linked statically)
if(PROJECT_IS_TOP_LEVEL)
  install(TARGETS ${PROJECT_NAME}
          LIBRARY DESTINATION lib
          ARCHIVE DESTINATION lib
          RUNTIME DESTINATION bin
          INCLUDES DESTINATION include)

  # Install configuration header
  install(FILES ${PROJECT_SOURCE_DIR}/config.h DESTINATION include/g2o)

  # Install all other public headers
  install(DIRECTORY g2o/core g2o/types g2o/stuff
          DESTINATION include/g2o
          FILES_MATCHING PATTERN "*.h" PATTERN "*.hpp"
  )
endif()
