# sqlcipher

## Objetivo
Criar, abrir e auditar um banco SQLite cifrado com SQLCipher 4.x, verificando KDF, HMAC e
comportamento com chave errada.

## Ambiente
- SQLCipher **4.19.0 community** (build local: `--with-tempstore=yes`,
  `CXXFLAGS="-DSQLITE_HAS_CODEC ..."`, `LDFLAGS="-lcrypto"`), binário `labs/tools/bin/sqlcipher`
- Base: `labs/work/enc.db` (tabela `t`, linha `1|dado cifrado`)

## Comandos
```bash
sqlcipher enc.db "PRAGMA key='testkey123';"
sqlcipher enc.db "PRAGMA key='testkey123'; SELECT * FROM t;"
sqlcipher enc.db "PRAGMA key='errada'; SELECT * FROM t;"
sqlcipher :memory: "PRAGMA cipher_default_kdf_iter;"
```

## Resultados
```
4.19.0 community

cipher_default_kdf_iter            256000
cipher_default_page_size           4096
cipher_default_use_hmac            1
cipher_default_kdf_algorithm       PBKDF2_HMAC_SHA512
cipher_default_hmac_algorithm      HMAC_SHA512
cipher_default_plaintext_header_size 0

chave errada:
ERROR CORE sqlcipher_page_cipher: hmac check failed for pgno=1
ERROR CORE sqlite3Codec: error decrypting page 1 data: 1
```
Header (32 bytes) totalmente aleatório — não há assinatura `SQLite format 3`.

## Achados / armadilhas
1. **`PRAGMA key` tem que ser o primeiro statement** da conexão; sem ele →
   `file is not a database (26)`.
2. **`PRAGMA use_hmac` / `kdf_algorithm` / `hmac_algorithm` são setter-only**: retornam `ok`
   e não informam o valor atual. Para ler os defaults usar `cipher_default_*` (funciona em
   `:memory:`).
3. **Vários statements no mesmo argumento**: o CLI só imprime o resultado do ÚLTIMO. Rodar
   um PRAGMA por invocação (ou via stdin, um por linha).
4. `cipher_version` retorna `4.19.0 community`; ausência/valor vazio indica build **sem**
   codec.
5. Com `PRAGMA plaintext_header_size=32` é possível manter o header legível (usado por apps
   iOS); o default 0 cifra tudo.
6. `PRAGMA rekey='nova'` troca a chave; `cipher_migrate` converte entre versões (3↔4).

## Veredito
Skill validada. Ajustar: `cipher_default_*` para leitura de parâmetros e o aviso de que
pragmas de configuração não são consultáveis.
