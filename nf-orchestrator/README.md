# Desafio Técnico — Orquestrador de NF (Python + FastAPI + PostgreSQL + Alembic + Poetry + RabbitMQ)

## Objetivo
Construir um **orquestrador de Notas Fiscais (NF)** capaz de:
1. **Receber webhooks** de invoices da **Iugu** (payload JSON).
2. **Persistir** cada webhook no PostgreSQL (JSON bruto + metadados: data de recebimento, número da invoice).
3. **Publicar mensagem** em fila **RabbitMQ** após persistir.
4. **Consumir** a fila em um **serviço worker** (no mesmo monorepo) que:
   - Verifica se **produto** existe no banco.
   - Carrega **configurações de NF**.
   - **Simula** chamada POST para um serviço externo de emissão de NF.
   
> **Não é necessário integrar com Iugu nem com o emissor real.** Use **mocks**/coleções simples para simular os cenários.

---

## Requisitos Técnicos
- **Linguagem:** Python 3.11+
- **Web:** FastAPI
- **Gerenciador de dependências:** Poetry
- **Migrations:** Alembic
- **Banco:** PostgreSQL
- **Mensageria:** RabbitMQ
- **Container:** Docker / docker-compose
- **Debug:** Deve ser possível **rodar no Docker e depurar no VS Code** (attach).
- **Estilo de código:** PEP8 + type hints
- **Logs estruturados** (JSON ou chave=valor) com correlação `invoice_id`.

---

## Entrega Esperada
- Repositório com:
  - Código-fonte.
  - `README.md` claro (como este).
  - `docker-compose.yml` funcional.
  - Scripts de **seed** (produtos e configs de NF).
  - **Migrations** Alembic.
  - Arquivos de debug VS Code (`.vscode/launch.json`).
  - Exemplos de payload (Iugu) e **curl** para testar.
- Deve subir com um comando (ex.: `make up` ou `docker compose up -d`) e permitir debug por attach.

---

## Arquitetura (monorepo sugerida)
```
.
├─ apps/
│  ├─ orchestrator/           # FastAPI: Webhook + endpoints de leitura
│  │  ├─ app/
│  │  │  ├─ api/              # rotas FastAPI
│  │  │  ├─ db/               # models SQLAlchemy, session, repo
│  │  │  ├─ schemas/          # pydantic
│  │  │  ├─ services/         # publisher RabbitMQ
│  │  │  └─ main.py           # create_app(), lifespans
│  │  ├─ alembic/             # migrations
│  │  ├─ pyproject.toml       # poetry
│  │  └─ Dockerfile
│  └─ worker/                 # consumidor RabbitMQ
│     ├─ worker/
│     │  ├─ db/               # reuso de models ou somente leitura
│     │  ├─ services/         # consumer + client "emissor NF" mock
│     │  └─ main.py           # bootstrap do consumer
│     ├─ pyproject.toml
│     └─ Dockerfile
├─ seeds/
│  ├─ products.json           # produtos mock
│  └─ nf_configs.json         # configs NF mock
├─ .vscode/launch.json        # configs de debug
├─ docker-compose.yml
├─ .env.example
└─ Makefile
```

---

## Modelo de Dados (mínimo)
**Tabela `iugu_webhooks`**
- `id` (PK, UUID)
- `received_at` (timestamp, default now)
- `invoice_number` (varchar, index)
- `payload` (JSONB)

**Tabela `products`** (seed simples)
- `id` (PK, UUID) ou inteiro
- `sku` (varchar, único)
- `name` (varchar)
- `is_active` (bool)

**Tabela `nf_configs`** (seed simples)
- `id` (PK)
- `issuer` (varchar) — ex: “ACME-ISSUER”
- `default_nature` (varchar) — ex: “Venda”
- `extra` (JSONB) — parâmetros variados

---

## Fluxo Esperado

1. **POST /webhooks/iugu**
   - Recebe payload JSON (exemplo abaixo).
   - Extrai **invoice_number** (ver “Payload exemplo”).
   - Persiste em `iugu_webhooks`.
   - Publica mensagem no RabbitMQ (`exchange="nf"`, `routing_key="invoice.received"`):
     ```json
     {
       "invoice_number": "INV-123",
       "webhook_id": "uuid",
       "received_at": "2025-08-08T12:00:00Z"
     }
     ```
   - Responde `202 Accepted`.

2. **Worker (consumer)**
   - Ouve `queue="nf-invoices"`.
   - Ao consumir: 
     - Carrega o webhook e **valida** a existência do produto no banco (por `sku`).
     - Lê `nf_configs` para parâmetros.
     - **Simula** POST no “serviço emissor” (mock, endpoint local ou log).
     - Loga sucesso/erro com `invoice_number`.

---

## Endpoints mínimos (orchestrator)
- `GET /health` → `200 {"status":"ok"}`
- `POST /webhooks/iugu` → `202` (persiste + publica)
- `GET /invoices/{invoice_number}` → retorna metadados + payload
- **Opcional:** `POST /_simulate/iugu` → helper para enviar payload de teste ao webhook interno

---

## Payload exemplo (Iugu — simplificado)
```json
{
  "event": "invoice.status_changed",
  "data": {
    "id": "f3c1a7b1",
    "invoice_id": "INV-123", 
    "status": "paid",
    "total": 199.90,
    "items": [
      {"sku": "SKU-ABC", "description": "Produto XPTO", "quantity": 1, "price_cents": 19990}
    ],
    "customer": {
      "name": "Cliente Teste",
      "email": "cliente@exemplo.com"
    }
  }
}
```

> **Regra:** considere `invoice_number = data.invoice_id`.

---

## Publicação na fila (contrato da mensagem)
- **Exchange:** `nf` (tipo: direct)
- **Routing key:** `invoice.received`
- **Queue:** `nf-invoices` (bind em `invoice.received`)
- **Mensagem (JSON):**
  ```json
  {
    "invoice_number": "INV-123",
    "webhook_id": "uuid",
    "received_at": "2025-08-08T12:00:00Z"
  }
  ```

---

## Simulação do Emissor de NF (mock)
No **worker**, implemente uma função que faça um `POST` simulado:
- Pode bater em um endpoint local (ex.: `http://mock-emitter:8080/emitir`) **ou** apenas logar a requisição.
- Corpo do POST (exemplo):
  ```json
  {
    "invoice_number": "INV-123",
    "sku": "SKU-ABC",
    "issuer": "ACME-ISSUER",
    "nature": "Venda",
    "amount": 199.90,
    "customer": {"name": "Cliente Teste"}
  }
  ```
- Retorne sucesso/erro mockado.

---

## Variáveis de Ambiente (`.env.example`)
```
POSTGRES_HOST=postgres
POSTGRES_DB=nfdb
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
POSTGRES_PORT=5432

RABBITMQ_HOST=rabbitmq
RABBITMQ_USER=guest
RABBITMQ_PASSWORD=guest
RABBITMQ_PORT=5672
RABBITMQ_EXCHANGE=nf
RABBITMQ_ROUTING_KEY=invoice.received
RABBITMQ_QUEUE=nf-invoices

APP_PORT=8000
LOG_LEVEL=INFO
```

---

## docker-compose.yml (exemplo mínimo)
```yaml
version: "3.9"

services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    ports: ["5432:5432"]
    volumes:
      - pgdata:/var/lib/postgresql/data

  rabbitmq:
    image: rabbitmq:3-management
    environment:
      RABBITMQ_DEFAULT_USER: ${RABBITMQ_USER}
      RABBITMQ_DEFAULT_PASS: ${RABBITMQ_PASSWORD}
    ports:
      - "5672:5672"
      - "15672:15672"

volumes:
  pgdata:
```

---

## Seeds (exemplo JSON)
`seeds/products.json`
```json
[
  {"sku": "SKU-ABC", "name": "Produto XPTO", "is_active": true},
  {"sku": "SKU-DEF", "name": "Outro Produto", "is_active": true}
]
```

`seeds/nf_configs.json`
```json
{
  "issuer": "ACME-ISSUER",
  "default_nature": "Venda",
  "extra": {"series": "1", "env": "HML"}
}
```

---

## Execução
```bash
cp .env.example .env
docker compose up -d --build
```

---

## Critérios de Aceite
- [ ] `POST /webhooks/iugu` persiste e publica na fila.
- [ ] Worker consome, verifica produto, lê config e simula POST.
- [ ] Projeto sobe no Docker e permite debug no VS Code.

---

### Boa sorte! 🎯
