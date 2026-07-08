# Capítulo 5: Prática e Referência

Os capítulos anteriores construíram a teoria — o que é um LLM (Cap. 1), o que é Claude (Cap. 2), como operá-lo no terminal (Cap. 3) e como escrever prompts (Cap. 4). Este é o capítulo do **playbook**: como transformar tudo isso em resultado, com técnicas, dicas, exemplos concretos e uma referência rápida para colar e usar. Começamos pela pergunta que resume o capítulo — *como se obtém o melhor de um LLM?* — e só então descemos aos casos de uso.

## Antes dos casos de uso: em que nível você está trabalhando?

Um erro comum é tratar todo problema como "escrever um prompt melhor". Na prática, uma tarefa pode ser atacada em **três níveis de complexidade crescente**, e escolher o nível certo importa mais que qualquer truque de redação. A distinção segue o guia [*Building Effective Agents*](https://www.anthropic.com/engineering/building-effective-agents) da Anthropic:

- **Prompt único** — uma chamada, uma resposta. É o nível certo para tarefas fechadas e bem definidas: resumir um texto, classificar um ticket, traduzir um parágrafo. Barato, previsível, fácil de avaliar.
- **Workflow** — vários passos de LLM e ferramentas **orquestrados por código que você escreveu** (uma sequência fixa: extrair → validar → formatar). É o nível certo quando a tarefa tem etapas conhecidas e você quer **previsibilidade**. O controle é seu; o modelo preenche as lacunas.
- **Agente (harness)** — o modelo **dirige o próprio processo** num loop, decidindo quais ferramentas chamar e quando parar (Cap. 8). É o nível certo para trabalho aberto e exploratório, onde as etapas não são conhecidas de antemão. Ganha em flexibilidade, mas paga em custo e latência.

A regra de ouro do guia: **comece pelo mais simples e só suba de nível quando o ganho justificar**. Muita coisa que as pessoas resolvem com um agente caro seria melhor servida por um workflow previsível — ou por um único prompt bem escrito. Guardar esse mapa evita a armadilha de construir um agente autônomo para um problema que pedia três linhas de prompt.

## Tipos de trabalho com LLM

Antes de mergulhar em "code review" ou "tradução", vale reconhecer as **famílias de tarefa** — elas herdam a taxonomia de NLP do Capítulo 1 e determinam o quanto você pode confiar na saída sem verificação:

| Família | Exemplos | Confiabilidade e cuidado |
| --- | --- | --- |
| **Extração e classificação** | rotular tickets, extrair campos, detectar idioma | Alta — saída fechada, fácil de validar por schema |
| **Transformação** | traduzir, reescrever, reformatar, refatorar | Alta a média — há um "original" para comparar |
| **Síntese e análise** | resumir, comparar documentos, gerar relatórios | Média — risco de omitir ou distorcer; exige rubrica |
| **Geração** | escrever texto, código ou conteúdo novo | Média a baixa — o modelo inventa o que não sabe; verifique |
| **Conversação e agêntico** | chatbots, assistentes com ferramentas | Variável — combine com validação e escalonamento humano |

A leitura prática da tabela: quanto mais **fechada** a tarefa (extração, classificação), mais você pode automatizar sem medo; quanto mais **aberta** (geração, agêntico), mais a verificação deixa de ser opcional. Essa é a mesma lógica de "portões de avaliação" da pilha agêntica do Capítulo 8.

## Como atingir os melhores resultados com LLMs

Se houvesse uma única página para levar, seria esta. São nove princípios que, na prática, respondem pela maior parte da diferença entre um resultado medíocre e um excelente — cada um aprofundado no capítulo indicado:

1. **Escolha o nível e o modelo certos.** Prompt, workflow ou agente (acima); Haiku, Sonnet, Opus ou Fable por tarefa (Cap. 2). A decisão mais barata é a de não usar o modelo mais caro para um trabalho simples.
2. **Dê contexto, não suposições.** O que o modelo não tem no contexto, ele inventa. Forneça o código, o documento, os exemplos, as regras — não confie na "memória" dele (Cap. 1).
3. **Estruture a entrada e declare o formato de saída.** Papel/tarefa/contexto/formato, com tags XML ou Markdown (Cap. 4). Metade das "respostas ruins" são respostas certas no formato errado.
4. **Mostre exemplos.** Um ou dois exemplos do formato desejado (*few-shot*) ensinam mais que parágrafos de descrição.
5. **Decomponha o que é grande.** Uma tarefa por prompt; quebre trabalho longo em etapas (workflow) ou subagentes (Cap. 8). Contexto inchado degrada a qualidade ("perdido no meio", Cap. 1).
6. **Verifique — nunca assuma 100%.** Valide a saída programaticamente, rode testes, use um passo cético de revisão. Confiança sem verificação é como o modelo entrega erro com a mesma naturalidade do acerto.
7. **Itere com medida.** Mude uma coisa por vez e compare contra um pequeno conjunto de casos de avaliação (Cap. 4). Ajustar no escuro troca acerto por regressão sem você perceber.
8. **Gerencie o contexto por relevância.** Traga o que importa, não "tudo por garantia". Cacheie o estável, reancore fatos duráveis (Cap. 7 e 8).
9. **Prefira o simples.** Só adicione complexidade (mais passos, mais ferramentas, mais autonomia) quando ela entregar valor mensurável. Simplicidade é uma decisão de arquitetura, não preguiça.

O resto deste capítulo aplica esses princípios a casos concretos.

## Casos de Uso Práticos

### Coding

Programar é hoje o maior caso de uso de LLMs, e vale situá-lo nos três níveis da abertura: uma **revisão de trecho** é um prompt único; um **pipeline de CI que revisa PRs** é um workflow; **corrigir um bug que atravessa dez arquivos** é trabalho de agente (Claude Code, Cap. 3 e 8). O que segue são padrões no nível de prompt, que também formam a base dos outros dois.

Para **code review**, dê o código, diga o que te preocupa e peça achados ordenados por severidade ("revise este trecho focando em segurança e concorrência; liste os problemas do mais grave ao menos grave"). Para **bug fixing**, descreva o sintoma, o comportamento esperado e o contexto — o que você já tentou ajuda muito. Para **explicação de código**, peça o nível certo ("explique para alguém que conhece programação mas não conhece este framework"). E para **refactoring**, seja explícito sobre o objetivo (legibilidade? performance? testabilidade?), porque "refatore" sem objetivo gera mudanças arbitrárias.

> [!tip] Modo de ensino no Claude Code
> Quando o objetivo é **aprender**, e não só entregar, o Claude Code tem *output styles* dedicados: `/output-style explanatory` faz o agente narrar o porquê de cada escolha de implementação, e `/output-style learning` é colaborativo — ele deixa marcadores `TODO(human)` para você escrever pequenos trechos e praticar. É a diferença entre receber o peixe e aprender a pescar.

### Análise e Research

Para processamento de dados e síntese de documentos, o segredo é dar estrutura: peça o formato de saída, defina os critérios de comparação, e — quando o volume for grande — divida em pedaços. Para análise comparativa, forneça uma **rubrica explícita** do que comparar, ou você receberá uma comparação genérica. Ex.: "compare estas duas propostas nos eixos custo, prazo e risco técnico; para cada eixo, dê uma nota de 1 a 5 e uma frase de justificativa; termine com uma recomendação."

### Escrita e Conteúdo

Em copywriting e revisão editorial, especifique **público, tom e tamanho**. Em tradução, dê o contexto (formal ou informal? técnico ou coloquial?). Em paráfrase e resumo, diga o comprimento alvo e o público. O padrão é sempre o mesmo: o que o modelo não sabe, ele inventa — então diga. Um exemplo de prompt que não deixa margem:

```xml
<papel>Você é um editor de conteúdo técnico.</papel>
<tarefa>Reescreva o texto abaixo para um público de gestores não técnicos.</tarefa>
<restricoes>
- Tom profissional, mas acessível. Máximo 150 palavras.
- Sem jargão; se um termo técnico for inevitável, explique-o em uma frase.
- Preserve todos os números e prazos do original.
</restricoes>
<texto>[cole o texto aqui]</texto>
```

### Educação e Ensino

O primeiro passo é **definir o público-alvo** — e, quando a tarefa se repete, construir **personas** ("aluno do 1º ano de engenharia, sabe cálculo, nunca viu programação"). Uma persona bem descrita calibra vocabulário, profundidade e exemplos de uma vez só. Para gerar exercícios, especifique nível e formato; para feedback automático, dê a rubrica.

Dois princípios elevam o resultado pedagógico. Primeiro, **explique a origem e o porquê das coisas**, não só o "como": entender *por que* um algoritmo existe fixa mais do que memorizar seus passos. Peça isso explicitamente ("explique o problema que motivou esta solução antes de descrevê-la"). Segundo, use os modos de ensino quando estiver programando junto — o `learning` do Claude Code (acima) transforma a sessão numa aula prática.

### Atendimento ao Cliente

> [!warning] Um tema grande por si só
> Atendimento automatizado (chatbots, triagem de tickets, FAQ) é um domínio complexo, com implicações de marca, jurídicas e de confiança. O que segue são princípios de partida, não um guia completo — um sistema de produção sério merece projeto dedicado, avaliação contínua e revisão humana.

O princípio central é o mesmo de qualquer aplicação séria: combine a capacidade do modelo com **validação** e, em casos sensíveis, **escalonamento para um humano** (Cap. 8). Exemplos concretos:

- **Como fazer:** definir explicitamente o escopo do bot ("respondo sobre pedidos e entregas; para pagamentos, transfiro a um atendente"); tratar entrada do usuário como não confiável; ter um gatilho de confiança que passa para humano quando a certeza cai ou o assunto é delicado (reembolso, reclamação, dado sensível); registrar tudo para auditoria.
- **Como não fazer:** deixar o bot responder sobre qualquer assunto ("alucinação com cara de suporte oficial"); nunca oferecer saída para um humano; prometer ações que ele não pode executar; inventar políticas da empresa que não estão na base de conhecimento. Um chatbot que nunca passa para um humano é um chatbot que eventualmente vai irritar alguém — ou pior, dar uma informação errada com autoridade.

## Boas Práticas e Armadilhas

### O que Fazer

Sempre valide outputs. Use exemplos específicos. Itere nos prompts em vez de esperar acertar de primeira. Documente as decisões — por que este prompt, por que este modelo, por que este formato.

### O que Evitar

Prompts vagos. Sobrecarga de contexto ("jogar tudo por garantia"). Assumir confiabilidade de 100%. E ignorar custo e performance até a fatura chegar.

### Documentação e Versionamento de Prompts, na prática

"Trate prompts como código" é fácil de dizer; na prática significa um punhado de hábitos concretos:

1. **Prompts em arquivos, no Git.** Tire o prompt de dentro do código-fonte e coloque-o num arquivo versionado (`prompts/classificar_ticket.md`). Assim cada mudança gera um *diff* legível e um histórico de quem mudou o quê.
2. **Versão e changelog.** Marque versões (`v1`, `v2`…) e mantenha um `CHANGELOG` de uma linha por mudança ("v3: adicionada instrução de não inventar números"). Você vai querer saber qual versão estava no ar quando um problema apareceu.
3. **Conjunto de avaliação como teste de regressão.** Guarde 5–20 casos com a resposta esperada e rode-os a cada alteração de prompt, como um teste automatizado. É o que impede uma "melhoria" de quebrar silenciosamente outro caso.
4. **Rollback.** Como o prompt está no Git com versões, voltar à anterior é um `git revert`. Um prompt em produção sem esse caminho de volta é um acidente esperando para acontecer.

### Ferramentas para gerenciar prompts

Além do Git, existem plataformas dedicadas a versionar, testar e observar prompts em produção: o próprio **Anthropic Console** (workbench com versionamento e avaliação), o **LangSmith** (LangChain), o **Langfuse** (open-source), o **PromptLayer** e o **Helicone** (observabilidade e logging de chamadas). Para times que rodam LLM em escala, elas centralizam o que, no começo, dá para fazer com arquivos e um conjunto de testes.

## Troubleshooting

A maioria dos problemas de prompt se resolve com mais estrutura e formato explícito. O sintoma que aparece sobretudo aqui, na iteração de prompts, é o **output repetitivo**: quando o modelo se repete, a causa costuma ser um prompt que induz o padrão ou a falta de variação no contexto — detecte comparando saídas e resolva variando o prompt (nos modelos recentes, o raciocínio adaptativo cuida da amostragem que antes se ajustava à mão).

Os demais sintomas de produção — não seguir instruções, verbosidade ou concisão excessiva, alucinações, erros de API, custo e latência — têm um guia de diagnóstico completo, organizado por sintoma, no **Capítulo 9**. Para não duplicar, remetemos você a ele em vez de repetir aqui.

## Referência Rápida

### Snippets de Prompts

Tenha templates prontos para suas tarefas recorrentes — análise, código, conteúdo, atendimento. Cada template encapsula papel, tarefa, contexto e formato, e poupa você de reescrever a estrutura toda vez. O [cookbook oficial da Anthropic](https://github.com/anthropics/anthropic-cookbook) é uma boa fonte de receitas testadas.

### Parâmetros Comuns

Nos modelos recentes, a "temperatura" deu lugar ao nível de **esforço** (`low`/`medium`/`high`/`xhigh`/`max`): use `low`/`medium` para tarefas rotineiras e `high`/`xhigh` para coding e trabalho agêntico. Para **max tokens**, não seja mesquinho — estourar o limite trunca a resposta no meio; um bom padrão é ~16.000 para requisições normais e ~64.000 quando estiver usando streaming. Saídas acima de ~16K exigem streaming para não esbarrar em timeout.

### Recursos Externos

A documentação oficial da Anthropic é a fonte de verdade para IDs de modelo, preços e recursos novos — consulte-a sempre que houver dúvida, porque o cenário muda rápido. Há comunidades ativas (Discord, Reddit), papers de pesquisa e ferramentas de terceiros como o Graphify do Capítulo 6.

## Roadmap e Futuro

O raciocínio estendido (extended thinking / adaptive thinking) já é realidade nos modelos atuais e tende a se aprofundar. Modelos posteriores ao corte de conhecimento — como o **Fable 5** e o **Mythos 5** — já são parte do presente, não do futuro, e a cadência de lançamentos da Anthropic é alta. Capacidades emergentes em pesquisa apontam para agentes mais autônomos, janelas de contexto maiores e melhor uso de memória. A constante é a mudança: a regra de "use o modelo mais recente da sua faixa" vale justamente porque a fronteira se move.

## Referências

- Anthropic, *Building Effective Agents* (prompt vs workflow vs agente) — <https://www.anthropic.com/engineering/building-effective-agents>
- Anthropic, *Prompt engineering overview* — <https://docs.claude.com/en/docs/build-with-claude/prompt-engineering/overview>
- Anthropic, *Claude Code — Output styles* (`explanatory`, `learning`) — <https://code.claude.com/docs/en/output-styles>
- Anthropic, *Anthropic Cookbook* (receitas e exemplos) — <https://github.com/anthropics/anthropic-cookbook>

### Resumo do Capítulo 5

Antes de qualquer receita, decida o **nível**: prompt único, workflow ou agente — e comece pelo mais simples. Reconheça a **família da tarefa** (extração e classificação são confiáveis; geração e agêntico exigem verificação). Os nove princípios de "melhores resultados" resumem o livro em uma página: nível e modelo certos, contexto em vez de suposição, estrutura e formato, exemplos, decomposição, verificação, iteração medida, contexto por relevância, e preferência pelo simples. Aplique-os aos casos de uso (coding, análise, escrita, educação, atendimento) com exemplos e escopo explícito, versione prompts como código (arquivos no Git, changelog, conjunto de avaliação, rollback), e mantenha a documentação oficial por perto — porque modelos e preços mudam.

---

## 🚀 Exercícios Práticos do Capítulo 5

1. **Escolha o nível:** Pegue três tarefas suas e classifique cada uma como prompt único, workflow ou agente. Justifique — e para a que você chamaria de "agente", pergunte-se honestamente se um workflow não resolveria mais barato.
2. **Os nove princípios:** Escolha uma tarefa real e escreva o prompt aplicando conscientemente pelo menos cinco dos nove princípios. Compare com sua primeira tentativa "de instinto".
3. **Persona pedagógica:** Construa uma persona de aluno detalhada e peça ao Claude uma explicação de um conceito para ela. Depois mude um traço da persona (nível, área) e veja como a explicação deveria mudar.
4. **Do/don't de atendimento:** Escreva o prompt de sistema de um chatbot de suporte que define escopo, trata entrada como não confiável e escala para humano. Depois liste três coisas que ele nunca deve fazer.
5. **Versionamento de prompt:** Coloque um prompt num arquivo, crie um conjunto de 5 casos de avaliação, faça uma alteração e rode os casos antes e depois. Registre o resultado num changelog de uma linha.
