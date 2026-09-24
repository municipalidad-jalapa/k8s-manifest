# Plan de Respuesta ante Incidentes — EcoRuta (Devops-104 / SCRUM-35)

Qué hacer cuando algo falla en producción, para restaurar el servicio rápido y
aprender del incidente. Complementa el runbook del Manual de Operación (§7).

## 1. Severidades

| Sev | Definición | Ejemplo | Objetivo de atención |
|---|---|---|---|
| **S1 — Crítico** | El servicio no funciona para los usuarios | Sitio caído, backend no responde, DB caída | Empezar **< 30 min**, 24h de cobertura |
| **S2 — Alto** | Función principal degradada | No llega telemetría, no cargan rutas | < 2 h en horario |
| **S3 — Menor** | Falla parcial o cosmética | Un panel del dashboard, un dispositivo sin enlazar | Siguiente día hábil |

## 2. Roles

- **Responsable de guardia (on-call):** DevOps (Evan / Marta / Victor). Primer contacto ante una alerta.
- **Escalamiento:** si en 30 min (S1) no hay avance, avisar al resto de DevOps y al líder del proyecto.
- **Comunicación:** el on-call informa estado a la Municipalidad/UMG cuando un S1/S2 dura más de 1 h.

## 3. Detección

- **Alertas automáticas** (Prometheus/Alertmanager): `BackendCaido`, `TelemetriaDetenida`.
- **Salud manual:** `GET https://mibusjalapa.lat/actuator/health`, `kubectl get pods -n production`.
- Reportes de usuarios/evaluadores.

## 4. Procedimiento (S1/S2)

1. **Confirmar** el alcance: ¿público o interno? ¿un componente o todo? (`kubectl get pods`, health, logs).
2. **Contener/mitigar** con el runbook (§7 del Manual): reiniciar pod (`kubectl rollout restart deploy/<x> -n production`), encender clúster si aplica, revertir imagen a la anterior si un deploy la rompió (`kubectl set image ...:<sha-anterior>`).
3. **Restaurar datos** si hubo pérdida: restaurar el último respaldo (`docs/restaurar-respaldo-postgres.md`).
4. **Verificar** recuperación: health 200, pods Running, la app carga.
5. **Registrar** hora de inicio/fin, causa y acciones (para el post-mortem).

## 5. Incidentes típicos y su fix

| Incidente | Fix |
|---|---|
| Deploy rompió producción | Revertir a la imagen previa (`kubectl set image` al `sha` anterior) |
| Backend "caído" en Prometheus pero vivo | Revisar el scrape (`/actuator/prometheus`); no siempre es caída real |
| Certificado inválido | `kubectl describe certificate -n <ns>`; si en backoff, borrar el `certificate` para reemisión |
| Base corrupta / pérdida | Restaurar respaldo nocturno desde Blob Storage |
| Clúster no responde | Encenderlo (`az aks start`); su DNS vuelve en minutos |

## 6. Acciones que el conductor puede tomar por sí solo

Delimitado a propósito: el conductor **no** toca servidores, credenciales ni la
configuración del GPS. Solo lo que puede resolver desde la unidad:

- **La app no carga o no actualiza:** cerrar y reabrir la app; verificar señal /
  datos móviles del dispositivo a bordo y reintentar.
- **No aparece la ubicación del bus:** confirmar que el equipo GPS esté encendido
  y con luz de señal; esperar 1–2 min a que reporte.
- **El dispositivo a bordo se colgó:** reiniciarlo (apagar/encender) y reabrir la app.
- **Sigue sin funcionar tras lo anterior:** avisar al on-call de DevOps con: hora,
  qué ve en pantalla, número de unidad/ruta y si hay señal. No seguir intentando arreglos.

**Lo que el conductor NO debe hacer:** reconfigurar el GPS, cambiar credenciales,
ni intervenir la infraestructura o los servidores (eso es del on-call).

## 7. Post-mortem (tras cada S1)

Breve, sin culpas: **qué pasó**, **impacto** (tiempo, usuarios), **causa raíz**,
**cómo se resolvió**, **qué hacer para que no se repita** (acción con responsable).

## 8. Respaldo del propio plan

Backups nocturnos automáticos (10:10pm GT) + IP estática del GPS + TLS con
renovación automática reducen la probabilidad de varios S1. Revisar trimestralmente
que las alertas sigan llegando (probar `TelemetriaDetenida` en un entorno).

---
*Contacto de guardia: DevOps — Evan / Marta / Victor.*
