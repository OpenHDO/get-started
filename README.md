# Get started: one-PC Wi-Fi path for a Sirius LED Smart C37

This guide describes the real-device path for one Sirius LED Smart C37 lamp
and one PC on the same Wi-Fi/LAN. The public OpenHDO model is an ordinary
abstract `light`: power, brightness, and RGB state/commands. Vendor names,
Tuya details, device IDs, local keys, IP addresses, protocol versions, and DP
indexes stay inside Linker and never cross the server contract boundary.

Tuya/Smart Life compatibility is plausible evidence for this product, not
confirmation of the local protocol for the exact lamp. Confirm the actual
model, revision, firmware, region, and local-device evidence before enabling
hardware access.

## Published components

- [`server` `master`](https://github.com/OpenHDO/server/tree/master) is the
  Python FastAPI/Starlette + uvicorn control-plane runtime. It provides HTTP
  health/light APIs, Linker WebSockets for registration/state/results, and an
  optional built-in React `/admin` panel.
- [`server/contracts/v1`](https://github.com/OpenHDO/server/tree/master/contracts/v1)
  is the source of truth for server-facing envelopes, light commands, and
  reported/changed state. Do not duplicate or redefine those payloads here.
- [`linker` `main`](https://github.com/OpenHDO/linker/tree/main) contains the
  runnable `openhdo-linker` process and a native Python Tuya-compatible local
  driver. Linker owns discovery/pairing, protocol and DP mapping, credentials,
  device state, and read-after-write confirmation.
- [`app`](https://github.com/OpenHDO/app) and
  [`server-dashboard`](https://github.com/OpenHDO/server-dashboard) are React
  clients. They are not hardware runtimes and do not replace Server or Linker.

## What is still blocked for the real Sirius lamp

Before a real run, the exact unit must provide verified values for:

- current LAN IP address;
- device ID;
- 16-byte `local_key`;
- local protocol (`3.1`, `3.2`, `3.3`, or `3.4`);
- power, brightness, color, and—if used—white DP mapping;
- brightness/color/white device ranges and color encoding;
- physical read-back proving that a command changed the lamp.

An app label or a community guess is not enough. Tuya/Smart Life remains
unconfirmed until the exact packaging, firmware, region, and approved device
inspection/evidence support it. A successful TCP response is not physical
confirmation.

Never send a `local_key` to chat. Keep it in a local environment managed by
the operator or a permissions-restricted protected file/secret store. Keep
the local config and secret files gitignored; never put credentials in this
README, source, logs, envelopes, or commits.

## One-PC startup sequence

The examples use PowerShell. Replace every angle-bracket value with a real,
verified value. Do not paste secrets into shell history.

### 1. Start Server

```powershell
git clone --branch master https://github.com/OpenHDO/server.git
Set-Location server\python
py -3 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -e ".[dev]"
openhdo-server --check
openhdo-server
```

The default server bind is `127.0.0.1:8000`. For this one-PC path, the
Linker connects to:

```text
ws://127.0.0.1:8000/api/v1/linkers/<linker_id>
```

Check readiness with `GET /api/v1/health`. The public HTTP light endpoints
are under `/api/v1/lights`; event subscribers use
`WS /api/v1/events`. A built panel is optional:

```powershell
Set-Location ..\web
npm ci
npm run build
```

When `web/dist/index.html` exists, Server serves it at `/admin`. `app` and
`server-dashboard` remain separate React clients.

### 2. Install Linker

In a second PowerShell window:

```powershell
git clone --branch main https://github.com/OpenHDO/linker.git
Set-Location linker
py -3 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -e .
```

### 3. Inspect, if the checked-out Linker publishes such a command

Do not invent an `inspect` command. Check the current checkout's documented
CLI help first:

```powershell
openhdo-linker --help
```

The published `linker/main` currently documents the read-only real-device
smoke command rather than a separate `inspect` command. Once the verified
values are available, it performs a real TCP connect, encrypted local poll,
response validation, and health reporting:

```powershell
python -m openhdo_linker.smoke `
  --ip '<actual LAN IP>' `
  --device-id '<actual device ID>' `
  --protocol-version '<3.1|3.2|3.3|3.4>' `
  --dp-power <actual power DP> `
  --dp-brightness <actual brightness DP> `
  --dp-color <actual color DP> `
  --color-format '<rgb_hex|hsv_hex>' `
  --brightness-min <actual device minimum> `
  --brightness-max <actual device maximum>
```

Store the key in a protected local file such as
`.secrets\tuya-local-key.txt` (that path must be in `.gitignore`), then load
it into the process environment without printing it:

```powershell
$env:OPENHDO_TUYA_LOCAL_KEY = (Get-Content .secrets\tuya-local-key.txt -Raw).Trim()
```

For RGBW, also provide the verified white DP and range to the smoke command.
Discovery, where explicitly enabled by the driver, can learn address/ID
metadata but cannot recover the local key or DP mapping.

### 4. Configure the verified real mapping and validate

Use local environment variables or the Linker's protected, gitignored JSON
configuration with environment overrides. The following values are mandatory
for the real native driver:

```powershell
$env:OPENHDO_SERVER = 'ws://127.0.0.1:8000'
$env:OPENHDO_LINKER_ID = '<linker_id>'
$env:OPENHDO_LIGHT_ID = '<lowercase_light_id>'
$env:OPENHDO_TUYA_IP = '<actual LAN IP>'
$env:OPENHDO_TUYA_DEVICE_ID = '<actual device ID>'
$env:OPENHDO_TUYA_PROTOCOL = '<3.1|3.2|3.3|3.4>'
$env:OPENHDO_TUYA_DP_POWER = '<actual power DP>'
$env:OPENHDO_TUYA_DP_BRIGHTNESS = '<actual brightness DP>'
$env:OPENHDO_TUYA_DP_COLOR = '<actual color DP>'
$env:OPENHDO_TUYA_COLOR_FORMAT = '<rgb_hex|hsv_hex>'
$env:OPENHDO_TUYA_BRIGHTNESS_MIN = '<actual brightness minimum>'
$env:OPENHDO_TUYA_BRIGHTNESS_MAX = '<actual brightness maximum>'
openhdo-linker --validate
```

For RGBW, additionally set the verified `OPENHDO_TUYA_DP_WHITE`,
`OPENHDO_TUYA_WHITE_MIN`, and `OPENHDO_TUYA_WHITE_MAX`. Validation must fail
closed when a mapping is missing or unconfirmed.

### 5. Run Linker and verify read-back

```powershell
openhdo-linker
```

Linker registers one abstract LED `light` with Server, reports observed
state, receives the versioned `light.command.*` messages, performs the real
device action, polls again, and sends `command.result` only after read-after-
write confirmation. Verify the physical lamp and the reported state. Stop if
the state or color semantics do not match; do not widen the public contract
with vendor fields.

## Contract and local checks

The normative server-facing messages are under `server/contracts/v1/`:

- `light.command.power`;
- `light.command.brightness` (`0..255`);
- `light.command.rgb_color` (`0..255` channels);
- `light.state.reported` and `light.state.changed`.

Run the published checks when needed:

```powershell
# server/python
python -m unittest discover -s tests -v

# linker
cmake -S . -B build
ctest --test-dir build -C Debug --output-on-failure
```

The end-to-end path is not complete until all blockers above—including
physical read-back—are satisfied.

## Contributing here

This repository is updated directly on `main`; do not create or switch
branches. Keep this guide tied to published behavior, keep hardware access
and secrets in Linker, and edit only this repository for this task.

```powershell
git diff --check
git add README.md
git commit -m "docs: align native server and linker onboarding"
git push origin main
```
