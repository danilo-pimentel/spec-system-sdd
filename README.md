# Spec System SDD

O Spec System SDD é um kit de *Spec-Driven Development* para executar trabalho de software com IA a partir de especificações versionadas no mesmo repositório do código. **Danilo Pimentel** desenvolveu o núcleo do kit, incluindo o protocolo de ciclo de vida das SPECs, os mecanismos de verificação e, a partir da v4, a CLI `specctl.mjs`.

Cada SPEC define o resultado esperado, os limites e os critérios de aceite. O agente planeja, implementa, testa, registra evidências e retoma o trabalho a partir desses arquivos. A pessoa responsável aprova o contrato, as mudanças de escopo e o encerramento.

O Claude Code fornece a sessão, as ferramentas e as skills. O kit adiciona as regras de trabalho, a memória entre sessões, os comandos de ciclo de vida e as verificações reproduzíveis. `scripts/specctl.mjs` é uma CLI Node.js sem dependências npm. Ela prepara contexto, controla transições e executa gates. A sessão do Claude conduz a implementação e escolhe o próximo passo de acordo com a SPEC e o journal.

## Versões

| Branch | Conteúdo |
| --- | --- |
| `main` e [`v4.1`](https://github.com/danilo-pimentel/spec-system-sdd/tree/v4.1) | Versão atual com fechamento via `close`, registro via `log` e `check`, hooks adicionais e as oito skills do pipeline no pacote. |
| [`v4`](https://github.com/danilo-pimentel/spec-system-sdd/tree/v4) | Versão 4 original, com `specctl.mjs`, journal e hooks. |
| [`v3`](https://github.com/danilo-pimentel/spec-system-sdd/tree/v3) | Versão 3, com `state.md`, `memory.md` e scripts de manutenção em shell. |

A linha 4.1 usa o schema de manifesto `sdd-4.0`. A CLI do pacote reporta `4.1.1` e usa esse valor como `template_revision` ao criar o manifesto. A abertura de `SKILL.md` conserva a referência histórica a `4.1.0`.

## Instalação no Claude Code

### 1. Prepare o terminal

Você precisa de Git, Node.js 18 ou superior e acesso ao Claude Code pela sua conta ou pelo provedor configurado pela sua organização. O `node` e o `git` precisam funcionar no ambiente em que o Claude chama os hooks. No Windows, o Git for Windows também disponibiliza o Git Bash.

Se precisar instalar a CLI do Claude, use um dos comandos oficiais. No PowerShell, use `winget install Anthropic.ClaudeCode` ou o instalador nativo abaixo.

```powershell
irm https://claude.ai/install.ps1 | iex
```

No macOS, Linux ou WSL, use o instalador nativo para shell.

```bash
curl -fsSL https://claude.ai/install.sh | bash
```

Abra um novo terminal, se necessário, e confira os executáveis. Se `claude` ou `node` não for reconhecido, corrija o `PATH` antes de continuar. No primeiro `claude`, siga o fluxo de login. Use `/login` no Claude Code se precisar autenticar novamente. [Guia oficial de início rápido do Claude Code](https://code.claude.com/docs/en/quickstart) · [Solução de problemas com a instalação e o PATH](https://code.claude.com/docs/en/troubleshoot-install#fix-your-path)

```text
git --version
node --version
claude --version
```

### 2. Coloque a skill no projeto de destino

Clone este repositório ao lado do projeto que receberá o kit. Para clonar enquanto o repositório estiver privado, autentique o Git em uma conta com acesso de leitura. Troque `meu-projeto` pelo caminho do seu projeto. Se estiver começando em uma pasta vazia, crie a pasta e inicialize seu repositório nela com `git init -b main`. Prepare um commit inicial antes de abrir a primeira SPEC, para que as verificações possam registrar o commit de referência.

No PowerShell, partindo da pasta que contém `meu-projeto`:

```powershell
git clone --branch main https://github.com/danilo-pimentel/spec-system-sdd.git
if ($LASTEXITCODE -ne 0) { throw 'O clone do kit falhou.' }
Set-Location -LiteralPath ./meu-projeto -ErrorAction Stop
$skillDir = '.claude/skills/spec-system-init'
if (Test-Path -LiteralPath $skillDir) {
    throw "Já existe $skillDir. Compare a versão instalada antes de substituí-la."
}
New-Item -ItemType Directory -Force -Path .claude/skills | Out-Null
Copy-Item -LiteralPath ../spec-system-sdd/spec-system-init -Destination $skillDir -Recurse -ErrorAction Stop
claude
```

No macOS, Linux ou WSL, a partir da pasta que contém `meu-projeto`:

```bash
if git clone --branch main https://github.com/danilo-pimentel/spec-system-sdd.git && cd ./meu-projeto; then
  skill_dir=.claude/skills/spec-system-init
  if [ -e "$skill_dir" ]; then
    printf '%s\n' "Já existe $skill_dir. Compare a versão instalada antes de substituí-la."
  else
    mkdir -p .claude/skills &&
      cp -R ../spec-system-sdd/spec-system-init "$skill_dir" &&
      claude
  fi
fi
```

O Claude Code carrega a skill do projeto em `.claude/skills/spec-system-init/SKILL.md`. Copie a pasta inteira para manter `templates/` ao lado de `SKILL.md`. Para disponibilizar o inicializador em todos os seus projetos locais, coloque essa mesma pasta em `~/.claude/skills/` (`%USERPROFILE%\.claude\skills\` no Windows). Se houver cópias com o mesmo nome, confira a origem selecionada pelo Claude com `/skills`. [Escopos de skills do Claude Code](https://code.claude.com/docs/en/skills#choose-where-skills-load)

### 3. Inicialize o sistema

No prompt do Claude Code, aberto na raiz do projeto de destino, rode o inicializador do kit.

```text
/spec-system-init
```

O inicializador examina os arquivos do projeto, propõe nome, descrição, stack e os comandos `test`, `typecheck`, `lint`, `dev` e `e2e`. Revise esses valores e confirme o plano. Os comandos ficam no manifesto e devem refletir o que de fato roda na raiz do seu projeto. Deixe vazio um comando que o projeto ainda não oferece.

| Estado do projeto | Fluxo do inicializador |
| --- | --- |
| Projeto novo | Cria a estrutura documental e instala a CLI, os hooks, `/spec`, as skills do pipeline e o template de CI. |
| Projeto existente com `docs/`, `CLAUDE.md`, `AGENTS.md` ou configurações do Claude | Pede confirmação para mesclar `docs/` existente, preserva as instruções específicas do projeto nos blocos previstos e integra os hooks ao JSON já existente. Confira o diff das configurações antes de compartilhar o setup. |
| Versão 3 instalada | Apresenta o plano de migração. Após aprovação, transfere o estado e o histórico para `journal.md`, guarda os arquivos antigos em `*.v3.bak` e propõe os campos novos das SPECs. |
| Manifesto v4 já presente | Reporta a versão instalada e oferece diagnóstico ou reparo. |
| Código existente com pouca documentação | Pode propor um bootstrap *brownfield* para documentar áreas reais do projeto sob uma SPEC. |

`standard` é a política inicial. `strict` acrescenta a confirmação de leitura de contexto de nível 1 prevista nas regras. O manifesto também declara `team: solo|multi`. Em `multi`, a ativação orienta a publicação do claim de trabalho na branch base para dar visibilidade ao time.

Depois de revisar `docs/TAXONOMY.md`, `docs/CONSTITUTION.md`, `docs/.spec-system.json` e o bloco do projeto em `CLAUDE.md`, execute na raiz do projeto:

```sh
node scripts/specctl.mjs --version
node scripts/specctl.mjs index
node scripts/specctl.mjs lint
node scripts/specctl.mjs brief
```

`index` gera os índices definitivos a partir dos arquivos do projeto, inclusive quando a instalação acabou de copiar o modelo de `docs/INDEX.md`. O resultado esperado de `lint` em uma instalação limpa é zero erros e avisos. `node scripts/specctl.mjs self-test` roda os testes internos da CLI. Reabra o Claude Code nessa raiz para carregar a configuração e experimente `/spec status`. Use `/skills` para conferir as skills e `/hooks` para inspecionar a configuração dos hooks. [Referência de hooks do Claude Code](https://code.claude.com/docs/en/hooks)

Se o projeto usa GitHub Actions, revise o workflow gerado em `.github/workflows/docs-gate.yml`. O job de PR executa lint estrito, auditoria do diff e das dependências, valida `docs/active/` nos PRs para `main` e verifica se os índices estão atualizados. Para exigir esse resultado no merge, configure o job como check obrigatório na proteção da branch principal. O template inclui uma auditoria semanal e um job de regeneração de índices após push em `main`.

## Primeiro ciclo de trabalho

Crie uma branch de trabalho a partir da base do seu projeto e abra uma sessão do Claude Code na raiz. Em branches protegidas, o kit admite criar rascunhos em `future/`. A ativação ocorre na branch de trabalho.

```sh
git switch -c feat/filtro-historico
claude
```

```text
/spec Adicionar filtro por período ao histórico de pedidos e provar o resultado com testes.
```

1. **Defina a entrega.** O Claude propõe a área da taxonomia, o porte e o responsável no mesmo checkpoint. Revise a minuta de `main.md` no modo de planejamento. Aprovar esse plano valida o contrato.
2. **Ative a SPEC.** A sessão usa `specctl new` e `activate` para criar o ID, registrar a ativação e abrir o claim. O trabalho ativo fica em `docs/active/SPEC-.../` na branch de trabalho.
3. **Repita implementação, validação e registro.** Avance por critério ou tarefa, rode as verificações, corrija falhas e registre resultados no journal na hora. O `SNAPSHOT` contém fase, estado e próximo passo. O `LOG` guarda eventos em ordem, com novas entradas via `specctl log`. Para retomar em outra sessão, indique a SPEC e siga o `brief` e o journal.
4. **Valide e encerre.** Use `/spec fechar` para revisar as pendências de `close <id> --dry`. Em porte P, `check <id> <n>` registra evidência de um critério sem comando `verify:`. Quando houver `verify:`, `specctl verify <id>` roda o comando e carimba os critérios que passaram. Pendências de aceite exigem decisão explícita do usuário por item e registro da resposta.
5. **Arquive com evidência.** O `close` executa as verificações pendentes e a suíte `commands.test` quando configurada, prepara o `digest.md`, arquiva a SPEC, remove o claim e regenera os índices. Se uma verificação ou a suíte falhar, a SPEC fica ativa. Critérios já verificados conservam a evidência para a próxima tentativa. Ao arquivar, o comando imprime um atestado e os próximos passos de Git para decisão do usuário. Os commits feitos sob a SPEC levam o trailer `Spec: SPEC-<id>`.

Outros comandos no prompt do Claude incluem `/spec status`, `/spec pausar`, `/spec retomar` e `/spec importar <artefato>`. A importação preserva o original em `intake/`, normaliza o material após confirmação e retoma pela fase pendente a partir do contrato local aprovado.

## Porte e escolha automática das skills

A sessão estima o porte pelo volume de arquivos, pelas áreas afetadas e pelo número de critérios. Ela apresenta a proposta no checkpoint e o usuário confirma. O porte orienta o processo conforme a tabela.

| Porte | Processo | Uso do pipeline |
| --- | --- | --- |
| **P — pontual** | Ajuste pequeno com objetivo, um a três critérios e journal. | Fluxo direto do SDD, com `check` para critérios sem `verify:` e suíte do projeto no fechamento quando configurada. |
| **M — médio** | `main.md` completo atua como mini-PRD. O journal acompanha as fases e os critérios têm `verify:` quando aplicável. | Fluxo direto do SDD, sem PRD ou TechSpec separados. Se a tarefa crescer, a sessão registra a decisão e pode escalonar a SPEC. |
| **G — grande** | Trabalho que abrange várias áreas ou sessões e pede uma escolha explícita antes de redigir `main.md`. | A sessão recomenda **completo**, **só TechSpec**, **só tasks** ou **sem pipeline**, conforme o trabalho. O usuário confirma o caminho e, se houver pipeline, o modo de execução. |

Para G, o fluxo completo é indicado quando requisitos, implementação e QA precisam de fases próprias. Só TechSpec faz sentido quando a principal dificuldade é arquitetural. Só tasks serve quando contrato e design já estão claros. Em uma execução parcial, o `main.md` pode suprir a exigência de um PRD. A skill `criar-tasks` exige `techspec.md` no workspace. Para usar essa skill no caminho só tasks, reaproveite uma TechSpec existente. Se o design está registrado apenas no contrato, use o formato nativo de tarefas do SDD. O modo `sem pipeline` usa o fluxo do SDD e os artefatos nativos adequados.

- **Autônomo.** Depois das escolhas iniciais, a sessão registra planos e listas no journal e avança pelas fases autorizadas, respeitando as permissões do Claude Code.
- **Com gates.** A sessão espera aprovação após o PRD, após a TechSpec, após a lista de tasks e antes de cada execução de task. Esse modo é o padrão quando não há escolha registrada.

Em ambos os modos, mudanças nos critérios ou no escopo e o encerramento dependem de decisão humana. O agente registra a escolha na ativação ou como decisão no journal e usa a fase pendente para chamar a skill correspondente. Os hooks fornecem o contexto e o workspace. O `specctl` executa comandos e valida artefatos. A execução das skills ocorre dentro da sessão do Claude.

### Skills incorporadas na v4.1

A v4.1 incorpora as oito skills listadas abaixo com crédito a **Pablo Hadler**, que autorizou sua publicação no kit. [Pablo Hadler no GitHub (`@pmhg`)](https://github.com/pmhg) · [Pablo Hadler no LinkedIn](https://www.linkedin.com/in/pablo-hadler-650a5223/)

A v4.1 inclui também os templates e as referências dessas skills e um subagente `task-reviewer` para revisão por tarefa.

| Skill | Quando a sessão a usa | Resultado |
| --- | --- | --- |
| [`/cria-prd`](spec-system-init/templates/skills/cria-prd/SKILL.md) | Requisitos do pipeline completo. | `prd.md` com requisitos funcionais numerados `RF-N`. |
| [`/cria-techspec`](spec-system-init/templates/skills/cria-techspec/SKILL.md) | Projeto técnico no fluxo completo ou quando a arquitetura justifica essa fase isolada. | `techspec.md` com desenho, decisões, riscos e estratégia de testes. |
| [`/criar-tasks`](spec-system-init/templates/skills/criar-tasks/SKILL.md) | Decomposição de requisitos e desenho em entregas com testes. | `tasks.md` e arquivos `NN_task.md`. |
| [`/executar-task`](spec-system-init/templates/skills/executar-task/SKILL.md) | Execução de uma tarefa identificada da lista. | Implementação, execução de testes e acionamento da revisão por tarefa. |
| [`/task-review`](spec-system-init/templates/skills/task-review/SKILL.md) | Revisão de uma tarefa executada, normalmente via `task-reviewer` quando disponível. | Artefato `NN_task_review.md` com achados e veredito. |
| [`/executar-review`](spec-system-init/templates/skills/executar-review/SKILL.md) | Revisão do conjunto da implementação, incluindo o diff da branch e mudanças de trabalho no escopo. | `review-report.md`. |
| [`/executar-qa`](spec-system-init/templates/skills/executar-qa/SKILL.md) | Validação funcional, visual e de acessibilidade quando o plano exigir QA. | `qa-report.md`, evidências e `bugs.md` com os problemas encontrados. |
| [`/executar-bugfix`](spec-system-init/templates/skills/executar-bugfix/SKILL.md) | Bugs registrados no QA que precisam de correção. | Correções, testes de regressão, atualização de `bugs.md` e `bugfix-report.md`. |

`executar-task` exige revisão antes de marcar a tarefa concluída. O agente corrige e repete a revisão em até três ciclos, depois escala os achados persistentes. No QA, um bug corrigido volta a ser testado antes de ser marcado `Verificado`. A skill de QA usa Playwright MCP quando disponível. Quando faltar essa integração, o fluxo prevê registrar a limitação e sugerir a suíte E2E do projeto ou verificação manual. A TechSpec pode usar Context7 MCP e recorrer à documentação técnica na web. O setup do kit copia as skills e o subagente para o projeto. Ferramentas MCP são configuradas à parte. [Configuração de MCP no Claude Code](https://code.claude.com/docs/en/mcp)

### Onde o pipeline grava os artefatos

| `interop` no manifesto | Comportamento |
| --- | --- |
| `inline` | Padrão do inicializador com a cópia local das skills. Com uma SPEC ativa, a cápsula informa o caminho `docs/active/SPEC-.../` para todas as leituras e escritas do pipeline. Seus artefatos acompanham a SPEC no arquivamento. |
| `external` | Usa o workspace declarado `tasks/prd-<slug>/`. Antes do fechamento, `specctl adopt-workspace <id>` copia os artefatos para a SPEC e preserva o workspace original. |
| `none` | Usa o fluxo SDD sem executar o pipeline integrado. |

As skills aplicam as regras de interop quando encontram **ambos** `docs/.spec-system.json` e `docs/RULES.md`. Nesse contexto, elas usam os comandos do manifesto, consultam o contrato e o journal antes de pedir esclarecimentos e devolvem decisões e resultados ao journal. Cada artefato mantém seu próprio nível de progresso. Uma tarefa concluída fica em `tasks.md`, a fase no `SNAPSHOT` e o aceite em `main.md`.

## Arquivos, hooks e verificações

| Caminho no projeto instalado | Função |
| --- | --- |
| `docs/.spec-system.json` | Schema, revisão, política, equipe, interop e comandos do projeto. |
| `CLAUDE.md`, `AGENTS.md`, `docs/RULES.md` e `docs/rules/` | Pontos de entrada, regras centrais e guias situacionais. Preserve conteúdo do projeto no bloco delimitado dos entrypoints gerados. |
| `docs/{future,active,archive,discard}/` | Estados do ciclo de vida. Cada SPEC ativa tem `main.md` e `journal.md`. A arquivada tem o resumo `digest.md`. |
| `docs/features/`, `docs/TAXONOMY.md`, `docs/ARCHITECTURE.md`, `docs/CONSTITUTION.md` | Contexto duradouro por área, vocabulário, mapa da arquitetura e princípios do projeto. |
| `docs/claims/`, `docs/programs/`, `docs/ROADMAP.md` | Trabalho em andamento, dependências entre SPECs e priorização humana das SPECs avulsas. |
| `docs/INDEX.md`, `docs/ARCHIVE-INDEX.md`, `docs/PROGRAMS.md`, `docs/DEFERRED.md` | Índices regenerados por `specctl index`. |
| `scripts/specctl.mjs`, `.claude/settings.json`, `.claude/commands/spec.md` | CLI, configuração dos hooks e comando interativo `/spec`. |

No início ou na retomada, `brief` informa a SPEC ativa, o estado do trabalho e o que ler em seguida. Em cada prompt, `capsule` entrega data, SPEC ativa e modo de workspace. Os hooks de leitura e escrita limitam operações previstas pelo protocolo. Os hooks após edições verificam carimbos e padrões de segredo. Há também verificações na compactação e ao encerrar uma resposta. Esses controles cobrem as ferramentas e os eventos configurados em `.claude/settings.json`. `lint`, `audit` e o workflow de CI verificam o conteúdo versionado.

```sh
node scripts/specctl.mjs lint --strict
node scripts/specctl.mjs audit
node scripts/specctl.mjs audit --deps
node scripts/specctl.mjs next
```

Para outras CLIs, `AGENTS.md` orienta executar `brief` na abertura da sessão e `lint` e `audit` antes do PR. Use frases como `inicie uma spec: <demanda>` ou `feche a spec` quando `/spec` não estiver disponível. O mecanismo de verificação continua acessível pela CLI Node e pelo CI.

## Referência do pacote

- [Skill de instalação, merge, migração e brownfield](spec-system-init/SKILL.md)
- [Ciclo de vida, aprovação e fechamento](spec-system-init/templates/rules/lifecycle.md)
- [Regras de interop](spec-system-init/templates/rules/interop.md)
- [Comando `/spec`](spec-system-init/templates/commands/spec.md)
- [CLI `specctl.mjs`](spec-system-init/templates/scripts/specctl.mjs)
