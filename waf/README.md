# AlgoCrack WAF V1

NGINX and ModSecurity v3 sit in front of the API Gateway. The Gateway stays the application gateway. The WAF is infrastructure.

## Architecture

```text
Client
  -> ingress or the published Compose port
      -> /       frontend, unchanged
      -> /api/*  waf (nginx + ModSecurity + OWASP CRS)
                   -> api-gateway (internal only)
                       -> microservices
```

Docker Compose publishes the WAF on host port 9090, which is the port clients already use. The `api-gateway` service has no host port.

Kubernetes and Helm send Ingress path `/api` to Service `waf:8080`. Service `api-gateway` stays ClusterIP. There is no second Ingress path to the Gateway.

In-cluster frontend server-side calls use `http://waf:8080` so they take the same hop. Prometheus still scrapes Gateway actuator metrics on the internal Service.

## Engine mode

Default is `SecRuleEngine DetectionOnly` (`MODSEC_RULE_ENGINE=DetectionOnly`). Matches are logged. Requests are not blocked by ModSecurity. NGINX rate limits and body-size limits still apply.

Blocking:

```bash
# Compose
MODSEC_RULE_ENGINE=On docker compose up -d waf

# Kubernetes
kubectl -n algocrack set env deploy/waf MODSEC_RULE_ENGINE=On

# Helm
helm upgrade algocrack ./helm/algocrack -f helm/my-values.local.yaml \
  --set waf.modsecRuleEngine=On
```

`Off` turns ModSecurity inspection off. Rate limits and the proxy remain. Use that for troubleshooting, then set the variable back. Do not add an Ingress path to `api-gateway`.

## Local startup

The WAF service definition is `docker/waf/compose.yaml`. The root Compose file includes it and does not publish the API Gateway.

```bash
docker compose up -d --build waf api-gateway
```

The WAF image is built from `docker/waf`. For a cluster, tag it the same way as the other images:

```bash
docker build -t hrishabhjoshi/waf:latest docker/waf
```

`scripts/dev-local.sh` starts the Java processes directly and does not pass through this container.

## CRS

Image: `owasp/modsecurity-crs:4.25.2-nginx-202610051210-lts` (ModSecurity v3, CRS 4.25.2).

| Setting | Value |
| --- | --- |
| Paranoia | 1 |
| Inbound anomaly threshold | 5 |
| Outbound anomaly threshold | 4 |
| Request body access | On |
| Request body limit | 1 MiB, reject when exceeded |
| Allowed methods | GET HEAD POST PUT PATCH DELETE OPTIONS |
| Audit log | RelevantOnly, JSON, stdout, parts AHKZ (no headers, request body, or response body) |

`crs-setup.conf` in this directory records those thresholds. The image entrypoint writes them into the CRS file at start.

Audit records omit request headers and request and response bodies, so submitted source, `Authorization`, and cookies are not written there. Rule messages can still quote the matched fragment that caused a block.

## Source-code exclusion

Submission, run, and custom-run DTOs send user source in JSON field `code`. ModSecurity exposes it as `ARGS:json.code`. Rule `10001001` drops that field from `attack-sqli` and `attack-xss` only, on:

- `POST /api/v1/submissions`
- `POST /api/v1/submissions/run`
- `POST /api/v1/submissions/custom`

Other fields, the query string, headers, and the path stay inspected. A query parameter named `code` is `ARGS:code` and is still inspected.

Playground create, update, and run send source in `sourceCode` (`ARGS:json.sourceCode`). The same SQLi/XSS removal applies on `POST /api/v1/playgrounds`, `POST /api/v1/playgrounds/run`, and `PUT /api/v1/playgrounds/{id}`. A query parameter named `sourceCode` stays inspected.

## Rate limits

Per client address, local/demo values, HTTP 429. The key is the TCP peer. Behind Kubernetes ingress, set `waf.trustedProxyCidrs` (Helm) or `SET_REAL_IP_FROM` (the base manifest) to the ingress controller source CIDR. The WAF then uses `X-Real-IP`, which ingress-nginx sets from the connection it accepted. `X-Forwarded-For` is not used, and a header from an untrusted peer does not change the key. Compose clients connect directly, so the default `127.0.0.1` trust list leaves their peer address in place.

| Route | Rate | Burst |
| --- | --- | --- |
| `/api/*` | 10 r/s | 20 |
| `/api/v1/auth/` | 2 r/s | 5 |
| `/api/v1/submissions` | 5 r/s | 10 |
| `/api/v1/submissions/{id}/complexity-analyses` | 1 r/s | 2 |

## Try a blocked request

With `MODSEC_RULE_ENGINE=On`:

```bash
curl -i "http://localhost:9090/api/v1/questions?id=1%27%20OR%20%271%27%3D%271"
curl -i "http://localhost:9090/api/v1/questions?q=%3Cscript%3Ealert(1)%3C/script%3E"
```

Expect HTTP 403. A normal `GET /api/v1/questions/1` should still reach the Gateway.

Rate limit:

```bash
for i in $(seq 1 15); do curl -s -o /dev/null -w "%{http_code}\n" http://localhost:9090/api/v1/auth/signin; done
```

Expect at least one `429`.

## Logs

Audit records for blocked requests go to container stdout (`MODSEC_AUDIT_LOG=/dev/stdout`). NGINX errors go to stderr.

Compose: Grafana Alloy already tails every container. In Loki: `{service="waf"}`.

Kubernetes: Alloy already tails pods in `algocrack` and labels `service` from the pod's `app` or component label. `{service="waf"}` or `kubectl -n algocrack logs deploy/waf`.

## Disable during troubleshooting

```bash
kubectl -n algocrack set env deploy/waf MODSEC_RULE_ENGINE=Off
# or
MODSEC_RULE_ENGINE=Off docker compose up -d waf
```

Traffic still enters through the WAF proxy. Set the variable back to `DetectionOnly` or `On` when finished.
