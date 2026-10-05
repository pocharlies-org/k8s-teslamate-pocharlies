Rol: developer · Fecha: 2026-10-05 · Sesión: developer-INFRA-569 · Estado: LISTO

# INFRA-569 — Sondas httpGet HTTPS en tesla-vehicle-proxy

## Qué se hizo
`k8s/tesla-control.yaml`, Deployment `tesla-vehicle-proxy`: `readinessProbe` y `livenessProbe` pasan de `tcpSocket: 4443` a `httpGet: { path: /health, port: 4443, scheme: HTTPS }`. `initialDelaySeconds` y `periodSeconds` sin cambios. Nada más tocado (imagen, args, `-verbose`).
Las sondas del contenedor `tesla-control-mcp` (`tcpSocket: 8888`) no se tocan: no son TLS.

## Medición del path (solo lectura, hecha por devops, comentario 22972 de INFRA-569)
Desde el pod `tesla-control-mcp` contra `https://tesla-vehicle-proxy:4443`:
- `/health` → 200 OK, sin OAuth ni mTLS
- `/healthz`, `/readyz`, `/livez`, `/` → 403 (client did not provide an OAuth token)
Solo `/health` da 2xx; los demás harían fallar la sonda. El kubelet no verifica el certificado autofirmado en `httpGet` HTTPS.
Fuente upstream: no la he consultado en esta sesión (sin acceso de lectura a su código); la medición en vivo es la evidencia. Hipótesis no verificada: que `/health` lo sirva `cmd/tesla-http-proxy` de tesla/vehicle-command; revisar al validar.

## PR
Ver PR de la rama `INFRA-569-sondas-proxy` contra `main` (título con INFRA-569).

## Cómo verificar (tras el despliegue)
- `kubectl -n teslamate get deploy tesla-vehicle-proxy -o jsonpath='{.spec.template.spec.containers[0].readinessProbe}'` → httpGet HTTPS /health
- `kubectl -n teslamate logs deploy/tesla-vehicle-proxy --since=1h | grep -c "TLS handshake"` → 0
- `kubectl -n teslamate get pods -l app=tesla-vehicle-proxy` → Ready, 0 reinicios

## Checklist 00-spec.md
- [x] Sonda readiness httpGet HTTPS (en el diff; verificación en vivo tras merge)
- [ ] 0 «TLS handshake» tras el despliegue (post-merge, qa/devops)
- [ ] Pods Ready sin reinicios por sonda (post-merge)
- [x] PR contra `main` con INFRA-569 en el título; CI sin esperar

## Reutilizado
- Existente extendido: `k8s/tesla-control.yaml` (solo las dos sondas).
- Búsquedas: `rg -n tcpSocket k8s/tesla-control.yaml` (quedan solo las de 8888, correctas); `company-duplicados` → sin duplicación.
- Código nuevo: ninguno (sin sidecars ni scripts).

Documento afectado: `ARCHITECTURE.md` no se edita aquí (lo toca solo INFRA-572).
