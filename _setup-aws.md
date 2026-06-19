# Setup AWS para docs.revolupay.es

Documento interno (no se publica). Pasos a ejecutar UNA SOLA VEZ en la consola AWS para montar la infraestructura que sirve `https://docs.revolupay.es`.

**Región:** `eu-west-1` para S3 y todo lo demás. **ACM cert para CloudFront DEBE ir en `us-east-1`.**

---

## 1. S3 — bucket privado

| Campo | Valor |
|---|---|
| Bucket name | `revolupay-api-docs-prod` |
| Region | `eu-west-1` |
| Block public access | **TODOS los 4 toggles ON** |
| Versioning | Enable |
| SSE | SSE-S3 (AES-256) |
| Object Ownership | Bucket owner enforced |

No habilitar "Static website hosting" — usaremos CloudFront con OAC. El bucket queda 100% privado.

---

## 2. ACM cert para `docs.revolupay.es`

**OJO — Region: us-east-1 (N. Virginia)**. CloudFront solo lee certs de ahí.

1. ACM → Request certificate.
2. Domain: `docs.revolupay.es`.
3. Validation: DNS.
4. Crea el CNAME que te indique en Route53 (zona `revolupay.es`).
5. Espera a que el cert pase a `Issued` (5-30 min).
6. Anota el ARN del cert.

---

## 3. CloudFront distribution

| Campo | Valor |
|---|---|
| Origin domain | `revolupay-api-docs-prod.s3.eu-west-1.amazonaws.com` |
| Origin access | **Origin access control settings (recommended)** → Create new OAC, sign requests, S3 → "Yes, update the bucket policy" cuando lo ofrezca al final |
| Viewer protocol policy | **Redirect HTTP to HTTPS** |
| Allowed HTTP methods | GET, HEAD |
| Cache policy | **CachingOptimized** (managed) |
| Origin request policy | (none) |
| Response headers policy | (nueva — ver paso 4) |
| Alternate domain names (CNAMEs) | `docs.revolupay.es` |
| SSL certificate | el ARN del paso 2 |
| Security policy (TLS) | TLSv1.2_2021 |
| Default root object | `index.html` |
| Standard logging | On, bucket `revolupay-logs-prod`, prefix `cloudfront/docs/` |
| Price class | "Only North America and Europe" (suficiente y barato) |

### Custom error responses (para SPAs/Redoc routing)

| HTTP error code | Response page path | HTTP response code | TTL |
|---|---|---|---|
| 403 | `/index.html` | 404 | 60 |
| 404 | `/index.html` | 404 | 60 |

Esto sirve la landing page si alguien pide una ruta inexistente, en vez de mostrar el XML feo de S3.

---

## 4. Response headers policy — Security

CloudFront → Policies → Response headers → Create.

Name: `revolupay-docs-security`.

```json
{
  "ResponseHeadersPolicyConfig": {
    "Name": "revolupay-docs-security",
    "Comment": "Security headers for docs.revolupay.es (DORA / PW-79)",
    "SecurityHeadersConfig": {
      "StrictTransportSecurity": {
        "AccessControlMaxAgeSec": 31536000,
        "IncludeSubdomains": false,
        "Preload": false,
        "Override": true
      },
      "XSSProtection": {
        "Protection": true,
        "ModeBlock": true,
        "Override": true
      },
      "FrameOptions": {
        "FrameOption": "SAMEORIGIN",
        "Override": true
      },
      "ContentTypeOptions": {
        "Override": true
      },
      "ReferrerPolicy": {
        "ReferrerPolicy": "strict-origin-when-cross-origin",
        "Override": true
      },
      "ContentSecurityPolicy": {
        "ContentSecurityPolicy": "default-src 'self'; script-src 'self' 'unsafe-inline' blob: https://cdn.jsdelivr.net; style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; img-src 'self' data: https:; font-src 'self' data: https://fonts.gstatic.com; worker-src 'self' blob:; object-src 'none'; base-uri 'self'; frame-ancestors 'self'",
        "Override": true
      }
    },
    "CustomHeadersConfig": {
      "Items": [
        {
          "Header": "Permissions-Policy",
          "Value": "geolocation=(), camera=(), microphone=()",
          "Override": true
        }
      ]
    }
  }
}
```

**Notas CSP:**
- `'unsafe-inline'` en script/style: Redoc inyecta CSS y a veces scripts inline. Si te molesta, otra opción es usar nonces (más complejo).
- `blob:` + `worker-src 'self' blob:`: Redoc usa workers para parsing del spec grande.
- `cdn.jsdelivr.net` + `fonts.googleapis.com/gstatic.com`: Redoc puede pedir font/icons. Si bundleas todo localmente con `--theme`, se pueden quitar.

Asocia esta policy a la default behavior de la distribution.

---

## 5. Bucket policy — solo CloudFront OAC

Después de crear la distribution CloudFront, AWS te ofrece "Copy policy" para el bucket. Aplícala. Queda así (sustituye `<ACCOUNT_ID>` y `<DIST_ID>`):

```json
{
  "Version": "2008-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudFrontServicePrincipal",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudfront.amazonaws.com"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::revolupay-api-docs-prod/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::<ACCOUNT_ID>:distribution/<DIST_ID>"
        }
      }
    }
  ]
}
```

Adicionalmente, una segunda policy para forzar TLS (defense in depth):

```json
{
  "Sid": "DenyInsecureTransport",
  "Effect": "Deny",
  "Principal": "*",
  "Action": "s3:*",
  "Resource": [
    "arn:aws:s3:::revolupay-api-docs-prod",
    "arn:aws:s3:::revolupay-api-docs-prod/*"
  ],
  "Condition": {
    "Bool": {
      "aws:SecureTransport": "false"
    }
  }
}
```

---

## 6. Route53

Zona `revolupay.es`:

| Record | Type | Alias to | Routing |
|---|---|---|---|
| `docs.revolupay.es` | A | CloudFront distribution `d1234.cloudfront.net` (selector "Alias to CloudFront distribution") | Simple |
| `docs.revolupay.es` | AAAA | misma CloudFront distribution | Simple |

(AAAA opcional pero recomendado: clientes IPv6 evitan un hop.)

---

## 7. IAM role para GitHub Actions OIDC

(Opcional pero recomendado vs guardar AWS access keys en GitHub secrets.)

### 7.1 OIDC provider en IAM

IAM → Identity providers → Add provider.
- Provider type: OpenID Connect
- Provider URL: `https://token.actions.githubusercontent.com`
- Audience: `sts.amazonaws.com`

### 7.2 Rol con trust policy

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
      },
      "StringLike": {
        "token.actions.githubusercontent.com:sub": "repo:rp-projects/api-docs:*"
      }
    }
  }]
}
```

### 7.3 Permission policy del rol

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "WriteApiDocsBucket",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::revolupay-api-docs-prod",
        "arn:aws:s3:::revolupay-api-docs-prod/*"
      ]
    },
    {
      "Sid": "InvalidateCloudFront",
      "Effect": "Allow",
      "Action": "cloudfront:CreateInvalidation",
      "Resource": "arn:aws:cloudfront::<ACCOUNT_ID>:distribution/<DIST_ID>"
    }
  ]
}
```

Nombre sugerido: `revolupay-api-docs-deployer`. Anota el ARN.

### 7.4 GitHub secrets (`rp-projects/api-docs`)

| Name | Value |
|---|---|
| `AWS_ROLE_TO_ASSUME` | ARN del rol del paso 7.2 |
| `AWS_REGION` | `eu-west-1` |
| `S3_BUCKET` | `revolupay-api-docs-prod` |
| `CLOUDFRONT_DISTRIBUTION_ID` | el ID `E1ABCD...` |

---

## 8. Primer deploy manual de prueba (antes del repo)

Para validar la cadena S3 → CloudFront → ACM → Route53 sin esperar al GHA:

```bash
echo '<h1>docs.revolupay.es OK</h1>' > /tmp/index.html
aws s3 cp /tmp/index.html s3://revolupay-api-docs-prod/index.html
aws cloudfront create-invalidation --distribution-id <DIST_ID> --paths '/*'
curl -sI https://docs.revolupay.es/ | head -15
```

Esperado: `HTTP/2 200`, `Server: CloudFront`, `Strict-Transport-Security: max-age=31536000`.

---

## 9. Migración del bucket actual `docs.revolupay.es`

Tienes ahora servido en `http://docs.revolupay.es.s3-website-eu-west-1.amazonaws.com/swagger-ui/...`.

Una vez la nueva infra esté arriba:

1. Verificar nuevo setup OK con `curl -sI https://docs.revolupay.es/` (Route53 ya apunta a CloudFront).
2. **Borrar el bucket viejo** `docs.revolupay.es`: vaciar + delete bucket.
3. Si publicaste la URL HTTP del bucket viejo a algún partner, comunicar la nueva.

---

## 10. Cleanup del repo tras setup

Una vez todo funciona y este documento ya no es necesario, borrar `_setup-aws.md` del repo (los pasos quedan en el git history). El GHA tiene `_setup-aws.md` en el exclude, así que **nunca se publica en S3**.
