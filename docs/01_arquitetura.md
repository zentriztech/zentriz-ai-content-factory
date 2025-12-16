# Arquitetura (AWS) — Zentriz AI Content Factory

## Objetivo
Construir um pipeline diário (ou sob demanda) que:
- Escolhe tema → pesquisa e notas → gera matéria (blog) → gera roteiro → produz vídeo completo (YouTube) → cria cortes → publica por canal.

## Componentes sugeridos (MVP → Escala)
- **EventBridge Scheduler**: dispara diariamente.
- **Step Functions**: orquestra o pipeline com estados e retries.
- **AWS Lambda**: etapas leves (texto, validações, postagem, indexação Google).
  - **Python**: para processamento (geradores, publishers, indexação).
  - **Node.js/TypeScript**: para APIs REST (API Gateway, endpoints Admin UI).
- **ECS Fargate** (ou Lambda container) + **FFmpeg**: renderização de vídeo/cortes quando for pesado.
- **S3**: armazenamento de assets (md/html, áudio, vídeo, cortes, thumbnails, legendas, sitemap.xml).
- **DynamoDB**: tracking de jobs, estados, artefatos e URLs publicados.
- **Secrets Manager**: tokens OAuth/keys (incluindo Google Search Console API).
- **CloudWatch Logs + Metrics + Alarms**: observabilidade.
- **SNS/Slack webhook**: aprovação humana (notificar + link preview).
- **Blog (web app público)**: `https://blog.zentriz.com.br/` — web app separado (React + Vite + Material UI + MobX, deploy via AWS Amplify) onde as matérias são exibidas publicamente.
- **Admin UI (web app de gerenciamento)**: `https://blogeditor.zentriz.com.br` — web app separado (React + Vite + Material UI + MobX, deploy via AWS Amplify) para aprovação, reprocesso, visualização de jobs e gestão do calendário editorial (temas).

**Padrões de desenvolvimento frontend**:
- **Controle de estado**: MobX para estado global, `useState` para estado local.
- **Persistência**: localStorage para dados que precisam persistir entre sessões.
- **Requisições HTTP**: fetch API (nativo) como preferencial, axios apenas quando extremamente necessário.
- **Estrutura**: seguir padrão do projeto `zentriz-landpage` (`/Users/mac/workspace/current/zentriz/zentriz-landpage/`).

**Nota**: Blog e Admin UI são **dois web apps distintos**, cada um com seu próprio código-fonte e deploy. Ambos são subdomínios de `zentriz.com.br`.
- **Google Search Console API**: indexação automática de novas páginas do blog após publicação.

## Diagrama de alto nível (Mermaid)
```mermaid
flowchart TD
  A[EventBridge Daily Trigger] --> B[Step Functions Orchestrator]
  B --> C[Selecionar tema + keywords]
  C --> D[Pesquisa + notas + fontes]
  D --> E[Gerar matéria Blog]
  E --> F[Gerar roteiro YouTube + plano de cortes]
  F --> G[Gerar áudio narração]
  G --> H[Render vídeo completo + legenda + thumb]
  H --> I[Gerar cortes 9:16 + captions]
  I --> J{Aprovação humana?}
  J -->|Sim| K[Notificar + preview links]
  K --> L[Publicar por canal]
  J -->|Não| L[Publicar por canal]
  L --> M[(DynamoDB Jobs/Posts)]
  L --> N[(S3 Assets)]
  L --> O[CloudWatch Metrics/Logs]
  L -->|Blog publicado| P[Indexar no Google Search]
  P --> P1[Submeter URL Search Console]
  P --> P2[Atualizar sitemap.xml]
  P --> P3[Ping sitemap Google]
  P1 --> Q[(DynamoDB seoIndexing)]
  P2 --> Q
  P3 --> Q
```

## Provedores de IA e mídia (abstrações)
- **LLM Provider** (texto):
  - OpenAI, Anthropic, etc., atrás de uma interface única (`generateBlog`, `generateScript`, `generateCaptions`).
- **Voice Provider** (narração):
  - AWS Polly (default) ou provedores externos (ex.: ElevenLabs) encapsulados em um módulo.
- **Video Provider** (renderização):
  - `FFmpeg` em ECS/Lambda container (template de cena) ou APIs de avatar/vídeo externas.
- **Image/Thumb Provider**:
  - Geração via DALL·E/Stable Diffusion ou templates internos.

```mermaid
flowchart LR
  subgraph LLM ["LLM Provider"]
    A1[generateBlog] --> A2[(OpenAI/Anthropic/...)]
    A3[generateScript] --> A2
    A4[generateCaptions] --> A2
  end

  subgraph MEDIA ["Media Providers"]
    B1[VoiceProvider] --> B2[(Polly/ElevenLabs)]
    C1[VideoProvider] --> C2[(FFmpeg/AvatarAPI)]
    D1[ImageProvider] --> D2[(DALL·E/SD/Template)]
  end

  LLM --> PIPE_NODE[Pipeline Lambdas / Step Functions]
  MEDIA --> PIPE_NODE
```

## Fluxo de aprovação com Admin UI (visão)
```mermaid
flowchart TD
  J1[Job SHORTS_RENDERED] --> J2[Set status = AWAITING_APPROVAL]
  J2 --> P1[Gerar Preview Page]
  P1 --> N1[Notificar Slack/SNS com link]
  N1 --> U1[Usuario abre Admin UI React Vite MUI]
  U1 --> U2[Revisar matéria, vídeo, cortes]
  U2 -->|Aprovar| A1[Update DynamoDB approved=true]
  U2 -->|Rejeitar| R1[Update status=NEEDS_ATTENTION]
  A1 --> S1[Step Functions continua para Publicação]
```

## Contratos entre módulos (interface mental)
- Tudo que “gera” escreve em S3 e registra em DynamoDB:
  - `jobId`, `topicId`, `stage`, `status`, `assetKeys[]`, `checks[]`.
- Tudo que “publica” precisa de:
  - `assetKey` (S3), `caption`, `title`, `tags`, `targetChannel`, `scheduleAt` (opcional).

## Estratégia de execução
- **Idempotência**: re-executar um job não duplica postagem.
- **Retries com backoff**: API rate limits e falhas temporárias.
- **DLQ/Dead-letter**: falha persistente vira item “needs_attention”.

## Custos (guideline)
- Texto/validações: Lambda + Step Functions (baixo).
- Renderização de vídeo: ECS Fargate (controlar vCPU/mem) ou MediaConvert (se preferir gerenciado).
- Armazenamento: S3 lifecycle (ex.: manter cortes 90d, masters 365d, logs 30d).

## Multi-tenant / multi-projeto (opcional)
Se você gerar conteúdo para mais de um canal/marca, trate como:
- `tenantId` (Zentriz) + `brandId` (FinOps/Monitor/etc.).

## Diagrama de Arquitetura de Componentes

```mermaid
C4Context
    title System Context - Zentriz AI Content Factory
    
    Person(editor, "Editor/Administrador", "Gerencia calendário editorial, aprova conteúdos gerados e monitora jobs.")
    Person(visitor, "Visitante do Blog", "Acessa e lê matérias publicadas no blog.")
    
    Enterprise_Boundary(aws, "AWS Cloud") {
        System_Boundary(webApps, "Web Applications") {
            System(blog, "Blog Público", "Web app React + Vite + MUI<br/>Exibe matérias publicadas<br/>blog.zentriz.com.br")
            System(adminUI, "Admin UI", "Web app React + Vite + MUI<br/>Painel de gerenciamento<br/>blogeditor.zentriz.com.br")
        }
        
        System_Boundary(orchestration, "Orquestração") {
            System(eventbridge, "EventBridge Scheduler", "Dispara jobs diários automaticamente")
            System(stepfunctions, "Step Functions", "Orquestra o pipeline de geração de conteúdo")
            System(lambda, "Lambda Functions", "Executa etapas leves: geração de texto, validações, publicação")
            System(ecs, "ECS Fargate", "Renderização de vídeo e cortes usando FFmpeg")
        }
        
        System_Boundary(storage, "Storage & Dados") {
            SystemDb(s3, "S3", "Armazena assets: vídeos, áudios, imagens, markdown, sitemap.xml")
            SystemDb(dynamodb, "DynamoDB", "Tracking de jobs, posts publicados e calendário editorial")
            SystemDb(secrets, "Secrets Manager", "Armazena credenciais e tokens OAuth")
        }
        
        System_Boundary(observability, "Observabilidade") {
            System(cloudwatch, "CloudWatch", "Logs, métricas e alarmes do sistema")
            System(sns, "SNS", "Notificações e alertas via Slack/Email")
        }
    }
    
    System_Ext(google, "Google Search Console API", "Indexação automática de páginas do blog")
    System_Ext(youtube, "YouTube API", "Upload e publicação de vídeos")
    System_Ext(linkedin, "LinkedIn API", "Publicação de posts no LinkedIn")
    System_Ext(social, "Redes Sociais APIs", "APIs do X, Instagram, TikTok para publicação de cortes")
    System_Ext(llm, "APIs de IA", "OpenAI, Anthropic para geração de texto")
    System_Ext(voice, "Voice Providers", "AWS Polly, ElevenLabs para síntese de voz")
    
    Rel(editor, adminUI, "Gerencia calendário e aprova conteúdos")
    Rel(visitor, blog, "Lê matérias publicadas")
    
    Rel(eventbridge, stepfunctions, "Dispara diariamente", "Schedule")
    Rel(stepfunctions, lambda, "Orquestra execução")
    Rel(stepfunctions, ecs, "Orquestra renderização")
    Rel(lambda, dynamodb, "Lê/Escreve", "DynamoDB")
    Rel(lambda, s3, "Lê/Escreve assets", "S3 API")
    Rel(lambda, secrets, "Obtém credenciais", "Secrets API")
    Rel(ecs, s3, "Salva vídeos renderizados", "S3 API")
    Rel(lambda, cloudwatch, "Envia logs e métricas", "CloudWatch API")
    Rel(lambda, sns, "Envia notificações", "SNS API")
    Rel(lambda, google, "Submete URLs para indexação", "Indexing API")
    Rel(lambda, youtube, "Publica vídeos", "YouTube Data API")
    Rel(lambda, linkedin, "Publica posts", "LinkedIn API")
    Rel(lambda, social, "Publica cortes", "Social Media APIs")
    Rel(lambda, llm, "Gera conteúdo", "LLM APIs")
    Rel(lambda, voice, "Gera narração", "TTS APIs")
    
    Rel(adminUI, dynamodb, "Consulta jobs e posts", "DynamoDB")
    Rel(adminUI, lambda, "Aprova/Reprocessa", "API Gateway")
    Rel(blog, s3, "Carrega conteúdo", "S3/CloudFront")
    Rel(sns, editor, "Notifica aprovações", "Email/Slack")
```
