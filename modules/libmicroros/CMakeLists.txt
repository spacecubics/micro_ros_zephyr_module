# SPDX-License-Identifier: Apache-2.0

if(CONFIG_MICROROS)

set(MICROROS_DIR ${ZEPHYR_CURRENT_MODULE_DIR})

if(CMAKE_GENERATOR STREQUAL "Unix Makefiles")
set(submake "$(MAKE)")
else()
set(submake "make")
endif()

# micro-ROS library

zephyr_get_include_directories_for_lang_as_string(       C includes)
zephyr_get_system_include_directories_for_lang_as_string(C system_includes)
zephyr_get_compile_definitions_for_lang_as_string(       C definitions)
zephyr_get_compile_options_for_lang_as_string(           C options)

zephyr_get_include_directories_for_lang_as_string(       CXX includes_cxx)
zephyr_get_system_include_directories_for_lang_as_string(CXX system_includes_cxx)
zephyr_get_compile_definitions_for_lang_as_string(       CXX definitions_cxx)
zephyr_get_compile_options_for_lang_as_string(           CXX options_cxx)

set(external_project_cflags
    "${includes} ${definitions} ${options} ${system_includes}"
    )

set(external_project_cxxflags
    "${includes_cxx} ${definitions_cxx} ${options_cxx} ${system_includes_cxx}"
    )

include(ExternalProject)

externalproject_add(libmicroros_project
    PREFIX     ${CMAKE_BINARY_DIR}/libmicroros
    SOURCE_DIR ${MICROROS_DIR}
    TMP_DIR    ${CMAKE_BINARY_DIR}/libmicroros/tmp
    STAMP_DIR  ${CMAKE_BINARY_DIR}/libmicroros/stamp
    BINARY_DIR ${CMAKE_BINARY_DIR}/libmicroros/build
    CONFIGURE_COMMAND ""
    BUILD_COMMAND
        ${submake} -C ${MICROROS_DIR} -f libmicroros.mk
            X_CFLAGS=${external_project_cflags}
            X_CXXFLAGS=${external_project_cxxflags}
            X_CC=${CMAKE_C_COMPILER}
            X_AR=${CMAKE_AR}
            X_RANLIB=${CMAKE_RANLIB}
            X_CXX=${CMAKE_CXX_COMPILER}
            COMPONENT_PATH=${MICROROS_DIR}
            ZEPHYR_BASE=${ZEPHYR_BASE}
            PROJECT_BINARY_DIR=${PROJECT_BINARY_DIR}
            WORK_DIR=${CMAKE_BINARY_DIR}/libmicroros
    INSTALL_COMMAND ""
    BUILD_BYPRODUCTS ${CMAKE_BINARY_DIR}/libmicroros/libmicroros.a
    DEPENDS zephyr_interface
    )

zephyr_library_import(libmicroros ${CMAKE_BINARY_DIR}/libmicroros/libmicroros.a)


zephyr_interface_library_named(microros)
add_dependencies(microros libmicroros)
add_dependencies(microros libmicroros_project)
target_include_directories(microros INTERFACE ${CMAKE_BINARY_DIR}/libmicroros/install/include)

execute_process(
    COMMAND
        make -f libmicroros.mk get_package_names
            COMPONENT_PATH=${MICROROS_DIR}
    WORKING_DIRECTORY
        ${MICROROS_DIR}
    OUTPUT_VARIABLE
        INCLUDE_ROS2_PACKAGES
    OUTPUT_STRIP_TRAILING_WHITESPACE
    )

foreach(pkg ${INCLUDE_ROS2_PACKAGES})
    target_include_directories(microros INTERFACE ${CMAKE_BINARY_DIR}/libmicroros/install/include/${pkg})
endforeach()

# micro-ROS transport library

zephyr_library_named(microros_transports)
if(CONFIG_MICROROS_TRANSPORT_SERIAL)
    set(MICROROS_TRANSPORT_DIR ${MICROROS_DIR}/microros_transports/serial)
elseif(CONFIG_MICROROS_TRANSPORT_SERIAL_USB)
    set(MICROROS_TRANSPORT_DIR ${MICROROS_DIR}/microros_transports/serial-usb)
elseif(CONFIG_MICROROS_TRANSPORT_UDP)
    set(MICROROS_TRANSPORT_DIR ${MICROROS_DIR}/microros_transports/udp)
    if(NOT CONFIG_SOC_POSIX)
        add_dependencies(microros_transports posix_subsys)
        target_include_directories(microros_transports PRIVATE ${ZEPHYR_BASE}/include/zephyr/posix)
    endif()
else()
    message(FATAL_ERROR Please set a micro-ROS transport)
endif()

zephyr_include_directories(${CMAKE_BINARY_DIR}/libmicroros/install/include)
zephyr_include_directories(${MICROROS_TRANSPORT_DIR})

zephyr_library_sources(
   ${MICROROS_TRANSPORT_DIR}/microros_transports.c
)

add_dependencies(microros microros_transports)
add_dependencies(microros_transports libmicroros_project)
add_dependencies(microros_transports libmicroros)

endif()
