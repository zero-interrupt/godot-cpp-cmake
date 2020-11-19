cmake_minimum_required(VERSION 3.10.0 FATAL_ERROR)
project(GodotCpp VERSION 1.0.1 LANGUAGES CXX)

option(ENABLE_GODOT_FORCE_GEN "Always regenerate Godot bindings" OFF)

find_package(PythonInterp QUIET)
if(PYTHONINTERP_FOUND)
	set(DEFAULT_PYTHON_PATH "${PYTHON_EXECUTABLE}")
	message(STATUS "Found Python interpreter: ${PYTHON_EXECUTABLE}")
else()
	set(DEFAULT_PYTHON_PATH "python")
endif()

set(PYTHON_COMMAND
	"${DEFAULT_PYTHON_PATH}"
	CACHE STRING
	"Path to Python interpreter")

# By default, use api.json from godot_headers repo
set(GODOT_API_JSON
	"${PROJECT_SOURCE_DIR}/godot-cpp/godot_headers/api.json"
	CACHE FILEPATH
	"Path to godot_api.json file for bindings generator")

if (NOT EXISTS "${GODOT_API_JSON}" OR IS_DIRECTORY "${GODOT_API_JSON}")
	message(FATAL_ERROR "Path GODOT_API_JSON to api.json file is required")
endif()

if(NOT CMAKE_BUILD_TYPE)
	set(CMAKE_BUILD_TYPE "RelWithDebInfo")
endif()

message(STATUS "CMAKE_BUILD_TYPE: ${CMAKE_BUILD_TYPE}")
message(STATUS "GODOT_API_JSON: ${GODOT_API_JSON}")
message(STATUS "PYTHON_COMMAND: ${PYTHON_COMMAND}")

# Default Android settings to follow Godot's build
# Use $ANDROID_NDK_ROOT/build/cmake/android.toolchain.cmake toolchain
if(CMAKE_SYSTEM_NAME STREQUAL Android)
	set(CMAKE_TOOLCHAIN_FILE "$ENV{ANDROID_NDK_ROOT}/build/cmake/android.toolchain.cmake" CACHE FILEPATH "Android NDK's CMake toolchain file")
	set(ANDROID_PLATFORM android-28 CACHE STRING "Android target platform API level")
	set(ANDROID_ABI armeabi-v7a CACHE STRING "Android target ABI architecture")
	set(ANDROID_TOOLCHAIN clang)
	set(ANDROID_STL c++_shared)
endif()

# Settings for Emscripten build
# Use $EMSCRIPTEN/cmake/Modules/Platform/Emscripten.cmake toolchain
if(CMAKE_SYSTEM_NAME STREQUAL Emscripten)
	set(CMAKE_TOOLCHAIN_FILE "$ENV{EMSCRIPTEN}/cmake/Modules/Platform/Emscripten.cmake" CACHE FILEPATH "Emscripten's CMake toolchain file")
endif()

# Force MSVC to statically linking with CRT
if (MSVC)
	string(REPLACE "/MD" "/MT" CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}")
	string(REPLACE "/MD" "/MT" CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG}")
	string(REPLACE "/MD" "/MT" CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE}")
endif()


if(NOT EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/godot-cpp/include/gen/__icalls.hpp")
	set(REGEN_BINDINGS ON)
	message(STATUS "Godot C++ binding generation - ENABLED")
elseif(ENABLE_GODOT_FORCE_GEN)
	set(REGEN_BINDINGS ON)
	message(STATUS "Godot C++ binding generation - FORCED")
else()
	set(REGEN_BINDINGS OFF)
	message(STATUS "Godot C++ binding generation - SKIPPED")
endif()

# Generate Godot's GDNative <-> C++ bindings using JSON file
if(REGEN_BINDINGS)
	message(STATUS "Generating Godot C++ binding source files...")
	set(GENERATOR_COMMAND "${CMAKE_CURRENT_SOURCE_DIR}/generate.py")
	set(GENERATOR_COMMAND "${GENERATOR_COMMAND}" "${GODOT_API_JSON}")
	execute_process(
		COMMAND ${PYTHON_COMMAND} ${GENERATOR_COMMAND}
		WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/godot-cpp"
	)
endif()

# List all binding C++ source files
file(GLOB GDNATIVECPP_CORE_SOURCE_FILES "godot-cpp/src/core/*.cpp")
file(GLOB GDNATIVECPP_GENERATED_SOURCE_FILES "godot-cpp/src/gen/*.cpp")

# Create library target
add_library(GodotCppLibrary STATIC
	${GDNATIVECPP_CORE_SOURCE_FILES}
	${GDNATIVECPP_GENERATED_SOURCE_FILES}
)

target_include_directories(GodotCppLibrary PUBLIC
	godot-cpp/godot_headers
	godot-cpp/include
	godot-cpp/include/gen
	godot-cpp/include/core
)

target_compile_features(GodotCppLibrary PUBLIC
	cxx_std_14
)

if(CMAKE_SYSTEM_NAME STREQUAL Emscripten)
	set_target_properties(GodotCppLibrary PROPERTIES
		OUTPUT_NAME_MINSIZEREL "godot-cpp.${LIB_SUFFIX_RELEASE}"
	)
endif()
