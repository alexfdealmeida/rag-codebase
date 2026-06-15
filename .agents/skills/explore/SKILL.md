---
name: explore
description: Explora e mapeia a codebase do projeto antes de propor mudanças
---

Faça uma exploração estruturada da codebase do projeto. Siga estas etapas:

1. Leia o `README.md` para entender o objetivo e a arquitetura geral
2. Leia o `app/main.py` identificando as seções principais (configuração, roteamento de modelos, busca semântica, prompt, endpoints)
3. Leia o `app/config.yml` para entender o mapeamento de repositórios
4. Liste os arquivos em `app/static/` para entender a estrutura do frontend
5. Verifique o `pyproject.toml` para dependências e ferramentas

Ao final, apresente um resumo com:
- Fluxo principal de uma pergunta (do browser até a resposta via SSE)
- Modelos disponíveis e quando cada um é acionado
- Arquivos que provavelmente precisam ser tocados para cada tipo de mudança:
  - Novo trigger de complexidade → `COMPLEX_TRIGGERS` em `app/main.py`
  - Novo trigger de assistente → `ASSISTANT_TRIGGERS` em `app/main.py`
  - Novo repositório/branch → `app/config.yml`
  - Mudança de UI → `app/static/index.html`
  - Nova dependência → `uv add <pacote>` (reflete em `pyproject.toml`)
