# Senior Middleware - JSON Facade Adapter

[Voltar para o README](../README.md)

> [!IMPORTANT]
> Esta documentação é não oficial. Foi montada a partir de testes práticos e da análise da implementação do adapter no Senior Middleware / Bridge G5.
>
> No Gestão Empresarial | ERP, o JSON Facade Adapter foi disponibilizado a partir da versão **5.10.4.120**, de **17/07/2026**, a mesma versão usada como base aqui. O comportamento pode variar em outras versões.
>
> Referência: [Notas da versão 5.10.4.120 - Senior](https://documentacao.senior.com.br/gestaoempresarialerp/notasdaversao/5-10-4/#MNTERP-48996)


## Objetivo

O JSON Facade Adapter permite executar serviços G5/Senior usando HTTP com corpo JSON, sem precisar montar manualmente uma requisição SOAP.

> [!IMPORTANT]
> O adapter atende corretamente parâmetros de **entrada escalares**. Estruturas aninhadas não são preservadas, e parâmetros do tipo `Set` podem ser [omitidos silenciosamente da requisição](#parâmetros-estruturados-na-entrada-não-chegam-ao-serviço), inclusive com a resposta indicando sucesso.
>
> A **saída** não tem essa restrição: respostas com estrutura profunda são convertidas corretamente.

Endpoint:

```text
POST http://SEU_HOST/g5-senior-services/api/v1/jsonfacadeadapter
```

Headers:

```http
Content-Type: application/json
Accept: application/json
Authorization: Basic ...
```

## Estrutura da requisição

```json
{
  "module": "sapiens",
  "service": "com.exemplo",
  "port": "Operacao",
  "execMode": "sync",
  "parameters": {
    "Parametro": "valor"
  }
}
```

| Campo | Obrigatório | Descrição |
|---|---|---|
| `module` | Sim | Módulo Senior, por exemplo `sapiens`. |
| `service` | Sim | Serviço G5. Aceita o nome puro ou o formato `servico@operacao`. |
| `port` | Condicional | Operação do serviço. Dispensável quando a operação já vem embutida em `service`. Se ambos forem informados, `port` prevalece. |
| `execMode` | Não | Modo de execução. Quando omitido, assume `sync`. |
| `parameters` | Sim | Parâmetros de entrada do serviço. Aceita objeto vazio (`{}`). |

> [!WARNING]
> A validação atua sobre o **valor** dos campos acima, não sobre a presença de campos extras.
>
> Um valor inválido é reclamado. `"execMode": "asyncc"` retorna `BAD_REQUEST` com a causa descrita na mensagem.
>
> Já um **nome** de campo grafado errado não é reclamado, porque propriedades desconhecidas na raiz do JSON são descartadas em silêncio. Confirmado em teste: `"exceMode": "async"` executa em modo `sync`, já que o `execMode` correto ficou ausente e assumiu o padrão.
>
> Confira a grafia dos nomes dos cinco campos, e não apenas os valores. Um erro ali não gera erro nenhum, só comportamento diferente do esperado.

### `service` e `port`

A operação pode ser informada de duas maneiras, e **as duas funcionam**.

Embutida em `service`, no formato `servico@operacao`. O adapter separa as partes e preenche `port` internamente:

```json
{
  "module": "sapiens",
  "service": "com.exemplo@FazConta",
  "execMode": "sync",
  "parameters": {
    "Number": 10
  }
}
```

Ou separada, com `service` sem o `@` e a operação em `port`:

```json
{
  "module": "sapiens",
  "service": "com.exemplo",
  "port": "FazConta",
  "execMode": "sync",
  "parameters": {
    "Number": 10
  }
}
```

Informar as duas coisas ao mesmo tempo, com `service` usando `@` e `port` repetindo a mesma operação, também funciona.

### Quando `service` e `port` divergem

Se os dois indicarem operações **diferentes**, prevalece o que estiver em `port`.

Confirmado em teste: com `FazConta3` e `FazConta` sendo operações distintas, a requisição abaixo executou o **`FazConta`**, indicado em `port`, e ignorou o `FazConta3` embutido em `service`:

```json
{
  "module": "sapiens",
  "service": "com.exemplo@FazConta3",
  "port": "FazConta",
  "execMode": "sync",
  "parameters": {
    "Number": 10
  }
}
```

O motivo é a ordem em que os campos são aplicados: a operação vinda do `@` é gravada primeiro e depois sobrescrita por `port`. É também o que faz a forma separada (`service` sem `@` mais `port`) funcionar.

> [!WARNING]
> Ainda que o comportamento seja previsível, evite informar operações diferentes nos dois campos. A requisição fica ambígua para quem lê e um erro de digitação no `@` passa despercebido, já que o `port` corrige silenciosamente.

### `execMode`

Pode ser omitido. A requisição abaixo, sem `execMode`, executou normalmente em modo síncrono:

```json
{
  "module": "sapiens",
  "service": "com.exemplo",
  "port": "FazConta",
  "parameters": {
    "Number": 10
  }
}
```

Omitir o campo é diferente de enviá-lo com valor inválido: um valor fora da lista aceita retorna erro de validação, descrito em [Erros HTTP](#erros-http).

## Autenticação

O adapter lê o header HTTP `Authorization`. Na implementação analisada existem quatro esquemas:

```http
Authorization: Basic BASE64(usuario:senha)
Authorization: Bearer TOKEN
Authorization: Encryption TOKEN
Authorization: Trusted TOKEN
```

> [!IMPORTANT]
> Os prefixos são **case-sensitive**. Use exatamente `Basic`, `Bearer`, `Encryption` e `Trusted`.

Detalhes observados na implementação:

- No `Basic`, o conteúdo é decodificado de Base64 e separado no **primeiro** `:`. Portanto a senha pode conter `:` sem escape. Por exemplo, `usuario:minha:senha:123` resulta em usuário `usuario` e senha `minha:senha:123`.
- O usuário não pode ser vazio. A senha não passa por validação equivalente nesse ponto.
- Header ausente, ou com um esquema que não seja um dos quatro acima, **não é rejeitado de imediato**: o processamento segue com credenciais vazias e a falha ocorre depois.

Para Basic Auth, o `curl` monta o header automaticamente a partir de:

```bash
--user 'SEU_USUARIO:SUA_SENHA'
```

> [!NOTE]
> Apenas o `Basic` foi testado em execução. Nos outros três, o adapter coloca o token no lugar da senha e deixa o usuário vazio.

### Credencial gravada na requisição

O Middleware persiste cada requisição na tabela `R960PAR`. O documento gravado carrega a credencial usada na chamada, como atributos do elemento `<request>`:

```xml
<request id='...' user='USUARIO' password='SENHA' encrypted='0'
         service='com.exemplo' port='FazConta'>
```

Esse atributo indica a versão de criptografia aplicada à credencial.

> [!NOTE]
> O campo tem dois nomes conforme onde é consultado. Na tabela de parâmetros da documentação da Senior ele é chamado de `encryption`. No XML `<request>`, tanto o gravado na `R960PAR` quanto os exemplos publicados pela Senior, o atributo é `encrypted`. É o mesmo campo.

Versões documentadas pela Senior:

| Valor | Significado | Disponível para terceiros |
|---|---|---|
| `0` | Sem criptografia. Usuário e senha ficam como texto UTF-8 no documento interno da requisição. | Sim |
| `1` | Senha cifrada com algoritmo próprio da Senior. | Não, apenas entre sistemas Senior |
| `2` | Em vez da senha, um token Senior de autenticação, obtido pelo Logon integrado com criptografia. | Sim |
| `3` | Em vez da senha, um token do serviço de usuários G7. | Não, apenas entre sistemas Senior |

Autenticando com `Basic`, o valor observado foi **`0`**, ou seja, senha em texto aberto.

> [!WARNING]
> Com `Basic`, a senha do usuário de integração fica **legível em texto puro** para qualquer pessoa com acesso de leitura à `R960PAR`.
>
> Leve isso em conta ao escolher qual usuário será usado nas integrações e ao conceder acesso ao banco. Prefira um usuário de serviço com permissões restritas, e não uma conta administrativa ou pessoal. Para não gravar a senha, use o esquema `Encryption`, descrito abaixo.

### Qual esquema usar

O adapter não expõe campo para escolher a versão de criptografia. Ela é definida pelo esquema do header `Authorization`:

| Header | Versão gerada | Como a credencial é enviada | Liberado para terceiros |
|---|---|---|---|
| `Basic` | `0` | Usuário e senha, ambos em texto aberto | Sim |
| `Encryption` | `2` | Usuário vazio, token no lugar da senha | Sim |
| `Bearer` | `3` | Usuário vazio, token da Plataforma Senior no lugar da senha | Não, apenas entre sistemas Senior |
| `Trusted` | `999` | Usuário vazio, token no lugar da senha | Não, uso interno para chamadas de métodos agendados |

Para sistemas terceiros, portanto, **`Encryption` é a única alternativa documentada ao texto aberto**. O `Bearer` gera a versão `3`, reservada pela Senior à integração entre sistemas próprios, e o `Trusted` é descrito como mecanismo interno para chamadas de métodos agendados. O valor `999` do `Trusted` não consta na documentação: foi identificado apenas na implementação analisada.

Referência: [Autenticação via Headers em Web Services](https://documentacao.senior.com.br/tecnologia/5.10.4/integracoes-com-outros-sistemas/web-services/autentica%C3%A7%C3%A3o-via-headers-em-web-services.htm)

O token do `Encryption` vem do Logon integrado com criptografia, que precisa ser habilitado em **Central de Configurações > Opções de Segurança**, onde também se define a chave usada para descriptografar. O conteúdo segue o formato abaixo e é gerado pelo aplicativo `crypt.jar`:

```text
DD/MM/YYYY HH:MM:SS|<nome do usuário>
```

> [!NOTE]
> O `Encryption` não foi testado em execução neste levantamento.
>
> A Senior alerta que o token tem tempo de expiração e que, nas execuções internas entre sistemas Senior, o mecanismo **só funciona em modo síncrono**: nos demais modos a execução falha com "Credenciais inválidas". Para sistemas terceiros, a administração do tempo de expiração é responsabilidade da integração.
>
> Valide esse comportamento antes de usar `Encryption` com `execMode: async`.

Referências: [Tipos de execução](https://documentacao.senior.com.br/tecnologia/5.10.4/integracoes-com-outros-sistemas/web-services/tipos_execucao.htm) e [Logon integrado com criptografia](https://documentacao.senior.com.br/tecnologia/5.10.4/central-de-configuracao/cfglogoncypherform.htm)

## Exemplo validado: `com.exemplo@FazConta`

Serviço usado no teste:

```text
Módulo:   sapiens
Serviço:  com.exemplo
Operação: FazConta
Entrada:  Number
Saída:    Resultado
```

Implementação do serviço:

```text
FazConta.Resultado = FazConta.Number * 4;
```

### Requisição JSON

```json
{
  "module": "sapiens",
  "service": "com.exemplo",
  "port": "FazConta",
  "execMode": "sync",
  "parameters": {
    "Number": 10
  }
}
```

### cURL

```bash
curl --location \
  'http://SEU_HOST/g5-senior-services/api/v1/jsonfacadeadapter' \
  --user 'SEU_USUARIO:SUA_SENHA' \
  --header 'Content-Type: application/json' \
  --header 'Accept: application/json' \
  --data '{
    "module": "sapiens",
    "service": "com.exemplo",
    "port": "FazConta",
    "execMode": "sync",
    "parameters": {
      "Number": 10
    }
  }'
```

### Resposta

Como o serviço multiplica `Number` por 4, para `Number = 10` o retorno observado foi:

```json
{
  "result": {
    "resultado": 40,
    "erroExecucao": null
  }
}
```

### `erroExecucao`

O status HTTP informa apenas se a chamada ao adapter foi processada. Tanto o erro do **serviço** quanto a falha de **autenticação** vêm no corpo da resposta, em `result.erroExecucao`.

> [!WARNING]
> Nos testes, o adapter respondeu **HTTP 200 mesmo com o serviço em erro**. Tratar a integração só pelo status HTTP faz o erro passar despercebido.

Quando não há erro, o campo vem como `null`, como na resposta de sucesso mostrada acima.

Os cenários abaixo foram provocados no `com.exemplo@FazConta`.

**Erro levantado pela regra**, com `Mensagem(Erro, "Number não pode ser zero")` e `"Number": 0`:

```json
{
  "result": {
    "resultado": 0,
    "erroExecucao": "Ocorreu um erro ao executar o serviço \" \": Number não pode ser zero"
  }
}
```

**Erro de execução**, trocando a implementação por `FazConta.Resultado = 100 / FazConta.Number;` e enviando `"Number": 0`:

```json
{
  "result": {
    "resultado": 0,
    "erroExecucao": "Ocorreu um erro ao executar o serviço \" \": Divisão por zero."
  }
}
```

**Falha de credencial** também chega por aqui, e não como erro HTTP de autorização. Usuário inexistente:

```json
{
  "result": {
    "resultado": 0,
    "erroExecucao": "Ocorreu um erro ao executar o serviço \" \": Credenciais inválidas."
  }
}
```

Senha incorreta:

```json
{
  "result": {
    "resultado": 0,
    "erroExecucao": "Ocorreu um erro ao executar o serviço \" \": Credenciais inválidas/desabilitadas/expiradas."
  }
}
```

> [!IMPORTANT]
> As duas respostas vêm com `HTTP 200`. Uma integração que trate autenticação apenas pelo status HTTP **nunca vai perceber que a credencial está errada**: vai receber `200`, ler o parâmetro de saída com o valor padrão do tipo e seguir como se tivesse dado certo.

Repare que a mensagem é diferente nos dois casos. Usuário inexistente responde `Credenciais inválidas.`, enquanto senha incorreta responde `Credenciais inválidas/desabilitadas/expiradas.`. Nos testes desta versão, portanto, foi possível distinguir os dois pela mensagem retornada.

Observações dos testes:

- `erroExecucao` é uma **string**, não um objeto. Não há código de erro separado: a mensagem do serviço vem concatenada ao prefixo `Ocorreu um erro ao executar o serviço "...":`.
- O nome do serviço dentro do prefixo não foi preenchido: veio literalmente como `" "`, um único espaço entre aspas, em todos os cenários testados. Não use esse trecho para identificar a origem do erro.
- A mensagem não é construída pelo adapter. O Bridge recebe esse conteúdo da camada de execução e o propaga para `erroExecucao`, sem preencher nem corrigir o nome do prefixo. A origem exata do texto não foi localizada na implementação analisada, então não trate esse formato como contrato estável nem faça leitura programática dele.
- Os parâmetros de saída retornam o **valor padrão do tipo**, não `null`. No exemplo, `resultado` veio `0` mesmo com o serviço em erro. Ou seja, `0` não significa sucesso.
- Erro levantado pela regra e erro de execução chegam no **mesmo formato**, ambos com `HTTP 200`. Não é possível distinguir a natureza do erro apenas pelo campo.

Por isso, em chamadas síncronas, valide sempre, nesta ordem:

1. o status HTTP;
2. se o nó `result` existe, já que em erro do adapter a resposta vem [em outro formato](#formatos-de-resposta), sem `result`;
3. se `result.erroExecucao` é `null`, **antes** de usar qualquer valor de saída.

### Maiúsculas e minúsculas nos campos

Na **entrada**, a caixa não importa. O nome enviado é comparado ao do contrato do serviço ignorando maiúsculas e minúsculas, então `codemp`, `CODEMP` e `codEmp` casam todos com o mesmo parâmetro.

Confirmado em teste: um serviço que declara `Number` como parâmetro de entrada respondeu normalmente ao receber `NUMBER`, com o cálculo correto. Se o nome tivesse sido descartado, o parâmetro de saída teria voltado com o valor padrão do tipo.

O elemento da requisição é reconstruído a partir do nome do contrato, com a normalização de nomes aplicada pelo Bridge, e não necessariamente com a grafia original.

A tolerância é apenas quanto à caixa. Qualquer outro caractere torna o nome diferente, e o parâmetro é descartado em silêncio:

```text
codemp    casa com codEmp
CodEmp    casa com codEmp

cod.emp   não casa
cod_emp   não casa
cod-emp   não casa
```

É por isso que as chaves com ponto, como `consulta.codSnf`, desaparecem sem aviso.

Na **saída** o comportamento é outro. Nas chaves de resposta, **apenas a letra inicial é convertida para minúscula**, e o restante do nome é preservado.

```text
Resultado                 ->  resultado
MeuResultadOO             ->  meuResultadOO
MensagemRetorno           ->  mensagemRetorno
TipoRetorno               ->  tipoRetorno
codEmp                    ->  codEmp
camposUsuarioOrcamento    ->  camposUsuarioOrcamento
```

Ou seja, um nome que já começa em minúscula chega intacto. Só os que começam com maiúscula mudam, e mudam somente no primeiro caractere.

O par `MeuResultadOO` veio de um serviço criado especificamente para isolar essa regra. As maiúsculas consecutivas no fim do nome descartam a hipótese de conversão para minúsculo integral, que produziria `meuresultadoo`.

`MensagemRetorno` e `TipoRetorno` confirmam o mesmo em serviço padrão da Senior: aparecem com essa grafia no registro binário da `R960PAR` e chegam ao JSON com a inicial minúscula e o resto preservado, mostrando os dois lados da conversão.

Na prática: na entrada não se preocupe com a caixa, apenas com a grafia exata do nome. Na resposta, considere que a inicial pode ter mudado.

## Parâmetros

Os parâmetros reconhecidos pelo contrato do serviço são reconstruídos como elementos de `<params>`, dentro do documento `<request>` que a Senior usa para acionar o serviço.

Este JSON:

```json
{
  "parameters": {
    "Number": 10
  }
}
```

resulta na estrutura abaixo, capturada da requisição efetivamente gravada pelo Middleware:

```xml
<request id='...' user='...' password='...' encrypted='0'
         service='com.exemplo' port='FazConta'>
  <params>
    <number>10</number>
  </params>
</request>
```

Dois pontos importantes nessa estrutura:

- O elemento é `<params>`, dentro de `<request>`. No envelope SOAP completo existe também um `<parameters>` em nível externo, que carrega esse documento inteiro. São camadas diferentes.
- O nome do parâmetro teve a **inicial convertida para minúscula**: `Number` virou `<number>`. A resolução funcionou mesmo assim. Veja [Maiúsculas e minúsculas nos campos](#maiúsculas-e-minúsculas-nos-campos).

Por isso, utilize os mesmos nomes definidos nos parâmetros do serviço.

Nos campos numéricos testados, o valor funcionou tanto como número JSON quanto como string. Enviar `"codEmp": 1` ou `"codEmp": "1"` produziu o mesmo resultado.

Para um serviço sem parâmetros de entrada, envie:

```json
{
  "parameters": {}
}
```

## Parâmetros estruturados na entrada não chegam ao serviço

> [!CAUTION]
> O JSON Facade não preserva objetos e listas como estruturas aninhadas. Quando o contrato do serviço espera um parâmetro do tipo `Set`, o valor acaba omitido da requisição final, mesmo que a chamada responda `HTTP 200`, com `erroExecucao: null` e mensagem de sucesso.
>
> Não use este endpoint para serviços cuja entrada exija estrutura aninhada.

Um objeto ou array enviado para um parâmetro declarado como `Set` não vira estrutura aninhada, e também não chega como texto achatado: ele é **descartado por completo**, sem gerar erro.

Foram tentadas quatro formas de enviar um parâmetro `Set`, e nenhuma funcionou: objeto aninhado, array, caminho achatado com ponto, e estrutura montada com os nomes exatos do contrato publicado. Para esses parâmetros, use o endpoint SOAP.

### Como isso foi verificado

O teste usou o serviço padrão [`com.senior.g5.co.mcm.ven.pedidos`](https://documentacao.senior.com.br/gestaoempresarialerp/5.10.4/webservices/com_senior_g5_co_mcm_ven_pedidos.htm), na porta `SimularPedidos`, que apenas calcula e não grava nada.

Foram feitas quatro chamadas:

1. Com um pedido completo, contendo oito produtos, cada um com sua própria lista de campos de usuário.
2. Sem o parâmetro `pedido`, enviando apenas parâmetros escalares.
3. Com o pedido completo, porém apontando para um código de cliente inexistente e com o tratamento de erros ligado.
4. Com um pedido mínimo, usando exatamente os nomes de campo do contrato publicado do serviço.

As quatro responderam exatamente a mesma coisa:

```json
{
  "result": {
    "mensagemRetorno": "Processado com sucesso.",
    "tipoRetorno": 0,
    "erroExecucao": null
  }
}
```

Uma simulação de pedido deveria devolver os valores calculados. Não devolveu em nenhuma delas.

A terceira chamada é a decisiva. Com o tratamento de erros ligado e um cliente que não existe, o serviço teria que recusar o pedido. Não recusou porque não recebeu pedido algum. A segunda mostra que a resposta é idêntica com ou sem o parâmetro complexo, ou seja, a mensagem de sucesso não diz nada sobre o conteúdo ter sido processado. E a quarta descarta a hipótese de erro de nomenclatura, já que usou os nomes exatos do contrato.

A requisição gravada confirma o efeito. O elemento de parâmetros veio **vazio**:

```xml
<request id='...' user='...' password='...' encrypted='0'
         service='com.senior.g5.co.mcm.ven.pedidos' port='SimularPedidos'><params/></request>
```

### Todas as formas testadas

Objeto e array se comportam da mesma maneira: a requisição é aceita, executa, e o parâmetro não chega ao serviço.

| Valor em `parameters` | Comportamento |
|---|---|
| Objeto, como `"consulta": { ... }` | Aceito. Omitido da requisição final. |
| Array, como `"consulta": [ { ... } ]` | Aceito. Omitido da requisição final. |

Também não funciona achatar o caminho com ponto, no formato que a própria documentação da Senior usa para nomear os campos de um `Set`. Enviando `"consulta.codSnf"` e `"consulta.numNfv"` junto de três parâmetros escalares, a requisição gravada saiu assim:

```xml
<params><codEmp>1</codEmp><codFil>1</codFil><tipExp>N</tipExp></params>
```

Os três escalares passaram. As duas chaves com ponto foram descartadas, e sequer viraram elementos com o nome literal.

O descarte acontece **chave por chave**, e independe da posição: repetindo o teste com a chave inválida em primeiro lugar, os escalares seguintes continuaram sendo transmitidos. Uma chave não reconhecida some sozinha, sem afetar as demais e sem gerar aviso.

O mesmo vale para valor objeto e para valor array. Enviando `"consulta"` das duas formas, ao lado dos três escalares, a requisição gravada saiu idêntica nos dois casos:

```xml
<params><codEmp>1</codEmp><codFil>1</codFil><tipExp>N</tipExp></params>
```

Não há elemento `<consulta>` algum, nem mesmo com o conteúdo achatado em texto. A resposta, nas duas formas, indicou processamento com sucesso.

### Por que não existe contorno

A conversão acontece em duas etapas, e a estrutura se perde na primeira.

Na etapa inicial, cada valor de `parameters` é transformado em texto. Um objeto vira algo como `{codSnf=01, numNfv=515555}` dentro do elemento correspondente.

Na etapa seguinte, o Middleware consulta o contrato do serviço, identifica que aquele parâmetro é do tipo `Set` e procura elementos XML filhos para remontar a estrutura. Como encontra apenas uma linha de texto, o conjunto resulta vazio e o parâmetro é omitido da requisição final. É por isso que ele não aparece nem achatado no que fica gravado.

Nenhuma variação de formato no JSON contorna isso, porque a estrutura já foi perdida antes de o Middleware tentar interpretá-la. Parâmetros do tipo `Set` só podem ser enviados pelo endpoint SOAP, onde o XML chega estruturado desde a origem.

### Consequência prática

O serviço executa com o parâmetro faltando, e nada na resposta indica isso.

No teste de consulta de baixas, o `Set` descartado era justamente o filtro da busca. A consulta rodou sem filtro algum e respondeu:

```json
{
  "result": {
    "mensagemRetorno": "Processado com sucesso.",
    "tipoRetorno": 1,
    "erroExecucao": null
  }
}
```

Nenhum título foi retornado, e o `tipoRetorno` igual a `1` significa "Processado" no contrato do serviço. O campo `erros` da resposta sequer apareceu. Mesmo com os campos do filtro sendo obrigatórios quando não se informa o número do título, não houve reclamação sobre a ausência deles.

> [!CAUTION]
> Uma integração escrita sobre este endpoint recebe `HTTP 200`, `erroExecucao: null` e mensagem de sucesso enquanto **o parâmetro não chegou e nada foi processado**. Uma consulta devolve vazio como se não houvesse registros, e uma operação não acontece como se tivesse acontecido.
>
> Para serviços que exigem estrutura aninhada na entrada, utilize o endpoint SOAP tradicional.

### Como detectar

Não existe sinal do descarte na resposta. E no fluxo analisado ele também **não gera registro no servidor**: o trecho que omite o parâmetro vazio não emite exceção, warning nem debug, e nos testes não apareceu nada correspondente no `server.log`.

Restam duas formas de perceber o problema, ambas do lado de quem integra:

- Conferir o `<params>` da requisição na Consulta de Requisições, comparando com o que foi enviado. É o método confiável, e o único que mostra exatamente o que o serviço recebeu.
- Validar o **conteúdo** da resposta, e não o status nem a mensagem. Numa consulta, tratar retorno vazio como suspeito em vez de conclusivo, já que a ausência de registros e o descarte do filtro produzem exatamente a mesma resposta.


> [!NOTE]
> O problema é de **entrada**. A conversão da resposta funciona corretamente, inclusive com estruturas profundas: veja [Resposta com estrutura complexa](#resposta-com-estrutura-complexa).

## Resposta com estrutura complexa

Serviços padrão da Senior costumam devolver estruturas profundas, com listas dentro de listas. O adapter converte essas estruturas corretamente, gerando objetos e arrays JSON de verdade, sem achatar nada em texto.

O exemplo abaixo usa o serviço padrão [`com.senior.g5.co.mcm.ven.orcamento`](https://documentacao.senior.com.br/gestaoempresarialerp/5.10.4/webservices/com_senior_g5_co_mcm_ven_orcamento.htm), na porta `CarregarOrcamentos`:

```json
{
  "module": "sapiens",
  "service": "com.senior.g5.co.mcm.ven.orcamento",
  "port": "CarregarOrcamentos",
  "execMode": "sync",
  "parameters": {
    "codEmp": 1,
    "numOct": 1,
    "verOct": 1
  }
}
```

A resposta, com os valores substituídos por dados de exemplo e reduzida para mostrar apenas a estrutura:

```json
{
  "result": {
    "mensagemRetorno": "Processado com sucesso.",
    "tipoRetorno": 0,
    "erroExecucao": null,
    "orcamento": {
      "codEmp": 1,
      "numOct": 1,
      "vlrLiq": 1000.00,
      "datGer": "01/01/2020",
      "vldOct": null,
      "itens": {
        "seqOci": 1,
        "codPro": "PRODUTO01",
        "desPro": "Produto de exemplo",
        "qtdOci": 10.0,
        "vlrUni": 100.00
      },
      "camposUsuarioOrcamento": [
        { "campo": "USU_CAMPO1", "valor": 1 },
        { "campo": "USU_CAMPO2", "valor": 0 },
        { "campo": "USU_CAMPO3", "valor": null }
      ]
    }
  }
}
```

Os tipos são preservados: números chegam como número, `null` como `null`, e datas como texto no formato `DD/MM/YYYY`.

Serviços padrão da Senior costumam trazer também `mensagemRetorno` e `tipoRetorno` ao lado do `erroExecucao`.

### Coleção com um item vira objeto, com vários vira array

Este é o ponto mais importante da conversão, e o mais fácil de não perceber.

No contrato SOAP, `itens` e `camposUsuarioOrcamento` são ambos coleções. Na resposta acima:

| Campo | Quantidade de elementos | Tipo no JSON |
|---|---|---|
| `itens` | 1 | **objeto** |
| `camposUsuarioOrcamento` | 3 | **array** |

A diferença não está no contrato, e sim na quantidade de registros retornados naquela chamada. Em SOAP, um elemento repetido uma vez e repetido três vezes são a mesma estrutura. Na conversão para JSON deixam de ser.

Isso foi confirmado com o **mesmo campo, no mesmo serviço**: consultando um orçamento com um único item, `itens` veio como objeto; consultando outro orçamento com vários itens, `itens` veio como array. Nada mudou na chamada além do registro consultado.

> [!WARNING]
> Uma integração que percorra `orcamento.itens` como lista funciona enquanto o orçamento tiver vários itens e **quebra quando tiver apenas um**. O inverso também vale: código que trate `itens` como objeto quebra assim que vier o segundo item.
>
> Antes de percorrer qualquer coleção da resposta, verifique se o valor é array ou objeto e normalize. Não confie no que apareceu no teste, porque a forma depende dos dados daquele registro.

## Modos de execução

Foram identificados os modos:

```text
sync
async
scheduled
```

O campo é opcional. Quando omitido, o adapter assume `sync`, que é o modo em que o cliente aguarda o resultado na própria resposta:

```json
{
  "execMode": "sync"
}
```

O Web Service em si possui um quarto modo, o **Local**, que processa na mesma instância da aplicação, dentro do próprio sistema, sem passar pelo Middleware. Ele não está disponível no adapter: a mensagem de erro de validação cita apenas "Sincrono, Assincrono ou agendado".

> [!NOTE]
> A mensagem de erro do adapter cita os modos como "Sincrono, Assincrono ou agendado", mas o valor aceito no JSON é o identificador em minúsculo, como `sync` e `async`.

Referências: [Tipos de execução](https://documentacao.senior.com.br/tecnologia/5.10.4/integracoes-com-outros-sistemas/web-services/tipos_execucao.htm) e [TECNOLOGIA - WebServices - Modos de execução](https://suporte.senior.com.br/hc/pt-br/articles/11915260541332-TECNOLOGIA-WebServices-Quais-s%C3%A3o-as-formas-m%C3%A9todos-modos-de-execu%C3%A7%C3%A3o-dispon%C3%ADveis-para-acionamento-de-um-WebService-S%C3%ADncrono-Ass%C3%ADncrono-Local-e-Agendado)

### `async`

No modo assíncrono o adapter confirma o recebimento e devolve o controle imediatamente. Não há dados do serviço na resposta:

```json
{
  "result": "ok"
}
```

> [!WARNING]
> Repare que `result` aqui é uma **string**, e não um objeto. Código que lê `result.erroExecucao` direto funciona no modo `sync` e quebra no `async`.

A resposta **não devolve nenhum identificador da requisição**. Para acompanhar a execução é preciso consultar a tabela de requisições e localizar o registro por serviço, usuário e horário.

Colunas observadas em uma execução assíncrona bem-sucedida:

| Coluna | Valor observado | Observação |
|---|---|---|
| `IDREQ` | string opaca de 20 caracteres | Identificador da requisição. Inclui caracteres não alfanuméricos, como `[`, `/` e `!`. Não é devolvido na resposta HTTP. |
| `TIPSER` | `com.exemplo@FazConta` | Gravado sempre no formato `servico@operacao`, mesmo que a requisição tenha enviado `service` e `port` separados. |
| `STATUS` | `4` | Estado final da requisição. |
| `PERCON` | `100` | Percentual concluído. |
| `DATINI` / `DATFIN` | mesmo horário | A execução foi imediata. |
| `CODUSU` | código do usuário | Usuário autenticado na chamada. |

O conteúdo da chamada fica na `R960PAR`, ligada pelo mesmo `IDREQ`. A coluna `TIPPAR` separa o que entrou do que saiu:

| `TIPPAR` | Conteúdo do `PARVAL` |
|---|---|
| `0` | Entrada: o documento `<request>` completo, incluindo os parâmetros enviados e a [credencial usada na chamada](#credencial-gravada-na-requisição). |
| `1` | Saída: o retorno do serviço, com os parâmetros de saída. |

O `PARVAL` é um BLOB, e os dois lados usam formatos diferentes.

A entrada é **XML**, legível assim que o BLOB é convertido para texto.

A saída **não é XML**. É um formato binário próprio, em que os nomes dos campos aparecem legíveis em ASCII entre bytes de controle. Em uma consulta de baixa de títulos, por exemplo, aparecem `BaixaTitulos`, `Erros`, `TipoRetorno` e `MensagemRetorno`, junto do texto da mensagem de retorno.

> [!NOTE]
> Recuperar o resultado de uma chamada assíncrona pela tabela exige interpretar esse formato binário. Não basta ler o BLOB como texto.
>
> Repare também que ali os nomes vêm capitalizados, como `MensagemRetorno`. É a mesma diferença descrita em [Maiúsculas e minúsculas nos campos](#maiúsculas-e-minúsculas-nos-campos): a resposta JSON entrega `mensagemRetorno`, com a inicial em minúscula.

Ambas as tabelas são consultáveis pela Consulta de Requisições.

### `scheduled`

> [!WARNING]
> O valor `scheduled` é aceito pela validação, mas **não é possível criar nem alterar agendamentos por este endpoint**. Toda tentativa de criação ou alteração retorna erro interno. A operação de [listar os agendamentos existentes](#listar-agendamentos), porém, funciona.

No modelo de execução agendada da Senior, a solicitação é armazenada no WildFly para execução posterior pelo Middleware, sem retorno da execução para quem solicitou. Os agendamentos são geridos pelo web service `ScheduledService`.

O agendamento exige metadados próprios, como operação, periodicidade, intervalo, data e hora iniciais. Eles estão especificados pela Senior em [Tipos de execução](https://documentacao.senior.com.br/tecnologia/5.10.4/integracoes-com-outros-sistemas/web-services/tipos_execucao.htm), e ficam **ao lado** de `parameters`, não dentro dele.

A estrutura da requisição JSON não possui campo para nenhum deles: só existem `module`, `service`, `port`, `execMode` e `parameters`.

Foram testadas as duas formas possíveis de tentar enviá-los, usando exatamente os nomes documentados acima, e ambas deram o mesmo resultado:

1. Dentro de `parameters`. Não funciona, porque tudo que está ali vira parâmetro do serviço dentro de `<params>`, e não metadado de agendamento.
2. Na raiz do JSON, ao lado de `module` e `service`. Também não funciona, porque campos desconhecidos são descartados.

Nos dois casos a resposta foi:

```json
{
    "message": "Internal server error.",
    "path": "/g5-senior-services/api/v1/jsonfacadeadapter",
    "internalCode": "INTERNAL_ERROR"
}
```

No `server.log`, o erro correspondente é um `NullPointerException` ao ler a operação de agendamento, que permaneceu nula:

```text
java.lang.NullPointerException: Cannot invoke "java.lang.Integer.intValue()"
because the return value of
"br.com.senior.bridge.model.ScheduledModel.getOperation()" is null
```

**Para agendar a execução de um serviço, utilize o endpoint SOAP tradicional**, onde os campos de agendamento têm lugar previsto no envelope.

#### Listar agendamentos

Apesar de não criar agendamentos, o modo `scheduled` dá acesso a uma operação de consulta. Informando `scheduledProcesses` em `port`, o adapter retorna a lista de processos agendados:

```json
{
  "module": "sapiens",
  "service": "com.exemplo",
  "port": "scheduledProcesses",
  "execMode": "scheduled",
  "parameters": {}
}
```

O `service` precisa existir, senão a chamada retorna `INTERNAL_ERROR` como qualquer serviço inválido. Fora isso, a operação não depende dos metadados de agendamento, e é por isso que funciona.

A resposta traz o `result` como **string contendo XML**, e não como objeto JSON:

```json
{
  "result": "<SERVICE></SERVICE>"
}
```

O exemplo acima é de um ambiente sem agendamentos. O formato do conteúdo com a lista preenchida não foi observado.

#### Cancelar agendamentos

Existe também a porta `cancel`, que remove um agendamento a partir do seu identificador.

> [!WARNING]
> Não utilize `cancel` por este endpoint. O identificador do agendamento precisaria chegar como metadado, do mesmo modo que os campos de criação, e o contrato JSON não tem onde colocá-lo. A operação é alcançada sem receber o alvo, e o comportamento nesse caso não foi verificado.

## Erros HTTP

| HTTP | `internalCode` | Causa | Observado em teste |
|---|---|---|---|
| `400` | `BAD_REQUEST` | Campo obrigatório ausente ou com valor inválido. Confirmado com `execMode` fora da lista aceita. | Sim |
| `500` | `INTERNAL_ERROR` | **Serviço inexistente**, erro interno no Bridge ou falha na conversão da requisição/resposta. | Sim |
| varia | nenhum | Rota inexistente ou corpo malformado. Responde em [outro formato](#erros-com-mensagem-e-error), sem `internalCode`. | Sim |

> [!IMPORTANT]
> Credencial inválida **não** aparece nesta tabela. Nos cenários testados, usuário ou senha errados retornaram `HTTP 200`, com a falha dentro de [`result.erroExecucao`](#erroexecucao), e não houve resposta `401`.

Esta tabela cobre falhas de **transporte e validação**, antes ou fora da execução da regra. Erro do próprio serviço não aparece aqui: retorna `HTTP 200` com [`result.erroExecucao`](#erroexecucao) preenchido.

> [!NOTE]
> Header `Authorization` malformado, como Base64 inválido no `Basic`, não foi testado. A leitura da implementação sugere que ele cai no tratamento genérico de erro, e não no de autorização.
>
> Em nenhum teste realizado o adapter respondeu `401`.

### Corpo do erro do adapter

Quando a falha é do adapter, e não do serviço, a resposta **não traz o nó `result`**. O formato é outro, e o campo `internalCode` identifica a natureza do erro.

**Serviço inexistente**, usando `com.exemploo@FazConta` com um `o` a mais:

```json
{
    "message": "Internal server error.",
    "path": "/g5-senior-services/api/v1/jsonfacadeadapter",
    "timestamp": "2026-08-10T14:50:56.205721500-03:00",
    "internalCode": "INTERNAL_ERROR"
}
```

Nome de serviço errado **não** é tratado como erro de validação da requisição: a validação inicial apenas confere se o campo foi informado, e a falha só acontece depois, ao resolver o serviço.

**Campo com valor inválido**, usando `"execMode": "xyz"`:

```json
{
    "message": "O modo de execução do serviço é inválido. Verifique se está utilizando o modo de execução: Sincrono, Assincrono ou agendado.",
    "path": "/g5-senior-services/api/v1/jsonfacadeadapter",
    "timestamp": "2026-08-10T15:05:55.987066800-03:00",
    "internalCode": "BAD_REQUEST"
}
```

Note a diferença: em `BAD_REQUEST` o campo `message` traz a causa real, enquanto em `INTERNAL_ERROR` vem apenas `"Internal server error."`, sem detalhe. Para diagnosticar, use o `internalCode`, não o texto de `message`.

### Formatos de resposta

São sete situações distintas, produzindo quatro estruturas de corpo. A integração precisa lidar com todas:

| Situação | HTTP | Corpo |
|---|---|---|
| Serviço executou com sucesso (`sync`) | `200` | `result` como **objeto**, com `erroExecucao: null` |
| Serviço executou e falhou (`sync`) | `200` | `result` como **objeto**, com `erroExecucao` preenchido |
| Credencial inválida | `200` | `result` como **objeto**, com o erro em `erroExecucao` |
| Chamada aceita (`async`) | `200` | `result` como a **string** `"ok"`, sem dados do serviço |
| Listagem de agendamentos | `200` | `result` como **string contendo XML**, sem conversão para JSON |
| Adapter falhou | `400` ou `500` | **sem `result`**; traz `message`, `path`, `timestamp` e `internalCode` |
| Falha antes do adapter, como rota inexistente ou corpo malformado | erro | **sem `result`**; traz apenas `mensagem` e `error` |

> [!WARNING]
> O nó `result` muda de tipo conforme o caso. Ler `result.erroExecucao` diretamente só é válido quando `result` é um **objeto**, o que acontece nas três primeiras linhas da tabela. No `async` e na listagem de agendamentos ele é uma string, e nos erros do adapter o nó nem existe.
>
> Verifique o status HTTP e o tipo de `result` antes de acessar qualquer campo.

### Erros com `mensagem` e `error`

Existe um terceiro envelope de erro, com os campos `mensagem` e `error`. Ele não traz `result`, e também não traz `path`, `timestamp` nem `internalCode`. Aparece quando a falha ocorre **antes** do adapter processar a requisição.

Rota inexistente, observado ao chamar `/api/v1/jsonfacadeadapterr` com um `r` a mais no final:

```json
{
    "mensagem": "Consulte os logs para detalhes.",
    "error": "Erro na requisição."
}
```

Corpo da requisição malformado:

```json
{
    "mensagem": "Verifique se há vírgulas, aspas, chaves ou caracteres inconsistentes na requisição.",
    "error": "Erro na estrutura do JSON enviado."
}
```

Esse formato indica falha anterior ao processamento normal da requisição. Nos testes, apareceu com URL inexistente e com corpo malformado, e vale conferir os dois antes de investigar o serviço ou as credenciais.
