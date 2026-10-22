# sqlite3-multiple-ciphers

## Objetivo
Abrir/criar bancos cifrados com cifras diferentes (AES, ChaCha20, RC4) usando o shell
`sqlite3mc`, e testar compatibilidade com bancos SQLCipher 3/4.

## Ambiente
- SQLite3MultipleCiphers (shell SQLite **3.54.0**), build local:
  `autoreconf -i && ./configure && make` → `sqlite3shell` copiado como `labs/tools/bin/sqlite3mc`
- Bancos: `work/enc-aes.db` (AES-256-CBC), `work/enc.db` (SQLCipher 4.19)

## Comandos
```bash
# criar
printf "PRAGMA cipher='aes256cbc';\nPRAGMA key='testkey123';\nCREATE TABLE t(x TEXT);\nINSERT INTO t VALUES('segredo-aes');\n" | sqlite3mc enc-aes.db

# abrir
printf "PRAGMA cipher='aes256cbc';\nPRAGMA key='testkey123';\nSELECT * FROM t;\n" | sqlite3mc enc-aes.db

# ler um banco SQLCipher 4.x
printf "PRAGMA cipher='sqlcipher';\nPRAGMA legacy=4;\nPRAGMA key='testkey123';\nSELECT * FROM t;\n" | sqlite3mc enc.db
```

## Resultados
```
criar/reabrir aes256cbc  -> segredo-aes
chave errada             -> file is not a database (26)
sem PRAGMA cipher        -> file is not a database (26)   (default chacha20 não decifra AES)
header enc-aes.db        -> fb d4 8c 09 53 4e d5 83 ...   (cifrado, sem magic)
PRAGMA cipher='aes256'   -> Cipher 'aes256' unknown.

SQLCipher 4.19 via sqlite3mc (cipher='sqlcipher' + legacy=4):
  t
  1|dado cifrado
```

## Achados / armadilhas
1. **`--key`/`-key` não existe**: no shell é sempre `PRAGMA key='...'`, e a ordem importa —
   `PRAGMA cipher` **antes** do `PRAGMA key` (o key usa a cifra ativa no momento).
2. **Não é possível passar múltiplos statements como argumento**: o shell executa tudo mas
   imprime só o último resultado, e `PRAGMA key` responde `ok`. Use **stdin, um statement por
   linha** para enxergar os dados.
3. **SQLCipher 4.x é suportado com `PRAGMA legacy=4`** (não com ajuste manual de
   `kdf_iter`/`hmac_algorithm`). Sem `legacy`, o banco SQLCipher 4 não abre.
4. Cifras aceitas: `aes128cbc`, `aes256cbc`, `chacha20` (default), `rc4`, `sqlcipher`;
   `aes256`/`aes` dão `Cipher 'X' unknown.`
5. `PRAGMA cipher_list`/`cipher_version` não retornam nada neste build (não use para
   detectar suporte).
6. Header totalmente cifrado em todas as cifras testadas.

## Veredito
Skill validada **com correção importante**: usar `PRAGMA legacy=4` para SQLCipher 4,
ordem `cipher` → `key`, e stdin em vez de statements no argumento.
