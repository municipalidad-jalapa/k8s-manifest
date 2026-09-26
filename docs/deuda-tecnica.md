# Deuda técnica — EcoRuta (SCRUM-47 / Desarrollo-116)

Inventario de lo que quedó a medias y por qué, priorizado para quien continúe el
proyecto. Estado al 2026-09-26, después del release a producción (main = develop).

**Esfuerzo:** S = menos de medio día · M = 1–2 días · L = más de 3 días.
**Prioridad:** P1 = resolver antes del piloto real / traspaso · P2 = pronto · P3 = cuando se pueda.

## P1 — Antes del piloto real o del traspaso

| # | Deuda | Impacto si no se atiende | Esfuerzo | Cómo resolverla |
|---|---|---|---|---|
| 1 | **Los clústeres solo están encendidos de 21:00 a 23:00 (GT).** Decisión vigente para ahorrar costo. | Producción no está disponible durante el día: un piloto con pasajeros reales no funciona. | S | Cambiar los schedules `encender-clusters` / `apagar-clusters` de Azure Automation (`auto-ecoruta`) al horario de servicio. Sube el costo mensual. |
| 2 | **El GPS físico solo llega a QA.** Traccar existe solo en `aks-buses-dev`; decisión vigente. | En producción no se ve el bus real; solo posiciones cargadas por otros medios. | M | Desplegar Traccar en `production` con IP estática propia (~US$3–4/mes) y forward al backend de prod (mismo patrón que QA: Secret `TRACCAR_TOKEN` + envsubst), y reconfigurar el equipo. |
| 3 | **Puerto del protocolo del GPS.** El LoadBalancer de Traccar expone solo **5001 (gps103)**. El Coban TK103A/B real suele hablar **tk103 en 5002**, y algunos clones hablan gt06/h02 (ver `docs/qa/HU-145-prueba-integracion-gps.md`). | Si el equipo físico usa tk103 u otro protocolo, sus tramas no llegan a Traccar. | S | Confirmar el protocolo del equipo (manual o SMS de configuración) y exponer ese puerto en el Service de `qa/traccar.yaml`. |
| 4 | **Traccar usa la base H2 embebida** (archivo en el PVC). | Riesgo de corrupción o pérdida del historial de posiciones y dispositivos; para escribir hay que apagar el pod. | M | Pasar Traccar a PostgreSQL (una base aparte en el mismo servidor) con `database.driver/url` en su ConfigMap. |
| 5 | **Credenciales a rotar antes del traspaso.** Contraseña de demo débil (`Admin123`), `ECORUTA_ADMIN_TOKEN` provisional, `TRACCAR_TOKEN` de QA, PAT con permiso de bypass guardado en un archivo local, y credenciales en documentos locales. | Quien tuvo acceso al proyecto conserva acceso después del traspaso. | S–M | Rotar cada secreto (GitHub Secrets / Firebase / Traccar), revocar el PAT y entregar las credenciales por un gestor de contraseñas. |
| 6 | **La restauración de un respaldo nunca se probó completa.** Los respaldos nocturnos sí corren (verificado). | Un respaldo que no se sabe restaurar no protege. | S | Restaurar el último respaldo en una base temporal de QA siguiendo `docs/restaurar-respaldo-postgres.md` y dejar registrado el tiempo que toma. |

## P2 — Pronto

| # | Deuda | Impacto | Esfuerzo | Cómo resolverla |
|---|---|---|---|---|
| 7 | **El deploy del frontend a producción es manual** (`kubectl set image`). | Paso fácil de olvidar; el frontend de prod puede quedar atrás del backend. | S | Agregar un job `desplegar-produccion` con gate de aprobación en `app-movil-buses/.github/workflows/desplegar.yml`, igual al del backend. |
| 8 | **Pushes directos a `main` saltando la protección de ramas** (el PAT es admin). | Cambios sin revisión entran a la rama de producción. | S | Trabajar siempre por Pull Request y quitarle al token el permiso de bypass. |
| 9 | **`production/backend-deployment.yaml` dice `:latest`.** Mitigado: `desplegar-produccion` ahora conserva la imagen viva. | Si el Deployment se recrea desde cero, tomaría el último build de cualquier rama. | S | Cambiar el manifiesto a un tag por SHA. |
| 10 | **El GPS no tiene TLS** (consola en `http://…:8082`) y **`registerUnknown` sigue activo**. | Credenciales de la consola viajan sin cifrar; cualquier dispositivo que reporte queda registrado. | S–M | Poner la consola detrás del ingress con TLS; apagar `registerUnknown` y dar de alta los equipos a mano. |
| 11 | **Una sola réplica por servicio** y sin alta disponibilidad. | Si un pod cae, el servicio se interrumpe hasta que se levanta otro. | M | Subir réplicas del backend/frontend y definir PodDisruptionBudgets (sube costo). |
| 12 | **El dominio vence en ~sept 2027** con auto-renovación apagada (política: nada pago se renueva solo). | Si nadie lo renueva, cae la web, el TLS y el subdominio del GPS. | S | Recordatorio en el calendario del responsable de la Municipalidad. |

## P3 — Cuando se pueda

| # | Deuda | Impacto | Esfuerzo | Cómo resolverla |
|---|---|---|---|---|
| 13 | `VITE_GOOGLE_MAPS_API_KEY` es obligatoria en el build pero el código usa OpenStreetMap. | Si la clave expira o se borra, el build del frontend falla. | S | Quitarla de `VITE_REQUIRED_BUILD_ARGS` y del Dockerfile. |
| 14 | El módulo `@amiceli/vitest-cucumber` no está en los `node_modules` locales (en CI sí). | Las pruebas de features no corren en local. | S | `npm ci` limpio y verificar el lockfile. |
| 15 | La cuenta `superadmin` no está enlazada a Firebase ni en QA ni en prod. | Nadie puede crear cuentas desde el panel (rol SUPERADMIN). | S | Crear el usuario en Firebase y definir `ECORUTA_SUPERADMIN_FIREBASE_UID` al desplegar. |
| 16 | **Migración pendiente al rastreador comercial.** El piloto usa un rastreador de bajo costo (serie TK103) con Traccar autohospedado; no hay una decisión documentada sobre el equipo comercial definitivo. | Sin decidir el equipo, no se puede cerrar la configuración de protocolo, puerto ni alta de dispositivos para la flota real. | M–L | La integración ya está desacoplada: Traccar recibe el protocolo y reenvía al backend (adaptador SCRUM-24), así que cualquier rastreador soportado por Traccar sirve cambiando protocolo/puerto y dando de alta el dispositivo. Si se opta por una plataforma comercial con su propia nube, usar su webhook hacia el mismo endpoint. |

## Resuelto recientemente (para contexto)

- Release a producción (backend `37345a3b`, frontend `7f308c9a`, Postgres con pgRouting, 22 migraciones).
- Tag-drift en QA y prod: los deploys de manifiestos conservan la imagen viva.
- Avisos push (clave VAPID), caché del sitio (`index.html` sin caché), dominios autorizados en Firebase.
- Versión de Traccar fijada en `6.15.3` (antes `:latest`).
