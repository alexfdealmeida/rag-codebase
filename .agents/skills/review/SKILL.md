---
name: review
description: Revisa o diff atual antes de commitar - verifica estilo, lógica e consistência
---

Revise as alterações pendentes neste repositório. Siga estas etapas:

1. Execute `git diff` para ver mudanças não staged e `git diff --staged` para staged
2. Para cada arquivo modificado, leia o contexto ao redor das mudanças
3. Verifique:
   - **Estilo:** linha máxima de 100 chars, aspas duplas, comentários de seção preservados
   - **Lógica:** o roteamento de modelos continua correto? thresholds fazem sentido?
   - **Triggers:** novas entradas em `COMPLEX_TRIGGERS` ou `ASSISTANT_TRIGGERS` têm versão acentuada e sem acento? (a normalização via `_normalize()` já cobre, mas é bom conferir consistência com as existentes)
   - **Dependências:** alguma nova importação foi adicionada sem `uv add`?
   - **Segredos:** nenhuma chave, token ou senha foi exposta no código
   - **Frontend:** mudanças em `index.html` preservam o funcionamento do dark mode, streaming e modal de confirmação?
4. Verifique se o `pyproject.toml` e `uv.lock` estão sincronizados com as importações usadas
5. Aponte problemas encontrados por ordem de severidade: crítico, aviso, sugestão

Finalize com um resumo indicando se o diff está pronto para commit ou lista de ajustes necessários.
