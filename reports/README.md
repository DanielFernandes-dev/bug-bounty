# Relatórios de validação das skills

Validação prática das 20 skills do repositório `skills` contra laboratórios locais.

- Skills de API/JSON: Juice Shop (B2B API), dv-bookshop, VulnShop, websec-sql-injection
- Skills de forense SQLite: bancos reais (Juice Shop, dv-bookshop, VulnShop) e corpus sintético

Cada arquivo segue a mesma estrutura: objetivo, ambiente, comandos executados, resultados,
achados e limitações encontradas na ferramenta.

## Índice

| Skill | Relatório | Veredito |
|---|---|---|
| ajv | [ajv.md](ajv.md) | OK |
| gitleaks | [gitleaks.md](gitleaks.md) | OK (precisa de ruleset próprio) |
| jsonschema | [jsonschema.md](jsonschema.md) | OK |
| jwt-audit | [jwt-audit.md](jwt-audit.md) | OK (upstream quebrado, patch aplicado) |
| jwt-tool | [jwt-tool.md](jwt-tool.md) | OK |
| owasp-zap | [owasp-zap.md](owasp-zap.md) | OK |
| restler | [restler.md](restler.md) | OK (build do fonte; imagem Docker inexistente) |
| schemathesis | [schemathesis.md](schemathesis.md) | OK |
| spectral | [spectral.md](spectral.md) | OK |
| trufflehog | [trufflehog.md](trufflehog.md) | OK (baixo sinal em segredos locais) |
| fqlite | [fqlite.md](fqlite.md) | LIMITADO (CLI quebrado no 5.0) |
| bring2lite | [bring2lite.md](bring2lite.md) | LIMITADO (CLI só gera logs) |
| sqlcipher | [sqlcipher.md](sqlcipher.md) | OK |
| sqlcipher-audit | [sqlcipher-audit.md](sqlcipher-audit.md) | OK |
| sqlite-carver-pro | [sqlite-carver-pro.md](sqlite-carver-pro.md) | OK (bug no `carve` sem `-wal`) |
| sqlite-delete | [sqlite-delete.md](sqlite-delete.md) | OK |
| sqlite-dissect | [sqlite-dissect.md](sqlite-dissect.md) | OK (melhor carving testado) |
| sqlite-forensic | [sqlite-forensic.md](sqlite-forensic.md) | OK (patch `rb` necessário) |
| sqlite3-multiple-ciphers | [sqlite3-multiple-ciphers.md](sqlite3-multiple-ciphers.md) | OK (não lê SQLCipher 4.x) |
| undark | [undark.md](undark.md) | OK |

## Resumo

- 18 skills validadas com sucesso; 2 com limitações documentadas (fqlite, bring2lite).
- 4 bugs upstream encontrados: `simemat/jwt-audit` (não compila), `dutchcoders/forensics-sqlite`
  (abre arquivo em modo texto), `Dam-FOR3K/SQLite_analyzer` (`wal_path` None), `bring2lite`
  (import `classes` + entry point quebrado).
- 1 repositório citado nas skills não existe mais: `SQLite-Carver-Pro` → substituído por
  `Dam-FOR3K/SQLite_analyzer`.
- A imagem Docker do RESTler em `mcr.microsoft.com/restler-fuzzer/restler-fuzzer` não existe
  com a tag `latest`; ver [restler.md](restler.md) para o caminho alternativo.
- Correção relevante em [sqlite3-multiple-ciphers.md](sqlite3-multiple-ciphers.md): o
  `sqlite3mc` **abre** bancos SQLCipher 4.x usando `PRAGMA cipher='sqlcipher'` +
  `PRAGMA legacy=4` (a conclusão anterior de incompatibilidade estava errada).
- Achados reais nos labs: stack trace + path interno no 500 do Juice Shop (RESTler), JWT
  HS256 com segredo `secret` (jwt-tool/jwt-audit), 13 segredos no dv-bookshop (gitleaks),
  PII + `sk_live_*` recuperados de freeblock/unallocated (sqlite_dissect/sqlite_carver) e
  WAL com senhas em texto plano (forensics-sqlite).
