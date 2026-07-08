# Capítulo 2: O que é Claude — Modelos, Plataformas e Fluxo

O Capítulo 1 tratou do LLM como categoria: o que é, como funciona, por que age como age. Este capítulo aterrissa essa teoria num produto concreto — **Claude**, a família de modelos da Anthropic. Vemos o que a distingue no mercado, o mapa dos modelos ativos e seus custos, as portas de entrada (Web, Desktop, API, Claude Code) e, ao final, o caminho que uma requisição percorre por dentro. É a ponte entre "como um LLM funciona" e "como eu escolho e uso Claude na prática".

## O que é Claude

### Claude como Modelo de Linguagem

Claude é a família de modelos de linguagem da **Anthropic**.

O que diferencia Claude no mercado não é apenas a capacidade bruta, mas a filosofia de design. A Anthropic adota uma abordagem chamada **Constitutional AI**: em vez de depender só de humanos rotulando respostas boas e ruins, o modelo é treinado para criticar e revisar as próprias respostas segundo um conjunto explícito de princípios — uma "constituição". O objetivo é um modelo que seja útil, honesto e inofensivo, nessa ordem de prioridade quando há conflito.

Na prática, isso se traduz numa postura de **safety first**: Claude tende a recusar pedidos que parecem perigosos, mas — e isso é um ponto fino — os modelos recentes erram menos para o lado do "recuso por precaução". Ser honesto sobre limitações é parte do design: Claude admite quando não sabe, em vez de inventar com confiança.

#### Políticas Públicas, Regulação e Disponibilidade

Toda decisão de design tem efeitos colaterais. O equilíbrio entre ser útil e ser seguro deixou de ser apenas uma decisão de engenharia e virou também questão de política pública. À medida que os modelos ficam mais capazes, governos correm para regular e, em alguns casos, restringir o acesso a IA de fronteira, e cada jurisdição tende a adotar regras próprias.

Essas restrições levantam questões legítimas em aberto: quem define os limites do que um modelo pode responder, e qual é o efeito dessas restrições sobre o desempenho do modelo e sobre os sistemas que dependem dele?

O rumo provável é de fragmentação: em vez de um punhado de modelos globais disponíveis a todos, é plausível um cenário de modelos com acesso restrito, ou de "IAs soberanas" homologadas país a país. Para quem constrói sobre Claude, a consequência é concreta — a disponibilidade de um modelo pode depender de onde você e o seu usuário estão. Trate isso como uma variável a acompanhar, não como algo estável.

### Linhagem de Modelos Disponíveis

A linha evoluiu por gerações. Convém conhecer o mapa, porque escolher o modelo certo é metade da batalha de custo e qualidade.

A **família Claude 3** (Haiku, Sonnet, Opus) foi a primeira geração amplamente adotada e estabeleceu o padrão de três tamanhos: um rápido e barato, um equilibrado, e um poderoso. É hoje, em sua maioria, histórica — útil para entender a nomenclatura, mas substituída na prática pela geração atual.

A geração atual é a **família Claude 4.x** mais o **Claude Fable 5**. Os principais modelos ativos, com suas características de referência:

| Modelo                | Contexto | Saída máx. | Entrada / Saída (por 1M tokens) | Quando usar                                                    |
| --------------------- | -------- | ---------- | ------------------------------- | -------------------------------------------------------------- |
| **Claude Fable 5**    | 1M       | 128K       | US$ 10 / US$ 50                 | Raciocínio mais exigente, trabalho agêntico de longo horizonte |
| **Claude Opus 4.8**   | 1M       | 128K       | US$ 5 / US$ 25                  | O Opus mais capaz; trabalho autônomo, conhecimento, memória    |
| **Claude Sonnet 5**   | 1M       | 128K       | US$ 3 / US$ 15                  | Melhor equilíbrio velocidade/inteligência                      |
| **Claude Haiku 4.5**  | 200K     | 64K        | US$ 1 / US$ 5                   | Tarefas simples, rápidas e baratas                             |

Há ainda o **Claude Mythos 5**, que tem as mesmas capacidades, preço e comportamento de API do Fable 5, mas só é acessível através de um programa específico (Project Glasswing). Para quase todo mundo, o "modelo mais capaz" é o Fable 5 — onde ele estiver disponível, já que restrições de acesso impostas por governos podem alterar essa conta conforme a região.

Um detalhe operacional fecha esta seção, porque é onde a escolha de modelo encontra o teclado: na API, cada modelo é referenciado por um **identificador exato**, uma string como `claude-opus-4-8`, `claude-sonnet-5` ou `claude-fable-5`. Não adianta inventar sufixos de data nem "arredondar" o nome — um ID que não bate exatamente devolve **erro 404**, e esse é um dos tropeços mais comuns de quem está começando. Guarde a ideia: ela reaparece no Capítulo 9, no diagnóstico de erros da API.

#### Roadmap da Anthropic

Quanto ao **roadmap e futuro**: a Anthropic libera modelos novos com regularidade, e a regra prática é "use o mais recente da sua faixa, a menos que tenha um motivo explícito para fixar um antigo". Modelos antigos são aposentados com aviso prévio e passam a devolver erro.

E existe um roadmap público? Não no sentido de um cronograma detalhado do que vem pela frente: a Anthropic não divulga datas nem nomes de modelos futuros, e anuncia cada geração quando ela fica pronta. O que dá para tratar como certo é a _direção_ — janelas de contexto maiores, raciocínio mais forte, melhor uso de ferramentas e de memória — e a _cadência_, que é alta. Por isso a regra prática não é prever o próximo modelo, e sim projetar o seu sistema para trocar de modelo com facilidade (veja o guia de migração no Capítulo 9).

## Plataformas e Interfaces

O mesmo motor está disponível por várias portas de entrada. Escolher a porta certa é uma decisão de produtividade.

### Claude Web (claude.ai)

A interface web é a porta de entrada mais simples. Tem um nível **gratuito** com limites de uso e um nível **Pro** com mais capacidade e acesso a modelos mais poderosos. Dois recursos merecem destaque. Os **Artifacts** são painéis laterais onde código, documentos ou pequenas aplicações aparecem editáveis e executáveis, separados do fluxo da conversa — ótimos para iterar sobre um mesmo artefato. A **busca na web integrada** permite que Claude consulte informações atuais quando a pergunta exige. A web é ideal para exploração e prototipagem; as **limitações** são os rate limits e a falta de integração profunda com seus arquivos locais.

### Claude Desktop

O aplicativo de desktop (Mac e Windows) traz Claude para perto do seu sistema. Sua configuração permite, entre outras coisas, conectar **MCP servers** (que veremos no Capítulo 6) e estender as capacidades do modelo com ferramentas locais. É também o ponto de contato natural com o **Claude Code**, a ferramenta de terminal. A ideia central é o contexto local: o modelo pode trabalhar com seus arquivos sem que você precise colar tudo manualmente.

### Claude API

A API é a porta para quem constrói software. Aqui você lida diretamente com **rate limiting e quotas** (limites de requisições e de tokens por minuto), **batch processing** (processar milhares de requisições de forma assíncrona, com 50% de desconto), e **streaming** (receber a resposta token a token, em vez de esperar o fim). A API é onde a economia e a robustez de uma aplicação de produção são realmente decididas, e é o foco de boa parte dos Capítulos 4 e 5.

### Claude Code (Ferramenta de Desenvolvimento)

Claude Code é um agente que vive no terminal e trabalha como um par de programação. Ele tem **integração com o terminal**, faz **gestão de arquivos** (lê, edita, cria), e opera num **fluxo agentic**: recebe um objetivo, planeja, executa passos com ferramentas, verifica o resultado e itera. Os casos de uso típicos são refatorações, correção de bugs, criação de features e automação de tarefas repetitivas de engenharia. É, em essência, o Capítulo 8 inteiro encarnado numa ferramenta.

## Quando Usar Cada Versão

Não existe "a melhor ferramenta", existe a ferramenta certa para a tarefa. Uma matriz simples de decisão:

- **Exploração e prototipagem rápida** → Claude Web. Custo zero de setup, ótimo para descobrir se uma ideia tem pernas.
- **Trabalho local com seus arquivos e desenvolvimento** → Claude Desktop / Claude Code.
- **Aplicações em produção** → API. Você controla custo, latência, retries e escala.
- **Engenharia e scripting agêntico** → Claude Code.

A regra geral: quanto mais perto da exploração, mais perto da Web; quanto mais perto da produção e da automação, mais perto da API.

## Landscape Competitivo

Nenhuma ferramenta existe no vácuo. Conhecer os concorrentes ajuda a saber quando Claude é a escolha certa — e quando não é.

Entre os **LLMs gerais**, o **ChatGPT (OpenAI)** é o concorrente mais visível, com ecossistema amplo; suas fraquezas em relação a Claude tendem a aparecer em tarefas de raciocínio longo e em seguir instruções com precisão. O **Gemini (Google)** brilha em multimodalidade e na integração com o Google Workspace. O **Llama (Meta)** é open-weight — você pode rodar em infraestrutura própria, o que importa para quem tem requisitos de privacidade ou custo em escala. A **Mistral** é a alternativa europeia, forte em modelos pequenos e eficientes.

Entre as **ferramentas de coding**, o **GitHub Copilot** integra-se nativamente às IDEs e tem modelo de negócio por assinatura. O **Cursor** é um editor "AI-first". O **Continue.dev** é open-source e altamente customizável. O **JetBrains AI Assistant** integra-se ao ecossistema JetBrains.

O **posicionamento de Claude** se sustenta em três pilares: segurança, qualidade de raciocínio e um design de API limpo. Os trade-offs honestos são custo versus desempenho e velocidade versus qualidade — os modelos de ponta são mais caros e, em alto esforço, mais lentos. A hora de escolher Claude é quando você precisa de raciocínio complexo, de conteúdo longo e coerente, ou de garantias de segurança.

## Arquitetura Geral e Fluxo de Processamento

Vale a pena abrir o capô e ver o que acontece entre apertar Enter e receber a resposta. O caminho de uma requisição passa por etapas bem definidas.

Tudo começa com a **entrada do usuário**, que é **validada** (o formato está correto? os papéis de mensagem alternam corretamente?), **roteada** ao modelo certo, **processada** e devolvida como **output**. Dentro do "processada" mora a parte interessante.

Primeiro vem a **tokenização**: o texto vira tokens. Como você paga por token e a janela é contada em tokens, há um overhead inerente — formatação, tags, espaços, tudo conta. Em seguida, o **embedding** converte cada token num vetor numérico de alta dimensão; é assim que o modelo representa significado. A posição de cada token também é codificada (positional encoding), porque "cão morde homem" e "homem morde cão" têm os mesmos tokens em ordens diferentes.

A **inferência** propriamente dita passa esses vetores pelas camadas do modelo, usando o mecanismo de **attention** — que permite a cada token "olhar" para os outros e pesar quais são relevantes. É a etapa cara em computação e a principal fonte de latência. Depois vem a **geração**: o modelo prevê um token por vez, e cada token gerado realimenta a entrada para o próximo. É aqui que entram os controles de amostragem (ou, nos modelos novos, o raciocínio adaptativo) e as **stop sequences** que dizem quando parar. Por fim, o **output** decodifica os vetores de volta para texto, formata e entrega.

Dois conceitos de **tempo e custo** valem ouro na prática. O **first token latency** é quanto demora até o primeiro token aparecer — dominado pela etapa de processar a entrada inteira. A **token generation speed** é quão rápido os tokens saem depois disso. Por isso o **streaming** melhora tanto a experiência percebida: você vê o primeiro token cedo, mesmo que a resposta completa demore. E há a assimetria fundamental de custo: **tokens de saída custam várias vezes mais que tokens de entrada** (veja a tabela de preços acima). Otimizar uma aplicação quase sempre significa, antes de tudo, controlar o volume de tokens de saída.

## Referências

- Anthropic, *Documentação do Claude* — <https://docs.claude.com>
- Anthropic, *Models overview* (linhagem, contexto e preços) — <https://docs.claude.com/en/docs/about-claude/models/overview>
- Anthropic, *Model deprecations* (aposentadoria de modelos) — <https://docs.claude.com/en/docs/about-claude/model-deprecations>
- Bai et al., *Constitutional AI: Harmlessness from AI Feedback* (2022) — <https://arxiv.org/abs/2212.08073>
- Anthropic, *Claude's Constitution* — <https://www.anthropic.com/news/claudes-constitution>
