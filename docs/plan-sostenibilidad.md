# Plan de Sostenibilidad post-entrega — EcoRuta (Devops-117 / SCRUM-48)

Cómo se mantiene EcoRuta vivo y a bajo costo después de la entrega del Seminario,
y cómo se traspasa a quien lo opere.

## 1. Costos recurrentes (detalle mensual)

> **Costo real (Azure Cost Management, sep‑2026, toda la suscripción = el proyecto):**
> **US$42.95 a la fecha (día 24)**, con **previsión de ~US$56.92 para el mes completo**
> (≈ **Q440–450/mes** a ~7.8 GTQ/USD). Patrón diario: base ~US$1/día con picos de
> hasta ~US$4/día en los días de más actividad. Es el gasto de **toda la infra
> (QA + Producción) ya con el apagado nocturno**; sin el scheduler sería varias
> veces más. Se consulta en el portal → *Análisis de costos* (ámbito de la
> suscripción), vista *AccumulatedCosts*.

> **Corrección importante:** la base de datos **no** es "Azure Database for
> PostgreSQL" (como asumía la HU). Corre **dentro del clúster** (Deployment
> `postgres` + disco administrado por PVC), tanto en QA como en Producción. Por
> eso **no hay una línea de DB gestionada aparte**: su costo va en el cómputo AKS
> + el disco PVC. El desglose por ítem y su driver, abajo.

| Ítem | Driver del costo | Control / nota |
|---|---|---|
| **AKS – nodos QA** (`aks-buses-dev`) | horas encendido × VM del nodepool | Scheduler los apaga 11pm–9pm GT (~2 h/día) |
| **AKS – nodos Producción** (`aks-buses-prod`) | horas encendido × VM del nodepool | Igual scheduler; **el grueso del gasto** |
| **PostgreSQL QA y Prod (in-cluster)** | cómputo del pod (dentro de AKS) + disco PVC | No es DB gestionada; sin línea aparte |
| **Discos administrados (PVC)** | GB aprovisionados (postgres, traccar, loki) | Tamaños chicos; vigilar retención de Loki |
| **Blob Storage** (respaldos) | GB de dumps almacenados | Política de ciclo de vida del storage |
| **Dominio** `mibusjalapa.lat` | ~$1.80 año 1 / **~$40.98 renovación** (~$3.4/mes amortizado) | Auto-renovación **APAGADA** |
| **Certificados TLS** | **$0** | Let's Encrypt + cert-manager |
| **Firebase** | Plan Spark (gratis) en el uso actual | Vigilar cuotas si crece |
| **Suscripción Azure** | umgseminario0@outlook.com ("Azure subscription 1") | — |

### Escenarios según la frecuencia de refresco del ETA

El refresco del ETA afecta el cómputo del backend y el tráfico SSE, **no** agrega
infraestructura (corre en el pod existente). Se ajusta en `application.yml`
(`ecoruta.eta.intervalo-minimo-recalculo-segundos` y la telemetría SSE):

| Escenario | Refresco | Impacto | Cuándo |
|---|---|---|---|
| **Alto** | recalcular ~cada 10 s (actual) | Más CPU del backend y más eventos SSE; sin costo extra de infra mientras el clúster ya está encendido | Operación normal en horario de servicio |
| **Medio** | ~cada 30 s | ~⅓ del cómputo de recálculo; ETA algo menos "en vivo" | Si el backend queda ajustado de CPU |
| **Bajo** | ~cada 60 s o más | Cómputo mínimo; ETA notoriamente rezagado | Pruebas o presupuesto muy ajustado |

Como los clústeres solo están encendidos ~2 h/día por el scheduler, el ETA **no**
genera costo fuera de esa ventana: la palanca real de ahorro es el **horario de
encendido**, no la frecuencia del ETA.

## 2. Renovaciones y vencimientos (calendario)

- **Dominio:** vence ~1 año tras la compra. Decidir antes si se renueva (ojo: la renovación del `.lat` es cara; evaluar `.com` u otro, o `.gob.gt` si la Muni lo tramita).
- **Certificados TLS:** automáticos (cert-manager), no requieren acción.
- **Credenciales:** rotar `ECORUTA_ADMIN_TOKEN`, contraseña admin de Traccar (hoy fuerte, antes era de fábrica) y claves de Tailscale/Firebase si cambia el equipo.

## 3. Mantenimiento mínimo

- **Trimestral:** verificar que las alertas lleguen, que los respaldos se generen (revisar el Blob), y que los certificados renueven.
- **Al cambiar de equipo:** traspasar accesos a Azure, GitHub (org `municipalidad-jalapa`), Namecheap, Firebase y Tailscale.
- **Actualizaciones:** imágenes fijadas por SHA; actualizar bajo demanda (Traccar, ingress-nginx, cert-manager) probando primero en QA.

## 4. Traspaso a la Municipalidad

- **Documentación:** este plan + `manual-de-operacion.md` + `plan-respuesta-incidentes.md` + guía de Traccar.
- **Accesos:** entregar cuentas/roles de Azure, GitHub, Namecheap, Firebase; o migrar los recursos a una suscripción de la Muni.
- **Dominio oficial:** si la Muni tramita `buses.jalapa.gob.gt` (`.gob.gt`), apuntar el DNS a las IPs de ingress y emitir el cert — el resto del stack no cambia.

## 5. Plan de bajo costo / apagado

- Si el proyecto queda inactivo: **dejar los clústeres apagados** (costo casi nulo) y no renovar el dominio → el sistema queda "dormido" sin gasto recurrente relevante.
- Para desmontar del todo: borrar los clústeres, el storage y liberar la IP estática; cancelar el dominio dejándolo vencer.

## 6. Riesgos de sostenibilidad y mitigación

| Riesgo | Mitigación |
|---|---|
| Nadie renueva el dominio | Recordatorio en el calendario; auto-renew apagado a propósito para no cobrar sorpresa |
| Se pierde el acceso a la suscripción Azure | Documentar cuenta dueña; agregar co-administradores |
| Costo sube por dejar clústeres encendidos | El scheduler los apaga; revisar que siga activo |
| El único que sabe operar se va | Este manual + runbooks reducen el "bus factor" |

---
*Mantenimiento post-entrega: DevOps — Evan / Marta / Victor, hasta el traspaso a la Muni.*
