# RevoluPAY · API Documentation

Documentación pública de las APIs de RevoluPAY, renderizada con [Redoc](https://redocly.com/redoc/) y publicada en https://docs.revolupay.es.

## Estructura

```
api-docs/
├── agent/
│   └── api-agent-doc.yaml         ← OpenAPI 3.x spec del API de agentes
│   └── index.html                 ← (generado por CI) Redoc bundle
├── index.html                     ← landing page (lista de APIs)
└── .github/workflows/deploy.yml   ← build & deploy a S3 + CloudFront
```

## Flujo de cambios

1. Editar el `.yaml` correspondiente (`agent/api-agent-doc.yaml`, etc.).
2. Validar localmente:
   ```bash
   npx @redocly/cli lint agent/api-agent-doc.yaml
   npx @redocly/cli preview-docs agent/api-agent-doc.yaml   # vista local
   ```
3. Commit + push a `main`. El GitHub Action:
   - bundlea cada `.yaml` con `@redocly/cli build-docs` → `index.html` self-contained.
   - sube el resultado a `s3://revolupay-api-docs-prod/`.
   - invalida la cache de CloudFront.
4. Cambios visibles en `https://docs.revolupay.es/` en 1-2 min.

## Versionado

Para mantener spec histórica accesible (clientes integrados con v1 mientras se desarrolla v2):

```bash
git tag -a v1.0 -m "Initial public release of agent API"
git push origin v1.0
```

El workflow detecta tags `vX.Y` y publica una copia en `s3://revolupay-api-docs-prod/vX/agent/` accesible en `https://docs.revolupay.es/v1/agent/`. El `/agent/` raíz siempre apunta a la última versión estable.

## Seguridad / DORA

- HTTPS-only (HSTS `max-age=31536000` desde CloudFront).
- Bucket S3 privado, accesible solo vía CloudFront OAC.
- Spec saneada antes de publicar: sin UUIDs reales, sin emails de empleados, sin URLs internas. Convención: nombres ficticios "Candelario Pega Fuegos" para datos personales en ejemplos.
- Audit trail: cada push a `main` queda en `git log`; CloudFront access logs en `s3://revolupay-logs-prod/cloudfront/docs/`.
- Cumple DORA Art. 28 (documentación de interfaces para terceros) y Art. 9 (cifrado en tránsito).

## Riesgos / Notas operativas

- El renderer es **Redoc**, no Swagger UI. No hay "Try it out" para evitar que un partner pegue un API key en una página pública.
- Si se añade Swagger UI en algún punto (`docs.revolupay.es/try/`), debe ir capado SOLO a sandbox y nunca apuntar a prod.
- Los ejemplos del spec usan UUIDs determinísticos sintéticos (`11111111-1111-4111-9111-111111111111`, etc.) — si ves uno con apariencia "real" en un PR, rechazar.
