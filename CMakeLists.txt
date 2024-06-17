cmake_minimum_required(VERSION 3.11)

project(QuickJS LANGUAGES C)

#
# build QuickJS as a shared library
#

set(QUICKJS_VERSION 2024-01-13)
set(CONFIG_BIGNUM ON CACHE BOOL "Include the code for BigFloat/BigDecimal and math mode.")

add_library(quickjs SHARED
    ${CMAKE_CURRENT_LIST_DIR}/cutils.c
    ${CMAKE_CURRENT_LIST_DIR}/libbf.c
    ${CMAKE_CURRENT_LIST_DIR}/libregexp.c
    ${CMAKE_CURRENT_LIST_DIR}/libunicode.c
    ${CMAKE_CURRENT_LIST_DIR}/quickjs.c
    ${CMAKE_CURRENT_LIST_DIR}/quickjs-libc.c
)

target_include_directories(quickjs PUBLIC ${CMAKE_CURRENT_LIST_DIR})

#
# TODO:
# https://cmake.org/cmake/help/latest/guide/importing-exporting/index.html
#

if(${CONFIG_BIGNUM})
    target_compile_definitions(quickjs PUBLIC CONFIG_BIGNUM)
endif()

file(STRINGS ${CMAKE_CURRENT_LIST_DIR}/VERSION QUICKJS_VERSION)

target_compile_definitions(quickjs PUBLIC CONFIG_VERSION="${QUICKJS_VERSION}")
target_link_libraries(quickjs PRIVATE pthread PRIVATE dl)

#
# build qjsc as an executable
#

add_executable(qjsc qjsc.c)
target_compile_definitions(qjsc PRIVATE CONFIG_VERSION="${QUICKJS_VERSION}")
target_link_libraries(qjsc PRIVATE quickjs)

#
# build qjscalc with qjsc
#

add_custom_target(qjscalc ALL
    BYPRODUCTS ${CMAKE_CURRENT_BINARY_DIR}/qjscalc.c
    COMMAND qjsc -fbignum -c -o ${CMAKE_CURRENT_BINARY_DIR}/qjscalc.c -m ${CMAKE_CURRENT_SOURCE_DIR}/qjscalc.js)

#
# build repl with qjsc
#

add_custom_target(repl ALL
    BYPRODUCTS ${CMAKE_CURRENT_BINARY_DIR}/repl.c
    COMMAND qjsc -c -o ${CMAKE_CURRENT_BINARY_DIR}/repl.c -m ${CMAKE_CURRENT_SOURCE_DIR}/repl.js
)

#
# build qjs
#

add_executable(qjs
    qjs.c
    qjscalc.c
    repl.c
)

target_compile_definitions(qjs PRIVATE CONFIG_VERSION="${QUICKJS_VERSION}")
target_link_libraries(qjs PRIVATE quickjs)

#
# qjscalc is apparently just a softlink???
# TODO: the command will fail if the user isn't on windows 10 and has developer
# mode enabled. the entire build process will return with an error if this fails
#

add_custom_command(
    TARGET qjs
    POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E create_symlink  $<TARGET_FILE_DIR:qjs>/qjs${CMAKE_EXECUTABLE_SUFFIX}  $<TARGET_FILE_DIR:qjs>/qjscalc${CMAKE_EXECUTABLE_SUFFIX}
)
