# Apostila Completa de Claude

> *"Os limites da minha linguagem significam os limites do meu mundo."*
> — Ludwig Wittgenstein, *Tractatus Logico-Philosophicus*

Uma apostila técnica, em português, para quem quer **entender Claude de verdade** — não apenas apertar botões, mas saber o que acontece por baixo, por que acontece, e como tirar proveito disso. O estilo é direto: explica o conceito, mostra um exemplo e é honesto sobre os custos e as armadilhas.

Vai dos **fundamentos de LLMs** (tokens, embeddings, atenção, Transformers) até a **arquitetura de agentes** (o loop agêntico, MCP, engenharia de arnês), passando por Claude Code na prática, prompt engineering, otimização de custo, integrações, resolução de problemas e as questões de segurança, legais e éticas.

## Para quem é

- Quem está começando com Claude e quer uma base sólida, não receitas soltas.
- Desenvolvedores que vão construir com a **API**, o **Claude Code** e o **MCP**.
- Quem se prepara para a certificação **Claude Certified Architect – Foundations (CCA-F)**.

## Índice

| # | Capítulo |
| --- | --- |
| — | [Índice e apresentação](00_indice.md) |
| 1 | [Fundamentos](01_fundamentos.md) — o que é um LLM, tokens, embeddings, Transformers, atenção |
| 2 | [O que é Claude](02_o_que_e_claude.md) — modelos, plataformas, preços e fluxo de processamento |
| 3 | [Claude Code na Prática](03_claude_code_na_pratica_instalacao_custos_e_operacao.md) — instalação, `settings.json`, `CLAUDE.md`, hooks |
| 4 | [Prompt Engineering](04_prompt_engineering.md) — estrutura, técnicas, CoT, TOON, avaliação |
| 5 | [Prática e Referência](05_pratica_e_referencia.md) — prompt × workflow × agente, playbook, casos de uso |
| 6 | [Integrações e Extensões](06_integracoes_e_extensoes.md) — capacidades nativas, Graphify, MCP |
| 7 | [Otimização e Performance](07_otimizacao_e_performance.md) — tokens, caching, custo, latência |
| 8 | [Arquitetura de Agentes, Claude Code e MCP](08_arquitetura_de_agentes_claude_code_e_mcp_em_profundidade.md) — loop agêntico, subagentes, arnês |
| 9 | [Integração e Resolução de Problemas](09_integracao_e_resolucao_de_problemas.md) — diagnóstico, controles, segurança (OWASP/MITRE) |
| 10 | [Segurança, Legal e Ética](10_seguranca_legal_e_etica.md) — IP, compliance, alinhamento, consciência |
| — | [Apêndice](11_apendice.md) — glossário, recursos e licença |

## Como estudar

Os capítulos seguem uma progressão pedagógica — da teoria à prática, ao avançado e à consolidação. Cada capítulo termina com um resumo e uma seção de referências. Se você é novo em Claude, comece pelo Capítulo 1 e siga na ordem; se busca algo específico, o índice acima é o atalho.

Para acompanhar na prática, o caminho mais barato é criar uma conta **Free** em [claude.ai](https://claude.ai) e, quando quiser usar o Claude Code de verdade, assinar o plano **Pro** — o Capítulo 3 explica os planos em detalhe. Se este material te ajudou, você pode criar sua conta por este link de indicação: **<https://claude.ai/referral/ZvWllcDXBg>**.

## Guia de leitura rápida

Não precisa ler tudo de uma vez. Escolha a trilha pelo seu objetivo:

- **🟢 Só um overview / o básico** (comece por aqui) — **Capítulos 1 → 2 → 3**. Você entende o que é um LLM, o que é Claude (modelos e planos) e sai usando o Claude Code na prática. Se quiser já escrever prompts melhores, some o **Capítulo 4**. É o suficiente para 80% do uso do dia a dia.
- **🔵 Usar bem no dia a dia** — a trilha básica **+ Capítulo 5** (playbook: como obter os melhores resultados) **e Capítulo 7** (controlar custo e latência).
- **🟣 Construir com API, agentes e integrações** — **+ Capítulos 6** (integrações e MCP) **e 8** (arquitetura de agentes, o loop, o arnês). É também a trilha de quem se prepara para a certificação **CCA-F**.
- **🔴 Levar para produção com responsabilidade** — **Capítulos 9** (diagnóstico, controles e segurança OWASP/MITRE) **e 10** (IP, compliance, ética e alinhamento).
- **📖 Consulta rápida** — o **Apêndice** traz um glossário e os recursos essenciais.

> Se você tem 30 minutos e só quer a ideia geral, leia os **resumos** ao fim dos Capítulos 1, 2 e 3.

## Autoria

Carlos Alvaro de M. S. Quintella, em colaboração com o Claude (Anthropic).

## Licença — CC BY 4.0 (uso livre com atribuição)

Esta obra está licenciada sob a **[Creative Commons Atribuição 4.0 Internacional (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/)**. Você pode **copiar, redistribuir, adaptar, traduzir, imprimir e reutilizar** este material — no todo ou em parte, para qualquer fim, **inclusive comercial** — sem pedir permissão e sem pagar nada, **desde que dê o devido crédito ao material original** (cite o autor, indique se houve mudanças e forneça o link para a fonte) e não sugira que os autores endossam você ou seu uso.

**Atribuição sugerida:** *"Apostila Completa de Claude*, de Carlos Alvaro de M. S. Quintella (com Claude/Anthropic) — https://github.com/cquintella/apostila-claude — licenciada sob CC BY 4.0."

> *Nota sobre atualidade:* o campo evolui rápido. Modelos, preços, termos e leis mudam; o texto reflete o cenário até meados de 2026, e cada capítulo aponta as fontes oficiais para consulta atualizada.
