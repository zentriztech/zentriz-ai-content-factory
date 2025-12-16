# Deploy e Operação

## Repositório sugerido
- `./docs` (esta documentação)
- `./infra` (IaC — Serverless Framework / Terraform)
- `./services`
  - `orchestrator` (Step Functions definitions)
  - `generators` (texto/roteiro/captions)
  - `render` (ECS/FFmpeg)
  - `publishers` (YouTube/LinkedIn/etc.)
  - `blog` (web app do blog público — React + Vite + Material UI + MobX)
  - `admin-ui` (web app do painel de gerenciamento — React + Vite + Material UI + MobX)

**Estrutura dos web apps** (seguir padrão do `zentriz-landpage`):
```
services/[blog|admin-ui]/
├── src/
│   ├── components/     # Componentes reutilizáveis
│   ├── pages/          # Componentes de página
│   ├── hooks/          # React hooks personalizados
│   ├── stores/         # Stores MobX (controle de estado)
│   ├── services/       # Serviços de API (fetch)
│   ├── utils/          # Utilitários
│   └── styles/         # Estilos globais (SCSS)
├── package.json
├── vite.config.ts
└── tsconfig.json
```

## Deploy (MVP)
- Serverless Framework para Lambdas + Step Functions.
- Bucket S3 para assets.
- DynamoDB tables.
- Secrets Manager secrets por canal.
- CloudWatch alarms (falha do state machine, erro de publish).

## Estrutura de domínios e URLs
- **Site principal**: `https://zentriz.com.br/` (projeto existente em `/Users/mac/workspace/current/zentriz/zentriz-landpage/`).
- **Blog público** (`https://blog.zentriz.com.br/`): **web app separado** — exibição pública das matérias geradas.
  - Subdomínio do site principal.
  - Implementado como React + Vite + Material UI, deploy via AWS Amplify.
- **Admin UI** (`https://blogeditor.zentriz.com.br`): **web app separado** — painel de gerenciamento (calendário editorial, aprovação, reprocesso).
  - Subdomínio do site principal.
  - Deploy via AWS Amplify (React + Vite + MUI).

**Importante**: Blog e Admin UI são **dois web apps distintos**, cada um com seu próprio código-fonte, deploy e configuração.

## Caminho de implementação (MVP técnico)
- **v0 — Monolito agendado**:
  - Script único (ex.: Python/Node) agendado (CRON ou GitHub Actions) que:
    - seleciona tema → gera notas → matéria do blog → roteiro YouTube → posts LinkedIn/X.
    - grava assets em S3 e metadados em DynamoDB seguindo as convenções deste blueprint.
- **v1 — Orquestração com Step Functions**:
  - migrar etapas do script v0 para estados individuais (Lambdas/ECS) na state machine.
  - manter os mesmos contratos de entrada/saída para facilitar evolução.
- **Admin UI (MVP+)**:
  - app React + Vite + MUI hospedado em **AWS Amplify** (deploy automático via GitHub).
  - **URL**: `https://blogeditor.zentriz.com.br`
  - lista jobs, mostra preview (link HTML), permite aprovar/rejeitar/reprocessar.
  - gestão do calendário editorial (criar/editar temas com descrição e prompt para IA).

## Observabilidade mínima
- Dashboard CloudWatch:
  - Jobs/dia
  - Falhas por estágio
  - Tempo médio de render
- Alarmes:
  - `StateMachineFailed > 0`
  - `PublishFailed > 0`

## Operação diária
- O job roda e produz:
  - link do post do blog (draft)
  - link do vídeo (unlisted, se preferir)
  - pacote de cortes
- Você aprova → publica.

## Estratégia de rollback
- Blog: publicar como draft e “publicar” só após aprovação.
- YouTube: subir como **unlisted** até aprovação.
- Redes: postar só após aprovação.

## SEO e indexação no Google Search
**Objetivo**: garantir que todas as novas páginas do blog sejam indexadas rapidamente pelo Google.

**Componentes necessários**:
- **Google Search Console API** (Indexing API v3):
  - Credenciais OAuth 2.0 armazenadas em Secrets Manager.
  - Lambda que submete URL após publicação bem-sucedida.
- **Sitemap.xml dinâmico**:
  - Gerado automaticamente a partir de `PublishedPosts` no DynamoDB (filtro: `channel=BLOG`, `status=published`).
  - Hospedado em `https://blog.zentriz.com.br/sitemap.xml`.
  - Atualizado após cada nova publicação.
- **Ping automático do sitemap**:
  - Após atualizar sitemap, fazer ping: `https://www.google.com/ping?sitemap=https://blog.zentriz.com.br/sitemap.xml`.
- **Structured Data (JSON-LD)**:
  - Incluir schema.org `Article` no HTML de cada post (gerado na etapa de publicação do blog).
- **robots.txt**:
  - Garantir que `/sitemap.xml` está acessível.
  - Não bloquear posts individuais.

**Implementação**:
- Lambda `index-new-post` executada após publicação do blog (Step Functions).
- Se falhar, registrar em DynamoDB para retry via cron diário.
- Ver `02_pipeline_conteudo.md` (etapa 11) e `04_integracoes_canais.md` (seção Blog).

**Monitoramento**:
- CloudWatch métrica: URLs submetidas vs URLs indexadas (via Search Console API).
- Alarme se taxa de indexação < 80% após 7 dias.

## Custos: como manter controlado
- Render em ECS com task size fixo (ex.: 1–2 vCPU, 2–4GB) e tempo limite.
- S3 lifecycle: limpar intermediários.
- Step Functions: evitar loops longos; preferir batch por canal.

## Diagrama de Deploy e Operação

```mermaid
flowchart TD
    Start[Início do Projeto] --> V0[v0 - Monolito Agendado<br/>Script único Python/Node<br/>CRON ou GitHub Actions]
    
    V0 --> V1[v1 - Orquestração<br/>Step Functions + Lambdas<br/>Serverless Framework]
    
    V1 --> MVP[MVP+ - Admin UI<br/>React + Vite + MUI<br/>AWS Amplify]
    
    subgraph Infra["Infraestrutura AWS"]
        SF[Step Functions<br/>Orquestrador]
        L[Lambda Functions<br/>Geradores e Publishers]
        ECS[ECS Fargate<br/>Render de Vídeo]
        S3[S3 Bucket<br/>Assets]
        DDB[DynamoDB<br/>Jobs e Posts]
        SM[Secrets Manager<br/>Credenciais]
        CW[CloudWatch<br/>Logs e Métricas]
    end
    
    subgraph WebApps["Web Apps"]
        Blog[Blog Público<br/>blog.zentriz.com.br<br/>AWS Amplify]
        Admin[Admin UI<br/>blogeditor.zentriz.com.br<br/>AWS Amplify]
    end
    
    subgraph CICD["CI/CD"]
        GH[GitHub<br/>Repositório]
        GA[GitHub Actions<br/>Build e Deploy]
        AMP[AWS Amplify<br/>Deploy Automático]
    end
    
    V1 --> SF
    SF --> L
    SF --> ECS
    L --> S3
    L --> DDB
    L --> SM
    L --> CW
    
    GH --> GA
    GA --> AMP
    AMP --> Blog
    AMP --> Admin
    
    Admin --> DDB
    Admin --> L
    
    style Start fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    style V0 fill:#fff9c4,stroke:#f57c00,stroke-width:2px
    style V1 fill:#c8e6c9,stroke:#388e3c,stroke-width:2px
    style MVP fill:#c8e6c9,stroke:#388e3c,stroke-width:2px
    style Blog fill:#e1bee7,stroke:#7b1fa2,stroke-width:2px
    style Admin fill:#e1bee7,stroke:#7b1fa2,stroke-width:2px
```
