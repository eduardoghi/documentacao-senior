# documentacao-senior
Documentação dos sistemas Senior — funcionalidades dos produtos, procedimentos e resolução de problemas.

---

## Índice
- [WMS Silt](#wms-silt)
  - [Logs](#logs)
- [Integrações](#integrações)
  - [ERP-WMS](#erp-wms)

---

## WMS Silt

### Logs
Logs do WMS Silt usados como referência para buscas na tela Log de Segurança.

#### Criação de usuário

```log
Tela: Usuário - Inseriu a entidade: Usuario - Primary Key: id(171), Campos: ativo(true), codBarra(0000188), codigoRedefinirSenha(), dataProximaAlteracao(Mon Dec 01 08:14:34 BRT 2025), departamento(TESTE), email(), entidade(), enviarSenha(false), nomeUsuario(TESTE), nomeUsuarioCompleto(TESTE), permiteAlterarDtRet(false), senha($argon2i$v=19$m=65536,t=10,p=1$sFUxGAhsyjWU7axgIatJXw$ZX5r5T/3lzuwgdSXMBGBISajk4Knh87ZDf5Ok5VtNJ0), senhaTmp(false), tipoUsuario(Operador), turnoTrabalho(), usuarioAD(), usuarioSeniorX(), utilizaSenhaCaseSensitive(true), utilizarLoginSeniorX(false), utzLoginAD(false)
```

#### Remoção do pedido na onda

```log
NOTA 255107 RETIRADA DA ONDA CANCELADA: 204891
```
```log
REMOVEU O IDNOTAFISCAL: 255107 DO IDONDA: 204891
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

Solução: No ERP altere o valor para Sim nas linhas "Alterar Situação Pedido", "Reabilitar NF Saída Emitida" e "Cancelar NF Saída" para o usuário de integração na tela "Parametros de Usuário para Vendas" (F099UVE - Cadastros -> Usuários -> Parametros por Gestão -> Vendas, Faturamento e Transporte)

No Senior-X você pode ver qual é o usuário que faz a integração em Tecnologia -> Configuração -> Por Tenant -> erp_isl -> int_integrador_bifrost -> Editar -> aba Sistema
<img width="1715" height="666" alt="image" src="https://github.com/user-attachments/assets/da52bdbd-f364-4311-978d-980a4df2a39a" />

<img width="950" height="576" alt="image" src="https://github.com/user-attachments/assets/0e8b6559-8f70-41a4-a1be-e1f8c76d1561" />





