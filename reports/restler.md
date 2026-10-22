# restler

## Objetivo
Fuzzing stateful do `POST /b2b/v2/orders` (Juice Shop) com OpenAPI, cobrindo type-confusion,
JSON inválido e violações de contrato que scanners genéricos não pegam.

## Ambiente
- Sem imagem oficial funcional: `mcr.microsoft.com/restler-fuzzer/restler-fuzzer:latest`
  → "manifest unknown"; MCR tags list 404; Docker Hub `restlerfuzzer/restler` vazio;
  releases do GitHub 404 (só tags, até v9.3.1).
- Build local a partir do fonte:
  ```bash
  dotnet-install.sh --channel 8.0 --install-dir ~/.dotnet
  git clone --depth 1 -b v9.3.1 https://github.com/microsoft/restler-fuzzer.git
  python3 build-restler.py --dest_dir ~/restler-out
  ```
  (compila `compiler`, `driver`/`restler`, `engine`, `resultsAnalyzer` em ~3 min;
  requer .NET 8 SDK + Python)

## Comandos
```bash
DLL=~/restler-out/restler/Restler.dll
dotnet $DLL compile --api_spec juiceshop-swagger.yml
dotnet $DLL fuzz-lean --grammar_file Compile/grammar.py \
  --dictionary_file Compile/dict.json --settings Compile/engine_settings.json \
  --no_ssl --host 127.0.0.1 --target_port 3000
```

## Resultados
```
Request coverage (successful / total): 1 / 1
Attempted requests: 1 / 1
Bugs were found!
Bug buckets:
  InvalidValueChecker_500: 1
  main_driver_500: 1
Requests sent: main_driver 2, PayloadBodyChecker 256
```
- **500 Internal Server Error** reproduzido em `POST /b2b/v2/orders` com corpo malformado.
- O Juice Shop vaza **stack trace completo** + paths internos do container
  (`/juice-shop/build/server.js:357`, `node_modules/express/...`) na resposta 500.
- `PayloadBodyChecker` gerou 256 variações; `InvalidValueChecker` isolou o caso de 500.
- Artefatos: `bug_buckets/bug_buckets.txt`, `main_driver_500_1.replay.txt` (reprodução).

## Achados / armadilhas
1. **A imagem Docker documentada não existe**; o caminho é compilar do fonte (v9.3.1) ou
   montar a imagem a partir de `docker/Dockerfile` do repo.
2. **`--host 127.0.0.1:3000` falha** com `Name or service not known` — o engine usa o host
   como hostname no socket. Usar `--host 127.0.0.1 --target_port 3000`.
3. **`--basepath` não existe** (o basepath vem do `servers` da spec, renderizado como
   `restler_basepath`); passar `--basepath` aborta com "Invalid argument".
4. **Token**: `--authentication_token` não existe no v9.3.1. `--token_refresh_command`
   quebra no **Python 3.13** (`malformed node or string: <ast.Attribute ...>`) e ainda exige
   `--token_refresh_interval`. Workaround usado: substituir no `Compile/grammar.py`
   ```python
   primitives.restler_refreshable_authentication_token("authentication_token_tag"),
   # ->
   primitives.restler_static_string("Authorization: Bearer <JWT>\r\n"),
   ```
5. Sem token, cobertura fica `0/1` (401) e o fuzzing de corpo não roda.
6. `fuzz-lean` = `directed-smoke-test` + checkers; para profundidade usar `fuzz`.

## Veredito
Skill validada **com correções obrigatórias**: imagem Docker inexistente, `--host`/`--target_port`,
token em Python 3.13 e o fluxo de build local.
