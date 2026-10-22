# trufflehog

## Objetivo
Caçar credenciais vazadas nos labs e verificar quais são válidas (`--only-verified`).

## Ambiente
- `trufflehog` 3.99.3 (`labs/tools/bin/trufflehog`)
- Alvo: `labs/dv-bookshop` (filesystem) e artefatos do Juice Shop

## Comandos
```bash
trufflehog filesystem dv-bookshop --only-verified --json --no-update
trufflehog filesystem dv-bookshop --json --no-update
```

## Resultados
- `--only-verified`: **0** achados (esperado; os segredos dos labs não são de provedores
  reais e não há como verificar online).
- Sem filtro: **1** detecção (`Postgres`, `Verified=false`).
- Varredura da árvore inteira de `labs/` (com `.venv`, `node_modules`, FS do Juice Shop)
  **não terminou em 5 min** — precisa de escopo.

## Achados / armadilhas
1. **TruffleHog prioriza credenciais de provedores** (AWS/GCP/GitHub/DB). Segredos de
   aplicação (`app.secret_key`, `JWT_SECRET`) **não** são detectados — para isso usar o
   `gitleaks` com ruleset próprio.
2. `--only-verified` é ótimo para triagem (zero ruído), mas em labs locais sempre dá 0;
   rodar as duas variantes.
3. A varredura é lenta em árvores grandes: limitar o path (`--exclude-paths`, diretórios
   específicos) e usar `timeout`.
4. `--no-update` evita checagem de versão em ambiente offline.

## Veredito
Skill validada. O `SKILL.md` deve deixar explícito que TruffleHog é para credenciais de
provedor (com verificação) e que segredos de aplicação são domínio do Gitleaks.
