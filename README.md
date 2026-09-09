# k8s-manifest — Infraestructura de EcoRuta

Manifiestos de Kubernetes y automatización de despliegue de EcoRuta
(Municipalidad de Jalapa) sobre Azure Kubernetes Service.

## Entornos

| Entorno | Cluster AKS | Namespace | Acceso público |
|---|---|---|---|
| QA | `aks-buses-dev` | `qa` | https://ecoruta-qa.tail47a5f7.ts.net (Tailscale Funnel) |
| Producción | `aks-buses-prod` | `production` | IP pública de Azure, sin dominio ni HTTPS todavía |

Ambos clusters se **apagan fuera del horario de operación** (7am–8pm hora de
Guatemala) para no pagar cómputo ocioso: ~USD 25/mes menos entre los dos.

## Workflows (GitHub Actions)

| Workflow | Cuándo corre | Qué hace |
|---|---|---|
| `validar-manifiestos.yml` | PR que toca `qa/` o `production/` | `kubectl apply --dry-run` de ambos entornos |
| `desplegar-qa.yml` | Push a `main` que toca `qa/` | Crea los Secrets y aplica `qa/` |
| `desplegar-produccion.yml` | Push a `main` que toca `production/` | Igual, en producción. **Requiere aprobación** (environment `production`) |
| `encender-clusters.yml` | 13:00 UTC (7am GT) | Enciende los dos clusters |
| `apagar-clusters.yml` | 02:00 UTC (8pm GT) | Apaga los dos clusters |
| `respaldo-programado.yml` | 03:00 UTC | Respaldo de Postgres de ambos entornos, encendiendo el cluster si hace falta |

Todos se pueden disparar a mano desde la pestaña **Actions** (`workflow_dispatch`).

## Autenticación a Azure

Los workflows usan **OIDC con credenciales federadas**: no hay ningún secreto
de Azure guardado en GitHub. El token que reciben dura lo que dura el job.

El Service Principal es `sp-ecoruta-backup-scheduler` (rol *Azure Kubernetes
Service Contributor* sobre `rg-buses-jalapa`). Sus credenciales federadas
apuntan a subjects que incluyen los IDs inmutables de la organización y el
repositorio, por ejemplo:

```
repo:municipalidad-jalapa@326809456/k8s-manifest@1362226172:environment:qa
```

Si se recrea el repo o la organización, esos IDs cambian y hay que volver a
registrar las credenciales federadas en Azure AD.

## Secretos y variables

Se definen por **Environment** (`qa` / `production`) en Settings del repo, no
en el código. Los Secrets se crean en el cluster desde el workflow con el
patrón `kubectl create secret ... --dry-run=client -o yaml | kubectl apply -f -`,
que permite actualizarlos si ya existen.

| Nombre | Tipo | Uso |
|---|---|---|
| `DB_PASSWORD`, `JWT_SECRET` | secret | Secret `db-secrets` |
| `ECORUTA_ADMIN_TOKEN` | secret (solo qa) | Secret `db-secrets` |
| `AZURE_STORAGE_ACCESS_KEY` | secret | Secret `azure-backup-secret` (respaldos) |
| `TAILSCALE_AUTH_KEY`, `TAILSCALE_API_TOKEN` | secret (solo qa) | Secret `tailscale-secret` |
| `ALERTMANAGER_SMTP_PASSWORD` | secret (solo producción) | Secret `alertmanager-smtp-secret` |
| `FIREBASE_ADMIN_SDK_JSON` | secret | Secret `firebase-secret` (JSON crudo, multilínea) |
| `FIREBASE_PROJECT_ID` | variable | Secret `firebase-secret` |
| `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID`, `ACR_LOGIN_SERVER` | variables de organización | Login OIDC y registry |

## Documentación

- [`docs/https-qa.md`](docs/https-qa.md) — cómo se expone QA por HTTPS con
  Tailscale Funnel, y el camino de Azure/cert-manager que se descartó.
- [`docs/restaurar-respaldo-postgres.md`](docs/restaurar-respaldo-postgres.md) —
  procedimiento de restauración de un respaldo de Postgres.

## Historia

Este proyecto vivía en GitLab. Se migró a GitHub porque el grupo privado
excedía el límite de 5 usuarios del plan gratuito de GitLab y quedó en modo
solo lectura. El acceso al cluster ya no usa el Agente de Kubernetes de
GitLab (que no tiene equivalente en GitHub): ahora es OIDC + `az aks
get-credentials`.
