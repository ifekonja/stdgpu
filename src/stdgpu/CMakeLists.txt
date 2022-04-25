# # Setup dependencies
# find_package(thrust 1.9.2 REQUIRED MODULE)

# set(STDGPU_DEPENDENCIES_INIT "
# find_dependency(thrust 1.9.2 REQUIRED MODULE)
# ")


# # Setup library options and config file
# if(CMAKE_BUILD_TYPE MATCHES "Release" OR CMAKE_BUILD_TYPE MATCHES "MinSizeRel")
#     set(STDGPU_ENABLE_CONTRACT_CHECKS_DEFAULT OFF)
# else()
#     set(STDGPU_ENABLE_CONTRACT_CHECKS_DEFAULT ON)
# endif()

# option(STDGPU_ENABLE_AUXILIARY_ARRAY_WARNING "Enable warnings when auxiliary arrays are allocated in memory API (deprecated), default: OFF" OFF)
# option(STDGPU_ENABLE_CONTRACT_CHECKS "Enable contract checks, default: OFF if CMAKE_BUILD_TYPE is Release or MinSizeRel, ON otherwise" ${STDGPU_ENABLE_CONTRACT_CHECKS_DEFAULT})
# option(STDGPU_ENABLE_MANAGED_ARRAY_WARNING "Enable warnings when managed memory is initialized on the host side but should be on device in memory API (deprecated), default: OFF" OFF)
# option(STDGPU_USE_32_BIT_INDEX "Use 32-bit instead of 64-bit signed integer for index_t, default: ON" ON)
# option(STDGPU_USE_FAST_DESTROY "Use fast destruction of allocated arrays (filled with a default value) by omitting destructor calls in memory API (deprecated), default: OFF" OFF)
# option(STDGPU_USE_FIBONACCI_HASHING "Use Fibonacci Hashing instead of Modulo to compute hash bucket indices (deprecated), default: ON" ON)

# configure_file("${STDGPU_INCLUDE_LOCAL_DIR}/stdgpu/config.h.in"
#                "${STDGPU_BUILD_INCLUDE_DIR}/stdgpu/config.h"
#                @ONLY)


# # FIXME:
# # The ROCm HCC compiler/linker seems to have trouble with template specializations.
# # While there are linker errors for STATIC libraries, SHARED libraries seem to work/link for some reason.
# if(STDGPU_BACKEND STREQUAL STDGPU_BACKEND_HIP)
#     message(WARNING "STDGPU_HIP_BACKEND: Building SHARED library to workaround linker errors with template specializations. This might be a bug in ROCm's HCC compiler/linker.")
#     add_library(stdgpu SHARED)
# else()
#     if(STDGPU_BUILD_SHARED_LIBS)
#         add_library(stdgpu SHARED)
#     else()
#         add_library(stdgpu STATIC)
#     endif()
# endif()

# target_sources(stdgpu PRIVATE impl/iterator.cpp
#                               impl/memory.cpp
#                               impl/limits.cpp)

# target_include_directories(stdgpu PUBLIC
#                                   $<BUILD_INTERFACE:${STDGPU_INCLUDE_LOCAL_DIR}>
#                                   $<BUILD_INTERFACE:${STDGPU_BUILD_INCLUDE_DIR}>
#                                   $<INSTALL_INTERFACE:${STDGPU_INCLUDE_INSTALL_DIR}>)

# target_compile_features(stdgpu PUBLIC cxx_std_14)

# target_compile_options(stdgpu PRIVATE ${STDGPU_DEVICE_FLAGS}
#                                       ${STDGPU_HOST_FLAGS})

# target_link_libraries(stdgpu PUBLIC thrust::thrust)

# set_target_properties(stdgpu PROPERTIES CXX_CLANG_TIDY "${STDGPU_PROPERTY_CLANG_TIDY}")
# set_target_properties(stdgpu PROPERTIES CXX_CPPCHECK "${STDGPU_PROPERTY_CPPCHECK}")

# add_library(stdgpu::stdgpu ALIAS stdgpu)


# add_subdirectory(${STDGPU_BACKEND_DIRECTORY})


# # Export targets and install header files
# # Installation
# include(GNUInstallDirs)
# install(TARGETS stdgpu
#         EXPORT stdgpu-targets
#         LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
#         ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
#         RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
#         PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/stdgpu/
#         COMPONENT stdgpu)

# install(DIRECTORY "${STDGPU_INCLUDE_LOCAL_DIR}/" "${STDGPU_BUILD_INCLUDE_DIR}/"
#         DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
#         COMPONENT stdgpu
#         FILES_MATCHING
#         PATTERN "*.h"
#         PATTERN "*.cuh"
#         PATTERN "*_fwd")

# # Install dependencies file and custom thrust module
# configure_file("${stdgpu_SOURCE_DIR}/cmake/stdgpu-dependencies.cmake.in"
#                "${STDGPU_BUILD_CMAKE_DIR}/stdgpu-dependencies.cmake"
#                @ONLY)

# install(FILES "${STDGPU_BUILD_CMAKE_DIR}/stdgpu-dependencies.cmake"
#         DESTINATION "${STDGPU_CMAKE_INSTALL_DIR}"
#         COMPONENT stdgpu)

# install(FILES "${stdgpu_SOURCE_DIR}/cmake/Findthrust.cmake"
#         DESTINATION "${STDGPU_CMAKE_INSTALL_DIR}"
#         COMPONENT stdgpu)

cmake_minimum_required(VERSION 3.16.0)

project(stdgpu LANGUAGES CXX)

# Options
option(STDGPU_ENABLE_AUXILIARY_ARRAY_WARNING "Enable warnings when auxiliary arrays are allocated in memory API (deprecated), default: OFF" OFF)
option(STDGPU_ENABLE_CONTRACT_CHECKS "Enable contract checks, default: OFF if CMAKE_BUILD_TYPE is Release or MinSizeRel, ON otherwise" ${STDGPU_ENABLE_CONTRACT_CHECKS_DEFAULT})
option(STDGPU_ENABLE_MANAGED_ARRAY_WARNING "Enable warnings when managed memory is initialized on the host side but should be on device in memory API (deprecated), default: OFF" OFF)
option(STDGPU_USE_32_BIT_INDEX "Use 32-bit instead of 64-bit signed integer for index_t, default: ON" ON)
option(STDGPU_USE_FAST_DESTROY "Use fast destruction of allocated arrays (filled with a default value) by omitting destructor calls in memory API (deprecated), default: OFF" OFF)
option(STDGPU_USE_FIBONACCI_HASHING "Use Fibonacci Hashing instead of Modulo to compute hash bucket indices (deprecated), default: ON" ON)

# Configuration
configure_file("config.h.in"
               "${CMAKE_CURRENT_SOURCE_DIR}/config.h"
               @ONLY)

set(CMAKE_CXX_STANDARD 17)

# Check for dependencies
find_package(CUDA REQUIRED)
find_path(THRUST_INCLUDE_DIRS "thrust/addressof.h")

enable_language("CUDA")
include_directories(${CUDA_INCLUDE_DIRS})
link_directories(${CUDA_LIBRARY_DIRS})
set(CUDA_NVCC_FLAGS ${CUDA_NVCC_FLAGS}
    --no-host-device-move-forward
    --expt-relaxed-constexpr
    --expt-extended-lambda
    --default-stream per-thread
    # --use_fast_math
#     -Xcudafe "--diag_suppress=integer_sign_change"
#     -Xcudafe "--diag_suppress=partial_override"
#     -Xcudafe "--diag_suppress=virtual_function_decl_hidden"
)

# Define the library
# General Source file
file(GLOB_RECURSE ALL_SOURCE_FILES "impl/*.cpp" "cuda/impl/*.cu")

cuda_add_library(${PROJECT_NAME} STATIC ${ALL_SOURCE_FILES})

set_property(TARGET ${PROJECT_NAME} PROPERTY CUDA_ARCHITECTURES 35)
#set_target_properties(${PROJECT_NAME} PROPERTIES CUDA_SEPARABLE_COMPILATION ON)

target_compile_options(${PROJECT_NAME} PRIVATE ${STDGPU_DEVICE_FLAGS} ${STDGPU_HOST_FLAGS})

target_include_directories(${PROJECT_NAME}
    PUBLIC $<INSTALL_INTERFACE:include> $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/../>
    PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/
)

#target_link_libraries(${PROJECT_NAME} thrust::thrust)

# Installation
# Add 'd' suffix to debug builds
set_property(TARGET ${PROJECT_NAME} PROPERTY DEBUG_POSTFIX d)

# Installation
include(GNUInstallDirs)

file(GLOB_RECURSE ALL_HEADER_FILES "*.h" "*.cuh" "*_fwd" "cuda/*.cuh" "cuda/*.h")

set_target_properties(${PROJECT_NAME} PROPERTIES PUBLIC_HEADER "${ALL_HEADER_FILES}")

export(TARGETS ${PROJECT_NAME} FILE "${PROJECT_NAME}Config.cmake")

install(TARGETS ${PROJECT_NAME}
    EXPORT "${PROJECT_NAME}Config"
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    #PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/stdgpu
)

install(DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}"
        DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
        FILES_MATCHING
        PATTERN "*.h"
        PATTERN "*.cuh"
        PATTERN "*_fwd")

install(EXPORT "${PROJECT_NAME}Config"
    FILE "${PROJECT_NAME}Config.cmake"
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/
)
