# gitleaks

## Objetivo
Detectar segredos hardcoded nos labs (código + histórico git), com regras padrão e regras
customizadas para o stack dos labs (Flask/JWT/CTF).

## Ambiente
- `gitleaks` 8.30.1 (`labs/tools/bin/gitleaks`)
- Alvos: `labs/dv-bookshop`, `labs/vulnshop`, `labs/websec-sqli`, `labs/artifacts/juiceshop-fs`
- Ruleset customizado: `labs/artifacts/.gitleaks.toml`

## Comandos
```bash
gitleaks dir dv-bookshop --config .gitleaks.toml --report-format json --report-path out.json
gitleaks git dv-bookshop --report-format json --report-path out-git.json
gitleaks dir vulnshop --config .gitleaks.toml
```

## Resultados
| Alvo | Regras padrão | Com ruleset customizado |
|---|---|---|
| dv-bookshop | 6 (`curl-auth-header`) | 13 |
| vulnshop | 0 | 3 |
| websec-sqli | 0 | 0 |
| juiceshop-fs | 14 (`generic-api-key` 9, `private-key` 3, `jwt` 2) | — |

Achados relevantes com ruleset próprio:
- `dv-bookshop/app.py:34` — `app.secret_key = 'bookshop_secret_key_2025_khitminnyo'`
- `dv-bookshop/app.py:64` — `STATIC_RESET_SECRET = 'khitminnyo2025'`
- `dv-bookshop/app.py:67` — `JWT_SECRET = 'secret'`
- `vulnshop/app.py:6` — `app.secret_key = 'supersecret123'`
- `vulnshop/app.py:88` — `INSERT INTO users` com senha em texto plano

## Achados / armadilhas
1. **As regras padrão não pegam `app.secret_key` nem `JWT_SECRET`** — o sinal em apps Flask
   só aparece com ruleset customizado. Sem isso, `vulnshop` dava "no leaks found".
2. **`secretGroup` conta grupos de captura**: com um grupo extra no meio (alternância),
   o índice aponta para o grupo errado e a regra silenciosamente não reporta. Usar
   `(?:...)` para alternância não capturante.
3. `--redact` mascara o segredo no relatório; use `--report-format json` para triagem.
4. `--no-banner`/`--log-level` mantêm a saída limpa para CI.
5. Varredura de `__pycache__/*.pyc` gera duplicatas do mesmo achado.

## Veredito
Skill validada, com a correção importante: a skill precisa de um ruleset customizado
(Flask/JWT/CTF) e do aviso sobre `secretGroup`.
