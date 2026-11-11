# Bot Defense CORS lab

Reproduces the F5 Bot Defense CORS preflight problem: the injected JS adds
`xa4vrhyp3q-*` headers to cross-origin API calls, and an API whose
`Access-Control-Allow-Headers` doesn't list them gets blocked by the browser.

## Topology

One nginx container, two server blocks, published as **two hostnames** so the
browser sees two origins (CORS only applies cross-origin):

```
                    Browser (test page + fake BD JS)
                    │ 1. load page          │ 2. API call (cross-origin → preflight)
                    ▼                       ▼
     https://yohr-web.f5xc.cloud     https://yohr.f5xc.cloud
        (XC LB, frontend)               (XC LB, API)
                    │                       │
     ┌──────────────┼───────────────────────┼──────────────┐
     │  nginx :8080 test page        nginx :8081 API       │
     │                               strict CORS           │
     └───────────── one container: bd-cors-lab ────────────┘
```

| Lab | Simulates |
|---|---|
| `yohr-web.f5xc.cloud` → nginx :8080 | yohr(frontend) |
| `yohr.f5xc.cloud` → nginx :8081 | yohr-api |

The API keeps the **original** CORS policy
(`Allow-Headers: Content-Type, Authorization`). The fix is applied on BIG-IP with
the iRule in `irule/`, not in nginx.

## Files

```
docker-compose.yml               # single nginx:alpine container
nginx/default.conf               # :8080 frontend, :8081 API, origin allow-list
html/index.html                  # test page + fetch/XHR hook adding xa4vrhyp3q-* headers
irule/bd_cors_allow_headers.tcl  # BIG-IP iRule: appends requested xa4vrhyp3q-* to Allow-Headers
```

## 1. Start nginx

```bash
docker compose up -d
docker exec bd-cors-lab nginx -t
curl -s http://localhost:8081/health      # {"status":"up"}
```

After editing `nginx/default.conf`:

```bash
docker exec bd-cors-lab nginx -t && docker exec bd-cors-lab nginx -s reload
```

## 2. Publish on F5 XC

Two HTTP load balancers (or one LB with two domains routed to different origin pools):

| Domain | Origin pool |
|---|---|
| `yohr-web.f5xc.cloud` | `<docker-host>:8080` |
| `yohr.f5xc.cloud` | `<docker-host>:8081` |

Keep any CORS policy on the XC LB **disabled**, so nginx's headers are what you test.

The allowed frontend origin is set in the `map` in `nginx/default.conf`
(`https://yohr-web.f5xc.cloud`). Allow the **frontend** origin there, not the API hostname.

## 3. Test – nginx direct (no iRule)

```bash
API=https://yohr.f5xc.cloud
FE=https://yohr-web.f5xc.cloud

# a) allowed origin, no BD headers → Allow-Origin present, passes
curl -si -X OPTIONS $API/api/login -H "Origin: $FE" \
  -H "Access-Control-Request-Method: POST" \
  -H "Access-Control-Request-Headers: content-type" | grep -iE "^HTTP|access-control"

# b) allowed origin + BD headers → Allow-Headers lacks xa4vrhyp3q-* → browser blocks
curl -si -X OPTIONS $API/api/login -H "Origin: $FE" \
  -H "Access-Control-Request-Method: POST" \
  -H "Access-Control-Request-Headers: content-type,xa4vrhyp3q-a" | grep -iE "^HTTP|access-control"

# c) other origin → no Access-Control-Allow-Origin → browser blocks
curl -si -X OPTIONS $API/api/login -H "Origin: https://other.example" \
  -H "Access-Control-Request-Method: POST" \
  -H "Access-Control-Request-Headers: content-type" | grep -iE "^HTTP|access-control"
```

curl always gets `204`; the browser makes the block decision from the headers.
A request without an `Origin` header (e.g. plain `curl -I`) gets no
`Access-Control-Allow-Origin` – that's expected.

Browser: open `https://yohr-web.f5xc.cloud` (API defaults to `https://yohr.f5xc.cloud`).

| BD JS | Expected |
|---|---|
| on  | **CORS** error: `xa4vrhyp3q-a is not allowed by Access-Control-Allow-Headers` |
| off | 200, `bd_header_seen` empty |

Shortcut without the frontend LB: open `http://<docker-host>:8080` – it's already a
different origin and matches the `:8080` entry in the map.

## 4. Test – through BIG-IP with the iRule (later)

Load `irule/bd_cors_allow_headers.tcl` and attach it to the API VS:

```bash
tmsh create ltm pool bd_cors_api_pool monitor http members add { <docker-host-ip>:8081 }
tmsh create ltm virtual vs_bd_cors_api destination <vs-ip>:80 ip-protocol tcp \
  profiles add { tcp http } pool bd_cors_api_pool \
  source-address-translation { type automap } rules { bd_cors_allow_headers }
tmsh save sys config
```

Point the test page's API field at the VS (or open `/?api=http://<vs-ip>`).
Expected: the preflight's `Allow-Headers` now also lists the requested
`xa4vrhyp3q-*` headers, and calls return 200 with `bd_header_seen` filled.

The iRule only acts when nginx already returned `Access-Control-Allow-Origin`, so
the origin allow-list still decides who's trusted. Keep `OPTIONS` out of the Bot
Defense protected endpoints.

## Notes

- `xa4vrhyp3q-` is the real Bot Defense header prefix; the suffixes the test page
  sends (`a, b, c, d, f, z`) are placeholders. The iRule matches on the prefix.
- CORS is enforced by browsers only. curl, Postman and bots are not blocked by it –
  that's Bot Defense's job.
