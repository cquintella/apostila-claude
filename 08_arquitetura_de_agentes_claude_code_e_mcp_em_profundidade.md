# Capítulo 8: Arquitetura de Agentes, Claude Code e MCP em Profundidade

Este capítulo desce ao nível em que Claude vira um _sistema_ que age: o loop agêntico, a orquestração de subagentes, a configuração do Claude Code, a saída estruturada confiável, o design de ferramentas e MCP, e a gestão de contexto sob pressão. É também a trilha exata cobrada na certificação **Claude Certified Architect – Foundations (CCA-F)** da Anthropic, e por isso fechamos o capítulo com o formato do exame e os erros que mais derrubam candidatos.

> [!info] As certificações e cursos da Anthropic — onde estudar
> A formação oficial vive na **Anthropic Academy** (<https://anthropic.skilljar.com>), com cursos gratuitos — *Claude 101*, *Claude Code 101*, *Building with the Claude API*, além de trilhas de MCP e de subagentes. A **certificação** de referência é a **Claude Certified Architect – Foundations (CCA-F)**, com página própria no portal de parceiros (<https://anthropic-partners.skilljar.com/claude-certified-architect-foundations-certification>). O ponto de entrada geral para materiais e guias é <https://www.anthropic.com/learn>. O que exatamente cai no exame — os cinco domínios e seus pesos — está detalhado no fim deste capítulo; como programas e provas mudam, trate a página oficial como a fonte de verdade.

## O que é um agente?

Antes de descer à arquitetura, vale desarmar uma palavra sobrecarregada. "Agente" é antigo e teve muitos sentidos: na economia, é quem age em nome de outro; na IA clássica, um **agente inteligente** é, na definição consagrada de Russell & Norvig, qualquer entidade que **percebe** o ambiente por sensores e **age** sobre ele por atuadores em busca de um objetivo; na aprendizagem por reforço, o agente é o que aprende por tentativa e erro maximizando recompensa. Nenhum desses sentidos é novo, e nenhum exige um LLM.

O que este capítulo chama de **agente** é o sentido recente e mais estreito: um **LLM que, dentro de um loop, decide quais ferramentas usar e quando parar** para cumprir um objetivo. A correspondência com a definição clássica é direta — a _percepção_ é ler o contexto e os resultados das ferramentas; a _ação_ é chamar uma ferramenta; o _objetivo_ é a tarefa dada. A novidade não é o conceito de agente; é o **motor** no centro do laço ser um modelo de linguagem. E convém distingui-lo desde já do **workflow** (Capítulo 5): no workflow, os passos são orquestrados por código que _você_ escreveu; no agente, é o **modelo** que dirige o próprio processo. Este capítulo é sobre construir esse laço de forma confiável.

## O Modelo Mental: uma API stateless e quatro camadas

No fundo de todo produto da Anthropic há uma única coisa: um modelo de linguagem atrás de um endpoint HTTP em `api.anthropic.com/v1/messages`. Você manda uma lista de mensagens, recebe uma resposta. Três fatos definem esse motor e não mudam: **o modelo não tem memória entre chamadas, não roda nenhum loop do lado da Anthropic, e não executa código ou ferramentas por conta própria**. Tudo o mais é uma camada de disciplina embrulhada em torno desse endpoint.

São quatro camadas que vale reconhecer pelo nome, porque cada uma resolve um problema diferente:

- **Claude API** — o endpoint HTTP cru mais os SDKs oficiais (Python e TypeScript) que o embrulham. É onde você controla tudo na mão.
- **Agent SDK** — o framework que a Anthropic publica _por cima_ da API. Ele roda o loop agêntico para você, gera subagentes e traz ferramentas embutidas, para você não reconstruir tudo do zero.
- **Claude Code** — o agente de terminal que sobe Claude dentro do seu repositório e lhe dá acesso ao sistema de arquivos através de ferramentas como `read`, `write`, `edit`, `bash`, `grep` e `glob`.
- **MCP (Model Context Protocol)** — o padrão aberto de como ferramentas e fontes de dados externas se expõem a qualquer modelo compatível, de modo que o mesmo conector de banco, wiki ou processador de pagamentos funcione nas quatro superfícies sem reescrever a cola.

Guardar esse mapa resolve metade dos problemas de arquitetura: quase toda decisão difícil é, no fundo, "qual dessas camadas é dona do comportamento que quebrou?".

## A Pilha da IA Agêntica: seis camadas de responsabilidade

O mapa das quatro camadas responde "onde meu código roda". Mas para _projetar_ um sistema agêntico — e é para isso que este capítulo caminha — falta um segundo mapa, que responde a uma pergunta diferente: **de que o meu agente precisa dar conta, do motor à governança?** É esse mapa que organiza o resto do capítulo: cada seção adiante (o loop, os subagentes, as ferramentas, a gestão de contexto, a segurança) preenche uma dessas camadas. Guardá-lo agora dá o fio condutor; sem ele, o capítulo pareceria uma lista solta de técnicas.

Não há ainda um "padrão ISO" da IA agêntica — o campo é jovem —, mas há referências amplamente aceitas que fundamentam esta visão: a definição clássica de agente inteligente de **Russell & Norvig** (percepção → ação com um objetivo), o artigo **ReAct** (Yao et al., 2022), que estabeleceu o laço de intercalar raciocínio e ação, e o guia **_Building Effective Agents_** da Anthropic (2024), que separa *workflows* de *agentes* e prega começar simples. A pilha abaixo é uma síntese prática dessas fontes, não um dogma.

As quatro camadas anteriores respondem a uma pergunta de _produto_: **onde o seu código roda** (no endpoint cru, no SDK, no terminal, num servidor MCP). Esta pilha responde a uma pergunta de _responsabilidade_: **quais funções uma aplicação agêntica de produção precisa cobrir, do modelo lá embaixo até a governança lá em cima**. Vale desenhá-la de cima para baixo, porque é assim que o valor flui: a governança define as regras, e cada camada abaixo executa uma fatia delas até chegar ao modelo, que é só o motor.

| Camada | Pergunta que responde | Onde já apareceu nesta apostila |
| --- | --- | --- |
| **Governance & Audit** | Isto é permitido, e ficou registrado? | Cap. 3 (auditoria, `/cost`), Cap. 10 (segurança, compliance) |
| **Evaluation Gates** | A saída é boa o bastante para seguir? | Cap. 8 (validação, verificação cética), Cap. 9 (falha é regra) |
| **Harness Orchestration** | Quando, onde e como o modelo age? | Cap. 8 (arnês, loop agêntico, subagentes) |
| **Tools / MCP** | Como o modelo toca o mundo externo? | Cap. 6 e Cap. 8 (design de ferramentas, MCP) |
| **Context Engineering** | O que entra na janela, e em que ordem? | Cap. 4 (prompts), Cap. 7 (cache, relevância) |
| **Model Layer** | Qual motor gera os tokens? | Cap. 1 (LLM, atenção), Cap. 2 (escolha do modelo por tarefa) |

O papel de cada uma, de cima para baixo:

- **Governance & Audit (governança e auditoria)** — a camada mais externa. Define _o que é permitido_ e garante que _tudo fique registrado_. É onde vivem as políticas de uso, os limites de gasto, o controle de quais dados o agente pode tocar, e a trilha de auditoria que prova, depois, o que o agente fez e por quê. Sem ela, você tem um agente que talvez funcione, mas que ninguém consegue aprovar para produção regulada — porque não há como responder "quem autorizou isso?". Materializa-se em hooks que registram cada ação, relatórios de conformidade direcionados a canais externos e uma chave de API por projeto para separar contas (Cap. 3), somada às exigências setoriais de compliance (Cap. 10).

- **Evaluation Gates (portões de avaliação)** — a camada que decide se a saída de um passo é boa o bastante para o próximo passo prosseguir. É o "cético" do sistema: testes que precisam passar, um schema que precisa validar, um subagente verificador que reprova trabalho complacente, um limiar de confiança que dispara escalonamento a um humano. O princípio de arquitetura é que _só evidência conta como progresso_ — uma suíte de testes verde, não a autodeclaração do modelo de que "terminei". Sem portões, o sistema produz "lixo confiante": passa adiante uma resposta errada com a mesma naturalidade de uma certa.

- **Harness Orchestration (orquestração do arnês)** — a camada que governa _quando, onde e como_ o modelo age. É o arnês (detalhado adiante neste capítulo): o loop agêntico que inspeciona `stop_reason` e anexa `tool_result`, a coordenação coordenador/subagente, os hooks determinísticos, o estado persistido em disco, o ciclo de vida da sessão. É aqui que a inteligência do modelo vira comportamento confiável — "o modelo é inteligente; o arnês é o que o torna confiável".

- **Tools / MCP (ferramentas e MCP)** — a camada pela qual o modelo, que sozinho só produz texto, toca o mundo externo: lê arquivos, consulta bancos, chama APIs, dispara pagamentos. O MCP padroniza esse contato para que o mesmo conector sirva a qualquer superfície. A qualidade desta camada mora nos detalhes já vistos: descrições de ferramenta escritas como doc de API, respostas de erro estruturadas e o transporte certo (STDIO local, SSE remoto).

- **Context Engineering (engenharia de contexto)** — a camada que decide _o que entra na janela do modelo e em que ordem_. É o que separa um prompt que funciona de um que desperdiça orçamento ou "se perde no meio": a estrutura papel/tarefa/contexto/formato (Cap. 4), o ordenamento do estável (cacheável) ao volátil, a reancoragem de fatos duráveis num _case block_ no fim da janela, o uso de subagentes como filtro de relevância (Cap. 7). O modelo só é tão bom quanto o contexto que você monta para ele.

- **Model Layer (camada do modelo)** — a base da pilha: o LLM stateless que, dado um contexto, gera os tokens seguintes. É o motor — poderoso, mas cego ao resto do sistema: não tem memória, não roda loop, não executa ferramentas. Toda a decisão que sobra aqui é _qual_ modelo usar por tarefa (Haiku para classificar barato, Opus ou Fable para o trabalho agêntico difícil). Tudo o mais nesta pilha existe justamente porque o modelo, sozinho, é só isto.

A lição prática espelha a das quatro camadas: diante de um agente que falha em produção, pergunte _qual dessas seis responsabilidades ficou descoberta_. Um agente que dá resposta errada com confiança não tem um problema de modelo — tem um **Evaluation Gate** faltando. Um agente que ninguém aprova para produção não tem problema de código — tem a camada de **Governance** vazia. Quase todo buraco de arquitetura é uma dessas seis camadas que ninguém foi dono.

## O Loop Agêntico

Um **loop**, em qualquer sistema, é um trecho que se repete até uma **condição de parada** — e essa condição não é opcional: um loop sem saída é um travamento. A ideia de pôr um LLM dentro de um laço, intercalando **raciocínio** e **ação** (pensar, chamar uma ferramenta, ler o resultado, decidir o próximo passo), foi proposta e popularizada pelo artigo **ReAct** (Yao et al., 2022). A vantagem sobre uma única chamada é direta: o modelo pode **agir, ver o resultado e se corrigir**, em vez de tentar acertar tudo de uma vez, no escuro. Use um loop quando a tarefa exige passos que dependem de resultados intermediários (buscar, ler, testar, ajustar); não o use para um pedido único e fechado, onde ele só adiciona custo e risco.

### A mecânica do laço

Como o modelo é stateless e não roda loop nenhum, **o seu código é o loop**. Quando você chama `messages.create` passando uma lista de ferramentas que Claude pode usar, a resposta volta com um campo decisivo: o **`stop_reason`**. Dois valores governam o que seu código faz em seguida:

- **`end_turn`** — Claude terminou de pensar e o texto da resposta é a resposta final.
- **`tool_use`** — Claude pausou no meio do raciocínio e está pedindo que o _seu_ código execute uma função por ele.

No caso de `tool_use`, o ciclo é sempre o mesmo: você lê o nome e os argumentos da função na resposta, **executa** essa função localmente, **anexa** o resultado de volta ao histórico como uma mensagem de `tool_result`, e chama `messages.create` de novo com o histórico atualizado. Claude retoma de onde parou e ou chama outra ferramenta ou finaliza. Esse ciclo — **chamar → inspecionar → executar → anexar → chamar** — é o agente inteiro.

Os dois erros clássicos têm assinatura inconfundível. Se você **esquece de inspecionar** o `stop_reason`, o agente fica mudo depois da primeira chamada de ferramenta. Se você **esquece de anexar** o `tool_result`, o modelo perde o resultado e tenta a mesma ferramenta de novo, num laço. Diante de um trecho de código que "para sozinho" ou "repete a ferramenta", procure qual desses dois passos faltou.

### Os dois níveis do loop: interno e externo

"Loop" é uma palavra sobrecarregada em sistemas agênticos, e confundir os dois sentidos gera muito debate estéril. Vale separá-los:

- **O loop interno (agêntico)** — é o ciclo que acabamos de descrever: **chamar → inspecionar → executar → anexar → chamar**, repetido _dentro de uma única tarefa_ até o modelo devolver `end_turn`. Cada volta consome tokens e uma chamada de API. Ele existe porque o modelo, sozinho, não age: sem esse laço, Claude pede uma ferramenta e trava. É o loop que _você_ escreve (ou que o Agent SDK escreve por você).
- **O loop externo (autônomo/operacional)** — é o ciclo que roda _entre tarefas ou entre sessões_: definir objetivo → agir → verificar → gravar memória → decidir se continua. É o que mantém um agente trabalhando sozinho por horas, retomando de onde parou. Este é detalhado adiante, em **O Loop Operacional** (na seção de Engenharia de Arnês), e é o que ferramentas como agendadores e comandos de repetição orquestram.

A regra de ouro que os une: **um loop só é seguro quando tem uma condição de parada que não depende da boa vontade do modelo**. O loop interno para em `end_turn`; o externo para quando o estado em disco diz que o objetivo foi atingido.

### Usando loops de forma apropriada

Um loop é uma faca de dois gumes: dá autonomia, mas também é onde o custo e os efeitos colaterais saem do controle. Cinco disciplinas mantêm o laço sob rédea:

1. **Sempre imponha um teto de iterações.** Todo loop — interno ou externo — precisa de um contador máximo (`max_iterations`) que o interrompe mesmo que o modelo nunca declare "pronto". É a rede de segurança contra o gasto que dobra a cada volta sem ninguém olhando.
2. **A condição de parada mora no estado, não no prompt.** O objetivo deve existir como arquivo físico em disco que o loop relê a cada volta, não só na janela do agente. Assim o laço decide continuar ou parar lendo evidência concreta (testes passando, feature marcada como pronta), e não uma autodeclaração otimista.
3. **Feche cada volta com verificação.** Um loop sem um portão de avaliação (a camada _Evaluation Gates_) não converge — só empilha erros com mais confiança a cada iteração. Depois de agir, verifique de modo cético, de preferência num subagente de contexto limpo.
4. **Torne os passos idempotentes ou rastreie efeitos colaterais.** Se uma volta grava no banco ou dispara e-mails, refazê-la os dispara de novo. Por isso a recuperação certa é o **retry direcionado só do que falhou**, nunca rerodar tudo (o mesmo princípio da orquestração multiagente).
5. **Nem toda tarefa merece um loop.** Loop é para trabalho aberto, iterativo, com verificação a cada passo. Para um pedido único e determinístico, uma só chamada é mais barata e previsível — enrolar isso num laço só adiciona custo e superfície de falha.

Os três modos de falha de um loop mal desenhado — _lixo confiante_, _podridão de contexto_ e os _loops inúteis (Ralph Wiggum)_, em que o agente refaz sem fim um passo já concluído porque o progresso não foi persistido — são detalhados em **O Loop Operacional**. Todos os três se resumem a violar uma das cinco disciplinas acima.

## Orquestração Multiagente

Quando um único Claude não basta, entra o padrão coordenador/subagente. Um Claude de topo — o **coordenador** — quebra o trabalho em pedaços e gera **subagentes** menores e de escopo restrito para cada pedaço. No Agent SDK, o coordenador faz isso através de uma função embutida, a **ferramenta `task`**.

O motivo do padrão é **orçamento de contexto**, não velocidade. Um coordenador que precise ler dez arquivos, rodar cinco buscas e redigir um resumo encheria a própria janela de contexto antes de terminar. Ao delegar cada leitura a um subagente, o coordenador recebe de volta apenas o resumo compacto de cada um e mantém o próprio contexto enxuto. Isso conecta diretamente ao filtro de relevância do Capítulo 7: subagente é relevância aplicada à arquitetura.

Há dois estilos de decomposição que convém distinguir:

- **Decomposição sequencial** — o coordenador termina o subagente 1 antes de gerar o 2. É a escolha certa quando cada passo depende da saída do anterior.
- **Decomposição adaptativa** — o coordenador escolhe o próximo subagente com base no que voltou. É a escolha certa para trabalho aberto e exploratório (investigar a estratégia de preços de um concorrente, por exemplo).

E quando um subagente falha no meio, a recuperação certa é quase sempre um **retry direcionado apenas do que falhou** — nunca rerodar todos. Os subagentes bem-sucedidos podem já ter gravado linhas num banco ou disparado e-mails; rerodá-los dispararia esses efeitos colaterais uma segunda vez. (Este é o mesmo princípio de "falha é regra" do Capítulo 9, aplicado a agentes.)

> [!example] Um loop útil: consertar testes até passarem
> O exemplo canônico de laço bem desenhado é a **correção de testes**. O objetivo — "a suíte inteira passa" — vive em disco (o próprio resultado dos testes), não na cabeça do modelo. A cada volta: (1) rodar os testes; (2) se todos passam, **`end_turn`** — parada por evidência concreta; (3) se algum falha, ler a mensagem de erro, editar o código e voltar ao passo 1. As cinco disciplinas aparecem todas: a **condição de parada** é objetiva (testes verdes, não "acho que resolvi"); há um **teto de iterações** para não girar para sempre se um teste for impossível; cada volta **verifica** (rodar o teste é o portão); e o trabalho é naturalmente **idempotente**. Troque "testes" por "lint limpo", "build sem erro" ou "todos os itens do `feature_list.json` marcados" e você tem o mesmo esqueleto — é o loop operacional detalhado no fim do capítulo.

## Configuração do Claude Code

O Claude Code roda Claude direto dentro do seu repositório e, por padrão, **cada nova sessão começa sem nenhuma memória do projeto**. O arquivo **`CLAUDE.md`** é o que conserta isso. Quando o Claude Code inicia, ele sobe a árvore de diretórios a partir da pasta atual, encontra todo arquivo `CLAUDE.md` no caminho e os costura no _system prompt_ da sessão.

O Capítulo 3 já apresentou esses arquivos pelo lado operacional; aqui os revisitamos pelo ângulo que **o exame cobra** — a hierarquia de três níveis e a confusão entre eles, que é a pegadinha campeã:

- **Nível de usuário** (`~/.claude/CLAUDE.md`) — aplica-se a todo projeto na sua máquina e nunca entra no controle de versão. O que está aqui é só seu.
- **Nível de projeto** (`CLAUDE.md` na raiz do repositório) — é commitado e viaja com cada clone. É onde moram as convenções do time ("todos usam `bun` em vez de `npm`").
- **Nível de diretório** (ex.: `packages/api/CLAUDE.md`) — regras que só valem quando Claude mexe naquela subpasta específica.

A pegadinha canônica: onde deve ficar a regra "todo mundo roda o lint antes do commit"? No **arquivo de projeto, na raiz do repositório**. Pôr no arquivo de usuário significa que o clone do seu colega nunca verá a regra.

E o **README**? Claude _não_ carrega o `README.md` automaticamente no system prompt — só o `CLAUDE.md` é costurado a cada sessão. O README entra no contexto quando o agente decide lê-lo (porque a tarefa o levou até ele) ou quando você o menciona. Se há informação que o agente **deve sempre** ter, ela pertence ao `CLAUDE.md`, não ao README — ou você aponta uma para a outra. É comum um `CLAUDE.md` enxuto que diz "para o panorama do projeto, veja o `README.md`", deixando o agente buscá-lo quando precisar.

### Slash commands

Um **slash command** é um prompt reutilizável que você invoca digitando uma barra (`/review`, por exemplo). Você define um deles soltando um arquivo Markdown em `.claude/commands`. O corpo do arquivo é o prompt injetado quando o comando roda; o **front matter YAML** no topo é onde você restringe o comportamento. O campo **`allowed-tools`** limita quais ferramentas o comando pode chamar — um comando de revisão pode ser travado em ferramentas somente-leitura (`read`, `grep`), sem acesso a `write` ou `bash`. O campo **`argument-hint`** é o placeholder mostrado no seletor, documentando que argumento o comando espera.

> [!tip] Comandos essenciais — cheatsheet
> Os comandos embutidos mais usados (`/init`, `/config`, `/status`, `/cost`, `/compact`, `/clear`, `/permissions`, `/hooks`, `/model`, `/output-style`) estão reunidos numa tabela de referência no **Capítulo 3**. A lista completa e sempre atualizada fica na documentação oficial: [slash commands](https://docs.claude.com/en/docs/claude-code/slash-commands) e a [referência de CLI](https://docs.claude.com/en/docs/claude-code/cli-reference).

### Execução headless em CI/CD

O Claude Code aceita a flag **`-p`**, que o roda de forma não interativa com um prompt embutido. Combinada a um _schema_ de saída estruturada, ela permite que o seu pipeline peça a Claude para revisar um pull request e devolver um objeto JSON com categorias de violação nomeadas. A lição de arquitetura que o exame repete: o pipeline deve **falhar apenas em categorias que o schema nomeia explicitamente** (`security_violation`, `breaking_api_change`), nunca quando Claude apenas "expressa uma preocupação vaga". Uma revisão que falha por vaguidão vira ruído que o time aprende a ignorar em uma semana — e uma revisão ignorada é pior que revisão nenhuma.

## Saída Estruturada Confiável

O Capítulo 4 mostrou que pedir JSON "na prosa" funciona na maior parte das vezes — até a manhã em que o modelo prefixa um "Claro, aqui está o seu JSON" e o seu parser explode em produção. A forma confiável é **declarar o formato de saída como uma ferramenta**: você define uma tool cujo campo **`input_schema`** descreve exatamente o JSON desejado, passa essa tool no array de ferramentas e usa **`tool_choice`** para forçar o modelo a chamá-la. Em vez de texto livre, o modelo devolve um bloco `tool_use` cujo campo `input` já é o JSON em conformidade com o schema. Funciona muito melhor porque a Anthropic treinou Claude intensamente contra schemas de ferramenta — a taxa de conformidade em `tool_use` é bem mais alta do que em instruções de formato no texto.

Ainda assim, schemas às vezes derivam, e o padrão para isso é um **loop de validação e retry**. A cada resposta, você passa o JSON pelo seu validador. Quando falha, **não retente às cegas**: pegue a mensagem de erro do validador e anexe-a ao histórico como um turno de usuário ("sua resposta anterior falhou na validação com este erro específico; devolva uma versão corrigida") e chame o modelo de novo. Agora o modelo tem em contexto a própria saída anterior e o campo exato que quebrou — a taxa de correção na segunda tentativa fica acima de 90% na prática.

Para volume, há a **Message Batches API**: você submete milhares de prompts independentes e recolhe os resultados em até 24 horas, a 50% do custo por token das chamadas em tempo real. Diante de "classificar 100 mil documentos da noite para o dia", a escolha é batch — sempre que o trabalho é offline, a latência pode ser medida em horas e os prompts são independentes (a ordem não importa).

## Design de Ferramentas e MCP

Recapitulando em uma frase (o tema foi apresentado no Capítulo 6): o **MCP — Model Context Protocol** é um **padrão aberto** de como ferramentas e fontes de dados externas se expõem ao modelo, como um "USB-C" para dados. Antes do MCP, cada integração era um _wrapper_ específico de um modelo; com ele, o modelo fala um único protocolo e qualquer servidor compatível — banco, wiki, processador de pagamentos, sua API interna — pluga sem mudar o código do modelo. Três pontos merecem profundidade.

> [!example] O ganho concreto do MCP: N×M vira N+M
> Suponha 3 assistentes (Claude Code, Codex, Cursor) que precisam falar com 3 sistemas (Postgres, GitHub, Slack). **Sem** MCP, cada dupla exige um conector próprio: 3×3 = **9 integrações** para escrever e manter. **Com** MCP, você escreve **um** servidor por sistema (3) e cada assistente fala o protocolo (3): **3+3 = 6**, e a conta melhora quanto mais peças você tem. Na prática, isso significa que o mesmo servidor MCP de Postgres que você escreveu para o Claude Code funciona no Cursor amanhã sem uma linha nova de cola — é o mesmo princípio de "separar o fluxo do fornecedor" que reaparece na seção de arnês.

**Descrições de ferramenta decidem a escolha da ferramenta.** No momento da decisão, o modelo só enxerga, de cada tool, o nome, a descrição e o `input_schema`. Se você publica três ferramentas `get_user`, `lookup_user` e `find_customer`, todas descritas como "retorna informações do usuário", o modelo escolhe praticamente no chute — e o exame pontua isso como um **defeito de schema**, não como falha do modelo. Escreva descrições como quem escreve doc de API: uma frase sobre o que a ferramenta faz, uma frase sobre quando usar esta e não as irmãs (a regra de desambiguação), um exemplo de invocação com argumentos realistas, e a lista de condições de erro que ela pode devolver.

**Respostas de erro estruturadas.** Quando uma ferramenta falha, o formato certo é um objeto JSON em que o modelo possa ramificar: um booleano `is_error` (para saber que caiu no caminho de erro), uma string `category` (`rate_limited`, `not_found`…) para escolher a recuperação pelo nome, um booleano `retryable` e, quando couber, um inteiro `retry_after_ms`. Com essa forma, o modelo escreve um plano de recuperação limpo num passo só. Sem ela, recebe uma string de exceção crua e ou desiste ou entra num loop de retry infinito que você só descobre no log às 3 da manhã.

**Seleção de transporte.** O MCP suporta dois transportes. O **STDIO** roda o servidor na mesma árvore de processos do cliente, por entrada/saída padrão: zero latência de rede, zero overhead de autenticação. O **SSE** (server-sent events) roda o servidor em outra máquina e o cliente conecta por HTTP, o que adiciona idas e voltas de rede e obriga a desenhar autenticação. A regra é direta: use **STDIO sempre que o servidor puder viver na mesma máquina** do cliente; use **SSE só quando o servidor precisar morar em outro lugar** (um conector central de banco servindo vários usuários a partir de um host compartilhado, por exemplo). Escolher SSE para um servidor em `localhost` é uma armadilha clássica.

## Gestão de Contexto e Confiabilidade

**O efeito "perdido no meio" (lost in the middle).** É um achado empírico que se mantém em todas as famílias de modelo, Claude incluído: quando você enche a janela com uma conversa ou documento longo, o modelo presta muita atenção ao que está bem no **começo** e bem no **fim**, e quase nenhuma ao que está no **meio**. O sintoma de exame: um agente de suporte conversou por 40 turnos e de repente "esquece" o número de conta dado no turno 3. A correção certa é identificar os **fatos duráveis** (números de conta, IDs de pedido, valores de reembolso), copiá-los para um pequeno bloco estruturado — muitas vezes chamado de **case block** — e **reancorar esse bloco no fim do contexto a cada turno**, para que a atenção do modelo sempre caia sobre ele. A tentação é pedir ao modelo que _resuma_ a conversa inteira — e esse é exatamente o distrator: a sumarização comprime com heurísticas com perda e silenciosamente derruba números e IDs.

**Prompt caching: o que cachear.** O mecanismo está detalhado no [Capítulo 7](07_otimizacao_e_performance.md) (prefixo exato, ~90% de desconto na leitura, TTL de 5 min ou 1 h); para não repetir, fica aqui só o ângulo arquitetural — _o que_ vale cachear. O **system prompt** e os **exemplos few-shot** valem, porque são idênticos em toda chamada; o **turno do usuário no fim não vale**, porque muda sempre e cachear só queima orçamento. A regra de layout decorre disso: ordene o prompt do **estável** (começo, cacheado) ao **volátil** (fim).

**Escalonamento: agentes que sabem quando não sabem.** Um agente que responde com confiança a uma pergunta que deveria ter escalado é uma falha _pior_ do que um agente que escala cedo demais — o primeiro entrega respostas erradas ao cliente; o segundo só custa alguns tíquetes de revisão humana, e essa é uma troca que você aceita o dia inteiro. O padrão a reconhecer é uma **checagem explícita de confiança** ao fim do raciocínio do agente, com dois gatilhos: ou a confiança declarada cai abaixo de um limiar, ou o modelo detecta uma ambiguidade que não consegue resolver pelo histórico. Qualquer um dos gatilhos passa a conversa a um humano, junto de um resumo estruturado do que o agente sabia, do que tentou e de onde travou.

## O Exame CCA-F: Formato e os Seis Cenários

A **Claude Certified Architect – Foundations** é a primeira certificação oficial da Anthropic (lançada em março de 2026), aplicada online com supervisão (proctored) pela Anthropic Academy. O formato: **60 questões** de múltipla escolha baseadas em cenário, **120 minutos** (cerca de 2 minutos por questão), e nota de corte de **720/1000** (em torno de 75%). As questões não testam decoreba: cada uma joga você dentro de um sistema em funcionamento — em geral já quebrando — e pergunta qual decisão arquitetural é a certa naquelas circunstâncias.

Os **cinco domínios** e seus pesos:

| Domínio | Peso | Onde mora |
| --- | --- | --- |
| 1. Arquitetura agêntica e orquestração | 27% | API e Agent SDK |
| 2. Configuração e workflows do Claude Code | 20% | Claude Code |
| 3. Prompt engineering e saída estruturada | 20% | API |
| 4. Design de ferramentas e integração MCP | 18% | MCP |
| 5. Gestão de contexto e confiabilidade | 15% | todo o sistema |

A Anthropic escreveu **seis cenários de produção**; o sistema serve **quatro ao acaso**, e todas as suas 60 questões se ancoram nesses quatro. Você estuda os seis porque não escolhe quais cairão: (1) **agente de suporte ao cliente** — onde caem escalonamento, retries e falha parcial; (2) **integração de time no Claude Code** — hierarquia de `CLAUDE.md` e slash commands; (3) **pipeline de pesquisa multiagente** — orquestração coordenador/subagente e estouro de contexto; (4) **ferramentas de produtividade do dev** — ferramentas embutidas (`read`/`write`/`edit`/`bash`/ `grep`/`glob`) e quando preferir um servidor MCP; (5) **Claude Code em CI/CD** — invocação headless, saída estruturada e minimização de falsos positivos; (6) **extração de dados estruturados** — JSON via schema e Message Batches.

### Os cinco distratores que mais derrubam

Num exame de múltipla escolha baseado em cenário, a resposta errada raramente é absurda — ela é uma opção **plausível** desenhada para atrair. Essas iscas são os *distratores*, e o banco de questões da CCA-F reusa o mesmo punhado deles. Reconhecê-los de imediato — e saber qual é a resposta certa que cada um imita — vale pontos:

- **Adjetivo vago** — qualquer opção que enfia "cuidadoso" ou "minucioso" num prompt para resolver falso positivo. A resposta certa troca o adjetivo por uma **lista de regras categóricas** verificáveis uma a uma.
- **Sumarização** — comprimir uma conversa longa num cenário que carrega IDs e valores. A resposta certa extrai os fatos duráveis para um **case block** no fim da janela.
- **Retry geral** — rerodar todos os subagentes depois que um falhou, quando os outros já executaram efeitos colaterais. A resposta certa **retenta só o que falhou**.
- **`CLAUDE.md` de usuário** — pôr configuração compartilhada do time no arquivo de usuário (que nunca chega ao controle de versão). A resposta certa usa o **arquivo de projeto na raiz**.
- **SSE em localhost** — escolher transporte SSE para um servidor que roda na mesma máquina do cliente. A resposta certa é **STDIO**.

## Engenharia de Arnês (Harness Engineering)

> [!note] Uma nota de tradução: por que "arnês" e não "sela"
> *Harness* é o **arreio completo** — o conjunto de tiras e conexões que controla e conduz o animal (ou, em escalada, o que prende e segura você com segurança; em eletrônica, o "chicote" que liga tudo). A **sela** é só o assento. A metáfora aqui é justamente a do conjunto que **governa e sustenta** o modelo — não um assento, mas todo o aparato de controle em volta dele. Por isso "arnês" traduz melhor que "sela". Na prática, muitos textos deixam o termo em inglês, *harness*; usamos os dois de forma intercambiável.

Há uma frase que resume esta seção: *"o modelo é inteligente; o arnês é o que o torna confiável"*. A maioria dos desenvolvedores tem dificuldade com o loop de execução autônoma (o ciclo do agente), mas o problema raramente é o loop em si — é que a infraestrutura (o *harness*, ou arnês) ao redor dele não foi bem projetada. **Engenharia de arnês** é a disciplina de desenhar esse ambiente: não se trata de escrever prompts melhores, e sim de construir o sistema dentro do qual o modelo opera. Se você abrir o diretório `.claude/` de qualquer projeto maduro, verá duas camadas que precisam trabalhar em harmonia.

> *Figura — Anatomia de um "Loopkit Vault": o diretório `.claude/` (contrato, permissões, hooks, subagentes, skills, MCP) e os arquivos de estado na raiz do projeto. A estrutura é destrinchada a seguir.*

### Anatomia de um Arnês: o Vault do Projeto

A figura acima ilustra a configuração de um "Vault" (aqui chamado *Loopkit Vault*) para projetos com o Claude. Ela centraliza as regras de operação, as ferramentas, os ganchos de automação (hooks) e os agentes auxiliares que guiam o comportamento do LLM. O que cada arquivo e diretório faz:

#### Diretório `.claude/` (o arnês principal)
O diretório de "arnês" que envolve a configuração específica do Claude para o projeto.
*   **`CLAUDE.md`** (*contrato / regras de operação*): o contrato principal de como o Claude deve se comportar no repositório — diretrizes de estilo, regras de arquitetura e limites.
*   **`settings.json`** (*permissões + hooks*): estipula as permissões concedidas ao ambiente e mapeia os gatilhos (hooks).
*   **`hooks/`** (*reflexos*): scripts disparados automaticamente ao longo do ciclo de interação do modelo.
    *   **`pre-tool-use.sh`** (*barrar chamadas ruins*): roda antes de o modelo invocar uma ferramenta; atua como "guarda" contra chamadas perigosas ou malformadas.
    *   **`post-tool-use.sh`** (*formatar + registrar*): roda após o uso de uma ferramenta, para formatar o resultado e registrar logs.
    *   **`stop.sh`** (*saída limpa*): garante encerramento seguro e limpeza de processos ao fim da sessão.
*   **`agents/`** (*subagentes em contexto novo*): definições de subagentes com contextos "frescos", para delegar tarefas sem poluir a janela do agente principal.
    *   **`verifier.md`** (*fiscal das notas de turno*): exemplo de subagente configurado para verificar a qualidade e a integridade do que foi feito.
*   **`skills/`** (*memória muscular*): biblioteca de habilidades reutilizáveis. Melhor três habilidades focadas e muito usadas do que 50 genéricas que só consomem tokens.
    *   **`agent-llm/`**: padrões e ferramentas de interação com o modelo.
    *   **`debug/`**: scripts e guias para triagem e reprodução de bugs.
    *   **`security/`**: auditoria de segurança e ameaças no código.
    *   **`frontend/`**: padrões de front-end (React, acessibilidade).
    *   **`testing/`**: automação de testes em navegador e frameworks como o Jest.
    *   **`refactor/`**: processos para refatorar código com segurança.
    *   **`docs/`**: gatilhos e modelos para gerar READMEs e documentação de APIs.
    *   **`data/`**: processamento de dados via SQL e Pandas.
    *   **`git-ops/`**: automação de Git (revisão de PRs, rebases).
*   **`.mcp.json`** (*ferramentas MCP registradas*): onde as integrações do Model Context Protocol estão registradas.

#### Arquivos na raiz do projeto
*   **`MEMORY.md`** (*log de turnos entre sessões*): registro de memória que sobrevive de uma sessão para a outra, preservando o contexto global do progresso.
*   **`run.sh`** (*o executor do loop*): o script orquestrador que inicia e mantém rodando o ciclo principal de execução.
*   **`install.sh`** (*bootstrap em uma linha*): script rápido de instalação (via `curl`) que baixa as dependências do ambiente.
*   **`README.md`** (*apresentação do projeto*): documento com as orientações iniciais.

### Um script para montar o scaffold

Para não criar essa estrutura à mão toda vez, um `bootstrap.sh` a monta de uma vez. Ele cria os diretórios, os arquivos-âncora vazios e stubs mínimos nos hooks — um ponto de partida que você depois preenche:

```bash
#!/usr/bin/env bash
set -euo pipefail

# Diretórios do arnês
mkdir -p .claude/{hooks,agents,skills/{debug,security,testing,git-ops}}

# Contrato e configuração
[ -f .claude/CLAUDE.md ]     || printf '# Contrato do projeto\n\n## Comandos\n\n## Convenções\n\n## Nunca\n' > .claude/CLAUDE.md
[ -f .claude/settings.json ] || printf '{\n  "permissions": { "allow": [], "deny": ["Bash(rm -rf:*)"] }\n}\n' > .claude/settings.json
[ -f .claude/.mcp.json ]     || printf '{ "mcpServers": {} }\n' > .claude/.mcp.json

# Hooks (reflexos determinísticos) — stubs executáveis
for h in pre-tool-use post-tool-use stop; do
  f=".claude/hooks/${h}.sh"
  [ -f "$f" ] || printf '#!/usr/bin/env bash\n# %s hook\nexit 0\n' "$h" > "$f"
  chmod +x "$f"
done

# Subagente verificador de exemplo
[ -f .claude/agents/verifier.md ] || printf -- '---\nname: verifier\n---\nVerifique de modo cético o trabalho feito: rode os testes e o lint e reprove se algo falhar.\n' > .claude/agents/verifier.md

# Estado e memória, na raiz
[ -f MEMORY.md ] || printf '# Memória entre sessões\n' > MEMORY.md
[ -f run.sh ]    || { printf '#!/usr/bin/env bash\n# executor do loop\n' > run.sh; chmod +x run.sh; }

echo "Scaffold do arnês criado em .claude/"
```

Ajuste as skills e os hooks ao seu fluxo — o valor não está no esqueleto, mas em preenchê-lo com as três ou quatro peças que você de fato usa.

### Duas Camadas: o Arnês e o Loop

1.  **A infraestrutura (o arnês):** é a pasta `.claude/`. Não muda entre execuções. É como a cozinha de um restaurante.
2.  **O loop:** é o que roda dentro dela — o objetivo, a ação, a verificação, a gravação de memória e a decisão de continuar ou parar. É a receita sendo executada.

Uma cozinha sem receita é inútil, e uma receita sem cozinha não tem como ser executada. Tratar tudo como uma "configuração de agente" única faz você perder tempo tentando arrumar um prompt (problema de loop) quando, na verdade, faltava uma permissão de sistema (problema de arnês).

### O que é Engenharia de Arnês

Formalizando: o arnês governa *quando, onde e como* o modelo age, e costuma ser descrito como cinco subsistemas — os mesmos que você reconhece no Vault acima:

1.  **Instruções** — divulgação progressiva (um mapa, não uma enciclopédia), distribuída em arquivos legíveis como o `CLAUDE.md`.
2.  **Estado** — progresso persistido em disco, que sobrevive entre sessões (`MEMORY.md`, um `claude-progress.md`).
3.  **Verificação** — testes, lint, checagem de tipos; *"só uma suíte de testes passando conta como evidência"*.
4.  **Escopo** — uma funcionalidade por vez, com definição explícita de "pronto" (por exemplo, um `feature_list.json`).
5.  **Ciclo de vida da sessão** — inicializar no começo (um `init.sh` que verifica o ambiente) e deixar estado limpo no fim.

Sem esses subsistemas, o loop sofre os modos de falha clássicos: perda de contexto entre sessões, declarações prematuras de "terminei", *scope creep* com features pela metade, verificação pulada e retrabalho constante por falta de memória do que já foi feito.

### A Infraestrutura (Arnês) — Arquivo por Arquivo

A infraestrutura dita o que o loop tem permissão para fazer em cada iteração:
*   **`CLAUDE.md`**: é o contexto estático de toda sessão. Deve conter o escopo do projeto, comandos válidos e o que o agente *nunca* deve fazer. O segredo é evitar inchaço: mantenha-o enxuto (uma regra de bolso comum é menos de ~300 linhas), ou a eficiência despenca.
*   **`settings.json`**: define as permissões (`allow`/`deny`). Você pode liberar comandos seguros de leitura (como `ls` ou `cat`) sem pedir confirmação toda vez, melhorando a fluidez, enquanto bloqueia ações destrutivas.
*   **`hooks/`**: scripts determinísticos executados antes (`PreToolUse`) ou depois (`PostToolUse`) das ferramentas. Exemplo clássico: rodar um formatador automático após cada edição. Sem hooks, cada execução do LLM é imprevisível.
*   **`agents/` (subagentes)**: a arquitetura ideal isola tarefas complexas, como revisão de código, em subagentes que rodam em contextos novos e "limpos". Um agente construtor que revisa o próprio trabalho é complacente; jogar a revisão para um subagente resolve esse viés.
*   **`skills/`**: habilidades carregadas em contexto só quando o gatilho as ativa. Melhor três habilidades focadas e muito usadas do que 50 genéricas que apenas consomem tokens.
*   **`MCP` (`.mcp.json`)**: acesso a servidores de contexto (como o GitHub) para conectar o agente ao mundo exterior.
*   **Estado e memória (`MEMORY.md` e `vault/`)**: a memória guarda decisões recentes e preferências e deve ser podada com frequência. O `vault/` guarda as "leis" imutáveis do projeto. Tratar a memória como algo que só cresce causa a "podridão de contexto" (*context rot*).

### O Loop Operacional

Com a fundação criada, o ciclo tem 5 etapas — e vulnerabilidades:
1.  **Especificação do objetivo (goal spec):** o alvo (ex.: um arquivo `PROMPT.md`) não deve ficar só na memória curta (janela) do agente, e sim existir como arquivo físico em disco, que ele relê e onde grava o progresso.
2.  **Planejar, agir, verificar:** depois de planejar e codificar, um subagente precisa verificar de modo cético. Eliminar a verificação faz o modelo dobrar a aposta em erros antigos a cada iteração.
3.  **Desdobramento (fan-out de subagentes):** se a tarefa exige pesquisar 10 arquivos grandes de forma independente, divida em subagentes rodando em paralelo em vez de estourar um único contexto com a soma de todos.
4.  **Agendamento (scheduler):** scripts locais (como cron jobs) mantêm o agente rodando validações contínuas sem você no controle. O agendador deve ser "burro": a decisão vem do loop lendo o estado atual do disco.
5.  **Os 3 modos de falha frequentes:**
    *   *Lixo confiante (confident garbage)*: falta uma camada de verificação rigorosa — testes passam, mas resolvendo a coisa errada.
    *   *Podridão de contexto (context rot)*: acumular centenas de milhares de tokens desnecessários faz o LLM ignorar diretrizes finas (o "perdido no meio").
    *   *Loops inúteis (Ralph Wiggum loops)*: o agente trava re-planejando e refazendo um passo que acabou de concluir, porque o progresso não foi persistido em disco.

### Aprendizado Contínuo e a Ferramenta Pi

Duas fronteiras dão o rumo da engenharia de arnês. A primeira é o **aprendizado contínuo de agentes**: em vez de melhorar só os pesos do modelo (caro e lento), melhora-se o comportamento em três camadas — os **pesos** do modelo, o **comportamento do arnês** e a **memória contextual**. As duas últimas são onde você atua: ajustar hooks, instruções e memória a partir dos rastros (*traces*) de execução faz o agente melhorar ao longo do tempo sem re-treinar nada.

A segunda é a comoditização do próprio arnês. Ferramentas como o **Pi** (de Mario Zechner) são arneses de agente de código minimalistas e extensíveis: mantêm um núcleo pequeno — loop do agente, runtime de chamada de ferramentas, API unificada de modelos, TUI — e deixam o resto aberto para você adaptar ao seu fluxo. O ponto arquitetural é o mesmo do MCP: **separar o fluxo do agente do fornecedor do modelo**, para trocar de modelo (ou de vendor) sem reescrever a cola.

## Referências

**Fundamentos de agentes e do loop**

- Yao et al., *ReAct: Synergizing Reasoning and Acting in Language Models* (2022) — <https://arxiv.org/abs/2210.03629>
- Anthropic, *Building Effective Agents* (workflows × agentes) — <https://www.anthropic.com/engineering/building-effective-agents>
- Russell & Norvig, *Artificial Intelligence: A Modern Approach* (definição de agente inteligente) — <https://aima.cs.berkeley.edu/>

**Claude Code, MCP e saída estruturada**

- Anthropic, *Model Context Protocol* — <https://docs.claude.com/en/docs/agents-and-tools/mcp> · especificação: <https://modelcontextprotocol.io>
- Anthropic, *Claude Code — slash commands e CLI* — <https://docs.claude.com/en/docs/claude-code/slash-commands> · <https://docs.claude.com/en/docs/claude-code/cli-reference>

**Certificação e formação**

- Anthropic Academy (cursos gratuitos) — <https://anthropic.skilljar.com> · <https://www.anthropic.com/learn>
- Claude Certified Architect – Foundations (CCA-F) — <https://anthropic-partners.skilljar.com/claude-certified-architect-foundations-certification>

**Engenharia de arnês**

- Curso *Learn Harness Engineering* — <https://github.com/walkinglabs/learn-harness-engineering>
- Lista *Awesome Harness Engineering* — <https://github.com/ai-boost/awesome-harness-engineering>
- **Pi**, arnês de agente de código (Mario Zechner) — <https://github.com/earendil-works/pi>

### Resumo do Capítulo 8

Claude é uma API stateless embaixo de quatro camadas de produto — API, Agent SDK, Claude Code e MCP — e quase todo problema de arquitetura é descobrir qual camada é dona do comportamento quebrado. Ortogonal a elas, a pilha de responsabilidade de todo sistema agêntico tem seis camadas — Governance & Audit, Evaluation Gates, Harness Orchestration, Tools/MCP, Context Engineering e Model Layer — e um agente que falha em produção quase sempre deixou uma delas sem dono. O loop agêntico é o seu código inspecionando `stop_reason` e anexando `tool_result` — um laço interno por tarefa que casa com um laço externo autônomo entre sessões, e ambos só são seguros com teto de iterações, condição de parada em disco e verificação a cada volta; subagentes existem para poupar contexto, com decomposição sequencial ou adaptativa e retry só do que falhou. No Claude Code, regras vão na camada certa do `CLAUDE.md` e comandos são travados por `allowed-tools`. Saída confiável vem de declarar o schema como tool com `tool_choice` mais um loop de validação e retry. Boas ferramentas têm descrições de doc de API, erros estruturados e o transporte certo (STDIO local, SSE remoto). E confiabilidade é reancorar fatos duráveis contra o "perdido no meio", cachear só o estável e escalar para humano quando a confiança cai. Por baixo de tudo está o arnês: a infraestrutura que torna o loop confiável. É essa a trilha que o exame CCA-F cobra — e que você passa por já ter construído cada peça.

---
