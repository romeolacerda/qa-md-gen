# qa-md-gen

Gerador de **checklist de QA vivo** para qualquer aplicação. A partir dos endpoints e fluxos de um repositório, produz um **HTML interativo autocontido** onde o QA marca cada funcionalidade como **✓ Passou** ou **✗ Falhou**, descreve o erro quando falha, e **exporta um prompt `.md` de correção** pronto para alimentar um agente de código.

> Nasceu do checklist de QA de um SaaS de geração de conteúdo e foi generalizado para servir qualquer stack (serverless, FastAPI, Express, Django, Rails, Spring, Go…).

## O que tem aqui

| Arquivo | Papel |
|---|---|
| `PROMPT.md` | O **prompt gerador**. Cole num agente com o repo-alvo aberto; ele varre os endpoints e gera o checklist. |
| `templates/checklist.template.html` | O **template** da UI (CSS + lógica genéricos) com placeholders `{{PROJECT_NAME}}`, `{{PROJECT_SLUG}}`, `{{DATA}}`. |

## Como usar

1. Abra o repositório da aplicação que você quer cobrir num agente de código.
2. Cole o conteúdo da seção **PROMPT** de [`PROMPT.md`](./PROMPT.md).
3. O agente lê os endpoints/workers/rotas, monta o array `DATA`, preenche o template e salva `qa/qa-checklist.html` no repo — e fixa a regra de "checklist vivo" no `CLAUDE.md`/`AGENTS.md`.
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

Defina uma antes de tornar público (ex.: MIT).
