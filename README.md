# Bug Bounty

Workspace de bug bounty: notas, evidências e automações de teste.

## Skills

As skills de segurança vivem no repositório próprio [`DanielFernandes-dev/skills`](https://github.com/DanielFernandes-dev/skills) e entram aqui como **submódulo**:

```bash
git clone --recurse-submodules <url>
# ou, em um clone existente
git submodule update --init --recursive
```

O caminho `skills/` é registrado no `opencode.json` deste projeto, então todas as skills são carregadas automaticamente ao abrir o opencode nesta pasta.

| Área | Skills |
|---|---|
| API / JSON | `jwt-tool`, `jwt-audit`, `schemathesis`, `restler`, `owasp-zap`, `ajv`, `jsonschema`, `spectral`, `gitleaks`, `trufflehog` |
| SQLite | `sqlcipher`, `sqlcipher-audit`, `sqlite3-multiple-ciphers`, `fqlite`, `sqlite-forensic`, `sqlite-carver-pro`, `sqlite-dissect`, `bring2lite`, `undark`, `sqlite-delete` |

## Aviso

Testes de segurança só devem ser executados em alvos com autorização explícita.
