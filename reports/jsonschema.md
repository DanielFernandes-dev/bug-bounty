# jsonschema

## Objetivo
Validar o mesmo conjunto de payloads do `ajv` usando a biblioteca Python `jsonschema`, com
`best_match` para destacar o erro mais relevante de cada payload.

## Ambiente
- `jsonschema` 4.26.0 no venv `labs/.venv`
- Schema: `labs/artifacts/api/order.schema.json` (Draft-07)

## Comandos
```python
from jsonschema import Draft7Validator
v = Draft7Validator(schema)
Draft7Validator.check_schema(schema)
errs = sorted(v.iter_errors(data), key=lambda e: list(e.path))
best = jsonschema.exceptions.best_match(errs)
```

## Resultados
```
valid                    VALID
invalid-missing-cid      INVALID (1) -> required @ []: 'cid' is a required property
invalid-quantity         INVALID (1) -> minimum @ ['orderLines',0,'quantity']: 0 < 1
invalid-type             INVALID (1) -> type @ ['orderLines']: 'nao-e-array' is not of type 'array'
invalid-extra            INVALID (1) -> additionalProperties @ []: 'admin' was unexpected
```

## Achados
- `Draft7Validator.check_schema()` valida o próprio schema antes de usar — pega erro de
  schema (ex.: `type` inválido) que o `ajv` só reportaria na compilação.
- `best_match()` ordena por relevância (propriedade faltante > tipo), o que ajuda em payloads
  com muitos erros encadeados.
- `jsonschema 4.x` removeu `RefResolver`; para schemas com `$ref` externo usar
  `referencing.Registry`.
- `jsonschema.__version__` está deprecado (usar `importlib.metadata`).

## Veredito
Skill validada; atualizar o `SKILL.md` para não usar `jsonschema.__version__` e citar
`referencing` para `$ref`.
