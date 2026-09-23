---
name: qa-md-gen
description: "Gera um checklist de QA vivo (HTML interativo autocontido) a partir dos endpoints e fluxos de um repositório. Cada item é marcável ✓ Passou / ✗ Falhou, com caixa de descrição de erro e botão que exporta um prompt .md de correção pronto para um agente. Use quando o usuário pedir checklist de testes/QA, cobertura de endpoints, roteiro de teste da plataforma, ou quer transformar a superfície da API num plano de QA rastreável. Trigger: /qa-md-gen."
---

# /qa-md-gen

Varre a superfície de uma aplicação (endpoints HTTP, workers/crons, rotas de frontend, regras de negócio/segurança), agrupa por domínio de produto e gera **um único HTML autocontido** de QA vivo, além de fixar a regra de manutenção do checklist no projeto.

## Uso

```
/qa-md-gen                       # gera o checklist do repositório atual
/qa-md-gen <caminho>             # gera para um caminho/módulo específico
/qa-md-gen --out docs/qa         # define a pasta de saída (default: internal-docs/qa, docs/qa ou qa/)
```

## Passos (executar nesta ordem)

### 1. Descobrir a superfície (leia o código — não invente)
Levantamento **exaustivo**, adaptando ao stack encontrado:
- **Endpoints HTTP**: `serverless.yml` / `sls/functions/*.yml` / `template.yaml` (eventos `httpApi`/`http`); FastAPI/Flask (`@app.get`, `APIRouter`); Express/Nest (`router.get`, `@Get`); Django/DRF (`urls.py`, `ViewSet`); Rails (`routes.rb`); Spring (`@GetMapping`); Go (`mux`/`chi`/`gin`); e `openapi.yaml`/`swagger.json` se houver. Extraia **método + caminho + intenção**.
- **Workers/filas/crons**: handlers SQS/SNS/Kafka, jobs agendados (cron/EventBridge/Celery beat), DLQs → cada um vira item `FLOW`.
- **Frontend/telas** (se houver): agrupar por domínio e captar fluxos de UI (loading/erro/vazio, troca de contexto, responsividade).
- **Regras de negócio e segurança** já presentes: multi-tenancy/autorização, quota/limites de plano, rate limiting, criptografia de tokens, webhooks assinados, LGPD/GDPR (export/exclusão) → viram cenários de teste.

### 2. Agrupar por domínio de produto
Não por camada técnica. Cada domínio: `id` (kebab único), `title`, `tag` (rótulo curto), `desc` (1 linha).

### 3. Escrever os itens (cenários concretos e verificáveis)
Não "testar login", e sim "login válido cria sessão httpOnly; credencial errada não revela qual campo falhou". Cubra quando aplicável: caminho feliz + validação (payload malformado → 4xx); **autorização/IDOR** (recurso de outro tenant negado); limites (quota, rate limit, tipo/tamanho de upload); idempotência/uso único (tokens, magic links, webhooks) e condições de corrida; erros não vazam stack trace/dados sensíveis.

Marque `sev:"crit"` os itens de **risco alto ao cliente** (auth, cross-tenant, cobrança, perda/vazamento de dados, publicação/entrega). Itens sem rota HTTP usam `m:"FLOW"`. Inclua um domínio final **"Transversais"** (IDOR global, autorização, rate limiting, validação, i18n, tema claro/escuro, responsividade, retenção de logs).

Forma de cada domínio no array `DATA`:
```js
{ id:"auth", title:"Autenticação & Sessão", tag:"auth",
  desc:"Login, refresh e logout.",
  items:[
    { m:"POST", p:"/auth/login", t:"Login válido cria sessão httpOnly; credencial errada não revela o campo", sev:"crit" },
    { m:"FLOW", t:"Sessão expira no TTL; acesso após expiração é negado" },
  ] }
```
Regras: `m ∈ GET|POST|PATCH|PUT|DELETE|FLOW`; `p` só em HTTP; `t` obrigatório; `sev:"crit"` opcional.

### 4. Gerar o HTML a partir do template bundled
Use `templates/checklist.template.html` (nesta skill) e substitua os placeholders — **não altere CSS nem JS**:
- `{{PROJECT_NAME}}` → nome de exibição do projeto (detecte de `package.json`/`pyproject.toml`/README).
- `{{PROJECT_SLUG}}` → slug kebab-case (usado em `localStorage` `<slug>-qa-v2` e no nome do `.md` exportado).
- `{{DATA}}` → o array `DATA` completo, como literal JS indentado.

Salve em `<out>/qa-checklist.html` (default: `internal-docs/qa/`, senão `docs/qa/` ou `qa/` na raiz — siga a convenção do repo).

### 5. Fixar a regra de "checklist vivo"
Adicione ao arquivo de diretrizes do repo (`CLAUDE.md` / `AGENTS.md` / `CONTRIBUTING.md` / README):
> **Checklist de QA vivo — obrigatório.** Toda nova funcionalidade ou endpoint DEVE virar um item no array `DATA` de `<out>/qa-checklist.html` antes de a feature ser considerada concluída.

Se houver checklist de conformidade/PR, adicione um item marcável equivalente.

### 6. Entregar
Reporte: nº de domínios e itens, quantos `crit`, caminho do HTML e onde a regra foi fixada. **Não** faça commit a menos que pedido.

## O que o HTML gerado faz
Um arquivo, zero dependências. Cada item tem **✓ Passou / ✗ Falhou**; ao falhar abre uma **caixa de descrição do erro**. Progresso persiste em `localStorage`. O botão **Exportar erros (.md) / Copiar prompt** gera um Markdown **só com as falhas + descrições**, formatado como **prompt de correção pronto** (agrupado por domínio, críticos primeiro, passos RED→GREEN, instrução de revalidar o checklist).

## Notas
- O template é a fonte da UI/UX; ajustes nele valem para todas as gerações futuras.
- Idioma padrão PT-BR; para outro idioma, traduza os rótulos fixos do template e escreva os `t` no idioma desejado.
- Ciclo: **gerar → QA marca ✓/✗ → exportar .md → agente corrige → revalidar itens**.
