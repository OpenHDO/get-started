# Get started with OpenHDO

OpenHDO is a local-first control plane for devices, computers, and
automations. This guide gets the current foundation running and shows what
is, and is not, connected yet.

The current server repository provides three independent success paths:

- the C++ server foundation and CLI smoke checks;
- the React/TypeScript panel shell;
- the dependency-free Python protocol SDK.

The server executable is not a long-running HTTP/WebSocket service yet. The
panel therefore runs as a UI foundation with sample data; it does not connect
to the server process. The SDK produces and validates protocol messages
locally. These are expected boundaries in the current release, not setup
failures.

## Prerequisites

Install these before starting:

- Git;
- CMake 3.24 or newer;
- a C++20 compiler and a CMake-supported generator;
- Node.js 20 and npm for the panel (Node 20 is the server CI baseline);
- Python 3.11 or newer for the SDK.

Platform notes:

- Linux: install GCC or Clang, CMake, and Ninja or Make through your
  distribution. The dev preset uses the default generator.
- macOS: install Xcode Command Line Tools (xcode-select --install), then
  install CMake and use the dev preset.
- Windows with Visual Studio: install the Desktop development with C++
  workload and CMake. Run the commands from PowerShell or a Developer PowerShell.
- Windows with MinGW-w64: make sure g++ and mingw32-make are on PATH; use
  the dev-mingw preset below.

Check the tools before cloning if you are unsure:

~~~text
git --version
cmake --version
node --version
npm --version
python --version
~~~

On Windows, py -3 --version and py -3 -m ... can replace python -m ...
if the Python launcher is installed but python is not on PATH.

## 1. Get the repositories

Clone the server repository. It contains the executable, panel, SDK, and
versioned contracts used by the first-run paths:

~~~bash
git clone https://github.com/OpenHDO/server.git server
~~~

The project overview and the separate hardware connector are documented in
the [about repository](https://github.com/OpenHDO/about) and the
[OpenHDO Linker repository](https://github.com/OpenHDO/linker).

## 2. Run the server foundation

Run these commands from the server repository root. The dev preset enables
the C++ tests and selects a Debug build.

### Linux or macOS

~~~bash
cd server
cmake --preset dev
cmake --build --preset dev
ctest --preset dev
./build/dev/openhdo-server --check
./build/dev/ohdocli --version
~~~

The first success checkpoint is output like “ok openhdo-server 0.1.0
(protocol v1)”. The command exits after the check; it does not start a daemon.

### Windows with Visual Studio

The normal Visual Studio generator is multi-config, so the executable is
under the Debug directory:

~~~powershell
git clone https://github.com/OpenHDO/server.git server
Set-Location server
cmake --preset dev
cmake --build --preset dev
ctest --preset dev
& .\build\dev\Debug\openhdo-server.exe --check
& .\build\dev\Debug\ohdocli.exe --version
~~~

If CMake selected a single-config generator instead, use these executable
paths:

~~~powershell
& .\build\dev\openhdo-server.exe --check
& .\build\dev\ohdocli.exe --version
~~~

### Windows with MinGW-w64

The repository includes a MinGW-specific preset using the MinGW Makefiles
generator:

~~~powershell
git clone https://github.com/OpenHDO/server.git server
Set-Location server
cmake --preset dev-mingw
cmake --build --preset dev-mingw
ctest --preset dev-mingw
& .\build\dev-mingw\openhdo-server.exe --check
& .\build\dev-mingw\ohdocli.exe --version
~~~

The server also has a ci preset. Use it when reproducing the server CI
configuration: cmake --preset ci, cmake --build --preset ci, and
ctest --preset ci.

## 3. Open the panel shell

Leave the server repository root in one terminal, then use another terminal:

~~~bash
cd server/web
npm ci
npm run build
npm run dev
~~~

PowerShell equivalent:

~~~powershell
Set-Location server\web
npm ci
npm run build
npm run dev
~~~

Open [http://localhost:4173/](http://localhost:4173/) when Vite prints its
local URL. The port is defined in web/vite.config.ts and is strict. Stop
the development server with Ctrl+C.

The panel is currently a static foundation preview. Its device, activity,
Linker, and flow values are UI sample data; the current server has no live
API or WebSocket endpoint for the panel to call.

## 4. Exercise the Python SDK

The SDK is stdlib-only, so no package installation is required for this
checkout. Run its tests from the SDK directory:

~~~bash
cd server/python
python -m unittest discover -s tests -v
~~~

PowerShell, using the Windows Python launcher when needed:

~~~powershell
Set-Location server\python
py -3 -m unittest discover -s tests -v
~~~

The command should finish with OK. To see the first versioned registration
message, run this from the same directory:

~~~bash
python -c 'from openhdo_sdk import LinkerManifest; print(LinkerManifest("linker.get-started", "0.1.0", "Get Started Linker", ("test",)).registration("linker.get-started").to_json())'
~~~

Envelope validates protocol version 1, message identity, timestamps, and
payload shape. LinkerManifest.registration() creates the link.register
message described by the server’s
[contracts/v1](https://github.com/OpenHDO/server/tree/master/contracts/v1).
Transport, credentials, reconnect policy, and device libraries remain the
responsibility of the application embedding the SDK.

## Troubleshooting

### CMake cannot find a compiler

Confirm that a C++20 compiler is installed and visible to the shell. On
Windows, use Developer PowerShell for Visual Studio, or check both
g++ --version and mingw32-make --version for MinGW. Re-run the matching
preset from the server root.

### The CMake preset is not recognized

Run cmake --version. The server requires CMake 3.24 or newer. Also make
sure the command is being run from the server repository root, where
CMakePresets.json exists.

### The server executable path does not exist on Windows

Visual Studio builds place the executable in a configuration directory. Look
under build\dev\Debug\ first. MinGW uses build\dev-mingw\. A single-config
generator uses build\dev\. Do not expect --check to keep a process
running.

### npm ci fails

Run it from server/web, not the server root. That directory contains the
tracked package-lock.json. If the panel port is already in use, stop the
process using port 4173 before running npm run dev; the Vite configuration
intentionally fails instead of silently changing ports.

### The panel shows sample devices or does not reflect the server check

That is expected in this release. openhdo-server --check is a one-shot
foundation check, and the panel has no live server transport yet.

### Python cannot import openhdo_sdk

Run the test or example from server/python, and check that Python is 3.11
or newer. The SDK intentionally does not require pip install for this local
first-run path.

## Contributing

1. Choose the repository and directory that own the boundary: server state and
   orchestration live in server/; the panel is server/web/; the reference
   SDK is server/python/; hardware transports belong in the separate Linker
   repository.
2. Create a focused branch, make the smallest change that fits the existing
   boundary, and keep public messages under server/contracts/v1/.
3. For a new public message, add its schema or payload definition, a
   representative example, and a compatibility test before using it.
4. Run the checks for the area you changed. For a full server check:

   ~~~bash
   cmake --preset ci
   cmake --build --preset ci
   ctest --preset ci
   cd web && npm ci && npm run build
   cd ../python && python -m unittest discover -s tests -v
   ~~~

5. Commit with a Conventional Commit message, push the branch, and open a
   pull request in the repository that owns the change.

The server’s detailed rules are in its
[CONTRIBUTING.md](https://github.com/OpenHDO/server/blob/master/CONTRIBUTING.md)
and [technical documentation](https://github.com/OpenHDO/server/blob/master/DOCS.md).

For a documentation-only change in this repository:

~~~bash
git switch -c docs/improve-onboarding
git diff --check
git add README.md
git commit -m "docs: improve onboarding"
git push -u origin docs/improve-onboarding
~~~

## Repository map

- [Project overview](https://github.com/OpenHDO/about)
- [Server](https://github.com/OpenHDO/server)
- [Server technical docs](https://github.com/OpenHDO/server/blob/master/DOCS.md)
- [Versioned server contracts](https://github.com/OpenHDO/server/tree/master/contracts/v1)
- [Python SDK in the server repository](https://github.com/OpenHDO/server/tree/master/python)
- [Linker](https://github.com/OpenHDO/linker)
