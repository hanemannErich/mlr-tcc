# Interfaces para Agentes de IA

> Referência de arquitetura para projetar interfaces robustas entre agentes LLM e sistemas externos.

---

## 1. O Que São Interfaces de Agentes

Uma **interface de agente** define o contrato entre um LLM e o mundo externo: quais ações o agente pode executar, que dados pode ler, e como resultados são devolvidos ao modelo para raciocínio subsequente.

Existem três camadas principais:

| Camada | Responsabilidade | Exemplos |
|--------|-----------------|----------|
| **Protocolo** | Serialização e transporte de chamadas | MCP (stdio, HTTP/SSE), OpenAI Functions, LangChain Tools |
| **Ferramenta (Tool)** | Unidade atômica de capacidade | query_database, send_email, get_account_balance |
| **Orquestrador** | Planejamento e seleção de ferramentas | ReAct loop, Plan-and-Execute, Multi-Agent |

---

## 2. Padrões de Design de Agentes (2025–2026)

### 2.1 ReAct (Reasoning + Acting)

O padrão mais usado. Alterna entre **Pensamento → Ação → Observação** em loop.

```
Pensamento: Preciso verificar o saldo da conta 12345.
Ação: get_account_balance({ account_id: "12345" })
Observação: { balance: 15430.00, currency: "BRL" }
Pensamento: Saldo disponível é R$15.430,00. Vou informar ao usuário.
Resposta: Seu saldo atual é R$15.430,00.
```

**Quando usar:** Tarefas com múltiplos passos onde o resultado de uma ação determina o próximo passo.

### 2.2 Plan-and-Execute

Separa o planejamento da execução. O agente primeiro cria um plano completo, depois executa cada etapa.

```
PLANO:
  1. Buscar extrato dos últimos 30 dias
  2. Agrupar transações por categoria
  3. Identificar as 3 maiores despesas
  4. Gerar relatório formatado

EXECUÇÃO: [executa etapas 1→4 em sequência]
```

**Quando usar:** Tarefas previsíveis e determinísticas, relatórios, processamento batch.

### 2.3 Multi-Agent (Orquestrador + Especialistas)

Um agente orquestrador delega subtarefas a agentes especialistas.

```
Orquestrador
├── Agente Compliance  → verifica limites regulatórios
├── Agente Analytics   → analisa padrões de fraude
└── Agente Notificação → envia alertas ao cliente
```

**Quando usar:** Domínios complexos onde especialização melhora qualidade e reduz contexto por agente.

### 2.4 Reflection (Auto-revisão)

O agente revisa a própria saída antes de entregar ao usuário.

```
Resposta inicial → [Agente Revisor] → Crítica → [Agente Principal] → Resposta final
```

**Quando usar:** Outputs críticos (relatórios financeiros, documentos jurídicos, análises de risco).

### 2.5 Human-in-the-Loop

Ações de alto risco requerem aprovação humana antes de execução.

```python
# Pseudocódigo de um checkpoint de aprovação
if action.risk_level >= RiskLevel.HIGH:
    approval = await request_human_approval(action)
    if not approval.granted:
        return abort(reason=approval.reason)
```

**Obrigatório para:** Transferências acima de limites, alterações cadastrais, estornos.

---

## 3. Anatomia de uma Ferramenta (Tool)

Toda ferramenta bem projetada tem:

```json
{
  "name": "get_account_transactions",
  "description": "Retorna transações de uma conta bancária num intervalo de datas. Use apenas para contas autorizadas pelo usuário autenticado.",
  "input_schema": {
    "type": "object",
    "properties": {
      "account_id": {
        "type": "string",
        "description": "ID interno da conta (UUID)"
      },
      "start_date": {
        "type": "string",
        "format": "date",
        "description": "Data inicial no formato YYYY-MM-DD"
      },
      "end_date": {
        "type": "string",
        "format": "date",
        "description": "Data final no formato YYYY-MM-DD"
      },
      "limit": {
        "type": "integer",
        "default": 50,
        "maximum": 500,
        "description": "Máximo de registros a retornar"
      }
    },
    "required": ["account_id", "start_date", "end_date"]
  }
}
```

### Boas práticas de descrição

- Seja específico sobre **o que a ferramenta faz** e **quando usá-la**
- Mencione **restrições de acesso** na descrição (o modelo leva isso em conta)
- Documente **valores de retorno** esperados
- Inclua exemplos de **erros possíveis**

---

## 4. Tipos de Interface por Protocolo

### 4.1 MCP (Model Context Protocol) — Padrão Recomendado

- Protocolo aberto da Anthropic (Nov 2024), adotado pela OpenAI em Mar 2025
- Suporta **Tools**, **Resources** (dados somente-leitura) e **Prompts**
- Transporte: `stdio` (local) ou `HTTP + SSE` (remoto)
- Mais de 200 servidores disponíveis no ecossistema (Mar 2026)

### 4.2 OpenAI Function Calling

- Nativo nos modelos GPT e compatíveis
- JSON Schema para definição de ferramentas
- Suporte a `parallel_tool_calls` para execução simultânea

### 4.3 LangChain / LangGraph Tools

- Framework Python com abstrações sobre múltiplos provedores
- `@tool` decorator para criação rápida
- Integração nativa com memoria, callbacks e traçabilidade

---

## 5. Comunicação Agente-para-Agente (A2A)

Quando agentes precisam se comunicar, use interfaces assíncronas:

```
[Agente A] → mensagem (JSON) → [Fila / Message Bus] → [Agente B]
                                                            ↓
[Agente A] ← resultado ← [Fila de resposta] ←────────────────
```

**Formatos recomendados:**
- Payload em JSON com `task_id`, `agent_id`, `payload`, `metadata`
- Timeout explícito em cada chamada
- Dead letter queue para falhas

---

## 6. Interfaces de Memória

| Tipo | Escopo | Implementação |
|------|--------|---------------|
| **In-context** | Dentro do prompt atual | Histórico de conversa |
| **Episódica** | Sessão do usuário | Redis / banco de sessão |
| **Semântica** | Conhecimento persistente | Vector DB (pgvector, Pinecone) |
| **Procedural** | Habilidades e rotinas | System prompt / skills |

---

## 7. Observabilidade e Rastreamento

Toda interface de agente deve expor:

- **Trace ID:** identificador único por sessão/conversa
- **Span por tool call:** início, fim, parâmetros, resultado
- **Tokens consumidos:** por chamada e acumulado
- **Latência:** tempo de resposta do LLM e de cada ferramenta
- **Erros:** tipo, mensagem, stack trace (sem dados sensíveis)

```python
# Exemplo com OpenTelemetry
with tracer.start_as_current_span("tool.get_account_balance") as span:
    span.set_attribute("account_id", account_id)  # nunca logar saldo aqui
    span.set_attribute("user_id", user_id)
    result = await get_balance(account_id)
    span.set_attribute("success", True)
```

---

## 8. Checklist de Interface Segura

- [ ] Autenticação exigida antes de qualquer tool call
- [ ] Autorização por recurso (o usuário X pode acessar a conta Y?)
- [ ] Rate limiting por usuário e por tool
- [ ] Validação de entrada (evitar SQL injection, command injection)
- [ ] Dados sensíveis nunca aparecem em logs
- [ ] Timeout definido em toda chamada externa
- [ ] Todas as ações auditadas com timestamp, user_id, parâmetros
- [ ] Confirmação humana para ações irreversíveis

---

## Referências

- [Agentic Design Patterns (Augment Code)](https://www.augmentcode.com/guides/agentic-design-patterns)
- [7 Design Patterns Every AI Agent Developer Should Know](https://pub.towardsai.net/the-7-design-patterns-every-ai-agent-developer-should-know-in-2026-c77f28b51565)
- [Model Context Protocol - Architecture](https://gregrobison.medium.com/the-model-context-protocol-the-architecture-of-agentic-intelligence-cfc0e4613c1e)
- [A Survey of AI Agent Protocols (arxiv)](https://arxiv.org/pdf/2504.16736)
