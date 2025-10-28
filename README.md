# Optimized TCP, TLS, QUIC & HTTP3 transports

µSockets is the non-blocking, thread-per-CPU foundation library used by [µWebSockets](https://github.com/uNetworking/uWebSockets). It provides optimized networking - using the same opaque API (programming interface) across all supported transports, event-loops and platforms (QUIC is work-in-progress, so is io_uring).

<a href="https://github.com/uNetworking/uSockets/releases"><img src="https://img.shields.io/github/v/release/uNetworking/uSockets"></a>

## Write code once
Based on µSockets, apps like µWebSockets can run on many platforms, over many transports and with many event-loops - all without any code changes or special execution paths. Moving data over TCP is just as easy as over QUIC.

Hit `make examples` to get started.

## Building with Make
The traditional build system uses Make:
```bash
make                    # Build the uSockets.a static library
make examples          # Build all example programs
WITH_OPENSSL=1 make    # Build with OpenSSL support
```

## Building with CMake
CMake is also supported with identical functionality:
```bash
mkdir build && cd build
cmake ..                                        # Configure basic build
cmake -DBUILD_EXAMPLES=ON ..                   # Configure with examples
cmake -DWITH_OPENSSL=ON -DBUILD_EXAMPLES=ON .. # Configure with OpenSSL and examples
make                                           # Build
```

Available CMake options:
- `WITH_LTO` - Enable Link Time Optimization (default: ON, except Windows)
- `WITH_OPENSSL` - Enable OpenSSL 1.1+ support
- `WITH_BORINGSSL` - Enable BoringSSL support (preferred over OpenSSL)
- `WITH_WOLFSSL` - Enable WolfSSL 4.2.0 support
- `WITH_QUIC` - Enable QUIC support
- `WITH_IO_URING` - Enable io_uring support
- `WITH_LIBUV` - Enable libuv as event-loop
- `WITH_ASIO` - Enable Boost ASIO as event-loop
- `WITH_GCD` - Enable libdispatch (GCD) as event-loop
- `WITH_ASAN` - Enable AddressSanitizer
- `BUILD_EXAMPLES` - Build example programs (default: OFF)

## Lightweight or featureful
In its minimal, TCP-only, configuration µSockets has no dependencies other than the very OS kernel and compiles down to a tiny binary. In its full configuration it depends on BoringSSL, lsquic and potentially some event-loop library.

Here are some configurations; WITH_IO_URING, WITH_LIBUV, WITH_ASIO, WITH_GCD, WITH_ASAN, WITH_QUIC, WITH_BORINGSSL, WITH_OPENSSL, WITH_WOLFSSL.

## Fast & stable
µWebSockets itself is known to have run with outstanding performance and stability since 2016. This thanks to, among other factors, the speed and stability of µSockets. We fuzz and randomly "hammer test" the library as part of security & stability testing done in the µWebSockets project.
