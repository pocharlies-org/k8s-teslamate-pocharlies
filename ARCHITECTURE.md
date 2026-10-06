# ARCHITECTURE.md — k8s-teslamate-pocharlies

> TeslaMate + Mosquitto MQTT (telemetría Tesla) y el servicio `tesla-control-mcp`. BD: Postgres compartido. Escrito por `architect` (SC-1426).

## 1. Clientes y versiones

| cliente | repositorio / ruta | versión desplegada | cómo se despliega |
|---|---|---|---|
| TeslaMate | `k8s/manifest.yaml` | `teslamate/teslamate:3.0.0`; Grafana `teslamate/grafana:latest` (**sin pin**) | ArgoCD app `teslamate` |
| Mosquitto | `k8s/manifest.yaml` | `eclipse-mosquitto:2` | ídem |
| Control de vehículo | `k8s/tesla-control.yaml` | `harbor.e-dani.com/homelab/tesla-control-mcp:20260610-1` | ídem |

## 2. Dependencias, en ambos sentidos

- **Depende de** — Postgres compartido (`postgres-shared`), API de Tesla (OAuth), init containers `busybox:1.36` y `postgres:16-alpine`, Harbor.
- **Dependen de él** — `k8s-teslamate-mcp-pocharlies` (lee la BD), agentes que llaman a `tesla-control-mcp`; hosts `*.e-dani.com` en AdGuard.
- **ArgoCD** `teslamate`: repo `pocharlies-org/k8s-teslamate-pocharlies`, path y sync según Application viva (tronco **`main`**, `origin/main` = 40a576d).
  **Solape**: `k8s/tesla-control.yaml` (namespace `teslamate`, imagen `tesla-control-mcp:20260610-1`, proxy por digest) es el despliegue **vivo** de
  `tesla-control-mcp` y `tesla-vehicle-proxy`; `k8s-tesla-pocharlies` define los mismos servicios (y contiene su código y CI) pero sin Application. No desplegar ambos
  (misma clave de flota, mismo refresh token rotatorio). Consolidación: ver §8 de su doc.

## 3. Stack

| pieza | versión | para qué | no se usa en su lugar |
|---|---|---|---|
| TeslaMate | 3.0.0 | registro de viajes/carga | — |
| Grafana de TeslaMate | `latest` | dashboards (distinta de la Grafana de observabilidad) | Grafana central |
| Mosquitto 2 | — | MQTT | — |

## 4. Componentes compartidos

Postgres compartido (`k8s-infra-pocharlies/databases/postgres-shared`). IngressRoutes: `teslamate-lan` (LAN + SSO) y `teslamate-public` (catch-all de `tm.e-dani.com` tras `sso-chain`, INFRA-570) viven en `k8s/manifest.yaml` de este repo, sincronizados por el ArgoCD app `teslamate`; el servicio `tesla-static` al que apuntan las 2 rutas Tesla sigue aplicado a mano (copia en `dgx-infra/k8s/apps/edge/teslamate/`).

## 5. Cómo se construye aquí

Dos manifiestos planos (`manifest.yaml`, `tesla-control.yaml`). Tokens de Tesla vía ExternalSecret (nunca en el repo). Pin de `grafana:latest` pendiente.

## 6. Tests y validaciones

Sin tests; `kustomize build k8s` lo ejecuta el CI estándar.

## 7. CI/CD y despliegue

- `ci.yml` (reusable), `release.yml`, `pr-review.yml`. Merge a `main` → ArgoCD.
- **Validación en producción**:
  `kubectl -n teslamate get pods` (teslamate, mosquitto, grafana, tesla-control Ready) y
  `kubectl -n teslamate logs deploy/teslamate --tail=30 | grep -i -E "error|connected"`; en la UI de TeslaMate el último viaje con fecha reciente.
  (Namespace a confirmar con `kubectl get app teslamate -n argocd -o jsonpath='{.spec.destination.namespace}'`.)

## 8. Decisiones y trampas

- README de plantilla (k3s v1.32.5).
- `teslamate/grafana:latest`: un pull cambia la imagen sin PR.

## 9. Reautenticación (Tesla Fleet API)

> Medido = leído en `k8s/manifest.yaml`, `k8s/tesla-control.yaml` y en el spec de INFRA-488 (mediciones del 04-10-2026). **No medido** = marcado así. Ningún valor de token, client secret ni clave va en este fichero ni en git.

1. **Síntomas** — el log de TeslaMate solo muestra `GET / 302` y `GET /sign_in 200`, sin llamadas a `fleet-api`
   (`kubectl -n teslamate logs deploy/teslamate --since=3h`); `select max(date) from positions` queda viejo (04-10-2026: `2026-05-07`).
   Causa habitual: Tesla revoca el refresh token (medido en brain, abril-mayo 2026) o el token se generó contra la Owner API.
2. **Generar los tokens** — Fleet API, nunca Owner API: el `aud` del token debe ser el host de la Fleet API **EU**. Generadores que funcionaron
   (brain, 2026-05): `myteslamate.com/tesla-fleet-api` o la app «Auth app for Tesla». El client_id antiguo de `tesla_auth` está deshabilitado.
   Se necesitan Access Token + Refresh Token.
3. **Dónde se pegan** — en la UI de TeslaMate, `https://teslamate.lan.e-dani.com/sign_in` (IngressRoute `teslamate-lan`, solo LAN/tailnet/pod CIDR + SSO).
   TeslaMate guarda los tokens en su BD cifrados con `ENCRYPTION_KEY` (Secret `teslamate-secrets`, ExternalSecret desde 1Password): si esa clave cambia, el login falla (valor no medido en cluster).
   Host de la Fleet API en el Deployment: lo fija INFRA-568 (`TESLA_API_HOST`), aún **no** está en `main`.
4. **Secretos imperativos** (creados a mano desde el x86, nunca en git; `k8s/tesla-control.yaml:5-12`):
   `tesla-vehicle-proxy-config` (`fleet-key.pem`, `tls-cert.pem`, `tls-key.pem`), `tesla-control-mcp-env` (`TESLA_CLIENT_ID/SECRET`,
   `TESLA_INITIAL_REFRESH_TOKEN`, `TESLA_MCP_BEARER`), `tesla-control-mcp-tokens` (`tokens.json`). Lo que viene de ExternalSecret es solo `teslamate-secrets`.
   **No compartir el refresh token** entre TeslaMate y `tesla-control-mcp`: el refresh rota y se invalidarían mutuamente. Cada uno lleva el suyo.
   Las rutas públicas de `tm.e-dani.com` (clave pública Tesla y `/sign_in/callback`, dominio partner) no se tocan al reautenticar, y siguen públicas tras INFRA-570: solo la catch-all (la UI) quedó detrás de `sso-chain`.
5. **Verificar** (comando de C1, INFRA-488):
   `export KUBECONFIG=~/.kube/config; kubectl -n databases exec postgres-shared-2 -c postgres -- env PGHOST=/controller/run psql -U postgres -d teslamate -tAc "select now() - max(date) < interval '15 minutes' from positions"` → `t`.
   Con el coche dormido: `max(date)` posterior a la reautenticación y `kubectl -n teslamate logs deploy/teslamate --since=1h | grep -c -i "fleet-api"` ≥ 1.
   Refresco automático tras caducar el access token: no medido hasta que INFRA-488 lo confirme.
6. **Cuándo hace falta Dani** — solo el login de Tesla con MFA al generar los tokens (paso 2). Todo lo demás (pegar, verificar, secretos) lo hace la compañía.

Última verificación contra el código: 2026-10-05 · d457f33 (origin/main) · §9 pendiente de contrastar con el cluster tras INFRA-568
