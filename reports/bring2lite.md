# bring2lite

## Objetivo
Reconstruir o "mini filesystem" de um banco SQLite e extrair os dados por página, testando a
CLI em modo headless.

## Ambiente
- `bring2lite/bring2lite` instalado via `pip install ./tools/src/bring2lite`
  (traz PyQt5, tqdm, sqlparse, colorama); requer **tkinter** (`python3-tk`)
- Execução headless a partir de `tools/src/bring2lite/bring2lite` com `PYTHONPATH=.`
- Base: `work/corpus/delete-lab.db`

## Problemas e workaround
1. **Entry point quebrado**: `bring2lite` → `ModuleNotFoundError: No module named 'classes'`
   (`from classes.gui import GUI` sem package relativo). Workaround:
   ```bash
   cd labs/tools/src/bring2lite/bring2lite
   PYTHONPATH=. python main.py --filename <db> --out <dir> --format JSON
   ```
2. Sem `tkinter` a inicialização aborta; instalar `python3-tk`.
3. Vários `SyntaxWarning` (`is` comparando strings) no código — ruído, não impede o uso.

## Comandos
```bash
PYTHONPATH=. python main.py \
  --filename work/corpus/delete-lab.db \
  --out /tmp/opencode/b2l-out --format JSON
```

## Resultados
```
Processing main files
sha-256: .../delete-lab.db/schemas/sql-schema.log => e64b6f2a...
sha-256: .../delete-lab.db/regular-page-parsing/2-page.log => 2037907c...
```
Arquivos gerados:
```
delete-lab.db/schemas/sql-schema.log
delete-lab.db/regular-page-parsing/2-page.log
```
Conteúdo:
```
# sql-schema.log
INT, TEXT, TEXT, TEXT, TEXT,
++++++++++++++++++++++++++++

# 2-page.log
INT,TEXT,TEXT,TEXT,TEXT,
NULL,Ana Silva,ana.silva@corp.com,123-45-6789,sk_live_9f2b7c4e1a8d3f6b,
NULL,Carla Dias,carla.dias@corp.com,456-78-9012,sk_live_8b3d5f1a2c6e9d4f,
```

## Achados / armadilhas
1. **`--format JSON` é ignorado no modo headless**: a saída é sempre um log CSV-like por
   página (`regular-page-parsing/N-page.log`) + log de schema. Não é um export JSON.
2. **Não recupera registros deletados**: apenas as células vivas da página (o Bruno, em
   freeblock, não aparece).
3. Cada entrada tem hash SHA-256 — útil para cadeia de custódia, inútil para triagem.
4. `--gui` exige display (Xvfb/VNC); o modo CLI não abre janela, mas depende de tkinter.
5. O "mini filesystem" gerado é o conjunto de arquivos em `--out/<db>/...`, não uma imagem
   montável.

## Veredito
Skill **parcialmente validada**: funciona apenas com workaround de `PYTHONPATH` e entrega
logs por página (não JSON); não recupera deleted. Ajustar o `SKILL.md` para refletir isso e
indicar `sqlite_dissect`/`undark` para deleted.
