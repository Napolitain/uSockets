cmake_minimum_required(VERSION 3.10)
project(uSockets C CXX)

# Set C and C++ standards
set(CMAKE_C_STANDARD 11)
set(CMAKE_CXX_STANDARD 17)

# Options matching Makefile configurations
option(WITH_LTO "Enable Link Time Optimization" ON)
option(WITH_OPENSSL "Enable OpenSSL 1.1+ support" OFF)
option(WITH_BORINGSSL "Enable BoringSSL support (preferred over OpenSSL)" OFF)
option(WITH_WOLFSSL "Enable WolfSSL 4.2.0 support" OFF)
option(WITH_QUIC "Enable QUIC support" OFF)
option(WITH_IO_URING "Enable io_uring support" OFF)
option(WITH_LIBUV "Enable libuv as event-loop" OFF)
option(WITH_ASIO "Enable Boost ASIO as event-loop" OFF)
option(WITH_GCD "Enable libdispatch (GCD) as event-loop" OFF)
option(WITH_ASAN "Enable AddressSanitizer" OFF)

# Build type default to Release
if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release)
endif()

# Collect source files
set(USOCKETS_SOURCES
    src/bsd.c
    src/context.c
    src/loop.c
    src/socket.c
    src/udp.c
    src/quic.c
    src/eventing/epoll_kqueue.c
    src/eventing/gcd.c
    src/eventing/libuv.c
    src/crypto/openssl.c
    src/io_uring/io_context.c
    src/io_uring/io_loop.c
    src/io_uring/io_socket.c
)

# Create static library
add_library(uSockets STATIC ${USOCKETS_SOURCES})

# Include directories
target_include_directories(uSockets PUBLIC src)

# Compile flags
if(WITH_LTO AND NOT WIN32)
    set_target_properties(uSockets PROPERTIES INTERPROCEDURAL_OPTIMIZATION TRUE)
    target_compile_options(uSockets PRIVATE -flto)
endif()

# Optimization flags
target_compile_options(uSockets PRIVATE -O3)

# SSL/TLS Configuration
if(WITH_BORINGSSL)
    target_compile_definitions(uSockets PUBLIC LIBUS_USE_OPENSSL)
    target_include_directories(uSockets PUBLIC boringssl/include)
    target_compile_options(uSockets PRIVATE -pthread)
    target_link_libraries(uSockets PUBLIC pthread)
    target_link_libraries(uSockets PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/boringssl/build/ssl/libssl.a)
    target_link_libraries(uSockets PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/boringssl/build/crypto/libcrypto.a)
    target_link_libraries(uSockets PUBLIC stdc++)
    
    # Add C++ crypto sources for BoringSSL
    target_sources(uSockets PRIVATE src/crypto/sni_tree.cpp)
    set_source_files_properties(src/crypto/sni_tree.cpp PROPERTIES COMPILE_FLAGS "-std=c++17 -flto -O3")
elseif(WITH_OPENSSL)
    target_compile_definitions(uSockets PUBLIC LIBUS_USE_OPENSSL)
    target_link_libraries(uSockets PUBLIC ssl crypto stdc++)
    
    # Add C++ crypto sources for OpenSSL
    target_sources(uSockets PRIVATE src/crypto/sni_tree.cpp)
    set_source_files_properties(src/crypto/sni_tree.cpp PROPERTIES COMPILE_FLAGS "-std=c++17 -flto -O3")
elseif(WITH_WOLFSSL)
    target_compile_definitions(uSockets PUBLIC LIBUS_USE_WOLFSSL)
    target_include_directories(uSockets PUBLIC /usr/local/include)
    target_link_directories(uSockets PUBLIC /usr/local/lib)
    target_link_libraries(uSockets PUBLIC wolfssl)
else()
    target_compile_definitions(uSockets PUBLIC LIBUS_NO_SSL)
endif()

# Event-loop configurations
if(WITH_IO_URING)
    target_compile_definitions(uSockets PUBLIC LIBUS_USE_IO_URING)
    target_link_libraries(uSockets PUBLIC /usr/lib/liburing.a)
endif()

if(WITH_LIBUV)
    target_compile_definitions(uSockets PUBLIC LIBUS_USE_LIBUV)
    target_link_libraries(uSockets PUBLIC uv)
endif()

if(WITH_ASIO)
    target_compile_definitions(uSockets PUBLIC LIBUS_USE_ASIO)
    target_compile_options(uSockets PRIVATE -pthread)
    target_link_libraries(uSockets PUBLIC stdc++ pthread)
    
    # Add ASIO C++ source
    target_sources(uSockets PRIVATE src/eventing/asio.cpp)
    set_source_files_properties(src/eventing/asio.cpp PROPERTIES COMPILE_FLAGS "-std=c++14 -flto -O3 -pthread -DLIBUS_USE_ASIO")
endif()

if(WITH_GCD)
    target_compile_definitions(uSockets PUBLIC LIBUS_USE_GCD)
    target_link_libraries(uSockets PUBLIC "-framework CoreFoundation")
endif()

# QUIC configuration
if(WITH_QUIC)
    target_compile_definitions(uSockets PUBLIC LIBUS_USE_QUIC)
    target_include_directories(uSockets PUBLIC lsquic/include)
    target_compile_options(uSockets PRIVATE -pthread)
    target_link_libraries(uSockets PUBLIC pthread z m ${CMAKE_CURRENT_SOURCE_DIR}/lsquic/src/liblsquic/liblsquic.a)
endif()

# AddressSanitizer
if(WITH_ASAN)
    target_compile_options(uSockets PRIVATE -fsanitize=address -g)
    target_link_options(uSockets PUBLIC -fsanitize=address)
endif()

# Examples
option(BUILD_EXAMPLES "Build example programs" OFF)

if(BUILD_EXAMPLES)
    # Find all example C files
    file(GLOB EXAMPLE_SOURCES "examples/*.c")
    
    foreach(EXAMPLE_SOURCE ${EXAMPLE_SOURCES})
        get_filename_component(EXAMPLE_NAME ${EXAMPLE_SOURCE} NAME_WE)
        add_executable(${EXAMPLE_NAME} ${EXAMPLE_SOURCE})
        target_link_libraries(${EXAMPLE_NAME} uSockets)
        target_compile_options(${EXAMPLE_NAME} PRIVATE -O3)
        
        # Add QUIC specific linking for examples
        if(WITH_QUIC)
            target_link_libraries(${EXAMPLE_NAME} pthread z m ${CMAKE_CURRENT_SOURCE_DIR}/lsquic/src/liblsquic/liblsquic.a)
        endif()
    endforeach()
endif()

# BoringSSL build target
add_custom_target(boringssl
    COMMAND cd boringssl && mkdir -p build && cd build && cmake -DCMAKE_BUILD_TYPE=Release .. && make -j8
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    COMMENT "Building BoringSSL"
)
