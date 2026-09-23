# qa-md-gen

Gerador de **checklist de QA vivo** para qualquer aplicação. A partir dos endpoints e fluxos de um repositório, produz um **HTML interativo autocontido** onde o QA marca cada funcionalidade como **✓ Passou** ou **✗ Falhou**, descreve o erro quando falha, e **exporta um prompt `.md` de correção** pronto para alimentar um agente de código.

> Nasceu do checklist de QA de um SaaS de geração de conteúdo e foi generalizado para servir qualquer stack (serverless, FastAPI, Express, Django, Rails, Spring, Go…).

**Este repositório É uma Claude Code skill** (`SKILL.md` na raiz): instale e invoque com `/qa-md-gen`. Também funciona como prompt copiável para qualquer outro agente (`PROMPT.md`).

## O que tem aqui

| Arquivo | Papel |
|---|---|
| `SKILL.md` | A **skill** (frontmatter `name`/`description` + instruções). Torna o gerador invocável via `/qa-md-gen`. |
| `templates/checklist.template.html` | O **template** da UI (CSS + lógica genéricos) com placeholders `{{PROJECT_NAME}}`, `{{PROJECT_SLUG}}`, `{{DATA}}`. |
| `PROMPT.md` | Fallback: o mesmo pipeline como **prompt copiável** para agentes que não carregam skills. |

## Instalar como skill

O diretório do repo já tem o formato de skill (`SKILL.md` + `templates/`). Basta colocá-lo onde o Claude Code procura skills:

```bash
# user-level (todos os projetos):
ln -s "$(pwd)" ~/.claude/skills/qa-md-gen        # ou: cp -r . ~/.claude/skills/qa-md-gen

# project-level (só um repo):
cp -r . <repo-alvo>/.claude/skills/qa-md-gen
```

Depois, dentro de um repositório:

```
/qa-md-gen                     # gera o checklist do repo atual
/qa-md-gen backend/            # escopo específico
/qa-md-gen --out docs/qa       # define a pasta de saída
```

> **Symlink vs cópia:** o symlink mantém uma fonte única (editar o repo atualiza a skill). A cópia duplica — se editar o template, sincronize as duas.

## Usar como prompt (sem instalar)

1. Abra o repositório-alvo num agente de código.
2. Cole a seção **PROMPT** de [`PROMPT.md`](./PROMPT.md).
3. O agente lê endpoints/workers/rotas, monta o `DATA`, preenche o template e salva `qa/qa-checklist.html` — e fixa a regra de "checklist vivo".
4. Abra o HTML no navegador e rode o QA.

## O ciclo de QA

```
gerar checklist  →  QA marca ✓/✗ e descreve os erros  →  "Exportar erros (.md)"
      ↑                                                          │
      └──────  revalidar itens no checklist  ←  agente corrige ──┘
```

## Recursos do checklist gerado

- Um único HTML, sem dependências externas — abre em qualquer navegador.
- Estados **Passou / Falhou** por item, com **caixa de descrição do erro** ao falhar.
- Badges de método HTTP (GET/POST/PATCH/PUT/DELETE) + `FLOW` para workers/crons/UI.
- Tag `crit` para itens de **risco alto ao cliente**.
- Progresso e descrições persistidos em `localStorage` (por navegador).
- Barras de progresso por domínio e anel geral (verde=passou / vermelho=falhou).
- **Exportar erros (.md) / Copiar prompt**: só as falhas + descrições, formatadas como prompt de correção (críticos primeiro, RED→GREEN, revalidar checklist).
- Tema claro/escuro e responsivo.

## Personalização

- Ajuste `templates/checklist.template.html` para mudar UI/UX — muda para todos os projetos gerados a partir dele.
- Idioma padrão: PT-BR. Traduza os rótulos fixos do template para gerar em outro idioma.

## Licença

[MIT](./LICENSE) — uso, modificação e distribuição livres, mantendo o aviso de copyright.
