# Apêndice

## Glossário de Termos

- **Token**: a unidade em que o texto é fragmentado para o modelo; a moeda de cobrança e a unidade da janela de contexto.
- **LLM (Large Language Model)**: modelo treinado para prever o próximo token; o "motor".
- **Context Window**: o máximo de tokens (entrada + saída) que o modelo considera de uma vez.
- **Prompt Caching**: reuso do processamento de um prefixo estável de prompt; ~90% de desconto na leitura; a escrita custa mais que o normal (1,25× no cache de 5 min, 2× no de 1 h).
- **Constitutional AI**: método de treinamento da Anthropic em que o modelo critica e revisa as próprias respostas segundo princípios explícitos.
- **Few-shot learning**: ensinar a tarefa por meio de exemplos no próprio prompt.
- **Chain-of-thought**: pedir ao modelo que raciocine em passos antes de responder.
- **Adaptive thinking**: raciocínio interno em que o modelo decide quando e quanto pensar, controlado por um nível de esforço; substitui o orçamento fixo de tokens de pensamento.
- **Effort (esforço)**: parâmetro (`low`/`medium`/`high`/`xhigh`/`max`) que controla a profundidade do raciocínio e o gasto de tokens nos modelos recentes.
- **MCP (Model Context Protocol)**: protocolo padrão para conectar modelos a ferramentas e fontes de dados externas.
- **Prompt injection**: ataque em que instruções maliciosas são inseridas em dados que o modelo processa.
- **Streaming**: receber a resposta token a token, em vez de esperar o fim.
- **Knowledge graph**: representação do conhecimento como nós e arestas; no Graphify, a estrutura consultável extraída de um codebase.
- **Loop agêntico**: o ciclo, rodado pelo seu código, de chamar o modelo, inspecionar o `stop_reason`, executar a ferramenta pedida e anexar o `tool_result` antes de chamar de novo.
- **`stop_reason`**: campo da resposta da API que diz se o modelo terminou (`end_turn`) ou está pedindo a execução de uma ferramenta (`tool_use`).
- **Agent SDK**: framework da Anthropic, por cima da API, que roda o loop agêntico, gera subagentes e traz ferramentas embutidas.
- **Subagente**: Claude de escopo restrito gerado por um coordenador (via ferramenta `task`) para isolar uma subtarefa e poupar contexto.
- **`tool_choice` / `input_schema`**: forma confiável de obter saída estruturada — declarar o formato como uma ferramenta e forçar o modelo a chamá-la.
- **Lost in the middle**: tendência do modelo a atender ao começo e ao fim do contexto e ignorar o meio; mitigada reancorando fatos duráveis num *case block* no fim da janela.
- **STDIO / SSE**: os dois transportes do MCP — STDIO para servidor na mesma máquina, SSE para servidor remoto via HTTP.
- **CCA-F**: Claude Certified Architect – Foundations, a primeira certificação oficial da Anthropic (60 questões, 5 domínios).
- **Agente**: LLM que, dentro de um loop, decide quais ferramentas usar e quando parar para cumprir um objetivo (percepção = ler contexto/resultados; ação = chamar ferramenta).
- **Workflow**: sistema de vários passos de LLM e ferramentas orquestrados por **código que você escreveu** — previsível; oposto do agente, em que o modelo dirige.
- **Arnês (harness)**: a infraestrutura em volta do modelo (`.claude/`, hooks, permissões, estado, memória) que governa *quando, onde e como* ele age; "o modelo é inteligente, o arnês o torna confiável".
- **Guardrails**: restrições sobre o que o modelo pode produzir (schema, listas de permissão, filtros de saída) — moldam a resposta.
- **Watchdog**: componente separado (regras ou outro modelo) que vigia entradas e saídas em busca de injeção, vazamento ou violação de política, e bloqueia ou alerta.
- **Human-in-the-loop / on-the-loop**: humano que **aprova antes** de agir (in) ou que **supervisiona e pode intervir** enquanto o sistema age sozinho (on).
- **Fail-safe**: arquitetura em que, na dúvida ou no erro, o padrão é a ação segura — parar, negar, escalar — nunca "seguir em frente".
- **RAG (Retrieval-Augmented Generation)**: dar ao modelo acesso a informação recuperada de uma base externa em vez de confiar só na memória dos pesos.
- **Alucinação**: resposta fabricada que parece verdadeira; mitigada com fontes verificáveis, permissão para "não sei" e validação de saída.
- **TOON (Token-Oriented Object Notation)**: formato compacto e sem perdas para enviar dados tabulares ao modelo gastando menos tokens que o JSON.
- **OWASP LLM Top 10 / MITRE ATLAS**: os dois frameworks de referência para ameaças de segurança de aplicações com LLM (checklist acionável e mapa de táticas de adversários, respectivamente).
- **Model welfare**: linha de pesquisa que investiga se, e sob que condições, um sistema de IA mereceria consideração moral.

## Recursos Úteis

- **Documentação oficial da Anthropic** (<https://docs.claude.com>) — a fonte de verdade para IDs de modelo, preços, limites e recursos novos. Consulte sempre que o cenário puder ter mudado.
- **Anthropic Academy** (<https://anthropic.skilljar.com>) e **anthropic.com/learn** — cursos gratuitos e a trilha de certificação (CCA-F).
- **Anthropic Console** (<https://console.anthropic.com>) — painel de uso e custo, chaves de API e o *prompt workbench*.
- **Comunidades** — Discord e Reddit dedicados, onde práticas e armadilhas circulam rápido.
- **Ferramentas recomendadas** — Claude Code (a CLI `claude`), os SDKs oficiais da Anthropic (Python e TypeScript), e o Graphify (grafo de conhecimento para codebases).
- **Segurança de LLM** — o OWASP Top 10 for LLM Applications (<https://genai.owasp.org/llm-top-10/>) e o MITRE ATLAS (<https://atlas.mitre.org/>) para modelar ameaças.

## Feedback e Contribuições

Esta apostila é um documento vivo. Erros, omissões e sugestões de melhoria são bem-vindos — o campo de IA, e Claude em particular, evolui rápido o suficiente para que nenhuma apostila fique "pronta" por muito tempo. Ao reportar um problema, indique o capítulo e a seção, descreva o que está incorreto ou faltando, e, se possível, proponha a correção. Modelos, preços e recursos mudam; manter o material atualizado é trabalho coletivo.

## Direitos e Uso (Anti-Copyright)

> **Anti-copyright — uso livre.** Esta obra é liberada deliberadamente de qualquer
> restrição de copyright. Você pode copiar, redistribuir, adaptar, traduzir, imprimir e
> reutilizar este material — no todo ou em parte, para qualquer fim, inclusive comercial —
> sem pedir permissão e sem pagar nada. Não há direitos reservados. A atribuição aos
> autores é apreciada, mas não é obrigatória. O conhecimento aqui reunido pertence a quem
> quiser usá-lo: copie à vontade.
