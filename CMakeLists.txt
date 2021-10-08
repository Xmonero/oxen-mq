cmake_minimum_required(VERSION 3.7)

project(libqueneromq CXX)

include(GNUInstallDirs)

set(LOKIMQ_VERSION_MAJOR 1)
set(LOKIMQ_VERSION_MINOR 1)
set(LOKIMQ_VERSION_PATCH 3)
set(LOKIMQ_VERSION "${LOKIMQ_VERSION_MAJOR}.${LOKIMQ_VERSION_MINOR}.${LOKIMQ_VERSION_PATCH}")
message(STATUS "queneromq v${LOKIMQ_VERSION}")

set(LOKIMQ_LIBVERSION 0)


option(BUILD_SHARED_LIBS "Build shared libraries instead of static ones" ON)


list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

configure_file(queneromq/version.h.in queneromq/version.h @ONLY)
configure_file(libqueneromq.pc.in libqueneromq.pc @ONLY)

add_library(queneromq
    queneromq/auth.cpp
    queneromq/bt_serialize.cpp
    queneromq/connections.cpp
    queneromq/jobs.cpp
    queneromq/queneromq.cpp
    queneromq/proxy.cpp
    queneromq/worker.cpp
)
set_target_properties(queneromq PROPERTIES SOVERSION ${LOKIMQ_LIBVERSION})

set(THREADS_PREFER_PTHREAD_FLAG ON)
find_package(Threads REQUIRED)
target_link_libraries(queneromq PRIVATE Threads::Threads)

# libzmq is nearly impossible to link statically from a system-installed static library: it depends
# on a ton of other libraries, some of which are not all statically available.  If the caller wants
# to mess with this, so be it: they can set up a libzmq target and we'll use it.  Otherwise if they
# asked us to do things statically, don't even try to find a system lib and just build it.
set(queneromq_build_static_libzmq OFF)
if(TARGET libzmq)
    target_link_libraries(queneromq PUBLIC libzmq)
elseif(BUILD_SHARED_LIBS)
    include(FindPkgConfig)
    pkg_check_modules(libzmq libzmq>=4.3 IMPORTED_TARGET)
    if(libzmq_FOUND)
        target_link_libraries(queneromq PUBLIC PkgConfig::libzmq)
    else()
        set(queneromq_build_static_libzmq ON)
    endif()
else()
    set(queneromq_build_static_libzmq ON)
endif()

if(queneromq_build_static_libzmq)
    message(STATUS "libzmq >= 4.3 not found or static build requested, building bundled 4.3.2")
    list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake/local-libzmq")
    include(LocalLibzmq)
    target_link_libraries(queneromq PUBLIC libzmq_vendor)
endif()

target_include_directories(queneromq
    PUBLIC
        $<INSTALL_INTERFACE:>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/cppzmq>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/mapbox-variant/include>
)

target_compile_options(queneromq PRIVATE -Wall -Wextra -Werror)
set_target_properties(queneromq PROPERTIES
    CXX_STANDARD 14
    CXX_STANDARD_REQUIRED ON
    CXX_EXTENSIONS OFF
    POSITION_INDEPENDENT_CODE ON
)

function(link_dep_libs target linktype libdirs)
    foreach(lib ${ARGN})
        find_library(link_lib-${lib} NAMES ${lib} PATHS ${libdirs})
        message(STATUS "FIND ${lib} FOUND ${link_lib-${lib}}")
        if(link_lib-${lib})
            target_link_libraries(${target} ${linktype} ${link_lib-${lib}})
        endif()
    endforeach()
endfunction()

# If the caller has already set up a sodium target then we will just link to it, otherwise we go
# looking for it.
if(TARGET sodium)
    target_link_libraries(queneromq PRIVATE sodium)
    if(queneromq_build_static_libzmq)
        target_link_libraries(libzmq_vendor INTERFACE sodium)
    endif()
else()
    pkg_check_modules(sodium REQUIRED libsodium IMPORTED_TARGET)

    if(BUILD_SHARED_LIBS)
        target_link_libraries(queneromq PRIVATE PkgConfig::sodium)
        if(queneromq_build_static_libzmq)
            target_link_libraries(libzmq_vendor INTERFACE PkgConfig::sodium)
        endif()
    else()
        link_dep_libs(queneromq PRIVATE "${sodium_STATIC_LIBRARY_DIRS}" ${sodium_STATIC_LIBRARIES})
        target_include_directories(queneromq PRIVATE ${sodium_STATIC_INCLUDE_DIRS})
        if(queneromq_build_static_libzmq)
            link_dep_libs(libzmq_vendor INTERFACE "${sodium_STATIC_LIBRARY_DIRS}" ${sodium_STATIC_LIBRARIES})
            target_link_libraries(libzmq_vendor INTERFACE ${sodium_STATIC_INCLUDE_DIRS})
        endif()
    endif()
endif()

add_library(queneromq::queneromq ALIAS queneromq)

export(
    TARGETS queneromq
    NAMESPACE queneromq::
    FILE queneromqTargets.cmake
)
install(
    TARGETS queneromq
    EXPORT queneromqConfig
    DESTINATION ${CMAKE_INSTALL_LIBDIR}
)

install(
    FILES queneromq/auth.h
          queneromq/batch.h
          queneromq/bt_serialize.h
          queneromq/connections.h
          queneromq/hex.h
          queneromq/queneromq.h
          queneromq/message.h
          queneromq/string_view.h
          ${CMAKE_CURRENT_BINARY_DIR}/queneromq/version.h
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/queneromq
)
option(LOKIMQ_INSTALL_MAPBOX_VARIANT "Install mapbox-variant headers with queneromq/ headers" ON)
if(LOKIMQ_INSTALL_MAPBOX_VARIANT)
  install(
    FILES mapbox-variant/include/mapbox/variant.hpp
          mapbox-variant/include/mapbox/variant_cast.hpp
          mapbox-variant/include/mapbox/variant_io.hpp
          mapbox-variant/include/mapbox/variant_visitor.hpp
          mapbox-variant/include/mapbox/recursive_wrapper.hpp
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/queneromq/mapbox
  )
endif()

option(LOKIMQ_INSTALL_CPPZMQ "Install cppzmq header with queneromq/ headers" ON)
if(LOKIMQ_INSTALL_CPPZMQ)
  install(
    FILES cppzmq/zmq.hpp
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/queneromq
  )
endif()


install(
    FILES ${CMAKE_CURRENT_BINARY_DIR}/libqueneromq.pc
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig
)

if(CMAKE_PROJECT_NAME STREQUAL PROJECT_NAME)
  set(queneromq_IS_TOPLEVEL_PROJECT TRUE)
else()
  set(queneromq_IS_TOPLEVEL_PROJECT FALSE)
endif()

option(LOKIMQ_BUILD_TESTS "Building and perform queneromq tests" ${queneromq_IS_TOPLEVEL_PROJECT})
if(LOKIMQ_BUILD_TESTS)
    add_subdirectory(tests)
endif()

