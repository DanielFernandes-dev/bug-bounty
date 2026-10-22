# jwt-audit

## Objetivo
Auditoria estática do JWT do dv-bookshop: algoritmo, chave e claims ausentes/sensíveis.

## Ambiente
- Repositório: `simemat/jwt-audit` (Rust) — **único** existente; `waylan/jwt-audit` do
  `SKILL.md` dá 404 e o crate `jwt-audit` não existe no crates.io.
- Clonado em `labs/tools/src/jwt-audit`, binário em `target/release/jwt-audit`.
- Toolchain: rustup (minimal) em `~/.cargo` + `~/.rustup`.
- Limitação de entrada: o CLI só aceita o token como **argumento posicional** (`jwt-audit <TOKEN>`).

## Patch aplicado (o upstream não compila)
Erros originais: `payload_claims` inexistente, `qq&payload_raw_json` (typo `&&`), `HashMap`
sem import, `;` faltando em statements. Correções em `src/main.rs`:
- `use std::collections::HashMap;`
- leitura serializada do payload e `payload_claims` montado a partir do JSON
- operador `?` em `Result` e `;` nos statements

## Comandos
```bash
cargo build --release
./target/release/jwt-audit "$(cat artifacts/dv-jwt.txt)"
```

## Resultados
```
header  : {"alg":"HS256","typ":"JWT"}
payload : {"iat":1792611542,"role":"user","sub":2,"username":"alice_hacker"}
sig len : 43 bytes

[HIGH]   algoritmo simetrico HS256: o segredo de assinatura e compartilhado
         com o servidor e pode ser adivinhado offline
[MEDIUM] claim 'exp' ausente: token nunca expira
[INFO]   claim sensivel presente: role = "user"
```
- HIGH confirma o que o `jwt-tool` provou empiricamente (segredo `secret`).
- MEDIUM (`exp` ausente) é um achado estático válido e reaproveitável.

## Achados / armadilhas
1. **Ferramenta abandonada e quebrada**: sem patch, `cargo build` falha. Se o objetivo é só
   auditar tokens, o `jwt_tool` (`-M at`) cobre o mesmo caso sem manutenção.
2. O binário não lê de stdin nem de arquivo; passar o token na linha de comando expõe o
   token no `ps`/histórico — usar `jwt-audit "$(cat token.txt)"` em script.
3. Saída é texto simples em pt-BR (não há `--json`), o que limita automação.

## Veredito
Skill validada **com correções**: trocar `waylan/jwt-audit` → `simemat/jwt-audit`,
documentar o patch necessário, ou substituir a dependência por `jwt_tool -M at`.
