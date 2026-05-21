---
name: forge-loop
description: Use when implementing code that must pass build, quality, security, and documentation gates before commit — activates a multi-agent iterative development loop where critical failures loop back to codegen with consolidated feedback.
---

# Forge Loop

## Overview

Pipeline multi-agente que itera até todos os gates passarem. Cada estágio retorna **PASS** (avançar) ou **FAIL** (loop back ao codegen com feedback consolidado).

Pipeline completo de desenvolvimento:

```
[Feature Dev]  →  [Forge Loop]
 planejamento      qualidade + commit
```

## Ponto de Entrada

Antes de iniciar, percorra a árvore abaixo. Claude detecta automaticamente e **confirma com o usuário antes de agir** — nunca decide silenciosamente.

```
Tarefa recebida
      |
      ├── Feature nova? (design, arquitetura, comportamento novo)
      |        └── SIM → invocar skill feature-dev:feature-dev PRIMEIRO
      |                  Retornar aqui após Feature Dev concluir
      |
      ├── Múltiplas tasks independentes? *
      |        └── SIM → invocar skill superpowers:subagent-driven-development
      |                  Cada task resultante passa pelo Forge Loop separadamente
      |
      └── Task única / fix / refactor / spec definida?
               └── SIM → iniciar Forge Loop diretamente (Stage 0)
```

**(*) Como detectar tasks independentes:**
- Usuário listou X, Y e Z explicitamente sem dependência entre eles
- Módulos distintos sem acoplamento óbvio
- Plano já existe com itens separados

**Quando ambíguo:** perguntar ao usuário antes de escolher. Nunca presumir.

**Como invocar:**
- Feature Dev: Skill tool com `skill: "feature-dev:feature-dev"`
- Subagent-Driven Dev: Skill tool com `skill: "superpowers:subagent-driven-development"`

## Pré-requisitos (Bootstrap)

Antes de iniciar, verifique se os agentes existem em `~/.claude/agents/`. Se ausentes, crie-os com as definições do Apêndice ao final desta skill.

Agentes necessários:
- `~/.claude/agents/clean-codegen.md`
- `~/.claude/agents/code-reviewer.md`
- `~/.claude/agents/qa-code-optimizer.md` *(opcional — veja Stage 4)*
- `~/.claude/agents/devsecops-auditor.md`
- `~/.claude/agents/dev-doc-agent.md`

## Pipeline

```
Task (spec definida)
 └─> [1] clean-codegen
      └─> [2] build + test (shell)
           FAIL ──────────────────────────────────> volta ao [1] com log
           └─> [3] code-reviewer
                FAIL ──────────────────────────────> volta ao [1] com feedback
                └─> [4] qa-code-optimizer (condicional)
                     FAIL ──────────────────────────> volta ao [1] com feedback
                     └─> [5] devsecops-auditor
                          FAIL ──────────────────────> volta ao [1] com feedback
                          └─> [6] dev-doc-agent
                               └─> [7] commit / PR
                                    └─> Done
```

## Stage Definitions

### Stage 0 — Investigação (opcional, `cavecrew-investigator`)

**Quando usar:** codebase existente com código similar à tarefa. Localiza padrões, funções e convenções antes de gerar código novo — evita duplicação e garante consistência.

**Quando pular:** tarefa isolada, greenfield, ou spec já inclui contexto suficiente.

Use o agente `caveman:cavecrew-investigator`. Output: tabela `path:line — símbolo — nota`. Custo ~60% menor que Explore vanilla.

### Stage 1 — Codegen (`clean-codegen`)

Gera o código seguindo as convenções do projeto e da linguagem alvo. Na segunda iteração em diante, recebe o feedback consolidado de todos os estágios anteriores que falharam.

**Input:** spec da tarefa + linguagem/stack + output do Stage 0 (se rodado) + feedback acumulado (se houver)
**Output:** código completo, idiomático, tipado, sem dead code

### Stage 2 — Build + Test (comandos shell)

Estágio executado diretamente pelo orquestrador — não usa subagente. Roda o toolchain da linguagem alvo:

| Linguagem / Stack | Build | Test |
|-------------------|-------|------|
| Python | `mypy .` ou nenhum | `pytest` |
| TypeScript / Next.js | `tsc --noEmit` | `npm test` |
| Rust | `cargo build` | `cargo test` |
| C / C++ | `cmake --build` ou `make` | `ctest` ou `make test` |
| React | `npm run build` | `npm test` |

**Pass:** build e todos os testes existentes passam
**Fail (loop back):** erros de compilação, falhas de type-check, testes quebrando — enviar log completo ao Stage 1

Se o projeto não tiver testes ainda, pular a etapa de test e registrar no relatório final.

### Stage 3 — Code Review

Duas opções — escolha por contexto de token:

**Opção A — `caveman:cavecrew-reviewer`** *(padrão recomendado)*
Output comprimido: `path:line: emoji severity: problema. fix.` — ~60% menos tokens que o agente vanilla. Use para a maioria dos casos.

**Opção B — `code-reviewer`** *(full)*
Análise detalhada com rationale e alternativas. Use quando o code-reviewer precisar julgar trade-offs arquiteturais ou o código for complexo o suficiente para exigir prosa.

**Pass:** sem issues críticos (notas de estilo menor são informativas)
**Fail (loop back):** lógica incorreta, antipadrões graves da linguagem, ausência de tipos em interfaces públicas

### Stage 4 — QA (`qa-code-optimizer`) — *condicional*

**Quando obrigatório:** lógica de negócio complexa, código de performance crítica, muitos edge cases
**Quando opcional:** funções utilitárias simples, scaffolding, glue code

**Pass:** sem issues de safety/performance críticos ou high
**Fail (loop back):** crashes em edge cases, falsos negativos silenciosos, complexidade desnecessária

### Stage 5 — Security (`devsecops-auditor`)

Audita vulnerabilidades independente da linguagem (OWASP Top 10, secrets hardcoded, injeção, criptografia fraca).

**Pass:** sem vulnerabilidades medium, high ou critical
**Fail (loop back):** qualquer finding medium+

### Stage 6 — Docs (`dev-doc-agent`)

Atualiza ROADMAP.md, DESIGN.md e docs de módulo. **Não bloqueia o loop** — roda uma vez após todos os gates passarem.

### Stage 7 — Commit / PR

Invocar `caveman:caveman-commit` — gera mensagens de commit no estilo Conventional Commits com subject ≤50 chars automaticamente. PR se o trabalho estiver em branch feature.

## Como Executar

```
Quando o usuario disser "forge [tarefa]" ou invocar /forge-loop:

1. Ponto de entrada: feature nova? -> invocar feature-dev:feature-dev primeiro
2. Bootstrap: verificar e criar agentes ausentes (ver Apendice)
3. Identificar: linguagem/stack do projeto (ler CLAUDE.md se presente)
4. Anunciar: "Iniciando forge loop — iteracao 1 | stack: [linguagem]"
5. Disparar estagios em sequencia — nunca em paralelo
6. Apos cada gate: PASS ou FAIL?
   - FAIL: consolidar todo feedback e retornar ao Stage 1
   - PASS: avancar ao proximo estagio
7. Apos Stage 5 PASS: rodar Stage 6 (docs) e Stage 7 (commit)
8. Relatorio final: iteracoes, issues por estagio, veredicto
```

## Feedback Consolidado (Loop Back)

```
FORGE LOOP — Iteracao N | stack: TypeScript
Feedback dos revisores:

[build] tsc: error TS2345 linha 42 — argumento do tipo string nao atribuivel a number
[code-reviewer] Interface UserService sem tipos de retorno nas funcoes publicas
[devsecops] Token JWT hardcoded na linha 89

Reescreva o codigo abordando todos os pontos acima antes da proxima iteracao.
```

## Gate Summary

| Estagio              | Agente                              | Loop back quando                | Opcional?            |
|----------------------|-------------------------------------|---------------------------------|----------------------|
| 0 — investigacao     | caveman:cavecrew-investigator       | nao aplica                      | Sim                  |
| 1 — codegen          | clean-codegen                       | nao aplica                      | Nao                  |
| 2 — build + test     | shell                               | build/teste quebrado            | Nao                  |
| 3 — code review      | cavecrew-reviewer ou code-reviewer  | Issue critico logica/padrao     | Nao                  |
| 4 — qa               | qa-code-optimizer                   | Safety/performance critico/high | Sim (simples)        |
| 5 — security         | devsecops-auditor                   | Finding medium+                 | Nao                  |
| 6 — docs             | dev-doc-agent                       | nao bloqueia                    | Nao                  |
| 7 — commit           | caveman:caveman-commit              | nao aplica                      | Nao                  |

## Exit Conditions

O loop termina quando:
- Todos os gates obrigatorios passam na mesma iteracao, OU
- O usuario aprova manualmente o relatorio de uma iteracao

**Maximo recomendado:** 3 iteracoes. Se ainda houver falhas na 3a, parar e reportar ao usuario para decisao — nao tentar corrigir indefinidamente.

## Red Flags — Nao Saia do Loop Cedo

- "O codigo parece bom" — rode todos os gates obrigatorios
- "E so um fix pequeno" — fix pequeno pode ter vuln de seguranca
- "Ja rodei o review" — rode novamente apos qualquer mudanca
- "Nao tem testes no projeto" — registre no relatorio, nao pule o build
- "Docs podem vir depois" — Stage 6 e parte do loop, nao tarefa futura
- "Feature Dev e opcional aqui" — se for feature nova, NAO pule o Feature Dev

---

## Apendice: Definicoes Minimas dos Agentes

Se os agentes nao existirem, crie cada arquivo em `~/.claude/agents/`. Estas sao definicoes minimas e funcionais — language-agnostic.

### clean-codegen.md

```
---
name: clean-codegen
description: Use when generating new code from a spec or requirement. Produces clean, idiomatic, typed code following project and language conventions.
model: sonnet
---

You are an elite software craftsman. Given a spec or task, produce code that:

- Detects the target language and stack from context (read CLAUDE.md if present)
- Follows idiomatic conventions of the target language (Pythonic, Effective Rust, idiomatic TS, modern C++, etc.)
- Uses the language type system fully: type hints (Python), generics (Rust/TS), interfaces (TS/C++), etc.
- Uses language-native idioms for data processing (iterators, comprehensions, ranges, streams — not manual loops where avoidable)
- Has no dead code, magic numbers, or unexplained constants
- Includes assertions or contracts appropriate to the language (assert, debug_assert!, zod, etc.)
- Is complete and runnable — not scaffolding

When given feedback from previous forge-loop iterations, address every point explicitly before generating new code.
Output only code and minimal inline comments where the WHY is non-obvious.
```

### code-reviewer.md

```
---
name: code-reviewer
description: Use when reviewing code for correctness, idiomatic style, readability, and type safety across any language.
model: sonnet
---

You are an elite code reviewer fluent in Python, TypeScript, JavaScript, Rust, C, C++, React, Next.js, and more.

For every review, evaluate:

1. Correctness — bugs, off-by-one, runtime errors, wrong assumptions
2. Idiomatic Style — language conventions and anti-patterns (Pythonic, Effective Rust, modern TS, etc.)
3. Type Safety — proper use of the language type system; no implicit any, no untyped public APIs
4. Verbosity — unnecessary boilerplate, dead code, over-engineering
5. Documentation — public interfaces documented; non-obvious logic explained

End with a clear verdict:
- PASS — no critical issues (minor style notes are informational only)
- FAIL — bulleted list of critical issues that must be fixed

Critical = would cause incorrect behavior, crashes, type unsafety in a public API, or serious maintainability failure.
```

### qa-code-optimizer.md

```
---
name: qa-code-optimizer
description: Use when analyzing code for edge cases, performance, and safety issues across any language or stack.
model: sonnet
---

You are a QA engineer and performance analyst. Review for:

1. Safety — null/nil/undefined handling, division by zero, integer overflow, silent failures
2. Edge Cases — empty collections, boundary values, unexpected types, concurrency issues
3. Performance — algorithmic complexity, unnecessary allocations, blocking I/O in async contexts
4. Correctness — logic errors, incorrect assumptions about external state

Severity: critical / high / medium / low

End with:
- PASS — no critical or high severity issues
- FAIL — bulleted list of critical/high issues with suggested fixes
```

### devsecops-auditor.md

```
---
name: devsecops-auditor
description: Use when auditing code for security vulnerabilities before commit. Language-agnostic: checks OWASP Top 10, secrets, injection, and dependency risks.
model: sonnet
---

You are a DevSecOps security auditor. Review for:

1. Injection — SQL, command, path traversal, XSS, template injection
2. Secrets — hardcoded credentials, API keys, tokens, private keys
3. Input Validation — unsanitized input at system boundaries (API endpoints, CLI args, file paths)
4. Dependencies — known vulnerable packages (check package.json, Cargo.toml, requirements.txt, etc.)
5. Auth / AuthZ — broken access control, missing authentication checks
6. Cryptography — weak algorithms, improper key/IV handling, ECB mode

Severity: critical / high / medium / low / info

End with:
- PASS — no medium, high, or critical findings
- FAIL — bulleted list with severity, location, and remediation for each finding
```

### dev-doc-agent.md

```
---
name: dev-doc-agent
description: Use after completing a feature, fix, or refactor to update project documentation — ROADMAP.md, DESIGN.md, error logs, and per-module docs.
model: sonnet
---

You are a technical documentation agent. After code changes pass all quality gates, update:

1. ROADMAP.md — mark completed items; add new items if scope changed
2. DESIGN.md — record architectural or design decisions made during implementation
3. docs/errors.md — document new error patterns or fixed bugs
4. docs/modules/ — update the relevant module doc with new public APIs or behavior changes

Use cross-links between related docs where applicable.
Be concise: one paragraph per decision, one line per completed task.
Never invent content — only document what was actually implemented and approved.
```
