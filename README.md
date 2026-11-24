# E-commerce - Projeto 4
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

Há dois modos de testar: com Docker (recomendado) e local 
(`npm run dev` + `npm run worker`, para desenvolvimento). Os comandos
de `curl` são os mesmos; o que muda é como você inspeciona a fila e os logs.

> 💡 **Produtos disponíveis no seed:** `productId: 1` (O Programador Pragmático,
> livro físico, R$ 100 + frete = R$ 110) e `productId: 2` (Clean Code, e-book,
> R$ 50).

### Subir a API e o worker

**Modo Docker (recomendado):**

```bash
cp .env.example .env
docker compose up --build
```

Isso sobe quatro containers: `banco` (PostgreSQL), `redis`, `api` (porta 3000)
e `worker`. A API fica em `http://localhost:3000`; o worker consome a fila.

Confirme que estão de pé:

```bash
docker compose ps
# Esperado: banco, redis, api e worker, todos "Up" (banco e redis "healthy")
```

**Modo local (sem Docker):**

Requer Postgres e Redis acessíveis em `localhost` (ver "Como rodar (local,
sem Docker)" acima). Em **dois terminais separados**:

```bash
# Terminal 1 — API
npm run dev

# Terminal 2 — worker
npm run worker
```

Esperado:
- T1: `Server rodando em http://localhost:3000`
- T2: `Worker conectado no Redis em redis://localhost:6379. Esperando pedidos em orders:email...`

### Criar um pedido (POST)

Em outro terminal, com a API e o worker rodando:

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

> ⚠️ Não há `emailPreview` na resposta (diferente do Projeto 3).
> O e-mail é assíncrono e sai no log do worker. A resposta volta em
> milissegundos — o tempo do envio do e-mail não entra na latência da API.

**Ver o e-mail no log do worker:**

```bash
# Modo Docker:
docker compose logs -f worker

# Modo local (o worker está no seu terminal, é só olhar a saída):
# (a linha aparece direto no terminal onde você rodou `npm run worker`)
```

Esperado:

```
Pedido #1 recebido da fila. Enviando para cliente@ethereal.email...
Email enviado: https://ethereal.email/message/...
E-mail do pedido #1 enviado!
```

### Testar com cartão de crédito

Basta trocar o final do JSON:

```bash
curl -X POST http://localhost:3000/orders \
  -H "Content-Type: application/json" \
  -d '{
    "customer": "cliente@ethereal.email",
    "items": [
      { "productId": 2, "quantity": 1 }
    ],
    "paymentMethod": "credit_card",
    "paymentDetails": { "cardNumber": "4111111111111111", "cvv": "123" }
  }'
```

Esperado: `{"message":"Pedido processado com sucesso","orderId":<id>,"total":50}`.

### Buscar um pedido (GET)

```bash
curl http://localhost:3000/orders/1
```

Retorna o pedido com `id: 1` (status 200), incluindo `items`, `total`, `status`
e `createdAt`.

### Pedido inexistente (GET)

```bash
curl -i http://localhost:3000/orders/999
```

Retorna status **404** com:

```json
{"error":"Pedido 999 não encontrado"}
```

---

### Testando a fila

Os testes abaixo provam os critérios de aceite 3 e 4 do Projeto 4. Escolha o
bloco do modo que você está usando (**Docker** ou **local**) — não misture os
comandos, porque o Redis do Compose (`redis`) e o Redis local (`redis-local`)
são containers diferentes.

#### Worker parado: pedidos esperam na fila

**Modo Docker:**

```bash
# 1. Parar o worker
docker compose stop worker

# 2. Publicar 2 pedidos válidos (produtos 1 e 2)
curl -s -X POST http://localhost:3000/orders -H "Content-Type: application/json" \
  -d '{"customer":"p1@ethereal.email","items":[{"productId":1,"quantity":1}],"paymentMethod":"pix"}' > /dev/null
curl -s -X POST http://localhost:3000/orders -H "Content-Type: application/json" \
  -d '{"customer":"p2@ethereal.email","items":[{"productId":2,"quantity":1}],"paymentMethod":"pix"}' > /dev/null

# 3. A PROVA: fila tem 2 itens parados (worker parado não consumiu)
docker compose exec redis redis-cli LLEN orders:email
# Esperado: (integer) 2

# 4. Subir o worker de novo
docker compose start worker
docker compose logs -f worker
# Esperado: processa os 2 pedidos (#X e #Y)

# 5. Fila vazia
docker compose exec redis redis-cli LLEN orders:email
# Esperado: (integer) 0
```

**Modo local:**

```bash
# 1. Parar o worker (Ctrl+C no terminal do `npm run worker`)

# 2. Publicar 2 pedidos válidos
curl -s -X POST http://localhost:3000/orders -H "Content-Type: application/json" \
  -d '{"customer":"p1@ethereal.email","items":[{"productId":1,"quantity":1}],"paymentMethod":"pix"}' > /dev/null
curl -s -X POST http://localhost:3000/orders -H "Content-Type: application/json" \
  -d '{"customer":"p2@ethereal.email","items":[{"productId":2,"quantity":1}],"paymentMethod":"pix"}' > /dev/null

# 3. A PROVA (atenção: redis-local, não redis)
docker exec redis-local redis-cli LLEN orders:email
# Esperado: (integer) 2

# 4. Subir o worker de novo
npm run worker
# Esperado: processa os 2 pedidos

# 5. Fila vazia
docker exec redis-local redis-cli LLEN orders:email
# Esperado: (integer) 0
```

> ⚠️ **Se o `LLEN` der `0` em vez de `2`:** o worker **não estava parado** quando
> você publicou. Confira com `docker compose ps worker` (modo Docker) ou
> `sudo lsof -i :3000` (modo local, para ver se há `tsx watch` duplicado).
> O `LLEN` só dá `2` se **nenhum** worker estiver consumindo no momento do POST.

#### Critério 4 — Dois workers: cada pedido enviado uma vez só

**Modo Docker:**

```bash
# 1. Escalar para 2 workers
docker compose up -d --scale worker=2

# 2. Confirmar que há dois
docker compose ps
# Esperado: ecommerce-worker-1 e ecommerce-worker-2, ambos Up

# 3. Publicar 6 pedidos de uma vez
for i in 1 2 3 4 5 6; do
  curl -s -X POST http://localhost:3000/orders -H "Content-Type: application/json" \
    -d "{\"customer\":\"c$i@ethereal.email\",\"items\":[{\"productId\":1,\"quantity\":1}],\"paymentMethod\":\"pix\"}" > /dev/null
done

# 4. Ver a distribuição nos logs
docker compose logs worker
# Esperado: os 6 orderIds aparecem UMA VEZ SÓ, distribuídos entre worker-1 e worker-2

# 5. Fila vazia (nada duplicado, nada perdido)
docker compose exec redis redis-cli LLEN orders:email
# Esperado: (integer) 0

# 6. Voltar para 1 worker
docker compose up -d --scale worker=1
```

**Modo local:** o scale de workers não é trivial sem Docker Compose (você teria
que abrir dois terminais rodando `npm run worker`). 

#### Inspecionar a fila manualmente

Para ver o que está na fila a qualquer momento:

```bash
# Modo Docker:
docker compose exec redis redis-cli LLEN orders:email
docker compose exec redis redis-cli LRANGE orders:email 0 -1   # vê o conteúdo (JSON)

# Modo local:
docker exec redis-local redis-cli LLEN orders:email
docker exec redis-local redis-cli LRANGE orders:email 0 -1
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
