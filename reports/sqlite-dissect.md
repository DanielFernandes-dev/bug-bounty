# sqlite-dissect

## Objetivo
Parsear o banco e **carvar** registros deletados de freeblocks e espaço não alocado,
comparando um banco "normal" com um criado com `secure_delete=ON`.

## Ambiente
- `sqlite_dissect` 1.0.0 no venv `labs/.venv` (binário `sqlite_dissect`)
- Bases: `work/corpus/delete-lab.db` e `delete-lab-secure.db`
- Saída: `artifacts/sd/<base>/<base>-customers.csv`

## Comandos
```bash
sqlite_dissect delete-lab.db -c -d artifacts/sd/delete-lab -e csv
sqlite_dissect delete-lab-secure.db -c -d artifacts/sd/delete-lab-secure -e csv
```

## Resultados
`delete-lab.db` — 19 registros extraídos (2 alocados + 17 carved de Freeblock):
```
Location=Freeblock, Carved, Row ID=Unknown,
  name="Bruno Costa", email="bruno.costa@corp.com",
  ssn="987-65-4321", api_key="sk_live_4c1e9b2d7a5f8e3c"
```
`delete-lab-secure.db` — apenas os 2 registros alocados; **nenhum** registro carved.
O CSV inclui metadados forenses por linha: `Page Number`, `Location`
(Allocated Space / Freeblock / Unallocated), `Operation` (Added/Carved),
`File Offset`, `Row ID`.

## Achados / armadilhas
1. **Ordem dos argumentos**: o caminho do banco é **posicional** e vem primeiro
   (`sqlite_dissect <db> -c ...`); inverter gera erro de argparse.
2. **`-c` = `--carve`** (habilita o carving). `-f` é `--carve-freelists` e **exige** `-c`
   junto — usar `-f` sozinho não faz nada.
3. `-e csv` exporta um CSV por tabela; sem `-e` o output é só console.
4. Fragmentos parciais aparecem como `"3c"`, `b'5f8e3c'` etc. (células truncadas) —
   normal em freeblock; o `name`/`email` completos indicam célula intacta.
5. O modo carving lê **freeblocks e espaço não alocado**, por isso encontra o que o
   `undark` não encontra.
6. Em banco com `secure_delete=ON` o carving não retorna nada — bom teste de eficácia.

## Veredito
Skill validada. É a ferramenta com melhor resultado de carving entre as testadas (recuperou
o registro completo com PII + chave de API). Corrigir no `SKILL.md` a ordem dos argumentos e
a diferença `-c`/`-f`.
