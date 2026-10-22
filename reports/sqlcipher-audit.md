# sqlcipher-audit

## Objetivo
Auditar a configuração criptográfica de um banco SQLCipher: iterações de KDF, tamanho de
página, HMAC e layout do header.

## Ambiente
- SQLCipher 4.19.0 community (`labs/tools/bin/sqlcipher`)
- Bancos: `labs/work/enc.db` (criado no lab), `labs/artifacts/juiceshop-fs/encryptionkeys/`

## Comandos
```bash
# defaults do build
for p in cipher_default_kdf_iter cipher_default_page_size \
         cipher_default_use_hmac cipher_default_kdf_algorithm \
         cipher_default_hmac_algorithm cipher_default_plaintext_header_size; do
  sqlcipher :memory: "PRAGMA $p;"
done

# parâmetros do banco (um por invocação)
sqlcipher enc.db "PRAGMA key='testkey123'; PRAGMA kdf_iter;"
sqlcipher enc.db "PRAGMA key='testkey123'; PRAGMA page_size;"
sqlcipher enc.db "PRAGMA key='testkey123'; PRAGMA cipher_plaintext_header_size;"

# layout do header
od -An -tx1 -N32 enc.db
```

## Resultados
| Parâmetro | Valor | Avaliação |
|---|---|---|
| `kdf_iter` | 256000 | OK (default SQLCipher 4, PBKDF2-HMAC-SHA512) |
| `page_size` | 4096 | OK |
| `use_hmac` | 1 | OK (HMAC-SHA512 por página) |
| `kdf_algorithm` | PBKDF2_HMAC_SHA512 | OK |
| `hmac_algorithm` | HMAC_SHA512 | OK |
| `plaintext_header_size` | 0 | OK (header inteiro cifrado) |

Header do `enc.db`: `61 62 d5 d5 e0 9d 7b 39 80 45 f6 b1 72 ec 7a 85 ...`
→ sem magic `SQLite format 3`, sem vazamento de page size/versão.
Chave errada: `hmac check failed for pgno=1` (HMAC validado **antes** de decifrar).

## Achados / armadilhas
1. `PRAGMA kdf_iter` etc. só respondem **depois** do `PRAGMA key` (antes disso o banco não
   está aberto).
2. `PRAGMA use_hmac` não é consultável (setter-only); o default vem de
   `cipher_default_use_hmac`.
3. **`kdf_iter` baixo (ex.: 1, 4000, 64000) é o achado clássico** de app legado/mobile —
   permite brute force offline viável.
4. Header legível (`SQLite format 3`) com `plaintext_header_size` > 0 vaza page size e
   versão, mas não a chave.
5. Sem `cipher_version` no binário → build sem `SQLITE_HAS_CODEC` (não é SQLCipher).

## Veredito
Skill validada. Complementar o `SKILL.md` com o uso de `cipher_default_*` (leitura) e com a
tabela de referência acima.
