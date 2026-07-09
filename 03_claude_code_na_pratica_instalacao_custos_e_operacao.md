
# Capítulo 3: Claude Code na Prática — Instalação, Configuração e Operação

Este capítulo é o lado operacional do dia a dia — instalar passo a passo, configurar os arquivos certos, escrever um `CLAUDE.md` que realmente guia o agente, vigiar o custo, manter o contexto sob controle, automatizar auditorias e integrar a ferramenta ao seu editor. É o conhecimento que separa "consegui rodar" de "uso isto o dia inteiro sem susto na fatura". A arquitetura interna do agente — o loop, os subagentes, a hierarquia completa de configuração — é o assunto do Capítulo 8; aqui o foco é colocar tudo para funcionar.

## Instalação passo a passo

**Pré-requisitos.** Claude Code roda em **macOS**, **Linux** e **Windows** (neste último, de preferência via WSL2). Você precisa de uma de duas formas de acesso: uma **assinatura Claude Pro ou Max** (o uso do Claude Code entra na cota da assinatura) **ou** uma conta no **Anthropic Console** com créditos de API (cobrança por token). A via da assinatura é a mais simples para uso individual; a da API dá controle fino de custo e é a escolha para times e automação.

**Passo 1 — Instalar.** A forma mais direta baixa e executa o script oficial:

```bash
curl -fsSL https://claude.ai/install.sh | bash
```

Quem já vive no ecossistema Node pode preferir:

```bash
npm install -g @anthropic-ai/claude-code
```

**Passo 2 — Primeira execução e login.** Entre num diretório de projeto e rode o comando:

```bash
cd meu-projeto
claude
```

Na primeira vez, o Claude Code pergunta o **tema de cores** e o **método de login**. Escolha entre entrar com a **assinatura** (abre o navegador para autenticar na sua conta Claude) ou colar uma **chave de API** do Console. A escolha fica salva; nas próximas vezes ele já entra autenticado.

**Passo 3 — Verificar.** Confirme que está tudo no lugar:

```bash
claude --version          # ex.: 2.1.204 (Claude Code)
```

E, dentro do console interativo, rode **`/status`** (mostra login, modelo ativo e diretório) e faça um teste rápido em modo não interativo:

```bash
claude -p "Olá, quem é você e em que diretório estamos?"
```

**Passo 4 — Gerar o primeiro `CLAUDE.md`.** Ainda dentro do projeto, rode **`/init`**. O agente varre o repositório e escreve um `CLAUDE.md` inicial com o que descobriu (linguagem, comandos de build/test, estrutura). É o ponto de partida — você o refina depois, como veremos adiante.

O básico de personalização (tema, modelo padrão, idioma, nível de esforço) é feito de forma interativa pelo comando **`/config`** dentro do console; não é preciso editar arquivo na mão para ajustar as preferências comuns.

### Planos de assinatura: qual escolher para aprender

Há dois modelos de cobrança para usar Claude, e vale entender a diferença antes de gastar. Um é a **API** (paga por token, detalhada no Capítulo 7) — flexível e granular, ideal para produção e automação. O outro é a **assinatura**, com valor fixo mensal e cota de uso, que **inclui o Claude Code** e é o caminho mais simples para quem está aprendendo. Os planos (valores aproximados; confira sempre a página oficial, porque mudam):

| Plano | Preço aprox. | Para quem | Claude Code? |
| --- | --- | --- | --- |
| **Free** | US$ 0 | experimentar o chat em claude.ai | Não (ou muito limitado) |
| **Pro** | ~US$ 20/mês | **aprender e uso individual** | Sim, com cota |
| **Max (5×)** | ~US$ 100/mês | quem esbarra na cota do Pro | Sim, cota bem maior |
| **Max (20×)** | ~US$ 200/mês | uso intenso o dia inteiro | Sim, cota máxima |
| **Team / Enterprise** | por assento / sob consulta | times e empresas | Sim, com administração |

> [!tip] Recomendação para quem está começando
> **Comece pelo mais barato.** Crie uma conta **Free** para explorar o chat e, quando quiser usar o Claude Code de verdade, assine o **Pro** (~US$ 20/mês) — é o menor plano que já libera o Claude Code com uma cota confortável para aprender, e cobre com folga tudo o que esta apostila pede. Só suba para o **Max** se você começar a esbarrar na cota do Pro com frequência; não faz sentido pagar por capacidade que você ainda não usa. Se este material te ajudou, você pode criar sua conta pelo meu link de indicação: <https://claude.ai/referral/ZvWllcDXBg>.

### Comandos essenciais (cheatsheet)

Os comandos são digitados com barra (`/`) dentro do console. Os que você mais vai usar:

| Comando | O que faz |
| --- | --- |
| `/init` | Analisa o repositório e gera um `CLAUDE.md` inicial |
| `/config` | Ajusta preferências (tema, modelo, idioma, esforço) |
| `/status` | Mostra login, modelo ativo e diretório |
| `/model` | Troca o modelo da sessão |
| `/cost` | Gasto acumulado da sessão ao vivo |
| `/compact` | Resume o histórico, liberando contexto (mesma tarefa) |
| `/clear` | Zera o contexto ativo (tarefa nova) |
| `/permissions` | Editor interativo das permissões de ferramentas |
| `/hooks` | Editor interativo dos hooks |
| `/output-style` | Alterna o estilo de resposta (`explanatory`, `learning`) |
| `/plugin` | Gerencia plugins e o marketplace |
| `/help` | Lista todos os comandos disponíveis |

A lista completa e sempre atualizada está na documentação oficial: [referência de comandos](https://code.claude.com/docs/en/commands) e [referência de CLI](https://code.claude.com/docs/en/cli-reference).

### Criando seus próprios comandos

Além dos embutidos, você pode definir **slash commands próprios** — prompts reutilizáveis que viram um comando com barra. Basta soltar um arquivo Markdown em **`.claude/commands/`** (por projeto) ou em `~/.claude/commands/` (pessoal): o nome do arquivo vira o nome do comando (`revisar.md` → `/revisar`), e o **corpo do arquivo é o prompt** injetado quando o comando roda. Um `$ARGUMENTS` no corpo é substituído pelo que você digitar depois do comando.

O **front matter YAML** no topo restringe o comportamento. Os dois campos mais úteis:

- **`allowed-tools`** — limita quais ferramentas o comando pode chamar. Um comando de revisão pode ser travado em ferramentas só-leitura (`Read`, `Grep`), sem `Write` nem `Bash`.
- **`argument-hint`** — o placeholder mostrado no seletor, documentando que argumento o comando espera.

Um exemplo, `.claude/commands/revisar.md`:

```markdown
---
allowed-tools: Read, Grep, Glob
argument-hint: [arquivo ou pasta]
---
Revise $ARGUMENTS focando em segurança e concorrência.
Liste os problemas do mais grave ao menos grave; não altere nenhum arquivo.
```

Depois é só digitar `/revisar src/auth.ts`. Como o comando está versionado no repositório, ele vira uma **ferramenta compartilhada do time** — todo mundo revisa do mesmo jeito. (O papel dos comandos na arquitetura do agente e no exame CCA-F reaparece no Capítulo 8.)

> [!note] Comandos e skills
> Nas versões recentes do Claude Code, os comandos customizados foram **unificados às [skills](https://code.claude.com/docs/en/skills)**: um arquivo em `.claude/commands/deploy.md` e uma skill em `.claude/skills/deploy/SKILL.md` criam o mesmo `/deploy` e funcionam igual. Os arquivos em `.claude/commands/` continuam válidos; a skill só adiciona recursos extras (pasta de arquivos de apoio e carregamento automático quando relevante).

### Criando uma skill

Criar uma skill é simples: é **uma pasta com um arquivo `SKILL.md`** dentro. O `SKILL.md` tem duas partes — um **front matter YAML** com dois campos obrigatórios e um **corpo em Markdown** com as instruções que o Claude segue quando a skill está ativa. A pasta vai em `.claude/skills/<nome>/` (por projeto, versionada) ou em `~/.claude/skills/<nome>/` (pessoal). O esqueleto:

```markdown
---
name: nome-da-skill
description: Descrição clara do que a skill faz e quando usá-la
---

# Nome da Skill

[Instruções que o Claude vai seguir quando esta skill estiver ativa.]

## Exemplos
- Exemplo de uso 1
- Exemplo de uso 2

## Diretrizes
- Diretriz 1
- Diretriz 2
```

Os dois campos do front matter:

- **`name`** — um identificador único (minúsculas, hífens no lugar de espaços).
- **`description`** — o campo que mais importa: é por ele que o Claude decide **quando** ativar a skill automaticamente. Descreva **o que a skill faz _e_ em que situação usá-la** ("Gera um CHANGELOG a partir dos commits desde a última tag; use ao preparar um release"). Uma descrição vaga faz a skill nunca disparar na hora certa.

O corpo carrega as instruções, exemplos e diretrizes. E aqui está o ganho de arquitetura (Capítulo 7): a skill **só entra no contexto quando o gatilho a ativa** — material de referência longo custa quase nada até ser usado, ao contrário do `CLAUDE.md`, que é pago em toda sessão. É por isso que "mova o detalhe para uma skill" é o conselho recorrente para manter o `CLAUDE.md` enxuto.

O jeito mais rápido de começar é copiar a **`template-skill`** do repositório [`anthropics/skills`](https://github.com/anthropics/skills) e adaptar; a documentação de [skills](https://code.claude.com/docs/en/skills) cobre os recursos avançados (controle de quem invoca, execução em subagente, injeção de contexto dinâmico).

## O mapa dos arquivos de configuração

Antes de configurar, vale ter o mapa de _onde cada coisa mora_. Claude Code separa **configuração operacional** (JSON, em `settings.json`) de **instruções em linguagem natural** (Markdown, em `CLAUDE.md`), e cada uma tem uma versão **pessoal** (na sua home, nunca versionada) e uma **de projeto** (no repositório, versionada e compartilhada com o time).

| Arquivo | Escopo | Versionado? | Para quê |
| --- | --- | --- | --- |
| `~/.claude/settings.json` | Pessoal, todos os projetos | Não | Seu modelo padrão, permissões e hooks globais |
| `~/.claude/CLAUDE.md` | Pessoal, todos os projetos | Não | Suas preferências de estilo e ferramentas que usa em tudo |
| `CLAUDE.md` (raiz do projeto) | Projeto (time) | **Sim** | Convenções, comandos e regras do repositório |
| `.claude/settings.json` | Projeto (time) | **Sim** | Permissões e hooks que devem valer para todos |
| `.claude/settings.local.json` | Projeto (só você) | Não (fica no `.gitignore`) | Ajustes pessoais naquele projeto |
| `.claude/commands/*.md` | Projeto | Sim | Slash commands reutilizáveis (Capítulo 8) |
| `.claude/agents/*.md` | Projeto | Sim | Subagentes de contexto próprio (Capítulo 8) |
| `.claude/skills/` e `~/.claude/skills/` | Projeto / pessoal | Projeto sim, pessoal não | Habilidades carregadas sob demanda |
| `.mcp.json` | Projeto | Sim | Servidores MCP registrados (Capítulo 6) |

A regra de ouro que atravessa a tabela inteira: **o que é seu vai na home (`~/.claude/`) e nunca entra no Git; o que é do time vai no repositório e é commitado**. Confundir os dois é o erro clássico — pôr uma convenção do time no seu arquivo pessoal significa que o clone do colega nunca a verá.

Para **skills globais** (uma habilidade de administração de sistemas, um padrão de revisão que você usa em todo projeto), o lugar é `~/.claude/skills/`, disponível em qualquer sessão sem repetir instruções. Para se inspirar e ver o formato pasta + `SKILL.md` na prática, o melhor ponto de partida é o **repositório oficial de skills da Anthropic** ([`anthropics/skills`](https://github.com/anthropics/skills)) — traz desde as skills de criação de documentos (PDF, DOCX, PPTX, XLSX) até exemplos criativos, técnicos e de workflow. Além dele, as listas *Awesome Claude Code* da comunidade e o marketplace de plugins do Claude Code (`/plugin`) reúnem coleções mantidas ativamente.

## O arquivo `settings.json` na prática

Enquanto os `CLAUDE.md` são instruções em linguagem natural, o **`settings.json`** guarda parâmetros estritamente operacionais em JSON: o **modelo** padrão, as **permissões** de ferramentas, **variáveis de ambiente**, o **nível de esforço** e os **hooks**. Um exemplo pessoal (`~/.claude/settings.json`) enxuto e comentado do que cada bloco faz:

```json
{
  "model": "claude-opus-4-8",
  "effortLevel": "high",
  "theme": "dark",
  "permissions": {
    "allow": [
      "Bash(git status)",
      "Bash(git diff:*)",
      "Bash(npm run test:*)",
      "Read(~/projetos/**)"
    ],
    "ask": [
      "Bash(git push:*)"
    ],
    "deny": [
      "Bash(rm -rf:*)",
      "Read(./.env)"
    ]
  },
  "env": {
    "BASH_DEFAULT_TIMEOUT_MS": "120000"
  }
}
```

O bloco que mais rende no dia a dia é o **`permissions`**. Cada regra tem a forma `Ferramenta(especificador)` e cai em uma de três listas:

- **`allow`** — roda **sem pedir confirmação**. Coloque aqui os comandos seguros e repetitivos (leitura, `git status`, sua suíte de testes). É o que dá fluidez: você para de apertar "sim" o tempo todo.
- **`ask`** — sempre **pede confirmação** antes de rodar. Bom para ações com efeito externo, como `git push`.
- **`deny`** — **bloqueia** de vez. Aqui vão as ações destrutivas (`rm -rf`) e a leitura de segredos (`./.env`), que o agente nunca deve tocar.

Um `settings.json` **de projeto** (`.claude/settings.json`, commitado) costuma liberar os comandos do próprio repositório para todo o time — por exemplo `Bash(pnpm build)` e `Bash(pnpm test:*)`. E quando você quer um ajuste que vale **só para você naquele projeto** (uma permissão extra, uma variável de ambiente local), o lugar é o **`.claude/settings.local.json`**, que o Claude Code já adiciona ao `.gitignore` automaticamente. Assim a governança fica limpa: pessoal na home, do time no `.claude/settings.json`, e o seu-neste-projeto no `.local.json`.

> Dica: você raramente precisa editar o JSON na mão para permissões. O comando **`/permissions`** abre um editor interativo, e sempre que o agente esbarra numa permissão nova ele oferece "permitir uma vez / sempre / negar" — e grava a escolha no arquivo certo.

## Escrevendo um bom `CLAUDE.md`

O `CLAUDE.md` é o documento que o Claude Code costura no _system prompt_ de **toda** sessão iniciada naquele diretório (a mecânica exata da hierarquia está no Capítulo 8). Sem ele, cada sessão começa sem nenhuma memória do projeto. É, na prática, o **contrato** de como o agente deve se comportar — e escrevê-lo bem é o que mais muda a qualidade do resultado.

O princípio que rege um bom `CLAUDE.md` é **"um mapa, não uma enciclopédia"**: informação densa e acionável, não prosa. Uma regra de bolso corrente é mantê-lo **abaixo de ~300 linhas**; passou disso, a eficiência despenca, porque o agente "se perde no meio" (o efeito do Capítulo 1) e passa a ignorar diretrizes. Se estiver crescendo demais, quebre em `CLAUDE.md` por subpasta ou mova detalhe para skills.

### O que vai no `CLAUDE.md` **de projeto** (versionado)

Tudo que vale para **qualquer pessoa** que trabalhe no repositório:

- **Visão de uma linha do projeto** — o que é e para que serve.
- **Comandos válidos** — como instalar, buildar, testar, rodar o lint. Ex.: `pnpm test`, `pnpm build`. O agente usa isto para se verificar.
- **Convenções** — o gerenciador de pacotes (`use bun, não npm`), estilo de commits, padrões de código, bibliotecas preferidas.
- **Arquitetura em pinceladas** — onde ficam as coisas ("API em `packages/api`, UI em `packages/web`").
- **O que o agente _nunca_ deve fazer** — não commitar direto na `main`, não tocar em migrations sem avisar, não editar arquivos gerados.

### O que vai no seu `CLAUDE.md` **pessoal** (`~/.claude/CLAUDE.md`, não versionado)

Tudo que é **preferência sua** e vale em todo projeto, independentemente do time:

- Seu estilo de interação ("seja direto, sem preâmbulos"; "explique o _porquê_ das decisões").
- Ferramentas que você usa em qualquer lugar (`ripgrep` no lugar de `grep`, `fd`, seu editor).
- Idioma das respostas e dos comentários de commit.

Nunca ponha convenção do time aqui — ela não viaja com o repositório.

### Estruturando o `CLAUDE.md` para os três modos de trabalho

Um mesmo `CLAUDE.md` precisa servir três formas de usar o agente, e cada uma se apoia numa parte diferente do arquivo:

- **Planejar** — quando você pede um plano antes de mexer no código (o *plan mode*, acionado com `Shift+Tab`). O que ajuda aqui é uma **definição explícita de "pronto"** e as **restrições** ("features uma de cada vez", "todo código novo tem teste"). Sem isso, o plano vem largo demais.
- **Desenvolver** — a execução. O que ajuda são os **comandos de build/test/lint** e as **convenções de código**, para o agente escrever no seu padrão e se auto-verificar rodando os testes.
- **Conversar** — quando você só quer entender o repositório ou discutir uma abordagem. O que ajuda é a **visão de arquitetura** e o **glossário** de termos do domínio, para o agente responder com o vocabulário certo.

Um esqueleto enxuto que cobre os três:

```markdown
# Projeto Acme API

API de faturamento em TypeScript. Monorepo pnpm.

## Comandos
- Instalar:  pnpm install
- Testar:    pnpm test
- Lint:      pnpm lint
- Build:     pnpm build

## Convenções
- Gerenciador: pnpm (nunca npm/yarn).
- Commits: Conventional Commits.
- Erros: sempre tipados, nunca `throw "string"`.

## Arquitetura
- `packages/api`  — rotas e serviços
- `packages/db`   — schema e migrations (NÃO editar migrations à mão)

## Definição de "pronto"
- Testes passando + lint limpo. Uma feature por PR.

## Nunca
- Commitar direto na `main`.
- Ler ou imprimir segredos (`.env`).
```

Repare que não há prosa: cada linha é acionável. O `/init` gera um bom ponto de partida para você entender a estrutura, mas o conselho de quem escreve muitos `CLAUDE.md` é **não deixar o arquivo auto-gerado como versão final**: vale a pena escrevê-lo à mão, podando e afinando conforme percebe o agente errando — o `CLAUDE.md` é um documento vivo.

### Menos é mais: o orçamento de instruções

Vale aprofundar por que a concisão importa tanto, com alguns pontos pouco óbvios (bem destrinchados no artigo *"Writing a good CLAUDE.md"*, da HumanLayer). Um bom `CLAUDE.md` cobre três perguntas — o **QUÊ** (stack, estrutura, mapa do código), o **PORQUÊ** (o propósito do projeto e de cada parte) e o **COMO** (comandos de build, teste e verificação) — e nada além disso.

O motivo de parar por aí é concreto: **modelos de fronteira seguem de forma confiável só cerca de 150–200 instruções**, e o system prompt do próprio Claude Code já consome umas ~50 delas. O que sobra para o seu `CLAUDE.md` é um **orçamento pequeno** — por isso a regra não é só "< 300 linhas", mas **idealmente abaixo de ~60**. Cada linha que não vale para *toda* sessão gasta esse orçamento à toa.

E há um detalhe que muda a forma de escrever: **o Claude muitas vezes ignora o `CLAUDE.md`**. O system prompt instrui o modelo a descartar contexto irrelevante, então instruções que ele julga não se aplicarem à tarefa atual são simplesmente puladas — e, pior, **encher o arquivo de coisa irrelevante enfraquece o que importa**. Três consequências práticas:

- **Só o que é universal.** Instrução específica de uma tarefa não vai no `CLAUDE.md`; vai numa **divulgação progressiva** — arquivos Markdown separados que o `CLAUDE.md` referencia e o agente lê só quando precisa:
  ```
  agent_docs/
    building_the_project.md
    running_tests.md
    code_conventions.md
  ```
- **Estilo de código não é para o `CLAUDE.md`.** Não gaste instruções pedindo formatação ao modelo; deixe isso para **linters e formatadores determinísticos** (rodados por hooks, como vimos). É mais confiável e não custa orçamento.
- **Linhas ruins cascateiam.** Como o `CLAUDE.md` entra em toda sessão, uma linha mal escrita contamina a pesquisa, o plano e o código — em todas as tarefas, o tempo todo. É o maior multiplicador de qualidade (para o bem e para o mal) da sua configuração.

## Gerenciamento de custos e contexto na CLI

Controlar gasto e saturação de contexto é o que mantém o fluxo saudável em escala. O Claude Code traz instrumentos para isso embutidos no terminal.

Para **custo**, o comando **`/cost`** mostra o gasto acumulado na sessão ativa — útil para perceber, ao vivo, quando uma tarefa está saindo cara. (Na assinatura Pro/Max, mostra o consumo da cota.) Para uma visão consolidada e histórica (por dia, por projeto, por chave de API), o lugar certo é o **Anthropic Console** na web, cujo painel detalha o consumo em gráficos; atribuir **uma chave de API por projeto** permite auditar as despesas de cada repositório separadamente.

Para **contexto**, o rodapé do console exibe um **indicador de saturação** — quanto da janela já está ocupado. Quando o contexto fica pesado, dois comandos resolvem:

- **`/compact`** resume as mensagens anteriores e libera espaço imediato, **sem perder as diretrizes principais** da sessão. Use no meio de uma tarefa longa que ainda continua.
- **`/clear`** esvazia por completo a memória ativa e zera o consumo da sessão. Use ao **terminar uma tarefa e começar outra** sem relação — é mais barato e mais limpo do que arrastar o histórico antigo adiante.

A regra prática: `/compact` quando a _mesma_ tarefa continua, `/clear` quando a tarefa _muda_. Você pode ainda incluir, no `CLAUDE.md`, diretrizes que limitem o volume de leitura simultânea de arquivos, forçando o agente a ser econômico com contexto por padrão.

## Hooks: automação determinística

A automação de qualidade vem dos **hooks**: gatilhos configurados no `settings.json` que executam **comandos seus** em momentos definidos do ciclo do agente. Diferente de uma instrução no `CLAUDE.md` (que o modelo _pode_ seguir), um hook é executado pelo _harness_, **deterministicamente** — é assim que você garante, por exemplo, que o formatador roda depois de toda edição, sem depender da boa vontade do modelo.

Os principais eventos:

- **`PreToolUse`** — antes de o agente rodar uma ferramenta. Pode inspecionar e até **bloquear** a chamada (barrar um comando perigoso).
- **`PostToolUse`** — depois. Ideal para formatar, rodar lint ou registrar log.
- **`UserPromptSubmit`** — quando você envia uma mensagem (injetar contexto extra).
- **`Stop` / `SubagentStop`** — ao encerrar uma resposta ou um subagente (limpeza, verificação final).
- **`SessionStart` / `PreCompact`** — no início da sessão e antes de um `/compact`.

Um hook que roda o Prettier em todo arquivo editado ou escrito fica assim, no `.claude/settings.json` do projeto:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          { "type": "command", "command": "pnpm prettier --write \"$CLAUDE_PROJECT_DIR\" >/dev/null 2>&1" }
        ]
      }
    ]
  }
}
```

O campo **`matcher`** filtra quais ferramentas disparam o hook (aqui, só `Edit` e `Write`). O comando roda de verdade no seu shell — por isso o hook é confiável onde a instrução em texto falha. O comando **`/hooks`** oferece um editor interativo para montá-los sem escrever JSON à mão.

## Observabilidade e auditoria

Com custo e hooks sob controle, a camada seguinte é **enxergar o estado do projeto** ao longo do tempo. A estratégia é definir rotinas de checagem a partir dos dados que o próprio repositório oferece: o `diff` do Git, o histórico de commits, os logs. Incluir no `CLAUDE.md` instruções de auditoria — "ao revisar, examine os commits recentes, gere uma tabela de progresso e aponte riscos ou vulnerabilidades" — faz o agente produzir relatórios estruturados de conformidade.

Um **subagente de auditoria** dedicado (Capítulo 8) isola esse trabalho, comparando o estado atual com os requisitos formais sem poluir o contexto principal. E relatórios gerados assim podem ser **direcionados a canais externos** — um canal corporativo no Slack, um repositório central de documentação — para governança de time. (Esta é a camada de *Governance & Audit* da pilha agêntica do Capítulo 8, posta para funcionar no dia a dia.)

## Automação e agendamento

Rotinas recorrentes não precisam de gatilho humano. O Claude Code permite **agendar execuções** — auditorias periódicas de integridade da base, geração de logs de conformidade, sincronização de backups dos materiais para um destino seguro. O padrão de **validação contínua** é especialmente valioso: após alterações no código, o agente roda a suíte de testes, audita os resultados e aplica correções quando algo falha, fechando o ciclo sem intervenção. (Isto é o loop agêntico do Capítulo 8 posto a serviço de manutenção, não de desenvolvimento.)

## Integração com editores

A produtividade sobe quando o Claude Code convive com o seu ambiente de desenvolvimento. Rodar o console no **terminal embutido do VS Code ou do Cursor** dá feedback visual imediato na árvore de arquivos: você vê o agente ler e modificar o código enquanto acompanha os testes. Há ainda extensões de IDE dedicadas que aprofundam essa integração.

Para sessões longas ou conexões remotas instáveis, o multiplexador **`tmux`** garante a persistência: a sessão do Claude Code sobrevive a uma queda de SSH e você alterna facilmente entre o painel do agente e a compilação manual. É o detalhe operacional que evita perder uma tarefa de uma hora por causa de uma rede ruim.

## Referências

- Anthropic, *Claude Code — visão geral e instalação* — <https://code.claude.com/docs/en/overview>
- Anthropic, *Claude Code — settings.json* — <https://code.claude.com/docs/en/settings>
- Anthropic, *Claude Code — hooks* — <https://code.claude.com/docs/en/hooks>
- Anthropic, *Skills* (repositório oficial de skills de exemplo) — <https://github.com/anthropics/skills>
- Anthropic, *Referência de comandos* e *CLI* — <https://code.claude.com/docs/en/commands> · <https://code.claude.com/docs/en/cli-reference>
- Kyle (HumanLayer), *Writing a good CLAUDE.md* (2025) — <https://www.humanlayer.dev/blog/writing-a-good-claude-md>

### Resumo do Capítulo 3

Instale com o script oficial (ou `npm`), rode `claude` no projeto, faça login (assinatura ou API) e gere o primeiro contrato com `/init`. Guarde o mapa dos arquivos: pessoal na home (`~/.claude/`, nunca versionado), do time no repositório (`CLAUDE.md` e `.claude/settings.json`, commitados), e o seu-neste-projeto em `.claude/settings.local.json`. O `settings.json` guarda o operacional — modelo, `permissions` (allow/ask/deny), env e hooks; o `CLAUDE.md` guarda as instruções, escrito como mapa enxuto (<~300 linhas) que serve a planejar, desenvolver e conversar. Vigie custo com `/cost` e o Console, e contexto com o indicador de saturação mais `/compact` (mesma tarefa) ou `/clear` (tarefa nova). Use hooks para checagens determinísticas, subagentes e instruções de auditoria para observabilidade, e agendamento para validação contínua. E integre ao VS Code/Cursor, com `tmux` para sessões que não podem cair.

---

## 🚀 Exercícios Práticos do Capítulo 3

1. **Configuração inicial:** Instale o Claude Code, faça login e rode `claude -p "Olá, quem é você?"` para garantir que o ambiente está operante. Em seguida, rode `/status` e `/init` num projeto real e leia o `CLAUDE.md` gerado.
2. **Afinar o `CLAUDE.md`:** Pegue o `CLAUDE.md` do `/init` e reescreva-o no formato "mapa": adicione os comandos de teste/lint, três convenções do projeto e uma seção "Nunca". Mantenha abaixo de 60 linhas.
3. **Permissões sem atrito:** Edite `permissions` (via `/permissions` ou no `settings.json`) para liberar em `allow` os comandos de leitura e teste do seu projeto, e bloquear em `deny` uma ação destrutiva. Observe a diferença na fluidez.
4. **Primeiro hook:** Configure um hook `PostToolUse` que rode o formatador do seu projeto após cada edição. Faça o agente alterar um arquivo e confirme que a formatação foi aplicada sozinha.
5. **Estimativa de custos:** Use o agente para resumir um arquivo Markdown, cheque o consumo com `/cost` e no Console, e tente reduzir 20% dos tokens de entrada refinando o prompt.
