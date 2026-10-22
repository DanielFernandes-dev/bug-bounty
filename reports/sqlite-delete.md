# sqlite-delete

## Objetivo
Verificar se `DELETE` em SQLite realmente apaga os dados no disco, comparando
`secure_delete=OFF` com `secure_delete=ON` + `wal_checkpoint(TRUNCATE)` + `VACUUM`.

## Ambiente
- `sqlite3mc` 3.54.0 (comportamento idêntico ao SQLite upstream)
- Base: `work/corpus/delete-lab.db` (tabela `customers`, registro "Bruno Costa")
- Cópias em `/tmp/opencode/sd-test/`

## Comandos
```bash
# cenário 1 — default (secure_delete=OFF)
sqlite3mc off.db "PRAGMA secure_delete=OFF;
                  DELETE FROM customers WHERE name='Bruno Costa';
                  PRAGMA wal_checkpoint(TRUNCATE);"

# cenário 2 — apagamento seguro
sqlite3mc on.db "PRAGMA secure_delete=ON;
                 DELETE FROM customers WHERE name='Bruno Costa';
                 PRAGMA wal_checkpoint(TRUNCATE);
                 VACUUM;"

strings off.db | grep -a "Bruno\|4c1e9b2d"
strings on.db  | grep -a "Bruno\|4c1e9b2d"
```

## Resultados
```
# secure_delete=OFF
J#5#=Bruno Costabruno.costa@corp.com987-65-4321sk_live_4c1e9b2d7a5f8e3cD

# secure_delete=ON + checkpoint + VACUUM
(nenhum resultado)
```
Arquivos com o mesmo tamanho (8.192 B) nos dois cenários — **o tamanho não indica** se os
dados foram zerados.

## Achados / armadilhas
1. `secure_delete` **não é default confiável**: depende do build
   (`SQLITE_SECURE_DELETE`) e da PRAGMA; sem ela, o registro continua legível via `strings`,
   `undark`, `sqlite_dissect` etc.
2. `secure_delete=ON` zera apenas o espaço liberado **dentro** da página; para eliminar
   páginas inteiras livres é necessário `VACUUM` (ou `PRAGMA incremental_vacuum` com
   auto_vacuum).
3. Em modo WAL, o conteúdo antigo também fica no `-wal` até o checkpoint — por isso
   `wal_checkpoint(TRUNCATE)` faz parte do procedimento.
4. `VACUUM` reescreve o arquivo inteiro (e pode falhar dentro de transação).
5. Para SSD/COW (APFS, btrfs, ZFS) nem o `secure_delete` garante apagamento físico.

## Veredito
Skill validada. Reforçar no `SKILL.md`: `secure_delete` sozinho não basta; o par
`wal_checkpoint(TRUNCATE)` + `VACUUM` é o mínimo, e o teste de verificação é `strings`/carver.
