cmake_minimum_required(VERSION 3.12)
project(mruby-lite)

# configure
option(MRB_USE_METHOD_T_STRUCT "add this on machines that use higher bits of function pointers" OFF)
option(MRB_NAN_BOXING "represent mrb_value in boxed double" OFF)
option(MRB_WORD_BOXING "represent mrb_value as a word (natural unit of data for the processor)" OFF)
option(MRB_NO_BOXING "represent mrb_value as a struct; occupies 2 words" OFF)
option(MRB_UTF8_STRING "string class to handle UTF-8 encoding" ON)
option(MRB_GC_FIXED_ARENA "fixed size GC arena" ON)
option(MRB_USE_DEBUG_HOOK "enable hooks for debugger" OFF)
set(MRUBY_CONF_CMAKE "#define MRB_NO_PRESYM\n")
if (MRB_USE_METHOD_T_STRUCT)
    set(MRUBY_CONF_CMAKE "${MRUBY_CONF_CMAKE}#define MRB_USE_METHOD_T_STRUCT\n")
endif()
if (MRB_NAN_BOXING)
    set(MRUBY_CONF_CMAKE "${MRUBY_CONF_CMAKE}#define MRB_NAN_BOXING\n")
endif()
if (MRB_WORD_BOXING)
    set(MRUBY_CONF_CMAKE "${MRUBY_CONF_CMAKE}#define MRB_WORD_BOXING\n")
endif()
if (MRB_NO_BOXING)
    set(MRUBY_CONF_CMAKE "${MRUBY_CONF_CMAKE}#define MRB_NO_BOXING\n")
endif()
if (MRB_UTF8_STRING)
    set(MRUBY_CONF_CMAKE "${MRUBY_CONF_CMAKE}#define MRB_UTF8_STRING\n")
endif()
if (MRB_GC_FIXED_ARENA)
    set(MRUBY_CONF_CMAKE "${MRUBY_CONF_CMAKE}#define MRB_GC_FIXED_ARENA\n")
endif()
if (MRB_USE_DEBUG_HOOK)
    set(MRUBY_CONF_CMAKE "${MRUBY_CONF_CMAKE}#define MRB_USE_DEBUG_HOOK\n")
endif()

# libmruby_core
configure_file(include/mrbconf.h.in ${CMAKE_BINARY_DIR}/mrbconf.h @ONLY)
file(GLOB_RECURSE MRB_CORE_SOURCES CONFIGURE_DEPENDS src/*.c**)
add_library(mruby_core STATIC ${MRB_CORE_SOURCES})
include_directories(include)
include_directories(${CMAKE_BINARY_DIR})
add_compile_options(-fPIC)

# mrbc
if (CMAKE_CROSSCOMPILING)
    set(MRUBY_MRBC "" CACHE FILEPATH "mrbc executable")
else()
    file(GLOB_RECURSE MRB_COMPILER_SOURCES CONFIGURE_DEPENDS mrbgems/mruby-compiler/core/*.c**)
    file(GLOB_RECURSE MRB_MRBC_SOURCES CONFIGURE_DEPENDS mrbgems/mruby-bin-mrbc/tools/*.c**)
    add_executable(mrbc ${MRB_COMPILER_SOURCES} ${MRB_MRBC_SOURCES})
    target_link_libraries(mrbc mruby_core)
    add_dependencies(mrbc mruby_core)
    set(MRUBY_MRBC ${CMAKE_BINARY_DIR}/mrbc CACHE FILEPATH "mrbc executable")
endif()

# base mrblib
file(GLOB_RECURSE MRB_MRBLIB_BASE_SOURCES CONFIGURE_DEPENDS mrblib/*.rb)
add_custom_target(mrblib_base ALL
    COMMAND ${MRUBY_MRBC} -B mrblib_base -o ${CMAKE_BINARY_DIR}/mrblib_base.h ${MRB_MRBLIB_BASE_SOURCES}
    BYPRODUCTS ${CMAKE_BINARY_DIR}/mrblib_base.h
    COMMENT "build mrblib base"
)