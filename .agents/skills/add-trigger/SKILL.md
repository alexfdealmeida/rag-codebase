---
name: add-trigger
description: Adiciona palavras-chave em COMPLEX_TRIGGERS ou ASSISTANT_TRIGGERS no main.py
---

Adicione uma nova palavra-chave ao roteador de modelos em `app/main.py`.

Argumento recebido: $ARGUMENTS

## Passos

1. Leia a seção de triggers em `app/main.py` (procure por `COMPLEX_TRIGGERS` e `ASSISTANT_TRIGGERS`)
2. Identifique em qual lista a nova palavra-chave pertence:
   - **COMPLEX_TRIGGERS** → palavras que indicam perguntas técnicas complexas (impacto, refatoração, segurança, etc.) - aciona o modelo Opus diretamente
   - **ASSISTANT_TRIGGERS** → expressões que indicam perguntas sobre o próprio assistente (quem você é, o que você faz, etc.) - ignora o `ccc search`
3. Verifique se a palavra-chave já existe (use grep para evitar duplicatas)
4. Adicione a nova entrada na lista correta, seguindo o agrupamento por categoria nos comentários existentes
5. A normalização via `_normalize()` já cobre acentos - não é necessário duplicar com/sem acento
6. Confirme a edição mostrando o trecho atualizado

## Regras

- Mantenha o agrupamento por categoria com os comentários existentes (`# Impacto e risco`, `# Identidade`, etc.)
- Use vírgula ao final de cada entrada (exceto a última do grupo)
- Não altere nada fora dos blocos `COMPLEX_TRIGGERS` e `ASSISTANT_TRIGGERS`
