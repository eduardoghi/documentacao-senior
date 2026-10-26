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

## Operadores `e` e `ou`

A LSP não tem avaliação de curto-circuito. Nos operadores `e` e `ou` os dois lados são sempre avaliados, mesmo quando o primeiro já decide o resultado. O compilador gera o código dos dois operandos e só depois a operação lógica, sem desvio entre eles.

Por isso o primeiro lado não serve de proteção para o segundo. Neste exemplo a divisão é feita mesmo com `nDivisor` igual a zero, e a regra dá erro de divisão por zero:

```lsp
se ((nDivisor <> 0) e (nValor / nDivisor > 1)) {
    ...
}
```

Use um `se` dentro do outro:

```lsp
se (nDivisor <> 0) {
    se (nValor / nDivisor > 1) {
        ...
    }
}
```

O mesmo vale para funções com efeito colateral, como `Mensagem`, e para leituras de cursor no segundo operando.

Para confirmar na sua versão, compile esta linha numa regra de teste. A mensagem aparece, apesar de `0 = 1` ser falso:

```lsp
se ((0 = 1) e (Mensagem(Retorna, "Lado direito avaliado [OK]") = 0)) {
}
```

### Precedência

O `e` é aplicado antes das comparações (`=`, `<>`, `<`, `>`, `<=` e `>=`), e o `ou` depois delas. Sem parênteses, `nA = 1 e nB = 2` não compara `nA` com 1 e `nB` com 2, porque o `e` junta o `1` e o `nB` primeiro. Já `nA = 1 ou nB = 2` funciona como esperado, mas a diferença entre os dois operadores é fácil de esquecer. Entre os dois, o `e` vem antes do `ou`, como na maioria das linguagens. Coloque cada comparação entre parênteses:

```lsp
se ((nA = 1) e (nB = 2)) {
    ...
}
```

Para confirmar na sua versão, compile o trecho abaixo. As duas mensagens aparecem. A primeira condição é lida como `0 = (1 e 0)`, que é verdadeira, e não como `(0 = 1) e 0`, que seria falsa. A segunda é lida como `(0 = 0) ou 1`, que é verdadeira, e não como `0 = (0 ou 1)`, que seria falsa.

```lsp
se (0 = 1 e 0) {
    Mensagem(Retorna, "e aplicado antes do =");
}

se (0 = 0 ou 1) {
    Mensagem(Retorna, "= aplicado antes do ou");
}
```

## Comparação de textos

A comparação entre valores alfa diferencia maiúsculas de minúsculas e considera os espaços à direita. Texto vazio também é diferente de um espaço. Nos três `se` abaixo a condição é falsa:

```lsp
aA = "abc";
aB = "ABC";
se (aA = aB) { ... }

aA = "A";
aB = "A ";
se (aA = aB) { ... }

aA = "";
aB = " ";
se (aA = aB) { ... }
```

Para ignorar a diferença de maiúsculas, converta os dois lados com `ConverteParaMaiusculo` antes de comparar.

O compilador não aceita comparar dois literais alfa. `se ("abc" = "ABC")` dá o erro "Variáveis alfanuméricas ou operadores não compatíveis não podem ser 2 constantes literais". Pelo menos um dos lados tem que ser variável ou campo.

## Variáveis declaradas dentro de função

Uma variável declarada com `definir` dentro de uma função não é zerada a cada chamada. Ela guarda o valor da chamada anterior, como uma variável global. Inicialize a variável no começo da função sempre que ela precisar começar do zero.

```lsp
definir numero nRes;
definir funcao Contar();

Contar();
Contar();
@ nRes vale 2 @

funcao Contar(); {
    definir numero nCont;
    nCont = nCont + 1;
    nRes = nCont;
}
```

## Recursão

Uma função pode chamar a si mesma. Os parâmetros são separados por chamada. Depois que a chamada interna volta, a função continua vendo os próprios valores. Já as variáveis declaradas com `definir` dentro da função são globais, como descrito acima, e a chamada interna pode sobrescrever o valor delas. Passe por parâmetro o que precisa sobreviver à chamada.

```lsp
definir numero nRes;
definir funcao Fatorial(numero n, numero end r);

Fatorial(3, nRes);
@ nRes vale 6 @

funcao Fatorial(numero n, numero end r); {
    definir numero nSub;
    definir numero nArg;
    se (n <= 1) {
        r = 1;
    } senao {
        nArg = n - 1;
        Fatorial(nArg, nSub);
        r = n * nSub;
    }
}
```

A pilha de chamadas tem 64 KB. Cada chamada ocupa 8 bytes, mais 10 por parâmetro `numero` e 4 por parâmetro `end`. Ao passar do limite, a regra para com o erro "Estouro da pilha de execução da regra". Uma função com um parâmetro `numero` aguenta 1.000 níveis, e com 10.000 a pilha estoura.

## Limites

### Tamanho do alfa

Uma variável alfa não tem o limite de 255 caracteres. Uma variável montada com 1.000 concatenações fica com 1.000 caracteres.

### Índice de tabela

O índice de uma `Tabela` começa em 1. Índice 0 ou maior que o tamanho declarado para a regra com o erro "Indexando fora dos limites". Índice com casas decimais é truncado, então `T[2.9]` acessa `T[2]`.

Índice negativo não é conferido. A regra não dá erro e acessa a memória antes da tabela, onde ficam as variáveis declaradas antes dela. No teste abaixo, `T[-1]` devolveu o valor de `nI`, e uma atribuição a `T[-1]` sobrescreve o `nI` sem nenhum aviso. Confira o índice antes de usar quando ele vier de uma conta:

```lsp
definir numero nA;
definir numero nI;
definir numero nTam;
definir Tabela T[3] = {Numero nVal;};

nI = -1;
nA = T[nI].nVal;
@ nA vale -1, o valor de nI @

T[nI].nVal = 77;
@ nI agora vale 77 @
```

### Precisão dos números

O tipo `numero` guarda o valor como ponto flutuante, mas toda operação aritmética (`+`, `-`, `*` e `/`) e toda comparação arredondam os operandos para 18 dígitos significativos antes de calcular. Por isso:

- `0.1 + 0.2 = 0.3` é verdadeiro, ao contrário do que acontece com ponto flutuante puro;
- `1 / 3 * 3 = 1` é falso, porque `1 / 3` vira 0,333333333333333333 e o resultado fica 0,999999999999999999;
- números com 19 dígitos ou mais perdem os últimos dígitos, e `nA = nA + 3` é verdadeiro com `nA` igual a 1234567890123456780.

Quando o resultado de uma divisão precisa ser comparado com um valor exato, arredonde antes de comparar:

```lsp
nC = nA / nB * nB;
Arredonda(nC, 2);
se (nC = 1) {
    ...
}
```

O arredondamento tem custo. Num laço de 100.000 voltas, trocar `nX = nY` por `nX = nY + nZ` levou o tempo de 422 ms para 827 ms, cerca de 4 microssegundos por soma. Só faz diferença em laços com centenas de milhares de contas.
