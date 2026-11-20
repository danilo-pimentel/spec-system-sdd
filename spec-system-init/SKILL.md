---
name: spec-system-init
description: Initialize, upgrade, or bootstrap the SPEC-driven documentation system v4 (journal + specctl + hooks + claims) in a workspace. Creates docs/ (RULES.md core + rules/, features/, active/, future/, archive/, discard/, claims/, programs/), scripts/specctl.mjs, Claude Code hooks, CI docs-gate and generated CLAUDE.md/AGENTS.md. Use when the user says "inicializar sistema de documentação", "setup specs", "init doc system", "bootstrap docs", "upgrade specs", "migrar specs v3 para v4", "atualizar sistema de documentação", "bootstrap brownfield", "documentar projeto existente com specs", or is starting a new project that needs disciplined spec-based documentation.
version: v4
---

# spec-system-init v4

Inicializa, atualiza (v3→v4) ou faz bootstrap brownfield do sistema SPEC-driven **v4** no workspace atual. Schema do manifesto: `sdd-4.0` · template_revision: `4.1.0`. Fonte da verdade dos formatos: templates desta skill.

**Regras de segurança desta skill:** NUNCA executar passo destrutivo (sobrescrever, mover, remover arquivo, rodar script que altere estado) sem listar o que será feito e obter confirmação explícita. O MODO UPGRADE **sempre apresenta o plano completo antes de aplicar qualquer mudança**. Nunca commitar nada — o usuário decide.

## O que mudou v3 → v4 (resumo)

1. `state.md` + `memory.md` fundidos em **`journal.md`** (SNAPSHOT sobrescrito nas 60 primeiras linhas + LOG append-only tipado).
2. Os 3 scripts `.sh` morrem; **`scripts/specctl.mjs`** (Node ≥18, single-file, zero deps, cross-platform) faz lint/audit/index/brief/verify/transições/self-test.
3. **Hooks** em `.claude/settings.json`: brief no SessionStart, cápsula por prompt, guards de leitura/escrita, stamp-check, session-close — contexto ENTREGUE, não lido; morre o ritual de Reads + frase de confirmação decorada.
4. Gate de leitura Nível 1 (R.10 v3) removido no default; `policy: strict` no manifesto restaura o comportamento v3.
5. **`docs/claims/`** torna o trabalho em voo visível na main (multi-dev); **`docs/programs/<slug>.md`** (manifesto único) substitui o dual-write R.14.
6. **Porte P|M|G** no main.md roteia a intensidade de processo; artefatos opcionais (prd.md, techspec.md, tasks.md, NN_task.md, qa/review/bugfix, evidence/, intake/) na RAIZ da pasta da SPEC, no molde do harness prd.
7. RULES.md vira **núcleo ≤5KB + docs/rules/** (progressive disclosure); CLAUDE.md/AGENTS.md **gerados** via `specctl entrypoints`, com bloco do projeto preservado entre `<!-- projeto:início -->` / `<!-- projeto:fim -->`.
8. Fechamento exige **digest.md ≤2.000 bytes**; INDEX.md, ARCHIVE-INDEX.md e **PROGRAMS.md** são **gerados** (`specctl index`), nunca editados à mão. **Descoberta do próximo trabalho:** `specctl next` (+ resumo no brief) cruza programas (ordem = grafo) + `docs/ROADMAP.md` (ordem = priorização humana de avulsas) + backlog não-priorizado.
9. Novas regras: **R.15** segredos (`<REDACTED:tipo>` + secret-scan), **R.16** orçamentos por camada lintados, **R.17** trailer `Spec: SPEC-<id>` em commits.
10. **Interop** com o harness prd via manifesto (`interop: none|external|inline`) — a v4 governa (contrato, memória, aceite, ciclo de vida); o pipeline executa.

### v4.1 (delta sobre v4.0 — `template_revision: 4.1.0`)

Princípio: **toda região que um gate valida tem UM escritor — a ferramenta.** O modelo emite significado; o specctl escreve os bytes.

1. **Fechamento transacional** — `specctl close <id>` = stage → valida TUDO → aplica (all-or-nothing, idempotente); `close --dry` lista pendências sem mutar. Substitui o ritual `verify`→`digest`→`archive` e mata o sweep de lint/audit.
2. **Scribe (bookkeeping por comando)** — `specctl log <id> <tipo> "título"` (append no LOG, deny-redirect no Edit direto; a ferramenta anexa commit+diffstat) e `specctl check <id> <n>` (carimba critério de porte P com timestamp+commit; recusa `verify:`). Anti-spoofing: guard nega "verify: exit 0" manual.
3. **digest AUTOMÁTICO no close e no discard** — ≤2.000B gerado e medido pela ferramenta; o agente **nunca conta bytes à mão**. Features e INDEX **não têm teto** (memória viva / enumeração gerada); `specctl rollup <area>` é ferramenta MANUAL/opcional.
4. **Porte P-lite** — critério P sem `| verify:`, fecha por `check`; `commands.test` roda 1× dentro do `close`; sem `evidence/`/`pass^3`. `escalate` (P→M→G, one-way) e `deescalate` (exige citação literal do usuário).
5. **Guards gated por `template_revision ≥ 4.1.0`** — denies novos (LOG, verify-spoof, guard-read de `scripts/specctl.mjs`, secret-scan pré-write, Grep/Glob em archive) só ativam nesse nível; repos v4.0 não quebram.
6. **formats fatiado** — `rules/formats.md` core + `rules/formats-pipeline.md` (só porte G); guardas PARE gated por template nos docs situacionais; `lifecycle.md` ganha "caminho quente" no topo.
7. **Regime de equipe declarado** — `"team": "solo"|"multi"` no manifesto (default solo). Em multi: `activate` imprime a coreografia de publicação do claim (PR trivial), `audit` avisa claim não-publicado na base, cápsula carrega `team=multi`; `rules/team.md` é gated pelo campo, não por inferência.

## Roteamento de modos

| Estado detectado no workspace | Modo |
|---|---|
| Sem `docs/` (ou `docs/` sem sistema de specs) | **INIT** |
| `CLAUDE.md`/`AGENTS.md`/`.claude/settings.json` já existem | **INIT** com sub-modo **MERGE** (nunca sobrescrever cego) |
| `docs/RULES.md` existe E `docs/.spec-system.json` NÃO existe | **UPGRADE v3→v4** |
| `docs/.spec-system.json` existe | Já é v4 → reportar; oferecer reparo (`entrypoints`, `lint`, re-merge de hooks) |
| Código substancial sem features documentadas (pós-init/upgrade) | Oferecer **BROWNFIELD** |

Em TODOS os modos, rodar também a **Detecção de ambiente** (pipeline prd + ECC, seção própria abaixo).

---

## MODO INIT

### Step 1 — Inspecionar estado atual

Detectar com `ls`/Read (try-catch):
- `docs/.spec-system.json` existe? → já é v4. Reportar versão do manifesto e parar (oferecer reparo).
- `docs/RULES.md` sem manifesto? → desviar para **MODO UPGRADE**.
- `docs/` existe com outro conteúdo? → **PARAR**. Perguntar: "Já existe `docs/`. Abortar (A) ou continuar mesclando (M)?" Default: abortar.
- `CLAUDE.md` / `AGENTS.md` no root existem? → marcar para MERGE (Step 7), não sobrescrever.
- `.claude/settings.json` existe? → marcar para merge de hooks (Step 6).
- `.github/workflows/docs-gate.yml` existe? → perguntar antes de sobrescrever.
- Repo git? Branch atual? Sem git: avisar que hooks de branch/claims/R.2 dependem de git e sugerir `git init` (só rodar com confirmação).
- Node ≥18 disponível (`node --version`)? Sem Node: avisar que specctl/hooks/CI não funcionarão e pedir decisão.

### Step 2 — Auto-analisar projeto (NÃO perguntar o que já está documentado)

**ANTES de fazer qualquer pergunta**, ler o que já existe no workspace para inferir valores dos placeholders. O usuário não deve redigitar o que já está documentado.

**Fontes de inferência (tentar nesta ordem, parar ao achar):**

**Nome do projeto (`{{PROJECT_NAME}}`):**
1. `CLAUDE.md` → primeiro heading `# CLAUDE.md — <NAME>` ou `# <NAME>`
2. `README.md` / `readme.md` → primeiro `# heading`
3. `package.json` → `"name"` (se monorepo, usar nome root ou pedir confirmação)
4. `pyproject.toml` → `[project] name` ou `[tool.poetry] name`
5. `Cargo.toml` → `[package] name`
6. `go.mod` → `module <name>` (última parte após `/`)
7. `composer.json` → `"name"`
8. `Gemfile` / `*.gemspec` → padrão específico
9. Fallback: nome da pasta-raiz (basename do cwd)

**Descrição (`{{PROJECT_DESC}}`):**
1. `CLAUDE.md` → primeiro parágrafo após o heading
2. `README.md` → primeiro parágrafo após o heading (pulando badges)
3. `package.json` → `"description"`
4. `pyproject.toml` → `description`
5. `Cargo.toml` → `description`
6. Fallback: `"_(adicionar descrição)_"`

**Stack (`{{STACK}}`):**
Combinar múltiplas fontes:
1. `package.json` → detectar `"type": "module"`, deps principais (react, vue, svelte, next, nuxt, astro, express, fastify, nest, hono, elysia), devDeps (typescript, vite, webpack). Formar string tipo "Node.js + TypeScript + React + Vite".
2. `pyproject.toml` / `requirements.txt` → Python + frameworks (fastapi, flask, django, pydantic)
3. `Cargo.toml` → Rust + frameworks (axum, actix, rocket, tokio)
4. `go.mod` → Go + libs (gin, echo, fiber)
5. `*.csproj` / `*.sln` → .NET
6. `pom.xml` / `build.gradle` → Java/Kotlin + Spring/etc.
7. `Gemfile` → Ruby + Rails/etc.
8. `Dockerfile` / `docker-compose.yml` → adicionar "Docker"
9. Bancos detectáveis em `.env.example`, `docker-compose.yml`, deps (`mysql2`, `pg`, `mongodb`, `mssql`, `@prisma/client`, `typeorm`)
10. Fallback: analisar extensões de arquivo mais frequentes

**Comandos (`{{DEV_COMMANDS}}`):**
1. `package.json` → `"scripts"`. Selecionar relevantes: `install`/`ci`, `dev`/`start`, `build`, `test`, `lint`.
2. `Makefile` → targets comuns (`install`, `dev`, `test`, `build`, `run`)
3. `pyproject.toml` → `[tool.poetry.scripts]` ou comandos comuns
4. `README.md` → seção "Installation" / "Getting started" — extrair blocos ```` ```bash/sh/shell ````
5. `CLAUDE.md` → se tiver seção "Comandos" / "Commands", copiar
6. Fallback: bloco placeholder ```` ```bash\n# (adicionar comandos)\n``` ````

**NOVO v4 — labels mecânicos:** dos comandos detectados, derivar os 5 labels do manifesto `commands: { test, typecheck, lint, dev, e2e }` (string vazia quando não houver). Detectar o gerenciador pelo lockfile (`bun.lock*`→bun, `pnpm-lock.yaml`→pnpm, `yarn.lock`→yarn, `package-lock.json`→npm) e gravar o comando COMPLETO executável (ex.: `pnpm test`, `pnpm tsc --noEmit`). Hooks, CI, `verify:` e skills externas consomem esses labels.

### Step 3 — Apresentar valores detectados e pedir confirmação

Mostrar ao usuário **em UMA mensagem** os valores pré-preenchidos com as fontes, e pedir para confirmar ou editar:

```
Detectei os seguintes valores no workspace:

📦 Nome: <valor>                    (fonte: <arquivo>)
📝 Descrição: <valor>               (fonte: <arquivo>)
🛠️ Stack: <valor>                   (fonte: <arquivo(s)>)
⚡ Comandos (labels do manifesto):
   test=<...> · typecheck=<...> · lint=<...> · dev=<...> · e2e=<...>
                                    (fonte: <arquivo>)
🔒 Policy: standard                 (strict disponível: leitura N1 volta a exigir confirmação, como na v3)
🔗 Interop: <none | proposta — ver Detecção de ambiente>

Confirme com "ok" ou "tudo certo" para usar esses valores, OU
edite os que quiser ajustar (ex.: "stack: Bun + React + Elysia").
```

Se responder "ok" → prossegue. Se editar um campo → aplica e reusa o resto. Se nenhum achou nada → fazer as perguntas clássicas uma a uma, sinalizando.

### Step 4 — Criar estrutura completa (contrato §1)

1. Copiar `templates/scripts/specctl.mjs` → `scripts/specctl.mjs`.
2. Rodar `node scripts/specctl.mjs init` — cria `docs/{features,active,future,archive,discard,claims,programs}/` vazias + `.scratch/`, o manifesto `docs/.spec-system.json` e o `.gitignore` (`docs/active/**/tmp/`, `.scratch/`). Se `.gitignore` já existe, specctl apenas apenda as linhas ausentes — verificar.
3. Escrever os templates no projeto, substituindo placeholders (`{{PROJECT_NAME}}`, `{{PROJECT_DESC}}`, `{{STACK}}`, `{{DEV_COMMANDS}}`, `{{TODAY}}`, `{{NOW}}`):

| Template da skill | Destino |
|---|---|
| `templates/RULES.md` | `docs/RULES.md` |
| `templates/rules/*.md` (formats, lifecycle, context, retrieval, verification, programs, team, interop, ci) | `docs/rules/` |
| `templates/TAXONOMY.md` | `docs/TAXONOMY.md` |
| `templates/ARCHITECTURE.md` | `docs/ARCHITECTURE.md` |
| `templates/CONSTITUTION.md` | `docs/CONSTITUTION.md` |
| `templates/INDEX.md` | `docs/INDEX.md` (stub "gerado"; `specctl index` assume depois) |
| `templates/commands/spec.md` | `.claude/commands/spec.md` |
| `templates/skills/*` (8 skills do pipeline prd + assets) | `.claude/skills/` do projeto (vendor por padrão — repo auto-contido) |
| `templates/agents/task-reviewer.md` | `.claude/agents/task-reviewer.md` |
| `templates/ci/docs-gate.yml` | `.github/workflows/docs-gate.yml` |

Data/hora SEMPRE do relógio real (`node scripts/specctl.mjs stamp`; antes do specctl, `date +"%Y-%m-%d %H:%M"`). NUNCA timestamps do treinamento.

### Step 5 — Gravar manifesto com commands detectados

Atualizar `docs/.spec-system.json` (JSON válido, schema do contrato §2): `installed_at` = NOW real, `policy` confirmada no Step 3, `interop` conforme Detecção de ambiente, `product: null`, `commands` = labels do Step 2. Validar com parse antes de gravar.

### Step 6 — Instalar hooks (merge não-destrutivo)

Fragmento: `templates/hooks-settings.json` (SessionStart→`brief`, UserPromptSubmit→`capsule`, PreToolUse Read→`guard-read`, PreToolUse Write|Edit→`guard-write`, PostToolUse→`stamp-check`, PreCompact→`brief --compact`, Stop→`session-close`).

- `.claude/settings.json` NÃO existe → gravar o fragmento como está.
- Existe → **MERGE**: Read + parse; para cada evento do fragmento, apendar as entradas specctl que ainda não estão lá (idempotência: pular se já houver entrada com `specctl.mjs` e mesmo comando). **NUNCA remover, reordenar ou alterar hooks que não mencionam `specctl.mjs`.** Validar JSON resultante antes de gravar; parse falhou → mostrar o fragmento e pedir merge manual, sem gravar.

### Step 7 — Gerar CLAUDE.md/AGENTS.md via entrypoints (merge mode)

**REGRA DURA:** o bloco imperativo gerado do harness (hard bans TIER-0 + primeira ação) **SEMPRE fica no INÍCIO do `CLAUDE.md`**, logo após o `# <título>` opcional. Antes de qualquer outra seção — descrição do produto, stack, arquitetura, convenções, comandos, etc. Isso garante que a IA, ao ler o arquivo do topo, encontre as instruções imperativas antes de qualquer contexto técnico. Vale igualmente para `AGENTS.md`.

- **Sem CLAUDE.md/AGENTS.md prévios:** rodar `node scripts/specctl.mjs entrypoints`; em seguida resolver quaisquer placeholders restantes com os valores do Step 3, mantendo TODO conteúdo específico do projeto (descrição, stack, comandos) DENTRO do bloco `<!-- projeto:início -->…<!-- projeto:fim -->` — só assim ele sobrevive a regenerações.
- **MERGE (arquivo existe):**
  1. Read do arquivo atual.
  2. Se contiver bloco v3 (`## AVISO IMPERATIVO PRA IA` / `Primeira ação obrigatória`): avisar e perguntar se substitui pelo bloco gerado v4 (recomendado) ou mantém.
  3. Montar o arquivo final: bloco gerado v4 no TOPO + `<!-- projeto:início -->` + conteúdo pré-existente do projeto (sem o bloco v3 substituído) + `<!-- projeto:fim -->`.
  4. Rodar `node scripts/specctl.mjs entrypoints` e conferir que a regeneração preserva o bloco projeto (diff zero).
  **NUNCA appendar as seções imperativas no final** — viola a regra dura de posicionamento.

### Step 8 — Smoke test

Rodar `node scripts/specctl.mjs lint`. Meta em init limpo: **0 erros, 0 warnings**. Qualquer saída ≠ 0/0 → corrigir antes do report (não entregar sistema que já nasce reprovado). Opcional: oferecer `node scripts/specctl.mjs self-test`.

### Step 9 — Report final

Usar o **Report padronizado** (seção abaixo), incluindo a instrução de branch protection.

---

## MODO UPGRADE v3→v4

**Detecção:** `docs/RULES.md` existe E `docs/.spec-system.json` NÃO existe.

### U.1 — Levantar e APRESENTAR O PLANO (obrigatório, antes de qualquer mudança)

Inventariar e listar em UMA mensagem: SPECs por pasta (active/future/archive/discard, com quais têm `state.md`/`memory.md`); os 3 `.sh` a remover (`generate-index.sh`, `lint-docs.sh`, `audit-docs.sh`); seções CUSTOM do `docs/RULES.md` (delta vs template v3 canônico); CLAUDE.md a mesclar; hooks/CI/`/spec` a instalar; SPECs de archive/discard sem digest (digest retroativo); SPECs ativas que ganharão claim + campos novos (Porte/Owner — propor valores); campos R.14 (`Depende de`/`Bloqueia`/`Ordem`) encontrados → manifestos `programs/<slug>.md` propostos; `tasks/prd-*/` existentes (ver Detecção de ambiente). **Aguardar confirmação explícita. Nada é aplicado antes.**

### U.2 — Infra v4 (após confirmação)

Steps 2–7 do INIT (auto-análise reaproveita o CLAUDE.md v3; confirmação em UMA mensagem; specctl + estrutura + manifesto + hooks + docs-gate + `/spec` + `.gitignore` ganha `.scratch/`). No merge do CLAUDE.md, o bloco imperativo v3 é substituído pelo gerado v4 (regra dura de topo).

### U.3 — Fusão state+memory → journal (por SPEC)

Para CADA pasta `SPEC-*` com `state.md`/`memory.md` (em active/, future/, archive/, discard/):
1. Criar `journal.md` no formato do contrato §3.3:
   - **SNAPSHOT** ← `memory.md` (Onde tô/Próximo passo/etc. do TL;DR; **em conflito entre os TL;DRs, o do memory.md vence o do state.md**); `### Fases` ← tabela de fases do state.md; Hipóteses/Respostas-chave/Tentativas que falharam/Arquivos tocados/Onde parei ← seções correspondentes do memory.md.
   - **LOG** ← Log cronológico do state.md, entradas copiadas VERBATIM na ordem original (append-only preservado; headings normalizados para `## YYYY-MM-DD HH:MM — [tipo] Título` apenas quando o tipo já for inequívoco; na dúvida, manter e anotar).
2. Renomear originais NA PRÓPRIA PASTA: `state.md` → `state.md.v3.bak`, `memory.md` → `memory.md.v3.bak`. **NUNCA deletar** (R.5).
3. Fusão ambígua/ilegível → journal mínimo com `[nota] migração v3→v4: revisar *.v3.bak` e listar no report.

### U.4 — RULES fatiado

Substituir `docs/RULES.md` v3 pelo núcleo v4 + `docs/rules/*` (templates). Conteúdo CUSTOM do projeto (regras/seções que não são do template v3 canônico) → `docs/rules/local-overrides.md` com nota de origem e data. Original preservado como `docs/RULES.md.v3.bak`.

### U.5 — Remoção dos 3 .sh

Remover `scripts/generate-index.sh`, `scripts/lint-docs.sh`, `scripts/audit-docs.sh` (já confirmado em U.1 — código do harness, não conteúdo do projeto; substituídos por specctl). Scripts `.sh` alheios ao harness NÃO são tocados.

### U.6 — Digest retroativo

Para cada SPEC em archive/ e discard/ sem `digest.md`: gerar a partir de main+state+memory (≤2.000 bytes, formato §3.4), com a primeira linha após o título marcada **`[digest-auto: revisar]`**. Listar todos no report para revisão humana.

### U.7 — Claims para SPECs ativas

Para cada SPEC em `docs/active/`: criar `docs/claims/SPEC-<id>.md` (formato §5) e gravar no main.md os campos novos confirmados em U.1 (`**Porte:**`, `**Owner:**`; `Features` se ausente). Campos R.14 confirmados viram `docs/programs/<slug>.md` (uma linha por nó; `Bloqueia`/ordem NUNCA são escritos — são derivados).

### U.8 — Fechamento do upgrade

1. `node scripts/specctl.mjs index` (regenera INDEX + ARCHIVE-INDEX).
2. Secret-scan inicial: `node scripts/specctl.mjs lint` cobre docs/ — relatar achados R.15.
3. **Janela de compatibilidade N-1:** o lint aceita SPECs ainda em formato v3 com **warning** (não erro) até a migração completa — informar isso e o objetivo de zerar warnings.
4. Report padronizado (inclui contagem migradas/`[digest-auto]`/pendências + branch protection).

---

## MODO BROWNFIELD

Para projeto com código substancial e docs vazias. Pré-requisito: v4 instalada (INIT ou UPGRADE). Tudo roda sob uma **SPEC formal** — o bootstrap é trabalho como outro qualquer.

1. **Abrir SPEC:** em branch própria (R.2), `node scripts/specctl.mjs new bootstrap-docs --porte M --owner @<user>` (G se o repo for muito grande) + `activate`. Fases no SNAPSHOT: varredura → recorte → geração → revisão.
2. **Varredura (read-only):** mapear diretórios, entrypoints, módulos, rotas, jobs, deps e fronteiras naturais do código.
3. **Recorte confirmado (R.13 em lote):** propor em UMA mensagem **três recortes alternativos** de áreas — **A** enxuto (~4–6 áreas), **B** médio (~7–12), **C** granular (13+) — cada um com nome da área, definição de 1 linha e paths principais. Usuário escolhe/edita → gravar em `docs/TAXONOMY.md` (a escolha É a confirmação R.13 das áreas).
4. **Subagentes — 1 por área, em paralelo:** cada um lê SÓ os paths da sua área e gera `docs/features/<area>.md` no formato do contrato §4, com linha `**Origem:** bootstrap — SPEC-<id> (YYYY-MM-DD HH:MM)`: Keywords, Arquivos principais, Resumo, Estado atual (do código REAL), Decisões arquiteturais ativas (inferidas do código, marcadas como inferidas), Gotchas evidentes; "Specs desta feature" = —. Orçamento 6.000 bytes/feature. Nada inventado: só o que o código evidencia.
5. **ARCHITECTURE.md:** gerar o mapa transversal região→feature (visão que nenhuma feature isolada tem).
6. **Fechar:** `specctl index` → lint → critérios da SPEC (TAXONOMY confirmada; 1 feature/área; ARCHITECTURE gerada; lint 0 erros) → digest → `specctl archive` → PR com docs-gate.

---

## Detecção de ambiente — pipeline prd + ECC

Rodar durante o Step 1 de INIT e U.1 de UPGRADE. A v4 é auto-contida: **nunca modificar o outro harness nem referenciar ECC nos docs gerados.**

- **Pipeline prd — VENDORIZADO POR PADRÃO (v4.1.1):** o harness CARREGA o pipeline em `templates/skills/` (8 skills + assets) e `templates/agents/task-reviewer.md`; o init copia para `.claude/skills/` e `.claude/agents/` DO PROJETO — repo auto-contido, time inteiro na mesma versão. Cópias globais em `~/.claude/skills/`, quando existirem, são executor equivalente e fonte de re-sync — nunca dependência (divergência global×vendorizado = warn no report). `interop` default com pipeline vendorizado: `inline`. Alternativas sob pedido: `external` (skills gravam em `tasks/prd-<slug>/`; `adopt-workspace` no fechamento) · `none` (sem pipeline; templates nativos de rules/formats-pipeline.md).
  - Em UPGRADE com `tasks/prd-*/` existentes: oferecer vínculo `**Workspace:**` nas SPECs correspondentes ou conversão em SPECs novas.
- **ECC** (`~/.claude/rules/ecc/`, agents ECC): apenas **reportar** coexistência. Regras ECC são do usuário/máquina; a v4 não depende delas nem as menciona.

---

## Report padronizado (todos os modos)

```
[spec-system-init v4 · modo: <init|upgrade|brownfield> · NOW=YYYY-MM-DD HH:MM]

Criados:      <paths>
Modificados:  <paths — merges de CLAUDE.md/AGENTS.md/settings.json/.gitignore>
Preservados:  <*.v3.bak> · Removidos: <os 3 .sh — upgrade>
Manifesto:    sdd-4.0 · policy=<...> · interop=<...> · commands={test,typecheck,lint,dev,e2e}
Smoke:        node scripts/specctl.mjs lint → <N erros / N warnings> (meta init: 0/0)
Pendências:   <[digest-auto: revisar] ×N · SPECs v3 em janela compat ×N · achados R.15 · ...>
```

**Próximos passos (incluir sempre):**
1. **Branch protection na `main`** exigindo os checks do workflow `docs-gate` (lint --strict, audit --pr, audit --deps, main-gate, index-check) — sem isso R.2/R.7 não têm dente. GitHub → Settings → Branches → Require status checks.
2. Revisar `docs/TAXONOMY.md` e `docs/CONSTITUTION.md` (stubs) e o bloco projeto do `CLAUDE.md`.
3. Primeiro trabalho: `/spec <demanda>` (abre checkpoint → plan mode → `specctl new` + `activate`).
4. Upgrade: revisar `[digest-auto: revisar]` e zerar warnings da janela compat.
5. Brownfield disponível se houver código sem features documentadas.

---

## Anti-patterns (desta skill)

- ❌ **Sobrescrever** `docs/`, `CLAUDE.md`, `AGENTS.md`, `.claude/settings.json` ou `docs-gate.yml` existentes sem confirmação
- ❌ Aplicar QUALQUER passo do UPGRADE sem o plano U.1 listado e confirmado
- ❌ Usar timestamps do treinamento em vez do relógio real (`specctl stamp` / `date`)
- ❌ **Perguntar info que já está em arquivo existente** — sempre inferir de `package.json`, `README.md`, `CLAUDE.md`, etc. antes
- ❌ Apresentar perguntas em branco ou em várias mensagens — mostrar tudo pré-preenchido em UMA mensagem
- ❌ Commitar arquivos criados (deixar para o usuário decidir)
- ❌ Criar SPECs ou features de exemplo — pastas ficam vazias (exceção: BROWNFIELD cria features REAIS sob SPEC formal)
- ❌ **Posicionar o bloco imperativo gerado em qualquer lugar que NÃO seja o INÍCIO** do `CLAUDE.md`/`AGENTS.md`; em merge mode, **appendar no final** é violação da regra dura (Step 7)
- ❌ Deletar `state.md`/`memory.md`/`RULES.md` v3 no upgrade — renomear `*.v3.bak` (R.5)
- ❌ Editar à mão `INDEX.md`/`ARCHIVE-INDEX.md` (gerados por `specctl index`)
- ❌ Fazer merge de hooks removendo/alterando hooks alheios ao specctl
- ❌ Modificar as skills GLOBAIS (`~/.claude/skills/`) do harness prd ou qualquer coisa da ECC — a evolução do pipeline vive nos `templates/skills/` desta skill (fonte canônica) e chega aos projetos por vendor/upgrade
- ❌ Pular o smoke `specctl lint` ou entregar init com erros/warnings

## Templates location

Ao lado deste SKILL.md: `templates/RULES.md`, `templates/rules/*` (10 arquivos: formats, formats-pipeline, lifecycle, context, retrieval, verification, programs, team, interop, ci), `templates/CLAUDE.md`, `templates/AGENTS.md`, `templates/TAXONOMY.md`, `templates/ARCHITECTURE.md`, `templates/CONSTITUTION.md`, `templates/INDEX.md`, `templates/commands/spec.md`, `templates/skills/*` (pipeline prd vendorizável: cria-prd, cria-techspec, criar-tasks, executar-task, task-review, executar-review, executar-qa, executar-bugfix — com assets), `templates/agents/task-reviewer.md`, `templates/hooks-settings.json`, `templates/ci/docs-gate.yml`, `templates/scripts/specctl.mjs`.
