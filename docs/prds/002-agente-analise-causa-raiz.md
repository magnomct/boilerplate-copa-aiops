---
prd_number: "002"
status: rascunho
priority: alta
created: 2026-03-22
issue:
depends_on: ["001"]
references: ["003", "004"]
---

# PRD 002: Agente de Analise de Causa Raiz e Geracao de Relatorio

## 1. Contexto

- **Sistema/produto**: Agente AIOps para Kubernetes — aplicacao Python com FastAPI, LangChain, Claude Sonnet e MCP Server Kubernetes. O PRD 001 implementa a coleta periodica de eventos Warning do cluster.
- **Estado atual**: Eventos sao coletados automaticamente (PRD 001), mas nao ha analise automatizada. O diagnostico de causa raiz e manual, dependente da experiencia do operador, e nao produz documentacao estruturada. Problemas recorrentes sao reinvestigados do zero.
- **Problema**: O tempo de diagnostico e alto (30min-2h) e varia com a experiencia do operador. Nao ha registro padronizado do processo de troubleshooting, dificultando a transferencia de conhecimento. Eventos ja em tratamento podem ser reprocessados desnecessariamente, desperdicando recursos de API e gerando relatorios duplicados.

## 2. Solucao Proposta

### Visao geral

- Implementa o `EventHandler` do PRD 001, recebendo lista de eventos no contrato de saida definido naquele PRD
- Filtra os eventos que ja estao em tratamento (deduplicacao por `metadata.uid` do evento Kubernetes)
- Agente construido com LangChain + Claude Sonnet usando loop de tool-calling (estilo ReAct, via `create_agent`) investiga a causa raiz
- O agente se conecta ao cluster via MCP Server Kubernetes (Flux159) para inspecionar o estado dos recursos, utilizando o pacote `langchain-mcp-adapters` para integracao; apenas as tools de leitura (`kubectl_get`, `kubectl_describe`, `kubectl_logs`) sao vinculadas ao agente
- Gera um relatorio Markdown estruturado por problema agrupado, com severidade, causa raiz, evidencias, comandos executados e passos de correcao
- Persiste cada relatorio no PostgreSQL com status e referencia aos UIDs dos eventos daquele problema

### Decisoes-chave

1. **LangChain como framework de agente** — Abstracoes prontas para ReAct, integracao com MCP via pacote `langchain-mcp-adapters` e gerenciamento de tools
2. **Claude Sonnet como LLM** — Equilibrio entre capacidade de raciocinio e custo/velocidade para troubleshooting
3. **MCP Server Kubernetes (Flux159)** — Servidor MCP existente para interacao com o cluster, evitando implementacao propria
4. **Loop de tool-calling (estilo ReAct)** — Permite iteracao entre analise e acao ate encontrar a causa raiz real; o `create_agent` implementa tool-calling nativo da API
5. **Deduplicacao por `metadata.uid`** — Eventos ja vinculados a relatorios ativos sao ignorados; eventos vinculados a relatorios com status CORRIGIDO ou INCOMPLETO sao reanalisados (possivel reincidencia / analise nao concluida). Limitacao conhecida: o EventAggregator do Kubernetes pode criar novos objetos Event com UIDs diferentes para o mesmo problema logico em alta frequencia (10+ ocorrencias em 10 minutos). Risco aceito na v1.
6. **Status EM_ANALISE para evitar race condition** — O relatorio e criado com status EM_ANALISE como **primeira operacao do tratamento, antes da chamada ao agente** (insert-first), garantindo que despachos concorrentes (tasks assincronas do PRD 001 que se sobrepoem) e a recuperacao de crash identifiquem os eventos como "em tratamento" e nao disparem analise duplicada. Com intervalo de coleta de ~3 min e ciclos de coleta serializados, a gravacao antecipada do EM_ANALISE dispensa trava de concorrencia (insert atomico/constraint) na v1; a trava atomica fica registrada como endurecimento futuro (intervalo muito menor ou re-disparo manual/externo)
7. **Somente tools de leitura vinculadas ao agente** — A lista de tools retornada pelo `langchain-mcp-adapters` e filtrada para expor apenas `kubectl_get`, `kubectl_describe` e `kubectl_logs`; tools de escrita do MCP Server nao chegam ao agente de analise
8. **Um relatorio por problema agrupado** — Cada problema identificado pelo agente vira um relatorio proprio com seus event_uids, permitindo acompanhar e corrigir cada incidente ate a resolucao de forma independente (status e correcao nos PRDs 003/004 operam por relatorio)

### Fora do escopo

- **Coleta de eventos** — Responsabilidade do PRD 001
- **Interface de visualizacao dos relatorios** — Responsabilidade do PRD 003
- **Execucao da correcao** — Responsabilidade do PRD 004
- **Historico de eventos brutos no banco** — Apenas relatorios sao persistidos

## 3. Funcionalidades

### US01: Filtragem de eventos ja em tratamento

Como SRE, quero que eventos ja vinculados a um relatorio ativo nao sejam reanalisados, para evitar relatorios duplicados e desperdicio de recursos de API.

**Rules:**
- Antes de disparar a analise, verificar se os `metadata.uid` dos eventos ja estao vinculados a relatorios existentes
- Eventos vinculados a relatorios com status EM_ANALISE, COMPLETO, CORRIGINDO ou FALHA_CORRECAO sao ignorados
- Eventos vinculados a relatorios com status CORRIGIDO ou INCOMPLETO sao reanalisados (problema pode ter reincidido; analise nao concluida deve ser reavaliada ate o incidente ser resolvido)
- A tabela de relatorios armazena o campo `event_uids` (`TEXT[]` com GIN index — lista de UIDs dos eventos Kubernetes analisados)
- Relatorios em EM_ANALISE ha mais de `ANALYSIS_STALE_TIMEOUT_MINUTES` minutos (padrao: 30) sao marcados como INCOMPLETO durante a consulta de deduplicacao — recuperacao de analises estagnadas, complementar a recuperacao do startup

**Edge cases:**
- Todos os eventos do batch ja estao em tratamento → nao disparar analise, aguardar proximo ciclo
- Parte dos eventos do batch ja esta em tratamento → filtrar e analisar apenas os novos
- Evento reaparece apos relatorio marcado como CORRIGIDO → reanalisar e gerar novo relatorio
- Banco de dados indisponivel para consulta de deduplicacao → registrar erro no log e descartar o batch; como o despacho do PRD 001 e assincrono (fire-and-forget), nao ha propagacao ao coletor nem hold de marca d'agua. Evento recorrente volta no proximo ciclo; evento one-shot e perdido (risco aceito na v1)

### US02: Analise automatizada de causa raiz

Como SRE, quero que um agente de IA investigue automaticamente a causa raiz dos eventos coletados, para reduzir o tempo de diagnostico e nao depender da experiencia individual de cada operador.

**Rules:**
- O agente e construido com LangChain + Claude Sonnet em loop de tool-calling (estilo ReAct), via `create_agent` de `langchain.agents`
- O modelo e configuravel via variavel de ambiente `AGENT_MODEL_NAME` (padrao: `claude-sonnet-4-6`)
- O agente se conecta ao cluster via MCP Server Kubernetes (Flux159), utilizando `langchain-mcp-adapters` para converter tools MCP em tools LangChain; o servidor e executado localmente via `npx` expondo transport HTTP (streamable), e o agente conecta por HTTP (`streamable_http`) em `MCP_SERVER_URL` (padrao `http://localhost:3001/mcp`)
- Apenas as tools de leitura `kubectl_get`, `kubectl_describe` e `kubectl_logs` sao vinculadas ao agente — a lista retornada pelo `langchain-mcp-adapters` e filtrada por nome antes da criacao do agente; tools de escrita (`kubectl_apply`, `kubectl_patch`, `kubectl_delete`, etc.) nao sao expostas
- A investigacao segue o ciclo: analise → acao → reflexao → proxima acao
- O agente itera ate identificar a causa raiz ou atingir o limite de iteracoes, configuravel via variavel de ambiente `AGENT_MAX_ITERATIONS` (padrao: 25), implementado com `ModelCallLimitMiddleware(run_limit, exit_behavior="end")` no `create_agent`. Com `exit_behavior="end"` o run encerra graciosamente sem uma chamada extra ao modelo — nao ha relatorio parcial garantido; a resposta final (possivelmente vazia ou sem secoes de problema identificaveis) mantem o relatorio do batch como INCOMPLETO (ver US03 edge cases)
- O agente deve investigar o estado dos recursos relacionados ao evento (pod, deployment, node, etc.)

**Prompt do agente:**

```
Voce e um analista especializado em diagnostico Kubernetes. Sua funcao e investigar e identificar problemas usando as ferramentas disponiveis.

Voce recebeu os seguintes eventos Warning coletados do cluster:

{events}

## REGRAS
- Apenas diagnostica e sugere correcoes. NUNCA execute kubectl_patch, kubectl_apply ou kubectl_delete.
- Use APENAS os parametros documentados das tools.
- Investigue por PROBLEMA AGRUPADO, nao por evento individual.
- Se ja tiver causa raiz + evidencias suficientes, passe ao proximo problema.
- NUNCA chame a mesma ferramenta com os mesmos parametros duas vezes.
- Em **Eventos**, liste o uid de CADA evento recebido em exatamente um problema.

## COMO INVESTIGAR
Voce tem 3 ferramentas de leitura. Use-as IMEDIATAMENTE para coletar evidencias:
- **kubectl_get**: visao geral dos recursos (status, restarts, idade). Use `name` OU `labelSelector`, NUNCA ambos juntos.
- **kubectl_describe**: detalhes completos (eventos, conditions, configuracao).
- **kubectl_logs**: logs do container. Use `previous: true` para logs de execucao anterior.

Para cada problema recebido:
1. Chame kubectl_describe no recurso afetado para obter detalhes
2. Se necessario, chame kubectl_logs para ver erros da aplicacao
3. Com as evidencias coletadas, escreva o relatorio

## TRATAMENTO DE ERROS
- "name cannot be provided when a selector is specified" -> Use APENAS name OU labelSelector
- Se uma ferramenta falhar 2 vezes, registre o erro como evidencia e prossiga

## SEVERIDADE
- **CRITICO**: Pod/Deployment indisponivel, servico fora do ar
- **ALTO**: Restarts frequentes, OOMKilled, recursos esgotados
- **MEDIO**: Warnings recorrentes mas servico funcional
- **BAIXO**: Eventos informativos, problemas cosmeticos

## FORMATO DE RESPOSTA (Markdown)

# Relatorio de Diagnostico Kubernetes

## Resumo
| Total | Criticos | Altos | Medios | Baixos |
|-------|----------|-------|--------|--------|
| X     | X        | X     | X      | X      |

---

## Problema 1: [causa raiz resumida]
- **Severidade:** CRITICO | ALTO | MEDIO | BAIXO
- **Namespace:** namespace-afetado
- **Recursos Afetados:** pod1, deployment/nome
- **Eventos:** uid-evento-1, uid-evento-2

### Causa Raiz
Descricao clara do problema.

### Evidencias
- evidencia 1
- evidencia 2

### Solucao Recomendada
Acao especifica para corrigir.

### Comando Sugerido
kubectl patch deployment nome -n namespace --type=merge -p '{"spec":...}'

Repita a secao para cada problema. Responda APENAS com o relatorio markdown.
```

**Edge cases:**
- MCP Server Kubernetes indisponivel → abortar analise, atualizar relatorio para INCOMPLETO e registrar erro no log; sem propagacao ao coletor (despacho assincrono do PRD 001). Recuperacao via dedup-reanalise na recorrencia do evento (eventos vinculados a INCOMPLETO sao reanalisados, US01) + recovery de startup/stale; evento one-shot que nao recorre e perdido (risco aceito na v1)
- Agente atinge limite de iteracoes sem identificar causa raiz → gerar relatorio parcial marcando causa raiz como "inconclusiva" com status INCOMPLETO
- Erro na API do Claude (rate limit, timeout) → retry automatico do SDK com backoff exponencial (`max_retries=3` no `ChatAnthropic`); esgotadas as tentativas, abortar, atualizar relatorio para INCOMPLETO e registrar erro, sem propagacao ao coletor (despacho assincrono do PRD 001)
- Evento ja corrigido entre a coleta e a analise → agente identifica estado saudavel e registra no relatorio
- Variavel de ambiente `AGENT_MAX_ITERATIONS` com valor invalido (nao-numerico, negativo ou zero) → usar o padrao de 25 iteracoes e registrar warning no log

### US03: Geracao e persistencia de relatorio estruturado

Como SRE, quero que cada investigacao produza um relatorio Markdown estruturado e persistido, para ter documentacao padronizada de troubleshooting e facilitar a transferencia de conhecimento.

**Rules:**
- A resposta do agente segue a estrutura definida no prompt: resumo com contagem por severidade e uma secao por problema (severidade, namespace, recursos afetados, eventos, causa raiz, evidencias, solucao recomendada e comando sugerido)
- Cada secao de problema e persistida como um relatorio proprio no PostgreSQL, com ID unico e os event_uids dos eventos daquele problema (mapeados pelo campo **Eventos** da resposta)
- Durante a analise existe um unico relatorio EM_ANALISE por batch (com todos os event_uids); ao final, ele e substituido pelo conjunto de relatorios por problema na mesma transacao
- A tabela de relatorios armazena: id (`UUID`), conteudo markdown, status, event_uids (`TEXT[]` — UIDs dos eventos Kubernetes analisados), `created_at` e `updated_at` (timestamps de criacao e atualizacao)
- Status possiveis do relatorio: EM_ANALISE, COMPLETO, INCOMPLETO, CORRIGINDO, CORRIGIDO, FALHA_CORRECAO
- Fluxo de status: EM_ANALISE → COMPLETO (analise OK) | INCOMPLETO (analise inconclusiva ou abortada por falha de infraestrutura); COMPLETO → CORRIGINDO → CORRIGIDO | FALHA_CORRECAO; FALHA_CORRECAO → CORRIGINDO (nova tentativa de correcao, PRD 004 — FALHA_CORRECAO nao e terminal)
- Relatorios em EM_ANALISE encontrados no startup da aplicacao sao marcados como INCOMPLETO (recuperacao de analises interrompidas por crash/restart)

**Edge cases:**
- PostgreSQL indisponivel no momento da persistencia → retry com backoff; apos falha definitiva, registrar erro no log incluindo o conteudo Markdown gerado (sem fallback em arquivo); sem propagacao ao coletor (despacho assincrono do PRD 001). O relatorio EM_ANALISE estagnado e recuperado pelo timeout da deduplicacao (stale) ou no startup; evento one-shot afetado e perdido (risco aceito na v1)
- Resposta do agente vazia ou sem secoes de problema identificaveis (falha do LLM) → manter o relatorio EM_ANALISE do batch como INCOMPLETO para rastreabilidade
- Dois relatorios gerados simultaneamente para eventos relacionados → cada um recebe ID proprio; deduplicacao por `event_uids` impede reanalise nos ciclos futuros
- Aplicacao reinicia durante uma analise → relatorio orfao em EM_ANALISE e marcado como INCOMPLETO no startup, liberando os eventos para reanalise
- Evento do batch nao referenciado no campo **Eventos** de nenhum problema da resposta → registrar warning no log; o evento fica sem relatorio proprio e sera reanalisado apenas se recorrer (recoletado em um ciclo futuro)

## 4. Visao de Arquitetura

```
┌─────────────────────────────────────────────────────────┐
│                    Pipeline de Analise                    │
│                                                          │
│  Eventos (PRD 001 via EventHandler)                      │
│       │                                                  │
│       ▼                                                  │
│  ┌──────────────────┐    ┌──────────────┐                │
│  │ Filtro de        │───▶│ PostgreSQL   │                │
│  │ Reprocessamento  │◀───│ (event_uids) │                │
│  │ (dedup por UID)  │    └──────────────┘                │
│  └────────┬─────────┘                                    │
│           │ eventos novos                                │
│           ▼                                              │
│  ┌──────────────────────┐    ┌──────────────┐            │
│  │ Criar relatorio      │───▶│ PostgreSQL   │            │
│  │ status: EM_ANALISE   │    │ (relatorios) │            │
│  └────────┬─────────────┘    └──────────────┘            │
│           │                                              │
│           ▼                                              │
│  ┌─────────────────────┐                                 │
│  │  Agente de Analise  │                                 │
│  │ (LangChain + Claude)│                                 │
│  │  Padrao ReAct       │                                 │
│  └────────┬────────────┘                                 │
│           │ MCP (via langchain-mcp-adapters)              │
│           ▼                                              │
│  ┌────────────────┐                                      │
│  │ MCP Server K8s │──▶ Cluster K8s                       │
│  │ (Flux159)      │                                      │
│  └────────────────┘                                      │
│           │                                              │
│           ▼                                              │
│  ┌─────────────────────┐    ┌──────────────┐             │
│  │ Persistir relatorios │───▶│ PostgreSQL   │             │
│  │ por problema (US03)  │    │ (relatorios) │             │
│  └─────────────────────┘    └──────────────┘             │
└─────────────────────────────────────────────────────────┘
```

## 5. Criterios de Aceite

### Tecnicos

| Criterio | Metodo de verificacao |
|----------|----------------------|
| Eventos com UID ja vinculado a relatorio ativo sao ignorados | Teste automatizado com eventos duplicados e relatorios em diferentes status |
| Eventos vinculados a relatorio CORRIGIDO sao reanalisados | Teste automatizado simulando reincidencia |
| Relatorio e criado com status EM_ANALISE antes de iniciar investigacao | Teste automatizado validando que ciclo subsequente ignora eventos em analise |
| Agente conecta ao cluster via MCP e executa pelo menos uma iteracao ReAct | Teste de integracao com cluster de teste e evento simulado |
| Relatorio segue a estrutura definida no prompt (resumo, severidade, causa raiz, evidencias, solucao) | Validacao de schema do Markdown via teste automatizado |
| Relatorio e persistido no PostgreSQL com ID unico e event_uids | Teste de integracao com banco de dados |
| Cada problema do batch gera um relatorio proprio com seus event_uids | Teste com batch contendo dois problemas independentes validando a criacao de dois relatorios |
| Relatorio EM_ANALISE estagnado alem do timeout e marcado INCOMPLETO na deduplicacao | Teste automatizado com relatorio EM_ANALISE antigo e nova consulta de deduplicacao |
| Relatorio inconclusivo e persistido com status INCOMPLETO | Teste com cenario onde o agente atinge limite de iteracoes |
| Eventos vinculados a relatorio INCOMPLETO sao reanalisados | Teste automatizado simulando analise inconclusiva seguida de recorrencia (recoleta) dos eventos |
| Relatorios EM_ANALISE orfaos sao marcados como INCOMPLETO no startup | Teste automatizado simulando restart com relatorio em EM_ANALISE |
| Agente possui apenas tools de leitura vinculadas (kubectl_get, kubectl_describe, kubectl_logs) | Teste automatizado inspecionando a lista de tools do agente |
| Limite de iteracoes e configuravel via `AGENT_MAX_ITERATIONS` | Teste variando o valor da env var e validando comportamento |

### De negocio

| Metrica | Baseline (fonte) | Meta | Prazo | Min. aceitavel | Responsavel |
|---------|-------------------|------|-------|-----------------|-------------|
| Tempo medio de diagnostico (da recepcao do evento ao relatorio) | 30min-2h manual (estimativa do time de SRE) | < 10 minutos | 30 dias apos deploy | < 20 minutos | Time de SRE |
| Taxa de identificacao correta da causa raiz | N/A — nao ha processo estruturado hoje | > 70% | 30 dias apos deploy | > 50% | Time de SRE |

## 6. Milestones

### Milestone 1: Implementar Filtragem de Reprocessamento

**Objetivo:** Garantir que eventos ja em tratamento nao sejam reanalisados.

**Funcionalidades:** US01, US03 (modelo de dados)

- [ ] Modelar tabela de relatorios no PostgreSQL (id `UUID`, markdown, status, event_uids `TEXT[]` com GIN index, `created_at`, `updated_at`) (US03)
- [ ] Implementar consulta de deduplicacao por `event_uids` e status do relatorio (US01)
- [ ] Implementar logica de filtragem antes de disparar analise (US01)
- [ ] Implementar criacao de relatorio com status EM_ANALISE antes de iniciar investigacao (US01, US03)
- [ ] Implementar recuperacao no startup: marcar relatorios EM_ANALISE orfaos como INCOMPLETO (US01, US03)
- [ ] Implementar timeout de estagnacao de EM_ANALISE na consulta de deduplicacao (`ANALYSIS_STALE_TIMEOUT_MINUTES`) (US01)

**Criterio de conclusao:**
- Condicao: Eventos duplicados sao corretamente filtrados com base no status dos relatorios existentes, incluindo EM_ANALISE
- Verificacao: Teste automatizado com cenarios de deduplicacao (evento novo, evento em analise, evento em tratamento, evento corrigido que reincide)
- Aprovador: Time de SRE

### Milestone 2: Implementar Agente de Analise

**Objetivo:** Agente investiga causa raiz automaticamente e gera relatorio estruturado.

**Funcionalidades:** US02, US03

- [ ] Implementar `EventHandler` do PRD 001 como ponto de entrada do pipeline de analise (US02)
- [ ] Configurar LangChain com Claude Sonnet (`AGENT_MODEL_NAME`, padrao `claude-sonnet-4-6`) em loop de tool-calling via `create_agent` de `langchain.agents` (US02)
- [ ] Configurar MCP Server Kubernetes (Flux159) via `langchain-mcp-adapters` (MCP executado via npx com transport HTTP/streamable, conexao `streamable_http` em `MCP_SERVER_URL`), filtrando apenas as tools de leitura kubectl_get, kubectl_describe e kubectl_logs (US02)
- [ ] Implementar logica de investigacao iterativa com limite de iteracoes configuravel via `AGENT_MAX_ITERATIONS`, usando `ModelCallLimitMiddleware` (US02)
- [ ] Gerar relatorios Markdown por problema seguindo a estrutura do prompt, com mapeamento de event_uids pelo campo **Eventos** (US03)
- [ ] Substituir o relatorio EM_ANALISE do batch pelos relatorios por problema com status COMPLETO/INCOMPLETO (US03)
- [ ] Configurar `max_retries=3` no `ChatAnthropic` (retry com backoff nativo do SDK) (US02)

**Criterio de conclusao:**
- Condicao: Dado um evento Warning, o agente investiga via MCP, identifica causa raiz e atualiza relatorio estruturado no banco
- Verificacao: Teste de integracao end-to-end com cluster de teste e evento Warning simulado
- Aprovador: Time de SRE

## 7. Riscos e Dependencias

| Risco | Impacto | Mitigacao | Status |
|-------|---------|-----------|--------|
| Qualidade da analise do LLM pode variar — causa raiz incorreta | Alto | Monitorar taxa de acerto e iterar nos prompts. Relatorio disponivel para revisao humana | Pendente |
| Custo de API do Claude pode escalar com volume alto de eventos e investigacoes longas | Medio | Monitorar uso de tokens. Limitar iteracoes do agente via `AGENT_MAX_ITERATIONS` | Pendente |
| MCP Server Kubernetes pode nao suportar todas as operacoes necessarias | Medio | Validar operacoes necessarias contra a API do MCP Server antes do desenvolvimento. Operacoes de leitura (kubectl_get, kubectl_describe, kubectl_logs) confirmadas como suportadas | Pendente |
| EventAggregator do Kubernetes cria novos Event objects com UIDs diferentes para o mesmo problema logico em alta frequencia (10+ ocorrencias em 10 min) | Baixo | Risco aceito na v1. Deduplicacao por `metadata.uid` nao colapsara eventos agregados. Mitigacao futura: deduplicar por `involvedObject` + `reason` + `namespace` | Aceito |
| Reanalise recorrente de incidentes com relatorio INCOMPLETO consome tokens a cada recorrencia (recoleta) dos eventos | Medio | Aceito — analise nao concluida deve ser reavaliada ate o incidente ser resolvido. Mitigacao futura: limite de reanalises por incidente | Aceito |

**Dependencias:**

| Dependencia | Tipo | Status | Impacto se bloqueado |
|-------------|------|--------|----------------------|
| PRD 001 — Coleta de eventos (contrato de saida e `EventHandler`) | Interna | Em desenvolvimento | Sem eventos coletados, nao ha o que analisar |
| LangChain (`langchain` + `langchain-anthropic` + `langgraph`) | Externa | Disponivel | Framework base do agente |
| `langchain-mcp-adapters` (>=0.3.0) | Externa | Disponivel | Integracao entre LangChain e MCP Server. Piso >=0.3.0: o prompt do agente assume que erro de tool retorna ao modelo como tool message; versoes anteriores lancam `ToolException` e quebram o loop |
| MCP Server Kubernetes (Flux159) | Externa | Disponivel | Sem MCP, agente nao interage com o cluster |
| Node.js + npx (runtime do MCP Server Kubernetes, executado localmente com transport HTTP) | Externa | A provisionar | Sem o runtime Node, o MCP server nao inicia |
| API Claude (Anthropic) | Externa | Disponivel | Sem LLM, agente nao funciona |
| Instancia PostgreSQL | Interna | A provisionar | Sem banco, relatorios nao sao persistidos |
| Driver PostgreSQL para Python (`asyncpg`) | Externa | Disponivel | Sem driver, aplicacao nao conecta ao banco |

## 8. Referencias

- [PRD 001 - Coleta de Eventos](./001-coleta-eventos-kubernetes.md) — fornece os eventos consumidos por este PRD (contrato de saida e `EventHandler`)
- [PRD 003 - Interface Web](./003-interface-web.md) — consome os relatorios gerados por este PRD
- [PRD 004 - Agente de Correcao](./004-agente-correcao-automatica.md) — consome os relatorios gerados por este PRD
- [MCP Server Kubernetes (Flux159)](https://github.com/Flux159/mcp-server-kubernetes) — servidor MCP utilizado pelo agente
- [LangChain Agents](https://docs.langchain.com/oss/python/langchain/agents) — guia de agents (`create_agent`)
- [Referencia da API `create_agent`](https://reference.langchain.com/python/langchain/agents/factory/create_agent) — assinatura e parametros
- [Middleware built-in do LangChain](https://docs.langchain.com/oss/python/langchain/middleware/built-in) — `ModelCallLimitMiddleware` usado no limite de iteracoes
- [langchain-mcp-adapters](https://github.com/langchain-ai/langchain-mcp-adapters) — pacote de integracao LangChain + MCP

## 9. Registro de Decisoes

- **2026-06-13:** Consumo assincrono do PRD 001: o `EventHandler` roda fire-and-forget e falhas de infraestrutura (banco na dedup, MCP indisponivel, API do Claude esgotada, PostgreSQL na persistencia) NAO propagam ao coletor — reverte as decisoes de 2026-06-10 de propagacao ao coletor. A deduplicacao por UID e o unico guardiao de correcao; eventos one-shot que falham antes de gerar relatorio sao perdidos (risco aceito na v1, alinhado ao PRD 001).
- **2026-06-13:** Relatorio EM_ANALISE criado insert-first (antes da chamada ao agente); sem trava de concorrencia (insert atomico/constraint) na v1. Motivo: com intervalo de coleta de ~3 min e ciclos serializados, a gravacao antecipada do EM_ANALISE ja fecha a janela de despacho duplicado; trava atomica fica como endurecimento futuro.
- **2026-06-13:** `exit_behavior="end"` esclarecido: ao atingir o limite o run encerra sem chamada extra ao modelo — nao ha relatorio parcial garantido; a classificacao INCOMPLETO vem do caminho de resposta vazia/sem secoes (US03). Alinha a redacao com o PRD 004.
- **2026-06-13:** Piso de `langchain-mcp-adapters` elevado para >=0.3.0. Motivo: o prompt do agente depende de erro de tool retornar ao modelo como tool message; versoes anteriores lancam `ToolException` e quebrariam o loop de investigacao. Fonte: docs LangChain (MCP loading tools).
- **2026-06-13:** FALHA_CORRECAO deixa de ser estado terminal — passa a admitir a transicao FALHA_CORRECAO → CORRIGINDO (nova tentativa de correcao, PRD 004). Motivo: sempre deve ser possivel re-acionar a correcao (decisao do usuario em revisao de 2026-06-13). As regras de deduplicacao (US01) NAO mudam: FALHA_CORRECAO e CORRIGINDO continuam ignorados na dedup — a recuperacao acontece via re-fix (botao do PRD 003), nao via reanalise.
- **2026-06-10:** Um relatorio por problema agrupado, em vez de um por batch. Motivo: status e correcao (PRD 004) operam por relatorio; com batch unico, correcao parcial marcaria o conjunto inteiro como FALHA_CORRECAO e eventos de problemas ja corrigidos ficariam presos (decisao do usuario em revisao de 2026-06-10). O relatorio EM_ANALISE do batch e substituido pelos relatorios por problema ao final; o prompt passa a exigir o campo **Eventos** com os uids por problema.
- **2026-06-10:** Removido o fallback de arquivo local na falha definitiva de persistencia. Motivo: filesystem de pod e efemero e o arquivo nao teria consumidor; o conteudo vai para o log e os eventos sao reanalisados via marca d'agua + timeout de estagnacao (decisao do usuario em revisao de 2026-06-10).
- **2026-06-10:** Timeout de estagnacao para EM_ANALISE (`ANALYSIS_STALE_TIMEOUT_MINUTES`, padrao 30 minutos) aplicado na consulta de deduplicacao. Motivo: complementa a recuperacao do startup — sem ele, falha definitiva de persistencia deixaria o relatorio preso em EM_ANALISE ate um restart.
- **2026-06-10:** Eventos vinculados a relatorios INCOMPLETO passam a ser reanalisados (antes eram ignorados). Motivo: analise nao concluida deve ser reavaliada — um incidente precisa ser resolvido (decisao do usuario em revisao de 2026-06-10). Custo de reanalise recorrente aceito como risco.
- **2026-06-10:** Todo caminho de aborto da analise (MCP indisponivel, API do Claude com retries esgotados) atualiza o relatorio para INCOMPLETO; relatorios EM_ANALISE orfaos sao marcados como INCOMPLETO no startup. Motivo: relatorio preso em EM_ANALISE bloquearia os UIDs para sempre na deduplicacao.
- **2026-06-10:** Falha na consulta de deduplicacao propaga erro ao coletor (PRD 001), que nao avanca a marca d'agua. Motivo: eventos brutos nao sao persistidos; sem segurar a marca d'agua, eventos one-shot seriam perdidos em falhas transitorias. *(Substituida em 2026-06-13 — ver acima: despacho assincrono fire-and-forget nao propaga ao coletor; perda de one-shot aceita.)*
- **2026-06-10:** Apenas as tools de leitura kubectl_get, kubectl_describe e kubectl_logs sao vinculadas ao agente, filtrando a lista do `langchain-mcp-adapters`. Motivo: o MCP Server expoe ~23 tools incluindo escrita (kubectl_apply/patch/delete); instrucao de prompt nao e garantia, e o modo `ALLOW_ONLY_NON_DESTRUCTIVE_TOOLS` do servidor mantem apply/patch habilitados.
- **2026-06-10:** Limite de iteracoes implementado via `ModelCallLimitMiddleware(run_limit, exit_behavior="end")`. Motivo: `create_agent` nao possui parametro de max iterations; `recursion_limit` lancaria `GraphRecursionError` sem permitir o relatorio parcial exigido pelo edge case de limite atingido.
- **2026-06-10:** Retry de erros da API do Claude via `max_retries=3` do `ChatAnthropic`, sem implementacao propria. Motivo: o SDK da Anthropic ja aplica backoff exponencial para 429/5xx.
- **2026-06-10:** Modelo definido como `claude-sonnet-4-6` (alias sem sufixo de data), configuravel via `AGENT_MODEL_NAME`. Motivo: a decisao de 2026-03-13 fixava a familia Sonnet; a implementacao exige o ID exato.
- **2026-06-13:** MCP Server executado localmente via `npx` expondo transport HTTP (streamable); o agente conecta por HTTP (`streamable_http`) em `MCP_SERVER_URL` (padrao `http://localhost:3001/mcp`), nao por stdio. Node.js/npx mantido como dependencia de runtime. Motivo: o `langchain-mcp-adapters` conecta ao MCP por HTTP, alinhado ao modelo de execucao local (Docker provisiona apenas o banco).
- **2026-06-10:** Terminologia ajustada de "padrao ReAct" para "loop de tool-calling (estilo ReAct)". Motivo: `create_agent` implementa tool-calling nativo da API, nao o ReAct textual classico; preserva a intencao da decisao de 2026-03-13.
- **2026-03-22:** Filtragem de reprocessamento fica neste PRD, nao na coleta (PRD 001). Motivo: a decisao depende do estado dos relatorios, que e responsabilidade deste modulo.
- **2026-06-10:** Campo `id` da tabela de relatorios definido como `UUID` e timestamps nomeados como `created_at` e `updated_at`. Motivo: PRD 003 expoe o `id` em URL (`GET /reports/{id}`) — UUID e mais seguro para exposicao publica; nomes de campos estabelecidos para eliminar ambiguidade entre PRDs.
- **2026-03-22:** Deduplicacao por `metadata.uid` do evento Kubernetes. Motivo: evitar relatorios duplicados e desperdicio de tokens de API. Limitacao aceita: EventAggregator do K8s pode gerar UIDs diferentes para o mesmo problema logico.
- **2026-03-22:** Eventos vinculados a relatorios CORRIGIDO sao reanalisados. Motivo: problema pode ter reincidido apos correcao.
- **2026-03-22:** Status EM_ANALISE adicionado ao fluxo. Motivo: evitar race condition onde ciclos subsequentes de coleta disparam analise duplicada para eventos ja em investigacao.
- **2026-03-22:** `event_uids` armazenado como `TEXT[]` com GIN index no PostgreSQL. Motivo: consulta de deduplicacao via `ANY()` e a operacao mais frequente; array nativo com GIN e a opcao mais performatica.
- **2026-03-22:** Limite de iteracoes do agente configuravel via `AGENT_MAX_ITERATIONS` (padrao: 25). Motivo: balancear profundidade da investigacao com custo de API e tempo de resposta.
- **2026-03-22:** Prompt do agente documentado no PRD. Motivo: o prompt determina a qualidade da analise e deve ser rastreavel como decisao tecnica.
- **2026-03-22:** `create_agent` de `langchain.agents` escolhido como API de criacao do agente. Motivo: API atual e recomendada do LangChain, substituindo `create_react_agent` depreciado no LangGraph v1.0.
- **2026-03-22:** Integracao MCP via `langchain-mcp-adapters`. Motivo: pacote oficial do langchain-ai para converter tools MCP em tools LangChain. Nao ha integracao MCP nativa no core do LangChain.
- **2026-03-13:** Claude Sonnet escolhido como LLM. Motivo: equilibrio entre capacidade de raciocinio e custo.
- **2026-03-13:** Padrao ReAct para investigacao. Motivo: permite iteracao ate encontrar causa raiz real.
