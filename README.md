# Microsoft Foundry AI Agents: agentes de IA com ferramentas, MCP e memória na Azure

🧩 **Curso Microsoft Foundry (Udemy) · Seção 4: Microsoft Foundry AI Agents.** Nesta seção construí, **pelo código** (Python + Foundry SDK), uma sequência de agentes no **Foundry Agent Service**. Cada um ganha uma capacidade nova: conversar com memória, pesquisar na web, executar Python, chamar APIs REST, usar servidores **MCP**, combinar várias ferramentas, operar um navegador e lembrar do usuário entre sessões com o **Memory Store**.

Cada lab tem um notebook documentado em português (para quem não é da área acompanhar), um diagrama de **arquitetura** e um de **fluxo** em SVG e, na última célula, uma **interface de chat em Streamlit** que reaproveita o código do notebook.

> ### ⭐ O que foi entregue
>
> | Lab | Agente | Capacidade nova | Serviços / ferramentas |
> | --- | --- | --- | --- |
> | [12](#lab-12--tour-pelo-agent-service-no-portal) | `Demo Agent` | Agente criado pelo portal: versões, tracing e avaliação | Foundry portal, OpenTelemetry, avaliadores |
> | [13](#lab-13--agente-básico-com-memória-de-conversa) | `batman-agent` | Agente criado pelo código + memória de conversa | Agent Service, Conversations, GPT-4 |
> | [14](#lab-14--agente-pesquisador-da-web) | `web-search-agent` | Informação atual da internet, com citações | Web Search tool |
> | [15](#lab-15--agente-analista-de-dados-com-code-interpreter) | `code-interpreter-agent` | Escreve e executa Python, gera gráficos a partir de um CSV | Code Interpreter, Files, Containers |
> | [16](#lab-16--agente-com-ferramenta-openapi-clima) | `weather-agent` | Chama qualquer API REST a partir da especificação OpenAPI | OpenAPI tool, API wttr.in |
> | [17](#lab-17--agente-com-servidor-mcp-microsoft-learn) | `MCP-Agent` | Usa um servidor MCP (documentação e exemplos da Microsoft) | MCPTool, Connections, Microsoft Learn MCP |
> | [18](#lab-18--agente-com-várias-ferramentas) | `multi-tool-agent` | Planeja e combina várias ferramentas em uma pergunta | MCP + OpenAPI |
> | [19](#lab-19--agente-que-navega-na-web-browser-automation) | `browser-automation-agent` | Controla um navegador real: digita, clica e lê a tela | Browser Automation, Azure Playwright Workspace |
> | [22](#lab-22--assistente-com-memória-de-longo-prazo-memory-store) | `helpful-assistant-agent` | Lembra fatos de cada usuário entre sessões | Memory Store, embeddings, RBAC |

---

## Visão geral

### O padrão que se repete em todos os labs

Todos os agentes seguem o mesmo esqueleto. O que muda de um lab para outro é a lista de **ferramentas** (`tools`) que o agente recebe.

```python
# 1. Conecta ao projeto Foundry, sem chave no código (login do az login / VS Code)
project_client = AIProjectClient(endpoint=FOUNDRY_PROJECT_ENDPOINT, credential=DefaultAzureCredential())

# 2. Cria (uma nova versão de) o agente: modelo + instruções + ferramentas
agent = project_client.agents.create_version(
    agent_name="meu-agente",
    definition=PromptAgentDefinition(model=MODEL_DEPLOYMENT_NAME, instructions="...", tools=[...]),
)

# 3. Cria a conversa (histórico) e envia a pergunta ao agente pela Responses API
openai_client = project_client.get_openai_client()
conversation = openai_client.conversations.create()
response = openai_client.responses.create(
    conversation=conversation.id,
    extra_body={"agent": {"name": "meu-agente", "type": "agent_reference"}},
    input="pergunta do usuário",
)
print(response.output_text)
```

| Peça | O que é | Analogia |
| --- | --- | --- |
| **Projeto Microsoft Foundry** | Onde ficam modelos, agentes, conexões, conversas e traces. | O escritório |
| **Foundry Agent Service** | Hospeda os agentes e orquestra as chamadas de ferramentas. | O RH que contrata e coordena os funcionários |
| **Agente** (`PromptAgentDefinition`) | Modelo + instruções + ferramentas, com **versões** (v1, v2…). | Um funcionário com manual de conduta |
| **Ferramentas** (`tools`) | O que o agente pode usar: web, Python, APIs, MCP, navegador. **Ele decide quando usar.** | As ferramentas de trabalho do funcionário |
| **Conversation** (`conversation.id`) | Histórico das mensagens de um diálogo. | O caderno de anotações da conversa |
| **Microsoft Entra ID** (`DefaultAzureCredential`) | Autenticação pelo login já feito na máquina: **nenhuma chave no código**. | O crachá |
| **Tracing** | Entradas, saídas, chamadas de ferramentas e tokens de cada execução (OpenTelemetry). | A câmera de segurança |

### Tecnologias

| Categoria | Itens |
| --- | --- |
| Plataforma | Microsoft Foundry, Foundry Agent Service, Foundry Memory Store |
| Modelos (Azure OpenAI) | GPT-4, GPT-4.1, text-embedding-3-small |
| Ferramentas de agente | Web Search, Code Interpreter, OpenAPI, MCP (Model Context Protocol), Browser Automation |
| Outros serviços Azure | Microsoft Entra ID, RBAC (Azure AI User), Managed Identity, Azure Playwright Workspace |
| SDKs | `azure-ai-projects` 2.0.0b2 (Foundry SDK v2), `openai`, `azure-identity`, `python-dotenv`, `jsonref` |
| Interface | Streamlit |

---

## Lab 12 · Tour pelo Agent Service no portal

Antes do código, um passeio pela área **Agentes** do portal do Foundry, criando o `Demo Agent` pela interface gráfica:

- **Versionamento de agentes:** cada **Salvar** gera uma nova versão, e dá para voltar a uma versão anterior e comparar versões, como em um registro de containers.
- **Tracing** com **OpenTelemetry**: tokens de entrada e saída, `conversation.id` (o equivalente às *threads* dos projetos baseados em hub), IDs de trace e span.
- **Avaliação de qualidade:** *Intent Resolution* (o agente entendeu a intenção?) e *Task Adherence* (executou as etapas pedidas?), com nota de 1 a 5 e justificativa.
- **Catálogo de ferramentas** (File Search, Code Interpreter, Azure AI Search, Bing, Browser Automation, Fabric, SharePoint), **Foundry IQ** e **Memory Store**.

> Lab feito inteiramente no portal, por isso não tem notebook.

---

## Lab 13 · Agente básico com memória de conversa

**Problema:** criar um assistente com personalidade e memória sem montar infraestrutura.
**Solução:** o `batman-agent` (GPT-4 com instruções para responder como o Batman), criado com `create_version`. O `conversation.id` faz o agente lembrar o nome dito na primeira mensagem. No tracing, os tokens sobem de **139 para 219** porque o histórico é reenviado a cada mensagem.

![Arquitetura do Lab 13](13%20-%20Lab%20Creating%20a%20Basic%20Agent%20from%20Python%20Code/arquitetura.svg)

📓 [`agent_run.ipynb`](13%20-%20Lab%20Creating%20a%20Basic%20Agent%20from%20Python%20Code/agent_run.ipynb) · 🧠 [como a conversa dá memória ao agente](13%20-%20Lab%20Creating%20a%20Basic%20Agent%20from%20Python%20Code/conversa_memoria.svg) · 🖥️ Streamlit: chat com botão "Nova conversa" e publicação de novas versões do agente

---

## Lab 14 · Agente pesquisador da web

**Problema:** o modelo só conhece o que viu no treinamento; perguntas sobre fatos recentes ficam desatualizadas e sem fonte.
**Solução:** o `web-search-agent` com `WebSearchPreviewTool()`. A pergunta **não** pede para buscar: o agente analisa a intenção e decide sozinho. A resposta volta com anotações `url_citation` (URL + título). Na demonstração: avanços em energia renovável em 2025, com cerca de **7.254 tokens** por causa do texto das páginas.

![Arquitetura do Lab 14](14%20-%20Lab%20Creating%20a%20Web%20Searcher%20Agent/arquitetura.svg)

📓 [`webSearch.ipynb`](14%20-%20Lab%20Creating%20a%20Web%20Searcher%20Agent/webSearch.ipynb) · 🔀 [fluxo](14%20-%20Lab%20Creating%20a%20Web%20Searcher%20Agent/fluxo_web_search.svg) · 🖥️ Streamlit: chat que mostra os links das fontes de cada resposta

---

## Lab 15 · Agente analista de dados com Code Interpreter

**Problema:** modelos erram contas e não desenham gráficos; analisar planilhas exige programar.
**Solução:** o CSV `electronics_products.csv` (50 produtos) é enviado ao projeto (`files.create`) e entregue ao `code-interpreter-agent` com `CodeInterpreterTool(container=CodeInterpreterToolAuto(file_ids=[...]))`. O agente escreve **pandas + matplotlib**, executa na **sandbox**, gera o gráfico de colunas e o notebook baixa o PNG pela anotação `container_file_citation`. O tracing mostra cada trecho de código executado.

![Arquitetura do Lab 15](15%20-%20Lab%20Creating%20a%20Code%20Interpreter%20Agent/arquitetura.svg)

📓 [`code_interpreter.ipynb`](15%20-%20Lab%20Creating%20a%20Code%20Interpreter%20Agent/code_interpreter.ipynb) · 🔀 [fluxo](15%20-%20Lab%20Creating%20a%20Code%20Interpreter%20Agent/fluxo_code_interpreter.svg) · 🖥️ Streamlit: gráfico direto no chat (com download), código executado e envio de outro CSV

---

## Lab 16 · Agente com ferramenta OpenAPI (clima)

**Problema:** integrar cada API a um assistente costuma exigir código específico para cada endpoint.
**Solução:** a especificação `weather_openapi.json` (OpenAPI 3.1.0 da API pública **wttr.in**) vira uma ferramenta com `{"type": "openapi", ...}`. O `weather-agent` lê **títulos e descrições** da spec, escolhe a operação `GetCurrentWeather` e preenche `location` e `format=j1`. O resultado (2 °C em Nova York, névoa congelante…) confere com a API. O JSON grande infla os tokens para cerca de **11.026**.

![Arquitetura do Lab 16](16%20-%20Lab%20Creating%20Agent%20with%20OpenAPl%20Tool/arquitetura.svg)

📓 [`openapi.ipynb`](16%20-%20Lab%20Creating%20Agent%20with%20OpenAPl%20Tool/openapi.ipynb) · 🔀 [fluxo](16%20-%20Lab%20Creating%20Agent%20with%20OpenAPl%20Tool/fluxo_openapi.svg) · 🖥️ Streamlit: chat do clima com as operações da spec na barra lateral

---

## Lab 17 · Agente com servidor MCP (Microsoft Learn)

**Problema:** documentação técnica muda o tempo todo e cada integração externa exige código próprio.
**Solução:** o servidor MCP público do **Microsoft Learn** (`https://learn.microsoft.com/api/mcp`) é registrado como **conexão** no portal. O código busca o `connection.id` (`connections.list()`) e monta o `MCPTool` (`require_approval="never"`). O `MCP-Agent` **descobre** as ferramentas do servidor (`mcp_list_tools`) e chama a certa (`microsoft_code_sample_search`, `microsoft_docs_search`). Resultado: exemplos de código e trilhas de estudo com links oficiais.

![Arquitetura do Lab 17](17%20-%20Lab%20Creating%20a%20MCP%20Server%20Agent/arquitetura.svg)

📓 [`mcp.ipynb`](17%20-%20Lab%20Creating%20a%20MCP%20Server%20Agent/mcp.ipynb) · 🔀 [fluxo](17%20-%20Lab%20Creating%20a%20MCP%20Server%20Agent/fluxo_mcp.svg) · 🖥️ Streamlit: chat que mostra as ferramentas MCP listadas e chamadas, com os argumentos

---

## Lab 18 · Agente com várias ferramentas

**Problema:** uma mesma pergunta pode precisar de fontes diferentes; regras fixas do tipo "se falar de X, chame Y" não escalam.
**Solução:** o `multi-tool-agent` recebe `tools=[weather_tool, mcp_tool]`. Para *"clima em Seattle + um módulo do Learn sobre o Microsoft Foundry"*, o **orquestrador interno** do agente divide a pergunta, chama `microsoft_docs_search` (MCP) e `GetCurrentWeather` (OpenAPI) e combina tudo em uma resposta (cerca de 17 mil tokens).

![Arquitetura do Lab 18](18%20-%20Lab%20Creating%20a%20Multi-Tool%20Agent/arquitetura.svg)

📓 [`run.ipynb`](18%20-%20Lab%20Creating%20a%20Multi-Tool%20Agent/run.ipynb) · 🔀 [fluxo](18%20-%20Lab%20Creating%20a%20Multi-Tool%20Agent/fluxo_multi_tool.svg) · 🖥️ Streamlit: chat com o plano executado pelo agente e botões de teste (só clima, só Learn, os dois)

---

## Lab 19 · Agente que navega na web (Browser Automation)

**Problema:** muita informação só existe na tela de um site, sem API; *scrapers* são frágeis.
**Solução:** um **Azure Playwright Workspace** fornece navegadores em sandbox, conectado ao Foundry (endpoint + token guardados na conexão, fora do código). O `browser-automation-agent` recebe instruções passo a passo (abrir finance.yahoo.com, buscar MSFT, clicar em YTD) e responde **14,3%**.
**Limitações observadas:** 2 a 3 minutos por tarefa, sensibilidade à forma do pedido e bloqueio por Cloudflare/CAPTCHA em alguns sites.

![Arquitetura do Lab 19](19%20-%20Lab%20Working%20with%20Browser%20Automation%20Tool/arquitetura.svg)

📓 [`automation.ipynb`](19%20-%20Lab%20Working%20with%20Browser%20Automation%20Tool/automation.ipynb) · 🔀 [fluxo](19%20-%20Lab%20Working%20with%20Browser%20Automation%20Tool/fluxo_browser_automation.svg) · 🖥️ Streamlit: formulário para tarefas passo a passo, com aviso de latência

---

## Lab 22 · Assistente com memória de longo prazo (Memory Store)

**Problema:** lembrar do usuário entre sessões exigia montar bancos vetoriais e lógica de embeddings, e reenviar todo o histórico estoura tokens.
**Solução:** o **Foundry Memory Store** (`demo_memory_store_v1`) usa um **LLM** (GPT-4.1) para extrair só os fatos úteis e um modelo de **embedding** (text-embedding-3-small) para a busca. As memórias ficam separadas por **escopo** (o usuário). A cada pergunta, `search_memories` traz até 5 fatos relevantes, anexados como contexto. O agente sugeriu receitas **sem amendoim e sem laticínios** e escreveu código em **Python** sem o usuário repetir nada. Pré-requisito: a função **Azure AI User** para a identidade gerenciada do projeto (Lab 21).

![Arquitetura do Lab 22](22%20-%20Lab%20Working%20with%20Foundry%20Memory%20Store/arquitetura.svg)

📓 [`memories.ipynb`](22%20-%20Lab%20Working%20with%20Foundry%20Memory%20Store/memories.ipynb) · 🔀 [fluxo](22%20-%20Lab%20Working%20with%20Foundry%20Memory%20Store/fluxo_memory_store.svg) · 🖥️ Streamlit: escolha do usuário (escopo), botão para ensinar fatos e as memórias usadas em cada resposta

---

## Comparando as ferramentas

| Ferramenta | Quando usar | Configuração no portal | Atenção |
| --- | --- | --- | --- |
| **Web Search** | Fatos recentes, com fonte | Nenhuma | Texto das páginas aumenta os tokens |
| **Code Interpreter** | Cálculos, análise de arquivos, gráficos | Nenhuma (upload do arquivo pelo código) | Cobrança por sessão de container |
| **OpenAPI** | APIs REST próprias ou de terceiros | Nenhuma (a spec vai no código) | Descrições ruins = chamadas erradas; JSON grande = muitos tokens |
| **MCP** | Servidores MCP (Microsoft Learn, GitHub, internos…) | Registrar a conexão em **Tools** | `require_approval` para ações que alteram dados |
| **Browser Automation** | Sites sem API | Playwright Workspace + conexão com token | Lento (minutos), CAPTCHA, pedidos precisam ser detalhados |
| **Memory Store** | Personalização entre sessões | Modelos de chat e embedding + função RBAC | Preview: limites por escopo e por minuto |

---

## Como executar

1. **Pré-requisitos:** assinatura Azure, um projeto no **Microsoft Foundry** com um deployment de modelo GPT (Azure OpenAI), **Azure CLI** e Python 3.10+.
2. **Login:** `az login`. O `DefaultAzureCredential` usa esse login, então nenhuma chave vai no código.
3. **Configuração:** em cada pasta, ajuste o arquivo **`.env`** com os valores do **seu** projeto: `FOUNDRY_PROJECT_ENDPOINT`, `MODEL_DEPLOYMENT_NAME` e as variáveis extras do lab (descritas na seção "Variáveis de ambiente" de cada notebook). No Lab 13, copie o `env.txt` para `.env`.
4. **Notebook:** execute as células em ordem. A primeira instala as bibliotecas.
5. **Interface:** a última célula gera o `app.py`. Rode `pip install streamlit` e `streamlit run app.py` e abra `http://localhost:8501`.

> Os pré-requisitos específicos (conexão MCP, Playwright Workspace, função RBAC do Memory Store) estão passo a passo nos notebooks dos Labs 17, 19 e 22.

---

## O que aprendi

- **Agente = modelo + instruções + ferramentas**, e o próprio agente decide **quando** usar cada ferramenta a partir da intenção da pergunta.
- **Versionamento** (`create_version`) e **tracing** tornam o agente auditável: dá para ver cada chamada de ferramenta, os argumentos e o custo em tokens.
- **Tokens são o principal custo escondido:** histórico, páginas da web, JSON de APIs e conteúdo de MCP entram no contexto. Monitorar e enxugar faz parte do projeto.
- **Conexões do projeto** (MCP, navegador) guardam endpoints e segredos **fora do código**; o código só referencia o `connection.id`.
- **MCP** padroniza integrações: registrou o servidor, o agente descobre as ferramentas sozinho.
- **Memória** tem dois níveis: a **conversa** (dentro de um diálogo) e o **Memory Store** (entre sessões, por usuário, com busca vetorial).
- **Segurança desde o início:** `DefaultAzureCredential` (Entra ID), RBAC com menor privilégio, `require_approval` para ações sensíveis e segredos guardados nas conexões do projeto, não no código nem no `.env`.

---

## Estrutura da pasta

```
Seção 4 - Microsoft Foundry Al Agents/
├── README.md
├── 13 - Lab Creating a Basic Agent from Python Code/
│   ├── agent_run.ipynb                              notebook documentado + app Streamlit
│   ├── arquitetura.svg · conversa_memoria.svg      diagramas
│   └── env.txt                                      modelo do .env (sem valores)
├── 14 … 22 - Lab …/
│   ├── <notebook>.ipynb                             notebook documentado + app Streamlit
│   ├── arquitetura.svg · fluxo_*.svg               diagramas
│   ├── .env                                         endpoint, deployment e nomes de conexões
│   └── dados do lab (electronics_products.csv, weather_openapi.json)
```

> 🔒 Os arquivos **`.env`** têm apenas endereços e nomes, sem chaves: a autenticação é feita pelo Microsoft Entra ID (`az login` + `DefaultAzureCredential`), e os segredos das ferramentas (como o token do Playwright) ficam guardados nas **conexões do projeto Foundry**, nunca no código. As transcrições das aulas (`.txt`) ficam fora do repositório pelo [`.gitignore`](.gitignore).
