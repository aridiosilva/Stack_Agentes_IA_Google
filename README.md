# Stack_Agentes_IA_Google

A pilha de agentes do Google: ADK, A2A e MCP no Google Cloud  (Isso é baseado em um codelab oficial do Google Cloud chamado exatamente assim, que usa como estudo de caso um app fictício de planejamento social chamado "InstaVibe") 

# A pilha de agentes do Google: ADK, A2A e MCP no Google Cloud

Baseado no codelab oficial do Google Cloud *"Google's Agent Stack in Action: ADK, A2A, MCP on Google Cloud"*, que usa como estudo de caso um app fictício de planejamento social chamado **InstaVibe**.

O workshop guia a construção, orquestração e conexão de agentes colaborativos usando ADK, comunicação A2A (Agent-to-Agent) e o Model Context Protocol (MCP), culminando no deploy do sistema multiagente no Google Cloud.

## A ideia por trás do caso de uso

A proposta é resolver o problema de usuários que acham o processo de planejar atividades em grupo trabalhoso — descobrir os interesses dos amigos, escolher eventos e coordenar tudo. A solução é um sistema com quatro agentes especializados:

- **Social Profiling Agent** — faz "social listening" para identificar interesses compartilhados analisando conexões e interações dos usuários.
- **Event Planning Agent** — busca eventos, locais e ideias online alinhados aos critérios identificados.
- **Platform Interaction Agent** — usa uma ferramenta MCP para criar posts e registrar eventos na plataforma.
- **Orchestrator Agent** — coordenador central que recebe o pedido do usuário e delega tarefas aos agentes especializados na ordem certa.

## 1. ADK (Agent Development Kit) — o framework

O ADK é o framework open-source do Google para construir sistemas de agentes inteligentes, com o objetivo de tornar o desenvolvimento de agentes parecido com desenvolvimento de software tradicional. Conceitos centrais:

- **Agent**: encapsula instruções, o modelo (Gemini) e um conjunto de `Tools`.
- **Session / State / Memory / Event**: Session é o "container" de uma conversa; State é a memória de curto prazo (dicionário mutável); Memory é o potencial de recall entre sessões; Event registra cada interação de forma imutável (mensagem do usuário, chamada de ferramenta, resposta do modelo etc).
- **Runner**: o motor de execução — orquestra o fluxo de Events, atualiza o State, invoca o LLM e coordena chamadas de ferramentas.
- **Workflow Agents**: agentes que não executam tarefas diretamente, mas orquestram sub-agentes. O ADK oferece três tipos prontos — **Sequential**, **Parallel** e **Loop**. No exemplo do InstaVibe, o Social Profiling Agent usa um `LoopAgent` que repete a sequência `profile_agent → summary_agent → check_agent → CheckCondition` até que todos os perfis tenham sido resumidos.
- **Callbacks**: permitem injetar lógica em pontos do ciclo de vida do agente — por exemplo, um `after_agent_callback` que extrai o resumo final do State quando o loop termina.

Versões recentes do ADK já trazem registries de Skills (locais e apoiados no Google Cloud) para capacidades reutilizáveis, suportam A2A 1.x nativamente e permitem expor um agente ADK via MCP usando `to_mcp_server` — ou seja, um agente pode ser tanto uma aplicação quanto uma capacidade usada por outros agentes.

## 2. MCP (Model Context Protocol) — agente ↔ ferramenta

O MCP é um padrão aberto que resolve o problema de precisar de integrações customizadas para cada combinação de aplicação de IA e fonte de dados, oferecendo uma interface universal via arquitetura cliente-servidor.

No InstaVibe, o **Platform Interaction Agent** usa um cliente MCP para falar com um servidor MCP que expõe as APIs internas da plataforma (`create_post`, `create_event`) como ferramentas. O servidor implementa duas funções centrais:

- `list_tools` — permite que o cliente descubra as ferramentas disponíveis (nome, descrição, schema JSON dos parâmetros).
- `call_tool` — executa a ferramenta pedida, recebendo nome e argumentos.

O transporte evoluiu de stdio local (bom para desenvolvimento) para servidores remotos via HTTP com Server-Sent Events (SSE), que permitem múltiplos clientes compartilharem um único servidor, centralizam a gestão de ferramentas e mantêm chaves de API no lado do servidor em vez de espalhadas pelos clientes. No Cloud, esse servidor MCP roda como um serviço no **Cloud Run**.

## 3. A2A (Agent-to-Agent) — agente ↔ agente

Enquanto o MCP resolve agente-para-ferramenta, o A2A resolve agente-para-agente. É um protocolo aberto, originalmente anunciado pelo Google em abril de 2025 e transferido em junho de 2025 para a Linux Foundation, que passou a fornecer governança neutra de fornecedor.

Elementos-chave:

- **Agent Card**: um JSON público (`/.well-known/agent.json`) que descreve nome, capacidades, skills e URL do agente — é assim que outros agentes o "descobrem".
- **A2A Server**: envolve o agente ADK, expõe o Agent Card e recebe tarefas de outros agentes (clientes A2A). Usa padrões web já conhecidos como HTTP, SSE e JSON-RPC, geralmente em modelo cliente-servidor, onde um agente cliente envia tarefas para um agente remoto/servidor.
- **a2a-python**: biblioteca concreta que faz os agentes ADK "falarem" A2A — cuida de servir o Agent Card e gerenciar as requisições de tarefa recebidas.
- **A2A Inspector**: ferramenta web de debug para conectar, inspecionar e interagir com agentes A2A durante o desenvolvimento (visualiza o Agent Card, permite chat direto e mostra as mensagens JSON-RPC cruas).

No exemplo, o **Orchestrator Agent** funciona como cliente A2A: ele resolve os Agent Cards dos agentes Planner, Platform e Social (via `A2ACardResolver`), guarda as conexões e usa uma ferramenta `send_message` para delegar tarefas — aguardando sempre a confirmação de sucesso antes de seguir para o próximo passo da sequência.

## 4. A infraestrutura no Google Cloud

| Serviço | Papel |
|---|---|
| **Vertex AI (Gemini)** | Modelos de raciocínio dos agentes (ex.: gemini-2.0-flash, gemini-2.5-flash) |
| **Vertex AI Agent Engine** | Serviço gerenciado para hospedar e escalar o agente orquestrador em produção |
| **Cloud Run** | Hospeda a webapp InstaVibe, cada agente A2A como microsserviço independente, e o servidor MCP |
| **Spanner** | Banco relacional usado como **grafo** (GRAPH DDL) para modelar relações sociais — pessoas, amizades, presença em eventos, posts |
| **Artifact Registry** | Armazena as imagens de container dos agentes/servidor MCP/app |
| **Cloud Build** | Constrói as imagens Docker a partir do código-fonte |
| **Cloud Storage** | Suporte a builds e ao Agent Engine |

Um detalhe interessante é o uso do Spanner como grafo: consultas em Graph SQL permitem encontrar amizades diretas entre pessoas, conexões indiretas via eventos compartilhados, ou pessoas mencionadas em posts de amigos — dando ao Social Profiling Agent uma forma rica de entender o contexto social do usuário.

## 5. Como tudo se conecta na prática

1. Usuário pede ao Orchestrator: "planeje um evento para mim e meus amigos".
2. Orchestrator (via A2A) delega ao **Social Agent** a análise dos perfis (ADK LoopAgent consultando o grafo no Spanner).
3. Orchestrator delega ao **Planner Agent** a geração de sugestões (ADK Agent usando a tool `google_search`), com base nos interesses identificados.
4. Orchestrator delega ao **Platform Interaction Agent** a criação do post/evento — esse agente usa MCP para chamar as APIs internas do InstaVibe através do servidor MCP.
5. Cada etapa só avança após confirmação explícita de sucesso vinda do `tool_output`.

---

# Tags sugeridas — A pilha de agentes do Google: ADK, A2A e MCP no Google Cloud

## Tecnologias e protocolos
- ADK / Agent-Development-Kit
- A2A / Agent2Agent
- MCP / Model-Context-Protocol
- Gemini
- Vertex-AI

## Infraestrutura Google Cloud
- Google-Cloud
- Cloud-Run
- Spanner
- Vertex-AI-Agent-Engine
- Artifact-Registry
- Cloud-Build

## Conceitos e arquitetura
- Multi-Agent-Systems
- Agentes-de-IA
- Orquestração-de-Agentes
- IA-Generativa
- LLM
- Graph-Database
- Microsserviços

## Nível/formato
- Codelab
- Tutorial
- Arquitetura-de-Software
- GRC

---

### Formato hashtag
#ADK #A2A #MCP #GoogleCloud #VertexAI #Gemini #CloudRun #Spanner #MultiAgentSystems #AgentesDeIA #IAGenerativa #LLM #GraphDatabase #Codelab #ArquiteturaDeSoftware #GRC

### Formato lista simples (minúsculas, separadas por vírgula)
adk, a2a, mcp, google-cloud, vertex-ai, gemini, cloud-run, spanner, vertex-ai-agent-engine, artifact-registry, cloud-build, multi-agent-systems, agentes-de-ia, orquestracao-de-agentes, ia-generativa, llm, graph-database, microsservicos, codelab, tutorial, arquitetura-de-software, grc

**Fonte do artigo original:** https://codelabs.developers.google.com/instavibe-adk-multi-agents/instructions

*Nota: essa é a arquitetura de um codelab educacional específico do Google (InstaVibe). O ADK, A2A e MCP em si são ferramentas genéricas que podem ser combinadas de formas diferentes conforme o caso de uso.*

**Fonte:** [Google's Agent Stack in Action: ADK, A2A, MCP on Google Cloud — Google Codelabs](https://codelabs.developers.google.com/instavibe-adk-multi-agents/instructions)
