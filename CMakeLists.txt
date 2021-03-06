# Copyright (C) 2011 David Sugar, Tycho Softworks.
#
# This file is free software; as a special exception the author gives
# unlimited permission to copy and/or distribute it, with or without
# modifications, as long as this notice is preserved.
#
# This program is distributed in the hope that it will be useful, but
# WITHOUT ANY WARRANTY, to the extent permitted by law; without even the
#

cmake_minimum_required(VERSION 2.6)
PROJECT(sipwitch)
INCLUDE(CheckIncludeFiles)
INCLUDE(CheckFunctionExists)
INCLUDE(FindPkgConfig)
INCLUDE(GNUInstallDirs)
set (VERSION 1.9.6)
set (PACKAGE "sipwitch")

set(RC_VERSION ${VERSION})
STRING(REGEX REPLACE "[.]" "," RC_VERSION ${RC_VERSION})
set(RC_VERSION "${RC_VERSION},0")

if(UNIX AND CMAKE_COMPILER_IS_GNUCXX)
    execute_process(WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} COMMAND "./cmake-abi.sh" OUTPUT_VARIABLE LIB_VERSION)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O2 -fPIE -fstack-protector -D_FORTIFY_SOURCE=2 --param ssp-buffer-size=4 -pie")
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -pie -z relro -z now")
else()
    set(LIB_VERSION ${VERSION})
endif()

set(SOVERSION ${LIB_VERSION})
STRING(REGEX REPLACE "[.].*$" "" SOVERSION ${SOVERSION})
set(MODULE_EXT "${CMAKE_SHARED_LIBRARY_SUFFIX}")
set(INSTALL_PLUGINS "${CMAKE_INSTALL_FULL_LIBDIR}/sipwitch")
set(CMAKE_INSTALL_CGIBINDIR "/srv/www/cgi-bin")

if(WIN32)
    set(MODULE_EXT ".dll")
else()
    set(MODULE_EXT ".so")
endif()

if (WIN32)
    # too broken to build dll's with const static members...
    if(MSVC60)
        set(BUILD_STATIC ON CACHE BOOL "static linking only" FORCE)
        MARK_AS_ADVANCED(FORCE BUILD_STATIC)
    else()
        option(BUILD_STATIC "Set to OFF to build shared libraries" ON)
    endif()
    set(HAVE_SIGWAIT false)
    set(HAVE_SIGWAIT2 off CACHE BOOL "disable sigwait on windows" FORCE)
    MARK_AS_ADVANCED(FORCE HAVE_SIGWAIT2)
else()
    option(BUILD_STATIC "Set to ON to build static libraries" OFF)
    option(HAVE_SIGWAIT2 "Set to OFF to build with single argument" ON)
    MARK_AS_ADVANCED(HAVE_SIGWAIT2)

    set(HAVE_SIGWAIT true)
endif()

MESSAGE( STATUS "Configuring GNU sipwitch ${VERSION}...")

# set to true for debug and trace during CMakeLists development
set(CMAKE_VERBOSE_MAKEFILE FALSE)

# Set defaults and pass common options.  Common build options can be passed
# to cmake using cmake -DWITH_CFLAGS="...", WITH_LIBS, and WITH_INCLUDES
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/inc ${WITH_INCLUDES})
add_definitions(${WITH_CFLAGS})
link_libraries(${WITH_LIBS})
find_package(PkgConfig)

if (USES_UCOMMON_INCLUDE_DIRS)
    message(STATUS "  Using local ucommon dependency")
else()
    pkg_check_modules(USES_UCOMMON REQUIRED ucommon>=6.0.0)
endif()

pkg_check_modules(USES_ZEROCONF avahi-client>=0.3.0)

if(USES_ZEROCONF_FOUND)
    set(ZEROCONF_AVAHI true)
endif()

pkg_check_modules(USES_SYSTEMD libsystemd-daemon>=44)
if(USES_SYSTEMD_FOUND)
    set(HAVE_SYSTEMD true)
endif()

include_directories(${USES_UCOMMON_INCLUDE_DIRS})
link_directories(${USES_UCOMMON_LIBRARY_DIRS})
add_definitions(${USES_UCOMMON_CFLAGS})

# by default we build static libs for windows, shared libs for unix.
# we may also set this from a top level cmake or -DWITH_XX_LIBS

if(BUILD_STATIC)
    set(BUILD_LIBRARY_TYPE STATIC)
else()
    set(BUILD_LIBRARY_TYPE SHARED)
endif()

if(NOT BUILD_RUNTIME_TYPE)
    set(BUILD_RUNTIME_TYPE ${BUILD_LIBRARY_TYPE})
endif()

check_include_files(sys/resource.h HAVE_SYS_RESOURCE_H)
check_include_files(sys/inotify.h HAVE_SYS_INOTIFY_H)
check_include_files(syslog.h HAVE_SYSLOG_H)
check_include_files(net/if.h HAVE_NET_IF_H)
check_include_files(sys/sockio.h HAVE_SYS_SOCKIO_H)
check_include_files(ioctl.h HAVE_IOCTL_H)
check_include_files(pwd.h HAVE_PWD_H)
check_include_files(eXosip2/eXosip.h HAVE_EXOSIP2)

if(HAVE_EXOSIP2)
    check_include_files(openssl/ssl.h HAVE_OPENSSL)
else()
    message(ERROR "libeXosip2 required to build")
endif()

if(HAVE_OPENSSL)
    if(WIN32 AND NOT CYGWIN AND NOT MINGW)
        set (EXOSIP2_LIBS eXosip2 osip2 osipparser2 ssleay32 libeay32 gdi32)
    else()
        set (EXOSIP2_LIBS eXosip2 osip2 osipparser2 ssl crypto z)
    endif()
else()
    set(EXOSIP2_LIBS eXosip2 osip2 osipparser2)
endif()

check_include_files(resolv.h HAVE_RESOLV_H)
if(HAVE_RESOLV_H)
    set(EXOSIP2_LIBS ${EXOSIP2_LIBS} resolv)
endif()

if(WIN32 AND NOT CYGWIN)
    set(EXOSIP2_LIBS ${EXOSIP2_LIBS} iphlpapi dnsapi)
endif()

check_function_exists(setrlimit HAVE_SETRLIMIT)
check_function_exists(setgroups HAVE_SETGROUPS)
check_function_exists(setpgrp HAVE_SETPGRP)
check_function_exists(getuid HAVE_GETUID)
check_function_exists(mkfifo HAVE_MKFIFO)
check_function_exists(symlink HAVE_SYMLINK)
check_function_exists(atexit HAVE_ATEXIT)

configure_file(${CMAKE_CURRENT_SOURCE_DIR}/sipwitch-config.h.cmake ${CMAKE_CURRENT_BINARY_DIR}/sipwitch-config.h)
include_directories(${CMAKE_CURRENT_BINARY_DIR})

file(GLOB runtime_src common/*.cpp)
file(GLOB runtime_inc inc/sipwitch/*.h)
file(GLOB sipwitch_man1 utils/*.1)
file(GLOB sipwitch_man8 utils/*.8 server/*.8)
file(GLOB sipwitch_conf server/*.conf)
file(GLOB sipwitch_xml server/*.xml*)

STRING(COMPARE EQUAL "${BUILD_RUNTIME_TYPE}" "SHARED" HAVE_PLUGINS)

add_library(sipwitch ${BUILD_RUNTIME_TYPE} ${runtime_src} ${runtime_inc})
set_target_properties(sipwitch PROPERTIES VERSION ${LIB_VERSION} SOVERSION ${SOVERSION})
add_dependencies(sipwitch ucommon)
target_link_libraries(sipwitch ${USES_UCOMMON_LIBRARIES} ${EXOSIP2_LIBS})
set_target_properties(sipwitch PROPERTIES OUTPUT_NAME sipwitch)

add_subdirectory(server)

add_executable(sipwitch-query utils/sipquery.cpp)
add_dependencies(sipwitch-query usecure ucommon eXosip2)
target_link_libraries(sipwitch-query usecure ucommon ${EXOSIP2_LIBS} ${USES_UCOMMON_LIBRARIES})
set_target_properties(sipwitch-query PROPERTIES OUTPUT_NAME sipquery)

install(TARGETS sipwitch-query DESTINATION ${CMAKE_INSTALL_BINDIR})

add_executable(sipwitch-cgi utils/cgiserver.cpp)
add_dependencies(sipwitch-cgi ucommon)
target_link_libraries(sipwitch-cgi ucommon ${USES_UCOMMON_LIBRARIES})
if(UNIX)
    set_target_properties(sipwitch-cgi PROPERTIES OUTPUT_NAME sipwitch.cgi)
endif()

add_executable(sipwitch-passwd utils/sippasswd.cpp)
add_dependencies(sipwitch-passwd usecure ucommon)
target_link_libraries(sipwitch-passwd usecure ucommon ${USES_UCOMMON_LIBRARIES})
set_target_properties(sipwitch-passwd PROPERTIES OUTPUT_NAME sippasswd)

add_executable(sipwitch-control utils/sipwitch.cpp)
add_dependencies(sipwitch-control usecure ucommon)
target_link_libraries(sipwitch-control usecure ucommon ${USES_UCOMMON_LIBRARIES})
set_target_properties(sipwitch-control PROPERTIES OUTPUT_NAME sipwitch)

if(CMAKE_SYSTEM_NAME STREQUAL Linux)
    # in DEBIAN we would set this off, in opensuse/rpm based, on...
    option(SYSTEM_CONFIG "Set to ON to write system config" OFF)
    if(SYSTEM_CONFIG)
        if(NOT CMAKE_INSTALL_UNITDIR)
            set(CMAKE_INSTALL_UNITDIR lib/systemd/system)
        endif()
        file(GLOB sipwitch_log server/*.log)
        file(GLOB sipwitch_cron server/*.cron)
        file(GLOB sipwitch_def server/*.defaults)
        file(GLOB sipwitch_init server/*.service)
        install(FILES   ${sipwitch_log} DESTINATION ${CMAKE_INSTALL_SYSCONFDIR}/logrotate.d RENAME sipwitch)
        install(FILES   ${sipwitch_cron} DESTINATION ${CMAKE_INSTALL_SYSCONFDIR}/cron.hourly RENAME sipwitch)
        install(FILES   ${sipwitch_def} DESTINATION ${CMAKE_INSTALL_SYSCONFDIR}/default RENAME sipwitch)
        install(FILES   ${sipwitch_init} DESTINATION ${CMAKE_INSTALL_UNITDIR})
    endif()
endif()

if(UNIX)
    foreach(flag ${EXOSIP2_FLAGS})
        set(PACKAGE_FLAGS "${PACKAGE_FLAGS} ${flag}")
    endforeach()
    foreach(lib ${EXOSIP2_LIBS})
        STRING(REGEX REPLACE "^[-]l" "" slib ${lib})
        set(PACKAGE_LIBS "${PACKAGE_LIBS} -l${slib}")
    endforeach()

    configure_file(${CMAKE_CURRENT_SOURCE_DIR}/libsipwitch.pc.cmake ${CMAKE_CURRENT_BINARY_DIR}/libsipwitch.pc)

    install(FILES ${CMAKE_CURRENT_BINARY_DIR}/libsipwitch.pc DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig)
endif()

install(FILES   ${sipwitch_conf} DESTINATION ${CMAKE_INSTALL_SYSCONFDIR})
install(FILES   ${sipwitch_xml} DESTINATION ${CMAKE_INSTALL_SYSCONFDIR}/sipwitch.d)
install(FILES   ${sipwitch_man1} DESTINATION ${CMAKE_INSTALL_MANDIR}/man1)
install(FILES   ${sipwitch_man8} DESTINATION ${CMAKE_INSTALL_MANDIR}/man8)
install(FILES   ${runtime_inc}  DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/sipwitch)
install(TARGETS sipwitch DESTINATION ${CMAKE_INSTALL_LIBDIR})
install(TARGETS sipwitch-control DESTINATION ${CMAKE_INSTALL_BINDIR})
install(TARGETS sipwitch-cgi DESTINATION ${CMAKE_INSTALL_CGIBINDIR})

if(SYSTEM_SETUID)
    install(TARGETS sipwitch-passwd DESTINATION ${CMAKE_INSTALL_BINDIR} PERMISSIONS SETUID)
else()
    install(TARGETS sipwitch-passwd DESTINATION ${CMAKE_INSTALL_BINDIR})
endif()

if(CMAKE_GENERATOR MATCHES "Unix Makefiles")
    add_custom_target(cleandist
        WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}"
        COMMAND rm -f "${CMAKE_CURRENT_BINARY_DIR}/${PACKAGE}[-_]*.gz"
        COMMAND rm -f "${CMAKE_CURRENT_BINARY_DIR}/${PACKAGE}_*.dsc"
        COMMAND rm -f "${CMAKE_CURRENT_BINARY_DIR}/${PACKAGE}-*.rpm"
        COMMAND rm -f "${CMAKE_CURRENT_BINARY_DIR}/${PACKAGE}[-_]*.deb"
        COMMAND rm -f "${CMAKE_CURRENT_BINARY_DIR}/*${PACKAGE}*[-_]*.deb"
        COMMAND rm -f "${CMAKE_CURRENT_BINARY_DIR}/${PACKAGE}-*.zip"
        COMMAND rm -f "${CMAKE_CURRENT_BINARY_DIR}/*${PACKAGE}*.changes"
    )

    add_custom_target(srpm
        WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}"
        COMMAND rm -f *.rpm
        COMMAND rpmbuild  -bs --nodeps --define "_sourcedir ." --define "_srcrpmdir ." --sign ${PACKAGE}.spec
    )

    add_custom_target(dist
        DEPENDS cleandist
        WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}"
        COMMAND git archive --format tar --output="${CMAKE_CURRENT_BINARY_DIR}/${PACKAGE}-${VERSION}.tar" --prefix="${PACKAGE}-${VERSION}/" HEAD
        COMMAND gzip "${CMAKE_CURRENT_BINARY_DIR}/${PACKAGE}-${VERSION}.tar"
    )

    add_custom_target(uninstall
        WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}"
        COMMAND xargs <install_manifest.txt
    )

    if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/packaging/debian/")
        add_custom_target(deb
            WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}"
            COMMAND rm -f *.deb *.debian.tar.gz *.dsc *.changes
            COMMAND cape-source --sign ${PACKAGE}-${VERSION}.tar.gz packaging
        )
    endif()
endif()


