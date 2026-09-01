# Get started: native Python Linker for Sirius LED Smart C37

This guide is for one physical Sirius LED Smart C37 lamp and one PC running
the local OpenHDO processes. The target is a Wi-Fi RGBW C37/E14 lamp. Wi-Fi is
only the network transport; OpenHDO does not define one universal Wi-Fi lamp
protocol.

The exact vendor and model are a hard prerequisite. Do not start with a
product family, a guessed endpoint, or a generic Wi-Fi implementation. Record
the full model, hardware revision, firmware, region, and the vendor's local
API documentation before choosing a driver path.

Public product evidence describes this SIRIUS Light C37/E14 lamp as RGBW with
Wi-Fi/BLE connectivity and Tuya/Smart Life compatibility
([public listing](https://m.integration.vs.market.yandex.net/card/umnaya-lampa-e14-rgbw-5w-wi-fi-yandeks-alisa-marusya-tuya-smart-life-sirius/102931012657)).
This is not authoritative local-API documentation. Treat Tuya/Smart Life as
unconfirmed until the exact lamp packaging, manual, firmware, region, and
vendor API confirm it.

The public OpenHDO object is an ordinary `light`. Public state and commands
must use vendor-neutral concepts such as light identity, power, brightness,
and RGB color. Vendor names, protocol fields, device keys, and network
addresses are adapter-internal and do not belong in the public light payload.

## Current stage

The target architecture is a Python server backend/runtime with HTTP,
WebSocket, and API boundaries, plus React web panels. The current published
repositories are not yet at that runtime stage:

- `server/contracts/v1/` contains the committed, versioned, vendor-neutral
  Light v1 schemas and examples;
- `server/python/` contains a reference SDK for validated envelopes and
  `link.register`, not the Python server runtime;
- `server/web/` contains the React panel shell and its build scripts, but it is
  not connected to a live API or lamp state;
- published `linker/main` contains the typed Python driver boundary,
  reconnect supervisor, JSON-line boundary, and contract tests, but no
  runnable Linker process or Sirius/Tuya driver.

The following are not currently runnable from the published repositories:

- a Python server process with HTTP or WebSocket endpoints;
- a native Python Linker process entry point;
- a server-to-Linker transport adapter;
- a Sirius-specific local Wi-Fi driver, discovery, or pairing implementation;
- physical lamp control or an end-to-end health result.

C++ code in the server repository is optional native/SDK foundation only. This
guide does not use a C++ server runtime as the onboarding path.

## Prerequisites

### Required lamp identity

Fill this out from the physical device and vendor documentation:

| Field | Required value |
| --- | --- |
| Vendor | Sirius / exact legal vendor from the lamp or manual |
| Model | Sirius LED Smart C37, plus the full model or part number |
| Hardware revision | Printed revision, if present |
| Firmware | Exact firmware version |
| Region | Region/account/API variant |
| Local API | Official local API or SDK documentation and version |
| Authentication | Token, API key, pairing code, or vendor-specific session |
| Discovery | Vendor-defined discovery method or a configured device address |
| RGB semantics | RGB range, brightness range, power behavior, and readback support |

The product shape above is a target identification, not proof that every
Sirius C37 unit shares one API. The exact unit still controls the driver
decision.

If the vendor exposes only a cloud API or only an undocumented mobile-app
protocol, this is not a verified native local Wi-Fi path. Do not substitute a
guessed protocol.

### PC, network, and software

- Python 3.11 or newer;
- Git and CMake 3.20 or newer for the published Linker contract check;
- Node.js and npm for the React panel shell check;
- a local network shared by the PC and lamp;
- the lamp provisioned onto that network according to the vendor procedure;
- firewall rules that allow the vendor's documented local traffic;
- vendor documentation for timeouts, rate limits, reconnects, and errors.

The reference SDK is stdlib-only. A real driver may need a vendor-specific
dependency, but do not install or select one until the exact model and API are
confirmed. The published Linker repository has no installable driver package
or universal Wi-Fi dependency.

### Credentials

Use the credential type and scope required by the confirmed vendor API. Keep
secrets outside Git, README files, shell history, source code, and structured
logs. Prefer a permissions-restricted local secret store or file, and document
rotation and expiry behavior.

The published OpenHDO SDK and Linker boundary do not provide a credential
store or a universal configuration-file schema. Do not invent environment
variables or config keys and assume the runtime will read them; the concrete
native driver must define and validate its own configuration.

## 1. Verify the Python server contract and panel shell

There is no server start command to document yet. The current server
repository does not publish a Python backend entry point, HTTP health route,
WebSocket endpoint, or API listener. Do not treat a panel screen as proof of
server or lamp health.

Run the committed Python reference SDK checks from a fresh server checkout.
These validate the versioned message helpers only:

### Linux or macOS

~~~bash
git clone https://github.com/OpenHDO/server.git server
cd server/python
python3 -m unittest discover -s tests -v
~~~

### Windows PowerShell

~~~powershell
git clone https://github.com/OpenHDO/server.git server
Set-Location server\python
py -3 -m unittest discover -s tests -v
~~~

The command should finish with `OK`. `LinkerManifest.registration()` can
serialize the current v1 `link.register` message, but no transport sends it to
the server and it does not control a lamp.

The React panel shell can be checked separately:

~~~powershell
Set-Location server\web
npm ci
npm run build
npm run dev
~~~

`npm run build` is the available panel check. `npm run dev` serves the React
shell for local inspection; the current shell has no live API connection and
must not be presented as a working device dashboard.

## 2. Verify the native Python Linker boundary

The published Linker repository is a library boundary, not an executable.
Its `VendorRgbDriver` interface owns real discovery, credentials, connection,
state polling/subscription, power/RGB operations, and health. Its
`LinkerBoundary` owns envelope validation, correlation, and duplicate-safe
command handling. No physical I/O occurs in this contract check.

### Linux or macOS

~~~bash
git clone https://github.com/OpenHDO/linker.git linker
cd linker
cmake -S . -B build
ctest --test-dir build -C Debug --output-on-failure
~~~

### Windows PowerShell

~~~powershell
git clone https://github.com/OpenHDO/linker.git linker
Set-Location linker
cmake -S . -B build
ctest --test-dir build -C Debug --output-on-failure
~~~

This check should finish with `OK`. The current published Linker source has
no process entry point, no vendor driver module for this lamp, and no command
that can be used to start a hardware session. Do not invent a `python -m`
startup command until that entry point is committed.

## 3. Public light contract

The public object is an ordinary `light`, never a vendor device type. The
committed server Light v1 contract defines these versioned message types:

- `light.command.power` — `light_id`, `command_id`, `idempotency_key`, and
  boolean `power`;
- `light.command.brightness` — the same command identity fields and integer
  `brightness` from 0 to 255;
- `light.command.rgb_color` — the same command identity fields and `r`, `g`,
  and `b` channels from 0 to 255;
- `light.state.reported` — the latest observed `power`, `brightness`, RGB
  color, and `state_revision`;
- `light.state.changed` — an observed state change correlated to the command
  and repeating its command identity metadata.

Every message is a v1 envelope. Commands have a correlation identifier;
retries reuse the logical command and idempotency key. The schemas are logical
contracts, not an HTTP or WebSocket transport.

The lamp is marketed as RGBW, but the current public contract exposes RGB
only. Do not add a white-channel field to public messages from vendor data.
Keep white-channel handling internal until a separately versioned public
contract exists.

The Linker boundary currently uses its own internal `link.register`,
`link.state`, `command`, and `command.result` messages. No committed adapter
connects those messages to the server's `light.command.*` and
`light.state.*` schemas. Therefore the two contract checks above are not an
end-to-end control path.

## 4. Native driver path for the physical lamp

This is the only hardware path covered by this guide:

1. Confirm the exact Sirius model, revision, firmware, region, and documented
   local API.
2. Confirm that the API is a supported local Wi-Fi protocol and that it
   exposes authentication, discovery or explicit addressing, state readback,
   power, RGB, and brightness operations for this exact unit.
3. Implement or select a native Python driver that satisfies the published
   Linker driver boundary. It must own protocol encoding/decoding, credentials,
   discovery, pairing, acknowledgements, bounded retries, reconnects, and
   vendor-error normalization.
4. Validate configuration before opening a connection. Require the exact
   model identity, device identity, vendor address/discovery settings,
   credential reference, protocol version, value ranges, timeout, retry, and
   Linker identity. Never log credentials.
5. Discover or address the device using the vendor procedure, then reject an
   ambiguous result. A reachable device is not proof of authentication or
   model compatibility.
6. Read state before the first write. Verify power, RGB, brightness, units,
   and readback semantics.
7. Issue one documented power or RGB change, read the state back, and record
   whether the physical result matches. A successful network response without
   readback is not proof of a lamp change.
8. Publish only the ordinary, versioned OpenHDO light state and commands at
   the boundary. Keep vendor fields and credentials inside the adapter.

The current published repositories stop before steps 3 through 8 are
complete for Sirius. The correct onboarding result today is a documented
blocker, not a claimed successful lamp control.

## Advanced adapter setup: possible Tuya-compatible details

Keep this section out of the public light model. Public evidence suggests
Tuya/Smart Life, but does not confirm the local protocol for this exact lamp.
Only after the packaging, firmware, region, and authoritative vendor/API
evidence confirm it may a native adapter investigate details such as:

- `tuya_local`: an internal adapter/protocol label, not a public capability or
  an OpenHDO runtime;
- DP mapping: the exact device-property mapping for power, mode, brightness,
  RGBW, and any color-temperature fields;
- `local_key`: a device credential kept in a protected secret store and never
  placed in source, envelopes, or logs;
- `IP`: the discovered or configured local address, which may change and must
  not become the public device identity;
- protocol version, port, timeout, retry, acknowledgement, and readback
  behavior: values taken from the exact device/API evidence, never copied from
  another compatible product.

Do not fill in DP IDs, credential-acquisition steps, ports, endpoints, or
payloads from another Tuya-compatible product. An app label or community
guess is not enough to write or enable this driver. The current published
Linker main does not ship a Sirius-specific Tuya adapter or a runnable process.

## 5. Pairing, discovery, and health checks

These are vendor-specific acceptance checks, not generic OpenHDO commands:

1. Provision the lamp and PC on the same LAN according to the vendor
   procedure.
2. Complete the vendor pairing flow and store the required credential through
   the driver's protected configuration path.
3. Discover only the exact Sirius model/revision or require an explicit stable
   device identifier. Never select the first device returned.
4. Check driver health and connection state before sending a command.
5. Read the lamp state and validate known power, RGB, and brightness semantics.
6. Send one command with a stable logical command identity, then verify state
   readback and the correlated state event.
7. Exercise invalid credentials, missing device, timeout, malformed response,
   rate limit, and reconnect after a short network loss.

Current availability:

| Check | What success means | Current availability |
| --- | --- | --- |
| Python SDK unittest | Versioned envelope/manifest helpers pass | Working |
| React panel build | The panel shell type-checks and bundles | Working; shell only |
| Linker contract CTest | Typed boundary, correlation, and reconnect checks pass | Working |
| Python backend health | A live API reports server readiness | Not published |
| Linker process health | A native process reports driver health | No process |
| Vendor reachability/authentication | The exact lamp accepts local connection and credentials | Requires exact driver |
| Discovery/pairing | The exact device identity is found and stable | Requires exact driver |
| Read state | Power, RGB, and brightness use confirmed semantics | Requires exact driver |
| Write/read-back | A physical change is observed and correlated | Requires exact driver |
| OpenHDO end-to-end | Python API, Linker, and lamp exchange real messages | Not published |

## Troubleshooting

### The exact lamp model is unknown

Stop. Obtain the full model/part number, hardware revision, firmware, region,
and official local API documentation. A vendor family name is insufficient.

### The vendor API works in the mobile app but not from the PC

Check whether the app uses a cloud-only service, a local API, or a
vendor-specific session. Confirm the PC is on the same LAN and that local use
is officially supported. Do not treat an undocumented app protocol as a
stable driver contract.

### Discovery finds no lamp

Verify the lamp is provisioned and powered, the PC is on the same network,
the vendor discovery method is enabled, and the firewall permits the
documented traffic. If explicit addressing or pairing is required, follow the
vendor documentation and confirm the returned identity.

### Authentication fails

Re-check credential type, account/region, scope, expiry, pairing status, and
clock requirements. Replace the credential without printing it. Never put it
in a commit or health log.

### A write reports success but color does not change

Confirm that this exact model supports RGB, that the value mapping and range
come from its documentation, and that the API returns or permits readback. A
successful network response alone is not proof of physical state.

### The panel shows healthy-looking data

The current React panel is a static shell. It is useful for checking the
frontend build and layout, but it has no published Python API connection and
does not prove server, Linker, or lamp health.

### No native Python Linker command exists

That is the current repository status. The Linker contract can be tested with
CTest, but the native process, server transport, vendor driver, discovery,
pairing, and health command still need committed implementation.

## Working versus planned

| Area | Status |
| --- | --- |
| Versioned server Light v1 schemas and examples | Working / committed |
| Python reference SDK for v1 envelopes and registration | Working / committed |
| React panel shell build | Working / not API-connected |
| Typed native Python Linker boundary and contract tests | Working / committed |
| Python server HTTP/WebSocket/API runtime | Planned in server repository |
| Native Python Linker process | Planned |
| Sirius-specific local Wi-Fi driver | Planned after exact model/API confirmation |
| Vendor credentials, pairing, and discovery implementation | Planned in the driver |
| Server-to-Linker transport adapter | Planned |
| Physical lamp end-to-end control | Not runnable from current published repositories |

## Contributing

This get-started repository is main-only for this workflow. Do not create or
switch branches here.

1. Keep this guide tied to committed behavior. Do not document a vendor
   endpoint, message, startup command, or health route without the exact
   model/API and a committed implementation or contract.
2. Keep native hardware access in the Linker process. Keep credentials out of
   source, docs, logs, and commits.
3. Add or update a versioned server-facing contract under
   `server/contracts/v1/` with an example and compatibility test before
   documenting a new public light message.
4. Run the relevant checks from the corresponding repository:

   ~~~bash
   # server/python
   python -m unittest discover -s tests -v

   # server/web
   npm ci
   npm run build

   # linker
   cmake -S . -B build
   ctest --test-dir build -C Debug --output-on-failure
   ~~~

5. For this repository, review and push directly to `main`:

   ~~~powershell
   git status --short --branch
   git diff --check
   git add README.md
   git commit -m "docs: clarify native Sirius Linker onboarding"
   git push origin main
   ~~~

The server's detailed rules are in its
[CONTRIBUTING.md](https://github.com/OpenHDO/server/blob/master/CONTRIBUTING.md)
and [technical documentation](https://github.com/OpenHDO/server/blob/master/DOCS.md).

## Repository map

- [Server](https://github.com/OpenHDO/server)
- [Server v1 contracts](https://github.com/OpenHDO/server/tree/master/contracts/v1)
- [Python reference SDK in the server repository](https://github.com/OpenHDO/server/tree/master/python)
- [React panel shell in the server repository](https://github.com/OpenHDO/server/tree/master/web)
- [Linker boundary](https://github.com/OpenHDO/linker)
- [OpenHDO architecture](https://github.com/OpenHDO/about)
