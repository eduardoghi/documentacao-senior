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
<img width="1699" height="445" src="https://github.com/user-attachments/assets/068fad78-ab26-4e0e-a66b-e8f635e181b4" />
Motivo: No WMS o produto tem algum lote com lote indústria igual e validade diferente.

Solução: No Gerenciador de Lote, altere a data de validade do lote. O botão “Divergência Lote Indústria” pode ser usado para listar todos os casos em que um mesmo lote indústria aparece com validades diferentes.


