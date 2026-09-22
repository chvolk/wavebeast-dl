# Integrating with WaveBeast

WaveBeast is a scan-to-creature game with **zero client coupling**: the engine only ever sees a
`ScanBundle` — a JSON set of real-world signals captured together — and
turns it deterministically into a creature. Anything that can produce signals (a phone, a Pi sensor rig,
a barcode scanner, another program) can feed WaveBeast. This doc is the contract for doing that on
**desktop / Linux** and on **mobile (Android)**.

The same barcode/QR/NFC value always resolves to the same beast on any device; a pure sensor sweep
derives a place-stable identity from ambient signals (WiFi BSSIDs, or a bucketed composite). Read the [public wiki](https://wavebeasts.com/docs/) for gameplay and the engine’s
`GET /capabilities` endpoint for live conventions.

---

## Desktop / Linux

The engine is a single static binary (no cgo). Run it, then integrate over HTTP or the CLI.

```sh
wavebeast                     # serve the HTTP/JSON API + web GUI on :8777 (WAVEBEAST_ADDR to change)
# env: WAVEBEAST_DB=<path>  WAVEBEAST_TOKEN=<tok>  (token gates mutating endpoints via X-WB-Token)
```

### CLI (headless — a Pi, a cron job)

No browser needed. Point it at a file holding a full `ScanBundle` **or** a bare JSON array of signals
(the array is wrapped for you):

```sh
wavebeast scan   -file scan.json   [-addr http://host:8777] [-token TOK]   # local scan → beast/resources
wavebeast submit -file scan.json   [-addr http://host:8777] [-token TOK]   # relay a snapshot to your linked account
```

A minimal bare-signals file (`scan.json`):

```json
[
  { "kind": "code", "strength": 1.0, "value": { "symbology": "EAN13", "data": "0012345678905" } },
  { "kind": "scalar", "strength": 0.6, "value": { "metric": "temp_c", "n": 21.5 } }
]
```

### HTTP API

All bodies are JSON. Mutating routes honor `X-WB-Token` when the engine was started with a token.

| Method / path | Purpose |
|---|---|
| `GET  /capabilities` | Signal kinds, strength-normalization conventions, feature flags. **Read this first.** |
| `POST /scan` | Submit a `ScanBundle` → a rolled beast (or resources). Rate-limited locally to one yield per 300 seconds. |
| `POST /generate` | **Stateless** resolver: bundle → beast identity + stats, no persistence (what the site uses). |
| `POST /render` | **Stateless** sprite PNG for a given identity. |
| `POST /battle/auto` | **Stateless** 3v3 auto-battle resolver. |
| `GET  /collection`, `GET /beast/{id}`, `GET /species/{sid}` | Read the local collection. |
| `GET  /sprite/{id}`, `GET /sprite/species/{sid}` | Sprite PNGs. |
| `POST /node/config`, `POST /node/submit` | Link a background node to an account and relay snapshots at the server’s cadence (normally 300 seconds). |

Example `ScanBundle` POST:

```sh
curl -s localhost:8777/scan -H 'content-type: application/json' -d '{
  "schema":"wavebeast.scanbundle","v":1,
  "signals":[{"kind":"code","strength":1.0,"value":{"data":"WB:GARDEN-7"}}]
}'
```

### Open a scan in the web GUI

The GUI reads a `#code=<url-encoded>` fragment on load, prefills the code field, and sweeps. Any tool
can hand a value to a running local GUI:

```
http://localhost:8777/#code=0012345678905
```

---

## Mobile (Android)

The standalone APK bundles the engine and a `WBNative` bridge that folds the phone's real sensors
(magnetometer, light, motion, WiFi, cellular gen/bars) into every scan. Other apps integrate two ways.

### Share text into WaveBeast (`ACTION_SEND`, `text/plain`)

Any app's **Share** sheet can send a barcode/QR value or a URL to WaveBeast. It lands in the code
field and auto-sweeps. From another Android app:

```java
Intent i = new Intent(Intent.ACTION_SEND);
i.setType("text/plain");
i.setPackage("net.wavebeasts.app");
i.putExtra(Intent.EXTRA_TEXT, "0012345678905");
startActivity(i);
```

### Deep link (`wavebeast://`)

Open WaveBeast on a specific scan from a link, an NFC tag, a QR code, or another app:

```
wavebeast://scan?code=0012345678905     # or the shorthand  wavebeast://0012345678905
```

```java
startActivity(new Intent(Intent.ACTION_VIEW, Uri.parse("wavebeast://scan?code=" + Uri.encode(code))));
```

Both paths resolve to the web GUI's `#code=` handler, so a deep-linked value scans identically to one
typed in or captured by the camera. The engine listens only on `127.0.0.1:8777` inside the app — it is
not exposed off-device. Other Android apps on the same phone can use that loopback API
while the standalone app is running. Read `/host` to display its selected authority; use
`X-WB-Token` if an engine token is configured. Intents/deep links also work without an HTTP client.

### Linked account identity (0.12.1)

`GET https://wavebeasts.com/api/account` with `X-WB-Node-Token` returns the linked
account's name, primary email (or null when unavailable), plan/subscription status,
shard/core balances, owned beast count, node count/limit, and current node name/kind/cadence.
This endpoint is available on Free and Premium and returns `Cache-Control: private, no-store`.
Treat the code and returned account information as private.

The local engine exposes the same response at `GET /node/account`, using the saved World
link and the engine's normal local authentication. `POST /node/config` validates a nonempty
link against the account endpoint before saving it; a failed relink preserves the working
configuration. An empty token unlinks. Redirects are not followed with account credentials.

The standalone app and local GUI display this identity in World. A successful camera code
scan automatically stages one camera input, shows “Code grabbed” with a shortened preview,
and stops the camera. The complete code is retained. Typed text has its own single input;
new camera inputs replace the camera slot, and other manual inputs replace their matching
kind (or scalar metric). Slot metadata is never sent in scan payloads.

The paged public manual is at https://wavebeasts.com/docs/ . Agent setup instructions for
both the thin listener and full local engine are at https://wavebeasts.com/docs/ai-setup/ .
Third-party API clients are independent applications, not part of the WaveBeasts suite.

### Wild sighting deadlines (0.12.2)

`GET /api/beasts` wild rows include `expires_in` (seconds), `expires_at` (ISO timestamp),
and `expiry_duration` (original lifetime in seconds). The linked engine exposes the same
private response at `GET /node/beasts`. Countdown bars use the remaining time without
extending the deadline on rerender. Expired sightings cannot consume a catch drive.

The iOS app is available as `wavebeast-ios-unsigned.ipa` in the public release. This is a
personal-signing/sideloading artifact, not a signed App Store or TestFlight distribution.

### Discovery queue controls (0.12.3)

`GET /api/beasts` includes `catch_drives`, containing only owned capture drives with positive stock
(`item_id`, `name`, `qty`). `POST /api/catch` accepts `{beast_id, drive}` and validates the selected
capture drive against current inventory. `POST /api/sightings/dismiss` accepts `{beast_id}` and
removes only an authenticated user's wild sighting; it never releases an owned beast or spends inventory.
The local engine relays these at `/node/catch` and `/node/sightings/dismiss`, using the saved account
link. Catch and dismiss controls live in the standalone Beasts tab and refresh stock after every attempt.

### Selected host and shared state (0.12.4)

A linked engine now uses the account as the authority for **all** gameplay, not just World.
`GET /host` returns `{ok, api_version: 2, host: {kind, url}}`; show that identity in every
custom client. Kinds are `account`, `engine` (a self-hosted target), or `local` (this engine's
own database). Credentials are never returned. `GET /state` supplies `beasts`, `currency`,
and `items` from the selected authority in one response.

The existing `/scan`, `/catch`, `/collection`, `/inventory`, `/economy/shop`, `/economy/buy`,
`/beast/{id}`, `/beast/release`, `/train`, and sprite routes follow that authority. Linked
scans create server-rolled account beasts; they never also credit the local save. Background
scans follow the same host. Authentication, stock, cooldown, payment and network errors are
returned without falling back to a different database. UI clients should refresh after every
mutation and on resume; the shared GUI also refreshes visible state every 15 seconds.

Link accounts with `/node/config` as before. Select an engine with POST `/host/config`:
`{"kind":"engine","url":"http://192.168.1.100:8777","token":"optional X-WB-Token"}`.
Select this engine's retained local save with `{"kind":"local"}`. Host replacement validates
reachability first. A remote engine must use its own database; relay chains are rejected.
Keep the previous local save for explicit import; do not sum balances or overwrite one host
with another. Node/account credentials and the optional local engine token are distinct.

Account clients using the standalone app's API can call the allowlisted `/account/api/*`
relay for account identity, Buddy care, shop, nickname/heal and sightings. It uses the selected
account's saved node credential. Pure self-hosted engines do not offer the paid account layer.
Omnitool is an independent example: it defaults to wavebeasts.com and explicitly offers a
self-hosted URL or the standalone app's selected host; it no longer silently prefers another
reachable engine.


### Legacy Buddy clients (0.12.5)

The selected-host relay preserves `GET /buddy` → `{individual_id, beast}` and
`POST /buddy` with `{individual_id: "account-beast-id"}`. An explicit empty string unslots;
a missing/invalid field is rejected. Rich account Buddy vitals/care remain available through
`/account/api/buddy` and `/account/api/buddy/care`.
