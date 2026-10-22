# spectral

## Objetivo
Lint da especificação OpenAPI do Juice Shop (`/b2b/v2`) com ruleset próprio, cobrindo HTTPS,
autenticação em operações de escrita e dados reais em exemplos.

## Ambiente
- `@stoplight/spectral-cli` (npm global) — binário `spectral`
- Spec: `labs/artifacts/juiceshop-swagger.yml` (OpenAPI 3.0.0, 1 path: `POST /orders`)
- Ruleset: `labs/artifacts/.spectral.yaml`

## Comandos
```bash
spectral lint juiceshop-swagger.yml --ruleset .spectral.yaml --format stylish
```

## Resultados
```
4:10  error  no-http-servers        Servidores devem usar HTTPS           servers[0].url
20:10 error  require-auth-on-write  Operacoes de escrita exigem security  paths./orders.post

✖ 2 problems (2 errors, 0 warnings, 0 infos, 0 hints)   (exit 1)
```

- 2 erros reais: `servers[0].url: /b2b/v2` sem TLS e `POST /orders` sem `security`.
- Com a regra de PII corrigida (`pattern` + `notMatch`), **nenhum** falso positivo.
- Antes da correção (com `schema` + `not: pattern`) a mesma regra gerava 6 warnings
  falsos — os exemplos que não eram e-mail eram marcados.

## Achados / armadilhas da ferramenta
1. **`--ruleset` não é obrigatório**: o Spectral descobre `.spectral.yaml` no diretório
   automaticamente. Rodar sem `--ruleset` no mesmo diretório usa o ruleset local (não o
   `spectral:oas` puro) — pode confundir a comparação de resultados.
2. **`schema` + `not: {pattern}` gera falso positivo**: a regra original marcava qualquer
   string. A forma correta é a função `pattern` com `notMatch`:
   ```yaml
   then:
     function: pattern
     functionOptions:
       notMatch: "@(gmail|hotmail|yahoo)\\.com$"
   ```
3. **JSONPath**: `given: "$..example"` é o seletor que funciona para todos os exemplos
   (inclusive dentro de `components`); `$.paths..example` não cobre `components`.
4. Saída "No results with a severity of 'error' found!" com exit 0 quando só há warnings.
5. Os erros do Spectral **corroboram** o achado do Schemathesis (401 não documentado).

## Veredito
Skill validada. Correções necessárias no `SKILL.md`: usar `pattern` + `notMatch`,
`given: "$..example"`, e avisar sobre a descoberta automática do ruleset.
