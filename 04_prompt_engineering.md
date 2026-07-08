# Capítulo 4: Prompt Engineering

Este capítulo dedica-se à interface de interação: a Engenharia de Prompts (*prompt engineering*). Na cultura popular ela costuma ser tratada de forma **empírica e assistemática** — e por um bom motivo: a maioria das pessoas aprende por tentativa e erro, colecionando "palavras mágicas" e receitas de fórum, sem medir nada e sem entender por que uma formulação funcionou e outra não. O resultado é um folclore de dicas que às vezes ajudam, às vezes não, e ninguém sabe dizer quando.

A proposta deste capítulo é o oposto disso: tratar a formulação de prompts como uma **prática de engenharia** — com estrutura, avaliação por métricas e iteração controlada. Mas é preciso honestidade sobre o que isso significa e o que **não** significa.

> [!warning] Uma ressalva honesta antes de começar
> Seria exagero chamar prompt engineering de disciplina "determinística". O mesmo prompt produz saídas diferentes conforme o **modelo**, os **parâmetros**, o **contexto** e a própria aleatoriedade da amostragem. O que existe de sólido é uma **base empírica** — o campo do *in-context learning*, aberto pelo artigo do GPT-3 (Brown et al., 2020), mostrou que modelos grandes aprendem a tarefa a partir de exemplos no próprio prompt — e um **método** de trabalho (estruturar, medir, iterar). Não uma garantia de reprodutibilidade exata.
>
> Vale também conhecer a crítica. Com os **modelos de raciocínio** atuais (que "pensam" internamente antes de responder), boa parte da engenharia de prompt elaborada perdeu importância: truques que eram decisivos em 2023 hoje rendem pouco, porque o modelo já faz internamente o que antes você induzia no texto. A meta-análise de Sprague et al. (2024) é um bom exemplo de ceticismo fundamentado — mostra que o *chain-of-thought*, a técnica mais celebrada, só traz ganho robusto em tarefas de **matemática e lógica simbólica**, e quase nada no resto. Ou seja: prompt engineering ajuda, mas o retorno depende da tarefa e do modelo, e a régua certa é sempre medir, não acreditar.

## Fundamentos de Prompt Engineering

### Estrutura Básica de Prompts

O **prompt** é a sequência estruturada de texto de entrada submetida ao modelo. Ele condiciona a distribuição de probabilidades do LLM, restringindo o universo de saídas prováveis para que a geração se alinhe à tarefa desejada. Não se trata tanto de "conversar" com a máquina, mas de **parametrizar** um motor de inferência com contexto.

A maioria dos prompts ruins falha por falta de estrutura, não por falta de capacidade do modelo. Um bom prompt costuma ter quatro componentes, mesmo que implícitos:

- **Role/persona**: quem Claude deve ser. "Você é um revisor de código sênior especializado em segurança" calibra o tom e a profundidade.
- **Task/objetivo**: o que fazer, com um verbo claro. "Revise", "resuma", "extraia", "traduza".
- **Context**: as informações relevantes — o código, o documento, os exemplos.
- **Output format**: como responder. JSON? Lista com marcadores? Um parágrafo?

A sistematização da entrada reduz a variância dos resultados. Estruturas com **tags XML** ou **Markdown** ajudam o modelo a isolar instruções, contexto e restrições. Abaixo, um template estruturado:

```xml
<instrucao_sistema>
Atue como Analista de Segurança de Software Sênior.
Seu objetivo é analisar o código fornecido e identificar estritamente vulnerabilidades de injeção de SQL ou XSS.
</instrucao_sistema>

<restricoes>
- Não inclua preâmbulos educacionais.
- Responda no formato JSON contendo as chaves: "vulnerabilidade", "linha" e "severidade".
- Caso não haja vulnerabilidade, retorne um JSON vazio {}.
</restricoes>

<codigo_alvo>
[Insira o código aqui]
</codigo_alvo>
```

A diferença entre "me ajuda com esse código" e um prompt com esses quatro elementos é, muitas vezes, a diferença entre uma resposta genérica e uma resposta utilizável.

> [!example] O custo de um escopo mal definido (caso real)
> Uma vez quase perdi o trabalho inteiro de uma pequena aplicação porque pedi ao modelo, sem restrições, para "melhorar o *look and feel*". Ele interpretou isso com liberdade total e **reescreveu a aplicação como uma ferramenta de design de páginas** sobre o programa original — descaracterizando o que existia. A lição não é "o modelo é burro"; é que um objetivo vago (*"melhore"*) sem **escopo e restrições explícitas** ("altere apenas o CSS em `styles/`, não toque na lógica, não crie arquivos novos") autoriza o agente a fazer exatamente o que você não queria. Delimitar o escopo é a metade do prompt que as pessoas esquecem.

### Técnicas Fundamentais

Na prática — e é o que a própria [documentação de prompt engineering da Anthropic](https://docs.claude.com/en/docs/build-with-claude/prompt-engineering/overview) prioriza —, três abordagens dão o maior retorno pelo menor esforço:

1. **Clareza e especificidade**: diga exatamente o que quer. "Resuma em três frases para um executivo não técnico" é melhor que "resuma".
2. **Exemplos** (*few-shot*): mostrar um ou dois exemplos do formato desejado ensina o modelo melhor do que parágrafos de descrição.
3. **Instruções negativas**: dizer o que _não_ fazer ("não inclua preâmbulo", "não invente números").

Combinadas com a **separação de responsabilidades** (uma tarefa por prompt, quando possível), essas abordagens resolvem a maioria dos problemas do dia a dia.

Uma objeção justa: **"dizer algo com clareza" é mesmo uma _técnica_?** Não no sentido de um truque específico de LLM. É melhor descrita como uma **disciplina** — a mesma que se usa para redigir a especificação de um projeto, um ticket de bug ou um contrato. O prompt não inventou a necessidade de ser claro; só a tornou implacável, porque o modelo executa exatamente o que foi escrito, sem preencher lacunas com bom senso como faria um colega. Por isso a habilidade transfere: quem escreve boas *specs* escreve bons prompts.

E como **avaliar se algo está claro**, sem cair no subjetivo? Três testes operacionais:

- **Teste da interpretação única**: existe mais de uma leitura razoável do que você pediu? Se sim, não está claro. "Melhore o desempenho" admite dez leituras; "reduza o tempo de resposta do endpoint `/busca` para menos de 200 ms" admite uma.
- **Teste do estranho competente**: uma pessoa qualificada, mas sem o contexto que está na sua cabeça, executaria a tarefa do jeito que você espera? Se ela precisaria adivinhar, o modelo também precisa.
- **Teste do critério verificável**: o "pronto" é checável objetivamente? "Faça ficar bom" não é; "retorne JSON com as chaves X, Y, Z e nada mais" é. Se você não consegue escrever o teste que decide se a saída passou, o pedido ainda está vago.

### Boas Práticas Imediatas

Para já: seja direto, evite ambiguidade, use exemplos quando a tarefa for complexa, e sempre especifique o formato do output. Esse último ponto é subestimado — metade das "respostas ruins" são, na verdade, respostas certas no formato errado.

## Técnicas Avançadas de Prompting

Algumas técnicas foram desenvolvidas, refinadas e são amplamente utilizadas; vale mantê-las à mão — sabendo, a cada uma, **quando** ela de fato ajuda.

### Chain-of-Thought

Para problemas que exigem raciocínio em vários passos, pedir ao modelo para "pensar em voz alta" antes de responder pode melhorar a qualidade. Isso é o **chain-of-thought** (CoT), popularizado por Wei et al. (2022). Pode ser explícito ("explique seu raciocínio passo a passo antes da resposta final") ou implícito.

Mas a pergunta "será que melhora mesmo?" é legítima, e a resposta é **depende da tarefa**. A meta-análise de Sprague et al. (2024), cobrindo mais de 100 estudos e 20 datasets, encontrou que o CoT traz ganho robusto **principalmente em matemática e raciocínio simbólico** — em tarefas de conhecimento ou linguagem, o ganho é pequeno ou nulo. Some-se a isso um ponto prático importante: nos **modelos de raciocínio** atuais (Fable 5, Opus 4.8), o CoT explícito virou em boa parte redundante, porque o modelo já raciocina internamente (veja *Extended Thinking* adiante). O custo também é real — mais tokens de raciocínio significam mais dinheiro e latência. A regra: use CoT quando o problema for genuinamente de **cálculo ou lógica**, não como ritual para toda pergunta.

### In-Context Learning

O modelo aprende com os exemplos que você coloca no próprio prompt — **in-context learning**. A versão sofisticada é o **dynamic few-shot**: escolher, em tempo de execução, os exemplos mais relevantes para a pergunta atual, em vez de exemplos fixos. Há retornos decrescentes: depois de alguns exemplos bem escolhidos, adicionar mais custa tokens sem melhorar muito. **Qualidade de exemplo importa mais que quantidade.**

### Gerar-e-Escolher e Auto-Refinamento

Duas táticas iterativas rendem muito em trabalho aberto e valem além do prompt único:

- **Gerar alternativas e escolher a melhor.** Em cada etapa de decisão, peça ao modelo **3 alternativas**, avalie-as segundo um critério explícito e siga com a melhor. É a intuição por trás de técnicas como *self-consistency* (Wang et al., 2022 — amostrar vários caminhos e votar) e *Tree of Thoughts* (Yao et al., 2023 — ramificar e podar). Reduz o risco de o modelo "casar" com a primeira ideia.
- **Refinar após cada etapa.** Depois de produzir um resultado, peça uma **etapa de melhoria** ("critique o que você acabou de fazer e proponha uma versão melhor"). É o padrão *Self-Refine* (Madaan et al., 2023): o modelo gera, avalia o próprio trabalho e revisa. Funciona bem quando há um critério claro de qualidade — e mal quando não há, porque aí ele só reescreve sem convergir.

### Prompt Compression

Quando o prompt cresce, vale comprimi-lo sem perder o essencial — remover redundância, trocar prosa por estrutura, cortar contexto irrelevante. O trade-off é entre economia de tokens e risco de cortar algo que importava. A regra: **comprima a forma, preserve a substância**.

### Structured Prompting e o formato TOON

A forma como você estrutura o prompt afeta o resultado. **Tags XML** (`<contexto>...</contexto>`) ajudam o modelo a separar partes e são especialmente eficazes com Claude. **JSON** é bom para saída estruturada e parseável. **Markdown** organiza instruções para leitura. Cada formato tem seu lugar: XML para delimitar seções, JSON para contratos de dados, Markdown para instruções legíveis.

Quando o problema é **enviar muitos dados tabulares** ao modelo, surgiu recentemente (final de 2025) um formato desenhado para isso: o **TOON — Token-Oriented Object Notation**. Ele é uma codificação **lossless** do mesmo modelo de dados do JSON, mas otimizada para gastar menos tokens. A ideia: em vez de repetir as chaves em cada objeto (como o JSON faz), o TOON combina a indentação do YAML com um layout **tabular** ao estilo CSV — declara os campos uma única vez num cabeçalho e lista as linhas embaixo, sem chaves, colchetes ou aspas repetidas. Cabeçalhos explícitos de tamanho `[N]` e de campos `{...}` dão ao modelo um esquema claro para seguir.

```toon
usuarios[2]{id,nome,plano}:
  1,Ana,pro
  2,Bruno,free
```

O mesmo dado em JSON repetiria `"id"`, `"nome"` e `"plano"` em cada objeto. Em **arrays uniformes de objetos** — o ponto forte do TOON — a economia relatada fica na faixa de **30–60% menos tokens** que o JSON formatado, muitas vezes sem perda de acurácia. A ressalva: para dados **profundamente aninhados ou irregulares**, a vantagem encolhe e o JSON pode ser mais claro. Use TOON para tabelas grandes; mantenha JSON para estruturas heterogêneas.

### Extended Thinking

Os modelos Claude recentes têm um modo de **raciocínio interno** antes de responder. Aqui há uma mudança importante em relação ao que muitos materiais antigos descrevem. O modelo de "orçamento fixo de pensamento" (`budget_tokens`) foi **substituído pelo raciocínio adaptativo**: você ativa `thinking: {type: "adaptive"}` e o próprio modelo decide quando e quanto pensar. A profundidade é controlada por um parâmetro de **esforço** (*effort*), com níveis `low`, `medium`, `high`, `xhigh` e `max`. Nos modelos de ponta (Fable 5, Opus 4.8, 4.7), o esforço importa mais do que em qualquer geração anterior — `high` ou `xhigh` é o ponto doce para a maioria do trabalho de coding e agêntico, e `low`/`medium` servem para tarefas rotineiras. O impacto em custo é direto: mais esforço, mais tokens de raciocínio. É também por causa desse raciocínio nativo que o *chain-of-thought* manual perdeu parte da relevância.

## Debugging e Iteração de Prompts

Um prompt não nasce pronto — ele é depurado como código. E há dois momentos de intervenção: **antes** de executar o trabalho (melhorar o prompt) e **depois** (avaliar o resultado e comparar versões).

### Melhorar o prompt antes de rodar

Antes de gastar tokens numa tarefa cara, vale refinar o prompt. Duas formas simples e eficazes:

- **Meta-prompting**: peça ao próprio modelo para criticar e reescrever o seu prompt ("aqui está meu prompt e meu objetivo; aponte ambiguidades e devolva uma versão mais clara e específica"). O modelo é bom em enxergar o que ficou vago.
- **Ferramentas dedicadas**: o Anthropic Console traz um *prompt improver* e um gerador de prompts que aplicam boas práticas automaticamente a partir de uma descrição da tarefa.

### Avaliar prompts com medida

"Esse prompt está bom?" é uma pergunta que só se responde medindo. Defina **métricas de qualidade** para a sua tarefa — acurácia de extração, aderência ao formato, ausência de alucinação — monte um **conjunto pequeno de casos representativos** (com respostas esperadas) e rode os candidatos contra ele. Sem esse conjunto de avaliação, você está ajustando no escuro e trocando um acerto por uma regressão sem perceber.

### Falhas comuns e como reconhecê-las

Os modos de falha têm assinatura reconhecível, e cada um aponta para uma correção:

| Sintoma | Causa provável | Correção |
| --- | --- | --- |
| **Alucinação** (fatos/números/APIs inventados) | O modelo preenche lacunas em vez de admitir que não sabe | Peça fontes, permita "não sei", dê contexto verificável |
| **Output genérico** | Prompt vago demais | Mais especificidade, exemplos, persona |
| **Não seguir instruções** | Instruções conflitantes ou enterradas num prompt longo | Simplifique, mova o essencial para o fim, uma tarefa por vez |
| **Verboso ou curto demais** | Formato e tamanho não especificados | Diga o formato e o comprimento desejados |

### Iteração efetiva e instrumentação

Itere com **mudanças incrementais** — altere uma coisa por vez, ou você não saberá o que causou a diferença — e use **teste A/B** quando puder comparar duas versões no mesmo conjunto de casos. Mantenha **registro** das versões e resultados: um prompt que melhora hoje e piora amanhã sem histórico é um mistério insolúvel.

Em produção, **trate prompts como código**: versionados no Git, testados contra o conjunto de avaliação e monitorados (desempenho, custo por requisição, taxa de falha). É essa disciplina — e não uma coleção de "palavras mágicas" — que separa um protótipo de um sistema confiável.

## Referências

- Brown et al., *Language Models are Few-Shot Learners* (GPT-3, in-context learning, 2020) — <https://arxiv.org/abs/2005.14165>
- Wei et al., *Chain-of-Thought Prompting Elicits Reasoning in Large Language Models* (2022) — <https://arxiv.org/abs/2201.11903>
- Sprague et al., *To CoT or not to CoT? Chain-of-thought helps mainly on math and symbolic reasoning* (2024) — <https://arxiv.org/abs/2409.12183>
- Wang et al., *Self-Consistency Improves Chain of Thought Reasoning in Language Models* (2022) — <https://arxiv.org/abs/2203.11171>
- Yao et al., *Tree of Thoughts: Deliberate Problem Solving with Large Language Models* (2023) — <https://arxiv.org/abs/2305.10601>
- Madaan et al., *Self-Refine: Iterative Refinement with Self-Feedback* (2023) — <https://arxiv.org/abs/2303.17651>
- Anthropic, *Prompt engineering overview* — <https://docs.claude.com/en/docs/build-with-claude/prompt-engineering/overview>
- TOON — *Token-Oriented Object Notation* (spec, benchmarks e SDKs) — <https://toonformat.dev/> · <https://github.com/toon-format/toon>

---

## 🚀 Exercícios Práticos do Capítulo 4

1. **Escopo que segura a mão do agente:** Pegue uma tarefa de edição e escreva duas versões do prompt — uma vaga ("melhore isto") e uma com escopo e restrições explícitas (o que tocar, o que _não_ tocar, o formato). Compare o quanto o resultado fica sob controle.
2. **Teste da clareza:** Pegue um prompt seu e aplique os três testes (interpretação única, estranho competente, critério verificável). Reescreva-o até passar nos três e veja o efeito na resposta.
3. **Chain-of-Thought, com ceticismo:** Rode um problema de lógica (o enigma de Einstein) com e sem "pense passo a passo". Depois rode uma pergunta de conhecimento geral do mesmo jeito. Confirme na prática a conclusão de Sprague et al.: onde o CoT ajuda e onde não.
4. **JSON vs TOON:** Pegue um array de ~20 objetos uniformes, represente-o em JSON e em TOON, conte os tokens de cada um e verifique a economia. Peça ao modelo para responder uma pergunta sobre os dados nos dois formatos e compare a acurácia.
5. **Gerar-e-escolher:** Para uma decisão de design, peça 3 alternativas com prós e contras, depois peça ao modelo para escolher a melhor segundo um critério que você definir. Compare com a resposta de tiro único.
