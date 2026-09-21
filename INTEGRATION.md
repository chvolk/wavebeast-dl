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
not exposed off-device; integration is via intents, not that port.

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
