# ajv

## Objetivo
Validar payloads JSON do endpoint `POST /b2b/v2/orders` (Juice Shop) contra um JSON Schema
Draft-07, com relatório de todos os erros.

## Ambiente
- `ajv-cli` (npm global) — binário `ajv`
- Schema: `labs/artifacts/api/order.schema.json`
- Payloads: `valid.json`, `invalid-missing-cid.json`, `invalid-quantity.json`,
  `invalid-type.json`, `invalid-extra.json`

## Comandos
```bash
ajv validate -s order.schema.json -d valid.json --all-errors
ajv validate -s order.schema.json -d invalid-quantity.json --all-errors
```

## Resultados
| Payload | Veredito | Erro |
|---|---|---|
| valid | valid | — |
| invalid-missing-cid | invalid | `required` em `''` |
| invalid-quantity | invalid | `minimum` em `/orderLines/0/quantity` |
| invalid-type | invalid | `type` em `/orderLines` |
| invalid-extra | invalid | `additionalProperties` em `''` |

5/5 vereditos corretos, com `instancePath`, `schemaPath` e `keyword`.

## Achados
- O `ajv-cli` valida **offline**: útil porque a API do Juice Shop bloqueia o corpo com 401
  antes de qualquer validação de schema (a autenticação precede o parsing).
- `--all-errors` é essencial; sem ele o AJV para no primeiro erro.
- `additionalProperties: false` pega mass assignment (`"admin": true`) — bom teste de
  contrato para APIs que aceitam objetos livres.

## Limitações
- Draft-07 por padrão; para 2019-09/2020-12 é preciso `ajv-cli` com `--spec=draft2020`.
- Datas/formats só são checados com `ajv-formats` (não vem no `ajv-cli` por padrão).

## Veredito
Skill validada sem correções.
