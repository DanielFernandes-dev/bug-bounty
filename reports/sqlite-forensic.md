# sqlite-forensic

## Objetivo
Analisar diretamente o arquivo **WAL** (write-ahead log): header, frames, salts e checksums,
para recuperar registros ainda não checkpointados.

## Ambiente
- Repositório usado: `dutchcoders/forensics-sqlite` (o `sandersonforensics/sqlite-forensic`
  citado no `SKILL.md` não corresponde ao código testado)
- `test.py` + módulo `forensics_sqlite` (`DB`, `WAL`)
- Snapshot: `work/wal/snap/wal-test.db-wal` (16.512 B) e `-shm`

## Comandos
```bash
# snapshot consistente: copiar -wal/-shm com a conexão AINDA ABERTA
cp wal-test.db-wal wal-test.db-shm work/wal/snap/

PYTHONPATH=tools/src/forensics-sqlite python3 test.py work/wal/snap/wal-test.db
```

## Resultados
```
Version 2de218
Page size 1000
Sequence 00
Salt1 f206ae3c   Salt2 42da42b6
Checksum1 11950a05  Checksum2 7fbfbbec

Current position 4152   Page number 02
... b'...wal_userwal@lab.comWALPASS123user4111111111111111...'
Current position 8272   Page number 03
... b'...adminadmin@bookshop.comSuperSecurePass2025!admin...'
    CTF{Bl1nd_SQL1_M4st3r_T1m3_B4s3d_2025}
```
- Frames do WAL contêm o **registro novo** `wal_user / wal@lab.com / WALPASS123` (inserido
  via WAL, com o comentário "nota inserida via WAL").
- Também vazam todas as senhas em texto plano e a flag CTF do admin.

## Bug encontrado
`test.py` abria os arquivos em **modo texto** e quebrava com `UnicodeDecodeError` em
qualquer WAL real. Patch aplicado (2 lugares):
```python
open("{0}-wal".format(sys.argv[1]), 'rb')   # era 'r'
open(sys.argv[1], 'rb')
```

## Achados / armadilhas
1. **O WAL é removido/apagado no checkpoint ao fechar a conexão** — é preciso copiar o
   `-wal` (e o `-shm`) **enquanto o processo que escreve está vivo**. Snapshot depois do
   close geralmente vem vazio.
2. `-shm` é necessário para o índice do WAL; sem ele alguns campos podem não ser
   interpretados.
3. O script imprime `bytes` crus (`print(page)`) — não há parser de células; para extrair
   registros estruturados combinar com `strings`/`sqlite_dissect -w`.
4. `page.size` e `version` em hex são fáceis de confundir com decimal na leitura.
5. Checksums/salts são úteis para detectar frames de um WAL **antigo** reutilizado depois de
   `VACUUM`/reset (salt muda).

## Veredito
Skill validada após o patch `'rb'`. Trocar a URL do projeto e documentar o snapshot
"com conexão aberta" (senão o WAL não existe).
