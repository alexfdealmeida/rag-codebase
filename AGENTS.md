# AliCodebase - Rules

## Projeto

Interface web de chat com RAG para análise de bases de código.
Backend em FastAPI (Python), frontend em React via CDN (sem build step).
Busca semântica via `cocoindex-code` (`ccc search`), LLM via Anthropic API.

## Stack

- **Backend:** FastAPI + Python 3.11+, gerenciado com `uv`
- **Frontend:** React 18 + Tailwind CSS + marked.js + highlight.js (todos via CDN, sem build)
- **Busca semântica:** `cocoindex-code` (`ccc search`) com modelo `nomic-ai/CodeRankEmbed`
- **LLM:** Anthropic API - Sonnet (padrão), Opus (complexo), Haiku (classificador)
- **Streaming:** Server-Sent Events (SSE)
- **Containerização:** Docker + Docker Compose

## Arquitetura - arquivo principal

Toda a lógica do backend está em `app/main.py`. As seções são:

1. Configuração inicial (modelos, triggers, thresholds)
2. Roteamento de modelos (`select_model`, `is_about_assistant`, `classify_with_haiku`)
3. Execução do `ccc search` e seleção de chunks
4. Montagem do prompt e chamada à API
5. Endpoints FastAPI (`/chat`, `/stream`, `/`)

## Convenções de código

- Sempre use `uv add <pacote>` para adicionar dependências (nunca edite `pyproject.toml` manualmente para deps)
- Linter: `ruff` - rode `uv run ruff check app/` e `uv run ruff format app/` antes de commitar
- Linha máxima: 100 caracteres
- Aspas duplas no Python
- Comentários de seção no estilo existente (`# ===... SECAO N: TITULO ...===`)
- Não remova comentários explicativos existentes no `main.py`

## Variáveis de ambiente

Definidas no `.env` (nunca commitado). Ver `.env.example` para referência:
- `ANTHROPIC_API_KEY` - chave da API Anthropic (obrigatória)
- `CODEBASE_ROOT` - path raiz dos repositórios indexados (obrigatório)

## Configuração de repositórios

Mapeamento repo/branch → path local em `app/config.yml`.

## Skills disponíveis

- Use `.agents/skills/add-trigger/SKILL.md` para adicionar palavras-chave em `COMPLEX_TRIGGERS` ou `ASSISTANT_TRIGGERS`
- Use `.agents/skills/coding-standards/SKILL.md` ao escrever ou revisar código Python ou React
- Use `.agents/skills/explore/SKILL.md` para entender a codebase antes de propor mudanças
- Use `.agents/skills/review/SKILL.md` para revisar o diff antes de commitar
