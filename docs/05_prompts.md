# Prompts base (PT-BR)

> Use como ponto de partida. Ajuste o “tom da Zentriz”: direto, técnico, com foco em AWS/FinOps/DevOps/IA, sem enrolação.

## 1) Prompt — Seleção de tema do dia (com suporte a calendário editorial)
**System (persona)**
Você é editor técnico da Zentriz (Cloud/FinOps/DevOps/IA). Seu objetivo é propor temas úteis e práticos para engenheiros.

**User**
Gere 10 ideias de tema para hoje considerando:
- Público: Tech Leads, arquitetos, DevOps/FinOps
- Stack: AWS (Lambda, API Gateway, S3, DynamoDB, CloudWatch, Cost Explorer, Budgets)
- Formato: matéria + vídeo 8–14min + 3 cortes (30–50s)
Se existir um item de calendário editorial para hoje, use como restrição principal:
- Título sugerido: {{calendar_title}}
- Descrição do assunto: {{subjectDescription}}
- Dica de prompt/comando da IA: {{aiPromptHint}}
Regras:
- Sem clickbait
- Sempre com “passo a passo” e “erros comuns”
Saída em JSON:
{ "ideas":[ { "title":"", "keywords":[], "angle":"", "why_it_matters":"" } ] }

> Observação: quando `calendar_title`/`subjectDescription`/`aiPromptHint` forem informados, as ideias devem ser variações desse eixo temático (não inventar tema totalmente diferente).

## 2) Prompt — Pesquisa e notas (sem copiar texto)
Você vai receber um tema. Gere:
- notas técnicas (bullet points)
- lista de claims (afirmações verificáveis)
- lista de termos e conceitos que precisam estar corretos
Saída:
- Markdown: notas
- JSON: claims e termos

Entrada:
Tema: {{title}}
Keywords: {{keywords}}

## 3) Prompt — Matéria do blog (canonical)
Escreva uma matéria original, em PT-BR, com estrutura:
1. Resumo (5–7 linhas)
2. Contexto (por que importa)
3. Passo a passo (com comandos/exemplos)
4. Erros comuns
5. Checklist final
6. CTA: “vídeo completo no YouTube” + “link”

Restrições:
- Frases curtas
- Listas compactas
- Evitar exageros e promessas absolutas
- Se citar números, seja conservador e explique suposições

Entrada:
Notas: {{research_notes}}

Saída:
- Markdown com título H1 e seções H2/H3
- Metadados em YAML frontmatter:
title, slug, tags, seoTitle, seoDescription

## 4) Prompt — Roteiro YouTube (8–14 min)
Crie roteiro com:
- Hook 0:00–0:20
- Contexto 0:20–1:30
- Corpo com 3–5 blocos (capítulos)
- Encerramento + CTA
- Sugestão de cenas (B-roll, telas, diagramas)
- 3 ideias de cortes com timestamps e “frase de impacto”

Entrada:
Matéria: {{blog_post_md}}

Saída:
- Markdown com timestamps e capítulos
- JSON com:
  { "youtube":{ "title":"", "description":"", "tags":[],"chapters":[] }, "shorts":[ ... ] }

## 5) Prompt — Captions de cortes (IG/TikTok/Kwai)
Para cada corte:
- 1 caption curta (máx 140 chars)
- 1 caption média (máx 300 chars)
- 10 hashtags relevantes (sem spam)
- 1 CTA (YouTube + Blog)

Entrada:
Shorts manifest: {{shorts_manifest}}

## 6) Prompt — Post LinkedIn (trecho do blog)
Gere um post com:
- 1 parágrafo de contexto
- 3 bullets (insights)
- 1 pergunta final
- Link para o blog (no final)

Entrada:
Matéria: {{blog_post_md}}

## 7) Prompt — Post X (Twitter)
Gere 3 variações:
- 1 linha (teaser)
- 1 mini-thread (4 posts)
- 1 post com bullet points
Sempre incluir link do blog.

Entrada:
Matéria: {{blog_post_md}}
