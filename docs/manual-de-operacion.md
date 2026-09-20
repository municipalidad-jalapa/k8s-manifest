# Manual de Operación — EcoRuta (Devops-107 / SCRUM-38)

Cómo operar el sistema EcoRuta día a día: dónde vive cada cosa, cómo prenderla,
desplegarla, respaldarla y diagnosticarla. Entornos: **QA** (`aks-buses-dev`) y
**Producción** (`aks-buses-prod`), ambos en Azure, resource group `rg-buses-jalapa`.

## 1. Mapa del sistema

| Componente | QA (ns `qa`) | Producción (ns `production`) |
|---|---|---|
| Frontend (React) | `frontend-app-buses` | `frontend-app-buses` |
| Backend (Spring Boot) | `backend-api-buses` | `backend-api-buses` |
| Base de datos | `postgres` (StatefulSet + PVC) | idem |
| Ingress público | nginx, IP `4.154.249.165` | nginx, IP `20.83.88.147` |
| GPS (Traccar) | `traccar`, IP estática `20.114.56.168` (8082 web, 5001 gps103) | — |
| Monitoreo | — | `prometheus` + `alertmanager` |
| URLs | https://qa.mibusjalapa.lat | https://mibusjalapa.lat |

Dominios: `qa.mibusjalapa.lat` (develop) y `mibusjalapa.lat` (main), con **TLS de
Let's Encrypt** emitido y renovado automáticamente por **cert-manager**.

## 2. Prender / apagar los clústeres

Los clústeres se **apagan de noche para ahorrar** y encienden solos:
- Automatización: **Azure Automation `auto-ecoruta`** (runbook `gestion-clusters`).
- Horario (America/Guatemala): **encender 9:00pm**, **apagar 11:00pm**.
- Manual: `az aks start -n aks-buses-dev -g rg-buses-jalapa` (o `--no-wait`); igual con `aks-buses-prod`.
- Si `kubectl` da "no such host": el clúster está apagado (su DNS de API server se retira al detenerlo).

## 3. Desplegar

CI/CD en **GitHub Actions** (migrado desde GitLab). Ramas: **`develop` → QA**, **`main` → Producción**.

- **Backend** (`api-buses-jalapa`): push a `develop` construye y despliega a QA; push a `main` a producción (`kubectl set image ...:<sha>`).
- **Frontend** (`app-movil-buses`): push a `develop` construye+despliega a QA. **Producción NO tiene deploy automático**: se despliega a mano → `kubectl --context aks-buses-prod set image deployment/frontend-app-buses frontend-app-buses=acrbusesjalapa.azurecr.io/frontend-app-buses:<sha> -n production`. La imagen se pinnea en `k8s-manifest/production/frontend-deployment.yaml`.
- **Manifiestos** (`k8s-manifest`): `deploy-qa` / `deploy-production` aplican `qa/` y `production/`.
- **Variable clave del frontend:** `VITE_API_BASE_URL` (GitHub → Variables por entorno) debe apuntar al dominio propio del entorno (`https://qa.mibusjalapa.lat` / `https://mibusjalapa.lat`), si no el navegador bloquea por contenido mixto.

## 4. Respaldos de la base

- **Respaldo nocturno** a Azure Blob Storage vía Azure Automation (runbook `respaldo-postgres`, **10:10pm GT**): enciende el clúster si hace falta, dispara el `Job` desde el CronJob `postgres-backup` con `runCommand`, y reapaga si él lo prendió.
- **Manual:** `kubectl create job respaldo-$(date +%s) --from=cronjob/postgres-backup -n <ns>`.
- **Restaurar:** ver `k8s-manifest/docs/restaurar-respaldo-postgres.md`.

## 5. GPS / Traccar

- El TK103A reporta por gps103/TCP a `20.114.56.168:5001`. Consola/API en `http://20.114.56.168:8082`.
- Los dispositivos auto-registrados nacen **sin dueño**: hay que enlazarlos a un usuario admin para verlos (ver guía Traccar).
- Guía completa para Desarrollo: `traccar-guia-desarrollo.md`.

## 6. Observabilidad y diagnóstico

- **Prometheus** (prod) scrapea el backend y evalúa alertas; **Alertmanager** las enruta.
- Alertas: `BackendCaido` (backend no responde 2m), `TelemetriaDetenida` (sin telemetría 15m en horario).
- **Logs correlacionados:** cada petición lleva un `X-Correlation-Id` en todas sus líneas de log (filtro `CorrelationIdFilter`); útil para seguir una posición de ingesta a difusión.
- Ver logs: `kubectl logs deploy/backend-api-buses -n <ns> --tail=100` (o `-f`). Traccar: su log está en `/opt/traccar/logs/tracker-server.log` dentro del pod.
- Salud rápida: `GET /actuator/health` (público). Pods: `kubectl get pods -n <ns>`.

## 7. Runbook rápido de incidentes comunes

| Síntoma | Causa probable | Acción |
|---|---|---|
| El sitio no carga | clúster apagado | encender el clúster (§2) |
| "Sin datos nuevos" en el navegador | bundle viejo cacheado o `VITE_API_BASE_URL` incorrecta | Ctrl+Shift+R / limpiar SW; verificar la variable y redeploy |
| Cert "no seguro" al recién emitir | cert falso cacheado durante la emisión | recargar; cert-manager ya sirve el válido |
| Pod en CrashLoopBackOff | config/DB | `kubectl logs --previous`; revisar ConfigMap/Secret |
| Traccar no muestra un equipo | dispositivo sin dueño | enlazarlo al usuario admin |

## 8. Cuentas y secretos (no en este repo)

Credenciales (DB, JWT, `ECORUTA_ADMIN_TOKEN`, Tailscale, Firebase, Traccar admin,
`eq_...` de equipos) viven en **GitHub Secrets por entorno** y en **Secrets de
Kubernetes**; las de demo/uso las custodia Evan (DevOps). Nunca en el repositorio.

---
*Responsable de operación (incidencias 24h): DevOps — Evan / Marta / Victor.*
