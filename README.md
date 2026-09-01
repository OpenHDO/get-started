# Get started: native Python Linker for one real Wi-Fi RGB lamp

This guide is for one physical Wi-Fi RGB lamp and one PC running the local
OpenHDO processes. Wi-Fi is only the network transport; OpenHDO does not
define one universal Wi-Fi lamp protocol.

The exact vendor and model are a hard prerequisite. Do not start with a
product family, a guessed endpoint, or a generic Wi-Fi implementation. Record
the full model, hardware revision, firmware, region, and the vendor’s local
API documentation before choosing a driver path.

## Current status

Working in the checked-in repositories:

- the C++ server can build and run a one-shot configuration/readiness check;
- the server’s versioned v1 envelope and Linker manifest contract exist;
- the Python SDK validates envelopes and creates link.register messages;
- the Linker repository defines the ownership boundary for hardware access.

Not shipped:

- a native Python Linker process or entry point;
- a vendor-specific Wi-Fi driver for any lamp;
- a device discovery or pairing implementation;
- an OpenHDO wire adapter for Linker traffic;
- a public RGB command/state contract;
- a long-running OpenHDO HTTP/WebSocket service.

Therefore the commands below verify the current server and Python SDK building
blocks. They do not claim that OpenHDO can control the lamp yet.

## Prerequisites

### Required lamp identity

Fill this out from the physical device and vendor documentation:

| Field | Required value |
| --- | --- |
| Vendor | Exact vendor/legal product owner |
| Model | Full model or part number, not only the family name |
| Hardware revision | Printed revision, if present |
| Firmware | Exact firmware version |
| Region | Region/account/API variant |
| Local API | Official local API or SDK documentation and version |
| Authentication | Token, API key, pairing code, or vendor-specific session |
| Discovery | Vendor-defined discovery method or a configured device address |
| RGB semantics | RGB range, brightness range, power behavior, and readback support |

If the vendor exposes only a cloud API or only an undocumented mobile-app
protocol, this is not yet a verified native local Wi-Fi path. Do not substitute
a guessed protocol.

### PC and network

- Python 3.11 or newer;
- Git;
- a local network shared by the PC and lamp;
- the lamp provisioned onto that network according to the vendor procedure;
- firewall rules that allow the vendor’s documented local traffic;
- vendor documentation for timeouts, rate limits, reconnects, and errors.

The current Python SDK is stdlib-only. A real driver may need a
vendor-specific dependency, but that choice cannot be made before the exact
model and API are known.

### Credentials

Use the credential type and scope required by the vendor API. Keep secrets
outside Git, README files, shell history, source code, and structured logs.
Prefer a permissions-restricted local secret store or file, and document
rotation and expiry behavior.

The current OpenHDO Python SDK has no credential store and no Linker
configuration schema. Do not invent environment variables or config keys and
assume the SDK will read them; the native Linker must define and validate its
own configuration as part of its implementation.

## 1. Server preflight

This checks the local OpenHDO foundation before any driver work. It is not a
lamp health check and does not open a network listener.

### Linux or macOS

~~~bash
git clone https://github.com/OpenHDO/server.git server
cd server
cmake --preset dev
cmake --build --preset dev
ctest --preset dev
./build/dev/openhdo-server --check
./build/dev/ohdocli --version
~~~

### Windows with Visual Studio

The normal Visual Studio generator is multi-config:

~~~powershell
git clone https://github.com/OpenHDO/server.git server
Set-Location server
cmake --preset dev
cmake --build --preset dev
ctest --preset dev
& .\build\dev\Debug\openhdo-server.exe --check
& .\build\dev\Debug\ohdocli.exe --version
~~~

If CMake selected a single-config generator, use:

~~~powershell
& .\build\dev\openhdo-server.exe --check
& .\build\dev\ohdocli.exe --version
~~~

### Windows with MinGW-w64

~~~powershell
git clone https://github.com/OpenHDO/server.git server
Set-Location server
cmake --preset dev-mingw
cmake --build --preset dev-mingw
ctest --preset dev-mingw
& .\build\dev-mingw\openhdo-server.exe --check
& .\build\dev-mingw\ohdocli.exe --version
~~~

A successful server check includes an info-level foundation.ready JSON record
and an ok openhdo-server 0.1.0 (protocol v1) line. The process exits after
the check. It is not the Linker process and it does not listen for lamp
traffic.

The server’s supported local settings are OPENHDO_CONFIG_VERSION,
OPENHDO_INSTANCE_NAME, and OPENHDO_LOG_LEVEL. They configure the server
foundation only. They do not configure a Python driver or lamp credentials.

## 2. Native Python Linker path

The only current Python implementation is the reference SDK under
server/python. It provides Envelope, ProtocolError, and LinkerManifest; it
does not provide a process, socket client, vendor driver, discovery, pairing,
or health endpoint.

Run its existing checks:

~~~bash
cd server/python
python -m unittest discover -s tests -v
~~~

PowerShell:

~~~powershell
Set-Location server\python
py -3 -m unittest discover -s tests -v
~~~

The test command should finish with OK. LinkerManifest.registration() can
serialize the current v1 link.register message, but the repository has no
transport that sends it to the server. The message contains v equal to 1,
type link.register, a unique id, a timestamp, a source, and the manifest
payload.

A real native Python Linker must add, in this order:

1. A validated configuration for the exact vendor/model, device address or
   discovery settings, credential reference, timeouts, retry policy, and
   Linker identity. No such configuration schema exists in the current SDK.
2. A vendor-specific driver that uses the documented local API to authenticate,
   discover or address the lamp, pair if required, read state, set power,
   set RGB color, set brightness, and normalize vendor errors.
3. A process entry point and lifecycle that starts only after configuration
   validation, never logs secrets, and reports useful health information.
4. A server-facing adapter that emits only committed, versioned OpenHDO
   contracts. The current checked-in public contract is link.register; do not
   invent an RGB command name or payload.
5. Reconnect and shutdown behavior that preserves device identity and makes
   failures observable without claiming a successful write.

The driver belongs in the separate OpenHDO Linker process boundary, not in the
server. The Linker repository is currently a scaffold and has no native
Python entry point or vendor driver to run.

## 3. Pairing and discovery for the physical lamp

These are vendor-specific acceptance steps, not generic OpenHDO commands:

1. Put the lamp on the same LAN as the PC and record the vendor-required
   address or discovery scope.
2. Complete the vendor’s pairing flow and record the exact credential or
   session requirement. A reachable IP is not proof that authentication works.
3. Discover only the exact vendor/model/revision or require an explicit stable
   device identifier. Never select the first Wi-Fi device returned.
4. Read the lamp state through the documented local API before issuing writes.
5. Confirm that the API exposes the requested RGB and brightness semantics;
   preserve the vendor’s reported values and units.
6. Issue one documented power/color change, then read back the state. If the
   API has no readback, report that limitation instead of inferring success.
7. Verify the failure paths: invalid credentials, missing device, timeout,
   rate limit, malformed response, and reconnect after a short network loss.

The current OpenHDO repositories do not implement any of these steps. Do not
document a vendor endpoint, payload, port, pairing code, or discovery packet
until the exact lamp model is supplied and its official API has been checked.

## 4. Health checks

Use these checks in the following order:

| Check | What success means | Current availability |
| --- | --- | --- |
| Server --check | The OpenHDO binary and local configuration boundary start | Working |
| Python SDK unittest | The checked-in envelope/manifest code passes | Working |
| Vendor reachability | The exact lamp responds on its documented local API | Requires the driver |
| Vendor authentication | Credentials are accepted without logging secrets | Requires the driver |
| Discovery/pairing | The exact device identity is found and stable | Requires the driver |
| Read state | Power, RGB, and brightness are returned with known semantics | Requires the driver |
| Write/read-back | A documented change reaches the lamp and is observed | Requires the driver |
| Linker liveness/health | The native Python process reports actionable health | No current process |
| OpenHDO end-to-end | Linker traffic reaches a server transport and state is observable | Planned |

A green server --check must never be reported as a green lamp check. The
current server command is one-shot and transport-free.

## Troubleshooting

### The exact lamp model is unknown

Stop at this point. Obtain the full model/part number, hardware revision,
firmware, region, and official local API documentation. A vendor family name
is insufficient for a native driver.

### The vendor API works in the mobile app but not from the PC

Check whether the app uses a cloud-only service, a local API, or a
vendor-specific session. Confirm the PC is on the same LAN and that the local
API is officially supported. Do not reverse-engineer an undocumented protocol
as if it were a stable integration contract.

### Discovery finds no lamp

Verify the lamp is provisioned and powered, the PC is on the same network,
the vendor discovery method is enabled, and the firewall permits the
documented traffic. If the vendor requires a fixed address or explicit
pairing, configure that according to the vendor documentation.

### The API returns 401 or 403

Re-check credential type, account/region, scope, expiry, pairing status, and
clock requirements. Replace the credential without printing it. Do not put
the token in a commit or a health log.

### A write reports success but color does not change

Confirm that the model supports RGB, that the payload uses the vendor’s exact
range and field names, and that the API returns or permits a read-back check.
A successful HTTP response alone is not proof of physical state.

### The server check does not keep running

That is expected. openhdo-server --check validates the server foundation and
exits; it is not a Linker runtime or a lamp gateway.

### No native Python Linker command exists

That is the current repository status. The Python SDK can validate and
serialize link.register, but the Linker process, vendor driver, discovery,
pairing, and health command still need implementation.

## Working versus planned

| Area | Status |
| --- | --- |
| Server build, CTest, and one-shot readiness check | Working |
| Python v1 envelope and Linker manifest helper | Working |
| Native Python Linker process | Planned |
| Exact vendor/model Wi-Fi driver | Planned after model/API identification |
| Credentials, pairing, and discovery implementation | Planned |
| RGB command/state contract | Planned and must be versioned |
| Linker-to-server transport | Planned |
| Physical lamp end-to-end control | Not runnable from current repositories |

## Contributing

This get-started repository is main-only for this workflow. Do not create or
switch branches here.

1. Keep this guide tied to committed behavior. Do not document a vendor
   endpoint or RGB message without the exact model/API and a committed
   implementation or contract.
2. Keep native hardware access in the Linker process. Keep credentials out of
   source, docs, logs, and commits.
3. Add a versioned server-facing contract under server/contracts/v1/ with an
   example and compatibility test before documenting a new RGB message.
4. Run the relevant checks:

   ~~~bash
   cmake --preset ci
   cmake --build --preset ci
   ctest --preset ci
   cd python && python -m unittest discover -s tests -v
   ~~~

5. Review and push directly to main:

   ~~~bash
   git status --short --branch
   git diff --check
   git add README.md
   git commit -m "docs: document native Wi-Fi Linker path"
   git push origin main
   ~~~

The server’s detailed rules are in its
[CONTRIBUTING.md](https://github.com/OpenHDO/server/blob/master/CONTRIBUTING.md)
and [technical documentation](https://github.com/OpenHDO/server/blob/master/DOCS.md).

## Repository map

- [Server](https://github.com/OpenHDO/server)
- [Server v1 contracts](https://github.com/OpenHDO/server/tree/master/contracts/v1)
- [Python SDK in the server repository](https://github.com/OpenHDO/server/tree/master/python)
- [Linker scaffold](https://github.com/OpenHDO/linker)
- [OpenHDO architecture](https://github.com/OpenHDO/about)
