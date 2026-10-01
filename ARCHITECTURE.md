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

Postgres compartido (`k8s-infra-pocharlies/databases/postgres-shared`); IngressRoutes en `k8s-infra-pocharlies`.

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

Última verificación contra el código: 2026-10-01 · 40a576d (origin/main)
