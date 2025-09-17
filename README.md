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
- [WMS Silt](#wms-silt)
  - [Procedimentos](#procedimentos-wms)
  - [Logs](#logs)
- [Integrações](#integrações)
  - [ERP-WMS](#erp-wms)

---

## ERP

### Erros e Situações

---

Situação: Na tela “Transferências de Produtos” (F210TPA — Suprimentos → Gestão de Estoques → Controle de Estoque → Transferência) a seguinte mensagem aparece ao tentar transferir estoque:

"As seguintes transferências não foram realizadas, pois os depósitos nos quais os produtos estão presentes ou que irão ser transferidos integram com o WMS."
<img width="1104" height="187" alt="image" src="https://github.com/user-attachments/assets/f184c53b-1aad-42af-8247-eee2ba8cf47e" />

Motivo: A partir da versão 5.10.4.72 do ERP foi incluído um bloqueio para impedir transferências quando o depósito integra com o WMS. (verificação do campo e205dep.intwms)

<img width="1119" height="233" alt="image" src="https://github.com/user-attachments/assets/a088da06-e9fd-4ada-a994-8abefd82d992" />


Solução: Se você tem certeza do que está fazendo e tem acesso ao banco como DBA, pode seguir o procedimento abaixo para executar a transferência.

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
CREATE OR REPLACE TRIGGER trg_maskerp_current_schema
AFTER LOGON ON SCHEMA
BEGIN
    IF UPPER(SYS_CONTEXT('USERENV', 'HOST')) LIKE '%EDUARDOG' THEN
        EXECUTE IMMEDIATE 'ALTER SESSION SET CURRENT_SCHEMA = maskerp';
    END IF;
END;
```

Agora, abra o ERP e tente a transferência: a mensagem não deve mais aparecer, pois o valor de intwms retornará 'N' pela view criada (e não pela tabela original). Assim, você executa a transferência sem alterar a tabela e sem impactar outros usuários.

Após concluir a transferência, desative a trigger e reabra o sistema para evitar que ações futuras em seu ERP usem o valor mascarado de intwms e impeçam a integração com o WMS por engano.

---

## WMS Silt

<a name="procedimentos-wms"></a>
### Procedimentos

#### Desfazer conferência concluída de pedido de venda (packing)
Para desfazer a conferência de um pedido de venda já concluída (packing), cancele todos os volumes vinculados ao pedido na tela **Gerenciador de Volume**.


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





