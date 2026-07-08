# Capítulo 9: Integração e Resolução de Problemas

Os capítulos anteriores trataram cada peça isoladamente. Este capítulo faz o que falta: mostra como as peças se encaixam num sistema real, cataloga os problemas que mais aparecem, oferece um guia de diagnóstico para quando as coisas dão errado — porque vão dar —, apresenta os padrões de controle que mantêm o sistema sob rédea e o modelo de ameaças de segurança específico de LLM. Pense nele como o manual de manutenção que vem depois do manual de uso.

## Os problemas mais comuns e suas causas

Antes do diagnóstico caso a caso, vale um mapa: a esmagadora maioria dos problemas em produção cai em seis famílias, e cada uma tem uma **causa-raiz** típica. Reconhecer a família já aponta para onde olhar.

| Família de problema | Sintoma típico | Causa-raiz mais comum |
| --- | --- | --- |
| **Configuração / API** | erros 4xx, comportamento inesperado após deploy | ID de modelo errado, parâmetro legado, papéis de mensagem fora de ordem |
| **Contexto** | o agente "esquece" algo dito antes; respostas incoerentes | janela saturada, "perdido no meio", cache invalidado |
| **Custo** | fatura sobe sem explicação | modelo caro onde bastava um barato, contexto inflado, caching quebrado |
| **Qualidade** | alucinação, output genérico, não segue instruções | prompt vago, sem verificação, sem acesso a informação verificável |
| **Ferramentas / integração** | ferramenta "falha em silêncio", loop de retry | erro estruturado não checado, credencial/rede, schema ambíguo |
| **Segurança** | vazamento, ação indevida, prompt injection | conteúdo externo tratado como confiável, excesso de permissão do agente |

As três primeiras famílias são tratadas no **Guia de Resolução de Problemas** adiante; a de qualidade recebe também o Capítulo 4; a de ferramentas conecta ao Capítulo 6; e a de segurança ganha seção própria neste capítulo (**modelagem de ameaças**) e o tratamento legal e ético no Capítulo 10. A regra que atravessa todas: **o problema quase nunca está no modelo — está na camada que você construiu em volta dele** (a pilha de seis camadas do Capítulo 8 é um bom checklist de "quem é o dono do que quebrou").

## Juntando as Peças: uma Arquitetura de Referência

Imagine uma aplicação realista: um assistente que ajuda uma equipe de engenharia a entender e modificar uma base de código grande. Ela usa quase tudo desta apostila, e ver como as peças se conectam vale mais que descrevê-las de novo.

No centro está a **API de Messages** (Capítulo 6), chamada com um **modelo escolhido pela tarefa** (Capítulo 2) — Haiku para classificar tickets baratos, Opus 4.8 ou Fable 5 para o trabalho agêntico difícil. Os **prompts** seguem a estrutura papel/tarefa/contexto/formato (Capítulo 4). O **system prompt grande e estável fica em cache** (Capítulo 7), com o conteúdo volátil — a pergunta do usuário, timestamps — depois do ponto de quebra, para não invalidar nada. Para entender o repositório sem despejá-lo no contexto, a aplicação usa o **Graphify** (Capítulo 6): consulta o grafo de conhecimento em vez de reler arquivos, cortando o custo por consulta em dezenas de vezes. Ferramentas externas — GitHub, um banco de dados — entram via **MCP** (Capítulo 6). E uma camada de **segurança e validação** (Capítulo 10) trata o conteúdo externo como não confiável e verifica todo output antes de agir sobre ele.

O fluxo de uma requisição típica:

1. O usuário pede algo ("por que a autenticação está lenta?").
2. A aplicação monta o prompt: system prompt cacheado + contexto relevante (puxado do grafo do Graphify, não do código inteiro) + a pergunta.
3. Claude raciocina (adaptive thinking, esforço `high`) e decide usar ferramentas — consultar o grafo, ler um arquivo específico, rodar um teste no sandbox.
4. Os resultados das ferramentas voltam; Claude itera até concluir.
5. A aplicação valida o output (o diff proposto compila? os testes passam?) antes de apresentá-lo.

Cada decisão dessa cadeia foi tomada por um motivo de custo, qualidade ou segurança discutido nos capítulos anteriores. É isso que significa "integrar": não usar todas as ferramentas, mas escolher conscientemente quais e por quê.

## Princípios de Integração

Alguns princípios atravessam qualquer integração e vale destilá-los:

- **Escolha o modelo por etapa, não por aplicação.** Um sistema bem feito usa modelos diferentes em pontos diferentes — um barato para triagem, um caro para o trabalho pesado. Trocar de modelo no meio de uma conversa, porém, invalida o cache de prompt; quando precisar de um modelo mais barato para uma subtarefa, delegue a um subagente separado em vez de trocar o modelo do loop principal.
- **Trate o cache como parte da arquitetura.** Ordene o prompt do mais estável ao mais volátil. Ferramentas e system prompt primeiro, conteúdo da requisição por último. Verifique `cache_read_input_tokens` para confirmar que está funcionando.
- **Valide nas fronteiras.** Entrada do usuário e respostas de APIs externas são as fronteiras; valide ali. Confie no interior do sistema.
- **Falha é regra, não exceção.** Timeouts, retries com backoff, fallbacks. Os SDKs já fazem retry de 429 e 5xx; você cuida do resto.
- **Mantenha segredos fora do alcance do modelo.** Chaves de API e tokens nunca vão dentro do prompt nem do sandbox; são injetados por uma camada externa (vaults, proxies). Conteúdo que entra no histórico fica persistido e legível.

## Guia de Resolução de Problemas

Esta é a parte que você vai reler quando algo quebrar. Está organizada por sintoma, no espírito de "se X acontece, verifique Y".

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

Um detalhe que pega muita gente nos modelos recentes (Fable 5, Opus 4.8/4.7): os parâmetros `temperature`, `top_p`, `top_k` foram **removidos** e devolvem **400** se enviados. O mesmo vale para `thinking: {type: "enabled", budget_tokens: N}` — use `thinking: {type: "adaptive"}`. E prefills na última mensagem do assistente também devolvem 400 nesses modelos. Se um código que funcionava começou a dar 400 sem motivo óbvio depois de uma troca de modelo, é quase sempre um desses parâmetros legados.

### O cache não está funcionando

Sintoma: `cache_read_input_tokens` sempre em zero apesar de prefixos "iguais". Causa quase certa: um **invalidador silencioso** no prefixo. Procure por `datetime.now()`, `Date.now()` ou um UUID no começo do prompt; por JSON serializado sem ordenar as chaves; por uma lista de ferramentas que muda entre requisições; ou por um ID de sessão interpolado no system prompt. A correção é mover o elemento dinâmico para depois do último ponto de quebra, torná-lo determinístico, ou removê-lo. Lembre também do mínimo cacheável (alguns milhares de tokens, dependendo do modelo): prefixos curtos demais simplesmente não cacheiam, sem erro.

### O modelo não segue instruções

Reveja três coisas. Primeiro, **conflito**: há instruções que se contradizem? Segundo, **posição**: a instrução importante está enterrada no meio de um prompt longo? Mova-a para o começo ou o fim. Terceiro, **intensidade**: nos modelos recentes, linguagem agressiva do tipo "CRÍTICO: VOCÊ DEVE SEMPRE" tende a fazer o modelo sobre-reagir e acionar ferramentas demais — suavize para "use quando...". E confirme que você de fato especificou o formato de saída; metade dos "não seguiu instruções" é "seguiu, mas no formato que você não pediu".

### Custo explodindo

Diagnóstico em ordem: (1) você está no modelo certo para a tarefa, ou usando um Opus onde um Haiku bastaria? (2) o caching está realmente funcionando (veja acima)? (3) você está incluindo contexto demais "por garantia"? (4) o volume de tokens de _saída_ — que custa mais — está controlado? Monte um dashboard de custo e alertas; o custo raramente explode de uma vez, ele sobe aos poucos até alguém reparar.

### Alucinações em produção

Mitigação em camadas: no prompt, peça fontes e permita explicitamente "não sei"; na arquitetura, dê ao modelo acesso a informação verificável (busca na web, RAG, o grafo do Graphify) em vez de confiar na memória dele; na saída, valide programaticamente o que for verificável. Nenhuma camada sozinha resolve; juntas, reduzem muito o problema.

### Latência alta

A latência tem duas partes: o tempo até o primeiro token (dominado por processar a entrada inteira) e a velocidade de geração depois disso. Para o primeiro, reduza o tamanho da entrada e use caching — um prefixo cacheado é processado muito mais rápido. Para a percepção, use **streaming**: o usuário vê o primeiro token cedo. Para trabalho pesado com modelos de ponta em alto esforço, lembre que turnos podem durar minutos — projete timeouts, streaming e indicadores de progresso, e estruture o trabalho para que o usuário acompanhe de forma assíncrona.

### Comportamento de ferramentas e integrações

Se uma ferramenta "falha em silêncio", verifique a fronteira: a chave/credencial está correta e acessível? Sob rede restrita, o host de destino está liberado? Resultados de ferramentas de servidor (busca na web, por exemplo) não levantam exceção — voltam como um bloco de resultado com um objeto de erro dentro, então cheque o conteúdo antes de assumir sucesso. E em chamadas paralelas de ferramentas, devolva todos os resultados numa única mensagem; espalhá-los em mensagens separadas ensina o modelo, sem querer, a parar de paralelizar.

### Migração de modelo

Ao trocar para um modelo mais novo, confirme o **ID exato** (sem sufixo inventado), remova os parâmetros legados (`temperature`/`top_p`/`top_k`, `budget_tokens`, prefills) que agora devolvem 400, e ajuste o `max_tokens` e os gatilhos de contexto, porque a contagem de tokens pode mudar entre gerações. Faça um teste com uma única requisição antes de implantar em massa, e verifique no `response.model` que o modelo certo foi de fato usado. Trocar o modelo também invalida o cache existente — a primeira requisição no modelo novo escreve o cache do zero.

## Checklist Final de Integração

Antes de colocar uma aplicação Claude em produção, percorra esta lista:

- [ ] **Modelo escolhido por etapa**, não por hábito — barato onde dá, caro onde precisa.
- [ ] **Prompts estruturados** (papel/tarefa/contexto/formato) e versionados.
- [ ] **Caching ativo e verificado** — prefixo estável primeiro, volátil por último, `cache_read_input_tokens` confirmando.
- [ ] **Tratamento de erros** com chain de exceções (retentável vs. não) e backoff.
- [ ] **Validação nas fronteiras** e verificação de todo output antes de agir.
- [ ] **Segredos fora do prompt e do sandbox.**
- [ ] **Conteúdo externo tratado como não confiável** (defesa contra prompt injection).
- [ ] **Streaming** para respostas longas e melhor latência percebida.
- [ ] **Monitoramento de custo** com dashboard e alertas.
- [ ] **Plano de migração de modelo** — IDs exatos, parâmetros legados removidos, teste antes do rollout.
- [ ] **Ponto de controle humano** definido para ações de alto risco (human-in/on-the-loop).
- [ ] **Arquitetura fail-safe** — em erro ou baixa confiança, o padrão é parar/escalar, não prosseguir.
- [ ] **Ameaças modeladas** contra o OWASP LLM Top 10 e o MITRE ATLAS, com mitigação para cada uma.

## Controle e Confiabilidade: mantendo o sistema sob rédea

Validar output e tratar erro é o mínimo. Sistemas sérios adicionam **controles** — mecanismos que decidem _quando o sistema pode agir sozinho, quando um humano entra, e o que fazer na dúvida_. Cinco padrões cobrem a maioria dos casos:

- **Human-in-the-loop (humano no laço).** Um humano **aprova ou edita antes** de a ação executar. Controle máximo, autonomia mínima — a escolha certa para ações de alto risco e irreversíveis (mover dinheiro, apagar dados, publicar para clientes). O agente propõe; a pessoa dispara.
- **Human-on-the-loop (humano sobre o laço).** O sistema age **sozinho por padrão**, mas um humano **supervisiona e pode intervir ou abortar** a qualquer momento — um painel de monitoramento e um "botão de parada". É o equilíbrio para volume alto onde aprovar cada ação seria inviável, mas o erro precisa ser pego a tempo. (O "human-on-frontiers" a que você se referiu é este padrão aplicado às **fronteiras** do sistema: a supervisão se concentra onde o agente toca o mundo externo.)
- **Arquitetura fail-safe (falha segura).** Quando algo dá errado ou a confiança cai, o padrão deve ser a **ação segura** — parar, negar, escalar — e nunca "seguir em frente na dúvida". É o oposto de *fail-open*. Para um agente: se a validação de saída falha, se a confiança declarada cai abaixo do limiar, ou se uma ferramenta retorna erro, o caminho padrão é **bloquear e escalar**, não improvisar. (Conecta ao escalonamento do Capítulo 8: um agente que sabe quando não sabe.)
- **Votação e ensemble.** Rodar a tarefa **mais de uma vez** — ou em **mais de um modelo** — e agregar reduz a variância e pega erros. A *self-consistency* (Wang et al., 2022) amostra vários caminhos de raciocínio e vota na resposta majoritária; o padrão **LLM-as-a-judge** / júris de modelos (Zheng et al., 2023) usa um segundo modelo para avaliar a saída do primeiro. Custa mais tokens, então reserve para decisões críticas.
- **Restrição de saída (guardrails) e watchdog.** Duas linhas de defesa complementares. Os **guardrails** _restringem o que o modelo pode produzir_: saída forçada por schema (Cap. 8), listas de permissão, classificadores que filtram conteúdo fora do escopo, recusa de temas proibidos. O **watchdog** (ou "firewall de LLM") é um **componente separado** — regras ou um segundo modelo — que inspeciona entradas e saídas em busca de injeção de prompt, vazamento de PII ou violação de política, e **bloqueia ou alerta** antes que o dano ocorra. Guardrail molda a resposta; watchdog vigia a fronteira.

A régua para escolher: quanto **maior o risco e a irreversibilidade** da ação, mais o controle pende para o humano-no-laço e o fail-safe; quanto maior o **volume e a tolerância a erro**, mais dá para operar com humano-sobre-o-laço e guardrails automáticos.

## Segurança de Sistemas LLM: modelando ameaças

Um sistema com LLM tem uma superfície de ataque que os sistemas tradicionais não têm — a começar pelo fato de que **modelo trata instrução e dado no mesmo canal**, o que abre a porta para a injeção de prompt. Projetar defesa exige um **modelo de ameaças**, e há três referências que valem como base (a dimensão legal, de privacidade e de jailbreak fica no Capítulo 10; aqui o foco é técnico e arquitetural).

### Dos frameworks clássicos ao específico de IA: ATT&CK e ATLAS

O **MITRE ATT&CK** é a base de conhecimento consagrada de táticas e técnicas de adversários contra sistemas de TI tradicionais — um vocabulário comum para descrever _como_ um ataque acontece, da reconnaissance ao impacto. Ele não cobre, porém, as ameaças **específicas de IA/ML**. Para isso existe o **MITRE ATLAS** (*Adversarial Threat Landscape for Artificial-Intelligence Systems*), que **estende o ATT&CK ao mundo da IA**: uma base viva com **16 táticas** e dezenas de técnicas observadas em ataques reais a sistemas de ML e LLM — de reconhecimento e envenenamento de dados a evasão de modelo, roubo de modelo, injeção de prompt e envenenamento de RAG. Usar o ATLAS é percorrer a "cadeia de ataque" de IA e perguntar, em cada etapa, "como um adversário exploraria isto, e o que me defende?".

### OWASP Top 10 para Aplicações de LLM (2025): o checklist acionável

Se o ATLAS é o mapa do território, o **OWASP Top 10 for LLM Applications** é a lista de verificação prática — os dez riscos mais críticos de quem _constrói_ com LLM, cada um com mitigação conhecida:

| # | Risco | Mitigação essencial |
| --- | --- | --- |
| **LLM01** | **Prompt Injection** (direta e indireta) | tratar conteúdo externo como não confiável; separar instrução de dado; validar saída; privilégio mínimo nas ferramentas |
| **LLM02** | **Vazamento de informação sensível** | não pôr segredos/PII no contexto; filtrar saída; minimizar dados |
| **LLM03** | **Cadeia de suprimentos** | verificar procedência de modelos, plugins e datasets |
| **LLM04** | **Envenenamento de dados e modelo** | curar e validar dados de treino/RAG; assinar fontes |
| **LLM05** | **Tratamento inseguro de saída** | nunca executar/renderizar saída do LLM sem validar (evita XSS, SQLi, RCE) |
| **LLM06** | **Agência excessiva** | limitar ferramentas, permissões e autonomia ao mínimo; human-in-the-loop para ações de risco |
| **LLM07** | **Vazamento do system prompt** | não guardar segredo no system prompt; assumir que ele pode vazar |
| **LLM08** | **Fraquezas de vetores e embeddings (RAG)** | controle de acesso no índice; validar o que entra no RAG |
| **LLM09** | **Desinformação** | dar fontes verificáveis; validar fatos; permitir "não sei" |
| **LLM10** | **Consumo ilimitado** | limites de taxa e de tokens; teto de custo; timeouts |

O fio que conecta quase todos: a **injeção de prompt (LLM01)** é a raiz de muitos outros, e a mitigação-mãe é a **fronteira de confiança** do Capítulo 6 — todo conteúdo que o modelo não gerou (página web, e-mail, resultado de ferramenta, documento de RAG) é entrada não confiável, e toda ação de efeito colateral passa por validação e privilégio mínimo. Some-se a isso o **NIST AI Risk Management Framework** para a camada de governança (as funções *Govern / Map / Measure / Manage*), que dá o processo organizacional em volta desses controles técnicos.

## Referências

- OWASP, *Top 10 for LLM Applications (2025)* — <https://genai.owasp.org/llm-top-10/>
- MITRE ATLAS, *Adversarial Threat Landscape for AI Systems* — <https://atlas.mitre.org/>
- MITRE ATT&CK — <https://attack.mitre.org/>
- NIST, *AI Risk Management Framework (AI RMF)* — <https://www.nist.gov/itl/ai-risk-management-framework>
- Wang et al., *Self-Consistency Improves Chain of Thought Reasoning* (votação) — <https://arxiv.org/abs/2203.11171>
- Zheng et al., *Judging LLM-as-a-Judge* (júris de modelos) — <https://arxiv.org/abs/2306.05685>
- Anthropic, *Responsible Scaling Policy* — <https://www.anthropic.com/rsp>

### Resumo do Capítulo 9

Integrar não é usar tudo; é escolher conscientemente cada peça por custo, qualidade ou segurança, e encaixá-las num fluxo coerente — modelo por etapa, cache na arquitetura, validação nas fronteiras, falha tratada como regra, segredos fora do alcance do modelo. Os problemas de produção caem em seis famílias (configuração, contexto, custo, qualidade, ferramentas, segurança), e o diagnóstico é metódico: identifique o sintoma, vá à causa provável, corrija na fronteira certa. Sobre isso vêm os **controles** — human-in/on-the-loop, arquitetura fail-safe, votação, guardrails e watchdog — que decidem quando o sistema age sozinho e quando o humano entra. E, porque um LLM amplia a superfície de ataque, a segurança se projeta contra um modelo de ameaças explícito: o **MITRE ATLAS** como mapa do território, o **OWASP Top 10 para LLM** como checklist acionável, e o **NIST AI RMF** como processo de governança — com a injeção de prompt no centro e a fronteira de confiança como mitigação-mãe. Com a teoria dos capítulos anteriores e o guia deste, você tem o que precisa para construir sistemas reais, seguros — e para consertá-los quando, inevitavelmente, eles pedirem conserto.

---
