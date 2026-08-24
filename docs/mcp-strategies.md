# Estratégias para Criar MCP Servers

> Guia prático de arquitetura e implementação de servidores MCP (Model Context Protocol) prontos para produção, incluindo integração com LangGraph e Aegra.

---

## 1. O Que é MCP

O **Model Context Protocol** é um padrão aberto criado pela Anthropic (Nov 2024) que define como LLMs se conectam a ferramentas e dados externos. Funciona como um "USB-C para agentes de IA": um único protocolo que qualquer modelo pode usar para falar com qualquer sistema.

```
[Claude / GPT / Gemini]
         ↕ MCP
[Seu Servidor MCP]
         ↕
[PostgreSQL / APIs / Arquivos / etc.]
```

Adotado pela OpenAI em março de 2025. Em março de 2026, o ecossistema conta com mais de 200 servidores disponíveis.

---

## 2. MCP vs Agent Protocol (Aegra/LangGraph)

Dois protocolos complementares atuam em camadas diferentes:

| Aspecto | MCP | Agent Protocol (Aegra) |
|---------|-----|----------------------|
| **Propósito** | Conectar LLM a ferramentas/dados | Deploy e gerenciamento de agentes |
| **Quem chama** | O LLM (via host como Claude) | O cliente/orquestrador |
| **Estado** | Stateless por chamada | Stateful (threads, runs, checkpoints) |
| **Primitivos** | Tools, Resources, Prompts | Agents, Threads, Runs, Checkpoints |
| **Transporte** | stdio, HTTP+SSE | HTTP REST + SSE |
| **Uso típico** | Expor capacidades (DB, APIs) | Hospedar agentes LangGraph em produção |

**Na prática, usam-se juntos:** o agente LangGraph (hospedado no Aegra) chama tools definidas via MCP para acessar o banco.

```
[Usuário] → [Aegra API] → [Agente LangGraph] → [MCP Server] → [PostgreSQL]
```

---

## 3. Primitivos do MCP

| Primitivo | Direção | Descrição |
|-----------|---------|----------|
| **Tools** | Modelo → Servidor | O modelo chama funções (efeitos colaterais permitidos) |
| **Resources** | Servidor → Modelo | Dados somente-leitura (contexto, documentos) |
| **Prompts** | Servidor → Modelo | Templates de prompt reutilizáveis |
| **Sampling** | Servidor → Modelo | Servidor pede ao modelo que gere texto |

---

## 4. Modos de Transporte

### 4.1 stdio (Local)

```json
// claude_desktop_config.json
{
  "mcpServers": {
    "banking": {
      "command": "node",
      "args": ["/path/to/banking-mcp/index.js"],
      "env": {
        "DATABASE_URL": "postgresql://readonly_user:pass@localhost:5432/banking"
      }
    }
  }
}
```

**Uso:** desenvolvimento local, ferramentas CLI, testes.

### 4.2 HTTP + SSE (Remoto)

```
Cliente MCP → POST /messages  → Servidor HTTP
Servidor    → SSE stream     → Cliente (notificações assíncronas)
```

**Uso:** produção, múltiplos clientes simultâneos, SaaS.

---

## 5. Estratégias de Arquitetura

### 5.1 MCP por Domínio (Recomendado)

Crie um servidor MCP por domínio de negócio, não por banco de dados:

```
mcp-contas/      → tools: get_balance, list_transactions, get_statement
mcp-pagamentos/  → tools: initiate_pix, schedule_payment, get_payment_status
mcp-clientes/    → tools: get_client_profile, update_contact, get_limits
mcp-compliance/  → tools: check_kyc, run_aml_check, get_risk_score
```

**Vantagem:** cada servidor tem escopo mínimo de permissões, falha isolada, auditoria por domínio.

### 5.2 MCP Gateway (Fachada Única)

Um servidor MCP único que roteia para microserviços internos:

```
[Agente] → [MCP Gateway]
                ├── /contas    → serviço de contas
                ├── /pix       → serviço de pagamentos
                └── /clientes  → CRM interno
```

**Vantagem:** um único ponto de controle de acesso, rate limiting centralizado.

### 5.3 MCP Read-Only vs Read-Write

Separe explicitamente servidores de leitura e escrita:

```
mcp-banking-read   → apenas SELECT, sem efeitos colaterais
mcp-banking-write  → INSERT/UPDATE, requer autenticação adicional
```

### 5.4 MCP + LangGraph + Aegra (Stack Completa)

Arquitetura completa para produção self-hosted:

```
[Cliente] → [Aegra] → [LangGraph Agent] → [MCP Server] → [PostgreSQL]
                              ↑
                    [LangGraph Checkpointer]
                              ↑
                    [PostgreSQL via Aegra]
```

O LangGraph usa o MCP server como fonte de tools, enquanto o Aegra provê persistência de estado (threads/checkpoints) via PostgreSQL.

```python
# Integrando MCP tools num agente LangGraph
from langchain_mcp_adapters.client import MultiServerMCPClient
from langgraph.prebuilt import create_react_agent

async def build_banking_agent():
    async with MultiServerMCPClient({
        "contas": {"url": "http://mcp-contas:8001/sse", "transport": "sse"},
        "pagamentos": {"url": "http://mcp-pagamentos:8002/sse", "transport": "sse"},
    }) as mcp_client:
        tools = mcp_client.get_tools()
        agent = create_react_agent(llm, tools, checkpointer=checkpointer)
        return agent
```

---

## 6. Estrutura de um Servidor MCP (TypeScript)

```typescript
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { CallToolRequestSchema, ListToolsRequestSchema } from "@modelcontextprotocol/sdk/types.js";

const server = new Server(
  { name: "banking-mcp", version: "1.0.0" },
  { capabilities: { tools: {} } }
);

server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    {
      name: "get_account_balance",
      description: "Retorna o saldo atual de uma conta. Requer que o usuário autenticado seja titular da conta.",
      inputSchema: {
        type: "object",
        properties: {
          account_id: { type: "string", description: "UUID da conta" }
        },
        required: ["account_id"]
      }
    }
  ]
}));

server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;
  switch (name) {
    case "get_account_balance":
      return await getAccountBalance(args.account_id);
    default:
      throw new Error(`Tool desconhecida: ${name}`);
  }
});

async function getAccountBalance(accountId: string) {
  const result = await db.query(
    "SELECT balance, currency FROM accounts WHERE id = $1 AND active = true",
    [accountId]
  );
  if (!result.rows[0]) throw new Error("Conta não encontrada ou inativa");
  return {
    content: [{ type: "text", text: JSON.stringify(result.rows[0]) }]
  };
}

const transport = new StdioServerTransport();
await server.connect(transport);
```

---

## 7. Estrutura de um Servidor MCP (Python)

```python
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import Tool, TextContent, CallToolResult
import asyncpg

app = Server("banking-mcp")

@app.list_tools()
async def list_tools():
    return [
        Tool(
            name="get_account_balance",
            description="Retorna saldo da conta do usuário autenticado.",
            inputSchema={
                "type": "object",
                "properties": {
                    "account_id": {"type": "string"}
                },
                "required": ["account_id"]
            }
        )
    ]

@app.call_tool()
async def call_tool(name: str, arguments: dict) -> CallToolResult:
    if name == "get_account_balance":
        account_id = arguments["account_id"]
        async with pool.acquire() as conn:
            row = await conn.fetchrow(
                "SELECT balance, currency FROM accounts WHERE id = $1",
                account_id
            )
        if not row:
            raise ValueError("Conta não encontrada")
        return CallToolResult(
            content=[TextContent(type="text", text=str(dict(row)))]
        )
    raise ValueError(f"Tool desconhecida: {name}")

async def main():
    global pool
    pool = await asyncpg.create_pool(dsn=os.environ["DATABASE_URL"])
    async with stdio_server() as (read, write):
        await app.run(read, write, app.create_initialization_options())

if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
```

---

## 8. Deploy com Aegra (Produção Self-Hosted)

### 8.1 Registrando um Agente LangGraph no Aegra

```python
# aegra.config.py
from aegra import AegraConfig
from agents.banking_agent import banking_graph

config = AegraConfig(
    agents=[
        {
            "name": "banking-assistant",
            "graph": banking_graph,
            "description": "Assistente bancário com acesso a contas e pagamentos",
        }
    ]
)
```

### 8.2 Docker Compose completo

```yaml
# docker-compose.yml
services:
  aegra:
    image: aegra/aegra:latest
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgresql://aegra:secret@postgres:5432/aegra
      REDIS_URL: redis://redis:6379
      OPENAI_API_KEY: ${OPENAI_API_KEY}
      ANTHROPIC_API_KEY: ${ANTHROPIC_API_KEY}
    depends_on: [postgres, redis]

  mcp-contas:
    build: ./mcp-contas
    environment:
      DATABASE_URL: postgresql://mcp_readonly:secret@postgres:5432/banking

  mcp-pagamentos:
    build: ./mcp-pagamentos
    environment:
      DATABASE_URL: postgresql://mcp_writer:secret@postgres:5432/banking

  postgres:
    image: pgvector/pgvector:pg16
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: secret

  redis:
    image: redis:7-alpine

volumes:
  pgdata:
```

### 8.3 Chamando o Agente via Agent Protocol

```python
# Cliente usando LangGraph SDK (funciona igual contra LangGraph Platform)
from langgraph_sdk import get_client

client = get_client(url="http://aegra:8000")

# Criar thread (conversa)
thread = await client.threads.create()

# Executar o agente
run = await client.runs.create(
    thread_id=thread["thread_id"],
    assistant_id="banking-assistant",
    input={"messages": [{"role": "user", "content": "Qual meu saldo?"}]},
)

# Streaming de resposta
async for chunk in client.runs.stream(thread["thread_id"], run["run_id"]):
    print(chunk)
```

---

## 9. Estratégias de Segurança no MCP

### 9.1 Autenticação por Contexto

O MCP não define autenticação nativamente — implemente-a no servidor:

```typescript
app.use("/mcp", async (req, res, next) => {
  const token = req.headers["x-api-key"];
  const session = await validateApiKey(token);
  if (!session) return res.status(401).json({ error: "Unauthorized" });
  req.session = session;
  next();
});
```

### 9.2 Autorização por Recurso

```typescript
async function getAccountBalance(accountId: string, userId: string) {
  const owned = await db.query(
    "SELECT 1 FROM accounts WHERE id = $1 AND owner_id = $2",
    [accountId, userId]
  );
  if (!owned.rows.length) throw new Error("Acesso negado");
}
```

### 9.3 Prevenção de SQL Injection

- **Sempre** use queries parametrizadas (`$1`, `$2`)
- Nunca permita que o modelo construa SQL diretamente
- Whitelist de operações no servidor

### 9.4 Rate Limiting

```typescript
const limiter = new RateLimiterMemory({ points: 100, duration: 60 });
await limiter.consume(userId);
```

---

## 10. Vulnerabilidade Comum: SQL Injection via MCP

Bypass de modo read-only:

```sql
COMMIT; DROP TABLE accounts; --
```

**Mitigações obrigatórias:**
1. Usuário de banco com permissão `SELECT` apenas
2. Queries 100% parametrizadas
3. Validação de tipo rigorosa
4. Transações `READ ONLY`:

```sql
BEGIN READ ONLY;
SELECT balance FROM accounts WHERE id = $1;
COMMIT;
```

---

## 11. Auditoria e Logging

```typescript
async function auditLog(event: {
  tool: string;
  userId: string;
  params: Record<string, unknown>;
  result: "success" | "error";
  duration_ms: number;
}) {
  await db.query(
    `INSERT INTO audit_log (tool, user_id, params_hash, result, duration_ms, created_at)
     VALUES ($1, $2, $3, $4, $5, NOW())`,
    [event.tool, event.userId, hash(event.params), event.result, event.duration_ms]
  );
}
```

---

## 12. Checklist de Produção

- [ ] Servidor MCP com escopo mínimo de permissões
- [ ] Autenticação implementada (JWT, API Key, OAuth)
- [ ] Autorização por recurso em cada tool
- [ ] Queries 100% parametrizadas
- [ ] Usuário PostgreSQL read-only para tools de consulta
- [ ] Rate limiting por usuário
- [ ] Auditoria de todas as chamadas
- [ ] Timeout configurado em todas as queries
- [ ] Dados sensíveis mascarados nos logs
- [ ] Health check endpoint
- [ ] Testes de integração para cada tool
- [ ] Aegra rodando com PostgreSQL + Redis em produção
- [ ] Checkpoints LangGraph persistidos no PostgreSQL

---

## Referências

- [Complete Guide to MCP in 2026 (DEV Community)](https://dev.to/x4nent/complete-guide-to-mcp-model-context-protocol-in-2026-architecture-implementation-and-4a11)
- [PostgreSQL MCP Server Guide (selfhost.dev)](https://selfhost.dev/blog/postgresql-mcp-server-guide/)
- [postgres-mcp Pro (crystaldba/postgres-mcp)](https://github.com/crystaldba/postgres-mcp)
- [MCP Security Checklist 2026](https://www.networkintelligence.ai/blogs/model-context-protocol-mcp-security-checklist/)
- [Postgres MCP Server Tools & Governance (Averta)](https://averta.io/mcp/postgres)
- [Aegra - Open Source LangGraph Platform Alternative](https://www.aegra.dev/)
- [Aegra GitHub](https://github.com/aegra/aegra)
- [aegra-api PyPI](https://pypi.org/project/aegra-api/)
- [Simplify AI agent deployments with Aegra (ADEO Tech Blog)](https://medium.com/adeo-tech/simplify-your-ai-agent-deployments-a-quick-look-at-aegra-d4c301c2dd59)
