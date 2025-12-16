# Roadmap sugerido (MVP → Escala)

## MVP (1–2 semanas)
- Pipeline diário com:
  - tema → notas → matéria do blog (draft)
  - roteiro YouTube
  - gerar assets (thumb + cortes) (pode ser placeholder inicial)
- Aprovação humana
- Publicação automatizada apenas:
  - Blog (draft/publish)
  - X (teaser) e LinkedIn (post) — opcional
 - Implementação inicial como script único (v0) agendado (CRON/GitHub Actions), já seguindo contratos de dados e S3/DynamoDB.

## Fase 2
- Render de vídeo completo automático (FFmpeg/ECS)
- Upload YouTube automático (unlisted → public após aprovação)
- Instagram Reels via API (se requisitos OK)
- Painel admin simples (listar jobs, aprovar, reprocessar) usando React + Vite + Material UI (AWS Amplify)

## Fase 3
- TikTok autopost (após auditoria e permissões)
- Coleta de métricas por canal + relatórios semanais
- A/B de títulos/thumbs (controlado)

## Fase 4
- Multi-brand (Zentriz Monitor, ZMCC FinOps, etc.)
- Série editorial automática (playlists e tags)
- Recomendações baseadas em performance (tema que converte mais)
