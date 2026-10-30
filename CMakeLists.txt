cmake_minimum_required(VERSION 3.16)
cmake_policy(SET CMP0091 NEW)
cmake_policy(SET CMP0042 NEW)

project(webbluetooth)

if (APPLE)
    set(CMAKE_OSX_ARCHITECTURES "x86_64;arm64" CACHE STRING "macOS architecture" FORCE)
    set(CMAKE_OSX_DEPLOYMENT_TARGET "10.15" CACHE STRING "macOS target" FORCE)
elseif(WIN32)
    set(CMAKE_SYSTEM_VERSION "10.0.22000.0" CACHE STRING "Windows version" FORCE)
endif()

# Apply local fixes to the SimpleBLE checkout (skipped when already applied).
# simpleble-keep-device-valid: without it, a device that disconnects is never reported by a scan again
# until the process restarts, because BlueZ removing one interface (Battery1) made the device look removed.
file(GLOB SIMPLEBLE_PATCHES ${CMAKE_CURRENT_SOURCE_DIR}/patches/*.patch)
foreach(PATCH ${SIMPLEBLE_PATCHES})
    execute_process(COMMAND git apply --reverse --check ${PATCH}
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/SimpleBLE
        RESULT_VARIABLE PATCH_ALREADY_APPLIED
        OUTPUT_QUIET ERROR_QUIET
    )
    if (NOT PATCH_ALREADY_APPLIED EQUAL 0)
        execute_process(COMMAND git apply ${PATCH}
            WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/SimpleBLE
            RESULT_VARIABLE PATCH_RESULT
        )
        if (NOT PATCH_RESULT EQUAL 0)
            message(FATAL_ERROR "Could not apply ${PATCH} to SimpleBLE")
        endif()
    endif()
endforeach()

add_subdirectory(SimpleBLE/simplecble)

# Add Node bindings.
execute_process(COMMAND node -p "require('node-addon-api').include_dir"
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    OUTPUT_VARIABLE NODE_ADDON_API_DIR
    OUTPUT_STRIP_TRAILING_WHITESPACE
)
add_library(simpleble-node SHARED
    lib/adapter.h
    lib/adapter.cpp
    lib/bindings.cpp
    lib/peripheral.h
    lib/peripheral.cpp
    ${CMAKE_JS_SRC}
)
target_include_directories(simpleble-node PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR}/lib
    ${CMAKE_JS_INC}
    ${NODE_ADDON_API_DIR}
)
target_link_libraries(simpleble-node PRIVATE simpleble-c ${CMAKE_JS_LIB})
target_compile_definitions(simpleble-node PRIVATE NAPI_VERSION=6)
set_target_properties(simpleble-node PROPERTIES
    OUTPUT_NAME "simpleble"
    CXX_STANDARD 17
    PREFIX ""
    SUFFIX ".node"
)

if (MSVC AND CMAKE_JS_NODELIB_DEF AND CMAKE_JS_NODELIB_TARGET)
    # Generate node.lib
    execute_process(COMMAND ${CMAKE_AR} /def:${CMAKE_JS_NODELIB_DEF} /out:${CMAKE_JS_NODELIB_TARGET} ${CMAKE_STATIC_LINKER_FLAGS})
endif()
