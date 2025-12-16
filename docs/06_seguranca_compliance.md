# Segurança, Compliance e Guardrails

## Segredos e autenticação
- Guardar tokens OAuth e keys em **AWS Secrets Manager**.
- Lambdas/ECS acessam via IAM Role (least privilege).
- Nunca logar:
  - access_token, refresh_token, client_secret
  - URLs assinadas do S3 com querystring completa

## Guardrails de conteúdo
- Checkpoints antes de publicar:
  - **copyright**: texto original (sem copiar trechos grandes)
  - **marca/claims**: evitar afirmações sem evidência
  - **segurança**: não ensinar abuso, invasão, fraude, etc.
  - **PII**: nunca incluir dados pessoais reais

## Aprovação humana (recomendado)
- No MVP: tudo passa por `AWAITING_APPROVAL`.
- Exceção: conteúdos “evergreen” previamente aprovados.

## Observabilidade e auditoria
- Log estruturado (JSON) com:
  - `jobId`, `stage`, `durationMs`, `status`, `errorCode`
- Métricas:
  - sucesso por etapa
  - tempo de render
  - falhas de integração por canal

## Rate limits e backoff
- Implementar retries com jitter:
  - `exponential backoff` (ex.: 2s, 4s, 8s, 16s)
- Tratar 429/5xx com retries.
- Persistir “tentativas” no item do job.

## Governança de conteúdo
- Versionar prompts (S3 + git).
- Salvar “prompt used” no job (hash) para rastreabilidade.

## Políticas das plataformas
- Manter referência às políticas oficiais de cada canal (YouTube, Meta/Instagram, TikTok, X, LinkedIn).
- Antes de ativar autopost em um canal, validar:
  - termos de uso para automação e conteúdo gerado por IA.
  - requisitos de auditoria/aprovação de app (ex.: TikTok, Instagram/Facebook).
- No job (tabela `ContentJobs`), usar flag `platformComplianceChecked=true` para indicar que o conteúdo/canal foi avaliado.
