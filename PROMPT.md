# qa-md-gen — Prompt gerador de checklist de QA vivo

Cole este prompt num agente de código (Claude Code, etc.) **com o repositório-alvo aberto**. Ele varre os endpoints/fluxos do projeto e gera um checklist de QA interativo idêntico em estrutura ao de referência, além de fixar a regra de "checklist vivo" no projeto.

> Este arquivo é o prompt. O agente também precisa do template `templates/checklist.template.html` (mesmo repo `qa-md-gen`). Se o agente não tiver acesso ao template, gere o HTML a partir da descrição da seção **4. Formato de saída**.

---

## PROMPT (copiar a partir daqui)

Você é um engenheiro de QA + plataforma. Sua tarefa: **gerar um checklist de QA vivo** para ESTE repositório, cobrindo o máximo de endpoints e fluxos, e salvá-lo como um único HTML autocontido. Não implemente features nem altere código de produção — só produza o checklist e a regra de manutenção.

### 1. Descobrir a superfície da aplicação (não invente — leia o código)

Faça um levantamento **exaustivo**, adaptando-se ao stack encontrado:

- **Endpoints HTTP.** Procure em todas as fontes possíveis:
  - Serverless: `serverless.yml`, `sls/functions/*.yml`, `template.yaml` (SAM), eventos `httpApi`/`http`.
  - Frameworks: FastAPI/Flask (`@app.get`, `APIRouter`), Express/Nest (`router.get`, `@Get`), Django/DRF (`urls.py`, `ViewSet`), Rails (`routes.rb`), Spring (`@GetMapping`), Go (`mux`, `chi`, `gin`), etc.
  - Contratos: `openapi.yaml`/`swagger.json` se existirem.
  Extraia **método + caminho + intenção** de cada rota.
- **Workers assíncronos, filas e crons.** Handlers SQS/SNS/Kafka, jobs agendados (cron/EventBridge/Celery beat), DLQs. Cada um vira um item `FLOW` (sem rota HTTP).
- **Rotas de frontend / telas** (se houver front no repo): use para agrupar por domínio de produto e captar fluxos de UI (estados de loading/erro/vazio, troca de contexto, responsividade).
- **Regras de negócio e segurança** já presentes: multi-tenancy/autorização, quota/limites de plano, rate limiting, criptografia de tokens, webhooks assinados, LGPD/GDPR (export/exclusão de dados). Vire cada uma em cenário de teste.

### 2. Organizar por domínio

Agrupe os endpoints/fluxos em **domínios de produto** (ex.: Autenticação, Onboarding, Billing, Conteúdo, Agendamento, Integrações, Privacidade…), não por camada técnica. Cada domínio recebe: `id` (kebab único), `title`, `tag` (rótulo curto, ex.: o namespace/módulo), `desc` (1 linha).

### 3. Escrever os itens de teste

Para **cada** endpoint e fluxo, escreva um cenário **concreto e verificável** (não "testar login", e sim "login válido cria sessão via cookie httpOnly; credencial errada não revela qual campo falhou"). Cubra, quando aplicável:

- Caminho feliz + validação de entrada (payload malformado → 4xx).
- **Autorização / IDOR**: recurso de outro tenant/usuário é negado.
- Limites: quota de plano, rate limiting, tamanho/tipo de upload.
- Idempotência/uso único (tokens, magic links, webhooks), condições de corrida.
- Erros não vazam stack trace / dados sensíveis.

Marque `sev:"crit"` os itens de **risco alto para o cliente** (auth, cross-tenant, cobrança, perda/vazamento de dados, publicação/entrega). Itens sem rota HTTP usam `m:"FLOW"`.

Formato de cada domínio (array `DATA`):

```js
{ id:"auth", title:"Autenticação & Sessão", tag:"auth",
  desc:"Login, refresh e logout.",
  items:[
    { m:"POST", p:"/auth/login", t:"Login válido cria sessão httpOnly; credencial errada não revela o campo", sev:"crit" },
    { m:"FLOW", t:"Sessão expira no TTL; acesso após expiração é negado" },
  ] }
```

Regras: `m ∈ GET|POST|PATCH|PUT|DELETE|FLOW`; `p` só em itens HTTP; `t` obrigatório; `sev:"crit"` opcional. Adicione um domínio final **"Transversais"** com IDOR global, autorização, rate limiting, validação, i18n, tema claro/escuro, responsividade e retenção de logs.

### 4. Formato de saída (o HTML)

Use `templates/checklist.template.html` e substitua os placeholders:

- `{{PROJECT_NAME}}` → nome de exibição do projeto.
- `{{PROJECT_SLUG}}` → slug kebab-case (usado em `localStorage` e no nome do `.md` exportado).
- `{{DATA}}` → o array `DATA` completo que você montou (JS literal, indentado).

Não altere CSS nem JS do template — a lógica de estado, export e temas é genérica. O HTML resultante:

- É **um arquivo autocontido** (sem dependências externas), abre direto no navegador.
- Cada item tem botões **✓ Passou / ✗ Falhou**; ao falhar, abre uma **caixa de texto** para descrever o erro.
- Progresso e descrições ficam em `localStorage` (chave `<slug>-qa-v2`), por navegador.
- Botão **Exportar erros (.md)** / **Copiar prompt**: gera um Markdown **só com as falhas + descrições**, formatado como um **prompt de correção pronto** (agrupado por domínio, críticos primeiro, com passos RED→GREEN e instrução de revalidar o checklist).

Salve em: `<pasta-de-docs>/qa/qa-checklist.html` (ex.: `internal-docs/qa/`, `docs/qa/` ou `qa/` na raiz — escolha a convenção do repo).

### 5. Fixar a regra de "checklist vivo"

Atualize o arquivo de diretrizes do repositório (`CLAUDE.md`, `AGENTS.md`, `CONTRIBUTING.md` ou o README) adicionando a regra:

> **Checklist de QA vivo — obrigatório.** Toda nova funcionalidade ou endpoint DEVE virar um item no array `DATA` de `<...>/qa/qa-checklist.html` antes de a feature ser considerada concluída.

Se houver um checklist de conformidade/PR no repo, adicione um item marcável equivalente.

### 6. Entregar

Ao final, reporte: quantos domínios e itens foram gerados, quantos `crit`, o caminho do HTML salvo e onde a regra foi fixada. **Não** faça commit a menos que seja pedido.

## (fim do prompt)

---

### Notas de manutenção do gerador

- O template é a **fonte da estrutura**; ajustes de UI/UX feitos nele se propagam a qualquer projeto na próxima geração.
- O idioma padrão do template é PT-BR. Para checklist em outro idioma, traduza os rótulos fixos do template e escreva os `t` no idioma desejado.
- O ciclo de uso: **gerar → QA marca passou/falhou → exportar .md → colar o .md num agente para corrigir → revalidar itens**.
