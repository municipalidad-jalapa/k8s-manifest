# HTTPS en QA (SCRUM-309 / HU-137)

## Nombre DNS asignado

`ecoruta-qa.westus2.cloudapp.azure.com` — etiqueta DNS de Azure sobre la
IP pública del Ingress de `aks-buses-dev` (gratis, sin dominio comprado).
La etiqueta vive en la IP pública, no en el clúster: si se recrea la IP
pública (por ejemplo al recrear el node pool desde cero) hay que volver a
asignarla con:

```bash
az network public-ip update -g <resource-group-de-los-nodos> -n <nombre-de-la-ip> --dns-name ecoruta-qa
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
