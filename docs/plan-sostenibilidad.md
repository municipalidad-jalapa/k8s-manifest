# Plan de Sostenibilidad post-entrega — EcoRuta (Devops-117 / SCRUM-48)

Cómo se mantiene EcoRuta vivo y a bajo costo después de la entrega del Seminario,
y cómo se traspasa a quien lo opere.

## 1. Costos recurrentes

| Ítem | Costo aprox. | Cómo se controla |
|---|---|---|
| **Cómputo AKS** (2 clústeres) | El grueso del gasto | **Se apagan 11pm–9pm GT** (solo ~2h/día encendidos) vía Azure Automation → ahorro grande |
| **Dominio** `mibusjalapa.lat` | ~$1.80 año 1 / **~$40.98 renovación** | Auto-renovación **APAGADA**: vence al año si no se renueva a propósito |
| **Certificados TLS** | **Gratis** | Let's Encrypt, renovación automática por cert-manager |
| **Blob Storage** (respaldos) | Bajo | Retención por política de ciclo de vida del storage account |
| **Suscripción Azure** | umgseminario0@outlook.com ("Azure subscription 1") | — |

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
