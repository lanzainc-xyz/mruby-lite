cmake_minimum_required(VERSION 3.3)
project(mruby-lite)

# configure
option(MRB_BUILD_TOOLS "build tools in mrbgems" ON)

# mruby configure
option(MRB_USE_METHOD_T_STRUCT "add this on machines that use higher bits of function pointers" OFF)
option(MRB_NAN_BOXING "represent mrb_value in boxed double" OFF)
option(MRB_WORD_BOXING "represent mrb_value as a word (natural unit of data for the processor)" OFF)
option(MRB_NO_BOXING "represent mrb_value as a struct; occupies 2 words" OFF)
option(MRB_UTF8_STRING "string class to handle UTF-8 encoding" ON)
option(MRB_GC_FIXED_ARENA "fixed size GC arena" ON)
option(MRB_USE_DEBUG_HOOK "enable hooks for debugger" OFF)
option(MRB_USE_CXX_ABI "use c++ ABI for mruby" OFF)
option(MRB_USE_CXX_EXCEPTION "use c++ exception to implement try/throw" OFF)
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
if (MRB_USE_CXX_ABI)
    set(MRUBY_CONF_CMAKE "${MRUBY_CONF_CMAKE}#define MRB_USE_CXX_ABI\n")
endif()
if (MRB_USE_CXX_EXCEPTION)
    set(MRUBY_CONF_CMAKE "${MRUBY_CONF_CMAKE}#define MRB_USE_CXX_EXCEPTION\n")
endif()

# libmruby_core
configure_file(include/mrbconf.h.in ${CMAKE_CURRENT_BINARY_DIR}/mrbconf.h @ONLY)
if (MRB_USE_CXX_ABI)
    file(GLOB MRB_CORE_C_SOURCES RELATIVE ${CMAKE_CURRENT_SOURCE_DIR}/src ${CMAKE_CURRENT_SOURCE_DIR}/src/*.c**)
    file(GLOB MRB_CORE_HEADERS RELATIVE ${CMAKE_CURRENT_SOURCE_DIR}/src ${CMAKE_CURRENT_SOURCE_DIR}/src/*.h**)
    set(MRB_AS_CXX vm.c error.c gc.c)
    set(MRB_CORE_SOURCES "")
    foreach(MRB_C_SOURCE ${MRB_CORE_C_SOURCES})
        file(COPY ${CMAKE_CURRENT_SOURCE_DIR}/src/${MRB_C_SOURCE} DESTINATION ${CMAKE_CURRENT_BINARY_DIR}/src)
        if (${MRB_C_SOURCE} IN_LIST MRB_AS_CXX)
            configure_file(${CMAKE_CURRENT_SOURCE_DIR}/c_as_cxx.cpp.in ${CMAKE_CURRENT_BINARY_DIR}/src/${MRB_C_SOURCE}.cpp @ONLY)
            set(MRB_CORE_SOURCES ${MRB_CORE_SOURCES};${CMAKE_CURRENT_BINARY_DIR}/src/${MRB_C_SOURCE}.cpp)
        else()
            set(MRB_CORE_SOURCES ${MRB_CORE_SOURCES};${CMAKE_CURRENT_BINARY_DIR}/src/${MRB_C_SOURCE})
        endif()
    endforeach()
    foreach(MRB_CORE_HEADER ${MRB_CORE_HEADERS})
        file(COPY ${CMAKE_CURRENT_SOURCE_DIR}/src/${MRB_CORE_HEADER} DESTINATION ${CMAKE_CURRENT_BINARY_DIR}/src)
    endforeach()
else()
    file(GLOB_RECURSE MRB_CORE_SOURCES CONFIGURE_DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/src/*.c**)
endif()
add_library(mruby_core STATIC ${MRB_CORE_SOURCES})
target_compile_options (mruby_core PRIVATE -fPIC)
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/include)
include_directories(${CMAKE_CURRENT_BINARY_DIR})
if (CMAKE_CROSSCOMPILING)
    set(MRUBY_MRBC_PATH "" CACHE FILEPATH "the parent folder of mrbc executable")
else()
    set(MRUBY_MRBC_PATH ${CMAKE_BINARY_DIR}/bin CACHE FILEPATH "the parent folder of mrbc executable")
endif()

# base mrblib
file(GLOB_RECURSE MRB_MRBLIB_BASE_SOURCES CONFIGURE_DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/mrblib/*.rb)
add_custom_target(mrblib_base ALL)
add_custom_command(TARGET mrblib_base
    PRE_BUILD
    DEPENDS ${MRB_MRBLIB_BASE_SOURCES}
    COMMAND ${MRUBY_MRBC_PATH}/mrbc -B mrblib_base -o ${CMAKE_CURRENT_BINARY_DIR}/mrblib_base.h ${MRB_MRBLIB_BASE_SOURCES}
    BYPRODUCTS ${CMAKE_CURRENT_BINARY_DIR}/mrblib_base.h
    # COMMENT "build mrblib base"
    )
set(MRB_MRBLIB_INCLUDE "#include \"mrblib_base.h\"\n")

# build mrbgems
set(MRB_DEPENDENCIES "")
set(MRB_MGEM_LIST_STDLIB "mruby-bin-mirb;mruby-bin-strip;mruby-compar-ext;mruby-enum-ext;mruby-string-ext;mruby-numeric-ext;mruby-array-ext;mruby-hash-ext;mruby-range-ext;mruby-proc-ext;mruby-symbol-ext;mruby-object-ext;mruby-objectspace;mruby-set;mruby-fiber;mruby-enumerator;mruby-enum-lazy;mruby-toplevel-ext;mruby-kernel-ext;mruby-class-ext;mruby-metaprog;mruby-method;mruby-eval;mruby-error;mruby-errno;mruby-pack;mruby-struct;mruby-sprintf;mruby-random;mruby-math;mruby-exit;mruby-enum-chain" CACHE STRING "built-in mrb mgems list")
set(MRB_MGEM_LIST_CUSTOM "" CACHE STRING "custom mrb mgems list")
set(MRB_MGEM_LIST_CUSTOM_PATH "" CACHE PATH "custom mrb mgems path")
set(MRB_MGEM_LIST ${MRB_MGEM_LIST_STDLIB})
set(MRB_MGEM_LIST ${MRB_MGEM_LIST} ${MRB_MGEM_LIST_CUSTOM})
set(MRB_SOURCES "")
set(MRB_TOOL_TARGETS "")
foreach(MRB_MGEM_NAME ${MRB_MGEM_LIST})
    string(REPLACE "-" "_" MRB_MGEM_SIGN ${MRB_MGEM_NAME})
    file(GLOB_RECURSE MRB_MGEM_CORE_SOURCES CONFIGURE_DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/mrbgems/${MRB_MGEM_NAME}/core/*.c** ${MRB_MGEM_LIST_CUSTOM_PATH}/${MRB_MGEM_NAME}/core/*.c**)
    file(GLOB_RECURSE MRB_MGEM_SOURCES CONFIGURE_DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/mrbgems/${MRB_MGEM_NAME}/src/*.c** ${MRB_MGEM_LIST_CUSTOM_PATH}/${MRB_MGEM_NAME}/src/*.c**)
    file(GLOB_RECURSE MRB_MGEM_RBSOURCES CONFIGURE_DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/mrbgems/${MRB_MGEM_NAME}/mrblib/*.rb ${MRB_MGEM_LIST_CUSTOM_PATH}/${MRB_MGEM_NAME}/mrblib/*.rb)
    # gem c/cpp sources
    list(LENGTH MRB_MGEM_SOURCES MRB_MGEM_SOURCES_COUNT)
    if (MRB_MGEM_SOURCES_COUNT GREATER 0)
        set(MRB_MRBLIB_DECLARES "${MRB_MRBLIB_DECLARES}extern void mrb_${MRB_MGEM_SIGN}_gem_init(mrb_state* mrb);\n")
        set(MRB_MRBLIB_DECLARES "${MRB_MRBLIB_DECLARES}extern void mrb_${MRB_MGEM_SIGN}_gem_final(mrb_state* mrb);\n")
        set(MRB_MRBLIB_INIT_EXPR "${MRB_MRBLIB_INIT_EXPR}MRB_GEMINIT_PROCESS(mrb_${MRB_MGEM_SIGN}_gem_init, ${MRB_MGEM_SIGN});\n")
        set(MRB_MRBLIB_FINAL_EXPR "${MRB_MRBLIB_FINAL_EXPR}mrb_${MRB_MGEM_SIGN}_gem_final(mrb);\n")
    endif()
    # gem rb sources
    list(LENGTH MRB_MGEM_RBSOURCES MRB_MGEM_RBSOURCES_COUNT)
    if (MRB_MGEM_RBSOURCES_COUNT GREATER 0)
        set(MRB_MRBLIB_DECLARES "${MRB_MRBLIB_DECLARES}extern void mrb_init_mrbgem_${MRB_MGEM_SIGN}_mrblib(mrb_state* mrb);\n")
        set(MRB_MRBLIB_DECLARES "${MRB_MRBLIB_DECLARES}#define MRB_${MRB_MGEM_SIGN}_init mrb_init_mrbgem_${MRB_MGEM_SIGN}_mrblib(mrb)\n")
        add_custom_target(mrbgem_${MRB_MGEM_SIGN} ALL)
        add_custom_command(TARGET mrbgem_${MRB_MGEM_SIGN}
            PRE_BUILD
            COMMAND ${MRUBY_MRBC_PATH}/mrbc -B mrbgem_${MRB_MGEM_SIGN} -o ${CMAKE_CURRENT_BINARY_DIR}/mrbgem_${MRB_MGEM_SIGN}.h ${MRB_MGEM_RBSOURCES}
            BYPRODUCTS ${CMAKE_CURRENT_BINARY_DIR}/mrbgem_${MRB_MGEM_SIGN}.h
            # COMMENT "build mrbgem_${MRB_MGEM_SIGN}"
            )
        add_dependencies(mrbgem_${MRB_MGEM_SIGN} mrblib_base)
        set(MRB_DEPENDENCIES ${MRB_DEPENDENCIES};mrbgem_${MRB_MGEM_SIGN})
        configure_file(mrbgems/mrbgem.c.in ${CMAKE_CURRENT_BINARY_DIR}/mrbgem_${MRB_MGEM_SIGN}.c @ONLY)
        set(MRB_SOURCES ${MRB_SOURCES};${CMAKE_CURRENT_BINARY_DIR}/mrbgem_${MRB_MGEM_SIGN}.c)
    else()
        set(MRB_MRBLIB_DECLARES "${MRB_MRBLIB_DECLARES}#define MRB_${MRB_MGEM_SIGN}_init\n")
    endif()
    # add include dir
    include_directories(${CMAKE_CURRENT_SOURCE_DIR}/mrbgems/${MRB_MGEM_NAME}/include)
    include_directories(${MRB_MGEM_LIST_CUSTOM_PATH}/${MRB_MGEM_NAME}/include)
    # add sources
    set(MRB_CORE_SOURCES ${MRB_CORE_SOURCES};${MRB_MGEM_CORE_SOURCES})
    set(MRB_SOURCES ${MRB_SOURCES};${MRB_MGEM_SOURCES})
    # add tool target
    if (MRB_BUILD_TOOLS AND EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/mrbgems/${MRB_MGEM_NAME}/tools)
        file(GLOB TOOL_NAMES RELATIVE ${CMAKE_CURRENT_SOURCE_DIR}/mrbgems/${MRB_MGEM_NAME}/tools ${CMAKE_CURRENT_SOURCE_DIR}/mrbgems/${MRB_MGEM_NAME}/tools/*)
        foreach(TOOL_NAME ${TOOL_NAMES})
            file(GLOB_RECURSE TOOL_SOURCES CONFIGURE_DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/mrbgems/${MRB_MGEM_NAME}/tools/${TOOL_NAME}/*.c**)
            set(TOOL_TARGET_NAME ${MRB_MGEM_NAME}_${TOOL_NAME})
            add_executable(${TOOL_TARGET_NAME} ${TOOL_SOURCES})
            set_target_properties(${TOOL_TARGET_NAME} PROPERTIES OUTPUT_NAME ${TOOL_NAME})
            set(MRB_TOOL_TARGETS ${MRB_TOOL_TARGETS};${TOOL_TARGET_NAME})
        endforeach()
    endif()
endforeach()
configure_file(mrbgems/mrbgems.c.in ${CMAKE_CURRENT_BINARY_DIR}/mrbgems.c @ONLY)
set(MRB_SOURCES ${MRB_CORE_SOURCES};${MRB_SOURCES};${CMAKE_CURRENT_BINARY_DIR}/mrbgems.c)

# build compiler
file(GLOB_RECURSE MRB_COMPILER_SOURCES CONFIGURE_DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/mrbgems/mruby-compiler/core/*.c**)
file(GLOB MRB_COMPILER_C_SOURCES RELATIVE ${CMAKE_CURRENT_SOURCE_DIR}/mrbgems/mruby-compiler/core ${CMAKE_CURRENT_SOURCE_DIR}/mrbgems/mruby-compiler/core/*.c)
file(GLOB MRB_COMPILER_SOURCES RELATIVE ${CMAKE_CURRENT_SOURCE_DIR}/mrbgems/mruby-compiler/core ${CMAKE_CURRENT_SOURCE_DIR}/mrbgems/mruby-compiler/core/*.*)
foreach(MRB_COMPILER_SOURCE ${MRB_COMPILER_SOURCES})
    file(COPY ${CMAKE_CURRENT_SOURCE_DIR}/mrbgems/mruby-compiler/core/${MRB_COMPILER_SOURCE} DESTINATION ${CMAKE_CURRENT_BINARY_DIR}/mrbgems/mruby-compiler/core)
endforeach()
set(MRB_COMPILER_SOURCES "")
foreach(MRB_C_SOURCE ${MRB_COMPILER_C_SOURCES})
    if (MRB_USE_CXX_ABI)
        configure_file(${CMAKE_CURRENT_SOURCE_DIR}/c_as_cxx.cpp.in ${CMAKE_CURRENT_BINARY_DIR}/mrbgems/mruby-compiler/core/${MRB_C_SOURCE}.cpp @ONLY)
        set(MRB_COMPILER_SOURCES ${MRB_COMPILER_SOURCES};${CMAKE_CURRENT_BINARY_DIR}/mrbgems/mruby-compiler/core/${MRB_C_SOURCE}.cpp)
    else()
        set(MRB_COMPILER_SOURCES ${MRB_COMPILER_SOURCES};${CMAKE_CURRENT_BINARY_DIR}/mrbgems/mruby-compiler/core/${MRB_C_SOURCE})
    endif()
endforeach()

# build mruby
configure_file(mrblib/mrblib.c.in ${CMAKE_CURRENT_BINARY_DIR}/mrblib.c @ONLY)
add_library(mruby STATIC ${MRB_SOURCES} ${CMAKE_CURRENT_BINARY_DIR}/mrblib.c ${MRB_COMPILER_SOURCES})
target_compile_options (mruby PRIVATE -fPIC)
foreach(MRB_DEPENDENCY ${MRB_DEPENDENCIES})
    add_dependencies(mruby ${MRB_DEPENDENCY})
endforeach()

# tools dependencies
foreach(MRB_TOOL_TARGET ${MRB_TOOL_TARGETS})
    target_link_libraries(${MRB_TOOL_TARGET} mruby)
    add_dependencies(${MRB_TOOL_TARGET} mruby)
endforeach()

# mrbc
if (NOT CMAKE_CROSSCOMPILING)
    file(GLOB_RECURSE MRB_MRBC_SOURCES CONFIGURE_DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/mrbgems/mruby-bin-mrbc/tools/*.c**)
    add_executable(mrbc ${MRB_MRBC_SOURCES} ${MRB_COMPILER_SOURCES})
    target_link_libraries(mrbc mruby_core)
    add_dependencies(mrbc mruby_core)
    add_dependencies(mrblib_base mrbc)
    add_dependencies(mruby mrbc)
    add_custom_command(TARGET mrbc
        POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E make_directory ${MRUBY_MRBC_PATH}
        )
    add_custom_command(TARGET mrbc
        POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy $<TARGET_FILE:mrbc> ${MRUBY_MRBC_PATH})
endif()