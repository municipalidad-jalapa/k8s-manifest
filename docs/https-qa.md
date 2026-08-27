# HTTPS en QA (SCRUM-309 / HU-137)

## Nombre DNS asignado

`ecoruta-qa-app.westus2.cloudapp.azure.com` — etiqueta DNS de Azure sobre la
IP pública **de salida** de `aks-buses-dev` (la que usa `aksOutboundRule`
para el egreso a internet del cluster), reutilizada también como entrada
para el Ingress. Gratis, sin dominio comprado.

**Importante:** la suscripción de Azure tiene un límite de 3 IPs públicas
por región, y las otras 2 ya las usa `aks-buses-prod` (una de salida, una
de entrada). Por eso el Ingress de QA NO tiene su propia IP dedicada —
comparte la de salida existente vía las anotaciones del Service:

```yaml
service.beta.kubernetes.io/azure-load-balancer-ipv4: "4.154.249.165"
```

en `ingress-nginx-controller` (namespace `ingress-nginx`, no versionado en
este repo porque se instaló con Helm fuera de git). **No borrar esa IP
pensando que no tiene tráfico real** — el load balancer de Azure la usa
para dos cosas a la vez (entrada del Ingress y salida del cluster), y
`az network public-ip delete` lo va a rechazar igual, pero primero hay que
sacarla como frontend del Ingress si algún día se necesita liberarla.

Si se recrea la IP pública (por ejemplo al recrear el node pool desde
cero) hay que volver a asignarle la etiqueta:

```bash
az network public-ip update -g <resource-group-de-los-nodos> -n <nombre-de-la-ip> --dns-name ecoruta-qa-app
```

## Quién lo administra

Equipo DevOps del proyecto (Seminario UMG). El `ClusterIssuer` y los
`Ingress` con TLS están versionados en este repo (`qa/cert-manager-issuer.yaml`,
`qa/backend-ingress.yaml`, `qa/backend-sse-ingress.yaml`).

## Cómo funciona

- `cert-manager` (instalado con el job manual `install-cert-manager` del
  pipeline, release oficial de terceros, no vendorizado en este repo)
  emite el certificado vía Let's Encrypt, desafío HTTP-01 a través del
  mismo `ingress-nginx` que ya está corriendo.
- El certificado dura 90 días; `cert-manager` intenta renovarlo
  automáticamente unos 30 días antes de que expire. No requiere
  intervención manual en el caso normal.
- `ingress-nginx` redirige HTTP → HTTPS automáticamente en cuanto el
  `Ingress` tiene una sección `tls:` (no hace falta ninguna anotación
  aparte para esto).

## Si la renovación falla

Let's Encrypt manda un correo de aviso a `umgseminario0@outlook.com`
(la dirección de registro del `ClusterIssuer`) unos 20 días antes de que
el certificado expire, si detecta que la renovación automática no está
funcionando — es el único mecanismo de alerta, no hay nada propio del
proyecto que lo revise activamente.

Para diagnosticar manualmente:

```bash
kubectl describe certificate ecoruta-qa-tls -n qa
kubectl describe challenge -n qa
```

Causa más común: se perdió la etiqueta DNS de la IP pública (ver sección
de arriba) — sin ella, Let's Encrypt no puede validar el dominio.
