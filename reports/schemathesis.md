# schemathesis

## Objetivo
Fuzzing property-based do endpoint `POST /b2b/v2/orders` a partir da spec OpenAPI do Juice Shop.

## Ambiente
- `schemathesis` 4.32.3 no venv `labs/.venv`
- Spec: `labs/artifacts/juiceshop-swagger.yml` (server `/b2b/v2`)
- API: `http://127.0.0.1:3000/b2b/v2`

## Comandos
```bash
schemathesis run juiceshop-swagger.yml \
  --url http://127.0.0.1:3000/b2b/v2 \
  --max-examples 15 --workers 1
```

## Resultados
```
API Operations: Selected 1/1, Tested 1
Test Phases: Examples ❌  Coverage ❌  Fuzzing ❌  Stateful n/a
Failures: ST103 Undocumented status code: 1
Warnings: ⚠️ Authentication failed: 1 operation returned only 401/403
Test cases: 3 generated, 1 unique failure
```

- Falha **ST103**: `POST /orders` responde `401` sem que a spec documente esse status
  (nenhum `security` declarado) — o mesmo achado do Spectral (`require-auth-on-write`).
- Warning de autenticação: todas as chamadas retornaram 401/403.

## Achados / armadilhas
1. **Flags mudaram no 4.x**: `--base-url` não existe mais (usar `--url`), `--hypothesis-seed`
   também foi removido. O `SKILL.md` precisa refletir isso.
2. Sem credenciais, o fuzzing real do corpo não acontece — a autenticação barra antes.
   Para exercitar o parser, usar `-H "Authorization: Bearer ..."` com token válido.
3. `st replay <id>` reproduz a falha; o curl de reprodução vem na saída.

## Veredito
Skill validada. Corrigir as flags (`--url`) e documentar a necessidade de `-H`/`--auth` para
que o fuzzing de corpo não seja bloqueado por 401.
