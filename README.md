# flow

Standalone extraction of [FoundationDB](https://github.com/apple/foundationdb)'s `flow` asynchronous runtime, together
with the part of `fdbrpc` that depends only on flow, so they can be built and used without the rest of FoundationDB.

Flow is a single-threaded, cooperative C++20-coroutine runtime. All I/O (network, disk, clock, randomness) sits behind
the `INetwork` interface, so the same code runs on the real event loop (`Net2`) or in a deterministic simulator
(`Sim2`).

## Layout

| Path | Contents |
|---|---|
| `flow/` | Core runtime: `Future`/`Promise`/`PromiseStream`, coroutines, `Net2` event loop, tracing, knobs, serialization, TLS, arenas. Tutorial: [flow/README.md](flow/README.md) |
| `fdbrpc/` | Flow-only subset of fdbrpc: `Sim2` simulator, `Net2FileSystem` and AsyncFile layers (EIO/KAIO/cached/non-durable), `FlowTransport`, HTTP, failure monitor, replication policies |
| `contrib/` | Bundled third-party code: crc32, libb64, SimpleOpt, stacktrace, folly_memcpy, md5, rapidjson |
| `examples/` | Minimal runnable example |
| `cmake/` | `FindCoroutines`, `FindJinja2` |

`flow` does not depend on `fdbrpc`; `fdbrpc` depends on `flow`.

## Requirements

- C++20 compiler (tested: GCC 13, Linux x86_64) and CMake >= 3.24
- Boost (header-only parts plus `filesystem`)
- OpenSSL
- Python 3 with Jinja2 (a venv is created automatically if Jinja2 is missing); used to generate `ProtocolVersion.h`
- `fmt`: found with `find_package`, otherwise fetched with FetchContent

## Build and test

```sh
cmake -G Ninja -B build .            # add -Dfmt_DIR=... / -DCMAKE_PREFIX_PATH=... for non-system Boost/fmt
ninja -C build
ctest --test-dir build               # flow_test, fdbrpc_test (several minutes)
```

Targets: `flow`, `fdbrpc` (static libraries), `flowlinktest`, `mkcert`, `flow_test`, `fdbrpc_test`, `hello`.
Set `-DFLOW_BUILD_TESTS=OFF` to skip the test runners.

- `flow_test` runs flow's unit tests. It installs no file system, so file-based tests are skipped.
- `fdbrpc_test` runs fdbrpc and flow tests, including simulation tests with `Net2FileSystem`.
- Both link their libraries with `WHOLE_ARCHIVE` so that `TEST_CASE`s in otherwise unreferenced objects are kept.

## Usage

```cpp
#include "flow/flow.h"
#include "flow/network.h"
#include "flow/TLSConfig.h"

Future<int> addLater(int a, int b) {
	co_await delay(0.1);
	co_return a + b;
}

Future<Void> run() {
	int sum = co_await addLater(1, 2);
	printf("1 + 2 = %d\n", sum);
	g_network->stop();
}

int main() {
	g_network = newNet2(TLSConfig());
	Future<Void> f = run();
	g_network->run();
}
```

See [examples/hello.cpp](examples/hello.cpp). Link `flow` (and `fdbrpc` for the simulator or file system). If you rely
on `TEST_CASE` registration or other static initializers, link with whole-archive as the test targets do.

Pitfalls worth knowing (details in the tutorial and in FoundationDB's `design/coroutines.md`):

- Everything runs on one thread; a blocking call stalls the whole event loop.
- Dropping the last `Future` cancels the coroutine; never swallow `actor_cancelled`.
- `co_await` is not allowed inside a `catch` block.
- `StringRef` does not own its bytes; use `Standalone<>` when it must outlive its arena.

## Differences from FoundationDB

Not included: gRPC support (`FlowGrpc*`, `FileTransfer`), the Swift build and `.swift` sources, benchmarks, the
coverage tool, and the rest of fdbrpc's tests/protos. The `testReadLines` generator test in `flow/CoroTests.cpp` is
skipped when no file system is installed. Only Linux + GCC has been built and tested; other platforms' code paths are
carried over unchanged.

## License

Apache License 2.0, see [LICENSE](LICENSE). Code originates from the FoundationDB project.
