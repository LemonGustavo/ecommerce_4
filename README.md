# E-commerce - Projeto 4
```markdown
Projeto de e-commerce refatorado para usar InversifyJS como container de IoC, com PostgreSQL e Docker Compose, e fila de e-mail assíncrona com Redis + worker, para obtenção de nota na matéria de Arquitetura de Software (2026). Contém dois modos de execução selecionáveis por variável de ambiente (`APP_ENV`), sem mudar regras de negócio do projeto original em https://github.com/LemonGustavo/ecommerce_3/tree/main.

## Estrutura

```
src/
├── container/     # container.ts (único lugar que importa classes concretas e monta o grafo
│                  # de dependências conforme APP_ENV) e types.ts (Symbols das dependências)
├── domain/        # IProduct, PhysicalProduct, DigitalProduct, ProductFactory
├── repositories/  # IOrderRepository, PrismaOrderRepository e InMemoryOrderRepository
├── providers/     # IMailProvider, EtherealMailProvider e ConsoleMailProvider
├── payment/       # IPaymentMethod, CreditCardPayment, PixPayment e PaymentMethodRegistry
├── queues/        # IOrderQueue (interface) e RedisOrderQueue (implementação com LPUSH)
├── services/      # OrderService (regra de negócio do pedido) e NotificationService
├── controllers/   # OrderController (só HTTP: valida entrada e traduz erros em status code)
├── lib/           # anota o que acontece no sistema, mostrando no terminal
│                  # e salvando no arquivo app.log pra consultar depois.
├── docs/          # contém a ADR explicando a escolha da lista Redis vs pub/sub
├── worker.ts      # consome a fila com BRPOP e envia o e-mail (processo separado da API)
└── app.ts         # pede o OrderController ao container e registra as rotas
```

## Pré-requisitos

- Docker e Docker Compose (plugin v2, comando `docker compose`)

Para rodar localmente sem Docker (opcional):

- Node.js
- npm
- Redis local (ou container Redis) — necessário para a fila

## Como rodar (Docker — recomendado)

```bash
cp .env.example .env
docker compose up --build
```

A API sobe em `http://localhost:3000`. O worker roda em outro container (`worker`) e consome a fila. O banco PostgreSQL roda no serviço `banco` (sem porta publicada) e os dados ficam no volume `pgdata` — `docker compose down` e `up` de novo não apagam os pedidos. Para apagar os dados, use `docker compose down -v`.

O Redis roda no serviço `redis` (sem porta publicada, somente a rede interna do Compose acessa).

Na primeira execução:

- O container da `api`:
  1. Gera o Prisma Client (`prisma generate`)
  2. Cria as tabelas no PostgreSQL (`prisma db push`)
  3. Roda o seed com os produtos (`prisma/seed.ts`)
  4. Sobe o servidor (`tsx src/server.ts`)
- O container do `worker`:
  1. Executa `npm run worker` (`tsx src/worker.ts`)
  2. Conecta no Redis e fica em loop com `BRPOP` aguardando pedidos

## Como rodar (local, sem Docker — opcional)

Útil para desenvolvimento rápido com `tsx watch`:

```bash
npm install
cp .env.example .env
# edite o .env e troque os hosts "banco" e "redis" por "localhost":
#   DATABASE_URL="postgresql://postgres:postgres@localhost:5432/ecommerce?schema=public"
#   REDIS_URL="redis://localhost:6379"
npm run prisma:migrate
npm run prisma:seed
npm run dev          # API
npm run worker       # worker (em outro terminal)
```

O modo de execução é definido pela variável `APP_ENV` no `.env`. O container (`src/container/container.ts`) lê essa variável e escolhe as implementações. Assim, nenhum service ou controller conhece `Prisma`, `Nodemailer`, `Redis` ou implementações concretas.

### Modo `dev` (padrão)

Usa Prisma como repositório e EtherealMailProvider para enviar e-mails de teste.

```env
APP_ENV=dev
```

### Modo `test` (sem banco, sem internet)

Usa **InMemoryOrderRepository** (array em memória) e **ConsoleMailProvider** (imprime o e-mail no console). Não precisa de banco nem de internet.

```env
APP_ENV=test
```

> Ao trocar o `APP_ENV`, reinicie o servidor (ou rode `docker compose up --build` de novo, se estiver no Docker).

## Fila de e-mail

No Projeto 3, o `POST /orders` enviava o e-mail de forma síncrona: a API só respondia depois que o e-mail era enviado, e uma falha no e-mail derrubava o pedido. No Projeto 4, o envio saiu da API;

1. O `OrderService` grava o pedido no banco e publica na fila (`IOrderQueue.publish`).
2. A `RedisOrderQueue` faz `LPUSH` na lista `orders:email` do Redis.
3. A API responde imediatamente (sem esperar o e-mail).
4. O `worker` (outro container) faz `BRPOP` na mesma lista, pega o pedido e chama o `NotificationService`, que envia o e-mail.

Pontos de arquitetura:

- O `OrderService` conhece somente a interface `IOrderQueue` — não importa Redis. Só `src/queues/` e `src/worker.ts` conhecem o Redis.
- A `RedisOrderQueue` é registrada no `container.ts` e injetada no `OrderService` via `TYPES.OrderQueue`.
- O worker pega o `NotificationService` do container (`container.get(TYPES.NotificationService)`), como a API faz com o controller.
- A fila usa lista (`LPUSH`/`BRPOP`), não pub/sub: itens ficam guardados se o worker estiver parado, e cada item vai para um worker só (com dois workers, cada pedido é enviado uma vez). A justificativa completa está no ADR `docs/adr-001-fila-de-email.md`.

## Testando

Com a API e o worker rodando, em outro terminal:

**Criar um pedido (POST):**

```bash
curl -X POST http://localhost:3000/orders \
  -H "Content-Type: application/json" \
  -d '{
    "customer": "cliente@ethereal.email",
    "items": [
      { "productId": 1, "quantity": 1 }
    ],
    "paymentMethod": "pix"
  }'
```

Resposta esperada (status 200) — **imediata**, sem esperar o e-mail:

```json
{
  "message": "Pedido processado com sucesso",
  "orderId": 1,
  "total": 110
}
```

- O e-mail **não** vem na resposta. Ele aparece no log do worker:

```bash
docker compose logs -f worker
# Esperado:
# Pedido #1 recebido da fila. Enviando para cliente@ethereal.email...
# Email enviado: https://ethereal.email/message/...
# E-mail do pedido #1 enviado!
```

**Buscar um pedido (GET):**

```bash
curl http://localhost:3000/orders/1
```

Retorna o pedido com `id: 1` (status 200).

**Pedido inexistente (GET):**

```bash
curl -i http://localhost:3000/orders/999
```

Retorna status **404** com `{"error":"Pedido 999 não encontrado"}`.

Também é possível testar com cartão de crédito trocando o final do JSON:

```json
"paymentMethod": "credit_card",
"paymentDetails": { "cardNumber": "4111111111111111", "cvv": "123" }
```

### Testando a fila

**Worker parado — pedidos esperam na fila:**

```bash
docker compose stop worker
# publique 2 pedidos com o curl acima
docker compose exec redis redis-cli LLEN orders:email
# Esperado: (integer) 2   ← pedidos parados na fila
docker compose start worker
# Esperado: worker processa os 2 pedidos, e LLEN volta a 0
```

**Dois workers — cada pedido enviado uma vez só:**

```bash
docker compose up -d --scale worker=2
# publique vários pedidos
docker compose logs worker
# Esperado: cada orderId aparece uma vez só, distribuído entre worker-1 e worker-2
docker compose exec redis redis-cli LLEN orders:email
# Esperado: (integer) 0
docker compose up -d --scale worker=1   # volta para 1 worker
```

## Arquitetura de injeção

- `src/container/types.ts` — possui um `Symbol` para cada dependência (incluindo `RedisUrl` e `OrderQueue`).
- `src/container/container.ts` — único arquivo que importa classes concretas e decide o que será injetado conforme `APP_ENV`.
- Todas as formas de pagamento estão no mesmo `TYPES.PaymentMethod`, e são consumidas via `@multiInject` no `PaymentMethodRegistry`.
- `MailProvider` e os repositórios usam `inSingletonScope()`, que define que essas dependências terão uma única instância compartilhada por toda a aplicação.
- A `RedisOrderQueue` também é `inSingletonScope()` — a API mantém **uma** conexão Redis, não uma por injeção.
- A rota `GET /orders/:id` passa pelas três camadas: `OrderController` → `OrderService` → `IOrderRepository`. O controller só trata HTTP (lê o `id`, devolve 200 ou 404), o service tem a regra (pedido inexistente é erro) e o repositório busca o pedido (implementado tanto no Prisma quanto no InMemory).
- O `OrderService` **não** conhece `NotificationService` nem Redis: conhece `IOrderQueue`. Quem envia e-mail é o worker.

## Infraestrutura

- `Dockerfile` — build da API (Node 22 Alpine, `npm ci`, `prisma generate`, `npm start`). O worker usa **a mesma imagem**, com `command: npm run worker` no compose.
- `compose.yaml` — quatro serviços:
  - `banco` (PostgreSQL 16, sem porta publicada, com healthcheck)
  - `redis` (Redis 7, **sem porta publicada**, com healthcheck `redis-cli ping`)
  - `api` (build local, exposta na porta 3000, `depends_on` com `condition: service_healthy` para `banco` e `redis`)
  - `worker` (mesmo build, `command: npm run worker`, sem porta, `depends_on` com `condition: service_healthy` para `banco` e `redis`)
- `.env.example` — `DATABASE_URL` apontando para o serviço `banco` e `REDIS_URL` apontando para o serviço `redis`.
- `prisma/schema.prisma` — provider `postgresql`, `url = env("DATABASE_URL")`.
- Volume `pgdata` — persiste os dados do PostgreSQL entre `down`/`up`.
- `docs/adr-001-fila-de-email.md` — ADR (Contexto / Decisão / Consequências) explicando por que lista (`LPUSH`/`BRPOP`) e não pub/sub.
