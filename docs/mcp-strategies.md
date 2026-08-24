# Estratégias para Criar MCP Servers

> Guia prático de arquitetura e implementação de servidores MCP (Model Context Protocol) prontos para produção.

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

## 2. Primitivos do MCP

| Primitivo | Direção | Descrição |
|-----------|---------|----------|
| **Tools** | Modelo → Servidor | O modelo chama funções (efeitos colaterais permitidos) |
| **Resources** | Servidor → Modelo | Dados somente-leitura (contexto, documentos) |
| **Prompts** | Servidor → Modelo | Templates de prompt reutilizáveis |
| **Sampling** | Servidor → Modelo | Servidor pede ao modelo que gere texto |

---

## 3. Modos de Transporte

### 3.1 stdio (Local)

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

### 3.2 HTTP + SSE (Remoto)

```
Cliente MCP → POST /messages  → Servidor HTTP
Servidor    → SSE stream     → Cliente (notificações assíncronas)
```

**Uso:** produção, múltiplos clientes simultâneos, SaaS.

---

## 4. Estratégias de Arquitetura

### 4.1 MCP por Domínio (Recomendado)

Crie um servidor MCP por domínio de negócio, não por banco de dados:

```
mcp-contas/      → tools: get_balance, list_transactions, get_statement
mcp-pagamentos/  → tools: initiate_pix, schedule_payment, get_payment_status
mcp-clientes/    → tools: get_client_profile, update_contact, get_limits
mcp-compliance/  → tools: check_kyc, run_aml_check, get_risk_score
```

**Vantagem:** cada servidor tem escopo mínimo de permissões, falha isolada, auditoria por domínio.

### 4.2 MCP Gateway (Fachada Única)

Um servidor MCP único que roteia para microserviços internos:

```
[Agente] → [MCP Gateway]
                ├── /contas    → serviço de contas
                ├── /pix       → serviço de pagamentos
                └── /clientes  → CRM interno
```

**Vantagem:** um único ponto de controle de acesso, rate limiting centralizado.

### 4.3 MCP Read-Only vs Read-Write

Separe explicitamente servidores de leitura e escrita:

```
mcp-banking-read   → apenas SELECT, sem efeitos colaterais
mcp-banking-write  → INSERT/UPDATE, requer autenticação adicional
```

---

## 5. Estrutura de um Servidor MCP (TypeScript)

```typescript
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { CallToolRequestSchema, ListToolsRequestSchema } from "@modelcontextprotocol/sdk/types.js";

const server = new Server(
  { name: "banking-mcp", version: "1.0.0" },
  { capabilities: { tools: {} } }
);

// Declaração de ferramentas disponíveis
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

// Execução das ferramentas
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
  // Validação, autorização e query ao banco
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

## 6. Estrutura de um Servidor MCP (Python)

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

## 7. Estratégias de Segurança no MCP

### 7.1 Autenticação por Contexto

O MCP não define autenticação nativamente — implemente-a no servidor:

```typescript
// Middleware de autenticação para HTTP transport
app.use("/mcp", async (req, res, next) => {
  const token = req.headers["x-api-key"];
  const session = await validateApiKey(token);
  if (!session) return res.status(401).json({ error: "Unauthorized" });
  req.session = session; // attach user context
  next();
});
```

### 7.2 Autorização por Recurso

Sempre valide que o usuário autenticado tem acesso ao recurso pedido:

```typescript
async function getAccountBalance(accountId: string, userId: string) {
  // Verifica ownership antes da query principal
  const owned = await db.query(
    "SELECT 1 FROM accounts WHERE id = $1 AND owner_id = $2",
    [accountId, userId]
  );
  if (!owned.rows.length) throw new Error("Acesso negado");
  // ... resto da lógica
}
```

### 7.3 Prevenção de SQL Injection

- **Sempre** use queries parametrizadas (`$1`, `$2`, não concatenação de strings)
- Nunca permita que o modelo construa SQL diretamente
- Whitelist de operações: o servidor define quais queries são possíveis

### 7.4 Rate Limiting

```typescript
import { RateLimiterMemory } from "rate-limiter-flexible";

const limiter = new RateLimiterMemory({
  points: 100,   // máximo de chamadas
  duration: 60,  // por minuto
});

// Antes de executar qualquer tool
await limiter.consume(userId); // lança exceção se limite excedido
```

---

## 8. Vulnerabilidade Comum: SQL Injection via MCP

A vulnerabilidade mais crítica em MCPs de banco de dados é o bypass de modo read-only:

```sql
-- Ataque: payload injetado no parâmetro
COMMIT; DROP TABLE accounts; --
```

**Mitigações obrigatórias:**
1. Usuário de banco com permissão `SELECT` apenas
2. Queries 100% parametrizadas (sem template strings com input do usuário)
3. Validação de tipo rigorosa nos parâmetros de entrada
4. Transações em modo `READ ONLY` no PostgreSQL:

```sql
BEGIN READ ONLY;
SELECT balance FROM accounts WHERE id = $1;
COMMIT;
```

---

## 9. Auditoria e Logging

Cada chamada de tool deve gerar um registro de auditoria:

```typescript
async function auditLog(event: {
  tool: string;
  userId: string;
  params: Record<string, unknown>;  // nunca logar dados sensíveis
  result: "success" | "error";
  duration_ms: number;
  ip?: string;
}) {
  await db.query(
    `INSERT INTO audit_log (tool, user_id, params_hash, result, duration_ms, created_at)
     VALUES ($1, $2, $3, $4, $5, NOW())`,
    [event.tool, event.userId, hash(event.params), event.result, event.duration_ms]
  );
}
```

**Nunca logar:** senhas, tokens, saldos, CPF completo, número de conta completo.

---

## 10. Checklist de Produção

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
- [ ] Documentação de todas as tools e parâmetros

---

## Referências

- [Complete Guide to MCP in 2026 (DEV Community)](https://dev.to/x4nent/complete-guide-to-mcp-model-context-protocol-in-2026-architecture-implementation-and-4a11)
- [PostgreSQL MCP Server Guide (selfhost.dev)](https://selfhost.dev/blog/postgresql-mcp-server-guide/)
- [postgres-mcp Pro (crystaldba/postgres-mcp)](https://github.com/crystaldba/postgres-mcp)
- [MCP Security Checklist 2026](https://www.networkintelligence.ai/blogs/model-context-protocol-mcp-security-checklist/)
- [Postgres MCP Server Tools & Governance (Averta)](https://averta.io/mcp/postgres)
