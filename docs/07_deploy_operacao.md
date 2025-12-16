# Deploy e Operação

## Repositório sugerido
- `./docs` (esta documentação)
- `./infra` (IaC — Serverless Framework / Terraform)
- `./services`
  - `orchestrator` (Step Functions definitions)
  - `generators` (Python — texto/roteiro/captions)
  - `render` (Python ou ECS — FFmpeg)
  - `publishers` (Python — YouTube/LinkedIn/etc.)
  - `api` (Node.js/TypeScript — API Gateway)
    - `admin-api` (endpoints para Admin UI)
    - `webhooks` (callbacks e webhooks)
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

## Estratégia de Linguagem para Lambdas
**Decisão técnica**: usar linguagens diferentes conforme o tipo de Lambda:

- **Node.js/TypeScript**: para Lambdas expostas via **API Gateway** (REST APIs).
  - Exemplos: endpoints do Admin UI, webhooks, APIs de consulta.
  - Motivo: melhor integração com frontend React, tipagem TypeScript, ecossistema npm.

- **Python**: para Lambdas de **processamento** (não expostas via API Gateway).
  - Exemplos: geradores de conteúdo (texto, roteiro, captions), publishers (YouTube, LinkedIn, etc.), indexação Google Search.
  - Motivo: excelente suporte para processamento de texto, bibliotecas de IA (OpenAI, Anthropic), manipulação de dados, APIs de LLM.

**Estrutura de serviços**:
```
services/
├── generators/          # Python (processamento de texto/IA)
├── publishers/          # Python (integrações com APIs externas)
├── render/             # Python ou ECS (FFmpeg)
└── api/                # Node.js/TypeScript (API Gateway)
    ├── admin-api/      # Endpoints para Admin UI
    └── webhooks/       # Webhooks e callbacks
```

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
    Start[Início do Projeto] --> V0[v0 - Monolito Agendado<br/>Script único Python<br/>CRON ou GitHub Actions]
    
    V0 --> V1[v1 - Orquestração<br/>Step Functions + Lambdas<br/>Serverless Framework]
    
    V1 --> MVP[MVP+ - Admin UI<br/>React + Vite + MUI<br/>AWS Amplify]
    
    subgraph Infra["Infraestrutura AWS"]
        SF[Step Functions<br/>Orquestrador]
        
        subgraph LambdasPython["Lambdas Python"]
            LP1[Generators<br/>Geração de conteúdo]
            LP2[Publishers<br/>YouTube, LinkedIn, etc]
            LP3[Indexação<br/>Google Search Console]
        end
        
        subgraph LambdasNode["Lambdas Node.js/TypeScript"]
            LN1[Admin API<br/>API Gateway]
            LN2[Webhooks<br/>Callbacks]
        end
        
        ECS[ECS Fargate<br/>Render de Vídeo Python]
        S3[S3 Bucket<br/>Assets]
        DDB[DynamoDB<br/>Jobs e Posts]
        SM[Secrets Manager<br/>Credenciais]
        CW[CloudWatch<br/>Logs e Métricas]
        APIGW[API Gateway<br/>REST APIs]
    end
    
    subgraph WebApps["Web Apps"]
        Blog[Blog Público<br/>blog.zentriz.com.br<br/>AWS Amplify]
        Admin[Admin UI<br/>blogeditor.zentriz.com.br<br/>AWS Amplify]
    end
    
    subgraph CICD["CI/CD"]
        GH[GitHub<br/>Repositório]
        GA[GitHub Actions<br/>Build e Deploy]
        SFW[Serverless Framework<br/>Deploy Lambdas Python]
        AMP[AWS Amplify<br/>Deploy Web Apps]
    end
    
    V1 --> SF
    SF --> LP1
    SF --> LP2
    SF --> LP3
    SF --> ECS
    
    LP1 --> S3
    LP1 --> DDB
    LP2 --> S3
    LP2 --> DDB
    LP3 --> SM
    ECS --> S3
    
    LP1 --> SM
    LP2 --> SM
    LP1 --> CW
    LP2 --> CW
    LP3 --> CW
    
    GH --> GA
    GA --> SFW
    GA --> AMP
    SFW --> LP1
    SFW --> LP2
    SFW --> LP3
    SFW --> LN1
    SFW --> LN2
    SFW --> SF
    AMP --> Blog
    AMP --> Admin
    
    Admin --> APIGW
    APIGW --> LN1
    LN1 --> DDB
    LN1 --> SF
    
    style Start fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    style V0 fill:#fff9c4,stroke:#f57c00,stroke-width:2px
    style V1 fill:#c8e6c9,stroke:#388e3c,stroke-width:2px
    style MVP fill:#c8e6c9,stroke:#388e3c,stroke-width:2px
    style Blog fill:#e1bee7,stroke:#7b1fa2,stroke-width:2px
    style Admin fill:#e1bee7,stroke:#7b1fa2,stroke-width:2px
    style LP1 fill:#ffebee,stroke:#c62828,stroke-width:2px
    style LP2 fill:#ffebee,stroke:#c62828,stroke-width:2px
    style LP3 fill:#ffebee,stroke:#c62828,stroke-width:2px
    style LN1 fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px
    style LN2 fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px
    style APIGW fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px
    style SFW fill:#fff3e0,stroke:#e65100,stroke-width:2px
    style GA fill:#fff3e0,stroke:#e65100,stroke-width:2px
```
