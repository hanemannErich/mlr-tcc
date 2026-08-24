# Agentes com Acesso a Dados Bancários no PostgreSQL

> Arquitetura de referência para agentes de IA que acessam dados financeiros sensíveis, com foco em segurança, compliance e implementação prática com PostgreSQL.

---

## 1. Contexto e Desafios

Agentes de IA que operam sobre dados bancários enfrentam requisitos únicos:

- **Dados altamente sensíveis:** saldos, transações, CPF, endereços
- **Regulação estrita:** LGPD, BACEN, CMN, Open Finance
- **Auditabilidade:** toda ação deve ser rastreável
- **Baixa tolerância a erros:** falhas têm impacto financeiro direto
- **Multi-tenancy:** cada cliente só vê seus próprios dados

---

## 2. Arquitetura de Referência

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
│              CAMADA DE AGENTE (LLM)                  │
│   Claude / GPT + Orquestrador (ReAct / Plan-Exec)   │
└──────────┬───────────────────────────────┬──────────┘
           │ MCP                           │ MCP
┌──────────▼──────────┐       ┌───────────▼──────────┐
│  MCP Server Contas  │       │  MCP Server Pagtos   │
│  (read-only user)   │       │  (write user + authz) │
└──────────┬──────────┘       └───────────┬──────────┘
           │                              │
┌──────────▼──────────────────────────────▼──────────┐
│              PostgreSQL (dados bancários)            │
│   Row-Level Security | Encryption | Audit Log        │
└────────────────────────────────────────────────────┘
```

---

## 3. Configuração do PostgreSQL para Dados Bancários

### 3.1 Usuários com Privilégios Mínimos

```sql
-- Usuário somente-leitura (para tools de consulta)
CREATE USER mcp_readonly WITH PASSWORD 'senha_forte_aqui';
GRANT CONNECT ON DATABASE banking TO mcp_readonly;
GRANT USAGE ON SCHEMA public TO mcp_readonly;
GRANT SELECT ON accounts, transactions, clients TO mcp_readonly;
-- Não conceder INSERT, UPDATE, DELETE, DDL

-- Usuário para operações (com permissões limitadas)
CREATE USER mcp_writer WITH PASSWORD 'outra_senha_forte';
GRANT CONNECT ON DATABASE banking TO mcp_writer;
GRANT USAGE ON SCHEMA public TO mcp_writer;
GRANT INSERT ON payment_orders TO mcp_writer;
GRANT UPDATE (status) ON payment_orders TO mcp_writer;
-- Apenas as colunas e tabelas necessárias
```

### 3.2 Row-Level Security (RLS) — Isolamento por Tenant

O RLS garante que mesmo se um bug no código enviar o `account_id` errado, o banco de dados bloqueia o acesso.

```sql
-- Habilitar RLS nas tabelas sensíveis
ALTER TABLE accounts ENABLE ROW LEVEL SECURITY;
ALTER TABLE transactions ENABLE ROW LEVEL SECURITY;

-- Política: usuário só vê contas que pertencem a ele
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

-- No servidor MCP, antes de qualquer query:
CREATE OR REPLACE FUNCTION set_user_context(user_id uuid)
RETURNS void AS $$
  SELECT set_config('app.current_user_id', user_id::text, true);
$$ LANGUAGE sql SECURITY DEFINER;
```

```python
# No servidor MCP Python
async def execute_query(conn, user_id: str, query: str, params: list):
    async with conn.transaction(readonly=True):
        await conn.execute("SELECT set_user_context($1)", user_id)
        return await conn.fetch(query, *params)
```

### 3.3 Criptografia de Dados Sensíveis

```sql
-- Instalar extensão pgcrypto
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- Exemplo: CPF criptografado em repouso
CREATE TABLE clients (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  cpf_hash    TEXT NOT NULL,          -- hash para busca
  cpf_enc     BYTEA NOT NULL,         -- valor criptografado
  full_name   TEXT NOT NULL,
  created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- Inserir com criptografia
INSERT INTO clients (cpf_hash, cpf_enc, full_name)
VALUES (
  digest('12345678900', 'sha256'),
  pgp_sym_encrypt('12345678900', current_setting('app.encryption_key')),
  'João Silva'
);

-- Buscar por CPF (usando hash)
SELECT id, full_name
FROM clients
WHERE cpf_hash = digest($1, 'sha256');
```

### 3.4 Tabela de Auditoria

```sql
CREATE TABLE audit_log (
  id            BIGSERIAL PRIMARY KEY,
  event_time    TIMESTAMPTZ DEFAULT NOW(),
  user_id       UUID NOT NULL,
  agent_session TEXT,
  tool_name     TEXT NOT NULL,
  params_hash   TEXT,                  -- hash dos params, nunca o valor
  resource_type TEXT,
  resource_id   UUID,
  result        TEXT CHECK (result IN ('success', 'error', 'denied')),
  duration_ms   INTEGER,
  ip_address    INET,
  error_message TEXT
);

-- Índices para consultas de auditoria
CREATE INDEX idx_audit_user ON audit_log (user_id, event_time DESC);
CREATE INDEX idx_audit_tool ON audit_log (tool_name, event_time DESC);

-- Política de retenção: manter por 5 anos (BACEN)
-- Implementar via pg_partman ou job de archiving
```

---

## 4. Tools Bancárias Essenciais

### 4.1 Consulta de Saldo

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

        # Nunca retornar número de conta completo
        return {
            "account_id": str(row["id"]),
            "account_last4": row["account_number"][-4:],
            "balance": float(row["balance"]),
            "available_balance": float(row["available_balance"]),
            "currency": row["currency"],
            "as_of": row["updated_at"].isoformat()
        }
```

### 4.2 Extrato de Transações

```python
    if name == "get_transactions":
        account_id = validate_uuid(args["account_id"])
        start_date = validate_date(args["start_date"])
        end_date   = validate_date(args["end_date"])
        limit      = min(int(args.get("limit", 50)), 500)  # cap no servidor

        if (end_date - start_date).days > 90:
            raise ValueError("Período máximo de consulta é 90 dias")

        async with pool.acquire() as conn:
            await conn.execute("SELECT set_user_context($1)", context.user_id)

            rows = await conn.fetch("""
                SELECT
                    t.id,
                    t.type,
                    t.amount,
                    t.currency,
                    t.description,
                    t.counterpart_name,
                    t.created_at
                FROM transactions t
                WHERE t.account_id = $1
                  AND t.created_at BETWEEN $2 AND $3
                ORDER BY t.created_at DESC
                LIMIT $4
            """, account_id, start_date, end_date, limit)

        return {"transactions": [dict(r) for r in rows], "count": len(rows)}
```

### 4.3 Iniciação de PIX (com Human-in-the-Loop)

```python
    if name == "initiate_pix_payment":
        # Validações rigorosas
        amount = Decimal(str(args["amount"]))
        if amount <= 0 or amount > Decimal("999999.99"):
            raise ValueError("Valor inválido")

        # Verificar limite do usuário
        daily_used = await get_daily_pix_total(context.user_id)
        if daily_used + amount > context.pix_daily_limit:
            raise ValueError(f"Limite diário de PIX excedido")

        # Para valores acima do threshold: exigir confirmação
        if amount > Decimal("1000.00"):
            return {
                "requires_confirmation": True,
                "confirmation_token": generate_token(),
                "summary": {
                    "amount": str(amount),
                    "recipient": args["recipient_key"],
                    "description": args.get("description", "")
                },
                "message": "Transferência acima de R$1.000 requer confirmação do usuário."
            }

        # Criar ordem de pagamento (não executa ainda)
        order_id = await create_payment_order(
            user_id=context.user_id,
            amount=amount,
            recipient=args["recipient_key"],
            description=args.get("description", "")
        )
        return {"order_id": order_id, "status": "PENDING_EXECUTION"}
```

---

## 5. Compliance: LGPD e Regulação Bancária

### 5.1 Princípios da LGPD Aplicados a Agentes

| Princípio LGPD | Implementação no Agente |
|---------------|------------------------|
| **Finalidade** | Tools com escopo declarado e limitado |
| **Necessidade** | Retornar apenas campos necessários (evitar `SELECT *`) |
| **Transparência** | Log de todos os acessos acessível ao titular |
| **Segurança** | TLS, criptografia em repouso, RLS |
| **Não discriminação** | Auditar decisões automatizadas |
| **Responsabilização** | Trilha de auditoria completa |

### 5.2 Mascaramento de Dados Sensíveis

```python
def mask_sensitive_data(data: dict) -> dict:
    """Aplicar antes de qualquer log ou exibição"""
    masked = data.copy()
    if "cpf" in masked:
        cpf = masked["cpf"]
        masked["cpf"] = f"***.***.{cpf[-6:-2]}-**"
    if "account_number" in masked:
        acc = masked["account_number"]
        masked["account_number"] = f"****{acc[-4:]}"
    if "balance" in masked:
        # Em logs, nunca o valor exato
        masked["balance"] = "[REDACTED]"
    return masked
```

### 5.3 Direitos do Titular

O agente deve suportar as seguintes operações para atender à LGPD:

```python
# Tool: exportar dados do titular (Art. 18 LGPD)
async def export_user_data(user_id: str) -> dict:
    """Gera export completo dos dados do usuário"""
    ...

# Tool: solicitar exclusão (Art. 18 LGPD)
async def request_data_deletion(user_id: str) -> dict:
    """Inicia fluxo de exclusão / anonimização"""
    ...
```

### 5.4 Retenção de Dados (Compliance BACEN)

```sql
-- Particionamento por data para facilitar retenção
CREATE TABLE transactions (
  id          UUID NOT NULL,
  created_at  TIMESTAMPTZ NOT NULL,
  -- demais colunas...
) PARTITION BY RANGE (created_at);

-- Partição por ano
CREATE TABLE transactions_2024 PARTITION OF transactions
  FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

CREATE TABLE transactions_2025 PARTITION OF transactions
  FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');

-- Job de archiving: mover dados > 5 anos para cold storage
-- Exigência BACEN: manter registros por no mínimo 5 anos
```

---

## 6. Segurança: Camadas de Proteção

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

- Transferências acima do limite configurado
- Alteração de dados cadastrais (endereço, telefone)
- Adição de novo destinatário PIX
- Solicitação de crédito
- Qualquer operação DDL no banco
- Bulk deletes ou updates em dados de clientes

---

## 7. Monitoramento e Detecção de Anomalias

```sql
-- View para detectar comportamentos suspeitos
CREATE VIEW suspicious_agent_activity AS
SELECT
  user_id,
  COUNT(*) as call_count,
  COUNT(DISTINCT resource_id) as distinct_resources,
  MIN(event_time) as first_event,
  MAX(event_time) as last_event
FROM audit_log
WHERE event_time > NOW() - INTERVAL '1 hour'
GROUP BY user_id
HAVING
  COUNT(*) > 200                    -- mais de 200 chamadas/hora
  OR COUNT(DISTINCT resource_id) > 50 -- acessando muitas contas distintas
ORDER BY call_count DESC;
```

---

## 8. Exemplo de Fluxo Completo

```
Usuário: "Qual foi meu maior gasto no mês passado?"

1. API Gateway valida JWT → extrai user_id=abc123, accounts=["conta-1", "conta-2"]
2. Agente recebe contexto com user_id e lista de contas autorizadas
3. Agente chama: get_transactions({ account_id: "conta-1", start_date: "2026-07-01", end_date: "2026-07-31" })
4. MCP Server:
   a. Valida que "conta-1" está na lista autorizada do usuário
   b. Executa SET app.current_user_id = 'abc123' (RLS)
   c. Query parametrizada ao PostgreSQL
   d. Registra na audit_log
5. Agente recebe transações, analisa e responde:
   "Seu maior gasto em julho foi R$1.234,56 na categoria Alimentação"
```

---

## 9. Stack Recomendada

| Componente | Opções |
|------------|--------|
| **LLM** | Claude (Anthropic), GPT-4o (OpenAI) |
| **Protocolo** | MCP (recomendado), OpenAI Function Calling |
| **Runtime MCP** | TypeScript SDK, Python SDK |
| **Banco** | PostgreSQL 15+ com pgcrypto, pg_partman |
| **Pool de conexões** | pgBouncer (produção), asyncpg (Python), pg (Node) |
| **Cache** | Redis (sessões, limites de taxa) |
| **Observabilidade** | OpenTelemetry + Jaeger/Grafana |
| **Segredos** | HashiCorp Vault, AWS Secrets Manager |
| **Autenticação** | OAuth 2.0 + PKCE, JWT com rotação |

---

## 10. Checklist de Go-Live

- [ ] Pen test realizado no servidor MCP
- [ ] Vulnerabilidade de SQL injection testada e mitigada
- [ ] RLS testado com usuários de diferentes tenants
- [ ] Auditoria validada: todos os acessos registrados
- [ ] LGPD: DPO revisou flows de dados
- [ ] BACEN: retenção de 5 anos implementada
- [ ] Human-in-the-loop em todas as operações de escrita críticas
- [ ] Rate limiting testado sob carga
- [ ] Plano de resposta a incidentes definido
- [ ] Alertas de anomalia configurados
- [ ] Documentação de tools revisada pelo time jurídico
- [ ] Treinamento do time de suporte concluído

---

## Referências

- [Agentic Banking Directory 2026](https://www.openbankingtracker.com/agentic-banking-and-mcp)
- [Safe AI Agent Access to Production Databases (Datapace)](https://datapace.ai/blog/safe-ai-agent-access-production-database)
- [AI Agent Compliance in Banking](https://www.prompthalo.ai/feeds/blog/ai-agent-compliance-banking)
- [Best Secure AI Agents for Banking 2026](https://gradient-labs.ai/guides/new-best-secure-ai-agents-for-banking-in-2026)
- [AI Agent GDPR Compliance 2026 (Technova)](https://technovapartners.com/en/insights/security-gdpr-enterprise-ai-agents)
- [MCP and Payments: 2026 Guide (Eco)](https://eco.com/support/en/articles/14845480-mcp-and-payments-a-2026-guide)
- [postgres-mcp Pro](https://github.com/crystaldba/postgres-mcp)
