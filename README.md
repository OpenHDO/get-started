# Get started with OpenHDO

OpenHDO is a local-first control plane for devices, computers, and
automations. This guide gets the current foundation running in a few minutes.

## 1. Read the map

Start with the [project overview](https://github.com/OpenHDO/about). The
important boundary is simple: `openhdo-server` owns state and policy;
`openhdo-linker` stays near hardware and owns device transports.

## 2. Run the server foundation

Requirements: CMake 3.24+, a C++20 compiler, and Ninja or another supported
generator.

```bash
git clone https://github.com/OpenHDO/server.git
cd server
cmake --preset dev
cmake --build --preset dev
ctest --preset dev
build/dev/openhdo-server --check
```

On Windows without Ninja, use `dev-mingw` in the three commands above.

## 3. Run the panel shell

```bash
cd web
npm ci
npm run dev
```

The panel is currently a UI foundation. The live API is introduced after the
server HTTP/WebSocket contract is implemented.

## 4. Try the Python protocol SDK

```bash
cd python
python -m unittest discover -s tests -v
```

`LinkerManifest` can create a validated `link.register` message without a
third-party dependency. Device libraries and network transport stay in the
application that embeds the SDK.

## Contributing

Pick the repository that owns the boundary you are changing. Keep public
messages versioned, add an example and compatibility test for new message
types, and run the checks documented in the server repository.

Useful links:

- [server](https://github.com/OpenHDO/server)
- [server technical docs](https://github.com/OpenHDO/server/blob/master/DOCS.md)
- [SDK](https://github.com/OpenHDO/sdk)
- [Linker](https://github.com/OpenHDO/linker)
