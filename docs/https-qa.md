# HTTPS en QA (SCRUM-309 / HU-137)

## Solución final: Tailscale Funnel

QA se expone a internet vía **Tailscale Funnel**, no con una IP pública de
Azure. Se intentó primero con `cert-manager` + Let's Encrypt sobre una IP
pública compartida (ver sección "Camino descartado" abajo) — la cuota de
IPs públicas de la suscripción está agotada (3/3) y no se puede aumentar
por autoservicio (plan Free de soporte de Azure). Tailscale evita el
problema por completo: el pod hace una conexión **saliente** hacia la red
de Tailscale, sin necesitar ninguna IP pública ni puerto de entrada propio.

## Nombre público

`https://ecoruta-qa.tail47a5f7.ts.net` — asignado automáticamente por
Tailscale (variable `TS_HOSTNAME=ecoruta-qa` en
`qa/tailscale-funnel.yaml`, combinado con el nombre del tailnet). Si se
recrea el pod desde cero con un tailnet distinto, este nombre cambiaría —
en ese caso hay que actualizar `qa/backend-ingress.yaml`,
`qa/backend-sse-ingress.yaml` y `CORS_ORIGENES` en
`qa/backend-deployment.yaml` con el nuevo nombre.

## Quién lo administra

Equipo DevOps del proyecto (Seminario UMG), cuenta de Tailscale con correo
`umgseminario0@outlook.com`. Todo lo versionado en este repo:
`qa/tailscale-funnel.yaml` (Deployment + PVC de estado + config de Serve),
`qa/backend-ingress.yaml`, `qa/backend-sse-ingress.yaml`.

**Importante — configuración fuera de este repo, en la cuenta de
Tailscale:** hubo que agregar un permiso de política (ACL) que no viene
por defecto, desde **console.tailscale.com → Policies → JSON editor**:

```json
"nodeAttrs": [
  {
    "target": ["autogroup:member"],
    "attr": ["funnel"]
  }
]
```

Sin esto, el pod reporta "Funnel on" localmente pero el dominio nunca se
publica de verdad en DNS público (queda como si no existiera). Si algún
día Funnel deja de funcionar después de reconfigurar la política de
acceso, revisar primero que este bloque siga presente.

## Cómo funciona

- El Deployment `tailscale-funnel` (namespace `qa`) corre el contenedor
  oficial `tailscale/tailscale`, autenticado con una auth key guardada
  como Secret de Kubernetes (`tailscale-secret`, viene de la variable de
  CI/CD `TAILSCALE_AUTH_KEY`, enmascarada).
- Modo `TS_USERSPACE=true`: no necesita `NET_ADMIN` ni acceso a
  `/dev/net/tun`, alcanza para este caso (no enruta tráfico de otros pods).
- `TS_KUBE_SECRET=""`: el estado del nodo se guarda en el PVC
  (`tailscale-state`), no en un Secret de Kubernetes — evita necesitar
  permisos RBAC extra.
- `TS_SERVE_CONFIG` apunta a un `ConfigMap` con la config de Serve/Funnel:
  proxea todo el tráfico hacia `ingress-nginx-controller` (el mismo
  Ingress controller que ya usa el resto de QA), que a su vez enruta por
  path/host como siempre. Tailscale termina el TLS en su propio borde y le
  habla a `ingress-nginx` en HTTP simple — por eso los `Ingress` de QA ya
  **no** tienen sección `tls:` ni anotación de `cert-manager`.
- El certificado lo emite y renueva Tailscale automáticamente (vía ACME
  con desafío DNS-01 usando su propia infraestructura de DNS) — no
  depende de `cert-manager` ni de que ningún puerto de Azure esté
  accesible desde internet.

## Evidencia de que funciona (probado real, no solo desplegado)

```
$ curl -sv https://ecoruta-qa.tail47a5f7.ts.net/
< HTTP/1.1 200 OK
(HTML real de la app, certificado aceptado sin -k / sin advertencia)

$ curl https://ecoruta-qa.tail47a5f7.ts.net/api/v1/telemetria/posicion
HTTP_CODE:204   (comportamiento esperado, documentado, sin posicion aun)

$ curl -N https://ecoruta-qa.tail47a5f7.ts.net/api/v1/telemetria/stream
:latido   (el stream SSE conecta y entrega datos al instante, sin cortes)
```

## Si algo falla / renovación del certificado

Tailscale renueva el certificado solo, sin intervención manual. Para
diagnosticar problemas:

```bash
kubectl logs -n qa deploy/tailscale-funnel
kubectl exec -n qa deploy/tailscale-funnel -- tailscale funnel status
```

Causas más probables si deja de funcionar:
1. El bloque `nodeAttrs` de la política de Tailscale se borró o modificó
   (ver sección de arriba).
2. La auth key expiró o se revocó — hay que generar una nueva en
   Tailscale y actualizar la variable de CI/CD `TAILSCALE_AUTH_KEY`.
3. El PVC de estado (`tailscale-state`) se perdió — el pod se re-registra
   como un dispositivo nuevo (posible nombre distinto si hay colisión),
   hay que revisar el hostname asignado y actualizar los `Ingress` si
   cambió.

## Camino descartado: IP pública de Azure + cert-manager

Se intentó primero exponer QA con una IP pública de Azure y `cert-manager`
+ Let's Encrypt (patrón estándar de Kubernetes). No se pudo completar:

- La suscripción tiene un límite de 3 IPs públicas Standard SKU por
  región, ya agotado (2 en producción, 1 de salida en QA).
- Se intentó compartir la IP de salida existente de QA también para
  entrada — técnicamente válido en Azure, pero la IP quedó en un estado
  roto que nunca se pudo reparar (confirmado comparando contra la IP
  nativa de producción, que sí funciona; y recreando la configuración de
  Kubernetes desde cero sin éxito).
- El aumento de cuota está bloqueado por autoservicio (API rechaza con
  `ResourceNotAvailableForOffer`) y por ticket de soporte (`InvalidSupportPlan`,
  el plan de soporte es Free). Pendiente intentarlo desde el portal web,
  que a veces permite esto aunque la API lo bloquee.
- Recrear la IP pública completa desde cero también choca con la misma
  cuota: hace falta tener la IP vieja y la nueva a la vez por un momento
  para no perder la salida a internet, y eso pide una 4ª IP.

Si en el futuro se libera cupo de IPs públicas (aumento de cuota
aprobado, o se libera una de las 3 actuales), esta ruta queda disponible
como alternativa — los manifiestos de `cert-manager` ya no están en el
repo, habría que rehacerlos siguiendo el mismo patrón que se usó acá.
