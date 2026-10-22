# owasp-zap

## Objetivo
Scan passivo/baseline do OWASP Juice Shop em execução (`http://127.0.0.1:3000`), com relatório
JSON para triagem.

## Ambiente
- `ghcr.io/zaproxy/zaproxy:stable` (2,44 GB), `docker run --network host`
- Alvo: Juice Shop (container `juiceshop`, porta 3000)
- Script: `zap-baseline.py`

## Comandos
```bash
docker run --rm --network host \
  -v "$PWD/reports/raw:/zap/wrk:rw" -t ghcr.io/zaproxy/zaproxy:stable \
  zap-baseline.py -t http://127.0.0.1:3000 -J zap-baseline.json -m 1 -T 5
```

## Resultados
```
FAIL-NEW: 0  WARN-NEW: 7  PASS: 61
```
Alertas (risco / plugin / ocorrências):

| Risco | Plugin | Alerta | Instâncias |
|---|---|---|---|
| Médio | 10038 | Content Security Policy (CSP) Header Not Set | 5 |
| Médio | 10098 | Cross-Domain Misconfiguration | 5 |
| Baixo | 10063 | Deprecated Feature Policy Header Set | 5 |
| Baixo | 10110 | Dangerous JS Functions | 2 |
| Baixo | 90004 | Cross-Origin-Embedder-Policy Missing/Invalid | 5 |
| Baixo | 90004 | Cross-Origin-Opener-Policy Missing/Invalid | 5 |
| Info | 10049 | Storable and Cacheable Content | 2 |
| Info | 10049 | Storable but Non-Cacheable Content | 5 |
| Info | 10109 | Modern Web Application | 5 |

## Achados / armadilhas
1. **Exit code 2 = warnings** (0 = pass, 1 = fail). Em CI, tratar 2 como "revisar".
2. **Permissão do volume**: o container roda como uid 1000; com `docker run` via root o
   relatório falha com `AccessDeniedException /zap/wrk/...`. Solução: `chmod 777` no dir
   de saída (ou `--user`).
3. `-m 1` (minutos de spider) e `-T 5` (timeout) tornam o baseline rápido para smoke test.
4. O baseline é **passivo**: não encontrou as falhas de lógica do Juice Shop (SQLi, JWT,
   broken access control). Para isso seria preciso `zap-full-scan.py` ou autenticação.
5. O relatório `-J` (legacy) traz `riskcode` numérico; `risk` vem `null` (usar `riskcode`).

## Veredito
Skill validada. Corrigir no `SKILL.md`: permissões do volume e semântica do exit code.
