# CLAUDE.md — Moonery System

Guia de contexto para trabalhar neste repositório.
Última atualização: 27/09/2026, após a fase 6 (Suporte e chat). **O escopo
declarado está fechado.**

---

## 1. O que é o Moonery

Sistema de **gestão de entregas (deliveries)** com quatro atores:

| Papel (role) | Função no domínio |
|---|---|
| **Admin** | Cadastra clientes, usuários e entregas; atribui entregador; cancela; acesso total |
| **Client** | Destinatário da entrega; vê as suas e cancela antes da coleta |
| **Delivery Man** | Entregador; pega entrega livre, avança o status, registra falha e devolução |
| **Support** | Atende cliente, entregador e admin por chat; lê entregas e pode alterar status ou cancelar como suporte |

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
| Consumidor de e-mail | Laravel (`customs:consume-emails`) | `moonery-email-consumer` | — |
| Consumidor do assistente | Laravel (`customs:consume-assistant`) | `moonery-assistant-consumer` | — |
| WebSocket / consumers | Hyperf 3.1 / Swoole (PHP 8.3) | `moonery-hyperf` | 9501 (http), 9502 (ws) |
| Frontend | Vue 3 + TS + Tailwind (vue-cli) | `moonery-frontend` | 8082 |

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

# o consumidor de e-mail sobe com a stack (serviço email-consumer, restart
# unless-stopped) -- ele reinicia sozinho se o broker ainda não estava pronto

# Hyperf (consumer AMQP de websocket)
docker compose exec hyperf composer install
# o hyperf sobe sozinho (command + restart no compose); isto é só para rodar
# em primeiro plano e ver o log
docker compose exec hyperf php bin/hyperf.php start

cd frontend && npm install && npm run serve
cd frontend && npm run test:unit      # Jest: só a lógica em TypeScript puro (ver seção 8)
```

`customs:refresh-db` (`app/Console/Commands/WipeMigrateSeed.php`) faz
`db:wipe` + `migrate` + `db:seed`, e leva ~2s. Os **1000 clientes fake** saíram para
um seeder próprio, não chamado por padrão:
`php artisan db:seed --class=ClientsDemoSeeder`.

### Testes

```bash
docker compose exec postgres createdb -U root moonery_test   # uma vez
docker compose exec laravel php artisan test
```

O `phpunit.xml` aponta para **`moonery_test`**. Não aponte para o banco de
desenvolvimento: `RefreshDatabase` apagaria seus dados. sqlite ficou de fora de
propósito — o `LIKE` do Postgres é case-sensitive e o do sqlite não, então um teste de
busca passaria aqui e falharia de verdade.

### Análise estática

```bash
docker compose exec laravel composer analyse     # api, larastan, nível 5
docker compose exec hyperf composer analyse      # websocket-api, nível 5
```

Os dois estão em **zero erros no nível 5**, sem baseline e sem `ignoreErrors` largo. O
CI reprova erro novo nos dois repos.

**Não existe `hyperf/phpstan-extension`** — se você ler isso em algum lugar, está
errado (eu mesmo afirmei antes de verificar). O que o Hyperf precisava era anotação de
relação nos models, `query()->find()` em vez do `find()` estático, um `@param` próprio
no `WebSocketController` (o contrato herda um tipo do Swow, engine que não usamos) e um
`phpstan-bootstrap.php` definindo `BASE_PATH`.

E-mails de dev aparecem no Mailpit em `http://localhost:8025`.

### Credenciais semeadas

| E-mail | Senha | Role |
|---|---|---|
| admin@gmail.com | `admin` | Admin |
| client@gmail.com | `client` | Client |
| deliveryman@gmail.com | `deliveryman` | Delivery Man |
| support@gmail.com | `support` | Support |

---

## 5. Modelo de dados

```
users ──┬── user_roles ──── roles ──── role_permissions ──── permissions
        ├── client_addresses
        ├── invites
        ├── user_notifications ──── notifications
        ├── logs
        ├── conversations ──── messages (sender_id, delivery_id nullable)
        └── deliveries (creator_id, delivery_man_id, client_id, client_address_id)
                 ├── delivery_items
                 ├── delivery_status_history
                 └── delivery_status
```

- **Não existe tabela `clients`.** Cliente é um `user` com a role `Client`.
  `deliveries.client_id` e `delivery_man_id` são FKs para `users`.
- `users.password` é **nullable** (criado por convite) e `activated_at` marca ativação.
- `users` e `deliveries` usam `softDeletes`.
- **`deliveries.client_address_id`** é obrigatório, FK com **`restrictOnDelete`**:
  entrega é registro histórico, apagar endereço não pode apagar entrega.
  `ClientAddressService` **recusa apagar e editar** endereço com entrega vinculada,
  respondendo 409 — editar reescreveria o destino de uma entrega passada. Nesse caso o
  cliente cadastra um endereço novo.
- `deliveries.tracking_code` é gerado na criação (`MNY-<ano>-<6 chars>`), único.
- `deliveries.delivered_at` é gravado ao entrar em `delivered`.
- **`delivery_status_history`** guarda um passo por mudança de estado, com o ator e
  `created_at`. Linhas são imutáveis (`const UPDATED_AT = null`). É a fonte da linha
  do tempo — `logs` não serve para isso (ver seção 6).
- `deliveries.scheduled_to` **nunca é preenchido** — agendamento não foi decidido.
- `client_addresses.is_primary` está comentado na migration.
- `logs.context` é `json`; `logs.user_id` é **NOT NULL** (ver `LogService`).
- **`conversations`** tem `user_id` **unique**: um canal por pessoa, e é isso que faz
  `GET /conversations/me` ser idempotente. O lado do Suporte não é armazenado —
  qualquer atendente responde.
- **`messages.delivery_id`** é nullable e fica **na mensagem, não na conversa**: com um
  canal só por pessoa, o campo na conversa significaria "a última entrega comentada",
  que é vago. Na mensagem é preciso. `read_at` também é por mensagem.
- **Não existe tabela de reset de senha.** O reset usa a tabela `invites`, que já tem
  token, validade e marca de uso — `password_resets` foi apagada por ser um mecanismo
  paralelo que nunca foi usado.

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
  `deliveries.{create,update,delete,view,viewAny,viewAll,attach,assign,cancel,cancelAny,cancelAsSupport}`
  e `chat.viewAll`.
- Por role: **Admin** todas · **Client** `clients.view`, `deliveries.view`,
  `deliveries.viewAny`, `deliveries.cancel` · **Delivery Man** `deliveries.view`,
  `deliveries.viewAny`, `deliveries.update`, `deliveries.attach` · **Support**
  `clients.view`, `deliveries.{view,viewAny,viewAll,update,cancelAsSupport}`,
  `chat.viewAll` — atende e resolve, mas **não** edita cadastro.

### Endpoints

```
# públicas (middleware guest)
POST   /api/auth/login
GET    /api/invite?token=          valida token de convite
POST   /api/invite                 gera/reenvia convite por e-mail
POST   /api/changePassword?token=  define senha (ativa a conta, ou apenas troca)
POST   /api/auth/forgot-password   pede o link de reset

# autenticadas (auth:api), cada uma com can:<permissão>
GET    /api/auth/user              devolve { user, permissions }
POST   /api/auth/refresh           token novo; o antigo vai para a blacklist
POST   /api/auth/logout

GET|POST   /api/users    GET|PUT|DELETE /api/users/{id}   (index paginado, ?search, ?role)
GET        /api/roles    lookup do formulário de usuário
GET|POST   /api/clients  GET|PUT|DELETE /api/clients/{id}
POST       /api/clients/{id}/addresses
PUT|DELETE /api/clients/{id}/addresses/{addressId}

GET    /api/users?role=Delivery Man      lista filtrada por role (tela de atribuição)

# chat -- a própria conversa não exige permissão; a caixa de entrada sim
GET  /api/conversations              caixa de entrada, can:chat.viewAll
GET  /api/conversations/me           a própria conversa, criando se não existir
GET  /api/conversations/{id}         404 fora do escopo
POST /api/conversations/{id}/messages
PUT  /api/conversations/{id}/read
GET  /api/conversations/unread-count

# notificações -- recurso próprio do usuário, SEM can:, escopo vem do auth()->id()
GET    /api/notifications                paginado, com read_at por linha
GET    /api/notifications/unread-count
PUT    /api/notifications/{id}/read      idempotente; 404 se não for sua

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

`GET /api/deliveries/{id}` devolve, junto da entrega, `status_history` (a linha do
tempo) e **`available_transitions`**: a lista de movimentos que *este* usuário pode
fazer *agora*. A tela renderiza um botão por item e **não** conhece as regras — sem
isso o frontend viraria uma segunda cópia da máquina de estados.

`GET /api/clients` e `GET /api/deliveries` têm paginação e busca
(`?search=&per_page=&page=`) e respondem no formato `ApiResponse::paginated`.
A busca de entrega casa `tracking_code` e nome do cliente.

### Testes

`tests/Feature/DeliveryTransitionTest.php` (máquina de estados) e
`DeliveryAuthorizationTest.php` (escopo de visibilidade) são o que a suíte cobre — é
onde um revisor tenta quebrar. Ampliar cobertura não é o objetivo.

Convenções, todas em `tests/TestCase.php`:

- `seedDomain()` semeia permissões, roles e status. **Não** use `DatabaseSeeder`.
- `admin()`, `client()`, `deliveryman()` criam usuário com a role.
- `fakeBroker()` troca o `RabbitMQPublisher` por um falso. Chame no `setUp` de qualquer
  coisa que mexa em entrega: o domínio publica notificação, e esperar timeout de
  conexão AMQP em cada troca de status deixa a suíte rastejando.
- `actingAsUser($user)` usa `actingAs($user, 'api')`, **não** um JWT de verdade. Com
  token real, a segunda requisição dentro do mesmo teste continua autenticada como o
  usuário anterior — o jwt-auth guarda o token parseado no singleton, e nem
  `forgetGuards()` nem `unsetToken()` resolvem. O sintoma é cruel: a suíte fica verde
  afirmando o comportamento do usuário errado.

### Chat

Um canal por pessoa (`conversations.user_id` unique) e **qualquer atendente responde**:
o lado do Suporte não é armazenado.

**"Não lida" é decidido pela direção da mensagem, não por quem lê.** Como qualquer
agente pode responder, "quem leu" não é chave útil: mensagem escrita pelo atendido está
aguardando o Suporte (*inbound*), o contrário aguarda o atendido (*outbound*). Em
`MessageRepository` isso é um `whereColumn` entre `messages.sender_id` e
`conversations.user_id`. Ler um lado **não** limpa o outro.

A **caixa de entrada é do atendente** (`can:chat.viewAll` na rota). O requerente não
lista nada: ele tem uma conversa só, alcançada por `GET /conversations/me`, que faz
find-or-create.

Ao enviar, `ConversationService` resolve os destinatários — o dono da conversa mais
quem tem `chat.viewAll`, menos o remetente — via
`UserRepository::findByPermission()`, **por permissão e não por nome de role**.

### Assistente de IA no chat (fase 8)

No chat com o Suporte, um **assistente** responde o cliente sobre as entregas dele, com
ferramentas, e encaminha ao Suporte quando não sabe. Detalhes, problemas e correções, com
data: `api/docs/assistant-decision-log.md`. Diagrama: `api/README.md`.

```
ConversationService::sendMessage
  → AssistantDispatcher   cliente elegível? publica assistant.requests {message_id}
                          humano do Suporte respondeu? conversa vira handed_off
  → customs:consume-assistant (serviço assistant-consumer, 1 consumidor, prefetch 1)
  → AssistantRunner       claim idempotente → laço ≤5 iterações → resposta do bot
       LlmClient (GuardedLlmClient → GeminiLlmClient)   ToolRegistry → 5 ferramentas
  → ConversationService::sendAsAssistant  → chat.messages (mesmo push, websocket-api intocado)
POST /api/assistant/actions/{id}/confirm|reject   → DeliveryService::cancelDelivery
```

- **O bot é um usuário** (`assistant@moonery.local`, sem senha) com a role `Assistant`, que
  **não tem nenhuma permissão** — em especial não `chat.viewAll`, que é o que o faria entrar do
  lado do Suporte de toda conversa. Quem dispara o assistente é decidido por permissão:
  remetente é o dono da conversa **e** tem `assistant.use` (Client e Admin) **e** não tem
  `chat.viewAll`. Nunca por nome de role.
- **Escopo por construção.** O usuário das ferramentas vem da conversa (`ToolContext`), nunca dos
  argumentos do modelo. `ValidatedTool` entrega ao `handle()` só as chaves que as regras
  declaram. Ferramenta nova: estenda `ValidatedTool`, alcance entrega **só** pelos métodos
  `...ForClient` do `DeliveryInterface`, e devolva só o que o `DeliveryPresenter` permite
  (sem entregador, sem e-mail/telefone, sem CEP/complemento, sem nome de quem mexeu).
  Entrega alheia e inexistente devolvem o **mesmo** conteúdo. O `DeliveryService` **não** serve às
  ferramentas: ele depende de `auth()`, que não existe no consumidor.
- **Cancelar não é uma ferramenta.** `request_cancel_delivery` só cria uma linha em
  `assistant_pending_actions` (com validade). Quem cancela é o endpoint de confirmação, pelo
  mesmo `DeliveryService::cancelDelivery()` — o validador continua sendo a fonte da regra (a
  consulta sem efeito colateral é `DeliveryTransitionValidator::checkTransition()`). Com uma
  confirmação pendente, o texto da resposta é **fixo** (config), não o do modelo: ele não pode
  dizer que cancelou o que só perguntou. "Sim" digitado no chat não cancela.
- **Encaminhamento é permanente na v1.** `conversations.assistant_status` = `active` |
  `handed_off` (+ `handoff_reason`); virar `handed_off` é update condicional. Vira quando o
  modelo pede, quando qualquer falha/limite/teto acontece (mensagem fixa de fallback, sem erro
  para o usuário) ou quando um humano do Suporte responde. Reativar é decisão de produto em aberto.
- **Limites** (`config/assistant.php`): espaçamento entre chamadas, retry com backoff+jitter,
  teto diário de chamadas e de tokens, limite por usuário, prazo total, 5 iterações. Tudo que é
  "quantas vezes/quão rápido" mora em `GuardedLlmClient`, não no provedor. O dia do teto vira
  em `America/Los_Angeles` (é onde a cota do Gemini zera). Contadores em
  `assistant_usage_daily`, por upsert atômico — não em cache. Cada execução vira uma linha em
  `assistant_runs` (`message_id` unique é o que dá a idempotência).
- **Gemini** (`generateContent`, `Http::`, sem SDK): a chave só no cabeçalho `x-goog-api-key`.
  As `parts` cruas do turno do modelo ficam em `LlmMessage::$providerState` e voltam
  **intactas** (a `thoughtSignature` vem na própria parte `functionCall`; verificado com uma
  sonda real em 27/09/2026). A doc do Google já promove a *Interactions API* e chama
  `generateContent` de legado; migrar é reescrever só `GeminiLlmClient`.
- **A fila só existe se o consumidor já rodou.** Mesma armadilha do e-mail: `assistant.requests`
  publicada sem o serviço `assistant-consumer` de pé se perde em silêncio (o cliente fica sem
  resposta, mas a mensagem está gravada). Latência real: ~6–11 s por resposta; não há indicador
  de "digitando".
- **Banco existente:** `php artisan migrate` e `db:seed --class=AssistantSeeder` (idempotente).
  As tabelas `roles`/`permissions` **não têm timestamps**: use `insert()`, nunca `firstOrCreate`.
- **Privacidade.** A camada gratuita do Gemini pode usar entradas/saídas para treino. Em dev,
  **só dados de seed**.
- **Testes** (`tests/Feature/Assistant*`, `GuardedLlmClientTest`, `GeminiLlmClientTest`,
  `ConsumeAssistantQueueTest`): sem rede e sem chave (`FakeLlmClient`, `FakeClock`,
  `Http::fake`). O `phpunit.xml` liga `ASSISTANT_ENABLED=false`; teste do assistente liga com
  `config()->set('assistant.enabled', true)`. Duas lições: `Http::fake()` **empilha** stubs
  (o primeiro que casa ganha — recrie a factory entre respostas) e teste de endpoint
  sequencial **esconde** condição atômica de update: teste o repositório direto.
- **Frontend: feito** (seção 8). Contrato que ele consome: `assistant_status`/`handoff_reason`
  na conversa; `is_assistant` (flag do model `Message`, via `AssistantBot`, uma query por
  requisição) e `pending_action` (`id`, `status`, `expires_at`, `delivery_id`) nas mensagens;
  endpoints `confirm`/`reject`. O push WS de `chat.message` **não** leva nada disso (e o
  `sender` dele é uma string, o do REST é um objeto), então a tela **rebusca** a conversa ao
  recebê-lo, em vez de anexar o payload. `pending_action.error` e a mensagem do 409 são texto
  técnico e nunca são exibidos.

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

todo estado acima       → canceled_by_support (cancelAsSupport, ator qualquer)

delivered, canceled_by_client, canceled_by_admin,
canceled_by_support, return_to_sender → TERMINAIS
```

`pending ↔ attached` **não está na tabela de propósito**: pegar e desistir passam
por `attach`/`detach`, que fazem isso com **update condicional** —
`whereNull('delivery_man_id')` mais checagem de linhas afetadas — para dois
entregadores não pegarem a mesma entrega. `find()` seguido de `save()` perderia a
corrida. A reatribuição pelo admin também não passa pela tabela: é troca de
responsável, não de estado (mas atribuir uma `pending` move para `attached`).

**Histórico de estado.** `DeliveryService::recordStatusHistory()` é chamado nos
**cinco** pontos que mexem em estado: `createDelivery`, `transitionTo`,
`attachDelivery`, `detachDelivery` e `assignDeliveryman` (só quando de fato move).
**Não tente trocar isso por um observer do Eloquent:** `attach` e `detach` mudam o
status com `update()` no query builder, que não dispara eventos de model — o observer
perderia exatamente essas duas transições. Por isso `logs` também não serve como
linha do tempo: lá o attach grava `delivery_assigned`, não `delivery_status_update`.

**Escopo de visibilidade**, decidido por permissão no service (o repositório só
recebe a consulta a fazer):

- tem `deliveries.viewAll` → todas
- tem `deliveries.attach` → as suas + o pool livre (`delivery_man_id` null e `pending`)
- caso contrário → só as suas como cliente

Vale para `index` e `show`. **Entrega fora do escopo responde 404, não 403** —
um 403 confirmaria que ela existe.

---

## 7. Arquitetura do websocket-api (Hyperf)

Dois papéis, lendo o mesmo Postgres da API: **push de notificação** (funcionando) e
**chat com o Suporte** (fase 6, sem uma linha escrita).

Sobe sozinho com a stack (`command: php bin/hyperf.php start`, `restart: unless-stopped`),
servindo HTTP na 9501 e **WebSocket na 9502**.

- `App\Controller\WebSocketController` — `onOpen`/`onMessage`/`onClose`.
- `App\Service\JwtVerifier` — valida HS256 com `firebase/php-jwt` e
  `config('jwt.secret')`, que **tem de ser o mesmo `JWT_SECRET` do `api/.env`**. Se
  dessincronizar, toda conexão é recusada em silêncio — é o primeiro lugar a olhar.
- `App\Service\ConnectionRegistry` — mapa de conexões numa `Swoole\Table`.
- `App\Listener\CreateConnectionTableListener` — aloca a tabela no
  `BeforeMainServerStart`, no master.
- `App\Service\WebSocketService` — empurra via `Hyperf\WebSocketServer\Sender`.
- `App\Amqp\Consumer\WebsocketNotificationConsumer` — consumer anotado, inalterado.

**O mapa é indexado por `fd`, não por `user_id`.** Uma `Swoole\Table` tem colunas
fixas, então uma lista variável de fds por usuário não caberia; invertendo a chave,
cada aba é uma linha, "várias conexões por usuário" sai de graça e a limpeza no
`onClose` é um `del($fd)`. O push percorre a tabela filtrando por `user_id`.

**A tabela é alocada antes do fork** dos workers. Alocada depois, cada processo fica
com memória própria: o worker do WS grava o fd e o consumidor AMQP vê a tabela vazia.
Limite aceito: é memória de **um processo** — com mais de um nó, isso vira Redis.

**O push usa `Sender`, não o servidor cru.** O consumidor AMQP roda num processo
separado dos workers, e o `Sender` encaminha para o processo dono do fd.

**O handshake recusa no `onOpen`, não no 101.** O `onHandShake` padrão do Hyperf
aceita a conexão antes de o controller rodar; recusar no upgrade exigiria handshake
próprio. O token vem no cookie httpOnly — **cookie ignora porta**, então o que a API
gravou em `localhost` chega em `ws://localhost:9502` sem nada na query string.

### Três armadilhas deste serviço

**`SCAN_CACHEABLE=(true)` no Dockerfile.** O scan de anotações vem de
`runtime/container/scan.cache`. **Classe anotada nova não é vista** até você rodar
`rm -rf runtime/container` e reiniciar — o sintoma é um listener ou consumer que
simplesmente nunca dispara, sem erro nenhum.

**Os contratos do Hyperf têm parâmetros sem tipo:**
`OnMessageInterface::onMessage($server, $frame)`. Tipar (`Frame $frame`) quebra a
compatibilidade e mata o worker com fatal **no meio do handshake**, em loop.

**O `LoggerFactory` escreve em `runtime/logs/hyperf.log`, não no stdout.**
`docker compose logs hyperf` não mostra nada do que a aplicação registra — leia o
arquivo.

### Contrato de mensageria

A API publica em um exchange **topic** `delivery.events`. O payload é mínimo — o
consumidor rebusca no banco (padrão *claim check*), e é por isso que o Hyperf tem
models próprios sobre as mesmas tabelas.

| Routing key | Payload | Fila | Consumidor | Estado |
|---|---|---|---|---|
| `invites.email` | `{invite_id}` | `emails.queue` | `customs:consume-emails` (Laravel) | funciona |
| `notifications.email` | `{notification_id}` | `emails.queue` | `customs:consume-emails` | funciona |
| `notifications.websocket` | `{notification_id}` | `notifications.queue.websocket` | Hyperf | consome e só imprime |
| `assistant.requests` | `{message_id}` | `assistant.queue` | `customs:consume-assistant` (Laravel) | funciona |

**Convite não é notificação.** Ele tem routing key própria porque o convidado não
consegue logar ainda (não veria notificação in-app) e porque o e-mail precisa do
token, que não deve ir para a coluna `description` de `notifications`.

**Dois cuidados operacionais:** quem declara a fila é o consumidor, então **se ele
nunca rodou a fila não existe** e o exchange topic descarta a mensagem em silêncio —
convite criado com o consumidor parado se perde (resgate: `POST /invite` reenvia).
O serviço `rabbitmq` tem volume nomeado (`rabbitmq-data`), então fila e mensagens
sobrevivem a recriar o container.

**Chat:** `ChatMessageConsumer` escuta `chat.messages` na fila
`chat.queue.websocket` — **nunca ligada ao `emails.queue`**, senão cada mensagem de
chat mandaria um e-mail. O payload é `{message_id, recipient_ids}`: o conteúdo é
rebuscado (claim check), mas **quem** recebe vem pronto, porque depende de permissão e
resolver isso aqui duplicaria a autorização num segundo serviço.
`WebSocketService::sendToUserIds()` é o caminho único de push; o de notificação usa o
mesmo.

**Routing key nova exige reiniciar o consumidor.** Os bindings são declarados quando
o consumidor sobe, então acrescentar uma chave no código não a cria no broker até
`docker compose restart email-consumer`. Até lá a mensagem é publicada e descartada em
silêncio.

`RabbitMQPublisher` e `RabbitMQConsumer` conectam **sob demanda**, não no construtor.
Isso é deliberado: o Laravel instancia todos os comandos registrados para montar o
console, então conexão no construtor faria **qualquer `artisan`** exigir o RabbitMQ
de pé (quebraria `migrate` em CI). Não reverta isso.

---

## 8. Arquitetura do frontend (Vue 3)

- **Vue CLI** (não Vite), TypeScript, Tailwind. Mistura Options API (`AppShell`, views
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

- `src/components/deliveries/` — `DeliveryStatusBadge` (selo com cor por estado; as
  classes Tailwind estão escritas por extenso de propósito, porque strings montadas em
  runtime seriam removidas na purga), `DeliveryTimeline`, `DeliveryCard`,
  `DeliveryStatusActions` (renderiza um botão por `available_transitions`),
  `ClientPicker`, `DeliveryItemsForm`.
- `src/utils/status.ts` — `humanizeStatus("client_address_not_found")`.
- `formatDateTime` (`src/utils/date.ts`) aceita `string | Date | null`, porque a API
  devolve string.
- `PaginationItems` tem prop `itemLabel` (default `"items"`).
- `src/services/api.ts` — além de normalizar o erro, tenta **um** refresh ao receber
  401 e repete a requisição. `/auth/refresh`, `/auth/user` e `/auth/login` estão numa
  lista de exclusão: o guard de rota chama `/auth/user` em toda navegação, e deixar o
  401 dele disparar refresh que chama `/auth/user` travaria a aplicação.
- `src/services/auth.ts` — `getUserData()` deduplica a **requisição em voo**, não só o
  resultado. Sem isso cada `PermissionGuard` que monta junto dispara seu próprio
  `/auth/user` (eram 3 por carregamento de página).
- `src/components/chat/` — `ChatWidget` (botão flutuante à esquerda, para não bater no
  toast à direita), `MessageList` e `MessageComposer`. O widget é "minha conversa" para
  todo usuário autenticado, inclusive o Admin.
- **Assistente no chat** (a UI segue **em inglês**, como o resto do app; só o corpo das
  mensagens do bot vem em português do backend). A lógica mora em composables **sem HTTP e sem
  DOM** (recebem as chamadas por parâmetro), e os componentes são finos:
  - `useAssistantConfirmation` — máquina de estados do cartão (`pending`, `submitting`,
    `confirmed`, `rejected`, `expired`, `refused`, `network_error`, `closed`). 409/403/404
    são recusa; sem status, 5xx e 401 são erro de rede (com "Try again", que repete a
    **mesma** intenção). Expira por timer, limitado a 2³¹−1 ms. O servidor tem a última
    palavra **só quando traz novidade**: um reload que ainda diz `pending` não apaga o erro
    de rede nem a expiração. O motivo da recusa **não** é interpretado da mensagem do 409:
    `AssistantActionCard` busca `GET /deliveries/{id}` e diz o status atual (e cai no texto
    genérico se a entrega voltou a ser cancelável, para não afirmar algo falso).
  - `useAssistantWaiting` — fase `idle | waiting | timed_out` do "The assistant is
    replying…". É **derivada de dados** (última mensagem é minha + `assistant_status=active` +
    elegível), não de evento; limite de **45 s**; conta o tempo pelo `created_at` da mensagem.
    Elegível = `assistant.use` **e não** `chat.viewAll` (`canUseAssistant`), a mesma regra do
    backend — sem isso entregador e Admin esperariam para sempre. Enquanto espera, o widget
    recarrega a cada 5 s (até 2 min): rede de segurança contra push perdido.
  - `utils/singleFlight.ts` — refetch coalescida (rajada de pushes vira **uma** repetição).
    `utils/handoff.ts` — rótulos do motivo do encaminhamento para o Suporte.
  - `AssistantActionCard` (cliente: botões; Suporte: **só leitura**), `AssistantPresence`
    (região `aria-live` com indicador, aviso de demora e faixa de encaminhamento),
    `HandoffBadge`. `MessageList` tem `interactive` (widget sim, caixa de entrada não) e rola
    até a última mensagem **ao montar** — a pergunta de confirmação é a última.
  - Classes de tom do selo ficam escritas por extenso (purga do Tailwind).
- `src/views/support/SupportInboxView.vue` — "conversas que eu atendo", em `/support`
  atrás de `chat.viewAll`. Para o Admin, que é requerente **e** atendente, os dois
  coexistem e a rotulagem é o que evita confusão. Mostra o selo de encaminhamento na lista e
  na thread e atualiza sozinha pelo push.
- **Testes do frontend:** Jest 27 + ts-jest, `testEnvironment: node`, em `tests/unit/`, **sem**
  plugin do vue-cli e sem `@vue/test-utils` — só a lógica em `.ts` é testada (composables e
  utils; 96 testes). Componentes `.vue` **não** têm teste. Cuidados: `tsconfig.types` inclui
  `jest`; sem `@types/node` (use `globalThis`/`setTimeout`, não `global`/`setImmediate`);
  `jest.useFakeTimers("modern")` para expiração e prazo. O plano de teste manual, com dados de
  seed, está em `frontend/docs/assistant-manual-test.md`.
- **O build e o lint não checam tipos dos `.vue`** (um erro de tipo proposital passou no
  `npm run build`). O `vue-tsc` não rodou aqui (conflito de versões com o TS 4.5). Verifiquei
  os `<script setup>` extraindo-os para `.ts` e rodando o `tsc` do projeto; não checa templates.
- `src/views/users/` e `src/components/users/UserForm.vue` — CRUD de usuários, com o
  select de role alimentado por `GET /roles`.
- `src/services/websocket.ts` — cliente singleton, com reconexão em backoff
  exponencial até 30s. `connect()` no `AppShell` quando a sessão é confirmada,
  `disconnect()` no logout, e `onNotification(handler)` devolve a função de
  cancelamento. O cookie viaja sozinho; nada de token em query string.
- `NotificationToast.vue` e `NotificationBell.vue` — toast do push e contador de não
  lidas no `AppShell`. A tela de detalhe da entrega chama `loadDelivery()` ao receber
  push: o payload é genérico (título e descrição), então recarregar mantém um caminho
  de dados só.

Telas: Login, **Invite** (definição de senha), Home (últimas entregas e atalhos por permissão), Clients (lista com
busca e paginação, criar, detalhe, editar, criar endereço), **Deliveries** (lista com
abas Disponíveis/Minhas para quem tem `deliveries.attach`, criar, detalhe com linha do
tempo e ações), NotFound, Unauthorized.

- **Design system (overhaul visual).** Tema escuro único, tokens em
  `tailwind.config.js`: `ink` (azul-noturno; **`ink-900` é exatamente o fundo do
  `logo_moonery.png`**, por isso a logo não tem caixa visível na sidebar), `cream`
  (off-white quente, nunca `#fff`), `ember` (laranja, primário), `gold` (dourado,
  secundário), `rust`/`moss` (erro/sucesso). Fontes Sora (títulos) + Manrope.
  Texto sobre `ember`/`gold` é `text-ink-950`, não branco (contraste).
- `src/components/layout/` — `AppShell` (sidebar fixa em `lg+`, drawer no mobile),
  `SidebarLink`, `PageHeader`. Views **não** trazem mais `min-h-screen`/`max-w-*
  mx-auto`: quem dá o respiro é o shell. `src/components/ui/` — `AppIcon` (glifos
  Lucide inline), `EmptyState`, `SearchInput`, `ListSkeleton`. **Sem emojis**; ícone só
  via `AppIcon`. Ativos da marca em `src/assets/` e favicon em `public/`.
- Vue CLI dev server: se aparecer overlay de erro de prettier que não bate com o
  arquivo em disco, é cache do eslint — `rm -rf node_modules/.cache/eslint`.

**Não existe tela de notificações (lista completa) ou logs.**

O peso do item é `decimal(10,2)` e chega como **string** — converta com `Number()`
antes de somar, senão o total sai concatenado.

---

## 9. Estado atual

| Área | Status |
|---|---|
| Infra Docker (Postgres, Nginx, RabbitMQ, Mailpit, Laravel, Hyperf) | ✅ |
| Modelo de dados (25 migrations + seeders) | ✅ |
| RBAC (roles/permissions + Gate) | ✅ quatro roles |
| Auth JWT por cookie httpOnly | ✅ |
| Onboarding por convite (e-mail → senha → ativação) | ✅ ponta a ponta |
| CRUD de usuários | ✅ |
| CRUD de clientes + endereços | ✅ |
| Entrega: criação com endereço e itens | ✅ |
| Entrega: máquina de estados, atribuição híbrida, escopo, paginação | ✅ |
| Entrega: telas (lista, criação, detalhe com linha do tempo e ações) | ✅ |
| Notificação por e-mail | ✅ (consumidor sobe com a stack) |
| Notificação por WebSocket | ✅ push ao vivo, várias conexões por usuário |
| Notificações in-app (lista, não lidas, marcar como lida) | ✅ |
| Chat com o Suporte | ✅ tempo real, caixa de entrada, não lidas |
| Papel `Support` e `canceled_by_support` | ✅ |
| Log de auditoria | ✅ escrita; ❌ nenhuma leitura |
| Assistente de IA no chat | ✅ backend e frontend, verificados ponta a ponta com Gemini em dev (só dados de seed) |
| Testes automatizados | ✅ 135 testes de feature no backend; ✅ 96 testes unitários no frontend (composables e utils); ❌ nenhum teste de componente `.vue` |
| CI | ✅ `api` (suíte + lint + phpstan), `websocket-api` (phpstan) |
| Análise estática | ✅ os dois backends limpos no **nível 5** |
| Reset de senha e refresh de JWT | ✅ |
| Telas de usuários | ✅ |

---

## 10. Lacunas conhecidas

O escopo declarado está completo. O que resta são pontas, em ordem de retorno.

**O maior furo é a falta de teste de componente no frontend.** A lógica do assistente tem
teste unitário (composables e utils), mas os `.vue` e as telas em geral só foram verificados à
mão no navegador, e o build não checa tipos de `.vue` (ver seção 8).

1. `deliveries.scheduled_to` sem uso — agendamento nunca foi decidido.
2. ~~`tailwind.config.js` fora do padrão do prettier~~ — resolvido: `npm run lint` está com
   0 erros (restam avisos `no-explicit-any` herdados).
3. `websocket-api` tem 21 advisories de segurança em 6 pacotes, todos transitivos do
   skeleton do Hyperf e presos pelas constraints dele (`composer audit`).
4. O phpstan está preso ao 1.12 porque o `larastan/larastan` 2.x o pina; o 3.x pede
   Laravel 11+. Subir de versão é um upgrade de framework, não de ferramenta.
5. Nenhum teste no `websocket-api`, e nenhum de componente no frontend.
6. `attach` numa entrega já tomada responde 404 (o escopo filtra antes do validador).
   A tela trata como "não disponível"; virar 409 exigiria buscar fora do escopo.
7. O Jest 27 traz ~214 pacotes transitivos de desenvolvimento (não vão para o bundle) e o
   `npm install` podou 13 entradas órfãs do lockfile (`pinia` e afins, que nada usa).
8. `sender` no JSON de mensagem é o usuário inteiro, **com e-mail**: cliente recebe o e-mail
   dos atendentes. Vale um ticket; não mexi.
9. O corpo das mensagens fixas do bot é sempre em português (config), mesmo com a UI em
   inglês e o cliente escrevendo em inglês. As respostas livres do modelo **deveriam** seguir o
   idioma do cliente (o prompt manda), mas isso não foi testado.

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
- Teste novo herda de `Tests\TestCase` e usa `seedDomain()` + `fakeBroker()` no
  `setUp`, e `actingAsUser()` para trocar de identidade.
- Erro de regra de negócio é `BusinessException` (vira 409 pelo `Handler`), não
  `return false`. `return false` fica para "não encontrei", que vira 404.
- **Relação nova exige tipo de retorno** (`: BelongsTo`, `: HasMany`, …). É assim que o
  larastan resolve a relação; sem ele, todo uso dela vira erro de análise. Atributo novo
  que não seja coluna exige `@property`.
- `match` sobre enum é **exaustivo, sem `default`**. Assim, acrescentar caso ao
  `NotificationTitleEnum` sem a Strategy correspondente falha no phpstan em vez de
  estourar em produção.
- Precisa saber quem tem um poder? `UserRepository::findByPermission()`. Nunca
  `findByRole()` com nome cravado — a não ser que o nome da role *seja* o dado, como no
  filtro `GET /users?role=`.
