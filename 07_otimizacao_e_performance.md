# Capítulo 7: Otimização e Performance

Este é o capítulo que paga a conta. Tudo até aqui foi sobre fazer Claude trabalhar; agora é sobre fazê-lo trabalhar **barato e rápido**, sem perder qualidade. São dois eixos que costumam ser confundidos: **custo** (quantos tokens você gasta) e **latência/performance** (quão rápido a resposta chega). As técnicas aqui têm efeito direto na fatura e na experiência — e a maioria não custa nada além de atenção.

## Pensar antes de gastar: um bom plano economiza tokens

Antes de qualquer truque de compressão, a maior economia costuma vir de uma coisa banal: **planejar antes de mandar executar**. Os tokens mais caros são os que você gasta indo pelo caminho errado — o agente lê dez arquivos que não importavam, escreve código que resolve o problema errado, e você paga por tudo isso mais o retrabalho. Um minuto de plano evita dez de execução perdida.

O padrão é: peça primeiro um **plano**, revise-o, e só então autorize a execução. No Claude Code, é literalmente um modo — o *plan mode* (acionado com `Shift+Tab`), em que o agente propõe a abordagem sem tocar em nada até você aprovar. Um bom plano cabe em poucas linhas e contém:

- **Objetivo** — o que se quer, em uma frase, com critério de "pronto" verificável.
- **Escopo** — quais arquivos/áreas serão tocados, e o que fica **fora**.
- **Passos** — a sequência de ações, na ordem.
- **Restrições** — o que não fazer (não mexer em migrations, não criar dependências novas).

Planejar não é burocracia; é a forma mais barata de filtro de relevância — decide o que entra no trabalho antes de o trabalho custar tokens.

## Otimização de Tokens e Contexto

### Técnicas de Redução de Tokens

Como você paga por token, reduzir tokens é reduzir custo. Quatro alavancas: **prompt conciso** (corte a gordura), **relevance filtering** (inclua só o contexto necessário, não o documento inteiro), **batch processing** (agrupe requisições — detalhado adiante) e **formatos estruturados** (JSON ou TOON costumam ser mais compactos e parseáveis que prosa; veja o TOON no Cap. 4). O maior ganho geralmente está no filtro de relevância — a tentação de "jogar tudo no contexto por garantia" é cara e, pior, dilui a atenção do modelo (o "perdido no meio" do Cap. 1).

### CLAUDE.md e contexto enxuto: o ângulo do custo

O `CLAUDE.md` e a família `.claude` já foram apresentados no Capítulo 3 (o que são, onde ficam, como escrevê-los). Aqui o ângulo é outro: **como usá-los para gastar menos token**. O `CLAUDE.md` entra no contexto de _toda_ chamada da sessão, então cada linha dele é paga repetidamente — é o lugar onde a economia mais rende.

- **Mantenha-o enxuto.** A regra de bolso de ~300 linhas do Capítulo 3 é também uma regra de custo: um `CLAUDE.md` inchado é imposto fixo sobre cada requisição. Prefira um mapa a uma enciclopédia.
- **Divulgação progressiva.** Detalhe que só importa às vezes vai para **skills** (carregadas sob demanda) ou para `CLAUDE.md` de subpasta (só entra quando o agente mexe ali), em vez de pesar no contexto global o tempo todo.
- **Instruções que economizam processamento.** O próprio `CLAUDE.md` pode conter diretrizes de economia — "prefira `grep`/`glob` a ler arquivos inteiros", "leia no máximo os arquivos relevantes à tarefa", "não repita o conteúdo de arquivos já lidos". São instruções que reduzem o volume que o agente puxa para o contexto a cada passo.

E o `CLAUDE.md` **dispara caching**? No **Claude Code, sim, automaticamente**: o prefixo estável da sessão (system prompt + `CLAUDE.md`) é cacheado sem você fazer nada — por isso mantê-lo estável importa, já que qualquer alteração invalida o cache (veja a seguir). Na **API**, o caching não é automático: você marca os pontos de quebra com `cache_control` você mesmo.

### Prompt Caching

Esta é, provavelmente, a técnica de maior impacto financeiro do capítulo. **Prompt caching** permite reaproveitar o processamento de um prefixo de prompt que se repete entre requisições — um system prompt grande, um conjunto de exemplos, um documento de referência.

O mecanismo funciona por **prefixo exato**: o cache é indexado pelos bytes do prompt até um ponto de quebra (*cache breakpoint*). A consequência prática, que merece estar tatuada: **qualquer alteração de um byte no prefixo invalida o cache de tudo o que vem depois**. Um timestamp no system prompt, um JSON com chaves em ordem não-determinística, uma ferramenta a mais na lista — qualquer um desses zera o cache silenciosamente.

A economia é concreta. Uma **leitura de cache** custa cerca de 10% do preço normal de entrada (um desconto de ~90%). Uma **escrita de cache** custa um pouco mais que o normal: 1,25× para o cache de 5 minutos, 2× para o de 1 hora. O ponto de equilíbrio é rápido — com o cache de 5 minutos, duas requisições já compensam. O cache tem TTL (tempo de vida) e limite de pontos de quebra (no máximo 4). Vale a pena sempre que você reusa um prefixo grande e estável — e, no Claude Code, isso inclui o `CLAUDE.md`, cacheado por padrão.

Como verificar se está funcionando? A resposta da API traz, no objeto `usage`, os campos `cache_read_input_tokens` e `cache_creation_input_tokens`. Se as leituras de cache estão sempre em zero apesar de prefixos idênticos, há um invalidador silencioso escondido — quase sempre um `datetime.now()` ou um UUID no começo do prompt.

### Memória: contexto que sobrevive à sessão

Contexto e memória são coisas diferentes, e confundi-las custa caro. **Contexto** é o que está na janela agora; some quando a conversa acaba. **Memória** é o que persiste entre sessões — preferências do usuário, fatos aprendidos, decisões anteriores. Os sistemas de memória dão ao modelo um lugar para escrever e ler ao longo do tempo (um diretório de arquivos, como o `MEMORY.md`), com continuidade de sessão e poda periódica (*memory pruning*) para não inchar. A regra de bolso: use contexto para o que importa agora, memória para o que precisa sobreviver ao "agora".

**A gente paga por memória?** Não há uma taxa de "armazenamento" — a memória são apenas arquivos. Você paga pelos **tokens que ela ocupa quando é carregada no contexto**. Ou seja, memória barata é memória enxuta: um `MEMORY.md` que só cresce vira imposto sobre toda sessão que o carrega, além de causar a "podridão de contexto" (*context rot*) do Capítulo 8.

**Dá para apagar toda a memória de uma vez?** Sim. No nível da sessão, `/clear` zera o contexto ativo (não a memória persistente). No nível persistente, basta apagar o `MEMORY.md` e os arquivos de memória — o que se perde é a **continuidade entre sessões**: preferências, decisões e fatos que o agente havia acumulado. Vale a pena quando a memória apodreceu (ficou grande, contraditória ou desatualizada e passou a atrapalhar mais que ajudar), quando você recomeça um trabalho sem relação com o anterior, ou por privacidade. O hábito saudável é **podar com frequência**, não deixar acumular até precisar do reset total.

## Economia e Custos

### Preço, cobrança e o mito do "horário barato"

A cobrança é por token, separada entre **entrada** e **saída**, e a saída custa várias vezes mais (reveja a tabela do Capítulo 2). Os modelos têm preços muito diferentes: o Haiku 4.5 é ordens de magnitude mais barato que o Fable 5. 

Uma dúvida comum: **existem horas mais baratas** (como em tarifa de energia)? **Não.** A Anthropic não pratica preço por horário — não há um "fuso barato" em GMT para mirar. As alavancas reais de desconto são três, e nenhuma depende do relógio: o **batch processing** (50% de desconto, abaixo), o **prompt caching** (~90% na leitura) e a **escolha do modelo** pela tarefa.

### Qual modelo para qual tarefa

A primeira pergunta de qualquer otimização de custo é: *"esta tarefa precisa mesmo do modelo mais caro?"*. Na prática, o roteamento por tarefa é o que mais economiza (detalhes de cada modelo no Capítulo 2):

| Modelo | Bom para | Exagero para |
| --- | --- | --- |
| **Haiku 4.5** | classificação, extração, triagem, respostas curtas em alto volume | raciocínio de vários passos |
| **Sonnet 5** | o "cavalo de batalha": a maioria das tarefas, bom equilíbrio | classificação trivial em massa (use Haiku) |
| **Opus 4.8** | código difícil, raciocínio complexo, trabalho agêntico | resumir um e-mail (use Haiku/Sonnet) |
| **Fable 5** | o raciocínio mais exigente e o agêntico de longo horizonte | qualquer coisa que Sonnet já resolva |

O padrão de ouro em produção é o **roteamento em cascata**: comece pelo modelo barato e só **escale** para o caro quando a tarefa (ou uma checagem de confiança) exigir. Classificar com Haiku e mandar ao Opus só os casos difíceis corta a conta sem custar qualidade onde ela importa.

### Batch Processing

Quando o trabalho é **offline, tolerante a latência e composto de prompts independentes**, a Message Batches API é a alavanca mais direta: você submete milhares de requisições de uma vez e recolhe os resultados em até **24 horas**, por **50% do preço** por token das chamadas em tempo real. Casos típicos: "classificar 100 mil documentos da noite para o dia", gerar embeddings de um acervo, avaliar um conjunto grande de prompts. O uso é simples — você monta um lote de requisições (cada uma com um `custom_id` para casar entrada e saída), submete, e consulta o status até concluir. A regra de decisão: se a ordem não importa, os prompts não dependem uns dos outros e você pode esperar horas, é batch — sempre.

### Trade-offs de Custo

O eixo central é **modelo versus desempenho**, já tratado acima. Há também os eixos qualidade versus custo, latência versus economia (batch é barato mas lento), e automação versus trabalho manual. Nenhum tem resposta universal; a resposta é **medir no seu caso**.

### Combinar fornecedores: roteamento multi-modelo

E se você tiver **créditos sobrando em outro fornecedor**? Roteá-los é uma estratégia legítima: mandar as tarefas simples para onde está mais barato (ou onde você já pagou), e reservar Claude para o que ele faz melhor. O ponto de arquitetura — o mesmo do MCP e de arneses como o Pi (Cap. 8) — é **manter o fluxo do agente independente do fornecedor do modelo**, atrás de uma interface única, para trocar de modelo (ou de vendor) sem reescrever a cola. As ressalvas são honestas: créditos e cotas normalmente **não transferem** entre provedores nem "sobram" para o mês seguinte; cada modelo tem capacidades e formatos de prompt distintos; e manter dois caminhos custa complexidade. Vale quando o volume justifica; não vale para um projeto pequeno, onde a simplicidade ganha.

### ROI de Automação

Automatizar com Claude tem retorno calculável. Estime o custo por tarefa automatizada, compare com o custo do trabalho manual equivalente, e encontre o ponto de equilíbrio (*break-even*). Em escala, a economia cresce de forma não-linear — o que justifica investir em otimização de prompts e caching para tarefas de alto volume.

### Monitoramento e dashboards de custo

Custo não é um problema que se resolve uma vez. Há dois níveis de monitoramento, e vale saber onde cada um mora:

- **Pronto e online:** o **Anthropic Console** já traz um painel de uso e custo (por dia, por modelo, por chave de API) — é onde você olha primeiro, sem construir nada. Atribuir **uma chave de API por projeto** faz esse painel separar as despesas por repositório. No Claude Code, o `/cost` mostra o gasto da sessão ao vivo (Cap. 3).
- **Sob medida e "offline":** para dashboards próprios e alertas de orçamento, você coleta os dados de consumo — o objeto `usage` que vem em cada resposta da API, ou a Usage/Cost API administrativa — e os joga no seu stack de observabilidade (um banco + Grafana, por exemplo). É o caminho quando você quer alertas automáticos ("me avise se o gasto diário passar de X") e correlação com métricas de negócio.

Monitorar é como você percebe, antes da fatura, que os workloads mudaram, os prompts incharam, ou um modelo mais caro entrou "só para testar" e nunca saiu.

## Latência e Performance

Custo é metade de "otimização"; a outra é **velocidade**. Dois números governam a experiência (ambos introduzidos no Cap. 2): o **first token latency** (quanto demora até o primeiro token aparecer, dominado pelo processamento da entrada inteira) e a **token generation speed** (quão rápido os tokens saem depois). Cinco alavancas mexem neles:

- **Streaming.** Receber a resposta token a token não a torna mais rápida, mas faz o **primeiro token** aparecer cedo — a percepção de velocidade melhora muito. Para respostas longas (acima de ~16K tokens) é praticamente obrigatório, para não esbarrar em timeout.
- **Modelo menor.** Haiku responde mais rápido que Opus. Onde a tarefa não exige o topo, o modelo barato também é o veloz.
- **Prompt mais curto.** O first token latency cresce com o tamanho da entrada — cada token de contexto é processado antes de o primeiro token sair. Contexto enxuto é resposta mais rápida, não só mais barata.
- **Caching.** Um prefixo cacheado não é reprocessado, o que reduz também a latência do primeiro token, não só o custo.
- **Esforço adequado.** Nível de esforço (`effort`) alto significa mais tokens de raciocínio interno antes da resposta — melhor qualidade, maior latência. Use `low`/`medium` onde a tarefa não precisa pensar muito (Cap. 4).

A escolha estrutural que resume tudo: **tempo real** (streaming, modelo certo, prompt enxuto) quando um humano espera; **batch** quando ninguém está esperando.

## Multimodalidade

### Imagens como Input

Claude aceita imagens, não só texto. Os formatos comuns (PNG, JPEG, etc.) são suportados, e a **resolução** importa em dois sentidos: alta demais desperdiça tokens, baixa demais perde detalhe. Os modelos recentes têm suporte a alta resolução, o que ajuda em tarefas visuais densas — mas imagens grandes consomem mais tokens de imagem. Otimize o tamanho para o nível de detalhe que a tarefa realmente exige.

### Casos de Uso

As imagens abrem casos práticos: **OCR e extração de texto** de documentos digitalizados, **análise de documentos** (faturas, formulários), **descrição e análise visual**, interpretação de **diagramas e gráficos**, e **comparação visual** entre versões. Em todos, combinar a imagem com instruções textuais claras melhora muito o resultado.

### Best Practices

Escolha a resolução pelo trade-off qualidade/tokens, sempre combine a imagem com contexto textual, e pergunte-se se a tarefa precisa mesmo de imagem — às vezes o texto já está disponível e mandar a imagem é só desperdício.

## Referências

- Anthropic, *Prompt caching* — <https://docs.claude.com/en/docs/build-with-claude/prompt-caching>
- Anthropic, *Message Batches (batch processing)* — <https://docs.claude.com/en/docs/build-with-claude/batch-processing>
- Anthropic, *Pricing* (preços por modelo, batch e caching) — <https://docs.claude.com/en/docs/about-claude/pricing>
- Anthropic, *Context windows* — <https://docs.claude.com/en/docs/build-with-claude/context-windows>
- Anthropic, *Vision* (imagens como input) — <https://docs.claude.com/en/docs/build-with-claude/vision>

### Resumo do Capítulo 7

Antes de otimizar tokens, **planeje** — o token mais caro é o gasto no caminho errado. Reduza contexto cortando o irrelevante e mantendo o `CLAUDE.md` enxuto (ele é pago em toda chamada). Use **prompt caching** para qualquer prefixo grande e estável — o maior ganho financeiro, automático no Claude Code, desde que você não invalide o cache por descuido. Separe contexto de memória, e pode a memória (você paga pelos tokens que ela ocupa, não por armazená-la). No custo, esqueça "horário barato": as alavancas são **modelo certo por tarefa** (roteamento em cascata), **batch** (50%, offline) e **caching**. Monitore pelo Console ou por um dashboard próprio. E lembre da outra metade — **latência**: streaming, modelo menor, prompt curto e caching deixam a resposta mais rápida. Imagens são poderosas e caras em tokens; use-as só quando a tarefa pede.

---

## 🚀 Exercícios Práticos do Capítulo 7

1. **Plano antes da execução:** Pegue uma tarefa de código não trivial, entre em *plan mode* (`Shift+Tab`) e revise o plano antes de aprovar. Compare o retrabalho com o de uma execução direta, sem plano.
2. **Cachear e verificar:** Rode duas chamadas de API com o mesmo prefixo grande e inspecione `cache_read_input_tokens` na segunda. Depois insira um `timestamp` no início do prompt e confirme que o cache zera.
3. **Roteamento em cascata:** Monte um classificador que resolve os casos fáceis com Haiku e escala só os difíceis para Opus. Compare o custo total com o de usar Opus para tudo.
4. **Batch vs tempo real:** Pegue 50 prompts independentes e rode-os via Message Batches API. Compare custo e tempo com os das mesmas 50 chamadas em tempo real.
5. **Dieta do `CLAUDE.md`:** Meça o tamanho em tokens do seu `CLAUDE.md`, corte 30% movendo detalhe para skills ou subpastas, e confirme que o comportamento do agente se manteve.
