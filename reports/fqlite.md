# fqlite

## Objetivo
Recuperar dados deletados de SQLite com FQLite 5.0 (o `SKILL.md` descreve o modo
`.recover`/CLI da versão C, que não existe mais nesta distribuição Java).

## Ambiente
- FQLite **5.0** (distribuição Java/JavaFX, `labs/tools/fqlite-app/opt/fqlite/lib/app`)
- Java 21 + OpenJFX 11 + Xvfb (`xvfb-run`), instalados via apt
- Bases: `work/corpus/delete-lab.db` (2 páginas, sem freelist) e
  `work/corpus/freelist.db` (13 páginas, 11 na freelist)

## Problemas e workaround
1. **`Toolkit not initialized`** ao rodar a classe principal: `fqlite.base.GUI` é carregada
   antes do JavaFX. É necessário iniciar a plataforma:
   ```java
   Platform.startup(() -> {});
   ```
2. **NPE em `AppLog.<clinit>`**: o campo estático `fqlite.base.GUI.baseDir` precisa estar
   preenchido (`new File(System.getProperty("user.dir"))`) — via reflexão, antes de chamar
   `fqlite.base.FQLite.main`.
3. Wrapper usado: `/tmp/opencode/fqlite-wrapper/FqliteCli.java`
   ```bash
   APP=$(pwd)   # .../fqlite/lib/app
   javac --module-path "$APP" --add-modules javafx.controls,javafx.fxml \
         -cp "$APP/fqlite-5.0.jar" -d /tmp/opencode/fqlite-wrapper FqliteCli.java
   xvfb-run -a java --module-path . \
     --add-modules javafx.controls,javafx.fxml,javafx.swing \
     -cp "/tmp/opencode/fqlite-wrapper:fqlite-5.0.jar:$(ls *.jar | tr '\n' ':')" \
     FqliteCli nogui --threads:2 --loglevel:ERROR <db>
   ```

## Resultados
```
# freelist.db
page size 4096 Bytes
Number of pages:13
Total number of free list (trunk) pages 0000000b
Start free page recovery...
 Number of Entries in freepage list 11
Number of pages in the free list: 10
Scanned 10
Lines after free page recovery: 0

# delete-lab.db
INFO: Couldn't locate any free pages to recover.
Lines after free page recovery: 0
```
- Parse do header correto (page size, nº de páginas, freelist trunk/entries).
- **0 registros recuperados** nos dois cenários; nenhum arquivo de exportação é gerado.

## Achados / armadilhas
1. **A versão C foi removida do GitHub**; o 5.0 é Java/JavaFX com foco em GUI — não há
   `.recover` nem flag de export no CLI.
2. Em modo `nogui` o programa lê, loga e sai — sem `--help` (o argumento é tratado como
   arquivo) e sem opção de dump (`CSV/JSON`).
3. O log é a única saída; para uso real é preciso a GUI (`java ... fqlite.base.FQLite` ou o
   wrapper com `--gui`, sob Xvfb com VNC).
4. Mesmo com 11 páginas na freelist o recovery reporta 0 linhas — o extrator de
   free-page não reconstrói registros sem schema/células válidas.

## Veredito
Skill **não utilizável headless**. Corrigir no `SKILL.md`: FQLite 5.0 é GUI Java, exige
JavaFX + wrapper e não exporta em CLI; para automação usar `sqlite_dissect -c`,
`sqlite_carver` ou `undark`.
