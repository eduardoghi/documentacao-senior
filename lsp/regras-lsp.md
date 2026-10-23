# Regras LSP

[Voltar para o README](../README.md)

Funcionamento das regras LSP do Senior Gestão Empresarial, com base no que foi observado na versão 5.10.4.

> [!IMPORTANT]
> Conteúdo baseado em testes e observação do comportamento do ERP, não em documentação oficial da Senior Sistemas. Confira na sua versão antes de depender dele.

---

## Funções

O código principal é executado de cima para baixo. O compilador coloca um "retorna" antes de cada implementação de `funcao`. Quando a execução chega nesse ponto sem que a função tenha sido chamada, a regra termina.

Na prática:

- o corpo de uma função só roda quando ela é chamada;
- o que estiver depois da primeira implementação de função nunca é executado, e nem a compilação nem a execução mostram erro.

Por isso o código principal deve ficar inteiro antes das implementações de função:

```lsp
definir numero nTotal;
definir funcao GerarLog();

nTotal = 0;
GerarLog();

funcao GerarLog(); {
    ...
}
```

Evite:

```lsp
definir numero nTotal;
definir funcao GerarLog();

GerarLog();

funcao GerarLog(); {
    ...
}

@ Este trecho nunca é executado @
nTotal = 0;
```

Para confirmar na sua versão, coloque uma `Mensagem` depois da última função de uma regra de teste. Ela não aparece.
