---
name: coding-standards
description: Padrões de codificação do projeto - aplicar ao escrever ou revisar código Python e React
---

Ao escrever ou modificar código neste projeto, siga rigorosamente os padrões abaixo.

## Geral

- Código sempre em **inglês** - variáveis, funções, classes, parâmetros, chaves de dicionário
- Comentários sempre em **português**
- Nomes de funções e variáveis descritivos e autoexplicativos - evite abreviações
- Funções e componentes pequenos, respeitando o Princípio da Responsabilidade Única (SRP)
- Nenhum "magic number" solto - extrair em constantes nomeadas com nome descritivo
- **Implementação minimalista** - ao modificar um arquivo, altere apenas o estritamente necessário. Não faça alterações de refactoring não necessárias (indentação, espaçamento, reordenação de imports, etc.)
- Estrutura de arquivo dividida em seções comentadas:
  - Python: `# ============================================================\n# SECAO N: TITULO\n# ============================================================`
  - JavaScript/React: `// === COMPONENTE: NomeDoComponente ===`

## Python

- Type hints em todas as funções - tanto nos parâmetros quanto no retorno
- Comentários explicando o **porquê**, não só o quê
- Sem one-liners "espertos" - preferir código verboso e óbvio
- Exemplo correto:

```python
# Normaliza o texto para comparação sem distinção de acentos ou maiúsculas.
# Necessário porque os triggers podem ser escritos com ou sem acento pelo usuário.
def _normalize(text: str) -> str:
    return unicodedata.normalize("NFKD", text).encode("ascii", "ignore").decode("ascii").lower()
```

## React

- Componentes pequenos e focados - cada um com uma única responsabilidade
- Comentário antes de cada componente explicando o papel dele na UI
- Nomes de estado (`useState`) devem deixar claro o que representam:
  - Correto: `const [isLoadingResponse, setIsLoadingResponse] = useState(false)`
  - Evitar: `const [loading, setLoading] = useState(false)`
- Sem encadeamentos complexos de `.map().filter().reduce()` - quebrar em etapas com variáveis intermediárias nomeadas
- Exemplo correto:

```jsx
// ChatMessage: exibe uma única mensagem do histórico, com suporte a markdown e syntax highlight.
function ChatMessage({ role, content }) {
  const isAssistant = role === "assistant";
  const formattedContent = marked.parse(content);

  return (
    <div className={isAssistant ? "message-assistant" : "message-user"}>
      <div dangerouslySetInnerHTML={{ __html: formattedContent }} />
    </div>
  );
}
```
