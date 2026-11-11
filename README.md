# Bot Defense CORS lab (nginx backend + BIG-IP iRule)

```
Browser (test page :8080, fake BD JS adds xa4vrhyp3q-* headers)
   │  cross-origin API calls
   ▼
BIG-IP VS (API)  ── iRule bd_cors_allow_headers ──►  nginx :8081 (yohr-api sim)
```

- **nginx :8081** – backend API with the ORIGINAL CORS policy
  (`Allow-Headers: Content-Type, Authorization`). Not changed for Bot Defense.
- **nginx :8080** – test page with a simulated Bot Defense script.
- **iRule** – appends the `xa4vrhyp3q-*` headers the browser requested to the
  preflight's `Access-Control-Allow-Headers`, only when the backend already
  returned `Access-Control-Allow-Origin`.

## 1. Start the backend

```bash
docker compose up -d
curl -s http://localhost:8081/health          # {"status":"up"}
docker compose logs -f                        # access log
```

## 2. BIG-IP config

Load the iRule: Local Traffic › iRules › Create, name `bd_cors_allow_headers`,
paste `irule/bd_cors_allow_headers.tcl`.

Then (replace IPs):

```bash
tmsh create ltm pool bd_cors_api_pool monitor http members add { <docker-host-ip>:8081 }
tmsh create ltm virtual vs_bd_cors_api destination <vs-ip>:80 ip-protocol tcp \
  profiles add { tcp http } pool bd_cors_api_pool \
  source-address-translation { type automap } rules { bd_cors_allow_headers }
tmsh save sys config
```

If the VS has the Bot Defense connector, keep `OPTIONS` out of the protected
endpoints – only the real `POST`/`GET` should be protected.

## 3. Test in the browser

Open `http://<docker-host>:8080`.

| API base | BD JS | Expected |
|---|---|---|
| nginx direct `http://<docker-host>:8081` | on  | CORS error (preflight lacks `xa4vrhyp3q-*`) |
| BIG-IP VS `http://<vs-ip>`               | on  | 200, `bd_header_seen` filled |
| either                                   | off | 200 |

You can also open `http://<docker-host>:8080/?api=http://<vs-ip>`.

## 4. Test with curl

```bash
# Direct to nginx – Allow-Headers has only Content-Type, Authorization
curl -si -X OPTIONS http://<docker-host>:8081/api/login \
  -H "Origin: http://<docker-host>:8080" \
  -H "Access-Control-Request-Method: POST" \
  -H "Access-Control-Request-Headers: content-type,xa4vrhyp3q-a,xa4vrhyp3q-b" | grep -i access-control

# Through the VS – Allow-Headers should also list xa4vrhyp3q-a, xa4vrhyp3q-b
curl -si -X OPTIONS http://<vs-ip>/api/login \
  -H "Origin: http://<docker-host>:8080" \
  -H "Access-Control-Request-Method: POST" \
  -H "Access-Control-Request-Headers: content-type,xa4vrhyp3q-a,xa4vrhyp3q-b" | grep -i access-control
```

## Notes

- The Origin must match the allow-list in `nginx/default.conf` (any host on
  :8080, or `https://yohr.yogyagroup.com`). A non-matching origin gets no
  `Allow-Origin`, and the iRule deliberately does nothing.
- If you serve the test page through a BIG-IP frontend VS, add that origin to
  the `map` in `nginx/default.conf`.
