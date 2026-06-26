# Solução de problemas

## Debug de erros no frontend

Se a interface apresentar tela branca ou comportamento inesperado, ative o modo debug adicionando `?debug=true` na URL:

```
http://localhost:8000/?debug=true
```

Isso exibe um painel no rodape capturando:
- Erros JavaScript nao tratados (`window.onerror`)
- Promessas rejeitadas (`onunhandledrejection`)
- Stack traces completos

Útil para diagnosticar problemas com CDN, JSX malformado, ou versões incompatíveis de bibliotecas.

### Problemas comuns de CDN

O frontend depende de bibliotecas via CDN (React, Babel Standalone, marked.js). Se houver atualizações que quebrem compatibilidade:

1. Acesse com `?debug=true` para ver o erro específico
2. Verifique as versoes fixadas no `index.html`
3. Se necessário, ajuste as URLs das CDNs para versões estáveis conhecidas
