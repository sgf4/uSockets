cmake_minimum_required(VERSION 3.14)

project(usockets)

option(WITH_SSL "Enables OpenSSL 1.1+ support" OFF)
option(WITH_WOLFSSL "Enables WolfSSL 4.2.0 support (mutually exclusive with OpenSSL)" OFF)
option(WITH_BORINGSSL "Enables BoringSSL support, linked statically (preferred over OpenSSL)" OFF)
option(WITH_IO_URING "Builds with io_uring as event-loop and network implementation" OFF)
option(WITH_LIBUV "Builds with libuv as event-loop" OFF)
option(WITH_ASIO "Builds with boot asio event-loop" OFF)
option(WITH_LTO "Builds with LTO (Windows does not support it)" OFF)
option(WITH_QUIC "Builds with Quic support" OFF)
option(WITH_EXAMPLES "Builds with examples" OFF)


if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE "Release")
elseif(CMAKE_BUILD_TYPE STREQUAL "Debug")
  set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} -fsanitize=address")
  set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -fsanitize=address")
  set(CMAKE_EXE_LINKER_FLAGS_DEBUG "${CMAKE_EXE_LINKER_FLAGS_DEBUG} -fsanitize=address")
endif()

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

file(GLOB_RECURSE SRC_FILES usockets/*.c usockets/*.h usockets/*.cpp)

add_library(
  usockets STATIC ${SRC_FILES}
)
add_library(usockets::usockets ALIAS usockets)

target_include_directories(usockets PRIVATE usockets)
target_include_directories(usockets INTERFACE ${PROJECT_SOURCE_DIR})

if(${WITH_SSL})
  add_compile_definitions(LIBUS_USE_OPENSSL)
  find_package(OpenSSL REQUIRED)
  target_link_libraries(usockets PRIVATE OpenSSL::SSL OpenSSL::Crypto)
elseif(${WITH_BORINGSSL})
  # no way to skip tests :/
  add_subdirectory(boringssl)
  target_link_libraries(usockets PRIVATE ssl)
elseif(${WITH_WOLFSSL})
  add_compile_definitions(LIBUS_USE_WOLFSSL OPENSSL_ALL)
  find_package(WolfSSL REQUIRED)
  find_package(Threads REQUIRED)
  target_link_libraries(usockets PRIVATE wolfssl)
  target_include_directories(usockets PUBLIC ${wolfssl_INCLUDE_DIRECTORIES})
else()
  add_compile_definitions(LIBUS_NO_SSL)
endif()

if(${WITH_IO_URING})
  add_compile_definitions(LIBUS_USE_IO_URING)
endif()

if(${WITH_LIBUV})
  add_compile_definitions(LIBUS_USE_LIBUV)
  set(THREADS_PREFER_PTHREAD_FLAG ON)
  find_package(Threads REQUIRED)
  target_link_libraries(usockets PRIVATE uv)
endif()

if(${WITH_ASIO})
  add_compile_definitions(LIBUS_USE_ASIO)
  find_package(Boost REQUIRED COMPONENTS system)
  target_link_libraries(usockets PRIVATE Boost::asio pthread)
endif()

if(${WITH_QUIC})

endif()

if(${WITH_LTO})
  set_property(TARGET usockets PROPERTY INTERPROCEDURAL_OPTIMIZATION ON)
endif()

if(${WITH_EXAMPLES})
  add_subdirectory(examples)
endif()

install(TARGETS usockets DESTINATION lib)
install(FILES usockets/usockets.h usockets/quic.h DESTINATION include/usockets)
