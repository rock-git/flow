# flow

Standalone copy of FoundationDB's `flow/` async runtime (futures, coroutines, Net2 event loop, tracing, serialization).
Docs: [flow/README.md](flow/README.md).

## Build

Requires: C++20 compiler, CMake >= 3.24, Boost (with `filesystem`), OpenSSL, Python3 + Jinja2 (auto-installed into a venv if missing).
`fmt` is found via `find_package` or fetched with FetchContent.

    cmake -G Ninja -B build . && ninja -C build
    cd build && ctest        # runs flow's unit tests (flow_test)

`fdbrpc/` is the part of FoundationDB's fdbrpc that depends only on flow: `Sim2` (deterministic simulator, `startNewSimulator`),
`Net2FileSystem` + AsyncFile layers (EIO/KAIO/cached/non-durable), FlowTransport, HTTP, failure monitor, replication policies.
`flow_test` runs flow's unit tests (no file system); `fdbrpc_test` runs fdbrpc + flow tests, including simulation.

Not included vs. FoundationDB: gRPC support (`FlowGrpc*`, `FileTransfer`), the Swift build, benchmarks.
