# documentacao-senior
Documentação dos sistemas Senior — funcionalidades dos produtos, procedimentos e resolução de problemas.

> [!IMPORTANT]
> **Aviso legal**  
> 
> Este repositório é um projeto **pessoal e independente**, mantido por uma única pessoa, **sem qualquer vínculo, associação, patrocínio, parceria comercial ou endosso** da **Senior Sistemas**.
>  
> Todo o conteúdo aqui publicado tem **finalidade exclusivamente informativa** e foi elaborado com base em experiências e referências públicas. **Não representa documentação oficial** da Senior Sistemas nem substitui materiais, contratos, garantias ou suporte do fabricante.  
>  
> **Marcas, nomes e logotipos** eventualmente mencionados são de propriedade de seus **respectivos titulares**. O uso nominativo serve apenas para referência e **não implica afiliação**.  
>  
> O material é fornecido **"no estado em que se encontra"**, **sem garantias** de qualquer tipo, explícitas ou implícitas. O uso é por sua conta e risco. Para orientações oficiais, consulte os canais e a documentação da Senior Sistemas.


> [!IMPORTANT]
> Todos os exemplos e consultas SQL relacionados ao ERP neste repositório foram escritos para **Oracle** (PL/SQL).  
> Se você utiliza outro tipo de banco de dados, adapte a sintaxe antes de executar.

---

## Índice
- [ERP](#erp)
  - [Erros e Situações](#erros-e-situações)
- [SDE](#sde)
- [WMS Silt](#wms-silt)
  - [Procedimentos](#procedimentos-wms)
  - [Logs](#logs)
- [Integrações](#integrações)
  - [ERP-WMS](#erp-wms)

---

## ERP

### Erros e Situações

---
#### Situação: Como saber quem está logado no sistema e qual é o código de usuário?

Para listar as sessões e vincular cada login ao código de usuário:

```sqlpl
SELECT
    r911sec.*,
    r910ent.codent
FROM
    r911sec
    LEFT JOIN r910ent
        ON r910ent.nomexb = r911sec.appusr
```

r911sec registra as sessões (campo appusr = usuário que abriu a sessão no sistema).  
r910ent guarda o cadastro do usuário (campo codent = código do usuário e nomexb = login).

---

#### Situação: Pré-fatura gerou com número 0:

<img width="362" height="95" alt="Screenshot" src="https://github.com/user-attachments/assets/769cbdde-bfa2-44fe-a6e5-9743b292bb83" />

Motivo: desconhecido.

Solução: excluir a pré-fatura e refazer. Para evitar recorrência, utilizar o identificador de regra `COM-135CGFCA02` para **impedir a geração da pré-fatura quando `numpfa = 0`**.

```lsp
definir numero VComNumAne;
definir numero VComNumPfa;

definir alfa VComOperacao;
se ((VComOperacao = "CONSISTINDO") e (VComNumPfa = 0)) {
    definir alfa aNumAne;
    IntParaAlfa(VComNumAne, aNumAne);
    
    definir alfa aMsg;
    aMsg = "Não foi possível gerar a pré-fatura da carga " + aNumAne + ". Número da pré-fatura = 0";

    Mensagem(Erro, aMsg);
}
```
Obs: Na versão [5.10.4.75](https://documentacao.senior.com.br/gestaoempresarialerp/notasdaversao/#5-10-4.htm#813105) a Senior reporta a correção desse comportamento. Ainda assim, mantenha o identificador de regra ativo como salvaguarda.

---

#### Situação: Na tela “Transferências de Produtos” (F210TPA — Suprimentos → Gestão de Estoques → Controle de Estoque → Transferência) a seguinte mensagem aparece ao tentar transferir estoque:

"As seguintes transferências não foram realizadas, pois os depósitos nos quais os produtos estão presentes ou que irão ser transferidos integram com o WMS."
<img width="1104" height="187" alt="image" src="https://github.com/user-attachments/assets/f184c53b-1aad-42af-8247-eee2ba8cf47e" />

Motivo: a partir da versão 5.10.4.72 do ERP foi incluído um bloqueio para impedir transferências quando o depósito integra com o WMS. (verificação do campo e205dep.intwms)

<img width="1119" height="233" alt="image" src="https://github.com/user-attachments/assets/a088da06-e9fd-4ada-a994-8abefd82d992" />


Solução: se você tem certeza do que está fazendo e tem acesso ao banco como DBA, pode seguir o procedimento abaixo para executar a transferência.

O sistema, para exibir essa mensagem, verifica o valor da coluna e205dep.intwms do depósito. Assim, é possível forçar a transferência sem alterar esse valor na tabela original, usando um schema auxiliar:

1 -  Criar schema auxiliar no banco.
```sqlpl
CREATE USER maskvalor IDENTIFIED BY "Senha123123";
```

2 - Conceder permissões para o novo schema logar, criar views e fazer SELECT na tabela e205dep.
```sqlpl
GRANT CREATE SESSION, CREATE VIEW TO maskvalor;
GRANT SELECT ON schemaerp.e205dep TO maskvalor;
```

3 - Criar uma VIEW no schema auxiliar que lê e205dep, mas retorna 'N' em intwms conforme a condição.
```sqlpl
CREATE OR REPLACE
VIEW maskvalor.e205dep AS
SELECT
    codemp, coddep, desdep, abrdep, tipdep, codmde, mskdep, nivdep, qtdpos, pribus, cpmdep, lardep, altdep, cappes, capvol, codfil, depcpr, estneg, disxpl, indclt, codccu,
    crirat, ctared, ctarcr, ctafdv, ctafcr, sitdep, obsdep, seqdep, geremb, conaep, depven, depiql, invemb,
    CASE
        WHEN coddep = 'WMS' THEN 'N'
        ELSE intwms
    END AS intwms,
    msgesn, usu_codtns, usu_datzer, usu_usuzer, depvir, criord, intwmw, intpos, codimr, sitwmw
FROM
    schemaerp.e205dep;
```

4 - Criar sinônimos no schema auxiliar para os objetos do schema do ERP, exceto e205dep.
```sqlpl
BEGIN
    FOR r IN (
        SELECT
            object_name,
            object_type
        FROM
            dba_objects
        WHERE
            owner = 'SCHEMAERP'
            AND object_type IN ('TABLE', 'VIEW', 'SEQUENCE', 'PACKAGE', 'FUNCTION', 'PROCEDURE', 'TYPE', 'SYNONYM')
            AND object_name <> 'E205DEP'
    ) LOOP
        BEGIN
            EXECUTE IMMEDIATE
                'CREATE OR REPLACE SYNONYM maskvalor.' || r.object_name || ' FOR schemaerp.' || r.object_name;
        EXCEPTION
            WHEN OTHERS THEN
                NULL;
        END;
    END LOOP;
END;
```

5 - Criar uma trigger para, ao logar no banco a partir do seu computador, mudar o CURRENT_SCHEMA para o schema auxiliar.
```sqlpl
CREATE OR REPLACE TRIGGER trg_maskvalor_current_schema
AFTER LOGON ON SCHEMA
BEGIN
    IF UPPER(SYS_CONTEXT('USERENV', 'HOST')) LIKE '%EDUARDOG' THEN
        EXECUTE IMMEDIATE 'ALTER SESSION SET CURRENT_SCHEMA = maskvalor';
    END IF;
END;
```

Agora, abra o ERP e tente a transferência: a mensagem não deve mais aparecer, pois o valor de intwms retornará 'N' pela view criada (e não pela tabela original). Assim, você executa a transferência sem alterar a tabela e sem impactar outros usuários.

Após concluir a transferência, desative a trigger e reabra o sistema para evitar que ações futuras em seu ERP usem o valor mascarado de intwms e impeçam a integração com o WMS por engano.

---

## SDE

### Posso alterar coisas no schema do banco do SDE?

#### Não.
Não devem ser criados ou alterados objetos no schema do SDE (por exemplo: *views*, tabelas, triggers, procedures).
No processo de atualização, o sistema valida/consiste a estrutura do banco; qualquer objeto extra ou modificação pode causar falhas problemas.

---

### Onde posso ver o dicionário de dados do SDE?

Conforme citado na [Documentação da Senior](https://suporte.senior.com.br/hc/pt-br/articles/4408629051540-EDOCS-Banco-de-Dados-Existe-algum-dicion%C3%A1rio-de-Banco-de-Dados-do-eDocs), não há um dicionário de dados público do banco do SDE, pois a Senior não recomenda intervenções diretas no banco de dados.

Ainda assim, para consultas (*SELECT*) e análises, seguem algumas tabelas e colunas úteis:

---

### **N130NFE** — Informações gerais das Notas Fiscais

* `seqnfe` — Sequência da NF-e

* `seqfil` — Sequência da filial

* `sequfd` — Sequência da UF do destinatário

* `sequfe` — Sequência da UF do emitente

* `sitnfe` — Situação da Nota Fiscal

  * `1` — Recebida
  * `4` — Enviada
  * `6` — Autorizada
  * `7` — Denegada
  * `8` — Rejeitada
  * `9` — Cancelada
  * `13` — Inutilizada
  * `41` — Registrado DPEC
  * `56` — Cancelada fora do prazo
  * `57` — Autorizada fora do prazo
  * `92` — Manifestação gerada automaticamente
  * `93` — Falha
  * `94` — Preparado para envio à SEFAZ
  * `95` — Aguardando saída da contingência
  * `96` — Impresso em contingência
  * `97` — Registrado EPEC (cliente remoto)

* `sitret` — Situação do Retorno (SDE → ERP)

  * `1` — Não retornado
  * `2` — Retornado
  * `3` — Erro no retorno
  * `4` — Desativado

* `sitimp` — Situação da impressão do DANFE

  * `1` — Pendente
  * `2` — Impressa
  * `3` — Desativado
  * `4` — Impresso em contingência
  * `5` — Impressão temporária

* `vernfe` — Versão do layout da NF-e

* `numnfe` — Número da nota fiscal

* `sernfe` — Série da nota fiscal

* `totnfe` — Valor total da nota fiscal

* `datemi` — Data/hora de emissão

* `datets` — Data/hora de entrada/saída

* `doctip` — Tipo do documento do destinatário

  * `1` — CNPJ
  * `2` — CPF
  * `3` — Estrangeiro

* `docdes` — Documento do destinatário (CNPJ/CPF)

* `nomdes` — Nome do destinatário

* `datrec` — Data/hora de recebimento

* `prorec` — Protocolo de autorização de uso

* `idenfe` — Chave de Acesso (SDE)

* `ideerp` — Chave de Acesso (ERP)

* `tippro` — Tipo do processamento

  * `E` — Emissão
  * `R` — Recebimento

* `datcon` — Data/hora da consulta na SEFAZ

* `sitant` — Situação anterior do documento

* `cnpemi` — CNPJ do emitente

* `nomemi` — Nome do emitente

* `tipemi` — Tipo de emissão da NF-e

  * `1` — Normal (emissão normal)
  * `2` — Contingência FS (Formulário de Segurança)
  * `3` — Contingência SCAN (Sistema de Contingência do Ambiente Nacional)
  * `4` — Contingência DPEC (Declaração Prévia de Emissão em Contingência)
  * `5` — Contingência FS-DA (Formulário de Segurança para Impressão do DANFE)
  * `6` — Contingência SVC-AN (SEFAZ Virtual do Ambiente Nacional)
  * `7` — Contingência SVC-RS (SEFAZ Virtual do RS)

* `toticm` — Valor do ICMS

* `subtri` — Valor do ICMS-ST

* `ieemis` — Inscrição estadual do emitente

* `datctg` — Data/hora da contingência

* `tipope` — ?

* `iedest` — Inscrição estadual do destinatário

* `fusctg` — Fuso horário da contingência

* `fusemi` — Fuso horário da emissão

* `fusrec` — Fuso horário do recebimento

* `cameml` — Caminho do arquivo EML

* `digval` — Digest Value da NF-e

* `motctg` — Motivo da contingência

* `tiprel` — ?

* `tipnfe` — ?

* `ideger` — ?

* `sitaud` — Situação da auditoria do documento

* `datpro` — ?

* `datimp` — ?

* `tipamb` — Tipo de ambiente da NF-e

  * `1` — Produção
  * `2` — Homologação

* `verori` — Versão do aplicativo/processo emissor

* `tpdemi` — ?
  
* `sitman` — Situação da manifestação do destinatário da NF-e (código do evento SEFAZ)

  * `210200` — Confirmação da Operação
  * `210210` — Ciência da Operação
  * `210220` — Desconhecimento da Operação
  * `210240` — Operação não Realizada

* `codsfz` —  ?

* `msgsfz` —  ?

* `sitene` —  ?

* `datman` — Data da manifestação do destinatário

* `emialf` — Identificador alfanumérico do emitente (CNPJ em formato texto)

---

### **N130XML** — Ligação entre as Notas Fiscais e o XML

* `seqnxm` — Sequência/ID do registro

* `seqxml` — Sequência do XML

* `seqnf3` — Sequência de NF3-e

* `hasarq` — Hash do arquivo XML

* `datsal` — Data/hora em que o arquivo XML foi salvo

* `tiparq` — Tipo de arquivo do XML

  * `1` — Emissão de NF-e
  * `2` — Cancelamento de NF-e
  * `3` — Cancelamento de evento
  * `4` — Manifestação do destinatário
  * `5` — Carta de correção
  * `6` — Evento de EPEC NF-e
  * `7` — Inutilização
  * `8` — Pedido de prorrogação de suspensão do ICMS
  * `9` — Cancelamento de pedido de prorrogação de suspensão do ICMS
  * `10` — Resposta do Fisco a um pedido de prorrogação
  * `11` — Resposta do Fisco a um cancelamento de pedido de prorrogação

* `seqinu` — Sequência de inutilização
* `envdat` — ?
* `seqrgs` — ?
* `seqcrs` — ?

---

### **N100XML** — Arquivos XML

* `seqxml` — Sequência/ID do XML
* `arqxml` — ?
* `binarq` — Conteúdo do XML

---

### **N100EST** — Estados

* `seqest` — Sequência do estado
* `codpai` — Código do país
* `sigest` — Sigla do estado
* `nomest` — Nome do estado
* `utcest` — Fuso horário (UTC)
* `utcver` — Fuso horário (UTC) - horário de verão

---

### **N100PAI** — Países

* `codpai` — Código do país
* `dscpai` — Descrição do país
* `codiso` — Código ISO numérico do país (ISO 3166-1 numeric)
* `alpcod` — Código ISO alfabético (2 letras) do país (ISO 3166-1 alpha-2)

---

### **N100EMP** — Empresas

* `seqemp` — Sequência da empresa
* `seremp` — Serial da empresa (identifica o cliente no sistema de licenciamento)
* `tipdoc` — Tipo do documento da empresa

  * `1` — CNPJ
  * `2` — CPF

* `nomemp` — Nome da empresa
* `cnpemp` — CNPJ da empresa

---

### **N100FIL** — Filiais

* `seqfil` — Sequência da filial (CNPJ)
* `seqemp` — Sequência da empresa
* `seqmun` — Sequência do município
* `seqest` — Sequência do estado
* `vernhi` — ?
* `tipdoc` — Tipo do documento da filial

  * `1` — CNPJ
  * `2` — CPF

* `sitfil` — Situação da filial

  * `0` — Ativa
  * `1` — Inativa, com permissão para visualizar documentos
  * `2` — Inativa, sem permissão para visualizar documentos

* `codfil` — Código da filial
* `nomfil` — Nome da filial
* `docfil` — Documento da filial (CNPJ)

---

### **N100MUN** — Municípios

* `seqmun` — Sequência do município
* `seqest` — Sequência do estado
* `codsia` — Código do município no SIAFI (Tesouro Nacional)
* `nommun` — Nome do município
* `codmds` — ?
* `codfgm` — ?
* `recpdf` — ?

---

## WMS Silt

<a name="procedimentos-wms"></a>
### Procedimentos

#### Desfazer conferência concluída de pedido de venda (packing)
Para desfazer a conferência de um pedido de venda já concluída (packing), cancele todos os volumes vinculados ao pedido na tela **Gerenciador de Volume**.
<br/><br/>

### Integração de Arquivo

A tela Integração de Arquivo pode ser usada para reenviar integrações.

Para localizar integrações de pré-fatura, informe o idNotaFiscal do pedido no campo idOperação.

<img width="1881" height="842" alt="2025-11-06 09_19_47-" src="https://github.com/user-attachments/assets/df3e0bc3-b277-4b34-be5e-d71bca66a539" />

<br/><br/>

### Logs
Logs do WMS Silt usados como referência para buscas na tela Log de Segurança.

#### Criação de usuário

```log
Tela: Usuário - Inseriu a entidade: Usuario - Primary Key: id(171), Campos: ativo(true), codBarra(0000188), codigoRedefinirSenha(), dataProximaAlteracao(Mon Dec 01 08:14:34 BRT 2025), departamento(TESTE), email(), entidade(), enviarSenha(false), nomeUsuario(TESTE), nomeUsuarioCompleto(TESTE), permiteAlterarDtRet(false), senha($argon2i$v=19$m=65536,t=10,p=1$sFUxGAhsyjWU7axgIatJXw$ZX5r5T/3lzuwgdSXMBGBISajk4Knh87ZDf5Ok5VtNJ0), senhaTmp(false), tipoUsuario(Operador), turnoTrabalho(), usuarioAD(), usuarioSeniorX(), utilizaSenhaCaseSensitive(true), utilizarLoginSeniorX(false), utzLoginAD(false)
```

#### Alteração no usuário
```log
Tela: Usuário - Alterou a entidade: Usuario - Primary Key: id(170), Campo(s) alterado(s): (departamento) Antes: TESTE2 Depois: TESTE3, (nomeUsuario) Antes: TESTE Depois: TESTE3, (nomeUsuarioCompleto) Antes: TESTE Depois: TESTE3
```

#### Remoção do pedido na onda

```log
NOTA 255107 RETIRADA DA ONDA CANCELADA: 204891
```
```log
REMOVEU O IDNOTAFISCAL: 255107 DO IDONDA: 204891
```

#### Vincular lote ao pedido na tela "Liberar Nota Fiscal para Expedição"
```log
VINCULOU O LOTE 9694688 NA SEPARAÇÃO ESPECIFICA PARA O IDNOTAFISCAL 258162
```

---

## Integrações

### ERP-WMS
#### Erros

Erro: Erro processando retorno da Ordem de separação Mensagem Original: Erro ao atualizar as tabelas de retorno de ordem de separação. Mensagem Original: Erro ao alterar o item da pré-fatura (empresa 1, filial 1, análise de embarque 48148, pré-fatura 19, item 2): Lote 1619C4 já está sendo utilizado neste item 2 da pré-fatura 19.
<img width="1699" height="445" alt="image" src="https://github.com/user-attachments/assets/068fad78-ab26-4e0e-a66b-e8f635e181b4" />
Motivo: No WMS o produto tem algum lote com lote indústria igual e validade diferente.

Solução: No Gerenciador de Lote, altere a data de validade do lote. O botão “Divergência Lote Indústria” pode ser usado para listar todos os casos em que um mesmo lote indústria aparece com validades diferentes.

***

Erro:	Erro ao processar cancelamento, identificador único: "31aac8d7-8820-47ae-b85f-2ab87542de45". Erro: Não foi possível cancelar a nota fiscal de saída: Não foi possível Cancelar a(s) Nota(s) - Nota 549462 - Erro ao atualizar o pedido 586.234 : Usuário não tem permissão para cancelar o pedido (não pode alterar a situação do pedido) | Falha ao gravar confirmação de cancelamento da separação: Identificador: 31aac8d7-8820-47ae-b85f-2ab87542de45
<img width="1717" height="440" alt="image" src="https://github.com/user-attachments/assets/5430f49c-c7df-494c-a460-f020d21a8c4e" />
Motivo: O usuário de integração não possui as permissões necessárias para fazer o cancelamento no ERP.

Solução: No ERP altere o valor para Sim nas linhas "Alterar Situação Pedido", "Alterar Situação NF Saída", "Cancelar NF-e Exp", "Reabilitar NF Saída Emitida" e "Cancelar NF Saída" para o usuário de integração na tela "Parametros de Usuário para Vendas" (F099UVE - Cadastros -> Usuários -> Parametros por Gestão -> Vendas, Faturamento e Transporte)

No Senior-X você pode ver qual é o usuário que faz a integração em Tecnologia -> Configuração -> Por Tenant -> erp_isl -> int_integrador_bifrost -> Editar -> aba Sistema
<img width="1715" height="666" alt="image" src="https://github.com/user-attachments/assets/da52bdbd-f364-4311-978d-980a4df2a39a" />

<img width="950" height="576" alt="image" src="https://github.com/user-attachments/assets/0e8b6559-8f70-41a4-a1be-e1f8c76d1561" />





