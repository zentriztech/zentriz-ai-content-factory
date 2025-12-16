# Pipeline diário — passo a passo

## Estados do job (sugestão)
- `CREATED`
- `TOPIC_SELECTED`
- `RESEARCH_DONE`
- `BLOG_DRAFTED`
- `SCRIPT_DONE`
- `AUDIO_DONE`
- `VIDEO_RENDERED`
- `SHORTS_RENDERED`
- `AWAITING_APPROVAL` (opcional)
- `PUBLISHING`
- `PUBLISHED`
- `FAILED` / `NEEDS_ATTENTION`

## 1) Seleção de tema
Entradas:
- calendário editorial (`EditorialCalendar` mantido via Admin UI web)
- backlog de ideias
- séries (ex.: “FinOps da Semana”, “Serverless na prática”, “Observabilidade”)

Saídas:
- `topic.title`
- `topic.keywords[]`
- `topic.angle` (ex.: tutorial, opinião técnica, estudo de caso)
- `persona` (tom da Zentriz: técnico + objetivo)
- `topic.source` (ex.: `EDITORIAL_CALENDAR` ou `AUTO_GENERATED`)
- `topic.calendarItemId` (se veio do calendário editorial)

Checks:
- evitar repetição (ex.: 14 dias)
- alinhamento com estratégia (Zentriz: FinOps, AWS, Cloud, IA, DevOps)

## 2) Pesquisa e notas (curadoria)
Objetivo:
- gerar um conjunto de **notas** e **fatos** (sem copiar texto) para embasar a matéria.
- opcional: salvar URLs de referência (não publicar automaticamente se for sensível).

Saídas:
- `research.notes.md`
- `research.bullets.json` (claims e evidências)
- `risk_flags[]` (copyright, marcas, números suspeitos, etc.)

## 3) Matéria do blog (fonte)
Saídas:
- `blog.post.md`
- `blog.post.html` (opcional)
- `blog.metadata.json` (slug, seoTitle, seoDescription, tags, canonicalUrl)

Regras:
- Estrutura padrão: `Resumo`, `Contexto`, `Passo a passo`, `Erros comuns`, `Checklist`, `Referências`.
- Sempre ter CTA no final: “assista o vídeo completo” + “inscreva-se”.

## 4) Roteiro do vídeo completo (YouTube)
Saídas:
- `youtube.script.md` (com timestamps aproximados)
- `youtube.title`
- `youtube.description`
- `youtube.tags[]`
- `youtube.chapters[]`

Regras:
- abrir com “por que isso importa” em 10–20s.
- capítulos claros, sem enrolação.
- CTA suave 1x no meio e 1x no fim.

## 5) Áudio de narração
Opções:
- TTS (ex.: Polly) ou voice provider externo.
Saídas:
- `narration.wav` / `narration.mp3`
- `narration.timestamps.json` (se disponível)

## 6) Render do vídeo completo
Entradas:
- áudio
- roteiro (cards, B-roll, imagens, slides)
- template (intro/outro, lower thirds)

Saídas:
- `youtube.video.mp4`
- `youtube.thumbnail.png`
- `youtube.captions.srt`

## 7) Cortes (9:16)
Entradas:
- vídeo master
- “plano de cortes” (timestamps ou detecção automática)

Saídas:
- `shorts/01.mp4 ...`
- `shorts/01.caption.txt ...`
- `shorts/manifest.json` (metadados de cada corte)

## 8) Aprovação humana (recomendado no MVP)
- Gera "preview page" (HTML simples no S3/CloudFront) com:
  - matéria, título, thumb, vídeo, captions, cortes
- Envia link no Slack/Email
- Aprovação via **Admin UI** (`https://blogeditor.zentriz.com.br`) = toggle em DynamoDB (`approved=true`)

## 9) Publicação por canal
- **Blog** (`https://blog.zentriz.com.br/`) — primeiro, via commit GitHub (Markdown em `content/posts/` → Amplify rebuild automático, draft → publicado após aprovação).
- **LinkedIn** (trecho + link para o blog).
- **YouTube** (vídeo completo, unlisted até aprovação).
- **Instagram/TikTok/X/Kwai** (cortes/teasers; conforme integrações).

## 10) Pós-publicação
- registrar `externalPostId`, `url`, `publishedAt`
- coletar métricas (views/likes/comments) via APIs quando disponível (cron diário)

## 11) Indexação no Google Search
Após publicação do blog, garantir indexação:
- **Submeter URL no Google Search Console** via API (Indexing API v3).
- **Atualizar sitemap.xml** (`https://blog.zentriz.com.br/sitemap.xml`) com nova URL.
- **Ping do sitemap** para Google (`https://www.google.com/ping?sitemap=https://blog.zentriz.com.br/sitemap.xml`).
- **Structured Data (JSON-LD)**: incluir schema.org `Article` no HTML do post (título, autor, data, imagem, etc.).
- **Verificar robots.txt**: garantir que `/sitemap.xml` está permitido e posts não estão bloqueados.

**Automação**:
- Lambda/Step Functions executa após publicação bem-sucedida do blog.
- Se falhar, registrar em DynamoDB para retry posterior (cron diário).
