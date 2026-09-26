# CLAUDE.md — Moonery System

Guia de contexto para trabalhar neste repositório.
Última atualização: 26/09/2026, após a fase 3 (fluxo de entrega).

---

## 1. O que é o Moonery

Sistema de **gestão de entregas (deliveries)** com quatro atores:

| Papel (role) | Função no domínio |
|---|---|
| **Admin** | Cadastra clientes, usuários e entregas; atribui entregador; cancela; acesso total |
| **Client** | Destinatário da entrega; vê as suas e cancela antes da coleta |
| **Delivery Man** | Entregador; pega entrega livre, avança o status, registra falha e devolução |
| **Support** | Atende por chat; lê entregas e pode alterar status ou cancelar. **Não existe no código** — nem role, nem permissões, nem chat |

### Contexto do projeto

**Portfólio / estudo de arquitetura.** O valor está em demonstrar decisões bem
tomadas (camadas, inversão de dependência, mensageria, máquina de estados, RBAC por
permissão), não em cobertura de features. Diante de um trade-off entre profundidade
e quantidade, prefira profundidade e coerência do desenho.

### Decisões de escopo confirmadas

- **O WebSocket tem dois propósitos**: push de mudança de status do pacote, e
  **chat em tempo real com o Suporte**.
- **Rastreio é só por status.** Sem GPS, sem mapa, sem página pública de rastreio.
- **O cliente é o destinatário**, não o remetente. O remetente é a empresa que
  contratou a Moonery. Essa empresa é **implícita** — não há tabela dela.
- **Chat hub-and-spoke: o Suporte está sempre numa das pontas.** Cliente ↔ Suporte,
  Entregador ↔ Suporte, Admin ↔ Suporte. Não existe Cliente ↔ Entregador.
- **Conversa é canal geral por usuário**, com `delivery_id` nullable quando o
  assunto for um pacote. Não é helpdesk com múltiplos tickets.
- **Suporte poderá** ler entregas, conversar, alterar status e cancelar entrega.
  Não edita dados do cliente.
- **Atribuição de entrega é híbrida:** admin atribui *e* entregador pega livre.
- **Cliente cancela só antes da coleta** (`pending` e `attached`).
- **E-mail está no escopo** e já funciona (fase 2).

### Recomendações em aberto (não decididas)

- **Manter mono-tenant.** Um `company_id` nullable entra depois sem quebrar nada;
  pagar o preço de escopo por tenant antes do núcleo funcionar não se justifica.
- **Não tornar a entrega bidirecional.** Se o cliente precisar iniciar algo, o
  caminho é uma **solicitação de entrega** que o Admin aprova e converte em
  `delivery` — e o primeiro caso natural é devolução.
- **404 vs 409 no `attach`.** Hoje, tentar pegar entrega já tomada dá **404**,
  porque o escopo de visibilidade filtra antes do validador. Trocar por 409
  ("no longer available") melhora a UX e enfraquece um pouco a garantia de escopo.

---

## 2. Estrutura do repositório

Este repositório (`moonery-system`) é **apenas o repo de infra**. Contém o
`docker-compose.yml` e **três repositórios Git independentes aninhados**, todos no
`.gitignore` da raiz (não são submódulos):

```
moonery-system/               ← repo de infra
├── docker-compose.yml
├── docker/nginx/default.conf ← a conf que o compose realmente monta
├── .env                      ← credenciais do Postgres
├── api/                      ← repo separado: github.com/moonery-system/api
├── websocket-api/            ← repo separado (Hyperf)
└── frontend/                 ← repo separado (Vue 3)
```

**Commits em `api/`, `websocket-api/` e `frontend/` vão para os repos deles.**
Sempre confira onde está antes de commitar (`git -C api status`).

Convenção de commit, extraída do histórico — seguir à risca:

```
[ escopo ] tipo: descrição em inglês, minúscula, sem ponto final
```

Escopos: `backend`, `frontend`, `infra`, `ws-notification`. Tipos: `feat`, `fix`,
`create` (quando nasce um conjunto de arquivos), `refactor`, `delete`. Duas coisas
na mesma mensagem separadas por `;`. Granularidade: mediana ~3 arquivos.

---

## 3. Stack e serviços

| Serviço | Tecnologia | Container | Portas |
|---|---|---|---|
| API REST | Laravel 9 / PHP 8.2-fpm | `moonery-laravel` | 9000 |
| Web server | Nginx | `moonery-nginx` | 80, 443 |
| Banco | PostgreSQL 15 | `moonery-postgres` | 5432 |
| Admin de banco | Adminer | `moonery-adminer` | 8081 |
| Mensageria | RabbitMQ 3-management | `moonery-rabbitmq` | 5672, 15672 |
| E-mail de dev | Mailpit | `moonery-mailpit` | 1025 (SMTP), 8025 (UI) |
| WebSocket / consumers | Hyperf 3.1 / Swoole (PHP 8.3) | `moonery-hyperf` | 9501, 9502 |
| Frontend | Vue 3 + TS + Tailwind (vue-cli) | **não containerizado** | 8082 |

Redes isoladas: `db-network`, `rabbitmq-network`, `api-network`.

**Dependências principais** — api: `laravel/framework ^9.19`, `tymon/jwt-auth ^2.2`,
`php-amqplib ^3.7` · websocket-api: `hyperf/amqp`, `hyperf/websocket-server`,
`hyperf/database-pgsql` · frontend: `vue ^3.2`, `vue-router ^4`, `axios`, `tailwindcss`.

---

## 4. Como rodar

```bash
docker compose up -d

docker compose exec laravel composer install
docker compose exec laravel php artisan key:generate
docker compose exec laravel php artisan jwt:secret
docker compose exec laravel php artisan customs:refresh-db    # wipe + migrate + seed

# consumidor de e-mail — NÃO sobe sozinho, precisa ficar rodando
docker compose exec laravel php artisan customs:consume-emails

# Hyperf (consumer AMQP de websocket)
docker compose exec hyperf composer install
docker compose exec hyperf php bin/hyperf.php start

cd frontend && npm install && npm run serve
```

`customs:refresh-db` (`app/Console/Commands/WipeMigrateSeed.php`) faz
`db:wipe` + `migrate` + `db:seed`. O `DatabaseSeeder` cria também **1000 clientes
fake** — útil para paginação, pesado para uso diário.

E-mails de dev aparecem no Mailpit em `http://localhost:8025`.

### Credenciais semeadas

| E-mail | Senha | Role |
|---|---|---|
| admin@gmail.com | `admin` | Admin |
| client@gmail.com | `client` | Client |
| deliveryman@gmail.com | `deliveryman` | Delivery Man |

---

## 5. Modelo de dados

```
users ──┬── user_roles ──── roles ──── role_permissions ──── permissions
        ├── client_addresses
        ├── invites
        ├── user_notifications ──── notifications
        ├── logs
        └── deliveries (creator_id, delivery_man_id, client_id, client_address_id)
                 ├── delivery_items
                 └── delivery_status
```

- **Não existe tabela `clients`.** Cliente é um `user` com a role `Client`.
  `deliveries.client_id` e `delivery_man_id` são FKs para `users`.
- `users.password` é **nullable** (criado por convite) e `activated_at` marca ativação.
- `users` e `deliveries` usam `softDeletes`.
- **`deliveries.client_address_id`** é obrigatório, FK com **`restrictOnDelete`**:
  entrega é registro histórico, apagar endereço não pode apagar entrega.
  `ClientAddressService` recusa apagar endereço com entrega vinculada.
- `deliveries.tracking_code` é gerado na criação (`MNY-<ano>-<6 chars>`), único.
- `deliveries.delivered_at` é gravado ao entrar em `delivered`.
- `deliveries.scheduled_to` **nunca é preenchido** — agendamento não foi decidido.
- `client_addresses.is_primary` está comentado na migration.
- `logs.context` é `json`; `logs.user_id` é **NOT NULL** (ver `LogService`).

---

## 6. Arquitetura da API (Laravel)

```
Route (routes/api.php, middleware can:*)
  → FormRequest (app/Http/Requests) — validação
  → Controller (app/Http/Controllers) — fino, só orquestra
  → Service (app/Services) — regra de negócio
  → Repository (app/Repositories) — acesso a dados
      ↑ sempre atrás de uma interface em app/Contracts/Repositories,
        com binding em AppServiceProvider::register()  (Dependency Inversion)
```

- **Respostas:** sempre via `App\Utils\ApiResponse` (`success`, `error`, `notFound`,
  `unauthorized`, `forbidden`, `conflict`, `serverError`, `paginated`).
- **DI por construtor** com propriedades promovidas; **named arguments** à vontade.
- **Auditoria:** todo service chama `LogService::record(LogEventTypeEnum::X, $context)`.
  A assinatura aceita um **terceiro parâmetro de ator explícito** (`?int $userId`),
  necessário em rota de guest, onde `auth()->id()` é null e `logs.user_id` é NOT NULL.
  Sem ator resolvido, o log é silenciosamente ignorado em vez de estourar.
- **Notificações:** `NotificationService::notify()` monta a descrição via
  Strategy + Factory, persiste e publica nas filas. Título novo exige **três**
  coisas: caso no `NotificationTitleEnum` + nova Strategy + caso no `match` da
  `NotificationDescriptionFactory`.
- **Erro de regra de negócio:** lance `App\Exceptions\BusinessException`. O
  `Handler` a renderiza como **409** com a mensagem, então nenhum controller
  precisa de try/catch. Nome espelha de propósito o `App\Exception\BusinessException`
  do lado Hyperf.

### Autenticação e autorização

- **JWT** (`tymon/jwt-auth`), guard `api`, sem sessão.
- Login **não devolve token no corpo** — devolve cookie `token`
  httpOnly/secure/SameSite=Strict, TTL 60 min. Funciona porque o
  `LaravelServiceProvider` do jwt-auth 2.x registra o parser `Cookies`,
  `config/jwt.php` tem `decrypt_cookies => false` e o grupo `api` não aplica
  `EncryptCookies`. **Não mexa em um desses três isoladamente.**
  Para testar com curl: extraia o JWT do `Set-Cookie` e mande como
  `Authorization: Bearer` (curl não envia cookie `Secure` por http).
- **Autorização por permissão-string, nunca por role.** `AuthServiceProvider`
  registra `Gate::before` que delega para `User::hasPermission($ability)`.
- Permissões semeadas: `{users,clients}.{create,update,delete,view,viewAny}` e
  `deliveries.{create,update,delete,view,viewAny,viewAll,attach,assign,cancel,cancelAny}`.
- Por role: **Admin** todas · **Client** `clients.view`, `deliveries.view`,
  `deliveries.viewAny`, `deliveries.cancel` · **Delivery Man** `deliveries.view`,
  `deliveries.viewAny`, `deliveries.update`, `deliveries.attach`.

### Endpoints

```
# públicas (middleware guest)
POST   /api/auth/login
GET    /api/invite?token=          valida token de convite
POST   /api/invite                 gera/reenvia convite por e-mail
POST   /api/changePassword?token=  define senha e ativa a conta

# autenticadas (auth:api), cada uma com can:<permissão>
GET    /api/auth/user              devolve { user, permissions }
POST   /api/auth/logout

GET|POST  /api/users     GET|PUT|DELETE /api/users/{id}
GET|POST  /api/clients   GET|PUT|DELETE /api/clients/{id}
POST      /api/clients/{id}/addresses

GET    /api/deliveries                   can:deliveries.viewAny  paginado + busca
POST   /api/deliveries                   can:deliveries.create
GET    /api/deliveries/{id}              can:deliveries.view
PUT    /api/deliveries/{id}/status       can:deliveries.update   avança o fluxo
POST   /api/deliveries/{id}/cancel       can:deliveries.cancel   cliente cancela
POST   /api/deliveries/{id}/attach       can:deliveries.attach   entregador assume
DELETE /api/deliveries/{id}/attach       can:deliveries.attach   entregador desiste
PUT    /api/deliveries/{id}/deliveryman  can:deliveries.assign   admin (re)atribui
DELETE /api/deliveries/{id}              can:deliveries.delete
```

`GET /api/clients` e `GET /api/deliveries` têm paginação e busca
(`?search=&per_page=&page=`) e respondem no formato `ApiResponse::paginated`.
A busca de entrega casa `tracking_code` e nome do cliente.

### Fluxo de entrega: três camadas ortogonais

A máquina de estados é **por papel** sem nunca checar nome de papel, porque a
decisão é quebrada em três perguntas independentes:

| Pergunta | Onde mora |
|---|---|
| Você pode esse tipo de ação? | middleware `can:` na rota |
| O movimento é legal a partir do estado atual? | `DeliveryTransitionValidator` |
| Essa entrega é sua? | checagem de posse no `DeliveryService` |

A terceira é o que o `can:` não consegue expressar (regra de linha, não de tipo).

**Tabela de transições** — fonte única da verdade em
`app/Services/DeliveryTransitionValidator.php`. Cada destino declara a permissão
exigida e o ator (`ACTOR_ANY`, `ACTOR_OWNER_CLIENT`, `ACTOR_OWNER_DELIVERYMAN`):

```
pending                  → canceled_by_client (cancel, cliente dono)
                         → canceled_by_admin  (cancelAny)
attached                 → picked_up          (update, entregador dono)
                         → canceled_by_client / canceled_by_admin
picked_up                → in_transit         (update, entregador dono)
                         → canceled_by_admin
in_transit               → delivered / client_address_not_found / client_not_found
                                             (update, entregador dono)
                         → canceled_by_admin
client_address_not_found → in_transit (nova tentativa) / return_to_sender
client_not_found         → in_transit / return_to_sender

delivered, canceled_by_client, canceled_by_admin, return_to_sender → TERMINAIS
```

`pending ↔ attached` **não está na tabela de propósito**: pegar e desistir passam
por `attach`/`detach`, que fazem isso com **update condicional** —
`whereNull('delivery_man_id')` mais checagem de linhas afetadas — para dois
entregadores não pegarem a mesma entrega. `find()` seguido de `save()` perderia a
corrida. A reatribuição pelo admin também não passa pela tabela: é troca de
responsável, não de estado (mas atribuir uma `pending` move para `attached`).

**Escopo de visibilidade**, decidido por permissão no service (o repositório só
recebe a consulta a fazer):

- tem `deliveries.viewAll` → todas
- tem `deliveries.attach` → as suas + o pool livre (`delivery_man_id` null e `pending`)
- caso contrário → só as suas como cliente

Vale para `index` e `show`. **Entrega fora do escopo responde 404, não 403** —
um 403 confirmaria que ela existe.

---

## 7. Arquitetura do websocket-api (Hyperf)

Papel pretendido, lendo o mesmo Postgres da API:

1. **Push de notificação** — status do pacote em tempo real.
2. **Chat com o Suporte** — ainda sem uma linha escrita.

Implementado: `App\Amqp\Consumer\WebsocketNotificationConsumer` (consumer anotado) e
`App\Service\WebSocketService::sendToUser()`, que **só faz `print_r`**. Models
`Notification` e `User` sobre as tabelas da API.

**Não implementado:** nenhum servidor WebSocket em `config/autoload/server.php` (só
HTTP na 9501), `hyperf/websocket-server` está no composer sem uso, sem handshake,
sem autenticação de conexão, sem mapa `user_id → fd`.

### Contrato de mensageria

A API publica em um exchange **topic** `delivery.events`. O payload é mínimo — o
consumidor rebusca no banco (padrão *claim check*), e é por isso que o Hyperf tem
models próprios sobre as mesmas tabelas.

| Routing key | Payload | Fila | Consumidor | Estado |
|---|---|---|---|---|
| `invites.email` | `{invite_id}` | `emails.queue` | `customs:consume-emails` (Laravel) | funciona |
| `notifications.email` | `{notification_id}` | `emails.queue` | `customs:consume-emails` | funciona |
| `notifications.websocket` | `{notification_id}` | `notifications.queue.websocket` | Hyperf | consome e só imprime |

**Convite não é notificação.** Ele tem routing key própria porque o convidado não
consegue logar ainda (não veria notificação in-app) e porque o e-mail precisa do
token, que não deve ir para a coluna `description` de `notifications`.

**Dois cuidados operacionais:** quem declara a fila é o consumidor, então **se ele
nunca rodou a fila não existe** e o exchange topic descarta a mensagem em silêncio —
convite criado com o consumidor parado se perde (resgate: `POST /invite` reenvia).
E o serviço `rabbitmq` **não tem volume** no compose: recriar o container apaga tudo.

`RabbitMQPublisher` e `RabbitMQConsumer` conectam **sob demanda**, não no construtor.
Isso é deliberado: o Laravel instancia todos os comandos registrados para montar o
console, então conexão no construtor faria **qualquer `artisan`** exigir o RabbitMQ
de pé (quebraria `migrate` em CI). Não reverta isso.

---

## 8. Arquitetura do frontend (Vue 3)

- **Vue CLI** (não Vite), TypeScript, Tailwind. Mistura Options API (`NavBar`, views
  de clients) e `<script setup>` (`AuthLogin`, `AuthInvite`).
- `src/services/api.ts` — axios com `withCredentials: true`, `baseURL` de
  `VUE_APP_API_URL`. O interceptor **normaliza o erro** para
  `{ status, message, errors }` — leia `err.errors` e `err.message`, nunca
  `err.response.data`.
- `src/services/auth.ts` — cache em memória de `/auth/user` e helpers de permissão
  **com wildcard** (`clients.*`).
- `src/router/index.ts` — `beforeEach` valida sessão e checa
  `meta.requiredPermissions` / `meta.requiredAllPermissions` / `meta.guestOnly`.
- `PermissionGuard.vue` esconde UI por permissão; `usePaginatedFetch<T>` (em
  `src/types/api.ts`) consome o formato de `ApiResponse::paginated`.
- `vue.config.js` tem `historyApiFallback: true` — sem isso todo deep link (inclusive
  o do e-mail de convite) dá 404 no dev server.

Telas: Login, **Invite** (definição de senha), Home (placeholder), Clients (lista com
busca e paginação, criar, detalhe, editar, criar endereço), NotFound, Unauthorized.

**Não existe tela de entregas, usuários, notificações ou logs.** O `NavBar` tem links
para `/about` e `/users` sem rota registrada.

---

## 9. Estado atual

| Área | Status |
|---|---|
| Infra Docker (Postgres, Nginx, RabbitMQ, Mailpit, Laravel, Hyperf) | ✅ |
| Modelo de dados (16 migrations + seeders) | ✅ |
| RBAC (roles/permissions + Gate) | ✅ estrutura; ❌ role `Support` |
| Auth JWT por cookie httpOnly | ✅ |
| Onboarding por convite (e-mail → senha → ativação) | ✅ ponta a ponta |
| CRUD de usuários | ✅ backend, ❌ frontend |
| CRUD de clientes + endereços | ✅ |
| Entrega: criação com endereço e itens | ✅ |
| Entrega: máquina de estados, atribuição híbrida, escopo, paginação | ✅ backend, ❌ frontend |
| Notificação por e-mail | ✅ (consumidor precisa estar rodando) |
| Notificação por WebSocket | 🟡 consumer só faz `print_r` |
| Chat com o Suporte | ❌ nada |
| Log de auditoria | ✅ escrita; ❌ nenhuma leitura |
| Testes automatizados | ❌ só exemplos default |
| CI/CD | ❌ |

---

## 10. Lacunas conhecidas

### Grandes (áreas do escopo sem uma linha escrita)

**A. Chat com o Suporte.** Sem tabela de conversa nem mensagem, sem endpoint, sem
tela. Desenho mínimo pelo escopo: conversa por usuário (`delivery_id` nullable),
Suporte sempre numa ponta, mensagem com autor/corpo/marca de leitura. Caminho de
escrita decidido: **Laravel persiste e publica na fila, Hyperf só faz fan-out** —
uma fonte de verdade, reusando o pipeline existente.

**B. Papel `Support`.** Ausente do `RolesSeeder`. Faltam as permissões de chat e as
dos poderes decididos. E o vocabulário de status **não acomoda** cancelamento pelo
Suporte: só há `canceled_by_client` e `canceled_by_admin`, falta `canceled_by_support`.

**C. Servidor WebSocket.** Nenhum ws server configurado. Precisa de handshake
validando o JWT do cookie (cookie ignora porta, então ele chega em `ws://localhost:9502`
de graça) e de `Swoole\Table` para o mapa `user_id → fd` — array em memória daria
mapa parcial, porque `worker_num = swoole_cpu_num()`.

**D. Frontend de entregas.** Nenhuma tela, apesar de o backend estar completo.

### Menores

1. `GET /users` sem paginação nem busca.
2. Sem endpoint para o usuário listar as próprias notificações, nem "lida/não lida"
   (`NotificationRepository::findById` existe; falta rota e conceito de leitura).
3. Rotas de endereço: só `POST` registrada. `destroy` existe no controller sem rota;
   `update` é um TODO.
4. Sem reset de senha para usuário já ativo (`password_resets` existe sem uso), sem
   refresh de JWT, sem blacklist explícita.
5. `deliveries.scheduled_to` sem uso — agendamento não decidido.
6. `changePassword` grava senha e convite **sem transação**; erro no meio deixa
   estado parcial.
7. `rabbitmq` sem volume no compose.
8. Consumidor de e-mail não sobe sozinho (falta serviço no compose ou supervisor).
9. `ExampleTest` **falha**: afirma `GET /` == 200 contra o `abort(404)` deliberado de
   `routes/web.php`. É deletar o teste.
10. Frontend fora do `docker-compose.yml`.
11. `DeliveryItemsRequest::authorize()` retorna `false` (classe não usada).
12. `AuthController::login()` monta o cookie antes de checar se o attempt falhou
    (inofensivo, mas invertido).
13. `DatabaseSeeder` cria 1000 clientes fake por padrão.

---

## 11. Ao trabalhar neste projeto

- Controller → Service → Repository-atrás-de-interface. Repositório novo exige
  interface em `Contracts/Repositories` + binding em `AppServiceProvider`.
- Toda resposta passa por `ApiResponse`. Toda escrita gera log via `LogService`.
  Toda notificação passa por `NotificationService`.
- Rotas protegidas usam permissão-string (`can:recurso.acao`), **nunca** role.
  Regra que depende da linha (posse) vai no service, não no middleware.
- Transição de status **só** pela tabela do `DeliveryTransitionValidator`. Permissão
  nova de entrega exige entrada em `PermissionsSeeder` **e** em
  `RelatesPermissionsToRoles`.
- Erro de regra de negócio é `BusinessException` (vira 409), não `return false`.
- Ao mudar `notifications`/`users`/`user_notifications`, o **Hyperf lê as mesmas
  tabelas** — atualize `websocket-api/app/Model/` também.
- Ao mudar o contrato de mensagem publicada, atualize publisher (Laravel) e consumer
  (Hyperf) juntos.
- Confira o repo Git correto antes de commitar (a raiz ignora as três pastas).
