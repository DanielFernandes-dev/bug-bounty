# sqlite-carver-pro

## Objetivo
Carvar registros ativos, de freeblock e de espaço não alocado (inclusive de imagem de disco
sem header SQLite), com score de confiança.

## Ambiente
- Repositório original do `SKILL.md` (`15711224433/SQLite-Carver-Pro`) → **404**.
  Substituto usado: `Dam-FOR3K/SQLite_analyzer`
  (`pip install ./tools/src/SQLite_analyzer`; módulo `sqlite_carver`).
- Bases: `work/corpus/delete-lab.db`, `work/corpus/disk.img`

## Comandos
```bash
python -m sqlite_carver info delete-lab.db
python -m sqlite_carver carve delete-lab.db
python -m sqlite_carver carve-raw disk.img
```

## Resultados
`info` (funciona bem): hash SHA-256/MD5, page size 4096, 2 páginas, freelist 0, encoding
UTF-8, auto-vacuum NONE, "Active Payloads 269 B", "Unallocated Space 7.727 B".

`carve` (após workaround do `-wal`):
```
Total Evidence Records: 5 | active: 3 | freeblock: 1 | unallocated: 1
offset 0xf74  freeblock   0.85  customers  -> name: Bruno Costa,
  email: bruno.costa@corp.com, ssn: 987-65-4321, api_key: sk_live_4c1e9b2d7a5f8e3c
```

`carve-raw disk.img`:
```
0xf81  active       0.75  -> CREATE TABLE customers (id INTEGER PRIMARY KEY, name ...
0xf28  active       1.00  -> id: 3, Carla Dias, ...
0xf74  freeblock    0.85  -> Bruno Costa, ...
0xfba  active       1.00  -> id: 1, Ana Silva, ...
0xc    unallocated  0.68  -> b'\x00\x00...'
```

## Bug encontrado (bloqueante)
`carve` quebra quando o banco **não tem** arquivo `-wal` ao lado:
```
File ".../sqlite_carver/cli.py", line 404, in cmd_carve
    if wal_path.exists():
AttributeError: 'NoneType' object has no attribute 'exists'
```
A variável `wal_path` é `None` (linha 324) quando o `-wal` não existe. **Workaround**:
`touch banco.db-wal` antes de rodar. Vale reportar upstream (fix: `if wal_path and
wal_path.exists():`).

## Achados / armadilhas
1. **Bug do `-wal`**: sem o arquivo, `carve` morre com `AttributeError`.
2. `info`, `carve-raw`, `search`, `wal-diff`, `decode-blob` **não** sofrem do bug (o
   `carve-raw` não abre o DB).
3. `carve` é interativo/formatado (rich); a saída de tabela é larga e truncada em terminais
   estreitos — usar `COLUMNS=200`.
4. `--key` serve para bancos SQLCipher; `--gui` abre interface gráfica.
5. `carve-raw` funciona sem header SQLite (imagem de disco/RAM) — melhor cenário do lab.
6. Scores de confiança (0.68–1.00) ajudam a priorizar; 0.68 costuma ser lixo binário.

## Veredito
Skill validada com **duas correções**: URL do projeto (404) e workaround do `-wal`. É a
ferramenta mais completa em cenários sem header (`carve-raw`).
