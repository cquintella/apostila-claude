# Apostila Completa de Claude

> *“Os limites da minha linguagem significam os limites do meu mundo.”*
> — Ludwig Wittgenstein, *Tractatus Logico-Philosophicus*

> Esta apostila foi escrita para quem quer entender Claude de verdade — não apenas
> apertar botões, mas saber o que acontece por baixo, por que acontece, e como tirar
> proveito disso. O estilo é deliberadamente direto: explicamos o conceito, mostramos
> um exemplo, e somos honestos sobre os custos e as armadilhas. Se em algum ponto você
> pensar "mas e se eu fizer X?", provavelmente a seção seguinte responde isso.

Por:

Carlos Alvaro de M. S. Quintella e
Antropic Claude

> **Anti-copyright — uso livre.** Esta obra é liberada deliberadamente de qualquer
> restrição de copyright. Você pode copiar, redistribuir, adaptar, traduzir, imprimir e
> reutilizar este material — no todo ou em parte, para qualquer fim, inclusive comercial —
> sem pedir permissão e sem pagar nada. Não há direitos reservados. A atribuição aos
> autores é apreciada, mas não é obrigatória. O conhecimento aqui reunido pertence a quem
> quiser usá-lo: copie à vontade.

---

# Capítulo 1: Fundamentos

Antes de usar uma ferramenta, vale a pena entender que tipo de máquina ela é. Muita
gente trata um modelo de linguagem como um oráculo ou como um banco de dados que
"sabe" coisas. Nenhuma das duas imagens está certa, e partir de uma imagem errada leva
a frustração previsível. Este capítulo constrói o modelo mental correto, de baixo para
cima.

## Conceitos Fundamentais

### O que é um Large Language Model (LLM)

Um Large Language Model é, no fundo, uma função estatística gigantesca treinada para
fazer uma única coisa: dado um pedaço de texto, prever qual é o próximo pedaço mais
provável. Só isso. Toda a aparente inteligência — escrever código, resumir contratos,
explicar física — emerge dessa tarefa simples repetida bilhões de vezes durante o
treinamento.

O texto não entra no modelo como letras. Ele é quebrado em **tokens**, que são
fragmentos de palavras. A palavra "computador" pode virar um único token; "anticonstitucionalmente"
pode virar cinco ou seis. Em inglês, a regra de bolso é que um token equivale a uns
quatro caracteres, ou cerca de ¾ de uma palavra. Isso importa por um motivo muito
prático: **você paga por token**, não por palavra ou por caractere. Quem escreve prompts
prolixos paga mais e, muitas vezes, recebe respostas piores.

Mas o que é um token _dentro_ do computador? No fim das contas, é apenas um **número
inteiro** — um índice numa tabela chamada vocabulário, que o modelo aprendeu durante o
treinamento. O modelo não vê letras nem palavras; ele vê uma sequência de inteiros. Quando
você digita uma frase, um _tokenizer_ a quebra em fragmentos e troca cada fragmento pelo seu
índice. Por exemplo, a frase "Claude é incrível" poderia ser representada assim:

```
Texto:    "Claude"   " é"     " incr"   "ível"
Tokens:   [ 34782 ,   1085 ,   98213 ,   4521 ]
```

Ou seja, quatro tokens, quatro inteiros. Repare em três detalhes que costumam surpreender
quem vê isso pela primeira vez. Primeiro, **o espaço faz parte do token**: " é" (com espaço)
é diferente de "é" (sem espaço) e tem outro índice. Segundo, **uma palavra pode quebrar no
meio**: "incrível" virou " incr" + "ível", porque o tokenizer aprendeu fragmentos comuns, e
não palavras inteiras. Terceiro, **os números são arbitrários** — o índice 34782 não tem
significado matemático; é só o endereço daquele fragmento na tabela. (Os valores acima são
ilustrativos; o vocabulário real de Claude tem dezenas de milhares de entradas.) É por isso
que, lá adiante, o modelo precisa converter cada inteiro num vetor — o _embedding_ — antes de
conseguir raciocinar sobre significado.

A previsão do próximo token não é determinística por padrão. O modelo produz uma
distribuição de probabilidades sobre todos os tokens possíveis, e então sorteia um. Dois
parâmetros controlam esse sorteio:

- **Temperature** controla o quanto o modelo "arrisca". Temperatura baixa (perto de 0)
  faz o modelo escolher quase sempre o token mais provável — respostas conservadoras e
  repetíveis. Temperatura alta espalha as escolhas — respostas mais criativas e mais
  imprevisíveis.
- **Top-p e top-k** (amostragem nucleus e top-k) limitam o conjunto de tokens elegíveis
  antes do sorteio, descartando a cauda de opções improváveis.

Vale uma observação importante e atual: os modelos Claude mais recentes (a família 4.x e
o Fable 5) **removeram os parâmetros de amostragem manual** — `temperature`, `top_p` e
`top_k` — em favor de mecanismos de raciocínio adaptativo que veremos no Capítulo 2. Ou
seja, o conceito de "temperatura" continua útil para entender _como_ um LLM funciona,
mas na prática, com os modelos novos, você guia o comportamento por prompt e por nível de
esforço, não por um botão de temperatura.

Finalmente, há a **context window**: a quantidade máxima de tokens que o modelo consegue
considerar de uma vez (entrada mais saída). Pense nela como a mesa de trabalho do modelo.
Tudo o que estiver fora da mesa simplesmente não existe para ele. Os modelos Claude atuais
trabalham com janelas grandes — até 1 milhão de tokens nos modelos de ponta —, o que é
muito, mas não é infinito, e enchê-la tem custo e custa latência.

### Revolução dos Transformers

... 

### LLM vs Chatbot

<reescrever>
Confundir o LLM com o chatbot é como confundir o motor com o carro. O **LLM é o motor**:
uma função que transforma tokens de entrada em tokens de saída. Ele não tem memória entre
chamadas, não tem botões, não tem tela. O **chatbot é o carro inteiro**: a interface de
conversa, o histórico que é reenviado a cada turno, os botões de anexar arquivo, o sistema
que decide quando buscar na web.
</reescrever>

Entre os dois há **camadas de abstração**. O chatbot guarda o histórico e o reempacota; uma
camada de orquestração decide quais ferramentas oferecer; uma camada de API expõe tudo isso
de forma programável. Entender essa separação evita confusões clássicas — por exemplo,
achar que o modelo "lembra" da conversa de ontem. Ele não lembra; o chatbot é que reenvia o
contexto. O modelo, a cada chamada, parte do zero.

### O que exatamente é o contexto

...

### Code Interpreters e Capacidades de Código

<melhorar reason="Não está claro">
LLMs são surpreendentemente bons com código porque código é texto altamente estruturado e
abundante no treinamento. Mas escrever código e _executar_ código são coisas diferentes.
Quando um modelo apenas escreve, ele pode estar enganado — o código parece certo e não
roda. Por isso existem os **code interpreters**: ambientes isolados (sandboxes) onde o
código que o modelo escreve é de fato executado, e o resultado volta para o modelo.

Vale distinguir três capacidades que frequentemente se misturam. **Code completion** é
completar o que você começou a digitar. **Code generation** é produzir um trecho inteiro a
partir de uma descrição. E **code explanation/refactoring** é ler código existente e
explicá-lo ou reestruturá-lo. São tarefas distintas, com prompts distintos, e o leitor que
souber qual está pedindo obterá resultados muito melhores.

</melhorar>

## O que é Claude

### Claude como Modelo de Linguagem

Claude é a família de modelos de linguagem da **Anthropic**. O que diferencia Claude no
mercado não é apenas a capacidade bruta, mas a filosofia de design. A Anthropic adota uma
abordagem chamada **Constitutional AI**: em vez de depender só de humanos rotulando
respostas boas e ruins, o modelo é treinado para criticar e revisar as próprias respostas
segundo um conjunto explícito de princípios — uma "constituição". O objetivo é um modelo
que seja útil, honesto e inofensivo, nessa ordem de prioridade quando há conflito.

Na prática, isso se traduz numa postura de **safety first**: Claude tende a recusar pedidos
que parecem perigosos, mas — e isso é um ponto fino — os modelos recentes erram menos para
o lado do "recuso por precaução". Ser honesto sobre limitações é parte do design: Claude
admite quando não sabe, em vez de inventar com confiança.

<reesecrever: elaborar em nova sessão>
Infelizmente, ultimamente, com advento do Fable 5, as coisas j;a não são claras para o usuário, e a tendência é ficar ainda pior, enquanto Governos correm para regular e restrigir o uso de IA, diferentes Governos terão diferentes legislações. A solução é uma IA etremamente restrita ou cada Governo ter sua própria IA.

</reescrever>

### Linhagem de Modelos Disponíveis

A linha evoluiu por gerações. Convém conhecer o mapa, porque escolher o modelo certo é
metade da batalha de custo e qualidade.

A **família Claude 3** (Haiku, Sonnet, Opus) foi a primeira geração amplamente adotada e
estabeleceu o padrão de três tamanhos: um rápido e barato, um equilibrado, e um poderoso.
É hoje, em sua maioria, histórica — útil para entender a nomenclatura, mas substituída na
prática pela geração atual.

A geração atual é a **família Claude 4.x** mais o **Claude Fable 5**. Os principais modelos
ativos, com suas características de referência:

| Modelo                | Contexto | Saída máx. | Entrada / Saída (por 1M tokens) | Quando usar                                                    |
| --------------------- | -------- | ---------- | ------------------------------- | -------------------------------------------------------------- |
| **Claude Fable 5**    | 1M       | 128K       | US$ 10 / US$ 50                 | Raciocínio mais exigente, trabalho agêntico de longo horizonte |
| **Claude Opus 4.8**   | 1M       | 128K       | US$ 5 / US$ 25                  | O Opus mais capaz; trabalho autônomo, conhecimento, memória    |
| **Claude Sonnet 4.6** | 1M       | 64K        | US$ 3 / US$ 15                  | Melhor equilíbrio velocidade/inteligência                      |
| **Claude Haiku 4.5**  | 200K     | 64K        | US$ 1 / US$ 5                   | Tarefas simples, rápidas e baratas                             |

Há ainda o **Claude Mythos 5**, que tem as mesmas capacidades, preço e comportamento de API
do Fable 5, mas só é acessível através de um programa específico (Project Glasswing). Para
quase todo mundo, o "modelo mais capaz" ia ser o Fable 5, até o Governo dos Estadudos UNidos Restringir o acesso.

<melhorar críticas: contexto nao está completo, qual o objetivo disto aqui?>
Um detalhe que economiza dor de cabeça: os identificadores de modelo são strings exatas
(`claude-opus-4-8`, `claude-sonnet-4-6`, `claude-fable-5`). Não invente sufixos de data nem
"melhore" o nome — um ID errado simplesmente devolve erro 404.
</melhorar>

Quanto ao **roadmap e futuro**: a Anthropic libera modelos novos com regularidade, e a regra
prática é "use o mais recente da sua faixa, a menos que tenha um motivo explícito para fixar
um antigo". Modelos antigos são aposentados com aviso prévio e passam a devolver erro.

<exclarecer: existe um roadmap?>

## Plataformas e Interfaces

O mesmo motor está disponível por várias portas de entrada. Escolher a porta certa é uma
decisão de produtividade.

### Claude Web (claude.ai)

A interface web é a porta de entrada mais simples. Tem um nível **gratuito** com limites de
uso e um nível **Pro** com mais capacidade e acesso a modelos mais poderosos. Dois recursos
merecem destaque. Os **Artifacts** são painéis laterais onde código, documentos ou pequenas
aplicações aparecem editáveis e executáveis, separados do fluxo da conversa — ótimos para
iterar sobre um mesmo artefato. A **busca na web integrada** permite que Claude consulte
informações atuais quando a pergunta exige. A web é ideal para exploração e prototipagem; as
**limitações** são os rate limits e a falta de integração profunda com seus arquivos locais.

### Claude Desktop

O aplicativo de desktop (Mac e Windows) traz Claude para perto do seu sistema. Sua
configuração permite, entre outras coisas, conectar **MCP servers** (que veremos no
Capítulo 4) e estender as capacidades do modelo com ferramentas locais. É também o ponto de
contato natural com o **Claude Code**, a ferramenta de terminal. A ideia central é o
contexto local: o modelo pode trabalhar com seus arquivos sem que você precise colar tudo
manualmente.

### Claude API

A API é a porta para quem constrói software. Aqui você lida diretamente com **rate
limiting e quotas** (limites de requisições e de tokens por minuto), **batch processing**
(processar milhares de requisições de forma assíncrona, com 50% de desconto), e **streaming**
(receber a resposta token a token, em vez de esperar o fim). A API é onde a economia e a
robustez de uma aplicação de produção são realmente decididas, e é o foco de boa parte dos
Capítulos 3 e 4.

### Claude Code (Ferramenta de Desenvolvimento)

Claude Code é um agente que vive no terminal e trabalha como um par de programação. Ele tem
**integração com o terminal**, faz **gestão de arquivos** (lê, edita, cria), e opera num
**fluxo agentic**: recebe um objetivo, planeja, executa passos com ferramentas, verifica o
resultado e itera. Os casos de uso típicos são refatorações, correção de bugs, criação de
features e automação de tarefas repetitivas de engenharia. É, em essência, o Capítulo 4
inteiro encarnado numa ferramenta.

## Quando Usar Cada Versão

Não existe "a melhor ferramenta", existe a ferramenta certa para a tarefa. Uma matriz
simples de decisão:

- **Exploração e prototipagem rápida** → Claude Web. Custo zero de setup, ótimo para
  descobrir se uma ideia tem pernas.
- **Trabalho local com seus arquivos e desenvolvimento** → Claude Desktop / Claude Code.
- **Aplicações em produção** → API. Você controla custo, latência, retries e escala.
- **Engenharia e scripting agêntico** → Claude Code.

A regra geral: quanto mais perto da exploração, mais perto da Web; quanto mais perto da
produção e da automação, mais perto da API.

## Landscape Competitivo

Nenhuma ferramenta existe no vácuo. Conhecer os concorrentes ajuda a saber quando Claude é a
escolha certa — e quando não é.

Entre os **LLMs gerais**, o **ChatGPT (OpenAI)** é o concorrente mais visível, com ecossistema
amplo; suas fraquezas em relação a Claude tendem a aparecer em tarefas de raciocínio longo e
em seguir instruções com precisão. O **Gemini (Google)** brilha em multimodalidade e na
integração com o Google Workspace. O **Llama (Meta)** é open-weight — você pode rodar em
infraestrutura própria, o que importa para quem tem requisitos de privacidade ou custo em
escala. A **Mistral** é a alternativa europeia, forte em modelos pequenos e eficientes.

Entre as **ferramentas de coding**, o **GitHub Copilot** integra-se nativamente às IDEs e tem
modelo de negócio por assinatura. O **Cursor** é um editor "AI-first". O **Continue.dev** é
open-source e altamente customizável. O **JetBrains AI Assistant** integra-se ao ecossistema
JetBrains.

O **posicionamento de Claude** se sustenta em três pilares: segurança, qualidade de raciocínio
e um design de API limpo. Os trade-offs honestos são custo versus desempenho e velocidade
versus qualidade — os modelos de ponta são mais caros e, em alto esforço, mais lentos. A hora
de escolher Claude é quando você precisa de raciocínio complexo, de conteúdo longo e
coerente, ou de garantias de segurança.

## Arquitetura Geral e Fluxo de Processamento

Vale a pena abrir o capô e ver o que acontece entre apertar Enter e receber a resposta. O
caminho de uma requisição passa por etapas bem definidas.

Tudo começa com a **entrada do usuário**, que é **validada** (o formato está correto? os
papéis de mensagem alternam corretamente?), **roteada** ao modelo certo, **processada** e
devolvida como **output**. Dentro do "processada" mora a parte interessante.

Primeiro vem a **tokenização**: o texto vira tokens. Como você paga por token e a janela é
contada em tokens, há um overhead inerente — formatação, tags, espaços, tudo conta. Em
seguida, o **embedding** converte cada token num vetor numérico de alta dimensão; é assim que
o modelo representa significado. A posição de cada token também é codificada (positional
encoding), porque "cão morde homem" e "homem morde cão" têm os mesmos tokens em ordens
diferentes.

A **inferência** propriamente dita passa esses vetores pelas camadas do modelo, usando o
mecanismo de **attention** — que permite a cada token "olhar" para os outros e pesar quais são
relevantes. É a etapa cara em computação e a principal fonte de latência. Depois vem a
**geração**: o modelo prevê um token por vez, e cada token gerado realimenta a entrada para o
próximo. É aqui que entram os controles de amostragem (ou, nos modelos novos, o raciocínio
adaptativo) e as **stop sequences** que dizem quando parar. Por fim, o **output** decodifica os
vetores de volta para texto, formata e entrega.

Dois conceitos de **tempo e custo** valem ouro na prática. O **first token latency** é quanto
demora até o primeiro token aparecer — dominado pela etapa de processar a entrada inteira. A
**token generation speed** é quão rápido os tokens saem depois disso. Por isso o **streaming**
melhora tanto a experiência percebida: você vê o primeiro token cedo, mesmo que a resposta
completa demore. E há a assimetria fundamental de custo: **tokens de saída custam várias vezes
mais que tokens de entrada** (veja a tabela de preços acima). Otimizar uma aplicação quase
sempre significa, antes de tudo, controlar o volume de tokens de saída.


---

# Capítulo 2: Prompt Engineering

Se o Capítulo 1 explicou _o que_ é a máquina, este explica _como falar com ela_. Prompt
engineering tem fama de arte mística, mas é mais perto de engenharia do que de feitiçaria:
princípios claros, resultados mensuráveis, iteração disciplinada.

## Fundamentos de Prompt Engineering

### Estrutura Básica de Prompts

A maioria dos prompts ruins falha por falta de estrutura, não por falta de capacidade do
modelo. Um bom prompt costuma ter quatro componentes, mesmo que implícitos:

- **Role/persona**: quem Claude deve ser. "Você é um revisor de código sênior especializado
  em segurança" calibra o tom e a profundidade.
- **Task/objetivo**: o que fazer, com um verbo claro. "Revise", "resuma", "extraia",
  "traduza".
- **Context**: as informações relevantes — o código, o documento, os exemplos.
- **Output format**: como responder. JSON? Lista com marcadores? Um parágrafo?

A diferença entre "me ajuda com esse código" e um prompt com esses quatro elementos é,
muitas vezes, a diferença entre uma resposta genérica e uma resposta utilizável.

### Técnicas Fundamentais

Três técnicas dão o maior retorno pelo menor esforço. A primeira é **clareza e
especificidade**: diga exatamente o que quer. "Resuma em três frases para um executivo não
técnico" é melhor que "resuma". A segunda é dar **exemplos** (few-shot learning): mostrar
um ou dois exemplos do formato desejado ensina o modelo melhor do que parágrafos de
descrição. A terceira são as **instruções negativas** — dizer o que _não_ fazer ("não
inclua preâmbulo", "não invente números"). Combinadas com a **separação de
responsabilidades** (uma tarefa por prompt, quando possível), essas técnicas resolvem a
maioria dos problemas do dia a dia.

### Boas Práticas Imediatas

Para já: seja direto, evite ambiguidade, use exemplos quando a tarefa for complexa, e
sempre especifique o formato do output. Esse último ponto é subestimado — metade das
"respostas ruins" são, na verdade, respostas certas no formato errado.

## Técnicas Avançadas de Prompting

### Chain-of-Thought

Para problemas que exigem raciocínio em vários passos, pedir ao modelo para "pensar em voz
alta" antes de responder melhora bastante a qualidade. Isso é o **chain-of-thought**. Pode
ser explícito ("explique seu raciocínio passo a passo antes de dar a resposta final") ou
implícito. O custo é real — mais tokens de raciocínio significam mais dinheiro e mais
latência — então use quando o problema realmente for de raciocínio, não para perguntas
triviais.

### In-Context Learning

O modelo aprende com os exemplos que você coloca no próprio prompt — **in-context
learning**. A versão sofisticada é o **dynamic few-shot**: escolher, em tempo de execução,
os exemplos mais relevantes para a pergunta atual, em vez de exemplos fixos. Há um limite de
retornos decrescentes: depois de alguns exemplos bem escolhidos, adicionar mais custa
tokens sem melhorar muito. Qualidade de exemplo importa mais que quantidade.

### Prompt Compression

Quando o prompt cresce, vale comprimi-lo sem perder o essencial — remover redundância,
trocar prosa por estrutura, cortar contexto irrelevante. O trade-off é entre economia de
tokens e risco de cortar algo que importava. A regra: comprima a forma, preserve a
substância.

### Structured Prompting

A forma como você estrutura o prompt afeta o resultado. **Tags XML** (`<contexto>...</contexto>`)
ajudam o modelo a separar partes do prompt e são especialmente eficazes com Claude.
**JSON** é bom quando você quer saída estruturada e parseável. **Markdown** organiza
instruções para leitura. Cada formato tem seu lugar: XML para delimitar seções, JSON para
contratos de dados, Markdown para instruções legíveis.

### Extended Thinking

Os modelos Claude recentes têm um modo de **raciocínio interno** antes de responder. Aqui há
uma mudança importante em relação ao que muitos materiais antigos descrevem. O modelo de
"orçamento fixo de pensamento" (`budget_tokens`) foi **substituído pelo raciocínio
adaptativo**: você ativa `thinking: {type: "adaptive"}` e o próprio modelo decide quando e
quanto pensar. A profundidade é controlada por um parâmetro de **esforço** (effort), com
níveis `low`, `medium`, `high`, `xhigh` e `max`. Nos modelos de ponta (Fable 5, Opus 4.8,
4.7), o esforço importa mais do que em qualquer geração anterior — `high` ou `xhigh` é o
ponto doce para a maioria do trabalho de coding e agêntico, e `low`/`medium` servem para
tarefas rotineiras. O impacto em custo é direto: mais esforço, mais tokens de raciocínio.

## Debugging e Iteração de Prompts

### Avaliação de Prompts

"Esse prompt está bom?" é uma pergunta que só se responde com medida. Defina **métricas de
qualidade** para a sua tarefa (acurácia de extração, aderência ao formato, ausência de
alucinação), monte um pequeno conjunto de testes representativos, e meça. Sem isso, você está
ajustando no escuro.

### Identificar Falhas Comuns

Os modos de falha mais comuns têm assinatura reconhecível. **Alucinações** são respostas
fabricadas com confiança — números, citações, APIs que não existem. **Outputs genéricos**
sinalizam prompt vago demais. **Não seguir instruções** costuma vir de instruções
conflitantes ou enterradas no meio de um prompt longo. **Verbosidade ou concisão excessiva**
indica que faltou especificar o formato e o tamanho desejados.

### Iteração Efetiva

Itere com **mudanças incrementais** — altere uma coisa por vez, ou você não saberá o que
causou a melhora. Use **A/B testing** quando puder comparar duas versões no mesmo conjunto de
testes. E mantenha **logs** das versões e dos resultados; um prompt que melhora hoje e piora
amanhã sem registro é um mistério insolúvel.

### Instrumentação

Em produção, instrumente: rastreie desempenho, monitore custo por requisição, e construa
dashboards de qualidade. Tratar prompts como código — versionados, testados, monitorados — é
o que separa um protótipo de um sistema confiável.


---

# Capítulo 3: Otimização e Performance

Este é o capítulo que paga a conta. Tudo até aqui foi sobre fazer Claude trabalhar; agora é
sobre fazê-lo trabalhar barato e rápido, sem perder qualidade. As técnicas aqui têm efeito
direto na fatura.

## Otimização de Tokens e Contexto

### Técnicas de Redução de Tokens

Como você paga por token, reduzir tokens é reduzir custo. Quatro alavancas: **prompt
conciso** (corte a gordura), **relevance filtering** (inclua só o contexto necessário, não o
documento inteiro), **batch processing** (agrupe requisições) e **formatos estruturados**
(JSON costuma ser mais compacto e parseável que prosa). O maior ganho geralmente está no
filtro de relevância — a tentação de "jogar tudo no contexto por garantia" é cara e, pior,
costuma diluir a atenção do modelo.

### Arquivo .claude e Context Management

Quando você trabalha com Claude Code ou Desktop, um arquivo de configuração de contexto (a
família de arquivos `.claude`) define o que o modelo sabe sobre o seu projeto sem que você
precise repetir a cada conversa. Uma estrutura recomendada separa contextos por tarefa
(profiles), carrega contexto dinamicamente conforme a necessidade, e pode ser automatizada
com scripts que trocam o contexto ativo. A ideia central é a mesma do filtro de relevância:
o modelo deve ter o contexto certo na hora certa, nem mais, nem menos.

### Prompt Caching

Esta é, provavelmente, a técnica de maior impacto financeiro do capítulo. **Prompt caching**
permite reaproveitar o processamento de um prefixo de prompt que se repete entre requisições
— um system prompt grande, um conjunto de exemplos, um documento de referência.

O mecanismo funciona por **prefixo exato**: o cache é indexado pelos bytes do prompt até um
ponto de quebra (cache breakpoint). A consequência prática, que merece estar tatuada: **qualquer
alteração de um byte no prefixo invalida o cache de tudo o que vem depois**. Um timestamp no
system prompt, um JSON com chaves em ordem não-determinística, uma ferramenta a mais na lista
— qualquer um desses zera o cache silenciosamente.

A economia é concreta. Uma **leitura de cache** custa cerca de 10% do preço normal de entrada
(um desconto de ~90%). Uma **escrita de cache** custa um pouco mais que o normal: 1,25× para o
cache de 5 minutos, 2× para o de 1 hora. O ponto de equilíbrio é rápido — com o cache de 5
minutos, duas requisições já compensam. O cache tem TTL (tempo de vida) e limite de pontos de
quebra (no máximo 4). Vale a pena sempre que você reusa um prefixo grande e estável.

Como verificar se está funcionando? A resposta da API traz, no objeto `usage`, os campos
`cache_read_input_tokens` e `cache_creation_input_tokens`. Se as leituras de cache estão
sempre em zero apesar de prefixos idênticos, há um invalidador silencioso escondido — quase
sempre um `datetime.now()` ou um UUID no começo do prompt.

### Memory Systems

Contexto e memória são coisas diferentes, e confundi-las custa caro. **Contexto** é o que
está na janela agora; some quando a conversa acaba. **Memória** é o que persiste entre
sessões — preferências do usuário, fatos aprendidos, decisões anteriores. Os sistemas de
memória dão ao modelo um lugar para escrever e ler ao longo do tempo (um diretório de
arquivos, por exemplo), com continuidade de sessão e poda periódica (memory pruning) para não
inchar. A regra de bolso: use contexto para o que importa agora, memória para o que precisa
sobreviver ao "agora".

## Economia e Custos

### Pricing e Modelos de Cobrança

A cobrança é por token, separada entre **entrada** e **saída**, e a saída custa várias vezes
mais (reveja a tabela do Capítulo 1). Os modelos têm preços muito diferentes: o Haiku 4.5 é
ordens de magnitude mais barato que o Fable 5. Há descontos por volume e o **batch
processing** corta 50% para trabalho não urgente. A primeira pergunta de qualquer otimização
de custo é: "esta tarefa precisa mesmo do modelo mais caro?"

### Trade-offs de Custo

O eixo central é **modelo versus desempenho**. Classificação, extração simples e respostas
curtas raramente precisam de um Opus — um Haiku ou Sonnet faz o trabalho por uma fração do
preço. Reserve os modelos de ponta para raciocínio complexo, código difícil e trabalho
agêntico de longo horizonte. Há também os eixos qualidade versus custo, latência versus
economia, e automação versus trabalho manual. Nenhum tem resposta universal; a resposta é
medir no seu caso.

### ROI de Automação

Automatizar com Claude tem retorno calculável. Estime o custo por tarefa automatizada,
compare com o custo do trabalho manual equivalente, e encontre o ponto de equilíbrio
(break-even). Em escala, a economia cresce de forma não-linear — o que justifica investir em
otimização de prompts e caching para tarefas de alto volume.

### Monitoramento Contínuo

Custo não é um problema que se resolve uma vez. Monte um **dashboard de custos**, configure
**alertas de orçamento** e trate a otimização como prática contínua. Workloads mudam, prompts
incham, e um modelo mais caro entra "só para testar" e nunca sai. Monitorar é como você
percebe isso antes da fatura.

## Multimodalidade

### Imagens como Input

Claude aceita imagens, não só texto. Os formatos comuns (PNG, JPEG, etc.) são suportados, e a
**resolução** importa em dois sentidos: alta demais desperdiça tokens, baixa demais perde
detalhe. Os modelos recentes têm suporte a alta resolução, o que ajuda em tarefas visuais
densas — mas imagens grandes consomem mais tokens de imagem. Otimize o tamanho para o nível de
detalhe que a tarefa realmente exige.

### Casos de Uso

As imagens abrem casos práticos: **OCR e extração de texto** de documentos digitalizados,
**análise de documentos** (faturas, formulários), **descrição e análise visual**,
interpretação de **diagramas e gráficos**, e **comparação visual** entre versões. Em todos,
combinar a imagem com instruções textuais claras melhora muito o resultado.

### Best Practices

Escolha a resolução pelo trade-off qualidade/tokens, sempre combine a imagem com contexto
textual, e pergunte-se se a tarefa precisa mesmo de imagem — às vezes o texto já está
disponível e mandar a imagem é só desperdício.

### Resumo do Capítulo 3

Reduza tokens cortando contexto irrelevante. Use prompt caching para qualquer prefixo grande e
estável — é o maior ganho financeiro, desde que você não invalide o cache por descuido. Separe
contexto de memória. Escolha o modelo pelo trabalho, não pelo hábito. Monitore custo
continuamente. E trate imagens como o que são: poderosas e caras em tokens.

---

# Capítulo 4: Integrações e Extensões

Claude sozinho é um modelo de linguagem. Conectado a ferramentas, vira um sistema. Este
capítulo é sobre as conexões — o que vem embutido, o que você pluga, e como orquestrar tudo.

## Plugins e Capacidades Nativas

### Code Interpreter

O code interpreter (execução de código) roda código que o modelo escreve dentro de um
ambiente isolado. Tecnicamente, é um contêiner sandboxed — tipicamente com Python e uma boa
biblioteca de ferramentas de dados pré-instaladas (pandas, numpy, matplotlib, e bibliotecas
para gerar arquivos como `.docx`, `.pptx` e PDFs). Não tem acesso à internet, por segurança,
e os contêineres podem ser reutilizados entre requisições para manter estado. Os casos de uso
clássicos são análise de dados, geração de gráficos e cálculos que o modelo não deve fazer "de
cabeça". A limitação principal é justamente o isolamento: sem rede, e com recursos finitos.

### Web Search

A busca na web dá ao modelo acesso a informação posterior ao seu corte de conhecimento. Pode
ser acionada manual ou automaticamente. A arte está em formular boas queries e em saber
**quando não usar** — para fatos estáveis, a busca só adiciona latência. Os resultados vêm com
citações, o que ajuda a verificar a confiabilidade. É a ferramenta certa para eventos
recentes, preços atuais e documentação que mudou.

### Artifacts

Os Artifacts são painéis onde código, documentos e pequenas aplicações aparecem separados da
conversa — editáveis, executáveis e compartilháveis. Surgem quando você pede algo substancial
e autônomo (um componente, um documento, um script). São ideais para iterar sobre um mesmo
artefato sem poluir o histórico. A limitação é que vivem dentro da interface; para integração
programática, você quer a API.

### Graphify (Knowledge Graph para Assistentes de Código)

O Graphify merece uma seção própria porque resolve um problema real e específico: fazer um
assistente de código _entender_ um repositório grande sem despejar o repositório inteiro no
contexto. Ele é uma **skill open-source (licença MIT)** que constrói um **grafo de
conhecimento consultável** a partir de um codebase multi-modal — código, documentação, papers
e diagramas. Funciona com Claude Code e outros assistentes (Codex, OpenCode), e é mantido por
Safi Shamsi.

A ideia central distingue o Graphify de um simples gerador de diagramas. Ele combina **análise
estática com Tree-sitter** (que extrai ASTs, grafos de chamada e docstrings localmente, sem
mandar o código-fonte para nenhum LLM) com **extração semântica via LLM** (para entender a
prosa e, com modelos de visão, ler diagramas). O resultado vai para um grafo NetworkX, sobre o
qual roda o algoritmo de **Leiden** para detectar comunidades semânticas — agrupamento por
topologia do grafo, sem precisar de vector embeddings nem vector store. Ele ainda identifica os
**"god nodes"** (os nós de maior grau, o coração do sistema) e as **"surprises"** (conexões
inesperadas entre arquivos ou domínios, que valem investigação).

O fluxo de trabalho é direto. Requer Python 3.10+; instala-se com `pip install graphifyy &&
graphify install` (note que o pacote PyPI chama-se `graphifyy`, com dois "y", mas o comando de
linha é `graphify`). Para construir o grafo de uma pasta, basta `/graphify ./raw`, e as saídas
caem em `graphify-out/`: um `graph.html` interativo, um `graph.json` consultável e um
`GRAPH_REPORT.md` legível e auditável. Dentro do assistente, os comandos são `/graphify`,
`/graphify query`, `/graphify path` e `/graphify explain`. A integração com Claude Code se dá
via diretivas no `CLAUDE.md` e um hook `PreToolUse`.

O **pipeline interno** é uma sequência de estágios isolados — detect (coleta arquivos) →
extract (nós e arestas via AST e LLM) → build (grafo NetworkX) → cluster (comunidades Leiden) →
analyze (god nodes e surprises) → report → export (HTML/JSON/Obsidian) —, o que facilita
estender qualquer etapa.

Por que isso importa? Pela **economia de tokens**. Em vez de responder a uma pergunta sobre o
código relendo arquivos inteiros, o assistente consulta o grafo. No corpus de exemplo (uma
mistura de repositórios e papers de atenção), o custo médio por consulta caiu de cerca de 123 mil
tokens "ingênuos" para cerca de 1,7 mil — uma **redução de aproximadamente 71,5×**. Esse é o
diferencial real do Graphify: não é o gráfico bonito, é a estrutura que torna a consulta barata.

Quanto à **segurança e confiança**, o desenho é cuidadoso. O Graphify **não embute um LLM** —
usa a chave de API que o seu assistente já tem configurada — e envia ao modelo **apenas
conteúdo semântico, nunca o código-fonte bruto**. Não há telemetria. A validação de entrada é
estrita: apenas URLs `http`/`https`, com limites de tamanho e timeout, contenção de paths e
labels com escape de HTML, defendendo contra SSRF, injeção e XSS.

As **limitações** são as esperadas para esse tipo de ferramenta. A etapa de extração semântica
depende de chamadas a um LLM, então tem custo e latência próprios. O retorno é melhor em
codebases médios ou grandes e multi-modais; para um projeto trivial, é exagero. E o
`GRAPH_REPORT.md` deve ser tratado como ponto de partida de investigação — uma boa primeira
pista —, não como verdade absoluta.

## Integrações Populares

Conectar Claude a ferramentas que você já usa multiplica o valor. O padrão se repete: cada
plataforma exige um setup básico e abre alguns casos de uso reais.

O **Zapier** e o **Make (Integromat)** são plataformas de automação sem código. Com elas,
você liga Claude a centenas de apps — processar e-mails recebidos, extrair dados de
formulários, automatizar fluxos. Zapier tende a ser mais simples; Make oferece mais controle
e templates prontos para fluxos complexos. A escolha entre os dois é uma questão de gosto e de
complexidade do workflow.

A **integração com Google Workspace** cobre o trio Gmail (processar e classificar e-mails),
Google Docs (criar e editar documentos) e Google Sheets (analisar e atualizar planilhas). O
**Slack** permite bots customizados, integração direta e automação de canais — ótimo para
levar Claude para onde o time já conversa. O **Notion** serve para popular bancos de dados,
automatizar templates e gerar respostas. E o **GitHub** abre o code review automático, a
automação de issues e a análise de pull requests.

Para cada uma dessas, o conselho é o mesmo: comece com um caso de uso concreto e estreito,
faça funcionar de ponta a ponta, e só então expanda.

## Orquestração Multi-Ferramenta

### Arquitetura de Workflows

Quando você combina várias ferramentas, precisa decidir a arquitetura. Os passos podem ser
**sequenciais** (um depende do anterior) ou **paralelos** (independentes, rodam juntos). O que
separa um protótipo de um sistema de produção é o tratamento de falhas: **error handling e
fallbacks** (o que fazer quando um passo falha), **timeouts** (não esperar para sempre) e
**retry logic** (tentar de novo, com backoff exponencial). Falhas não são exceção; são parte
do projeto.

### Fluxo de Dados

Os dados precisam fluir entre sistemas que falam línguas diferentes, o que exige
**transformação** entre formatos, respeito aos **rate limits** de cada serviço, e **validação**
em cada fronteira. A regra clássica: valide nas bordas do sistema (entrada do usuário, APIs
externas), confie no interior.

### Casos de Uso Completos

Três pipelines ilustram o padrão. Um **pipeline de análise**: raspar dados → processar com
Claude → gravar num banco. Uma **automação de atendimento**: e-mail recebido → classificação →
ação apropriada. E um **workflow de geração de conteúdo**: briefing → rascunho → revisão →
publicação. Em todos, Claude é uma etapa de um fluxo maior — poderosa, mas uma etapa.

## APIs Custom e MCP

### API da Anthropic

Para integração programática, a API é o caminho. Os pontos a dominar são **autenticação**
(chave de API ou perfil OAuth), os **endpoints principais** (sobretudo `/v1/messages`),
a escolha entre **streaming e batch**, e o **tratamento de erros** com as classes de exceção
tipadas que o SDK oferece. Vale conhecer os códigos de erro mais comuns: 429 (rate limit,
retentável), 400 (requisição malformada, não retentável), 401 (autenticação), 529
(sobrecarga, retentável). Os SDKs já fazem retry automático de 429 e 5xx com backoff.

### Webhooks e Custom Integrations

Para integrações dirigidas por eventos, webhooks permitem que sistemas externos notifiquem
sua aplicação. O setup básico envolve um endpoint HTTPS; a parte crítica é a **segurança** —
verificar a assinatura HMAC de cada entrega para garantir que veio de quem diz ter vindo — e o
**debugging** de entregas que falham (dedupe por ID de evento, porque retentativas chegam com
o mesmo ID).

### MCP (Model Context Protocol)

O **MCP** é um protocolo padronizado para conectar Claude a fontes de dados e ferramentas
externas. Em vez de cada integração reinventar a roda, um **MCP server** expõe capacidades
(ler um banco, consultar uma API, acessar arquivos) de forma que qualquer cliente compatível —
incluindo Claude — possa usar. Você pode criar servers customizados para suas próprias fontes.
É a base sobre a qual integrações modernas e reutilizáveis são construídas, e o motivo de o
ecossistema de extensões crescer tão rápido.

### Resumo do Capítulo 4

Claude traz capacidades nativas — execução de código, busca na web, artifacts — e aceita
extensões. O Graphify é um bom exemplo do tipo de extensão que muda o jogo: transforma um
repositório num grafo consultável e barato. Integrações populares ligam Claude ao seu
ecossistema; a orquestração multi-ferramenta exige tratar falha como regra, não exceção. E o
MCP é o protocolo que torna tudo isso reutilizável.

---

# Capítulo 5: Segurança, Legal e Ética

Quanto mais Claude entra em fluxos reais, mais essas questões deixam de ser teóricas. Este
capítulo é prático e sem juridiquês: o que você precisa saber para usar Claude sem sustos.

## Propriedade Intelectual

### Propriedade do Output Gerado

A pergunta que todo mundo faz: de quem é o que eu crio com Claude? Em termos gerais, o output
gerado é seu para usar, inclusive comercialmente, segundo os termos da Anthropic. Mas
"propriedade" e "direitos autorais" são conceitos distintos, e a proteção autoral de conteúdo
gerado por IA é uma área em evolução em muitas jurisdições. A regra prática: você pode usar e
comercializar, mas não assuma que tem o mesmo tipo de monopólio autoral que teria sobre uma
obra puramente humana.

### Copyright e Dados de Treinamento

Os modelos são treinados em grandes volumes de texto, parte do qual é material protegido por
copyright. Isso levanta questões sobre reprodução não autorizada e sobre fair use. Na prática,
o risco para o usuário aparece quando o output reproduz de forma muito próxima um material
protegido específico. A transformação e o uso de fair use estão no centro do debate jurídico,
que ainda não está fechado.

### Disclosure e Atribuição

Quando você deve revelar que usou IA? Depende do contexto. Algumas indústrias e publicações
têm obrigações de transparência; muitos contextos não têm. O princípio orientador é a
transparência com quem consome o conteúdo — e, em caso de dúvida, divulgar costuma ser a
escolha mais segura.

### Uso Comercial

Monetizar conteúdo gerado, construir produtos baseados em Claude, revender e redistribuir —
tudo isso é possível dentro dos termos de serviço, que valem a leitura. Há restrições
específicas (você não pode, por exemplo, usar a saída para treinar um modelo concorrente, em
geral). Conheça os limites antes de construir um negócio em cima deles.

## Indústria-Específico

A propriedade intelectual muda de cara conforme o setor. Em **código e software**, importam as
implicações de licenças open-source (GPL, MIT) e a conformidade de licenças quando o output se
parece com código existente. Em **conteúdo criativo** (texto, arte, música), o copyright e a
originalidade são o centro. Em **pesquisa acadêmica**, a questão é como citar LLMs e como tratar
autoria e coautoria — as principais revistas já têm guias. E em **conteúdo regulado** (jurídico,
médico, financeiro), entra a camada de compliance, onde o erro tem consequências sérias e a
supervisão humana é obrigatória.

## Segurança e Confiabilidade

### Prompt Injection

O **prompt injection** é o ataque mais característico de aplicações com LLM. A ideia: um
atacante insere instruções maliciosas em dados que o modelo vai processar (um e-mail, uma
página web, um arquivo), tentando sequestrar o comportamento do modelo. Há vários tipos, e a
defesa em produção combina separar claramente instruções de dados, desconfiar de conteúdo
externo, e usar canais de instrução que não possam ser forjados pelo conteúdo do usuário. É um
problema de arquitetura, não de prompt.

### Jailbreaking

**Jailbreaking** é a tentativa de fazer o modelo contornar suas próprias salvaguardas. Claude
é treinado para resistir, mas nenhuma defesa é perfeita. Em aplicações, a postura correta é
não depender só do modelo: adicione validação e limites no seu próprio sistema, porque o
modelo é uma camada de defesa, não a única.

### Alucinações

**Alucinações** são respostas fabricadas que parecem verdadeiras. Ocorrem mais quando o modelo
é pressionado a responder sobre algo que não sabe. Os sinais de alerta: confiança excessiva
sobre detalhes específicos, citações que você não consegue verificar, números muito redondos
ou muito precisos sem fonte. A mitigação combina prompting (peça que o modelo admita quando não
sabe, peça fontes) com validação programática.

### Verificação de Outputs

Nunca confie 100% no output. Construa **sanity checks** (o resultado faz sentido?), **validação
estrutural** (o JSON está no formato certo? os campos obrigatórios estão lá?) e **fact-checking**
para afirmações verificáveis. Em sistemas sérios, a verificação não é opcional — é parte do
design.

## Conformidade e Regulação

Sobre **GDPR e dados pessoais**: processar dados pessoais através de Claude implica obrigações
de privacidade, direitos dos titulares e atenção a onde os dados residem (data residency). As
**leis de IA emergentes** — com destaque para o EU AI Act — estão mudando o cenário, e
regulações nacionais surgem em ritmo acelerado; vale acompanhar. Setores específicos (**saúde,
finanças, governo**) têm camadas adicionais de exigência. E os **termos de serviço da Anthropic**
definem as responsabilidades do usuário e as restrições de uso — que mudam ao longo do tempo,
então reler periodicamente é prudente.

### Resumo do Capítulo 5

O output é seu para usar, mas a proteção autoral é área em evolução — não assuma monopólio. As
regras de IP mudam por setor. Em segurança, trate prompt injection como problema de arquitetura,
desconfie de jailbreaks, vigie alucinações e sempre verifique outputs. Em compliance, conheça
GDPR, as leis de IA emergentes e os termos de serviço, e some supervisão humana onde o erro
custa caro.

---

# Capítulo 6: Prática e Referência

Os capítulos anteriores construíram a teoria. Este é o capítulo das receitas — exemplos
concretos, checklists e referência rápida para colar e usar.

## Casos de Uso Práticos

### Coding

Para **code review**, dê o código, diga o que te preocupa e peça achados ordenados por
severidade ("revise este trecho focando em segurança e concorrência; liste os problemas do mais
grave ao menos grave"). Para **bug fixing**, descreva o sintoma, o comportamento esperado e o
contexto — o que você já tentou ajuda muito. Para **explicação de código**, peça o nível certo
("explique para alguém que conhece programação mas não conhece este framework"). E para
**refactoring**, seja explícito sobre o objetivo (legibilidade? performance? testabilidade?),
porque "refatore" sem objetivo gera mudanças arbitrárias.

### Análise e Research

Para processamento de dados e síntese de documentos, o segredo é dar estrutura: peça o formato
de saída, defina os critérios de comparação, e — quando o volume for grande — divida em pedaços.
Para análise comparativa, forneça uma rubrica explícita do que comparar, ou você receberá uma
comparação genérica.

### Escrita e Conteúdo

Em copywriting, especifique público, tom e tamanho. Em tradução, dê o contexto (formal ou
informal? técnico ou coloquial?). Em paráfrase e resumo, diga o comprimento alvo e o público.
O padrão é sempre o mesmo: o que o modelo não sabe, ele inventa — então diga.

### Educação e Ensino

Para gerar exercícios, especifique nível e formato. Para explicações personalizadas, descreva o
aluno. Para feedback automático, dê a rubrica. Templates prontos economizam muito quando a
tarefa se repete.

### Atendimento ao Cliente

Chatbots, triagem de tickets, geração de FAQ e automação de respostas seguem o mesmo princípio
de qualquer aplicação séria: combine a capacidade do modelo com validação e, em casos
sensíveis, escalonamento para um humano. Um chatbot que nunca passa para um humano é um chatbot
que eventualmente vai irritar alguém.

## Boas Práticas e Armadilhas

### O que Fazer

Sempre valide outputs. Use exemplos específicos. Itere nos prompts em vez de esperar acertar de
primeira. Documente as decisões — por que este prompt, por que este modelo, por que este
formato.

### O que Evitar

Prompts vagos. Sobrecarga de contexto ("jogar tudo por garantia"). Assumir confiabilidade de
100%. E ignorar custo e performance até a fatura chegar.

### Documentação e Versionamento

Trate prompts como código: rastreie versões, mantenha um changelog, e tenha um procedimento de
rollback para quando uma mudança piorar as coisas. Um prompt em produção sem versionamento é um
acidente esperando para acontecer.

## Troubleshooting

### Outputs Repetitivos

Quando o modelo se repete, as causas comuns são prompts que induzem o padrão ou falta de
variação no contexto. Detecte comparando saídas; resolva variando o prompt e, nos modelos
antigos, ajustando amostragem (nos novos, o raciocínio adaptativo cuida disso).

### Não Segue Instruções

Geralmente acontece por instruções conflitantes, enterradas no meio de um prompt longo, ou
agressivas demais. Estruture melhor — instruções importantes no começo ou no fim, formato
explícito — e, com os modelos recentes, suavize linguagem do tipo "CRÍTICO: VOCÊ DEVE", que
tende a fazer o modelo sobre-reagir.

### Muito Verbose ou Muito Conciso

Controle o tamanho explicitamente. "Em três frases", "em no máximo 200 palavras", "responda
apenas com o código". O modelo não adivinha o tamanho que você quer; ele estima — e estima
errado com frequência.

### Alucinações e Erros Factuais

Vigie os sinais de alerta (confiança sobre detalhes não verificáveis), mitigue com prompting
(peça fontes, permita "não sei") e valide programaticamente o que for verificável.

## Referência Rápida

### Snippets de Prompts

Tenha templates prontos para suas tarefas recorrentes — análise, código, conteúdo,
atendimento. Cada template encapsula papel, tarefa, contexto e formato, e poupa você de
reescrever a estrutura toda vez.

### Parâmetros Comuns

Nos modelos recentes, a "temperatura" deu lugar ao nível de **esforço** (`low`/`medium`/`high`/
`xhigh`/`max`): use `low`/`medium` para tarefas rotineiras e `high`/`xhigh` para coding e
trabalho agêntico. Para **max tokens**, não seja mesquinho — estourar o limite trunca a
resposta no meio; um bom padrão é ~16.000 para requisições normais e ~64.000 quando estiver
usando streaming. Saídas acima de ~16K exigem streaming para não esbarrar em timeout.

### Recursos Externos

A documentação oficial da Anthropic é a fonte de verdade para IDs de modelo, preços e
recursos novos — consulte-a sempre que houver dúvida, porque o cenário muda rápido. Há
comunidades ativas (Discord, Reddit), papers de pesquisa e ferramentas de terceiros como o
Graphify do Capítulo 4.

## Roadmap e Futuro

O raciocínio estendido (extended thinking / adaptive thinking) já é realidade nos modelos
atuais e tende a se aprofundar. Modelos posteriores ao corte de conhecimento — como o **Fable
5** e o **Mythos 5** — já são parte do presente, não do futuro, e a cadência de lançamentos da
Anthropic é alta. Capacidades emergentes em pesquisa apontam para agentes mais autônomos,
janelas de contexto maiores e melhor uso de memória. A constante é a mudança: a regra de "use o
modelo mais recente da sua faixa" vale justamente porque a fronteira se move.

### Resumo do Capítulo 6

Tenha receitas prontas para suas tarefas recorrentes — todas seguem papel/tarefa/contexto/
formato. Faça o que funciona (valide, exemplifique, itere, documente) e evite o que falha
(vago, sobrecarregado, ingênuo, caro). No troubleshooting, a maioria dos problemas se resolve
com mais estrutura e formato explícito. E mantenha a documentação oficial por perto, porque
modelos e preços mudam.

---

# Capítulo 7: Integração e Resolução de Problemas

Os capítulos anteriores trataram cada peça isoladamente. Este capítulo final faz o que falta:
mostra como as peças se encaixam num sistema real e oferece um guia de diagnóstico para quando
as coisas dão errado — porque vão dar. Pense nele como o manual de manutenção que vem depois do
manual de uso.

## Juntando as Peças: uma Arquitetura de Referência

Imagine uma aplicação realista: um assistente que ajuda uma equipe de engenharia a entender e
modificar uma base de código grande. Ela usa quase tudo desta apostila, e ver como as peças se
conectam vale mais que descrevê-las de novo.

No centro está a **API de Messages** (Capítulo 4), chamada com um **modelo escolhido pela
tarefa** (Capítulo 1) — Haiku para classificar tickets baratos, Opus 4.8 ou Fable 5 para o
trabalho agêntico difícil. Os **prompts** seguem a estrutura papel/tarefa/contexto/formato
(Capítulo 2). O **system prompt grande e estável fica em cache** (Capítulo 3), com o conteúdo
volátil — a pergunta do usuário, timestamps — depois do ponto de quebra, para não invalidar
nada. Para entender o repositório sem despejá-lo no contexto, a aplicação usa o **Graphify**
(Capítulo 4): consulta o grafo de conhecimento em vez de reler arquivos, cortando o custo por
consulta em dezenas de vezes. Ferramentas externas — GitHub, um banco de dados — entram via
**MCP** (Capítulo 4). E uma camada de **segurança e validação** (Capítulo 5) trata o conteúdo
externo como não confiável e verifica todo output antes de agir sobre ele.

O fluxo de uma requisição típica:

1. O usuário pede algo ("por que a autenticação está lenta?").
2. A aplicação monta o prompt: system prompt cacheado + contexto relevante (puxado do grafo do
   Graphify, não do código inteiro) + a pergunta.
3. Claude raciocina (adaptive thinking, esforço `high`) e decide usar ferramentas — consultar o
   grafo, ler um arquivo específico, rodar um teste no sandbox.
4. Os resultados das ferramentas voltam; Claude itera até concluir.
5. A aplicação valida o output (o diff proposto compila? os testes passam?) antes de apresentá-lo.

Cada decisão dessa cadeia foi tomada por um motivo de custo, qualidade ou segurança discutido
nos capítulos anteriores. É isso que significa "integrar": não usar todas as ferramentas, mas
escolher conscientemente quais e por quê.

## Princípios de Integração

Alguns princípios atravessam qualquer integração e vale destilá-los:

- **Escolha o modelo por etapa, não por aplicação.** Um sistema bem feito usa modelos
  diferentes em pontos diferentes — um barato para triagem, um caro para o trabalho pesado.
  Trocar de modelo no meio de uma conversa, porém, invalida o cache de prompt; quando precisar
  de um modelo mais barato para uma subtarefa, delegue a um subagente separado em vez de trocar
  o modelo do loop principal.
- **Trate o cache como parte da arquitetura.** Ordene o prompt do mais estável ao mais volátil.
  Ferramentas e system prompt primeiro, conteúdo da requisição por último. Verifique
  `cache_read_input_tokens` para confirmar que está funcionando.
- **Valide nas fronteiras.** Entrada do usuário e respostas de APIs externas são as fronteiras;
  valide ali. Confie no interior do sistema.
- **Falha é regra, não exceção.** Timeouts, retries com backoff, fallbacks. Os SDKs já fazem
  retry de 429 e 5xx; você cuida do resto.
- **Mantenha segredos fora do alcance do modelo.** Chaves de API e tokens nunca vão dentro do
  prompt nem do sandbox; são injetados por uma camada externa (vaults, proxies). Conteúdo que
  entra no histórico fica persistido e legível.

## Guia de Resolução de Problemas

Esta é a parte que você vai reler quando algo quebrar. Está organizada por sintoma, no espírito
de "se X acontece, verifique Y".

### Erros de API

| Código  | Significado              | O que verificar                                                                                                        |
| ------- | ------------------------ | ---------------------------------------------------------------------------------------------------------------------- |
| **400** | Requisição malformada    | Papéis de mensagem alternam (user/assistant)? `max_tokens` é positivo? Modelo válido? Parâmetro removido (ver abaixo)? |
| **401** | Autenticação             | Chave de API definida? Não há `ANTHROPIC_API_KEY` e `ANTHROPIC_AUTH_TOKEN` setados ao mesmo tempo?                     |
| **403** | Sem permissão            | A chave tem acesso a esse modelo/recurso?                                                                              |
| **404** | Não encontrado           | ID de modelo com erro de digitação? Não invente sufixos de data.                                                       |
| **413** | Requisição grande demais | Reduza a entrada — trunque histórico, comprima imagens, divida documentos.                                             |
| **429** | Rate limit               | Cabeçalho `retry-after`. Os SDKs já fazem backoff; para volume, suba de tier ou enfileire.                             |
| **529** | Sobrecarga               | Retentável com backoff. Considere um modelo menos carregado.                                                           |

Um detalhe que pega muita gente nos modelos recentes (Fable 5, Opus 4.8/4.7): os parâmetros
`temperature`, `top_p`, `top_k` foram **removidos** e devolvem **400** se enviados. O mesmo vale
para `thinking: {type: "enabled", budget_tokens: N}` — use `thinking: {type: "adaptive"}`. E
prefills na última mensagem do assistente também devolvem 400 nesses modelos. Se um código que
funcionava começou a dar 400 sem motivo óbvio depois de uma troca de modelo, é quase sempre um
desses parâmetros legados.

### O cache não está funcionando

Sintoma: `cache_read_input_tokens` sempre em zero apesar de prefixos "iguais". Causa quase
certa: um **invalidador silencioso** no prefixo. Procure por `datetime.now()`, `Date.now()` ou
um UUID no começo do prompt; por JSON serializado sem ordenar as chaves; por uma lista de
ferramentas que muda entre requisições; ou por um ID de sessão interpolado no system prompt. A
correção é mover o elemento dinâmico para depois do último ponto de quebra, torná-lo
determinístico, ou removê-lo. Lembre também do mínimo cacheável (alguns milhares de tokens,
dependendo do modelo): prefixos curtos demais simplesmente não cacheiam, sem erro.

### O modelo não segue instruções

Reveja três coisas. Primeiro, **conflito**: há instruções que se contradizem? Segundo,
**posição**: a instrução importante está enterrada no meio de um prompt longo? Mova-a para o
começo ou o fim. Terceiro, **intensidade**: nos modelos recentes, linguagem agressiva do tipo
"CRÍTICO: VOCÊ DEVE SEMPRE" tende a fazer o modelo sobre-reagir e acionar ferramentas demais —
suavize para "use quando...". E confirme que você de fato especificou o formato de saída; metade
dos "não seguiu instruções" é "seguiu, mas no formato que você não pediu".

### Custo explodindo

Diagnóstico em ordem: (1) você está no modelo certo para a tarefa, ou usando um Opus onde um
Haiku bastaria? (2) o caching está realmente funcionando (veja acima)? (3) você está incluindo
contexto demais "por garantia"? (4) o volume de tokens de _saída_ — que custa mais — está
controlado? Monte um dashboard de custo e alertas; o custo raramente explode de uma vez, ele
sobe aos poucos até alguém reparar.

### Alucinações em produção

Mitigação em camadas: no prompt, peça fontes e permita explicitamente "não sei"; na
arquitetura, dê ao modelo acesso a informação verificável (busca na web, RAG, o grafo do
Graphify) em vez de confiar na memória dele; na saída, valide programaticamente o que for
verificável. Nenhuma camada sozinha resolve; juntas, reduzem muito o problema.

### Latência alta

A latência tem duas partes: o tempo até o primeiro token (dominado por processar a entrada
inteira) e a velocidade de geração depois disso. Para o primeiro, reduza o tamanho da entrada e
use caching — um prefixo cacheado é processado muito mais rápido. Para a percepção, use
**streaming**: o usuário vê o primeiro token cedo. Para trabalho pesado com modelos de ponta em
alto esforço, lembre que turnos podem durar minutos — projete timeouts, streaming e indicadores
de progresso, e estruture o trabalho para que o usuário acompanhe de forma assíncrona.

### Comportamento de ferramentas e integrações

Se uma ferramenta "falha em silêncio", verifique a fronteira: a chave/credencial está correta e
acessível? Sob rede restrita, o host de destino está liberado? Resultados de ferramentas de
servidor (busca na web, por exemplo) não levantam exceção — voltam como um bloco de resultado
com um objeto de erro dentro, então cheque o conteúdo antes de assumir sucesso. E em chamadas
paralelas de ferramentas, devolva todos os resultados numa única mensagem; espalhá-los em
mensagens separadas ensina o modelo, sem querer, a parar de paralelizar.

### Migração de modelo

Ao trocar para um modelo mais novo, confirme o **ID exato** (sem sufixo inventado), remova os
parâmetros legados (`temperature`/`top_p`/`top_k`, `budget_tokens`, prefills) que agora devolvem
400, e ajuste o `max_tokens` e os gatilhos de contexto, porque a contagem de tokens pode mudar
entre gerações. Faça um teste com uma única requisição antes de implantar em massa, e verifique
no `response.model` que o modelo certo foi de fato usado. Trocar o modelo também invalida o
cache existente — a primeira requisição no modelo novo escreve o cache do zero.

## Checklist Final de Integração

Antes de colocar uma aplicação Claude em produção, percorra esta lista:

- [ ] **Modelo escolhido por etapa**, não por hábito — barato onde dá, caro onde precisa.
- [ ] **Prompts estruturados** (papel/tarefa/contexto/formato) e versionados.
- [ ] **Caching ativo e verificado** — prefixo estável primeiro, volátil por último,
      `cache_read_input_tokens` confirmando.
- [ ] **Tratamento de erros** com chain de exceções (retentável vs. não) e backoff.
- [ ] **Validação nas fronteiras** e verificação de todo output antes de agir.
- [ ] **Segredos fora do prompt e do sandbox.**
- [ ] **Conteúdo externo tratado como não confiável** (defesa contra prompt injection).
- [ ] **Streaming** para respostas longas e melhor latência percebida.
- [ ] **Monitoramento de custo** com dashboard e alertas.
- [ ] **Plano de migração de modelo** — IDs exatos, parâmetros legados removidos, teste antes do
      rollout.

### Resumo do Capítulo 7

Integrar não é usar tudo; é escolher conscientemente cada peça por custo, qualidade ou
segurança, e encaixá-las num fluxo coerente — modelo por etapa, cache na arquitetura, validação
nas fronteiras, falha tratada como regra, segredos fora do alcance do modelo. E quando algo
quebra, o diagnóstico é metódico: identifique o sintoma, vá à causa provável, corrija na
fronteira certa. Com a teoria dos capítulos anteriores e o guia deste, você tem o que precisa
para construir sistemas reais — e para consertá-los quando, inevitavelmente, eles pedirem
conserto.

---

# Capítulo 8: Arquitetura de Agentes, Claude Code e MCP em Profundidade

Os capítulos anteriores trataram Claude sobretudo como um modelo que você chama. Este
capítulo desce ao nível em que Claude vira um _sistema_ que age: o loop agêntico, a
orquestração de subagentes, a configuração do Claude Code, a saída estruturada confiável, o
design de ferramentas e MCP, e a gestão de contexto sob pressão. É também a trilha exata
cobrada na certificação **Claude Certified Architect – Foundations (CCA-F)** da Anthropic, e
por isso fechamos o capítulo com o formato do exame e os erros que mais derrubam candidatos.

## O Modelo Mental: uma API stateless e quatro camadas

No fundo de todo produto da Anthropic há uma única coisa: um modelo de linguagem atrás de um
endpoint HTTP em `api.anthropic.com/v1/messages`. Você manda uma lista de mensagens, recebe
uma resposta. Três fatos definem esse motor e não mudam: **o modelo não tem memória entre
chamadas, não roda nenhum loop do lado da Anthropic, e não executa código ou ferramentas por
conta própria**. Tudo o mais é uma camada de disciplina embrulhada em torno desse endpoint.

São quatro camadas que vale reconhecer pelo nome, porque cada uma resolve um problema
diferente:

- **Claude API** — o endpoint HTTP cru mais os SDKs oficiais (Python e TypeScript) que o
  embrulham. É onde você controla tudo na mão.
- **Agent SDK** — o framework que a Anthropic publica _por cima_ da API. Ele roda o loop
  agêntico para você, gera subagentes e traz ferramentas embutidas, para você não reconstruir
  tudo do zero.
- **Claude Code** — o agente de terminal que sobe Claude dentro do seu repositório e lhe dá
  acesso ao sistema de arquivos através de ferramentas como `read`, `write`, `edit`, `bash`,
  `grep` e `glob`.
- **MCP (Model Context Protocol)** — o padrão aberto de como ferramentas e fontes de dados
  externas se expõem a qualquer modelo compatível, de modo que o mesmo conector de banco,
  wiki ou processador de pagamentos funcione nas quatro superfícies sem reescrever a cola.

Guardar esse mapa resolve metade dos problemas de arquitetura: quase toda decisão difícil é,
no fundo, "qual dessas camadas é dona do comportamento que quebrou?".

## O Loop Agêntico

Como o modelo é stateless e não roda loop nenhum, **o seu código é o loop**. Quando você chama
`messages.create` passando uma lista de ferramentas que Claude pode usar, a resposta volta com
um campo decisivo: o **`stop_reason`**. Dois valores governam o que seu código faz em seguida:

- **`end_turn`** — Claude terminou de pensar e o texto da resposta é a resposta final.
- **`tool_use`** — Claude pausou no meio do raciocínio e está pedindo que o _seu_ código execute
  uma função por ele.

No caso de `tool_use`, o ciclo é sempre o mesmo: você lê o nome e os argumentos da função na
resposta, **executa** essa função localmente, **anexa** o resultado de volta ao histórico como
uma mensagem de `tool_result`, e chama `messages.create` de novo com o histórico atualizado.
Claude retoma de onde parou e ou chama outra ferramenta ou finaliza. Esse ciclo — **chamar →
inspecionar → executar → anexar → chamar** — é o agente inteiro.

Os dois erros clássicos têm assinatura inconfundível. Se você **esquece de inspecionar** o
`stop_reason`, o agente fica mudo depois da primeira chamada de ferramenta. Se você **esquece
de anexar** o `tool_result`, o modelo perde o resultado e tenta a mesma ferramenta de novo,
num laço. Diante de um trecho de código que "para sozinho" ou "repete a ferramenta", procure
qual desses dois passos faltou.

## Orquestração Multiagente

Quando um único Claude não basta, entra o padrão coordenador/subagente. Um Claude de topo — o
**coordenador** — quebra o trabalho em pedaços e gera **subagentes** menores e de escopo
restrito para cada pedaço. No Agent SDK, o coordenador faz isso através de uma função embutida,
a **ferramenta `task`**.

O motivo do padrão é **orçamento de contexto**, não velocidade. Um coordenador que precise ler
dez arquivos, rodar cinco buscas e redigir um resumo encheria a própria janela de contexto
antes de terminar. Ao delegar cada leitura a um subagente, o coordenador recebe de volta apenas
o resumo compacto de cada um e mantém o próprio contexto enxuto. Isso conecta diretamente ao
filtro de relevância do Capítulo 3: subagente é relevância aplicada à arquitetura.

Há dois estilos de decomposição que convém distinguir:

- **Decomposição sequencial** — o coordenador termina o subagente 1 antes de gerar o 2. É a
  escolha certa quando cada passo depende da saída do anterior.
- **Decomposição adaptativa** — o coordenador escolhe o próximo subagente com base no que
  voltou. É a escolha certa para trabalho aberto e exploratório (investigar a estratégia de
  preços de um concorrente, por exemplo).

E quando um subagente falha no meio, a recuperação certa é quase sempre um **retry direcionado
apenas do que falhou** — nunca rerodar todos. Os subagentes bem-sucedidos podem já ter gravado
linhas num banco ou disparado e-mails; rerodá-los dispararia esses efeitos colaterais uma
segunda vez. (Este é o mesmo princípio de "falha é regra" do Capítulo 7, aplicado a agentes.)

## Configuração do Claude Code

O Claude Code roda Claude direto dentro do seu repositório e, por padrão, **cada nova sessão
começa sem nenhuma memória do projeto**. O arquivo **`CLAUDE.md`** é o que conserta isso.
Quando o Claude Code inicia, ele sobe a árvore de diretórios a partir da pasta atual, encontra
todo arquivo `CLAUDE.md` no caminho e os costura no _system prompt_ da sessão.

A hierarquia tem três níveis, e a confusão entre eles é justamente o que o exame adora cobrar:

- **Nível de usuário** (`~/.claude/CLAUDE.md`) — aplica-se a todo projeto na sua máquina e
  nunca entra no controle de versão. O que está aqui é só seu.
- **Nível de projeto** (`CLAUDE.md` na raiz do repositório) — é commitado e viaja com cada
  clone. É onde moram as convenções do time ("todos usam `bun` em vez de `npm`").
- **Nível de diretório** (ex.: `packages/api/CLAUDE.md`) — regras que só valem quando Claude
  mexe naquela subpasta específica.

A pegadinha canônica: onde deve ficar a regra "todo mundo roda o lint antes do commit"? No
**arquivo de projeto, na raiz do repositório**. Pôr no arquivo de usuário significa que o clone
do seu colega nunca verá a regra.

### Slash commands

Um **slash command** é um prompt reutilizável que você invoca digitando uma barra (`/review`,
por exemplo). Você define um deles soltando um arquivo Markdown em `.claude/commands`. O corpo
do arquivo é o prompt injetado quando o comando roda; o **front matter YAML** no topo é onde
você restringe o comportamento. O campo **`allowed-tools`** limita quais ferramentas o comando
pode chamar — um comando de revisão pode ser travado em ferramentas somente-leitura (`read`,
`grep`), sem acesso a `write` ou `bash`. O campo **`argument-hint`** é o placeholder mostrado no
seletor, documentando que argumento o comando espera.

### Execução headless em CI/CD

O Claude Code aceita a flag **`-p`**, que o roda de forma não interativa com um prompt embutido.
Combinada a um _schema_ de saída estruturada, ela permite que o seu pipeline peça a Claude para
revisar um pull request e devolver um objeto JSON com categorias de violação nomeadas. A lição
de arquitetura que o exame repete: o pipeline deve **falhar apenas em categorias que o schema
nomeia explicitamente** (`security_violation`, `breaking_api_change`), nunca quando Claude
apenas "expressa uma preocupação vaga". Uma revisão que falha por vaguidão vira ruído que o time
aprende a ignorar em uma semana — e uma revisão ignorada é pior que revisão nenhuma.

## Saída Estruturada Confiável

O Capítulo 2 mostrou que pedir JSON "na prosa" funciona na maior parte das vezes — até a manhã
em que o modelo prefixa um "Claro, aqui está o seu JSON" e o seu parser explode em produção. A
forma confiável é **declarar o formato de saída como uma ferramenta**: você define uma tool cujo
campo **`input_schema`** descreve exatamente o JSON desejado, passa essa tool no array de
ferramentas e usa **`tool_choice`** para forçar o modelo a chamá-la. Em vez de texto livre, o
modelo devolve um bloco `tool_use` cujo campo `input` já é o JSON em conformidade com o schema.
Funciona muito melhor porque a Anthropic treinou Claude intensamente contra schemas de
ferramenta — a taxa de conformidade em `tool_use` é bem mais alta do que em instruções de
formato no texto.

Ainda assim, schemas às vezes derivam, e o padrão para isso é um **loop de validação e retry**.
A cada resposta, você passa o JSON pelo seu validador. Quando falha, **não retente às cegas**:
pegue a mensagem de erro do validador e anexe-a ao histórico como um turno de usuário ("sua
resposta anterior falhou na validação com este erro específico; devolva uma versão corrigida") e
chame o modelo de novo. Agora o modelo tem em contexto a própria saída anterior e o campo exato
que quebrou — a taxa de correção na segunda tentativa fica acima de 90% na prática.

Para volume, há a **Message Batches API**: você submete milhares de prompts independentes e
recolhe os resultados em até 24 horas, a 50% do custo por token das chamadas em tempo real.
Diante de "classificar 100 mil documentos da noite para o dia", a escolha é batch — sempre que o
trabalho é offline, a latência pode ser medida em horas e os prompts são independentes (a ordem
não importa).

## Design de Ferramentas e MCP

O **MCP** já apareceu no Capítulo 4 como o protocolo aberto que padroniza como ferramentas e
fontes externas se expõem ao modelo. Antes do MCP, cada integração era um _wrapper_ específico
de um modelo; com o MCP, o modelo fala um único protocolo e qualquer servidor compatível — banco,
wiki, processador de pagamentos, sua API interna — pluga sem mudar o código do modelo. Três
pontos merecem profundidade.

**Descrições de ferramenta decidem a escolha da ferramenta.** No momento da decisão, o modelo só
enxerga, de cada tool, o nome, a descrição e o `input_schema`. Se você publica três ferramentas
`get_user`, `lookup_user` e `find_customer`, todas descritas como "retorna informações do
usuário", o modelo escolhe praticamente no chute — e o exame pontua isso como um **defeito de
schema**, não como falha do modelo. Escreva descrições como quem escreve doc de API: uma frase
sobre o que a ferramenta faz, uma frase sobre quando usar esta e não as irmãs (a regra de
desambiguação), um exemplo de invocação com argumentos realistas, e a lista de condições de erro
que ela pode devolver.

**Respostas de erro estruturadas.** Quando uma ferramenta falha, o formato certo é um objeto
JSON em que o modelo possa ramificar: um booleano `is_error` (para saber que caiu no caminho de
erro), uma string `category` (`rate_limited`, `not_found`…) para escolher a recuperação pelo
nome, um booleano `retryable` e, quando couber, um inteiro `retry_after_ms`. Com essa forma, o
modelo escreve um plano de recuperação limpo num passo só. Sem ela, recebe uma string de exceção
crua e ou desiste ou entra num loop de retry infinito que você só descobre no log às 3 da manhã.

**Seleção de transporte.** O MCP suporta dois transportes. O **STDIO** roda o servidor na mesma
árvore de processos do cliente, por entrada/saída padrão: zero latência de rede, zero overhead de
autenticação. O **SSE** (server-sent events) roda o servidor em outra máquina e o cliente conecta
por HTTP, o que adiciona idas e voltas de rede e obriga a desenhar autenticação. A regra é
direta: use **STDIO sempre que o servidor puder viver na mesma máquina** do cliente; use **SSE só
quando o servidor precisar morar em outro lugar** (um conector central de banco servindo vários
usuários a partir de um host compartilhado, por exemplo). Escolher SSE para um servidor em
`localhost` é uma armadilha clássica.

## Gestão de Contexto e Confiabilidade

**O efeito "perdido no meio" (lost in the middle).** É um achado empírico que se mantém em todas
as famílias de modelo, Claude incluído: quando você enche a janela com uma conversa ou documento
longo, o modelo presta muita atenção ao que está bem no **começo** e bem no **fim**, e quase
nenhuma ao que está no **meio**. O sintoma de exame: um agente de suporte conversou por 40 turnos
e de repente "esquece" o número de conta dado no turno 3. A correção certa é identificar os
**fatos duráveis** (números de conta, IDs de pedido, valores de reembolso), copiá-los para um
pequeno bloco estruturado — muitas vezes chamado de **case block** — e **reancorar esse bloco no
fim do contexto a cada turno**, para que a atenção do modelo sempre caia sobre ele. A tentação é
pedir ao modelo que _resuma_ a conversa inteira — e esse é exatamente o distrator: a sumarização
comprime com heurísticas com perda e silenciosamente derruba números e IDs.

**Prompt caching: o que cachear.** O mecanismo já foi detalhado no Capítulo 3 (prefixo exato,
~90% de desconto na leitura, TTL de 5 minutos ou 1 hora). O ângulo arquitetural aqui é _o que_
vale cachear: o **system prompt** vale, porque é idêntico em toda chamada; os **exemplos
few-shot** valem, pela mesma razão; o **turno do usuário no fim não vale**, porque cada usuário
digita algo diferente e cachear isso só queima orçamento em entradas que nunca mais batem. Ordene
o prompt do estável (no começo, cacheado) ao volátil (no fim).

**Escalonamento: agentes que sabem quando não sabem.** Um agente que responde com confiança a uma
pergunta que deveria ter escalado é uma falha _pior_ do que um agente que escala cedo demais — o
primeiro entrega respostas erradas ao cliente; o segundo só custa alguns tíquetes de revisão
humana, e essa é uma troca que você aceita o dia inteiro. O padrão a reconhecer é uma **checagem
explícita de confiança** ao fim do raciocínio do agente, com dois gatilhos: ou a confiança
declarada cai abaixo de um limiar, ou o modelo detecta uma ambiguidade que não consegue resolver
pelo histórico. Qualquer um dos gatilhos passa a conversa a um humano, junto de um resumo
estruturado do que o agente sabia, do que tentou e de onde travou.

## O Exame CCA-F: Formato e os Seis Cenários

A **Claude Certified Architect – Foundations** é a primeira certificação oficial da Anthropic,
aplicada online com supervisão (proctored) pela Anthropic Academy. O formato: **60 questões** de
múltipla escolha baseadas em cenário, **120 minutos** (cerca de 2 minutos por questão), e nota de
corte de **720/1000** (em torno de 75%). As questões não testam decoreba: cada uma joga você
dentro de um sistema em funcionamento — em geral já quebrando — e pergunta qual decisão
arquitetural é a certa naquelas circunstâncias.

Os **cinco domínios** e seus pesos:

| Domínio | Peso | Onde mora |
| --- | --- | --- |
| 1. Arquitetura agêntica e orquestração | 27% | API e Agent SDK |
| 2. Configuração e workflows do Claude Code | 20% | Claude Code |
| 3. Prompt engineering e saída estruturada | 20% | API |
| 4. Design de ferramentas e integração MCP | 18% | MCP |
| 5. Gestão de contexto e confiabilidade | 15% | todo o sistema |

A Anthropic escreveu **seis cenários de produção**; o sistema serve **quatro ao acaso**, e todas
as suas 60 questões se ancoram nesses quatro. Você estuda os seis porque não escolhe quais cairão:
(1) **agente de suporte ao cliente** — onde caem escalonamento, retries e falha parcial; (2)
**integração de time no Claude Code** — hierarquia de `CLAUDE.md` e slash commands; (3) **pipeline
de pesquisa multiagente** — orquestração coordenador/subagente e estouro de contexto; (4)
**ferramentas de produtividade do dev** — ferramentas embutidas (`read`/`write`/`edit`/`bash`/
`grep`/`glob`) e quando preferir um servidor MCP; (5) **Claude Code em CI/CD** — invocação
headless, saída estruturada e minimização de falsos positivos; (6) **extração de dados
estruturados** — JSON via schema e Message Batches.

### Os cinco distratores que mais derrubam

O mesmo punhado de armadilhas reaparece pelo banco de questões. Reconhecê-las de imediato vale
pontos:

- **Adjetivo vago** — qualquer opção que enfia "cuidadoso" ou "minucioso" num prompt para
  resolver falso positivo. A resposta certa troca o adjetivo por uma **lista de regras
  categóricas** verificáveis uma a uma.
- **Sumarização** — comprimir uma conversa longa num cenário que carrega IDs e valores. A
  resposta certa extrai os fatos duráveis para um **case block** no fim da janela.
- **Retry geral** — rerodar todos os subagentes depois que um falhou, quando os outros já
  executaram efeitos colaterais. A resposta certa **retenta só o que falhou**.
- **`CLAUDE.md` de usuário** — pôr configuração compartilhada do time no arquivo de usuário (que
  nunca chega ao controle de versão). A resposta certa usa o **arquivo de projeto na raiz**.
- **SSE em localhost** — escolher transporte SSE para um servidor que roda na mesma máquina do
  cliente. A resposta certa é **STDIO**.

### Resumo do Capítulo 8

Claude é uma API stateless embaixo de quatro camadas — API, Agent SDK, Claude Code e MCP — e
quase todo problema de arquitetura é descobrir qual camada é dona do comportamento quebrado. O
loop agêntico é o seu código inspecionando `stop_reason` e anexando `tool_result`; subagentes
existem para poupar contexto, com decomposição sequencial ou adaptativa e retry só do que falhou.
No Claude Code, regras vão na camada certa do `CLAUDE.md` e comandos são travados por
`allowed-tools`. Saída confiável vem de declarar o schema como tool com `tool_choice` mais um loop
de validação e retry. Boas ferramentas têm descrições de doc de API, erros estruturados e o
transporte certo (STDIO local, SSE remoto). E confiabilidade é reancorar fatos duráveis contra o
"perdido no meio", cachear só o estável e escalar para humano quando a confiança cai. É essa a
trilha que o exame CCA-F cobra — e que você passa por já ter construído cada peça.

---

# Capítulo 9: Claude Code na Prática — Instalação, Custos e Operação

O Capítulo 8 tratou da _arquitetura_ do Claude Code: o loop, os subagentes, o `CLAUDE.md`. Este
capítulo é o lado operacional do dia a dia — instalar, configurar, vigiar o custo, manter o
contexto sob controle, automatizar auditorias e integrar a ferramenta ao seu editor. É o
conhecimento que separa "consegui rodar" de "uso isto o dia inteiro sem susto na fatura".

## Instalação e Configuração Inicial

O Claude Code é um agente de linha de comando que roda em macOS, Linux e Windows. A instalação
mais simples baixa e executa o script oficial:

```bash
curl -fsSL https://claude.ai/install.sh | bash
```

(Há também a via `npm install -g @anthropic-ai/claude-code` para quem já vive no ecossistema
Node.) Depois de instalado, você roda `claude` dentro de um repositório e autentica na primeira
execução. A personalização de tema de cores, modelo padrão e idioma é feita de forma interativa
pelo comando **`/config`** dentro do console — não é preciso editar arquivo na mão para o
básico.

Para estender o agente com instruções e habilidades reutilizáveis que valem em _todos_ os seus
projetos, existe a pasta de **skills globais** em `~/.claude/`. Uma skill de administração de
sistemas, por exemplo, pode viver ali e ficar disponível em qualquer sessão, sem repetir
instruções a cada projeto. (Lembre da hierarquia do Capítulo 8: o que é pessoal e global fica em
`~/.claude/`; o que é do time vai versionado no repositório.)

## O Arquivo `settings.json`

Diferente dos arquivos `CLAUDE.md` — que são instruções em linguagem natural por escopo de
diretório —, o arquivo **`~/.claude/settings.json`** guarda parâmetros estritamente
operacionais em JSON: o modelo padrão, as **permissões** de ferramentas (o que o agente pode
rodar sem perguntar e o que exige confirmação), variáveis de ambiente, e os **hooks** (que
veremos adiante). Há também um `settings.json` por projeto, em `.claude/settings.json`, para
configurações que devem viajar com o repositório. O comando **`/config`** é o atalho interativo
para ajustar as preferências mais comuns sem abrir o arquivo.

A lição de governança aqui é a mesma do `CLAUDE.md`: separe o que é pessoal (no
`~/.claude/settings.json`, fora do versionamento) do que é do time (no `.claude/settings.json`
do projeto, commitado).

## Gerenciamento de Custos e Contexto na CLI

Controlar gasto e saturação de contexto é o que mantém o fluxo saudável em escala. O Claude Code
traz instrumentos para isso embutidos no terminal.

Para **custo**, o comando **`/cost`** mostra o gasto acumulado na sessão ativa — útil para
perceber, ao vivo, quando uma tarefa está saindo cara. Para uma visão consolidada e histórica
(por dia, por projeto, por chave de API), o lugar certo é o **Anthropic Console** na web, cujo
painel detalha o consumo em gráficos; atribuir **uma chave de API por projeto** permite auditar
as despesas de cada repositório separadamente.

Para **contexto**, o rodapé/status do console exibe um **indicador de saturação** — quanto da
janela já está ocupado. Quando o contexto fica pesado, dois comandos resolvem:

- **`/compact`** resume as mensagens anteriores e libera espaço imediato, **sem perder as
  diretrizes principais** da sessão. Use no meio de uma tarefa longa que ainda continua.
- **`/clear`** esvazia por completo a memória ativa e zera o consumo da sessão. Use ao
  **terminar uma tarefa e começar outra** sem relação — é mais barato e mais limpo do que
  arrastar o histórico antigo adiante.

A regra prática: `/compact` quando a _mesma_ tarefa continua, `/clear` quando a tarefa _muda_.
Você pode ainda incluir, no `CLAUDE.md`, diretrizes que limitem o volume de leitura simultânea
de arquivos, forçando o agente a ser econômico com contexto por padrão.

## Hooks, Observabilidade e Auditoria

A automação de qualidade vem dos **hooks**: gatilhos configurados no `settings.json` que
executam comandos seus em momentos definidos do ciclo do agente — antes de uma ferramenta rodar
(`PreToolUse`), depois (`PostToolUse`), ao finalizar uma resposta, e assim por diante.
Diferente de uma instrução no `CLAUDE.md` (que o modelo _pode_ seguir), um hook é executado pelo
_harness_, deterministicamente — é assim que você garante, por exemplo, que o lint roda depois
de toda edição, sem depender da boa vontade do modelo.

Para **observabilidade**, a estratégia é definir rotinas de checagem do estado do projeto a
partir dos dados que o próprio repositório oferece: o `diff` do Git, o histórico de commits, os
logs. Incluir no `CLAUDE.md` instruções de auditoria — "ao revisar, examine os commits recentes,
gere uma tabela de progresso e aponte riscos ou vulnerabilidades" — faz o agente produzir
relatórios estruturados de conformidade. Um **subagente de auditoria** dedicado (Capítulo 8)
isola esse trabalho, comparando o estado atual com os requisitos formais sem poluir o contexto
principal. E relatórios gerados assim podem ser **direcionados a canais externos** — um canal
corporativo no Slack, um repositório central de documentação — para governança de time.

## Automação e Agendamento

Rotinas recorrentes não precisam de gatilho humano. O Claude Code permite **agendar execuções**
— auditorias periódicas de integridade da base, geração de logs de conformidade, sincronização
de backups dos materiais para um destino seguro. O padrão de **validação contínua** é
especialmente valioso: após alterações no código, o agente roda a suíte de testes, audita os
resultados e aplica correções quando algo falha, fechando o ciclo sem intervenção. (Isto é o
loop agêntico do Capítulo 8 posto a serviço de manutenção, não de desenvolvimento.)

## Integração com Editores

A produtividade sobe quando o Claude Code convive com o seu ambiente de desenvolvimento. Rodar o
console no **terminal embutido do VS Code ou do Cursor** dá feedback visual imediato na árvore de
arquivos: você vê o agente ler e modificar o código enquanto acompanha os testes. Há ainda
extensões de IDE dedicadas que aprofundam essa integração.

Para sessões longas ou conexões remotas instáveis, o multiplexador **`tmux`** garante a
persistência: a sessão do Claude Code sobrevive a uma queda de SSH e você alterna facilmente
entre o painel do agente e a compilação manual. É o detalhe operacional que evita perder uma
tarefa de uma hora por causa de uma rede ruim.

### Resumo do Capítulo 9

Instale com o script oficial e configure pelo `/config`; deixe o pessoal em `~/.claude/` e o do
time versionado no projeto. O `settings.json` guarda o operacional (modelo, permissões, env,
hooks); os `CLAUDE.md`, as instruções. Vigie custo com `/cost` e o Anthropic Console, e contexto
com o indicador de saturação mais `/compact` (mesma tarefa) ou `/clear` (tarefa nova). Use hooks
para garantir checagens deterministicamente, subagentes e instruções de auditoria no `CLAUDE.md`
para observabilidade, e agendamento para validação contínua e backups. E integre ao VS
Code/Cursor, com `tmux` para sessões que não podem cair.

---

# Apêndice

## Glossário de Termos

- **Token**: a unidade em que o texto é fragmentado para o modelo; a moeda de cobrança e a
  unidade da janela de contexto.
- **LLM (Large Language Model)**: modelo treinado para prever o próximo token; o "motor".
- **Context Window**: o máximo de tokens (entrada + saída) que o modelo considera de uma vez.
- **Prompt Caching**: reuso do processamento de um prefixo estável de prompt; ~90% de desconto
  na leitura, ~25% a mais na escrita.
- **Constitutional AI**: método de treinamento da Anthropic em que o modelo critica e revisa as
  próprias respostas segundo princípios explícitos.
- **Few-shot learning**: ensinar a tarefa por meio de exemplos no próprio prompt.
- **Chain-of-thought**: pedir ao modelo que raciocine em passos antes de responder.
- **Adaptive thinking**: raciocínio interno em que o modelo decide quando e quanto pensar,
  controlado por um nível de esforço; substitui o orçamento fixo de tokens de pensamento.
- **Effort (esforço)**: parâmetro (`low`/`medium`/`high`/`xhigh`/`max`) que controla a
  profundidade do raciocínio e o gasto de tokens nos modelos recentes.
- **MCP (Model Context Protocol)**: protocolo padrão para conectar modelos a ferramentas e
  fontes de dados externas.
- **Prompt injection**: ataque em que instruções maliciosas são inseridas em dados que o modelo
  processa.
- **Streaming**: receber a resposta token a token, em vez de esperar o fim.
- **Knowledge graph**: representação do conhecimento como nós e arestas; no Graphify, a estrutura
  consultável extraída de um codebase.
- **Loop agêntico**: o ciclo, rodado pelo seu código, de chamar o modelo, inspecionar o
  `stop_reason`, executar a ferramenta pedida e anexar o `tool_result` antes de chamar de novo.
- **`stop_reason`**: campo da resposta da API que diz se o modelo terminou (`end_turn`) ou está
  pedindo a execução de uma ferramenta (`tool_use`).
- **Agent SDK**: framework da Anthropic, por cima da API, que roda o loop agêntico, gera
  subagentes e traz ferramentas embutidas.
- **Subagente**: Claude de escopo restrito gerado por um coordenador (via ferramenta `task`)
  para isolar uma subtarefa e poupar contexto.
- **`tool_choice` / `input_schema`**: forma confiável de obter saída estruturada — declarar o
  formato como uma ferramenta e forçar o modelo a chamá-la.
- **Lost in the middle**: tendência do modelo a atender ao começo e ao fim do contexto e
  ignorar o meio; mitigada reancorando fatos duráveis num *case block* no fim da janela.
- **STDIO / SSE**: os dois transportes do MCP — STDIO para servidor na mesma máquina, SSE para
  servidor remoto via HTTP.
- **CCA-F**: Claude Certified Architect – Foundations, a primeira certificação oficial da
  Anthropic (60 questões, 5 domínios).

## Recursos Úteis

- **Documentação oficial da Anthropic** — a fonte de verdade para IDs de modelo, preços, limites
  e recursos novos. Consulte sempre que o cenário puder ter mudado.
- **Comunidades** — Discord e Reddit dedicados, onde práticas e armadilhas circulam rápido.
- **Ferramentas recomendadas** — Graphify (grafo de conhecimento para codebases), SDKs oficiais
  da Anthropic para as principais linguagens, e a CLI `ant` para acesso ao painel pelo terminal.

## Feedback e Contribuições

Esta apostila é um documento vivo. Erros, omissões e sugestões de melhoria são bem-vindos — o
campo de IA, e Claude em particular, evolui rápido o suficiente para que nenhuma apostila fique
"pronta" por muito tempo. Ao reportar um problema, indique o capítulo e a seção, descreva o que
está incorreto ou faltando, e, se possível, proponha a correção. Modelos, preços e recursos
mudam; manter o material atualizado é trabalho coletivo.
