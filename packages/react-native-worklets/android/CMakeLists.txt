project(Worklets)
cmake_minimum_required(VERSION 3.8)

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

set(CMAKE_CXX_STANDARD 20)

# default CMAKE_CXX_FLAGS: "-g -DANDROID -fdata-sections -ffunction-sections
# -funwind-tables -fstack-protector-strong -no-canonical-prefixes
# -D_FORTIFY_SOURCE=2 -Wformat -Werror=format-security -fstack-protector-all"
include("${REACT_NATIVE_DIR}/ReactAndroid/cmake-utils/folly-flags.cmake")
add_compile_options(${folly_FLAGS})

string(APPEND CMAKE_CXX_FLAGS
       " -DREACT_NATIVE_MINOR_VERSION=${REACT_NATIVE_MINOR_VERSION} \
  -DREANIMATED_VERSION=${REANIMATED_VERSION}")

string(APPEND CMAKE_CXX_FLAGS
       " -fexceptions -fno-omit-frame-pointer -frtti -fstack-protector-all \
  -std=c++${CMAKE_CXX_STANDARD} -Wall -Werror")

if(${IS_NEW_ARCHITECTURE_ENABLED})
  string(APPEND CMAKE_CXX_FLAGS " -DRCT_NEW_ARCH_ENABLED")
endif()

if(${IS_REANIMATED_EXAMPLE_APP})
  string(APPEND CMAKE_CXX_FLAGS " -DIS_REANIMATED_EXAMPLE_APP -Wpedantic")
endif()

if(NOT ${CMAKE_BUILD_TYPE} MATCHES "Debug")
  string(APPEND CMAKE_CXX_FLAGS " -DNDEBUG")
endif()

if(${JS_RUNTIME} STREQUAL "hermes")
  string(APPEND CMAKE_CXX_FLAGS " -DJS_RUNTIME_HERMES=1")
elseif(${JS_RUNTIME} STREQUAL "jsc")
  string(APPEND CMAKE_CXX_FLAGS " -DJS_RUNTIME_JSC=1")
elseif(${JS_RUNTIME} STREQUAL "v8")
  string(APPEND CMAKE_CXX_FLAGS " -DJS_RUNTIME_V8=1")
else()
  message(FATAL_ERROR "Unknown JS runtime ${JS_RUNTIME}.")
endif()

# Resolves "CMake Warning: Manually-specified variables were not used by the
# project" when any of the following variables is not used in some build
# configuration.
set(IGNORE_ME "${JS_RUNTIME_DIR}")

set(BUILD_DIR "${CMAKE_SOURCE_DIR}/build")
set(ANDROID_CPP_DIR "${CMAKE_SOURCE_DIR}/src/main/cpp")
set(COMMON_CPP_DIR "${CMAKE_SOURCE_DIR}/../Common/cpp")

file(GLOB_RECURSE WORKLETS_COMMON_CPP_SOURCES CONFIGURE_DEPENDS
     "${COMMON_CPP_DIR}/worklets/*.cpp")
file(GLOB_RECURSE WORKLETS_ANDROID_CPP_SOURCES CONFIGURE_DEPENDS
     "${ANDROID_CPP_DIR}/worklets/*.cpp")

# Consume shared libraries and headers from prefabs
find_package(fbjni REQUIRED CONFIG)
find_package(ReactAndroid REQUIRED CONFIG)

if(${JS_RUNTIME} STREQUAL "hermes")
  find_package(hermes-engine REQUIRED CONFIG)
endif()

add_library(worklets SHARED ${WORKLETS_COMMON_CPP_SOURCES}
                            ${WORKLETS_ANDROID_CPP_SOURCES})

# includes
target_include_directories(worklets PUBLIC "${COMMON_CPP_DIR}"
                                           "${ANDROID_CPP_DIR}")

target_include_directories(
  worklets
  PRIVATE "${REACT_NATIVE_DIR}/ReactCommon"
          "${REACT_NATIVE_DIR}/ReactAndroid/src/main/jni/react/turbomodule"
          "${REACT_NATIVE_DIR}/ReactCommon/react/nativemodule/core/ReactCommon"
          "${REACT_NATIVE_DIR}/ReactCommon/callinvoker"
          "${REACT_NATIVE_DIR}/ReactCommon/runtimeexecutor")

if(${IS_NEW_ARCHITECTURE_ENABLED})
  target_include_directories(
    worklets
    PRIVATE
      "${REACT_NATIVE_DIR}/ReactCommon/yoga"
      "${REACT_NATIVE_DIR}/ReactCommon/react/renderer/graphics/platform/cxx")

  if(ReactAndroid_VERSION_MINOR LESS 76)
    target_link_libraries(
      worklets ReactAndroid::fabricjni ReactAndroid::react_debug
      ReactAndroid::react_render_core
      ReactAndroid::react_render_componentregistry ReactAndroid::rrc_view)
  endif()
endif()

# build shared lib
set_target_properties(worklets PROPERTIES LINKER_LANGUAGE CXX)

target_link_libraries(worklets log ReactAndroid::jsi fbjni::fbjni)

if(ReactAndroid_VERSION_MINOR GREATER_EQUAL 76)
  target_link_libraries(worklets ReactAndroid::reactnative)
else()
  target_link_libraries(
    worklets ReactAndroid::react_nativemodule_core ReactAndroid::folly_runtime
    ReactAndroid::glog ReactAndroid::reactnativejni)
endif()

if(${JS_RUNTIME} STREQUAL "hermes")
  target_link_libraries(worklets hermes-engine::libhermes)

  if(${HERMES_ENABLE_DEBUGGER})
    string(APPEND CMAKE_CXX_FLAGS " -DHERMES_ENABLE_DEBUGGER=1")
    if(ReactAndroid_VERSION_MINOR GREATER_EQUAL 76)
      target_link_libraries(worklets ReactAndroid::hermestooling)
    else()
      target_link_libraries(worklets ReactAndroid::hermes_executor)
    endif()
  endif()
elseif(${JS_RUNTIME} STREQUAL "jsc")
  if(ReactAndroid_VERSION_MINOR GREATER_EQUAL 76)
    target_link_libraries(worklets ReactAndroid::jsctooling)
  else()
    target_link_libraries(worklets ReactAndroid::jscexecutor)
  endif()
elseif(${JS_RUNTIME} STREQUAL "v8")
  # TODO: Refactor this when adding support for newest V8
  target_include_directories(worklets PRIVATE "${JS_RUNTIME_DIR}/src")
  file(GLOB V8_SO_DIR "${JS_RUNTIME_DIR}/android/build/intermediates/\
    library_jni/**/jni/${ANDROID_ABI}")
  find_library(
    V8EXECUTOR_LIB v8executor
    PATHS ${V8_SO_DIR}
    NO_DEFAULT_PATH NO_CMAKE_FIND_ROOT_PATH)
  target_link_libraries(worklets ${V8EXECUTOR_LIB})
endif()
