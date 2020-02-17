cmake_minimum_required (VERSION 3.0)
project(sentry LANGUAGES C CXX ASM)

include(GNUInstallDirs)

set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Werror=implicit-function-declaration -Werror=incompatible-function-pointer-types")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++14")

include_directories("include")

set(SENTRY_NATIVE_SOURCE_ROOT ${CMAKE_CURRENT_SOURCE_DIR}/src)

file(GLOB_RECURSE SENTRY_NATIVE_SOURCES
    ${SENTRY_NATIVE_SOURCE_ROOT}/*.c
    ${SENTRY_NATIVE_SOURCE_ROOT}/*.cpp
)

add_definitions(-DSENTRY_WITH_INPROC_BACKEND)
add_library(sentry SHARED ${SENTRY_NATIVE_SOURCES})

include_directories(sentry PUBLIC include)

install(TARGETS sentry
    EXPORT ${PROJECT_NAME}-config
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)

set(HEADERS include/sentry.h)
install(FILES ${HEADERS} DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/sentry)
install(EXPORT ${PROJECT_NAME}-config DESTINATION cmake)