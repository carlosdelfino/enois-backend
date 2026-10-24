# Rede — Social Platform Backend

Rede is a social platform backend built with **Spring Boot 4** (Java 25) following **hexagonal architecture** (ports & adapters).

## Prerequisites

- **JDK 25** (Eclipse Temurin recommended)
- **Maven** (or use the bundled `mvnw` wrapper)
- **Docker** (optional — for MariaDB/MySQL or running via docker-compose)

## Quick Start (dev profile)

```bash
./mvnw spring-boot:run
```

This starts with the `dev` profile using an **H2 in-memory database**. Swagger UI is available at [http://localhost:8081/swagger-ui.html](http://localhost:8081/swagger-ui.html).

## Profiles

| Profile | Database    | How to activate                          |
|---------|-------------|------------------------------------------|
| `dev`   | H2 (memory) | `SPRING_PROFILES_ACTIVE=dev` (default)   |
| `hml`   | MariaDB     | `SPRING_PROFILES_ACTIVE=hml`             |
| `prd`   | MySQL       | `SPRING_PROFILES_ACTIVE=prd`             |

### hml (MariaDB)

```bash
# Start MariaDB
docker run -d --name rede-mariadb \
  -e MARIADB_ROOT_PASSWORD=root \
  -e MARIADB_DATABASE=rede_hml \
  -e MARIADB_USER=rede_hml \
  -e MARIADB_PASSWORD=rede_hml \
  -p 3306:3306 mariadb:11

# Run with hml profile
SPRING_PROFILES_ACTIVE=hml ./mvnw spring-boot:run
```

### prd (MySQL 8+)

```bash
# Start MySQL
docker run -d --name rede-mysql \
  -e MYSQL_ROOT_PASSWORD=root \
  -e MYSQL_DATABASE=rede_prd \
  -e MYSQL_USER=rede_prd \
  -e MYSQL_PASSWORD=rede_prd \
  -p 3307:3306 mysql:8

# Run with prd profile
SPRING_PROFILES_ACTIVE=prd ./mvnw spring-boot:run
```

## Building

```bash
./mvnw clean package -DskipTests
```

Produces `target/rede-0.0.1-SNAPSHOT.jar`.

## Running Tests

```bash
./mvnw test
```

All 24 integration tests must pass.

## Pentest local e staging autorizado

O helper foi movido para `scripts/tests/pentest.sh`. Execute somente contra ambiente
próprio/autorizado, nunca produção. São necessários **Bash 4+, Python 3.11+ e curl**; nenhuma
ferramenta é instalada automaticamente. O padrão é `http://localhost:8081`, alinhado a
`server.port=8081`. Se a porta ou o prefixo mudar no ambiente, informe a URL-base:

```bash
scripts/tests/pentest.sh
scripts/tests/pentest.sh http://127.0.0.1:8081
```

Por padrão, somente `localhost`, `127.0.0.1` e `::1` são autorizados. Para staging,
defina `ENOIS_STAGING_HOSTS` no ambiente confiável do operador: hosts canônicos exatos
separados por vírgula, sem esquema, porta ou curingas. Informe também a URL-base do
host autorizado. Ter `staging` no nome não concede autorização. A lista não deve vir
de entrada arbitrária de PR; o próprio script pode ser alterado por um PR, portanto
aprovações e controle de saída de rede precisam existir fora dele.

Antes de qualquer scanner, o helper consulta `${TARGET}/health` com `curl -fsS`,
timeout de conexão de 5 segundos e total de 15 segundos. Apenas HTTP 2xx passa;
redirecionamentos não são seguidos. Alvo inválido/não autorizado, curl ausente ou
pré-check com falha encerram com código **2**, sem iniciar scanners. Esse teste
comprova disponibilidade HTTP, não a identidade do serviço: confirme o ambiente
antes de autorizar. Um gateway deve encaminhar `/health` sob o prefixo informado.

O ZAP recebe uma cópia JSON local validada da especificação em
`${TARGET}/api/openapi.json`, ou de `OPENAPI_URL` na mesma origem autorizada. São
aceitos OpenAPI 3.0/3.1 e Swagger 2.0; referências externas, servidores fora da
URL-base e callbacks/webhooks são recusados. O helper normaliza os servidores para
o alvo validado antes de importar. Se Springdoc estiver desabilitado (padrão), a
resposta não for uma especificação válida ou faltar ZAP API scan, a etapa será
registrada como falha: os demais scanners continuam, mas a saída final será **1**.
Um baseline não substitui o API scan. Habilite `SPRINGDOC_ENABLED=true` **no backend**
apenas em dev/staging protegido por VPN/controle de acesso; definir a variável só
no shell do helper não habilita o serviço remoto. O script não muda essa configuração.

Scanners disponíveis são detectados via PATH. SQLMap exige opt-in
`PENTEST_SQLMAP=1`; mesmo com risco baixo, scans ativos podem alterar dados ou causar
carga. Use um ambiente descartável. Ferramentas podem descobrir links, seguir
redirecionamentos ou resolver DNS novamente: a validação inicial não substitui
isolamento de rede/egress nem protege sozinha contra DNS rebinding.

Relatórios ficam na raiz do repositório em `reports/pentest/<data>_<id>/`, inclusive
quando o helper é chamado a partir de outro diretório. Cada ferramenta mantém seu
log; `SUMMARY.log` registra código real e resultado. Falhas/achados/alertas geram
saída **1**; **0** significa que as etapas executadas terminaram sem sinalização,
não que a aplicação está segura. Ferramentas opcionais ausentes deixam o resumo
**INCOMPLETO** e são listadas em `MISSING.txt`.

Gitleaks usa `--redact` para logs e JSON e carrega `.devin/gitleaks.toml`, que precisa
acompanhar o helper no checkout. As regras padrão são preservadas; somente segmentos
`node_modules`, `target`, `.git` e `reports` são excluídos. Fontes e `.env` continuam
sob análise. `.gitleaksignore` ignora fingerprints de achados, não diretórios.
Relatórios podem conter informações sensíveis de outros scanners: não publique nem
versione a pasta `reports/` (já ignorada pelo Git).

### Validação e critérios de aceitação do helper

```bash
bash -n scripts/tests/pentest.sh scripts/tests/pentest_test.sh
bash scripts/tests/pentest_test.sh
shellcheck scripts/tests/pentest.sh scripts/tests/pentest_test.sh
```

Os testes usam ferramentas de rede simuladas e um repositório temporário, sem
chamadas externas. Se Gitleaks estiver instalado, validam também exclusões e redação
em uma fixture sintética local; caso contrário, essa checagem é explicitamente
marcada como não executada. Nenhum backend ativo é necessário para a suíte Bash.

| Critério | Evidência automatizada |
|---|---|
| `PENTEST-01` | Sem argumento, usar porta 8081 e consultar `/health` antes dos scanners. |
| `PENTEST-02` | Autorizar apenas loopback ou hosts exatos; validar também `OPENAPI_URL`. |
| `PENTEST-03` | Dependência/HTTP indisponível impede todos os scanners. |
| `PENTEST-04` | OpenAPI inválido, externo ou indisponível causa falha final sem impedir outras etapas. |
| `PENTEST-05` | Gitleaks preserva regras, exclui artefatos e não grava a fixture em claro. |
| `PENTEST-06` | Novo caminho resolve a raiz independentemente do diretório de invocação. |
| `PENTEST-07` | Preservar códigos de ferramentas, sinalizar falhas/achados e continuar coleta. |
| `PENTEST-08` | Exercitar ZAP local/Docker, SQLMap opt-in e ausência de API scanner. |

## Docker

```bash
# Build and start (defaults to dev profile with H2)
docker compose up --build

# With hml profile and MariaDB
docker compose --profile hml up --build
```

## API Documentation

- **Swagger UI**: [http://localhost:8081/swagger-ui.html](http://localhost:8081/swagger-ui.html)
- **OpenAPI JSON**: [http://localhost:8081/api/openapi.json](http://localhost:8081/api/openapi.json)

## Monitoring

- **Health**: `GET /actuator/health`
- **Metrics**: `GET /actuator/metrics`
- **Prometheus**: `GET /actuator/prometheus`

## Architecture

The project follows **hexagonal architecture**:

```
src/main/java/com/socialcoisa/rede/
├── {domain}/domain/         # Entities, value objects, domain services
├── {domain}/application/    # Use cases / ports (inbound)
├── {domain}/adapters/in/    # REST controllers (inbound adapters)
└── {domain}/adapters/out/   # JPA/HTTP repositories (outbound adapters)
```

Modules: `identity`, `feed`, `media`, `network`, `profile`, `messaging`, `notifications`, `search`, `community`, `admin`.

## Deploy (EKS via GitOps)

O deploy **não** se faz por `fly deploy`. O fluxo é:

1. Push em `main` de `florestaencantada/enois-backend`.
2. GitHub Actions (`.github/workflows/build-and-push.yml`) builda a imagem Docker, publica no ECR
   (`818928662779.dkr.ecr.us-east-1.amazonaws.com/enois/backend:<sha>`) e atualiza o manifesto em
   `florestaencantada/integration_gitops` → `clusters/forest-dev/applications/enois-app.yaml`.
3. O Argo CD (Application `forest-dev-applications`) sincroniza e faz o rollout no cluster `forest-dev`,
   namespace `forest-apps`, Deployment `socialcoiso-back`.

### Roteamento (importante)

O `HTTPRoute` do gateway faz `URLRewrite ReplacePrefixMatch: /` para o caminho `/api`, ou seja, **remove o
prefixo `/api` antes de encaminhar ao backend**. Por isso os controllers deste projeto são mapeados na
**raiz** (`/auth/...`, `/feed`, `/health`) e o prefixo `/api` é responsabilidade do gateway. O front sempre
chama `https://enois.ia.br/api/...`. Esse é o mesmo padrão do app irmão (skillify) — não adicione `/api`
nos `@RequestMapping` do backend.

Aviso de segurança decorrente disso: como o gateway encaminha `/api/*` **sem autenticação**, tudo que o
backend expõe fica público em `https://enois.ia.br/api/...` — inclusive `/api/actuator/*` (que chega em
`/actuator/*`) e `/api/api/openapi.json`. Por isso os defaults abaixo são restritivos.

## Configuração e ações manuais

### Variáveis de ambiente (runtime do Deployment)

| Variável | Obrigatória | Observação |
|---|---|---|
| `SPRING_PROFILES_ACTIVE` | sim | `qa` no cluster |
| `DB_HOST` / `DB_PORT` / `DB_NAME` / `DB_USER` / `DB_PASSWORD` | sim | vêm do Secret `socialthing-db-secrets` |
| `REDE_ADMIN_EMAILS` | sim | allowlist (separada por vírgula) para `/admin/reports/**`. **Vazio = ninguém é admin.** |
| `REDE_FRONTEND_URL` | sim | base do front; o callback OAuth volta em `${REDE_FRONTEND_URL}/auth/callback` |
| `REDE_BACKEND_URL` | sim | base pública do back; monta os redirect URIs do OAuth |
| `FRONTEND_URL` | sim | link do e-mail de reset de senha: `${FRONTEND_URL}/auth/reset?token=...` |
| `MAIL_HOST` / `MAIL_PORT` / `MAIL_USERNAME` / `MAIL_PASSWORD` | sim | SMTP do "esqueci minha senha". Sem isso o e-mail não sai (o erro é logado e engolido). |
| `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` | p/ login Google | |
| `APPLE_CLIENT_ID` / `APPLE_TEAM_ID` / `APPLE_KEY_ID` / `APPLE_PRIVATE_KEY` | p/ login Apple | |
| `IMAGEKIT_URL_ENDPOINT` / `IMAGEKIT_PUBLIC_KEY` / `IMAGEKIT_PRIVATE_KEY` | p/ mídia | |
| `CORS_ALLOWED_ORIGINS` | não | padrão `*` |
| `OAUTH_STATE_COOKIE_SECURE` | não | padrão `true`; use `false` só em dev http |
| `SPRINGDOC_ENABLED` | não | padrão `false`. `true` liga `/api/openapi.json` + Swagger UI (use só em dev/VPN) |
| `MANAGEMENT_ENDPOINTS` | não | padrão `health`. Para métricas use `health,prometheus` **e bloqueie `/api/actuator/*` no gateway** |
| `SUPPORT_EMAIL` | não | e-mail do time que recebe os pedidos de suporte. Sem ele o pedido ainda fica registrado e visível em `/admin/support` |

### Passo a passo das ações manuais

1. **Criar/atualizar o Secret do banco** (`socialthing-db-secrets`) com as chaves `host`, `port`, `dbname`,
   `username`, `password`. Sem isso o pod não sobe (Flyway falha no boot).
2. **Definir `REDE_ADMIN_EMAILS`** no Deployment com os e-mails que podem moderar. Enquanto estiver vazio,
   `/admin/reports/**` responde 403 para todos.
3. **Configurar o SMTP** (`MAIL_*`) e `FRONTEND_URL` para o reset de senha funcionar de ponta a ponta.
4. **Configurar o OAuth**: criar as credenciais no Google Cloud e no Apple Developer e cadastrar os
   redirect URIs exatamente como
   `${REDE_BACKEND_URL}/api/auth/oauth/google/callback` e `.../apple/callback`. Para Apple, use
   `response_mode=form_post` (o backend aceita o callback via POST).
5. **Rotacionar segredos vazados** (ver seção "Segurança").
6. Verificar o rollout: `kubectl -n forest-apps get pods -l app=socialcoiso-back` e
   `kubectl -n forest-apps logs deploy/socialcoiso-back`.

## Fluxos de produto (referência rápida)

- **Vagas**: `POST /jobs` (exige reputação > 150) cria; `GET /jobs` lista (com `applied`/`saved`/`owner` por
  usuário); `GET /jobs/mine` = vagas que publiquei (traz `applicationsCount`); `GET /jobs/applications` = vagas
  em que me candidatei; `POST /jobs/{id}/applications` é a inscrição (`409` se repetida); `PUT`/`DELETE /jobs/{id}`
  são do autor. No app: `/vagas` (listar/criar/inscrever), `/vagas/{id}` (detalhe + inscrever/salvar/excluir) e
  `/vagas/minhas` (minhas vagas e minhas candidaturas).
- **Suporte**: `POST /support/requests` (autenticado) abre o pedido; `GET /admin/support` (admin) lista a fila em
  aberto. No app: `/suporte`, alcançável pelo menu da conta.
- **Consentimento**: `POST /auth/signup` exige `acceptedTerms: true` (termos + 18 anos); o instante do aceite é
  gravado em `users.terms_accepted_at`. Sem esse campo a resposta é `400 TERMS_NOT_ACCEPTED`.

## Segurança — pendências que exigem ação manual

- **`dump_rede.sql` está versionado** com dados pessoais reais (e-mails, hashes de senha) — foi commitado
  com a mensagem "foda-se a LGPD". Remover do HEAD **não** apaga do histórico do Git. Ação: remover o
  arquivo, reescrever o histórico (`git filter-repo`) ou considerar o repositório comprometido, **trocar as
  senhas dos usuários afetados** e avaliar notificação conforme a LGPD.
- **Segredos em texto puro**: conferir e rotacionar `DB_PASSWORD`, credenciais de e-mail, chaves do
  ImageKit e credenciais OAuth caso tenham vazado (o `.env` do front já vazou a `IMAGEKIT_PRIVATE_KEY`).
- **`/admin/reports/**`** agora exige admin via `REDE_ADMIN_EMAILS`; antes qualquer usuário logado
  aprovava/descartava denúncias.
- **Recomendações** (`GET /recommendations/received|given/{userId}`) seguem **sem autenticação** por serem
  exibidas em perfis públicos — avalie se devem exigir login.

### Já corrigido

- **Doc da API público**: `GET /api/openapi.json` (stub manual) e `GET /api/api/openapi.json` + Swagger UI
  (springdoc) respondiam 200 sem autenticação, entregando o mapa completo da API. Agora o stub foi removido
  e o springdoc fica desligado por padrão (`SPRINGDOC_ENABLED=false`).
- **Actuator público**: `/api/actuator/prometheus` devolvia ~110 KB de métricas (JVM, pool de conexões,
  URIs) e `/api/actuator/metrics` idem. Agora só `health` é exposto e o índice `/actuator` também.
- **Sem headers de segurança**: adicionados HSTS, `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`,
  `Referrer-Policy: no-referrer`, `Permissions-Policy` e CSP restritiva (`default-src 'none'`) nas respostas
  da API — ver `SecurityHeadersFilter`.
- **Email de terceiros** em `GET /users/{username}`, `/users/by-id/{id}`, `/users/search` e `/ranking`
  (era público, sem login): removido da projeção pública.
- **`/admin/reports/**`** exigia apenas sessão: agora exige admin (`REDE_ADMIN_EMAILS`).

Testes de regressão cobrem os dois primeiros itens em `ContractControllerTests`.
