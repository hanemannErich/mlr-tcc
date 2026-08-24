# Agentes com Acesso a Dados Bancários no PostgreSQL

> Arquitetura de referência para agentes de IA que acessam dados financeiros sensíveis, com foco em segurança, compliance e implementação prática com LangGraph, Aegra e PostgreSQL.

---

## 1. Contexto e Desafios

Agentes de IA que operam sobre dados bancários enfrentam requisitos únicos:

- **Dados altamente sensíveis:** saldos, transações, CPF, endereços
- **Regulação estrita:** LGPD, BACEN, CMN, Open Finance
- **Auditabilidade:** toda ação deve ser rastreável
- **Baixa tolerância a erros:** falhas têm impacto financeiro direto
- **Multi-tenancy:** cada cliente só vê seus próprios dados

---

## 2. Arquitetura de Referência (LangGraph + Aegra + MCP + PostgreSQL)

```
┌─────────────────────────────────────────────────────┐
│                   CLIENTE / CANAL                   │
│     (App Mobile, Internet Banking, Chatbot Web)      │
└──────────────────────┬──────────────────────────────┘
                       │ HTTPS
┌──────────────────────▼──────────────────────────────┐
│              API GATEWAY / BFF                       │
│   Autenticação, Rate Limiting, TLS Termination       │
└──────────────────────┬──────────────────────────────┘
                       │ JWT com user_id + account_ids
┌──────────────────────▼──────────────────────────────┐
│            AEGRA (Agent Protocol Server)             │
│   FastAPI + Agent Protocol + SSE Streaming           │
│   Threads / Runs / Checkpoints → PostgreSQL          │
└──────────────────────┬──────────────────────────────┘
                       │ executa
┌──────────────────────▼──────────────────────────────┐
│          LANGGRAPH AGENT (StateGraph)                │
│   ReAct loop, Human-in-the-Loop, State Persistence  │
└──────────┬───────────────────────────────┬──────────┘
           │ MCP                           │ MCP
┌──────────▼──────────┐       ┌───────────▼──────────┐
│  MCP Server Contas  │       │  MCP Server Pagtos   │
│  (read-only user)   │       │  (write user + authz) │
└──────────┬──────────┘       └───────────┬──────────┘
           │                              │
┌──────────▼──────────────────────────────▼──────────┐
│              PostgreSQL (dados bancários)            │
│   Row-Level Security | pgcrypto | pgvector           │
│   Audit Log | Particionamento | Retenção 5 anos      │
└────────────────────────────────────────────────────┘
```

---

## 3. Implementação do Agente Bancário com LangGraph

### 3.1 Definição do Estado

```python
from typing import TypedDict, Annotated
from langgraph.graph import StateGraph, END
import operator

class BankingAgentState(TypedDict):
    messages: Annotated[list, operator.add]
    user_id: str
    account_ids: list[str]          # contas autorizadas para este usuário
    pix_daily_limit: float          # limite carregado no início da sessão
    pending_confirmation: dict | None  # ação aguardando aprovação humana
```

### 3.2 Nós do Grafo

```python
from langgraph.graph import StateGraph, END
from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver

def route_after_llm(state: BankingAgentState):
    last = state["messages"][-1]
    if not last.tool_calls:
        return END
    # Tools de escrita → checkpoint antes de executar (human-in-the-loop)
    write_tools = {"initiate_pix_payment", "schedule_transfer", "update_contact"}
    if any(tc["name"] in write_tools for tc in last.tool_calls):
        return "await_confirmation"
    return "tools"

graph = StateGraph(BankingAgentState)
graph.add_node("agent", call_llm_node)
graph.add_node("tools", call_tools_node)
graph.add_node("await_confirmation", await_confirmation_node)
graph.set_entry_point("agent")
graph.add_conditional_edges("agent", route_after_llm)
graph.add_edge("tools", "agent")
graph.add_edge("await_confirmation", "agent")

# Checkpointer via PostgreSQL (Aegra gerencia esta conexão)
checkpointer = AsyncPostgresSaver.from_conn_string(os.environ["DATABASE_URL"])
app = graph.compile(checkpointer=checkpointer)
```

### 3.3 Conectando MCP Tools ao LangGraph

```python
from langchain_mcp_adapters.client import MultiServerMCPClient
from langgraph.prebuilt import create_react_agent

async def create_banking_agent():
    mcp_client = MultiServerMCPClient({
        "contas": {
            "url": "http://mcp-contas:8001/sse",
            "transport": "sse",
            "headers": {"X-Internal-Secret": os.environ["MCP_INTERNAL_SECRET"]}
        },
        "pagamentos": {
            "url": "http://mcp-pagamentos:8002/sse",
            "transport": "sse",
            "headers": {"X-Internal-Secret": os.environ["MCP_INTERNAL_SECRET"]}
        },
    })
    tools = await mcp_client.get_tools()
    return mcp_client, tools
```

---

## 4. Configuração do PostgreSQL para Dados Bancários

### 4.1 Usuários com Privilégios Mínimos

```sql
-- Usuário somente-leitura (para tools de consulta)
CREATE USER mcp_readonly WITH PASSWORD 'senha_forte_aqui';
GRANT CONNECT ON DATABASE banking TO mcp_readonly;
GRANT USAGE ON SCHEMA public TO mcp_readonly;
GRANT SELECT ON accounts, transactions, clients TO mcp_readonly;

-- Usuário para operações (com permissões limitadas)
CREATE USER mcp_writer WITH PASSWORD 'outra_senha_forte';
GRANT CONNECT ON DATABASE banking TO mcp_writer;
GRANT USAGE ON SCHEMA public TO mcp_writer;
GRANT INSERT ON payment_orders TO mcp_writer;
GRANT UPDATE (status) ON payment_orders TO mcp_writer;

-- Usuário do Aegra (para checkpoints e threads do LangGraph)
CREATE USER aegra WITH PASSWORD 'aegra_senha_forte';
CREATE DATABASE aegra OWNER aegra;
-- Aegra cria suas próprias tabelas de estado
```

### 4.2 Row-Level Security (RLS) — Isolamento por Tenant

```sql
ALTER TABLE accounts ENABLE ROW LEVEL SECURITY;
ALTER TABLE transactions ENABLE ROW LEVEL SECURITY;

CREATE POLICY account_isolation ON accounts
  FOR SELECT
  USING (owner_id = current_setting('app.current_user_id')::uuid);

CREATE POLICY transaction_isolation ON transactions
  FOR SELECT
  USING (
    account_id IN (
      SELECT id FROM accounts
      WHERE owner_id = current_setting('app.current_user_id')::uuid
    )
  );

CREATE OR REPLACE FUNCTION set_user_context(user_id uuid)
RETURNS void AS $$
  SELECT set_config('app.current_user_id', user_id::text, true);
$$ LANGUAGE sql SECURITY DEFINER;
```

```python
# No servidor MCP Python — sempre antes de qualquer query
async def execute_query(conn, user_id: str, query: str, params: list):
    async with conn.transaction(readonly=True):
        await conn.execute("SELECT set_user_context($1)", user_id)
        return await conn.fetch(query, *params)
```

### 4.3 Criptografia de Dados Sensíveis

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;
CREATE EXTENSION IF NOT EXISTS vector;  -- pgvector para memória semântica (Aegra)

CREATE TABLE clients (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  cpf_hash    TEXT NOT NULL,
  cpf_enc     BYTEA NOT NULL,
  full_name   TEXT NOT NULL,
  created_at  TIMESTAMPTZ DEFAULT NOW()
);

INSERT INTO clients (cpf_hash, cpf_enc, full_name)
VALUES (
  digest('12345678900', 'sha256'),
  pgp_sym_encrypt('12345678900', current_setting('app.encryption_key')),
  'João Silva'
);
```

### 4.4 Tabela de Auditoria

```sql
CREATE TABLE audit_log (
  id            BIGSERIAL PRIMARY KEY,
  event_time    TIMESTAMPTZ DEFAULT NOW(),
  user_id       UUID NOT NULL,
  agent_session TEXT,   -- thread_id do LangGraph/Aegra
  run_id        TEXT,   -- run_id do Aegra
  tool_name     TEXT NOT NULL,
  params_hash   TEXT,
  resource_type TEXT,
  resource_id   UUID,
  result        TEXT CHECK (result IN ('success', 'error', 'denied')),
  duration_ms   INTEGER,
  ip_address    INET,
  error_message TEXT
);

CREATE INDEX idx_audit_user ON audit_log (user_id, event_time DESC);
CREATE INDEX idx_audit_tool ON audit_log (tool_name, event_time DESC);
CREATE INDEX idx_audit_session ON audit_log (agent_session, event_time DESC);
```

---

## 5. Tools Bancárias Essenciais

### 5.1 Consulta de Saldo

```python
@app.call_tool()
async def call_tool(name: str, args: dict, context: UserContext):
    if name == "get_account_balance":
        account_id = validate_uuid(args["account_id"])

        async with pool.acquire() as conn:
            await conn.execute("SELECT set_user_context($1)", context.user_id)
            row = await conn.fetchrow("""
                SELECT
                    a.id,
                    a.account_number,
                    a.balance,
                    a.available_balance,
                    a.currency,
                    a.updated_at
                FROM accounts a
                WHERE a.id = $1
                  AND a.status = 'ACTIVE'
            """, account_id)

        if not row:
            raise PermissionError("Conta não encontrada ou acesso negado")

        return {
            "account_id": str(row["id"]),
            "account_last4": row["account_number"][-4:],
            "balance": float(row["balance"]),
            "available_balance": float(row["available_balance"]),
            "currency": row["currency"],
            "as_of": row["updated_at"].isoformat()
        }
```

### 5.2 Extrato de Transações

```python
    if name == "get_transactions":
        account_id = validate_uuid(args["account_id"])
        start_date = validate_date(args["start_date"])
        end_date   = validate_date(args["end_date"])
        limit      = min(int(args.get("limit", 50)), 500)

        if (end_date - start_date).days > 90:
            raise ValueError("Período máximo de consulta é 90 dias")

        async with pool.acquire() as conn:
            await conn.execute("SELECT set_user_context($1)", context.user_id)
            rows = await conn.fetch("""
                SELECT t.id, t.type, t.amount, t.currency,
                       t.description, t.counterpart_name, t.created_at
                FROM transactions t
                WHERE t.account_id = $1
                  AND t.created_at BETWEEN $2 AND $3
                ORDER BY t.created_at DESC
                LIMIT $4
            """, account_id, start_date, end_date, limit)

        return {"transactions": [dict(r) for r in rows], "count": len(rows)}
```

### 5.3 Iniciação de PIX (com Human-in-the-Loop via LangGraph)

O LangGraph interrompe o grafo antes de executar a tool, permitindo aprovação humana:

```python
    if name == "initiate_pix_payment":
        amount = Decimal(str(args["amount"]))
        if amount <= 0 or amount > Decimal("999999.99"):
            raise ValueError("Valor inválido")

        daily_used = await get_daily_pix_total(context.user_id)
        if daily_used + amount > context.pix_daily_limit:
            raise ValueError("Limite diário de PIX excedido")

        # Valores altos: retorna pendente para o nó await_confirmation do LangGraph
        if amount > Decimal("1000.00"):
            return {
                "requires_confirmation": True,
                "confirmation_token": generate_token(),
                "summary": {
                    "amount": str(amount),
                    "recipient": args["recipient_key"],
                    "description": args.get("description", "")
                },
                "message": "Transferência acima de R$1.000 requer confirmação."
            }

        order_id = await create_payment_order(
            user_id=context.user_id,
            amount=amount,
            recipient=args["recipient_key"],
            description=args.get("description", "")
        )
        return {"order_id": order_id, "status": "PENDING_EXECUTION"}
```

---

## 6. Compliance: LGPD e Regulação Bancária

### 6.1 Princípios da LGPD Aplicados a Agentes

| Princípio LGPD | Implementação no Agente |
|---------------|------------------------|
| **Finalidade** | Tools com escopo declarado e limitado |
| **Necessidade** | Retornar apenas campos necessários (evitar `SELECT *`) |
| **Transparência** | Log de todos os acessos acessível ao titular |
| **Segurança** | TLS, criptografia em repouso, RLS |
| **Não discriminação** | Auditar decisões automatizadas |
| **Responsabilização** | Trilha de auditoria completa (audit_log + traces Aegra) |

### 6.2 Mascaramento de Dados Sensíveis

```python
def mask_sensitive_data(data: dict) -> dict:
    masked = data.copy()
    if "cpf" in masked:
        cpf = masked["cpf"]
        masked["cpf"] = f"***.***.{cpf[-6:-2]}-**"
    if "account_number" in masked:
        acc = masked["account_number"]
        masked["account_number"] = f"****{acc[-4:]}"
    if "balance" in masked:
        masked["balance"] = "[REDACTED]"
    return masked
```

### 6.3 Direitos do Titular (Art. 18 LGPD)

```python
async def export_user_data(user_id: str) -> dict:
    """Gera export completo dos dados do usuário"""
    ...

async def request_data_deletion(user_id: str) -> dict:
    """Inicia fluxo de exclusão / anonimização"""
    ...
```

### 6.4 Retenção de Dados (Compliance BACEN — 5 anos)

```sql
CREATE TABLE transactions (
  id          UUID NOT NULL,
  created_at  TIMESTAMPTZ NOT NULL
  -- demais colunas...
) PARTITION BY RANGE (created_at);

CREATE TABLE transactions_2025 PARTITION OF transactions
  FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');

-- Job de archiving: mover dados > 5 anos para cold storage
```

---

## 7. Segurança: 7 Camadas de Proteção

```
┌─────────────────────────────────────────────┐
│  Camada 1: Autenticação (JWT / OAuth 2.0)   │
│  Camada 2: Autorização (RBAC + ownership)   │
│  Camada 3: Row-Level Security (PostgreSQL)  │
│  Camada 4: Validação de entrada (whitelist) │
│  Camada 5: Rate Limiting (por usuário)      │
│  Camada 6: Auditoria (imutável)             │
│  Camada 7: Monitoramento de anomalias       │
└─────────────────────────────────────────────┘
```

### Operações que sempre requerem aprovação humana

- Transferências acima do limite configurado (nó `await_confirmation` no LangGraph)
- Alteração de dados cadastrais
- Adição de novo destinatário PIX
- Solicitação de crédito
- Qualquer operação DDL no banco
- Bulk deletes ou updates em dados de clientes

---

## 8. Observabilidade com Aegra

Aegra emite traces OpenTelemetry nativamente. Cada run do agente gera spans para:
- Tempo total de execução do grafo
- Chamadas ao LLM (tokens, latência)
- Tool calls (nome, params hash, duração, resultado)
- Checkpoints salvos

```python
# Configuração de exportador OTEL para Langfuse
import os
os.environ["OTEL_EXPORTER_OTLP_ENDPOINT"] = "https://cloud.langfuse.com/api/public/otel"
os.environ["OTEL_EXPORTER_OTLP_HEADERS"] = f"Authorization=Basic {langfuse_b64_key}"
```

---

## 9. Detecção de Anomalias

```sql
CREATE VIEW suspicious_agent_activity AS
SELECT
  user_id,
  COUNT(*) as call_count,
  COUNT(DISTINCT resource_id) as distinct_resources,
  COUNT(DISTINCT agent_session) as distinct_sessions,
  MIN(event_time) as first_event,
  MAX(event_time) as last_event
FROM audit_log
WHERE event_time > NOW() - INTERVAL '1 hour'
GROUP BY user_id
HAVING
  COUNT(*) > 200
  OR COUNT(DISTINCT resource_id) > 50
  OR COUNT(DISTINCT agent_session) > 10
ORDER BY call_count DESC;
```

---

## 10. Exemplo de Fluxo Completo

```
Usuário: "Qual foi meu maior gasto no mês passado?"

1. API Gateway valida JWT → extrai user_id=abc123, accounts=["conta-1", "conta-2"]
2. Cliente chama Aegra: POST /agents/banking-assistant/runs
   { input: { messages: [...], user_id: "abc123", account_ids: ["conta-1", "conta-2"] } }
3. Aegra recupera checkpoint do thread no PostgreSQL (conversa anterior)
4. LangGraph executa o nó "agent" → LLM decide chamar get_transactions
5. LangGraph executa o nó "tools" → chama MCP Server Contas via SSE
6. MCP Server:
   a. Valida que "conta-1" pertence ao user_id=abc123
   b. SET app.current_user_id = 'abc123' (RLS)
   c. Query parametrizada ao PostgreSQL
   d. Registra na audit_log com thread_id e run_id do Aegra
7. Resultado retorna ao LangGraph → LLM analisa → resposta gerada
8. Aegra salva novo checkpoint no PostgreSQL
9. Resposta streamada via SSE ao cliente:
   "Seu maior gasto em julho foi R$1.234,56 na categoria Alimentação"
```

---

## 11. Stack Recomendada

| Componente | Recomendação |
|------------|-------------|
| **LLM** | Claude (Anthropic), GPT-4o (OpenAI) |
| **Orquestrador** | LangGraph (StateGraph com ciclos e checkpoints) |
| **Plataforma de Deploy** | Aegra (self-hosted, open-source, Agent Protocol) |
| **Protocolo de Tools** | MCP (HTTP+SSE em produção) |
| **Banco (dados bancários)** | PostgreSQL 16 com pgcrypto, pg_partman, RLS |
| **Banco (estado Aegra)** | PostgreSQL 16 com pgvector |
| **Pool de conexões** | pgBouncer (produção), asyncpg (Python) |
| **Workers** | Redis (execução assíncrona Aegra) |
| **Observabilidade** | OpenTelemetry → Langfuse / Jaeger / Phoenix |
| **Segredos** | HashiCorp Vault, AWS Secrets Manager |
| **Autenticação** | OAuth 2.0 + PKCE, JWT com rotação |

---

## 12. Checklist de Go-Live

- [ ] Pen test realizado no servidor MCP
- [ ] SQL injection testado e mitigado
- [ ] RLS testado com usuários de diferentes tenants
- [ ] Auditoria validada: todos os acessos registrados (com thread_id e run_id Aegra)
- [ ] LGPD: DPO revisou flows de dados
- [ ] BACEN: retenção de 5 anos implementada
- [ ] Human-in-the-loop testado para operações de escrita críticas
- [ ] Rate limiting testado sob carga
- [ ] Plano de resposta a incidentes definido
- [ ] Alertas de anomalia configurados
- [ ] Aegra: PostgreSQL + Redis em produção com backup
- [ ] Checkpoints LangGraph: recovery testado após falha
- [ ] Documentação de tools revisada pelo time jurídico

---

## Referências

- [Agentic Banking Directory 2026](https://www.openbankingtracker.com/agentic-banking-and-mcp)
- [Safe AI Agent Access to Production Databases (Datapace)](https://datapace.ai/blog/safe-ai-agent-access-production-database)
- [AI Agent Compliance in Banking](https://www.prompthalo.ai/feeds/blog/ai-agent-compliance-banking)
- [Best Secure AI Agents for Banking 2026](https://gradient-labs.ai/guides/new-best-secure-ai-agents-for-banking-in-2026)
- [AI Agent GDPR Compliance 2026 (Technova)](https://technovapartners.com/en/insights/security-gdpr-enterprise-ai-agents)
- [MCP and Payments: 2026 Guide (Eco)](https://eco.com/support/en/articles/14845480-mcp-and-payments-a-2026-guide)
- [postgres-mcp Pro](https://github.com/crystaldba/postgres-mcp)
- [Aegra - Open Source LangGraph Platform Alternative](https://www.aegra.dev/)
- [Aegra GitHub](https://github.com/aegra/aegra)
- [Aegra: The Self-Hosted AI Agent Backend (Bright Coding)](https://blog.brightcoding.dev/2026/05/31/aegra-the-revolutionary-self-hosted-ai-agent-backend)
- [Simplify AI agent deployments with Aegra (ADEO Tech Blog)](https://medium.com/adeo-tech/simplify-your-ai-agent-deployments-a-quick-look-at-aegra-d4c301c2dd59)
