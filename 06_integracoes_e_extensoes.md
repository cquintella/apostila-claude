# Capítulo 6: Integrações e Extensões

"Claude sozinho é apenas um modelo de linguagem." A frase é uma meia-verdade que vale desfazer logo de início. O **modelo** cru — o LLM do Capítulo 1 — de fato só transforma tokens em tokens: não busca na web, não roda código, não abre um arquivo. Mas o **Claude que você usa** quase nunca é o modelo cru: quando você abre o Claude Code ou o Desktop, já há um *harness* em volta dele (Cap. 8) que injeta o system prompt, carrega o seu `CLAUDE.md`, registra suas skills e expõe ferramentas. Ou seja, "Claude conectado a ferramentas" não é um estado futuro que você constrói — é, desde o primeiro `claude`, como a coisa já funciona. Este capítulo é sobre **essa camada de conexões**: o que vem embutido, o que você pluga, e como orquestrar tudo.

## Objetivo e mapa do capítulo

O objetivo é prático: dar a você um **mapa das formas de estender Claude** e critério para escolher entre elas. Antes de descer aos detalhes, vale a taxonomia — porque "integração" é uma palavra que cobre coisas muito diferentes, e saber em que categoria você está evita confusão:

| Categoria | Exemplos | O que você ganha |
| --- | --- | --- |
| **Capacidades nativas** | code interpreter, web search, artifacts | executar código, acessar dados atuais, iterar em artefatos |
| **Ferramentas de conhecimento** | Graphify, understand-anything | entender codebases grandes de forma barata; onboarding |
| **Integrações de plataforma** | Zapier, Make, Google, Slack, Notion, GitHub | agir dentro dos apps que o time já usa |
| **Orquestração multi-ferramenta** | workflows sequenciais/paralelos | combinar etapas com tratamento de falha |
| **Protocolos e API** | Anthropic API, webhooks, MCP | integração programática e reutilizável |

No fundo, tudo o que este capítulo cobre entrega uma de três coisas: **acesso** (a dados e informação atual que não estão nos pesos), **ação** (efeito no mundo — gravar num banco, enviar um e-mail, abrir um PR) e **economia** (token e reuso — não reprocessar o que já se sabe). Guarde essa tríade: diante de qualquer extensão nova, a pergunta útil é "isto me dá acesso, ação ou economia — e vale o custo e o risco?".

> [!warning] A fronteira de confiança
> Cada ferramenta que você conecta amplia a superfície de ataque. No momento em que o modelo lê uma página web, um e-mail ou o resultado de uma API, ele está consumindo **conteúdo não confiável** que pode conter instruções maliciosas (*prompt injection*). A regra que atravessa o capítulo inteiro: **valide nas bordas** — trate toda entrada externa como suspeita e verifique toda ação de efeito colateral antes de executá-la. O tratamento a fundo está no Capítulo 10.

## Capacidades Nativas

São as ferramentas que já vêm embutidas — você não instala nada, apenas aciona.

### Code Interpreter

O code interpreter (execução de código) roda código que o modelo escreve dentro de um ambiente isolado. Tecnicamente, é um contêiner *sandboxed* — tipicamente com Python e uma boa biblioteca de ferramentas de dados pré-instaladas (pandas, numpy, matplotlib, e bibliotecas para gerar arquivos como `.docx`, `.pptx` e PDFs). Não tem acesso à internet, por segurança, e os contêineres podem ser reutilizados entre requisições para manter estado. Os casos de uso clássicos são análise de dados, geração de gráficos e cálculos que o modelo não deve fazer "de cabeça". A limitação principal é justamente o isolamento: sem rede, e com recursos finitos. É a materialização do princípio do Capítulo 1 — escrever código é prever, executá-lo é verificar.

### Web Search

A busca na web dá ao modelo acesso a informação posterior ao seu corte de conhecimento. Pode ser acionada manual ou automaticamente. A arte está em formular boas queries e em saber **quando não usar** — para fatos estáveis, a busca só adiciona latência. Os resultados vêm com citações, o que ajuda a verificar a confiabilidade. É a ferramenta certa para eventos recentes, preços atuais e documentação que mudou.

### Artifacts

Os Artifacts são painéis onde código, documentos e pequenas aplicações aparecem separados da conversa — editáveis, executáveis e compartilháveis. Surgem quando você pede algo substancial e autônomo (um componente, um documento, um script). São ideais para iterar sobre um mesmo artefato sem poluir o histórico. A limitação é que vivem dentro da interface; para integração programática, você quer a API.

## Ferramentas de Conhecimento: entender codebases sem estourar o contexto

Há uma categoria de extensão que resolve um problema específico e cada vez mais comum: fazer um assistente **entender um repositório grande** sem despejar o repositório inteiro no contexto. Duas ferramentas maduras ocupam esse espaço, e vale conhecer as duas porque servem a objetivos ligeiramente diferentes.

### Graphify

O **Graphify** é uma **skill open-source (licença MIT)** que constrói um **grafo de conhecimento consultável** a partir de um codebase multi-modal — código, documentação, papers e diagramas. Funciona com Claude Code e outros assistentes (Codex, OpenCode, Cursor, Gemini CLI), e é mantido por Safi Shamsi.

A ideia central distingue o Graphify de um simples gerador de diagramas. Ele combina **análise estática com Tree-sitter** (que extrai ASTs, grafos de chamada e docstrings localmente, sem mandar o código-fonte para nenhum LLM) com **extração semântica via LLM** (para entender a prosa e, com modelos de visão, ler diagramas). O resultado vai para um grafo NetworkX, sobre o qual roda o algoritmo de **Leiden** para detectar comunidades semânticas — agrupamento por topologia do grafo, sem precisar de *vector embeddings* nem *vector store*. Ele ainda identifica os **"god nodes"** (os nós de maior grau, o coração do sistema) e as **"surprises"** (conexões inesperadas entre arquivos ou domínios, que valem investigação).

O fluxo de trabalho é direto. Requer Python 3.10+; instala-se com `pip install graphifyy && graphify install` (note que o pacote PyPI chama-se `graphifyy`, com dois "y", mas o comando de linha é `graphify`). Para construir o grafo de uma pasta, basta `/graphify ./raw`, e as saídas caem em `graphify-out/`: um `graph.html` interativo, um `graph.json` consultável e um `GRAPH_REPORT.md` legível e auditável. Dentro do assistente, os comandos são `/graphify`, `/graphify query`, `/graphify path` e `/graphify explain`. A integração com Claude Code se dá via diretivas no `CLAUDE.md` e um hook `PreToolUse`.

O **pipeline interno** é uma sequência de estágios isolados — detect (coleta arquivos) → extract (nós e arestas via AST e LLM) → build (grafo NetworkX) → cluster (comunidades Leiden) → analyze (god nodes e surprises) → report → export (HTML/JSON/Obsidian) —, o que facilita estender qualquer etapa.

Por que isso importa? Pela **economia de tokens**. Em vez de responder a uma pergunta sobre o código relendo arquivos inteiros, o assistente consulta o grafo. No corpus de exemplo (uma mistura de repositórios e papers de atenção), o custo médio por consulta caiu de cerca de 123 mil tokens "ingênuos" para cerca de 1,7 mil — uma **redução de aproximadamente 71,5×** (número do próprio benchmark do projeto; o ganho real varia conforme o tamanho e a natureza do codebase). Esse é o diferencial real do Graphify: não é o gráfico bonito, é a estrutura que torna a consulta barata.

Quanto à **segurança e confiança**, o desenho é cuidadoso. O Graphify **não embute um LLM** — usa a chave de API que o seu assistente já tem configurada — e envia ao modelo **apenas conteúdo semântico, nunca o código-fonte bruto**. Não há telemetria. A validação de entrada é estrita: apenas URLs `http`/`https`, com limites de tamanho e timeout, contenção de paths e labels com escape de HTML, defendendo contra SSRF, injeção e XSS.

As **limitações** são as esperadas para esse tipo de ferramenta. A etapa de extração semântica depende de chamadas a um LLM, então tem custo e latência próprios. O retorno é melhor em codebases médios ou grandes e multi-modais; para um projeto trivial, é exagero. E o `GRAPH_REPORT.md` deve ser tratado como ponto de partida de investigação — uma boa primeira pista —, não como verdade absoluta.

### understand-anything, e quando usar cada um

O **understand-anything** é um **plugin do Claude Code** que também transforma um codebase num grafo de conhecimento, mas com um foco diferente: um pipeline **multiagente** mapeia arquivos, funções, classes e dependências, e o resultado alimenta um **dashboard interativo** para explorar a arquitetura visualmente. Ele traz uma visão de **domínio** (mapeando código a fluxos de negócio), **tours guiados** ordenados por dependência, análise de **impacto de diff** antes do merge, e geração de **guias de onboarding**. Seus comandos incluem `/understand`, `/understand-chat`, `/understand-explain`, `/understand-diff`, `/understand-domain` e `/understand-onboard`.

Os dois constroem grafos consultáveis e se sobrepõem, mas a regra de escolha é razoavelmente limpa:

- Use o **Graphify** quando o objetivo é **economia de tokens para o agente**: consultas repetidas e baratas ao código, detecção de comunidades semânticas e de nós centrais, num fluxo que prioriza custo por consulta.
- Use o **understand-anything** quando o objetivo é **entendimento humano e onboarding**: um mapa visual navegável, tours pedagógicos ordenados por dependência, visão de domínio de negócio e impacto de mudanças — "grafos que ensinam mais do que impressionam".

Na prática: o Graphify pende para o lado *agente/economia*; o understand-anything, para o lado *pessoa/didática*. Em times, não é incomum usar os dois — o segundo para humanos entrarem no projeto, o primeiro para o agente trabalhar barato.

## Integrações com Plataformas

Conectar Claude a ferramentas que você já usa multiplica o valor. O padrão se repete: cada plataforma exige um setup básico e abre alguns casos de uso reais.

O **Zapier** e o **Make (Integromat)** são plataformas de automação sem código. Com elas, você liga Claude a centenas de apps — processar e-mails recebidos, extrair dados de formulários, automatizar fluxos. Zapier tende a ser mais simples; Make oferece mais controle e templates prontos para fluxos complexos. A escolha entre os dois é uma questão de gosto e de complexidade do workflow.

A **integração com Google Workspace** cobre o trio Gmail (processar e classificar e-mails), Google Docs (criar e editar documentos) e Google Sheets (analisar e atualizar planilhas). O **Slack** permite bots customizados, integração direta e automação de canais — ótimo para levar Claude para onde o time já conversa. O **Notion** serve para popular bancos de dados, automatizar templates e gerar respostas. E o **GitHub** abre o code review automático, a automação de issues e a análise de pull requests.

Para cada uma dessas, o conselho é o mesmo: comece com um caso de uso concreto e estreito, faça funcionar de ponta a ponta, e só então expanda.

## Orquestração Multi-Ferramenta

### Arquitetura de Workflows

Quando você combina várias ferramentas, precisa decidir a arquitetura. Os passos podem ser **sequenciais** (um depende do anterior) ou **paralelos** (independentes, rodam juntos). O que separa um protótipo de um sistema de produção é o tratamento de falhas: **error handling e fallbacks** (o que fazer quando um passo falha), **timeouts** (não esperar para sempre) e **retry logic** (tentar de novo, com backoff exponencial). Falhas não são exceção; são parte do projeto. (Esta é a distinção *workflow × agente* do Capítulo 5: aqui o fluxo é orquestrado por código seu, previsível.)

### Fluxo de Dados

Os dados precisam fluir entre sistemas que falam línguas diferentes, o que exige **transformação** entre formatos, respeito aos **rate limits** de cada serviço, e **validação** em cada fronteira. A regra clássica: valide nas bordas do sistema (entrada do usuário, APIs externas), confie no interior.

### Casos de Uso Completos

Três pipelines ilustram o padrão. Um **pipeline de análise**: raspar dados → processar com Claude → gravar num banco. Uma **automação de atendimento**: e-mail recebido → classificação → ação apropriada. E um **workflow de geração de conteúdo**: briefing → rascunho → revisão → publicação. Em todos, Claude é uma etapa de um fluxo maior — poderosa, mas uma etapa.

## APIs Custom e MCP

### API da Anthropic

Para integração programática, a API é o caminho. Os pontos a dominar são **autenticação** (chave de API ou perfil OAuth), os **endpoints principais** (sobretudo `/v1/messages`), a escolha entre **streaming e batch**, e o **tratamento de erros** com as classes de exceção tipadas que o SDK oferece. Vale conhecer os códigos de erro mais comuns: 429 (rate limit, retentável), 400 (requisição malformada, não retentável), 401 (autenticação), 529 (sobrecarga, retentável). Os SDKs já fazem retry automático de 429 e 5xx com backoff.

### Webhooks e Custom Integrations

Para integrações dirigidas por eventos, webhooks permitem que sistemas externos notifiquem sua aplicação. O setup básico envolve um endpoint HTTPS; a parte crítica é a **segurança** — verificar a assinatura HMAC de cada entrega para garantir que veio de quem diz ter vindo — e o **debugging** de entregas que falham (dedupe por ID de evento, porque retentativas chegam com o mesmo ID).

### MCP (Model Context Protocol)

O **MCP** é um protocolo aberto e padronizado para conectar Claude a fontes de dados e ferramentas externas. Em vez de cada integração reinventar a roda, um **MCP server** expõe capacidades (ler um banco, consultar uma API, acessar arquivos) de forma que qualquer cliente compatível — incluindo Claude — possa usar. Você pode criar servers customizados para suas próprias fontes.

Dois detalhes valem ouro na prática, e ambos são aprofundados no Capítulo 8. Primeiro, o MCP tem **dois transportes**: o **STDIO**, para servers que rodam na mesma máquina do cliente (sem latência de rede nem autenticação), e o **SSE/HTTP**, para servers remotos (que exigem autenticação). A regra: STDIO quando o server pode viver localmente, remoto só quando precisa morar em outro lugar. Segundo, a **qualidade das descrições de ferramenta** decide se o modelo escolhe a tool certa — descreva-as como documentação de API, não como rótulos vagos. O MCP é a base sobre a qual integrações modernas e reutilizáveis são construídas, e o motivo de o ecossistema de extensões crescer tão rápido.

> [!note] E o prompt caching?
> O *prompt caching* — reaproveitar o processamento de um prefixo grande e estável (system prompt, exemplos, um documento) por ~10% do custo — é uma técnica de **economia**, não de integração, e por isso vive no **Capítulo 7**, junto das demais alavancas de custo. Mencionamos aqui só para fechar o mapa: quando um MCP server ou uma skill injeta um bloco grande e fixo de contexto, esse bloco é um ótimo candidato a cache.

### Resumo do Capítulo 6

O modelo cru só gera texto, mas o Claude que você usa já vem embrulhado num harness com ferramentas — e este capítulo mapeia essa camada em cinco categorias (nativas, conhecimento, plataforma, orquestração, protocolos), cada uma entregando **acesso**, **ação** ou **economia**. As capacidades nativas (code interpreter, web search, artifacts) não exigem instalação. As ferramentas de conhecimento (Graphify, understand-anything) transformam repositórios em grafos consultáveis — Graphify para economia do agente, understand-anything para onboarding humano. Integrações de plataforma levam Claude aos apps do time; a orquestração multi-ferramenta trata falha como regra; e o MCP é o protocolo que torna tudo reutilizável. Em todas, a fronteira de confiança vale: conteúdo externo é não confiável, valide nas bordas.

## Referências

- Anthropic, *Model Context Protocol* (visão geral) — <https://docs.claude.com/en/docs/agents-and-tools/mcp>
- *Model Context Protocol* — especificação oficial — <https://modelcontextprotocol.io>
- Graphify (Safi Shamsi) — <https://github.com/safishamsi/graphify> · <https://graphify.net/>
- understand-anything (plugin do Claude Code) — <https://github.com/Lum1104/Understand-Anything>
- Anthropic, *Building Effective Agents* (workflows e orquestração) — <https://www.anthropic.com/engineering/building-effective-agents>

---

## 🚀 Exercícios Práticos do Capítulo 6

1. **Acesso, ação ou economia:** Liste cinco integrações que você usaria e classifique cada uma pela tríade (acesso / ação / economia). Para as de "ação", anote qual efeito colateral precisa de verificação antes de executar.
2. **Grafo de um repositório:** Rode o Graphify (ou o understand-anything) num projeto real e formule três perguntas que você responderia consultando o grafo em vez de reler arquivos. Compare o custo em tokens.
3. **Graphify × understand-anything:** Para um mesmo repositório, decida qual das duas ferramentas usaria para (a) integrar um dev novo e (b) fazer o agente responder perguntas baratas sobre o código. Justifique.
4. **Fronteira de confiança:** Desenhe um workflow que leia um e-mail externo e tome uma ação. Marque onde está a entrada não confiável e que validação você poria em cada borda.
5. **MCP local vs remoto:** Escolha uma fonte de dados sua e decida se o MCP server correspondente deveria usar transporte STDIO ou remoto. Justifique com base em onde o server precisa viver.
