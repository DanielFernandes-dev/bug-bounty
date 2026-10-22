# undark

## Objetivo
Recuperar registros deletados de um SQLite usando dois modos distintos: `--freespace`
(células no freeblock) e `--removed-only` (chaves `-1`).

## Ambiente
- `undark` (C, build via `Makefile`, `labs/tools/bin/undark`)
- Bases: `work/corpus/delete-lab.db`, `/tmp/opencode/sd-test/off.db` (Bruno deletado com
  `secure_delete=OFF`), `work/corpus/juiceshop.sqlite`

## Comandos
```bash
undark -i db.sqlite                       # modo padrão
undark -i db.sqlite --freespace           # varre freeblocks
undark -i db.sqlite --removed-only        # só linhas com key = -1
undark -i db.sqlite --fine-search         # varre byte a byte
```

## Resultados
```
$ undark -i delete-lab.db --freespace
-1,NULL,"Ana Silva","ana.silva@corp.com","123-45-6789","sk_live_9f2b7c4e1a8d3f6b"

$ undark -i delete-lab.db --removed-only
(nada)

$ undark -i delete-lab.db
-1,x'0000000000000000000000000000',".",x'0000000000000000000000000000',NULL,NULL,...

$ undark -i juiceshop.sqlite --freespace
-1,".etabl",151590625731913736517006528461546113137493762167783735514099286965...,"Walle","ts.CREA","TE TABL",x69
-1,"index","sqlite_autoindex_SecurityAnswers_1","SecurityAnswers",x27,NULL
```
- No `off.db` (Bruno deletado) o `--freespace` recuperou uma **cópia antiga do registro da
  Ana**, mas **não** o registro do Bruno.
- O modo padrão emite lixo de baixo valor (`x'0000...'`, `"."`).

## Achados / armadilhas
1. **O modo padrão é praticamente inútil** para triagem: gera linhas com `-1` e colunas
   binárias zeradas. Use `--freespace` e `--removed-only` explicitamente.
2. **Limite de alcance**: `--freespace` varre apenas células dentro de páginas alocadas
   (freeblocks). Registros em **espaço não alocado** (fim do arquivo) não são vistos — foi o
   caso do Bruno. Para isso usar `sqlite_dissect -c` ou `sqlite_carver carve`.
3. `--removed-only` só retorna linhas cuja chave primária sobrou como `-1`; depende do
   layout da célula (funciona melhor em tabelas `INTEGER PRIMARY KEY`).
4. `--fine-search` é muito mais lento e gera mais falsos positivos; usar só quando o
   parse normal falha (header corrompido).
5. Saída é CSV sem header por padrão; `--no-blobs` e `--blob-size-limit` evitam despejar
   blobs gigantes no terminal.
6. `undark` não lê WAL (precisa de `--wal` separado? não — não há suporte); para WAL usar
   `sqlite_dissect -w` ou `sqlite_carver wal-diff`.

## Veredito
Skill validada. Corrigir a expectativa: `undark` recupera freeblocks, **não** unallocated
space; e enfatizar `--freespace` em vez do modo padrão.
