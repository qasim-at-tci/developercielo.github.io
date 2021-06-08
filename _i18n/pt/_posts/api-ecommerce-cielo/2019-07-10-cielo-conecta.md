---
layout: manual
title: Manual de Integração Cielo Conecta
description: API para integração de vendas no físico e OnLine
search: true
translated: true
toc_footers: true
categories: manual
sort_order: 1
tags:
  - API Pagamento
language_tabs:
  json: JSON
  shell: cURL
---

# Visão geral - API Cielo Conecta

# Objetivo

Possibilitar a integração de parceiros de negócio/Subadquirentes com a Cielo para transações com cartões não-presentes (transações digitadas) e cartões presentes nas modalidades Chip e Tarja.

# Glossário

|ID|Descrição|
|BC|Biblioteca Compartilhada para PINPad|
|DUKPT|(Devired Unique Key Per Transaction) Método de criptografia utilizado na Cielo|
|PIN|Senha do cartão|
|BDK|(Base Derived Key) Chave do Sub a ser instalada no HSM|
|HSM|(Hardware Security Module) Servidor para geração, armazenamento, gerenciamento e funcionalidades criptográficas de chaves digitais|
|OAUTH2|Protocolo de autenticação utilizado nas APIs|

# Pré-requisitos

Para a integração é necessário que a solução de captura do parceiro de negócio/Subadquirente possua os seguintes componentes:

* Biblioteca Compartilhada para PINPad ou biblioteca proprietária certificada com as bandeiras.
* Chaves de Criptografia DUKPT implementada para PIN.
* Disponibilizar sua BDK para instalação no HSM Cielo.
* Para soluções de pagamento que utilizam Pinpad externo (conexão Bluetooth ou cabo), é obrigatório o uso de criptografia WKPAN para dados. Clientes desse tipo de solução precisam apresentar certificado PCI DSS e PA DSS.

Formato da Chave exigida pela Cielo:

O HSM Cielo está parametrizado para um KSN da seguinte forma:

* **KSI -** Número de identificação da Chave
* **DID –** Device ID
* **TC –** Transaction Counter

No cadastro da chave somente é inserido o KSI que possui 5 caracteres numéricos e a chave, conforme exemplo abaixo:

**FFFFF**030331234500012

<aside class="warning">Obs.: Os F’s devem ser preenchidos automaticamente pela Solução de Captura.</aside>

# Autenticação

A autenticação é uma operação necessária para obtenção do token que será utilizado nas demais chamadas de APIs.

> **URL**: https://authsandbox.cieloecommerce.cielo.com.br

|Security scheme type:|OAuth2|
|clientCredentials OAuth Flow|**Username:** ClientId<br><br>**Password:** ClientSecret<br>**Scopes:**<br><br>* `PhysicalCieloMaster` - Cadastrar de Lojas e Terminais<br><br>* `PhysicalCieloTransactional` - Transacionar e consultar<br><br>* Se não solicitar um escopo ele é atribuido por padrão|

Para testes em sandbox você pode gerar uma credencial a qualquer momento através do site abaixo.

> **URL**: https://omnichannelcadastrosandbox.cieloecommerce.cielo.com.br/

## Criar Token

> **POST** https://authsandbox.cieloecommerce.cielo.com.br/oauth2/token
>
>| Key | Value |
>|---|---|
>|**AUTHORIZATION** |*Username{Auth_ClientId}*|
>|**AUTHORIZATION** |*Password{Auth_ClientSecret}*|
>|**HEADERS** |*Content-Type text/plain*|
>|**BODY** |*grant_type client_credentials*|
>|**scope** |*{scope}*|

### Request

```shell
curl --location --request POST 'https://authsandbox.cieloecommerce.cielo.com.br/oauth2/token' \
--header 'Content-Type: text/plain' \
--header 'Authorization: Basic {base64(ClientId:ClientSecret)}' \
--data-urlencode 'grant_type=client_credentials'
```

### Response

```json
{
  "access_token": "{Auth_AccessToken}",
  "token_type": "bearer",
  "expires_in": 86399
}
```

# Pagamento

## Autorização

Quando um pagamento é criado (201 - Created), deve-se analisar o Status (Payment.Status) na resposta para certificar-se que o pagamento foi gerado com sucesso ou se houve alguma falha.

| SandBox                                             | Produção                                      |
|:---------------------------------------------------:|:---------------------------------------------:|
| https://apisandbox.cieloecommerce.cielo.com.br      | https://api.cieloecommerce.cielo.com.br/      |

### Crédito Digitado

#### Requisição

<aside class="request"><span class="method post">POST</span> <span class="endpoint">/1/physicalSales/</span></aside>

```json
{
  "MerchantOrderId": "201904150001",
  "Payment": {
    "Type": "PhysicalCreditCard",
    "SoftDescriptor": "Description",
    "PaymentDateTime": "2019-04-15T12:00:00Z",
    "Amount": 15798,
    "Installments": 1,
    "Capture": true,
    "Interest": "ByMerchant",
    "ProductId": 1,
    "CreditCard": {
      "CardNumber": 1234567812345678,
      "ExpirationDate": "12/2020",
      "SecurityCodeStatus": "Collected",
      "SecurityCode": 1230,
      "BrandId": 1,
      "IssuerId": 2,
      "InputMode": "Typed",
      "AuthenticationMethod": "NoPassword",
      "TruncateCardNumberWhenPrinting": true,
      "SaveCard": false,
      "IsFallback": false
    },
    "PinPadInformation": {
      "TerminalId": "10000001",
      "SerialNumber": "ABC123",
      "PhysicalCharacteristics": "PinPadWithChipReaderWithSamModule",
      "ReturnDataInfo": "00"
    }
  }
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`MerchantOrderId`|String|---|---| Número do documento gerado automaticamente pelo terminal e incrementado de 1 a cada transação realizada no terminal. Aceita apenas valores numéricos de 1 a 15 dígitos.|
|`Payment.Type`|String|---|Sim|Value: `PhysicalCreditCard` / Tipo da Transação|
|`Payment.SoftDescriptor`|String|13|---|Identificação do estabelecimento (nome reduzido) a ser impresso e identificado na fatura.|
|`Payment.PaymentDateTime`|String|date-time|Sim|Data e Hora da captura da transação|
|`Payment.Amount`|Integer(int64)|---|Sim|Valor da transação (1079 = R$10,79)|
|`Payment.Capture`|Booleano|---|---|Default: false / Booleano que identifica que a autorização deve ser com captura automática. A autorização sem captura automática é conhecida também como pré-autorização.|
|`Payment.Installments`|Integer|---|---|Default: 1 / Quantidade de Parcelas: Varia de 2 a 99 para transação de financiamento. Deve ser verificado os atributos maxOfPayments1, maxOfPayments2, maxOfPayments3 e minValOfPayments da tabela productTable.|
|`Payment.Interest`|String|---|---|Default: `ByMerchant` <br><br> Enum: `ByMerchant` `ByIssuer` <br><br> Tipo de Parcelamento: <br><br> - Se o bit 6 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 6 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento sem juros pode ser efetuado. <br><br> - Se o bit 7 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 7 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento com juros pode ser efetuado. Sem juros = “ByMerchant”; Com juros = “ByIssuer”.|
|`Payment.ProductId`|Integer|---|Sim|Código do produto identificado através do bin do cartão.|
|`CreditCard.CardNumber`|String|19|---|Número do cartão (PAN)|
|`CreditCard.ExpirationDate`|String|MM/yyyy|Sim|Data de validade do cartão. <br><br>Dado obtido através do comando PP_GetCard na BC no momento da captura da transação.|
|`CreditCard.SecurityCodeStatus`|String|---|---|Enum: `Collected` `Unreadable` `Nonexistent` <br><br> Status da coleta de código de segurança (CVV)|
|`CreditCard.SecurityCode`|String|3 ou 4|---|Código de segurança (CVV)|
|`CreditCard.BrandId`|Integer|---|Sim|Identificação da bandeira obtida através do campo BrandId da PRODUCT TABLE.|
|`CreditCard.IssuerId`|Integer|---|Sim|Código do emissor obtido através do campo IssuerId da BIN TABLE.|
|`CreditCard.InputMode`|String|---|Sim|Enum: `Typed` `MagStripe` `Emv` <br><br>Identificação do modo de captura do cartão na transação. Essa informação deve ser obtida através do retorno da função PP_GetCard da BC. <br><br>"00" – Magnético <br><br>"01" - Moedeiro VISA Cash sobre TIBC v1 <br><br>"02" - Moedeiro VISA Cash sobre TIBC v3 <br><br>"03" – EMV com contato <br><br>"04" - Easy-Entry sobre TIBC v1 <br><br>"05" - Chip sem contato simulando tarja <br><br> "06" - EMV sem contato.|
|`CreditCard.AuthenticationMethod`|String|---|Sim|Enum: `NoPassword` `OnlineAuthentication` `OfflineAuthentication` <br><br>Método de autenticação <br><br>- Se o cartão foi lido a partir da digitação verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, a senha deve ser capturada e o authenticationMethod assume valor 2. Caso contrário, assume valor 1; <br><br>- Se o cartão foi lido a partir da trilha verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, deve ser verificado o bit 2 do mesmo campo. Se este estiver com valor 1 deve ser capturada a senha. Se estiver com valor 0 a captura da senha vai depender do último dígito do service code; <br><br>- Se o cartão foi lido através do chip EMV, o authenticationMethod será preenchido com base no retorno da função PP_GoOnChip(). No resultado PP_GoOnChip(), onde se o campo da posição 003 do retorno da PP_GoOnChip() estiver com valor 1 indica que o pin foi validado off-line, o authenticationMethod será 3. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valor 0, o authenticationMethod será 1. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valores 0 e 1 respectivamente, o authenticationMethod será 2. <br><br>1 - Sem senha = “NoPassword”; <br><br>2 - Senha online = “Online Authentication”; <br><br>3 - Senha off-line = “Offline Authentication”.|
|`CreditCard.TruncateCardNumberWhenPrinting`|Booleano|---|---|Indica se o número do cartão será truncado no momento da impressão do comprovante. A solução de captura deve tomar essa decisão com base no `confParamOp03` presente nas tabelas BIN TABLE, PARAMETER TABLE e ISSUER TABLE.|
|`CreditCard.SaveCard`|---|---|---|
|`CreditCard.IsFallback`|---|---|---|
|`PinPadInformation.TerminalId`|String|---|Sim|Número Lógico definido no Concentrador Cielo.|
|`PinPadInformation.SerialNumber`|String|---|Sim|Número de Série do Equipamento.|
|`PinPadInformation.PhysicalCharacteristics`|String|---|Sim|Enum: `WithoutPinPad` `PinPadWithoutChipReader` `PinPadWithChipReaderWithoutSamModule` `PinPadWithChipReaderWithSamModule` `NotCertifiedPinPad` `PinPadWithChipReaderWithoutSamAndContactless` `PinPadWithChipReaderWithSamModuleAndContactless` <br><br> Sem PIN-pad = `WithoutPinPad`; <br><br> PIN-pad sem leitor de Chip = `PinpadWithoutChipReader`; <br><br>PIN-pad com leitor de Chip sem módulo SAM = `PinPadWithChipReaderWithoutSamModule`; <br><br> PIN-pad com leitor de Chip com módulo SAM = `PinPadWithChipReaderWithSamModule`; <br><br> PIN-pad não homologado = `NotCertifiedPinPad`; <br><br> PIN-pad com leitor de Chip sem SAM e Cartão Sem Contato = `PinpadWithChipReaderWithoutSamAndContactless`; <br><br> PIN-pad com leitor de Chip com SAM e Cartão Sem Contato = `PinpadWithChipReaderWithSamAndContactless`. <br><br><br> Obs. Caso a aplicação não consiga informar os dados acima, deve obter tais informações através do retorno da função PP_GetInfo() da BC.|
|`PinPadInformation.ReturnDataInfo`|String|---|Sim|Retorno da função PP_GetInfo() da biblioteca compartilhada|

#### Resposta

```json
{
  "MerchantOrderId": "20180204",
  "Customer": {
    "Name": "Comprador crédito completo",
    "Identity": "11225468954",
    "IdentityType": "CPF",
    "Email": "compradorteste@teste.com",
    "Birthday": "1991-01-02",
    "Address": {
      "Street": "Rua Teste",
      "Number": "123",
      "Complement": "AP 123",
      "ZipCode": "12345987",
      "City": "São Paulo",
      "State": "SP",
      "Country": "BRA"
    },
    "DeliveryAddress": {
      "Street": "Rua Teste",
      "Number": "123",
      "Complement": "AP 123",
      "ZipCode": "12345987",
      "City": "São Paulo",
      "State": "SP",
      "Country": "BRA"
    }
  },
  "Payment": {
    "Installments": 1,
    "Interest": "ByMerchant",
    "Capture": true,
    "CreditCard": {
      "ExpirationDate": "12/2020",
      "BrandId": 1,
      "IssuerId": 2,
      "TruncateCardNumberWhenPrinting": true,
      "InputMode": "Emv",
      "AuthenticationMethod": "OnlineAuthentication",
      "EmvData": "112233445566778899011AABBC012D3456789E0123FF45678AB901234C5D112233445566778800",
      "PinBlock": {
        "EncryptedPinBlock": "2280F6BDFD0C038D",
        "EncryptionType": "Dukpt3Des",
        "KsnIdentification": "1231vg31fv231313123"
      },
      "PanSequenceNumber": 123,
      "SaveCard": false,
      "IsFallback": false
    },
    "PaymentDateTime": "2019-04-15T12:00:00Z",
    "ServiceTaxAmount": 0,
    "SoftDescriptor": "Description",
    "ProductId": 1,
    "PinPadInformation": {
      "TerminalId": "10000001",
      "SerialNumber": "ABC123",
      "PhysicalCharacteristics": "PinPadWithChipReaderWithSamModule",
      "ReturnDataInfo": "00"
    },
    "Amount": 15798,
    "ReceivedDate": "2019-04-15T12:00:00Z",
    "CapturedAmount": 15798,
    "Provider": "Cielo",
    "ConfirmationStatus": 0,
    "InitializationVersion": 1558708320029,
    "EmvResponseData": "123456789ABCD1345DEA",
    "Status": 2,
    "IsSplitted": false,
    "ReturnCode": 0,
    "ReturnMessage": "Successful",
    "PaymentId": "f15889ea-5719-4e1a-a2da-f4e50d5bd702",
    "Type": "PhysicalDebitCard",
    "Currency": "BRL",
    "Country": "BRA",
    "Links": [
      {
        "Method": "GET",
        "Rel": "self",
        "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702"
      },
      {
        "Method": "DELETE",
        "Rel": "self",
        "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702"
      },
      {
        "Method": "PUT",
        "Rel": "self",
        "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702/confirmation"
      }
    ],
    "PrintMessage": [
      {
        "Position": "Top",
        "Message": "Transação autorizada"
      },
      {
        "Position": "Middle",
        "Message": "Informação adicional"
      },
      {
        "Position": "Bottom",
        "Message": "Obrigado e volte sempre!"
      }
    ],
    "ReceiptInformation": [
      {
        "Field": "MERCHANT_NAME",
        "Label": "NOME DO ESTABELECIMENTO",
        "Content": "Estabelecimento"
      },
      {
        "Field": "MERCHANT_ADDRESS",
        "Label": "ENDEREÇO DO ESTABELECIMENTO",
        "Content": "Rua Sem Saida, 0"
      },
      {
        "Field": "MERCHANT_CITY",
        "Label": "CIDADE DO ESTABELECIMENTO",
        "Content": "Cidade"
      },
      {
        "Field": "MERCHANT_STATE",
        "Label": "ESTADO DO ESTABELECIMENTO",
        "Content": "WA"
      },
      {
        "Field": "MERCHANT_CODE",
        "Label": "COD.ESTAB.",
        "Content": 1234567890123456
      },
      {
        "Field": "TERMINAL",
        "Label": "POS",
        "Content": 12345678
      },
      {
        "Field": "NSU",
        "Label": "DOC",
        "Content": 123456
      },
      {
        "Field": "DATE",
        "Label": "DATA",
        "Content": "01/01/20"
      },
      {
        "Field": "HOUR",
        "Label": "HORA",
        "Content": "01:01"
      },
      {
        "Field": "ISSUER_NAME",
        "Label": "EMISSOR",
        "Content": "NOME DO EMISSOR"
      },
      {
        "Field": "CARD_NUMBER",
        "Label": "CARTÃO",
        "Content": 5432123454321234
      },
      {
        "Field": "TRANSACTION_TYPE",
        "Label": "TIPO DE TRANSAÇÃO",
        "Content": "VENDA A CREDITO"
      },
      {
        "Field": "AUTHORIZATION_CODE",
        "Label": "AUTORIZAÇÃO",
        "Content": 123456
      },
      {
        "Field": "TRANSACTION_MODE",
        "Label": "MODO DA TRANSAÇÃO",
        "Content": "ONL"
      },
      {
        "Field": "INPUT_METHOD",
        "Label": "MODO DE ENTRADA",
        "Content": "X"
      },
      {
        "Field": "VALUE",
        "Label": "VALOR",
        "Content": "1,23"
      },
      {
        "Field": "SOFT_DESCRIPTOR",
        "Label": "SOFT DESCRIPTOR",
        "Content": "Simulado"
      }
    ],
    "Receipt": {
        "MerchantName": "Estabelecimento",
        "MerchantAddress": "Rua Sem Saida, 0",
        "MerchantCity": "Cidade",
        "MerchantState": "WA",
        "MerchantCode": 1234567890123456,
        "Terminal": 12345678,
        "Nsu": 123456,
        "Date": "01/01/20",
        "Hour": "01:01",
        "IssuerName": "NOME DO EMISSOR",
        "CardNumber": 5432123454321234,
        "TransactionType": "VENDA A CREDITO",
        "AuthorizationCode": 123456,
        "TransactionMode": "ONL",
        "InputMethod": "X",
        "Value": "1,23",
        "SoftDescriptor": "Simulado"
      },
        "RecurrentPayment": {
        "RecurrentPaymentId": "a6b719fa-a8df-ab11-4e1a-f4e50d5bd702",
        "ReasonCode": 0,
        "ReasonMessage": "Successful",
        "NextRecurrency": "2019-12-01",
        "EndDate": "2019-12-01",
        "Interval": 6
      },
        "SplitPayments": [
        {
           "SubordinateMerchantId": "491daf20-35f2-4379-874c-e7552ae8dc10",
           "Amount": 100,
           "Fares": {
              "Mdr": 5,
              "Fee": 0
           }
        },
        {
           "SubordinateMerchantId": "7e2846be-4e80-4f86-8ca9-eb35db6aea00",
           "Amount": 80,
           "Fares": {
           "Mdr": 3,
           "Fee": 1
        }
        }
      ],
        "SplitErrors": [
      {
        "Code": 326,
        "Message": "SubordinatePayment amount must be greater than zero"
      }
    ]
  }
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`MerchantOrderId`|String|---|---| Número do documento gerado automaticamente pelo terminal e incrementado de 1 a cada transação realizada no terminal. Aceita apenas valores numéricos de 1 a 15 dígitos.|
|`Customer.Name`|String|---|---|---||---|
|`Customer.Identity`|---|---|---|---|
|`Customer.IdentityType`|---|---|---|---|
|`Customer.Email`|---|---|---||---|
|`Customer.Birthday`|---|---|---|---|
|`Address.Street`|String|---|---|---|
|`Address.Number`|String|---|---|---|
|`Address.Complement`|String|---|---|---|
|`Address.ZipCode`|String|---|---|---|
|`Address.City`|String|---|---|---|
|`Address.State`|String|---|---|---|
|`Address.Country`|String|---|---|---|
|`DeliveryAddress.Street`|String|---|---|---|
|`DeliveryAddress.Number`|String|---|---|---|
|`DeliveryAddress.Complement`|String|---|---|---|
|`DeliveryAddress.ZipCode`|String|---|---|---|
|`DeliveryAddress.City`|String|---|---|---|
|`DeliveryAddress.State`|String|---|---|---|
|`DeliveryAddress.Country`|String|---|---|---|
|`Payment.Installments`|Integer|---|---|Default: 1 / Quantidade de Parcelas: Varia de 2 a 99 para transação de financiamento. Deve ser verificado os atributos maxOfPayments1, maxOfPayments2, maxOfPayments3 e minValOfPayments da tabela productTable.|
|`Payment.Interest`|String|---|---|Default: `ByMerchant` <br><br> Enum: `ByMerchant` `ByIssuer` <br><br> Tipo de Parcelamento: <br><br> - Se o bit 6 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 6 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento sem juros pode ser efetuado. <br><br> - Se o bit 7 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 7 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento com juros pode ser efetuado. Sem juros = “ByMerchant”; Com juros = “ByIssuer”.|
|`Payment.Capture`|Booleano|---|---|Default: false / Booleano que identifica que a autorização deve ser com captura automática. A autorização sem captura automática é conhecida também como pré-autorização.|
|`CreditCard.ExpirationDate`|String|MM/yyyy|Sim|Data de validade do cartão. <br><br>Dado obtido através do comando PP_GetCard na BC no momento da captura da transação.|
|`CreditCard.BrandId`|Integer|---|Sim|Identificação da bandeira obtida através do campo BrandId da PRODUCT TABLE.|
|`CreditCard.IssuerId`|Integer|---|Sim|Código do emissor obtido através do campo IssuerId da BIN TABLE.|
|`CreditCard.TruncateCardNumberWhenPrinting`|Booleano|---|---|Indica se o número do cartão será truncado no momento da impressão do comprovante. A solução de captura deve tomar essa decisão com base no `confParamOp03` presente nas tabelas BIN TABLE, PARAMETER TABLE e ISSUER TABLE.|
|`CreditCard.InputMode`|String|---|Sim|Enum: `Typed` `MagStripe` `Emv` <br><br>Identificação do modo de captura do cartão na transação. Essa informação deve ser obtida através do retorno da função PP_GetCard da BC. <br><br>"00" – Magnético <br><br>"01" - Moedeiro VISA Cash sobre TIBC v1 <br><br>"02" - Moedeiro VISA Cash sobre TIBC v3 <br><br>"03" – EMV com contato <br><br>"04" - Easy-Entry sobre TIBC v1 <br><br>"05" - Chip sem contato simulando tarja <br><br> "06" - EMV sem contato.|
|`CreditCard.AuthenticationMethod`|String|---|Sim|Enum: `NoPassword` `OnlineAuthentication` `OfflineAuthentication` <br><br>Método de autenticação <br><br>- Se o cartão foi lido a partir da digitação verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, a senha deve ser capturada e o authenticationMethod assume valor 2. Caso contrário, assume valor 1; <br><br>- Se o cartão foi lido a partir da trilha verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, deve ser verificado o bit 2 do mesmo campo. Se este estiver com valor 1 deve ser capturada a senha. Se estiver com valor 0 a captura da senha vai depender do último dígito do service code; <br><br>- Se o cartão foi lido através do chip EMV, o authenticationMethod será preenchido com base no retorno da função PP_GoOnChip(). No resultado PP_GoOnChip(), onde se o campo da posição 003 do retorno da PP_GoOnChip() estiver com valor 1 indica que o pin foi validado off-line, o authenticationMethod será 3. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valor 0, o authenticationMethod será 1. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valores 0 e 1 respectivamente, o authenticationMethod será 2. <br><br>1 - Sem senha = “NoPassword”; <br><br>2 - Senha online = “Online Authentication”; <br><br>3 - Senha off-line = “Offline Authentication”.|
|`CreditCard.EmvData`|String|---|---|Dados da transação EMV <br><br>Obtidos através do comando PP_GoOnChip na BC|
|`PinBlock.EncryptedPinBlock`|---|---|---|---|
|`PinBlock.EncryptionType`|String|---|---|---|
|`PinBlock.KsnIdentification`|String|---|---|---|
|`CreditCard.PanSequenceNumber`|Number|---|---|Número sequencial do cartão, utilizado para identificar a conta corrente do cartão adicional. Mandatório para transações com cartões Chip EMV e que possuam PAN Sequence Number (Tag 5F34).|
|`CreditCard.SaveCard`|---|---|---|---|
|`CreditCard.IsFallback`|---|---|---|---|
|`Payment.PaymentDateTime`|String|date-time|Sim|Data e Hora da captura da transação|
|`Payment.ServiceTaxAmount`|---|---|---|---|
|`Payment.SoftDescriptor`|String|13|---|Identificação do estabelecimento (nome reduzido) a ser impresso e identificado na fatura.|
|`Payment.ProductId`|Integer|---|Sim|Código do produto identificado através do bin do cartão.|
|`PinPadInformation.TerminalId`|String|---|Sim|Número Lógico definido no Concentrador Cielo.|
|`PinPadInformation.SerialNumber`|String|---|Sim|Número de Série do Equipamento.|
|`PinPadInformation.PhysicalCharacteristics`|String|---|Sim|Enum: `WithoutPinPad` `PinPadWithoutChipReader` `PinPadWithChipReaderWithoutSamModule` `PinPadWithChipReaderWithSamModule` `NotCertifiedPinPad` `PinPadWithChipReaderWithoutSamAndContactless` `PinPadWithChipReaderWithSamModuleAndContactless` <br><br> Sem PIN-pad = `WithoutPinPad`; <br><br> PIN-pad sem leitor de Chip = `PinpadWithoutChipReader`; <br><br>PIN-pad com leitor de Chip sem módulo SAM = `PinPadWithChipReaderWithoutSamModule`; <br><br> PIN-pad com leitor de Chip com módulo SAM = `PinPadWithChipReaderWithSamModule`; <br><br> PIN-pad não homologado = `NotCertifiedPinPad`; <br><br> PIN-pad com leitor de Chip sem SAM e Cartão Sem Contato = `PinpadWithChipReaderWithoutSamAndContactless`; <br><br> PIN-pad com leitor de Chip com SAM e Cartão Sem Contato = `PinpadWithChipReaderWithSamAndContactless`. <br><br><br> Obs. Caso a aplicação não consiga informar os dados acima, deve obter tais informações através do retorno da função PP_GetInfo() da BC.|
|`PinPadInformation.ReturnDataInfo`|String|---|Sim|Retorno da função PP_GetInfo() da biblioteca compartilhada|
|`Payment.Amount`|Integer(int64)|---|Sim|Valor da transação (1079 = R$10,79)|
|`Payment.ReceivedDate`|---|---|---|---|
|`Payment.CapturedAmount`|---|---|---|---|
|`Payment.Provider`|String|---|---|---|
|`Payment.ConfirmationStatus`|---|---|---|---|
|`Payment.InitializationVersion`|---|---|---|---|
|`Payment.EmvResponseData`|---|---|---|---|
|`Payment.Status`|---|---|---|---|
|`Payment.IsSplitted`|Booleano|---|---|---|
|`Payment.ReturnCode`|---|---|---|---|
|`Payment.ReturnMessage`|String|---|---|---|
|`Payment.PaymentId`|---|---|---|---|
|`Payment.Type`|String|---|Sim|Value: `PhysicalCreditCard` / Tipo da Transação|
|`Payment.Currency`|String|---|---|Default: "BRL" / Value: "BRL" / Moeda (Preencher com “BRL”).|
|`Payment.Country`|String|---|---|Default: "BRA" / Value: "BRA" / País (Preencher com “BRA”).|
|`Receipt.MerchantName`|---|---|---|---|
|`Receipt.MerchantAddress`|---|---|---|---|
|`Receipt.MerchantCity`|---|---|---|---|
|`Receipt.MerchantState`|---|---|---|---|
|`Receipt.MerchantCode`|---|---|---|---|
|`Receipt.Terminal`|---|---|---|---|
|`Receipt.Nsu`|---|---|---|---|
|`Receipt.Date`|---|---|---|---|
|`Receipt.Hour`|---|---|---|---|
|`Receipt.IssuerName`|---|---|---|---|
|`Receipt.CardNumber`|---|---|---|---|
|`Receipt.TransactionType`|---|---|---|---|
|`Receipt.AuthorizationCode`|---|---|---|---|
|`Receipt.TransactionMode`|---|---|---|---|
|`Receipt.InputMethod`|---|---|---|---|
|`Receipt.Value`|---|---|---|---|
|`Receipt.SoftDescriptor`|---|---|---|---|
|`RecurrentPayment.RecurrentPaymentId`|---|---|---|---|
|`RecurrentPayment.ReasonCode`|---|---|---|---|
|`RecurrentPayment.ReasonMessage`|---|---|---|---|
|`RecurrentPayment.NextRecurrency`|---|---|---|---|
|`RecurrentPayment.EndDate`|---|---|---|---|
|`RecurrentPayment.Interval`|---|---|---|---|
|`SplitPayments.SubordinateMerchantId`|---|---|---|---|
|`SplitPayments.Amount`|---|---|---|---|
|`SplitPayments.Fares.Mdr`|---|---|---|---|
|`SplitPayments.Fares.Fee`|---|---|---|---|
|`SplitErrors.Code`|---|---|---|---|
|`SplitErrors.Message`|---|---|---|---|

### Crédito Digitado com Recorrência

#### Requisição

<aside class="request"><span class="method post">POST</span> <span class="endpoint">/1/physicalSales/</span></aside>

```json
{
  "MerchantOrderId": "201904150001",
  "Payment": {
    "Type": "PhysicalCreditCard",
    "SoftDescriptor": "Description",
    "PaymentDateTime": "2019-04-15T12:00:00Z",
    "Amount": 15798,
    "Installments": 1,
    "Capture": true,
    "Interest": "ByMerchant",
    "ProductId": 1,
    "CreditCard": {
      "CardNumber": 1234567812345678,
      "ExpirationDate": "12/2020",
      "SecurityCodeStatus": "Collected",
      "SecurityCode": 1230,
      "BrandId": 1,
      "IssuerId": 2,
      "InputMode": "Typed",
      "AuthenticationMethod": "NoPassword",
      "TruncateCardNumberWhenPrinting": true,
      "SaveCard": false,
      "IsFallback": false
    },
    "PinPadInformation": {
      "TerminalId": "10000001",
      "SerialNumber": "ABC123",
      "PhysicalCharacteristics": "PinPadWithChipReaderWithSamModule",
      "ReturnDataInfo": "00"
    },
    "RecurrentPayment": {
      "EndDate": "2019-12-01",
      "Interval": "SemiAnual"
    }
  }
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`MerchantOrderId`|String|---|---| Número do documento gerado automaticamente pelo terminal e incrementado de 1 a cada transação realizada no terminal. Aceita apenas valores numéricos de 1 a 15 dígitos.|
|`Payment.Type`|String|---|Sim|Value: `PhysicalCreditCard` / Tipo da Transação|
|`Payment.SoftDescriptor`|String|13|---|Identificação do estabelecimento (nome reduzido) a ser impresso e identificado na fatura.|
|`Payment.PaymentDateTime`|String|date-time|Sim|Data e Hora da captura da transação|
|`Payment.Amount`|Integer(int64)|---|Sim|Valor da transação (1079 = R$10,79)|
|`Payment.Installments`|Integer|---|---|Default: 1 / Quantidade de Parcelas: Varia de 2 a 99 para transação de financiamento. Deve ser verificado os atributos maxOfPayments1, maxOfPayments2, maxOfPayments3 e minValOfPayments da tabela productTable.|
|`Payment.Interest`|String|---|---|Default: `ByMerchant` <br><br> Enum: `ByMerchant` `ByIssuer` <br><br> Tipo de Parcelamento: <br><br> - Se o bit 6 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 6 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento sem juros pode ser efetuado. <br><br> - Se o bit 7 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 7 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento com juros pode ser efetuado. Sem juros = “ByMerchant”; Com juros = “ByIssuer”.|
|`Payment.Capture`|Booleano|---|---|Default: false / Booleano que identifica que a autorização deve ser com captura automática. A autorização sem captura automática é conhecida também como pré-autorização.|
|`Payment.ProductId`|Integer|---|Sim|Código do produto identificado através do bin do cartão.|
|`CreditCard.CardNumber`|String|19|---|Número do cartão (PAN)|
|`CreditCard.ExpirationDate`|String|MM/yyyy|Sim|Data de validade do cartão. <br><br>Dado obtido através do comando PP_GetCard na BC no momento da captura da transação.|
|`CreditCard.SecurityCodeStatus`|String|---|---|Enum: `Collected` `Unreadable` `Nonexistent` <br><br> Status da coleta de código de segurança (CVV)|
|`CreditCard.SecurityCode`|String|3 ou 4|---|Código de segurança (CVV)|
|`CreditCard.BrandId`|Integer|---|Sim|Identificação da bandeira obtida através do campo BrandId da PRODUCT TABLE.|
|`CreditCard.IssuerId`|Integer|---|Sim|Código do emissor obtido através do campo IssuerId da BIN TABLE.|
|`CreditCard.InputMode`|String|---|Sim|Enum: `Typed` `MagStripe` `Emv` <br><br>Identificação do modo de captura do cartão na transação. Essa informação deve ser obtida através do retorno da função PP_GetCard da BC. <br><br>"00" – Magnético <br><br>"01" - Moedeiro VISA Cash sobre TIBC v1 <br><br>"02" - Moedeiro VISA Cash sobre TIBC v3 <br><br>"03" – EMV com contato <br><br>"04" - Easy-Entry sobre TIBC v1 <br><br>"05" - Chip sem contato simulando tarja <br><br> "06" - EMV sem contato.|
|`CreditCard.AuthenticationMethod`|String|---|Sim|Enum: `NoPassword` `OnlineAuthentication` `OfflineAuthentication` <br><br>Método de autenticação <br><br>- Se o cartão foi lido a partir da digitação verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, a senha deve ser capturada e o authenticationMethod assume valor 2. Caso contrário, assume valor 1; <br><br>- Se o cartão foi lido a partir da trilha verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, deve ser verificado o bit 2 do mesmo campo. Se este estiver com valor 1 deve ser capturada a senha. Se estiver com valor 0 a captura da senha vai depender do último dígito do service code; <br><br>- Se o cartão foi lido através do chip EMV, o authenticationMethod será preenchido com base no retorno da função PP_GoOnChip(). No resultado PP_GoOnChip(), onde se o campo da posição 003 do retorno da PP_GoOnChip() estiver com valor 1 indica que o pin foi validado off-line, o authenticationMethod será 3. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valor 0, o authenticationMethod será 1. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valores 0 e 1 respectivamente, o authenticationMethod será 2. <br><br>1 - Sem senha = “NoPassword”; <br><br>2 - Senha online = “Online Authentication”; <br><br>3 - Senha off-line = “Offline Authentication”.|
|`CreditCard.TruncateCardNumberWhenPrinting`|Booleano|---|---|Indica se o número do cartão será truncado no momento da impressão do comprovante. A solução de captura deve tomar essa decisão com base no `confParamOp03` presente nas tabelas BIN TABLE, PARAMETER TABLE e ISSUER TABLE.|
|`CreditCard.SaveCard`|---|---|---|---|
|`CreditCard.IsFallback`|---|---|---|---|
|`PinPadInformation.TerminalId`|String|---|Sim|Número Lógico definido no Concentrador Cielo.|
|`PinPadInformation.SerialNumber`|String|---|Sim|Número de Série do Equipamento.|
|`PinPadInformation.PhysicalCharacteristics`|String|---|Sim|Enum: `WithoutPinPad` `PinPadWithoutChipReader` `PinPadWithChipReaderWithoutSamModule` `PinPadWithChipReaderWithSamModule` `NotCertifiedPinPad` `PinPadWithChipReaderWithoutSamAndContactless` `PinPadWithChipReaderWithSamModuleAndContactless` <br><br> Sem PIN-pad = `WithoutPinPad`; <br><br> PIN-pad sem leitor de Chip = `PinpadWithoutChipReader`; <br><br>PIN-pad com leitor de Chip sem módulo SAM = `PinPadWithChipReaderWithoutSamModule`; <br><br> PIN-pad com leitor de Chip com módulo SAM = `PinPadWithChipReaderWithSamModule`; <br><br> PIN-pad não homologado = `NotCertifiedPinPad`; <br><br> PIN-pad com leitor de Chip sem SAM e Cartão Sem Contato = `PinpadWithChipReaderWithoutSamAndContactless`; <br><br> PIN-pad com leitor de Chip com SAM e Cartão Sem Contato = `PinpadWithChipReaderWithSamAndContactless`. <br><br><br> Obs. Caso a aplicação não consiga informar os dados acima, deve obter tais informações através do retorno da função PP_GetInfo() da BC.|
|`PinPadInformation.ReturnDataInfo`|String|---|Sim|Retorno da função PP_GetInfo() da biblioteca compartilhada|
|`RecurrentPayment.EndDate`|---|---|---|---|
|`RecurrentPayment.Interval`|---|---|---|---|

#### Resposta

```json
{
  "MerchantOrderId": "20180204",
  "Customer": {
    "Name": "Comprador crédito completo",
    "Identity": "11225468954",
    "IdentityType": "CPF",
    "Email": "compradorteste@teste.com",
    "Birthday": "1991-01-02",
    "Address": {
      "Street": "Rua Teste",
      "Number": "123",
      "Complement": "AP 123",
      "ZipCode": "12345987",
      "City": "São Paulo",
      "State": "SP",
      "Country": "BRA"
    },
    "DeliveryAddress": {
      "Street": "Rua Teste",
      "Number": "123",
      "Complement": "AP 123",
      "ZipCode": "12345987",
      "City": "São Paulo",
      "State": "SP",
      "Country": "BRA"
    }
  },
  "Payment": {
    "Installments": 1,
    "Interest": "ByMerchant",
    "Capture": true,
    "CreditCard": {
      "ExpirationDate": "12/2020",
      "BrandId": 1,
      "IssuerId": 2,
      "TruncateCardNumberWhenPrinting": true,
      "InputMode": "Emv",
      "AuthenticationMethod": "OnlineAuthentication",
      "EmvData": "112233445566778899011AABBC012D3456789E0123FF45678AB901234C5D112233445566778800",
      "PinBlock": {
        "EncryptedPinBlock": "2280F6BDFD0C038D",
        "EncryptionType": "Dukpt3Des",
        "KsnIdentification": "1231vg31fv231313123"
      },
      "PanSequenceNumber": 123,
      "SaveCard": false,
      "IsFallback": false
    },
    "PaymentDateTime": "2019-04-15T12:00:00Z",
    "ServiceTaxAmount": 0,
    "SoftDescriptor": "Description",
    "ProductId": 1,
    "PinPadInformation": {
      "TerminalId": "10000001",
      "SerialNumber": "ABC123",
      "PhysicalCharacteristics": "PinPadWithChipReaderWithSamModule",
      "ReturnDataInfo": "00"
    },
    "Amount": 15798,
    "ReceivedDate": "2019-04-15T12:00:00Z",
    "CapturedAmount": 15798,
    "Provider": "Cielo",
    "ConfirmationStatus": 0,
    "InitializationVersion": 1558708320029,
    "EmvResponseData": "123456789ABCD1345DEA",
    "Status": 2,
    "IsSplitted": false,
    "ReturnCode": 0,
    "ReturnMessage": "Successful",
    "PaymentId": "f15889ea-5719-4e1a-a2da-f4e50d5bd702",
    "Type": "PhysicalDebitCard",
    "Currency": "BRL",
    "Country": "BRA",
    "Links": [
      {
        "Method": "GET",
        "Rel": "self",
        "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702"
      },
      {
        "Method": "DELETE",
        "Rel": "self",
        "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702"
      },
      {
        "Method": "PUT",
        "Rel": "self",
        "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702/confirmation"
      }
    ],
    "PrintMessage": [
      {
        "Position": "Top",
        "Message": "Transação autorizada"
      },
      {
        "Position": "Middle",
        "Message": "Informação adicional"
      },
      {
        "Position": "Bottom",
        "Message": "Obrigado e volte sempre!"
      }
    ],
    "ReceiptInformation": [
      {
        "Field": "MERCHANT_NAME",
        "Label": "NOME DO ESTABELECIMENTO",
        "Content": "Estabelecimento"
      },
      {
        "Field": "MERCHANT_ADDRESS",
        "Label": "ENDEREÇO DO ESTABELECIMENTO",
        "Content": "Rua Sem Saida, 0"
      },
      {
        "Field": "MERCHANT_CITY",
        "Label": "CIDADE DO ESTABELECIMENTO",
        "Content": "Cidade"
      },
      {
        "Field": "MERCHANT_STATE",
        "Label": "ESTADO DO ESTABELECIMENTO",
        "Content": "WA"
      },
      {
        "Field": "MERCHANT_CODE",
        "Label": "COD.ESTAB.",
        "Content": 1234567890123456
      },
      {
        "Field": "TERMINAL",
        "Label": "POS",
        "Content": 12345678
      },
      {
        "Field": "NSU",
        "Label": "DOC",
        "Content": 123456
      },
      {
        "Field": "DATE",
        "Label": "DATA",
        "Content": "01/01/20"
      },
      {
        "Field": "HOUR",
        "Label": "HORA",
        "Content": "01:01"
      },
      {
        "Field": "ISSUER_NAME",
        "Label": "EMISSOR",
        "Content": "NOME DO EMISSOR"
      },
      {
        "Field": "CARD_NUMBER",
        "Label": "CARTÃO",
        "Content": 5432123454321234
      },
      {
        "Field": "TRANSACTION_TYPE",
        "Label": "TIPO DE TRANSAÇÃO",
        "Content": "VENDA A CREDITO"
      },
      {
        "Field": "AUTHORIZATION_CODE",
        "Label": "AUTORIZAÇÃO",
        "Content": 123456
      },
      {
        "Field": "TRANSACTION_MODE",
        "Label": "MODO DA TRANSAÇÃO",
        "Content": "ONL"
      },
      {
        "Field": "INPUT_METHOD",
        "Label": "MODO DE ENTRADA",
        "Content": "X"
      },
      {
        "Field": "VALUE",
        "Label": "VALOR",
        "Content": "1,23"
      },
      {
        "Field": "SOFT_DESCRIPTOR",
        "Label": "SOFT DESCRIPTOR",
        "Content": "Simulado"
      }
    ],
    "Receipt": {
      "MerchantName": "Estabelecimento",
      "MerchantAddress": "Rua Sem Saida, 0",
      "MerchantCity": "Cidade",
      "MerchantState": "WA",
      "MerchantCode": 1234567890123456,
      "Terminal": 12345678,
      "Nsu": 123456,
      "Date": "01/01/20",
      "Hour": "01:01",
      "IssuerName": "NOME DO EMISSOR",
      "CardNumber": 5432123454321234,
      "TransactionType": "VENDA A CREDITO",
      "AuthorizationCode": 123456,
      "TransactionMode": "ONL",
      "InputMethod": "X",
      "Value": "1,23",
      "SoftDescriptor": "Simulado"
    },
    "RecurrentPayment": {
      "RecurrentPaymentId": "a6b719fa-a8df-ab11-4e1a-f4e50d5bd702",
      "ReasonCode": 0,
      "ReasonMessage": "Successful",
      "NextRecurrency": "2019-12-01",
      "EndDate": "2019-12-01",
      "Interval": 6
    },
    "SplitPayments": [
      {
        "SubordinateMerchantId": "491daf20-35f2-4379-874c-e7552ae8dc10",
        "Amount": 100,
        "Fares": {
          "Mdr": 5,
          "Fee": 0
        }
      },
      {
        "SubordinateMerchantId": "7e2846be-4e80-4f86-8ca9-eb35db6aea00",
        "Amount": 80,
        "Fares": {
          "Mdr": 3,
          "Fee": 1
        }
      }
    ],
    "SplitErrors": [
      {
        "Code": 326,
        "Message": "SubordinatePayment amount must be greater than zero"
      }
    ]
  }
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`MerchantOrderId`|String|---|---| Número do documento gerado automaticamente pelo terminal e incrementado de 1 a cada transação realizada no terminal. Aceita apenas valores numéricos de 1 a 15 dígitos.|
|`Customer.Name`|---|---|---|---|
|`Customer.Identity`|---|---|---|---|
|`Customer.IdentityType`|---|---|---|---|
|`Customer.Email`|---|---|---|---|
|`Customer.Birthday`|---|---|---|---|
|`Address.Street`|String|---|---||---|
|`Address.Number`|String|---|---||---|
|`Address.Complement`|String|---|---||---|
|`Address.ZipCode`|String|---|---||---|
|`Address.City`|String|---|---||---|
|`Address.State`|String|---|---||---|
|`Address.Country`|String|---|---||---|
|`DeliveryAddress.Street`|String|---|---||---|
|`DeliveryAddress.Number`|String|---|---||---|
|`DeliveryAddress.Complement`|String|---|---||---|
|`DeliveryAddress.ZipCode`|String|---|---||---|
|`DeliveryAddress.City`|String|---|---||---|
|`DeliveryAddress.State`|String|---|---||---|
|`DeliveryAddress.Country`|String|---|---||---|
|`Payment.Installments`|Integer|---|---|Default: 1 / Quantidade de Parcelas: Varia de 2 a 99 para transação de financiamento. Deve ser verificado os atributos maxOfPayments1, maxOfPayments2, maxOfPayments3 e minValOfPayments da tabela productTable.|
|`Payment.Interest`|String|---|---|Default: `ByMerchant` <br><br> Enum: `ByMerchant` `ByIssuer` <br><br> Tipo de Parcelamento: <br><br> - Se o bit 6 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 6 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento sem juros pode ser efetuado. <br><br> - Se o bit 7 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 7 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento com juros pode ser efetuado. Sem juros = “ByMerchant”; Com juros = “ByIssuer”.|
|`Payment.Capture`|Booleano|---|---|Default: false / Booleano que identifica que a autorização deve ser com captura automática. A autorização sem captura automática é conhecida também como pré-autorização.|
|`CreditCard.ExpirationDate`|String|MM/yyyy|Sim|Data de validade do cartão. <br><br>Dado obtido através do comando PP_GetCard na BC no momento da captura da transação.|
|`CreditCard.BrandId`|Integer|---|Sim|Identificação da bandeira obtida através do campo BrandId da PRODUCT TABLE.|
|`CreditCard.IssuerId`|Integer|---|Sim|Código do emissor obtido através do campo IssuerId da BIN TABLE.|
|`CreditCard.TruncateCardNumberWhenPrinting`|Booleano|---|---|Indica se o número do cartão será truncado no momento da impressão do comprovante. A solução de captura deve tomar essa decisão com base no `confParamOp03` presente nas tabelas BIN TABLE, PARAMETER TABLE e ISSUER TABLE.|
|`CreditCard.InputMode`|String|---|Sim|Enum: `Typed` `MagStripe` `Emv` <br><br>Identificação do modo de captura do cartão na transação. Essa informação deve ser obtida através do retorno da função PP_GetCard da BC. <br><br>"00" – Magnético <br><br>"01" - Moedeiro VISA Cash sobre TIBC v1 <br><br>"02" - Moedeiro VISA Cash sobre TIBC v3 <br><br>"03" – EMV com contato <br><br>"04" - Easy-Entry sobre TIBC v1 <br><br>"05" - Chip sem contato simulando tarja <br><br> "06" - EMV sem contato.|
|`CreditCard.AuthenticationMethod`|String|---|Sim|Enum: `NoPassword` `OnlineAuthentication` `OfflineAuthentication` <br><br>Método de autenticação <br><br>- Se o cartão foi lido a partir da digitação verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, a senha deve ser capturada e o authenticationMethod assume valor 2. Caso contrário, assume valor 1; <br><br>- Se o cartão foi lido a partir da trilha verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, deve ser verificado o bit 2 do mesmo campo. Se este estiver com valor 1 deve ser capturada a senha. Se estiver com valor 0 a captura da senha vai depender do último dígito do service code; <br><br>- Se o cartão foi lido através do chip EMV, o authenticationMethod será preenchido com base no retorno da função PP_GoOnChip(). No resultado PP_GoOnChip(), onde se o campo da posição 003 do retorno da PP_GoOnChip() estiver com valor 1 indica que o pin foi validado off-line, o authenticationMethod será 3. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valor 0, o authenticationMethod será 1. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valores 0 e 1 respectivamente, o authenticationMethod será 2. <br><br>1 - Sem senha = “NoPassword”; <br><br>2 - Senha online = “Online Authentication”; <br><br>3 - Senha off-line = “Offline Authentication”.|
|`CreditCard.EmvData`|String|---|---|Dados da transação EMV <br><br>Obtidos através do comando PP_GoOnChip na BC|
|`PinBlock.EncryptedPinBlock`|---|---|---|---|
|`PinBlock.EncryptionType`|---|---|---|---|
|`PinBlock.KsnIdentification`|---|---|---|---|
|`CreditCard.PanSequenceNumber`|Number|---|---|Número sequencial do cartão, utilizado para identificar a conta corrente do cartão adicional. Mandatório para transações com cartões Chip EMV e que possuam PAN Sequence Number (Tag 5F34).|
|`CreditCard.SaveCard`|---|---|---|---|
|`CreditCard.IsFallback`|---|---|---|---|
|`Payment.PaymentDateTime`|String|date-time|Sim|Data e Hora da captura da transação|
|`Payment.ServiceTaxAmount`|---|---|---|---|
|`Payment.SoftDescriptor`|String|13|---|Identificação do estabelecimento (nome reduzido) a ser impresso e identificado na fatura.|
|`Payment.ProductId`|Integer|---|Sim|Código do produto identificado através do bin do cartão.|
|`PinPadInformation.TerminalId`|String|---|Sim|Número Lógico definido no Concentrador Cielo.|
|`PinPadInformation.SerialNumber`|String|---|Sim|Número de Série do Equipamento.|
|`PinPadInformation.PhysicalCharacteristics`|String|---|Sim|Enum: `WithoutPinPad` `PinPadWithoutChipReader` `PinPadWithChipReaderWithoutSamModule` `PinPadWithChipReaderWithSamModule` `NotCertifiedPinPad` `PinPadWithChipReaderWithoutSamAndContactless` `PinPadWithChipReaderWithSamModuleAndContactless` <br><br> Sem PIN-pad = `WithoutPinPad`; <br><br> PIN-pad sem leitor de Chip = `PinpadWithoutChipReader`; <br><br>PIN-pad com leitor de Chip sem módulo SAM = `PinPadWithChipReaderWithoutSamModule`; <br><br> PIN-pad com leitor de Chip com módulo SAM = `PinPadWithChipReaderWithSamModule`; <br><br> PIN-pad não homologado = `NotCertifiedPinPad`; <br><br> PIN-pad com leitor de Chip sem SAM e Cartão Sem Contato = `PinpadWithChipReaderWithoutSamAndContactless`; <br><br> PIN-pad com leitor de Chip com SAM e Cartão Sem Contato = `PinpadWithChipReaderWithSamAndContactless`. <br><br><br> Obs. Caso a aplicação não consiga informar os dados acima, deve obter tais informações através do retorno da função PP_GetInfo() da BC.|
|`PinPadInformation.ReturnDataInfo`|String|---|Sim|Retorno da função PP_GetInfo() da biblioteca compartilhada|
|`Payment.Amount`|Integer(int64)|---|Sim|Valor da transação (1079 = R$10,79)|
|`Payment.ReceivedDate`|---|---|---|---|
|`Payment.CapturedAmount`|---|---|---|---|
|`Payment.Provider`|String|---|---|---|
|`Payment.ConfirmationStatus`|---|---|---|---|
|`Payment.InitializationVersion`|---|---|---|---|
|`Payment.EmvResponseData`|---|---|---|---|
|`Payment.Status`|---|---|---|---|
|`Payment.IsSplitted`|Booleano|---|---|---|
|`Payment.ReturnCode`|---|---|---|---|
|`Payment.ReturnMessage`|String|---|---|---|
|`Payment.PaymentId`|---|---|---|---|
|`Payment.Type`|String|---|Sim|Value: `PhysicalCreditCard` / Tipo da Transação|
|`Payment.Currency`|String|---|---|Default: "BRL" / Value: "BRL" / Moeda (Preencher com “BRL”)|
|`Payment.Country`|String|---|---|Default: "BRA" / Value: "BRA" / País (Preencher com “BRA”)|
|`Receipt.MerchantName`|---|---|---|---|
|`Receipt.MerchantAddress`|---|---|---|---|
|`Receipt.MerchantCity`|---|---|---|---|
|`Receipt.MerchantState`|---|---|---|---|
|`Receipt.MerchantCode`|---|---|---|---|
|`Receipt.Terminal`|---|---|---|---|
|`Receipt.Nsu`|---|---|---|---|
|`Receipt.Date`|---|---|---|---|
|`Receipt.Hour`|---|---|---|---|
|`Receipt.IssuerName`|---|---|---|---|
|`Receipt.CardNumber`|---|---|---|---|
|`Receipt.TransactionType`|---|---|---|---|
|`Receipt.AuthorizationCode`|---|---|---|---|
|`Receipt.TransactionMode`|---|---|---|---|
|`Receipt.InputMethod`|---|---|---|---|
|`Receipt.Value`|---|---|---|---|
|`Receipt.SoftDescriptor`|---|---|---|---|
|`RecurrentPayment.RecurrentPaymentId`|---|---|---|---|
|`RecurrentPayment.ReasonCode`|---|---|---|---|
|`RecurrentPayment.ReasonMessage`|---|---|---|---|
|`RecurrentPayment.NextRecurrency`|---|---|---|---|
|`RecurrentPayment.EndDate`|---|---|---|---|
|`RecurrentPayment.Interval`|---|---|---|---|
|`SplitPayments.SubordinateMerchantId`|---|---|---|---|
|`SplitPayments.Amount`|---|---|---|---|
|`SplitPayments.Fares.Mdr`|---|---|---|---|
|`SplitPayments.Fares.Fee`|---|---|---|---|
|`SplitErrors.Code`|---|---|---|---|
|`SplitErrors.Message`|---|---|---|---|

### Crédito Digitado com dados de Comprador

#### Requisição

<aside class="request"><span class="method post">POST</span> <span class="endpoint">/1/physicalSales/</span></aside>

```json
{
  "MerchantOrderId": "201904150001",
  "Payment": {
    "Type": "PhysicalCreditCard",
    "SoftDescriptor": "Description",
    "PaymentDateTime": "2019-04-15T12:00:00Z",
    "Amount": 15798,
    "Installments": 1,
    "Capture": true,
    "Interest": "ByMerchant",
    "ProductId": 1,
    "CreditCard": {
      "CardNumber": 1234567812345678,
      "ExpirationDate": "12/2020",
      "SecurityCodeStatus": "Collected",
      "SecurityCode": 1230,
      "BrandId": 1,
      "IssuerId": 2,
      "InputMode": "Typed",
      "AuthenticationMethod": "NoPassword",
      "TruncateCardNumberWhenPrinting": true,
      "SaveCard": false,
      "IsFallback": false
    },
    "Customer": {
      "Name": "Comprador crédito completo",
      "Identity": "11225468954",
      "IdentityType": "CPF",
      "Email": "compradorteste@teste.com",
      "Birthday": "1991-01-02",
      "Address": {
        "Street": "Rua Teste",
        "Number": "123",
        "Complement": "AP 123",
        "ZipCode": "12345987",
        "City": "São Paulo",
        "State": "SP",
        "Country": "BRA"
      },
      "DeliveryAddress": {
        "Street": "Rua Teste",
        "Number": "123",
        "Complement": "AP 123",
        "ZipCode": "12345987",
        "City": "São Paulo",
        "State": "SP",
        "Country": "BRA"
      }
    },
    "PinPadInformation": {
      "TerminalId": "10000001",
      "SerialNumber": "ABC123",
      "PhysicalCharacteristics": "PinPadWithChipReaderWithSamModule",
      "ReturnDataInfo": "00"
    }
  }
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`MerchantOrderId`|String|---|---| Número do documento gerado automaticamente pelo terminal e incrementado de 1 a cada transação realizada no terminal. Aceita apenas valores numéricos de 1 a 15 dígitos.|
|`Payment.Type`|String|---|Sim|Value: `PhysicalCreditCard` / Tipo da Transação|
|`Payment.SoftDescriptor`|String|13|---|Identificação do estabelecimento (nome reduzido) a ser impresso e identificado na fatura.|
|`Payment.PaymentDateTime`|String|date-time|Sim|Data e Hora da captura da transação|
|`Payment.Amount`|Integer(int64)|---|Sim|Valor da transação (1079 = R$10,79)|
|`Payment.Capture`|Booleano|---|---|Default: false / Booleano que identifica que a autorização deve ser com captura automática. A autorização sem captura automática é conhecida também como pré-autorização.|
|`Payment.Installments`|Integer|---|---|Default: 1 / Quantidade de Parcelas: Varia de 2 a 99 para transação de financiamento. Deve ser verificado os atributos maxOfPayments1, maxOfPayments2, maxOfPayments3 e minValOfPayments da tabela productTable.|
|`Payment.Interest`|String|---|---|Default: `ByMerchant` <br><br> Enum: `ByMerchant` `ByIssuer` <br><br> Tipo de Parcelamento: <br><br> - Se o bit 6 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 6 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento sem juros pode ser efetuado. <br><br> - Se o bit 7 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 7 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento com juros pode ser efetuado. Sem juros = “ByMerchant”; Com juros = “ByIssuer”.|
|`Payment.ProductId`|Integer|---|Sim|Código do produto identificado através do bin do cartão.|
|`CreditCard.CardNumber`|String|19|---|Número do cartão (PAN)|
|`CreditCard.ExpirationDate`|String|MM/yyyy|Sim|Data de validade do cartão. <br><br>Dado obtido através do comando PP_GetCard na BC no momento da captura da transação.|
|`CreditCard.SecurityCodeStatus`|String|---|---|Enum: `Collected` `Unreadable` `Nonexistent` <br><br> Status da coleta de código de segurança (CVV)|
|`CreditCard.SecurityCode`|String|3 ou 4|---|Código de segurança (CVV)|
|`CreditCard.BrandId`|Integer|---|Sim|Identificação da bandeira obtida através do campo BrandId da PRODUCT TABLE.|
|`CreditCard.IssuerId`|Integer|---|Sim|Código do emissor obtido através do campo IssuerId da BIN TABLE.|
|`CreditCard.InputMode`|String|---|Sim|Enum: `Typed` `MagStripe` `Emv` <br><br>Identificação do modo de captura do cartão na transação. Essa informação deve ser obtida através do retorno da função PP_GetCard da BC. <br><br>"00" – Magnético <br><br>"01" - Moedeiro VISA Cash sobre TIBC v1 <br><br>"02" - Moedeiro VISA Cash sobre TIBC v3 <br><br>"03" – EMV com contato <br><br>"04" - Easy-Entry sobre TIBC v1 <br><br>"05" - Chip sem contato simulando tarja <br><br> "06" - EMV sem contato.|
|`CreditCard.AuthenticationMethod`|String|---|Sim|Enum: `NoPassword` `OnlineAuthentication` `OfflineAuthentication` <br><br>Método de autenticação <br><br>- Se o cartão foi lido a partir da digitação verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, a senha deve ser capturada e o authenticationMethod assume valor 2. Caso contrário, assume valor 1; <br><br>- Se o cartão foi lido a partir da trilha verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, deve ser verificado o bit 2 do mesmo campo. Se este estiver com valor 1 deve ser capturada a senha. Se estiver com valor 0 a captura da senha vai depender do último dígito do service code; <br><br>- Se o cartão foi lido através do chip EMV, o authenticationMethod será preenchido com base no retorno da função PP_GoOnChip(). No resultado PP_GoOnChip(), onde se o campo da posição 003 do retorno da PP_GoOnChip() estiver com valor 1 indica que o pin foi validado off-line, o authenticationMethod será 3. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valor 0, o authenticationMethod será 1. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valores 0 e 1 respectivamente, o authenticationMethod será 2. <br><br>1 - Sem senha = “NoPassword”; <br><br>2 - Senha online = “Online Authentication”; <br><br>3 - Senha off-line = “Offline Authentication”.|
|`CreditCard.TruncateCardNumberWhenPrinting`|Booleano|---|---|Indica se o número do cartão será truncado no momento da impressão do comprovante. A solução de captura deve tomar essa decisão com base no `confParamOp03` presente nas tabelas BIN TABLE, PARAMETER TABLE e ISSUER TABLE.|
|`CreditCard.SaveCard`|---|---|---|---|
|`CreditCard.IsFallback`|---|---|---|---|
|`Customer.Name`|---|---|---|---|
|`Customer.Identity`|---|---|---|---|
|`Customer.IdentityType`|---|---|---|---|
|`Customer.Email`|---|---|---|---|
|`Customer.Birthday`|---|---|---|---|
|`Address.Street`|String|---|---|---|
|`Address.Number`|String|---|---|---|
|`Address.Complement`|String|---|---|---|
|`Address.ZipCode`|String|---|---|---|
|`Address.City`|String|---|---|---|
|`Address.State`|String|---|---|---|
|`Address.Country`|String|---|---|---|
|`DeliveryAddress.Street`|String|---|---|---|
|`DeliveryAddress.Number`|String|---|---|---|
|`DeliveryAddress.Complement`|String|---|---|---|
|`DeliveryAddress.ZipCode`|String|---|---|---|
|`DeliveryAddress.City`|String|---|---|---|
|`DeliveryAddress.State`|String|---|---|---|
|`DeliveryAddress.Country`|---|---|---|---|
|`PinPadInformation.TerminalId`|String|---|Sim|Número Lógico definido no Concentrador Cielo.|
|`PinPadInformation.SerialNumber`|String|---|Sim|Número de Série do Equipamento.|
|`PinPadInformation.PhysicalCharacteristics`|String|---|Sim|Enum: `WithoutPinPad` `PinPadWithoutChipReader` `PinPadWithChipReaderWithoutSamModule` `PinPadWithChipReaderWithSamModule` `NotCertifiedPinPad` `PinPadWithChipReaderWithoutSamAndContactless` `PinPadWithChipReaderWithSamModuleAndContactless` <br><br> Sem PIN-pad = `WithoutPinPad`; <br><br> PIN-pad sem leitor de Chip = `PinpadWithoutChipReader`; <br><br>PIN-pad com leitor de Chip sem módulo SAM = `PinPadWithChipReaderWithoutSamModule`; <br><br> PIN-pad com leitor de Chip com módulo SAM = `PinPadWithChipReaderWithSamModule`; <br><br> PIN-pad não homologado = `NotCertifiedPinPad`; <br><br> PIN-pad com leitor de Chip sem SAM e Cartão Sem Contato = `PinpadWithChipReaderWithoutSamAndContactless`; <br><br> PIN-pad com leitor de Chip com SAM e Cartão Sem Contato = `PinpadWithChipReaderWithSamAndContactless`. <br><br><br> Obs. Caso a aplicação não consiga informar os dados acima, deve obter tais informações através do retorno da função PP_GetInfo() da BC.|
|`PinPadInformation.ReturnDataInfo`|String|---|Sim|Retorno da função PP_GetInfo() da biblioteca compartilhada|

#### Resposta

```json
{
  "MerchantOrderId": "20180204",
  "Customer": {
    "Name": "Comprador crédito completo",
    "Identity": "11225468954",
    "IdentityType": "CPF",
    "Email": "compradorteste@teste.com",
    "Birthday": "1991-01-02",
    "Address": {
      "Street": "Rua Teste",
      "Number": "123",
      "Complement": "AP 123",
      "ZipCode": "12345987",
      "City": "São Paulo",
      "State": "SP",
      "Country": "BRA"
    },
    "DeliveryAddress": {
      "Street": "Rua Teste",
      "Number": "123",
      "Complement": "AP 123",
      "ZipCode": "12345987",
      "City": "São Paulo",
      "State": "SP",
      "Country": "BRA"
    }
  },
  "Payment": {
    "Installments": 1,
    "Interest": "ByMerchant",
    "Capture": true,
    "CreditCard": {
      "ExpirationDate": "12/2020",
      "BrandId": 1,
      "IssuerId": 2,
      "TruncateCardNumberWhenPrinting": true,
      "InputMode": "Emv",
      "AuthenticationMethod": "OnlineAuthentication",
      "EmvData": "112233445566778899011AABBC012D3456789E0123FF45678AB901234C5D112233445566778800",
      "PinBlock": {
        "EncryptedPinBlock": "2280F6BDFD0C038D",
        "EncryptionType": "Dukpt3Des",
        "KsnIdentification": "1231vg31fv231313123"
      },
      "PanSequenceNumber": 123,
      "SaveCard": false,
      "IsFallback": false
    },
    "PaymentDateTime": "2019-04-15T12:00:00Z",
    "ServiceTaxAmount": 0,
    "SoftDescriptor": "Description",
    "ProductId": 1,
    "PinPadInformation": {
      "TerminalId": "10000001",
      "SerialNumber": "ABC123",
      "PhysicalCharacteristics": "PinPadWithChipReaderWithSamModule",
      "ReturnDataInfo": "00"
    },
    "Amount": 15798,
    "ReceivedDate": "2019-04-15T12:00:00Z",
    "CapturedAmount": 15798,
    "Provider": "Cielo",
    "ConfirmationStatus": 0,
    "InitializationVersion": 1558708320029,
    "EmvResponseData": "123456789ABCD1345DEA",
    "Status": 2,
    "IsSplitted": false,
    "ReturnCode": 0,
    "ReturnMessage": "Successful",
    "PaymentId": "f15889ea-5719-4e1a-a2da-f4e50d5bd702",
    "Type": "PhysicalDebitCard",
    "Currency": "BRL",
    "Country": "BRA",
    "Links": [
      {
        "Method": "GET",
        "Rel": "self",
        "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702"
      },
      {
        "Method": "DELETE",
        "Rel": "self",
        "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702"
      },
      {
        "Method": "PUT",
        "Rel": "self",
        "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702/confirmation"
      }
    ],
    "PrintMessage": [
      {
        "Position": "Top",
        "Message": "Transação autorizada"
      },
      {
        "Position": "Middle",
        "Message": "Informação adicional"
      },
      {
        "Position": "Bottom",
        "Message": "Obrigado e volte sempre!"
      }
    ],
    "ReceiptInformation": [
      {
        "Field": "MERCHANT_NAME",
        "Label": "NOME DO ESTABELECIMENTO",
        "Content": "Estabelecimento"
      },
      {
        "Field": "MERCHANT_ADDRESS",
        "Label": "ENDEREÇO DO ESTABELECIMENTO",
        "Content": "Rua Sem Saida, 0"
      },
      {
        "Field": "MERCHANT_CITY",
        "Label": "CIDADE DO ESTABELECIMENTO",
        "Content": "Cidade"
      },
      {
        "Field": "MERCHANT_STATE",
        "Label": "ESTADO DO ESTABELECIMENTO",
        "Content": "WA"
      },
      {
        "Field": "MERCHANT_CODE",
        "Label": "COD.ESTAB.",
        "Content": 1234567890123456
      },
      {
        "Field": "TERMINAL",
        "Label": "POS",
        "Content": 12345678
      },
      {
        "Field": "NSU",
        "Label": "DOC",
        "Content": 123456
      },
      {
        "Field": "DATE",
        "Label": "DATA",
        "Content": "01/01/20"
      },
      {
        "Field": "HOUR",
        "Label": "HORA",
        "Content": "01:01"
      },
      {
        "Field": "ISSUER_NAME",
        "Label": "EMISSOR",
        "Content": "NOME DO EMISSOR"
      },
      {
        "Field": "CARD_NUMBER",
        "Label": "CARTÃO",
        "Content": 5432123454321234
      },
      {
        "Field": "TRANSACTION_TYPE",
        "Label": "TIPO DE TRANSAÇÃO",
        "Content": "VENDA A CREDITO"
      },
      {
        "Field": "AUTHORIZATION_CODE",
        "Label": "AUTORIZAÇÃO",
        "Content": 123456
      },
      {
        "Field": "TRANSACTION_MODE",
        "Label": "MODO DA TRANSAÇÃO",
        "Content": "ONL"
      },
      {
        "Field": "INPUT_METHOD",
        "Label": "MODO DE ENTRADA",
        "Content": "X"
      },
      {
        "Field": "VALUE",
        "Label": "VALOR",
        "Content": "1,23"
      },
      {
        "Field": "SOFT_DESCRIPTOR",
        "Label": "SOFT DESCRIPTOR",
        "Content": "Simulado"
      }
    ],
    "Receipt": {
      "MerchantName": "Estabelecimento",
      "MerchantAddress": "Rua Sem Saida, 0",
      "MerchantCity": "Cidade",
      "MerchantState": "WA",
      "MerchantCode": 1234567890123456,
      "Terminal": 12345678,
      "Nsu": 123456,
      "Date": "01/01/20",
      "Hour": "01:01",
      "IssuerName": "NOME DO EMISSOR",
      "CardNumber": 5432123454321234,
      "TransactionType": "VENDA A CREDITO",
      "AuthorizationCode": 123456,
      "TransactionMode": "ONL",
      "InputMethod": "X",
      "Value": "1,23",
      "SoftDescriptor": "Simulado"
    },
    "RecurrentPayment": {
      "RecurrentPaymentId": "a6b719fa-a8df-ab11-4e1a-f4e50d5bd702",
      "ReasonCode": 0,
      "ReasonMessage": "Successful",
      "NextRecurrency": "2019-12-01",
      "EndDate": "2019-12-01",
      "Interval": 6
    },
    "SplitPayments": [
      {
        "SubordinateMerchantId": "491daf20-35f2-4379-874c-e7552ae8dc10",
        "Amount": 100,
        "Fares": {
          "Mdr": 5,
          "Fee": 0
        }
      },
      {
        "SubordinateMerchantId": "7e2846be-4e80-4f86-8ca9-eb35db6aea00",
        "Amount": 80,
        "Fares": {
          "Mdr": 3,
          "Fee": 1
        }
      }
    ],
    "SplitErrors": [
      {
        "Code": 326,
        "Message": "SubordinatePayment amount must be greater than zero"
      }
    ]
  }
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`MerchantOrderId`|String|---|---| Número do documento gerado automaticamente pelo terminal e incrementado de 1 a cada transação realizada no terminal. Aceita apenas valores numéricos de 1 a 15 dígitos.|
|`Customer.Name`|String|---|---|---|---|
|`Customer.Identity`|---|---|---|---|
|`Customer.IdentityType`|---|---|---|---|
|`Customer.Email`|---|---|---|---|
|`Customer.Birthday`|---|---|---|---|
|`Address.Street`|String|---|---|---|
|`Address.Number`|String|---|---|---|
|`Address.Complement`|String|---|---|---|
|`Address.ZipCode`|String|---|---|---|
|`Address.City`|String|---|---|---|
|`Address.State`|String|---|---|---|
|`Address.Country`|String|---|---||---|
|`DeliveryAddress.Street`|String|---|---|---|
|`DeliveryAddress.Number`|String|---|---|---|
|`DeliveryAddress.Complement`|String|---|---|---|
|`DeliveryAddress.ZipCode`|String|---|---|---|
|`DeliveryAddress.City`|String|---|---|---|
|`DeliveryAddress.State`|String|---|---|---|
|`DeliveryAddress.Country`|String|---|---|---|
||`Payment.Installments`|Integer|---|---|Default: 1 / Quantidade de Parcelas: Varia de 2 a 99 para transação de financiamento. Deve ser verificado os atributos maxOfPayments1, maxOfPayments2, maxOfPayments3 e minValOfPayments da tabela productTable.|
|`Payment.Interest`|String|---|---|Default: `ByMerchant` <br><br> Enum: `ByMerchant` `ByIssuer` <br><br> Tipo de Parcelamento: <br><br> - Se o bit 6 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 6 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento sem juros pode ser efetuado. <br><br> - Se o bit 7 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 7 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento com juros pode ser efetuado. Sem juros = “ByMerchant”; Com juros = “ByIssuer”.|
|`Payment.Capture`|Booleano|---|---|Default: false / Booleano que identifica que a autorização deve ser com captura automática. A autorização sem captura automática é conhecida também como pré-autorização.|
|`CreditCard.ExpirationDate`|String|MM/yyyy|Sim|Data de validade do cartão. <br><br>Dado obtido através do comando PP_GetCard na BC no momento da captura da transação.|
|`CreditCard.BrandId`|Integer|---|Sim|Identificação da bandeira obtida através do campo BrandId da PRODUCT TABLE.|
|`CreditCard.IssuerId`|Integer|---|Sim|Código do emissor obtido através do campo IssuerId da BIN TABLE.|
|`CreditCard.TruncateCardNumberWhenPrinting`|Booleano|---|---|Indica se o número do cartão será truncado no momento da impressão do comprovante. A solução de captura deve tomar essa decisão com base no `confParamOp03` presente nas tabelas BIN TABLE, PARAMETER TABLE e ISSUER TABLE.|
|`CreditCard.InputMode`|String|---|Sim|Enum: `Typed` `MagStripe` `Emv` <br><br>Identificação do modo de captura do cartão na transação. Essa informação deve ser obtida através do retorno da função PP_GetCard da BC. <br><br>"00" – Magnético <br><br>"01" - Moedeiro VISA Cash sobre TIBC v1 <br><br>"02" - Moedeiro VISA Cash sobre TIBC v3 <br><br>"03" – EMV com contato <br><br>"04" - Easy-Entry sobre TIBC v1 <br><br>"05" - Chip sem contato simulando tarja <br><br> "06" - EMV sem contato.|
|`CreditCard.AuthenticationMethod`|String|---|Sim|Enum: `NoPassword` `OnlineAuthentication` `OfflineAuthentication` <br><br>Método de autenticação <br><br>- Se o cartão foi lido a partir da digitação verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, a senha deve ser capturada e o authenticationMethod assume valor 2. Caso contrário, assume valor 1; <br><br>- Se o cartão foi lido a partir da trilha verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, deve ser verificado o bit 2 do mesmo campo. Se este estiver com valor 1 deve ser capturada a senha. Se estiver com valor 0 a captura da senha vai depender do último dígito do service code; <br><br>- Se o cartão foi lido através do chip EMV, o authenticationMethod será preenchido com base no retorno da função PP_GoOnChip(). No resultado PP_GoOnChip(), onde se o campo da posição 003 do retorno da PP_GoOnChip() estiver com valor 1 indica que o pin foi validado off-line, o authenticationMethod será 3. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valor 0, o authenticationMethod será 1. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valores 0 e 1 respectivamente, o authenticationMethod será 2. <br><br>1 - Sem senha = “NoPassword”; <br><br>2 - Senha online = “Online Authentication”; <br><br>3 - Senha off-line = “Offline Authentication”.|
|`CreditCard.EmvData`|String|---|---|Dados da transação EMV <br><br>Obtidos através do comando PP_GoOnChip na BC|
|`PinBlock.EncryptedPinBlock`|---|---|---|---|
|`PinBlock.EncryptionType`|String|---|---|---|
|`PinBlock.KsnIdentification`|String|---|---|---|
|`CreditCard.PanSequenceNumber`|Number|---|---|Número sequencial do cartão, utilizado para identificar a conta corrente do cartão adicional. Mandatório para transações com cartões Chip EMV e que possuam PAN Sequence Number (Tag 5F34).|
|`CreditCard.SaveCard`|---|---|---|---|
|`CreditCard.IsFallback`|---|---|---|---|
|`Payment.PaymentDateTime`|String|date-time|Sim|Data e Hora da captura da transação|
|`Payment.ServiceTaxAmount`|---|---|---|---|
|`Payment.SoftDescriptor`|String|13|---|Identificação do estabelecimento (nome reduzido) a ser impresso e identificado na fatura.|
|`Payment.ProductId`|Integer|---|Sim|Código do produto identificado através do bin do cartão.|
|`PinPadInformation.TerminalId`|String|---|Sim|Número Lógico definido no Concentrador Cielo.|
|`PinPadInformation.SerialNumber`|String|---|Sim|Número de Série do Equipamento.|
|`PinPadInformation.PhysicalCharacteristics`|String|---|Sim|Enum: `WithoutPinPad` `PinPadWithoutChipReader` `PinPadWithChipReaderWithoutSamModule` `PinPadWithChipReaderWithSamModule` `NotCertifiedPinPad` `PinPadWithChipReaderWithoutSamAndContactless` `PinPadWithChipReaderWithSamModuleAndContactless` <br><br> Sem PIN-pad = `WithoutPinPad`; <br><br> PIN-pad sem leitor de Chip = `PinpadWithoutChipReader`; <br><br>PIN-pad com leitor de Chip sem módulo SAM = `PinPadWithChipReaderWithoutSamModule`; <br><br> PIN-pad com leitor de Chip com módulo SAM = `PinPadWithChipReaderWithSamModule`; <br><br> PIN-pad não homologado = `NotCertifiedPinPad`; <br><br> PIN-pad com leitor de Chip sem SAM e Cartão Sem Contato = `PinpadWithChipReaderWithoutSamAndContactless`; <br><br> PIN-pad com leitor de Chip com SAM e Cartão Sem Contato = `PinpadWithChipReaderWithSamAndContactless`. <br><br><br> Obs. Caso a aplicação não consiga informar os dados acima, deve obter tais informações através do retorno da função PP_GetInfo() da BC.|
|`PinPadInformation.ReturnDataInfo`|String|---|Sim|Retorno da função PP_GetInfo() da biblioteca compartilhada|
|`Payment.Amount`|Integer(int64)|---|Sim|Valor da transação (1079 = R$10,79)|
|`Payment.ReceivedDate`|---|---|---|---|
|`Payment.CapturedAmount`|---|---|---|---|
|`Payment.Provider`|String|---|---|---|
|`Payment.ConfirmationStatus`|---|---|---|---|
|`Payment.InitializationVersion`|---|---|---|---|
|`Payment.EmvResponseData`|---|---|---|---|
|`Payment.Status`|---|---|---|---|
|`Payment.IsSplitted`|Booleano|---|---|---|
|`Payment.ReturnCode`|---|---|---|---|
|`Payment.ReturnMessage`|String|---|---|---|
|`Payment.PaymentId`|---|---|---|---|
|`Payment.Type`|String|---|Sim|Value: `PhysicalCreditCard` / Tipo da Transação|
|`Payment.Currency`|String|---|---|Default: "BRL" / Value: "BRL" / Moeda (Preencher com “BRL”)|
|`Payment.Country`|String|---|---|Default: "BRA" / Value: "BRA" / País (Preencher com “BRA”)|
|`Receipt.MerchantName`|---|---|---|---|
|`Receipt.MerchantAddress`|---|---|---|---|
|`Receipt.MerchantCity`|---|---|---|---|
|`Receipt.MerchantState`|---|---|---|---|
|`Receipt.MerchantCode`|---|---|---|---|
|`Receipt.Terminal`|---|---|---|---|
|`Receipt.Nsu`|---|---|---|---|
|`Receipt.Date`|---|---|---|---|
|`Receipt.Hour`|---|---|---|---|
|`Receipt.IssuerName`|---|---|---|---|
|`Receipt.CardNumber`|---|---|---|---|
|`Receipt.TransactionType`|---|---|---|---|
|`Receipt.AuthorizationCode`|---|---|---|---|
|`Receipt.TransactionMode`|---|---|---|---|
|`Receipt.InputMethod`|---|---|---|---|
|`Receipt.Value`|---|---|---|---|
|`Receipt.SoftDescriptor`|---|---|---|---|
|`RecurrentPayment.RecurrentPaymentId`|---|---|---|---|
|`RecurrentPayment.ReasonCode`|---|---|---|---|
|`RecurrentPayment.ReasonMessage`|---|---|---|---|
|`RecurrentPayment.NextRecurrency`|---|---|---|---|
|`RecurrentPayment.EndDate`|---|---|---|---|
|`RecurrentPayment.Interval`|---|---|---|---|
|`SplitPayments.SubordinateMerchantId`|---|---|---|---|
|`SplitPayments.Amount`|---|---|---|---|
|`SplitPayments.Fares.Mdr`|---|---|---|---|
|`SplitPayments.Fares.Fee`|---|---|---|---|
|`SplitErrors.Code`|---|---|---|---|
|`SplitErrors.Message`|---|---|---|---|

### Crédito digitado com cartão criptografado

#### Requisição

<aside class="request"><span class="method post">POST</span> <span class="endpoint">/1/physicalSales/</span></aside>

```json
{
  "MerchantOrderId": "1596226820548",
  "Customer": {
    "Name": "Comprador crédito completo",
    "Identity": "11225468954",
    "IdentityType": "CPF",
    "Email": "compradorteste@teste.com",
    "Birthday": "1991-01-02",
    "Address": {
      "Street": "Rua Teste",
      "Number": "123",
      "Complement": "AP 123",
      "ZipCode": "12345987",
      "City": "São Paulo",
      "State": "SP",
      "Country": "BRA"
    },
    "DeliveryAddress": {
      "Street": "Rua Teste",
      "Number": "123",
      "Complement": "AP 123",
      "ZipCode": "12345987",
      "City": "São Paulo",
      "State": "SP",
      "Country": "BRA"
    }
  },
  "Payment": {
    "Type": "PhysicalCreditCard",
    "SoftDescriptor": "Description",
    "PaymentDateTime": "2019-04-15T12:00:00Z",
    "Amount": 15798,
    "Installments": 1,
    "Capture": true,
    "Interest": "ByMerchant",
    "ProductId": 1,
    "CreditCard": {
      "CardNumber": "encrypted1234567812345678encrypted",
      "EncryptedCardData": {
          "EncryptionType": "DUKPT3DES",
          "CardNumberKSN": "KSNforCardNumber"
      },
      "ExpirationDate": "12/2020",
      "SecurityCodeStatus": "Collected",
      "SecurityCode": 1230,
      "BrandId": 1,
      "IssuerId": 2,
      "InputMode": "Typed",
      "AuthenticationMethod": "NoPassword",
      "TruncateCardNumberWhenPrinting": true,
      "SaveCard": false,
      "IsFallback": false
    },
    "PinPadInformation": {
      "TerminalId": "10000001",
      "SerialNumber": "ABC123",
      "PhysicalCharacteristics": "PinPadWithChipReaderWithSamModule",
      "ReturnDataInfo": "00"
    },
    "RecurrentPayment": {
      "EndDate": "2019-12-01",
      "Interval": "SemiAnnual"
    },
    "PaymentFacilitator": {
      "EstablishmentCode": "12345678901",
      "SubEstablishment": {
        "EstablishmentCode": "123456789012345",
        "Mcc": "1234",
        "Address": "1234567890abcdefghji12",
        "City": "1234567890abc",
        "State": "ab",
        "PostalCode": "123456789",
        "PhoneNumber": "1234567890123",
        "CountryCode": "076",
        "DocumentType": "Cpf",
        "DocumentNumber": "12345678901"
      }
    }
  }
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`MerchantOrderId`|String|---|---|Número do documento gerado automaticamente pelo terminal e incrementado de 1 a cada transação realizada no terminal. Aceita apenas valores numéricos de 1 a 15 dígitos.|
|`Payment.Type`|String|---|Sim|Value: PhysicalCreditCard / Tipo da Transação|
|`Payment.SoftDescriptor`|String |13|---|Identificação do estabelecimento (nome reduzido) a ser impresso e identificado na fatura.|
|`Payment.PaymentDateTime`|String |date-time|Sim|Data e Hora da captura da transação|
|`Payment.Amount`|Integer(int64)|---|Sim|Valor da transação (1079 = R$10,79)|
|`Payment.Capture`|Booleano|---|---|Default: false / Booleano que identifica que a autorização deve ser com captura automática. A autorização sem captura automática é conhecida também como pré-autorização.|
|`Payment.Installments`|Integer|---|---|Default: 1 / Quantidade de Parcelas: Varia de 2 a 99 para transação de financiamento. Deve ser verificado os atributos maxOfPayments1, maxOfPayments2, maxOfPayments3 e minValOfPayments da tabela productTable.|
|`Payment.Interest`|String|---|---|Default: `ByMerchant` <br><br> Enum: `ByMerchant` `ByIssuer` <br><br> Tipo de Parcelamento: <br><br> - Se o bit 6 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 6 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento sem juros pode ser efetuado. <br><br> - Se o bit 7 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 7 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento com juros pode ser efetuado. Sem juros = “ByMerchant”; Com juros = “ByIssuer”.|
|`Payment.ProductId`|Integer|---|Sim|Código do produto identificado através do bin do cartão.|
|`CreditCard.CardNumber`|String|---|---|Número do cartão (PAN) criptografado|
|`CreditCard.EncryptedCardData.EncryptionType`|String|---|Sim|Tipo de encriptação utilizada<br><br>Enum:<br><br>"DukptDes" = 1,<br><br>"MasterKey" = 2 <br<br>"Dukpt3Des" = 3,<br><br>"Dukpt3DesCBC" = 4|
|`CreditCard.EncryptedCardData.CardNumberKSN`|String|---|---|Identificador KSN da criptografia do número do cartão|
|`CreditCard.EncryptedCardData.IsDataInTLVFormat`|Bool|---|Não|Identifica se os dados criptografados estão no formato TLV (tag / length / value).|
|`CreditCard.EncryptedCardData.InitializationVector`|String|---|Sim|Vetor de inicialização da encryptação|
|`CreditCard.ExpirationDate`|String|MM/yyyy|Sim|Data de validade do cartão.<br><br>Dado obtido através do comando PP_GetCard na BC no momento da captura da transação.|
|`CreditCard.SecurityCodeStatus`|String|---|---|Enum: Collected Unreadable Nonexistent<br><br>Status da coleta de código de segurança (CVV)|
|`CreditCard.SecurityCode`|String|3 ou 4|---|Código de segurança (CVV)|
|`CreditCard.BrandId`|Integer|---|Sim|Identificação da bandeira obtida através do campo BrandId da PRODUCT TABLE.|
|`CreditCard.IssuerId`|Integer|---|Sim|Código do emissor obtido através do campo IssuerId da BIN TABLE.|
|`CreditCard.InputMode`|String|---|Sim|Enum: Typed MagStripe Emv<br><br>Identificação do modo de captura do cartão na transação. Essa informação deve ser obtida através do retorno da função<br><br> PP_GetCard da BC.<br><br>“00” – Magnético<br><br>“01” - Moedeiro VISA Cash sobre TIBC v1<br><br>“02” - Moedeiro VISA Cash sobre TIBC v3<br><br>“03” – EMV com contato<br><br>“04” - Easy-Entry sobre TIBC v1<br><br>“05” - Chip sem contato simulando tarja<br><br>“06” - EMV sem contato.<br><br>|
|`CreditCard.AuthenticationMethod`|String|---|Sim|Enum: NoPassword OnlineAuthentication OfflineAuthentication<br><br>Método de autenticação<br><br>- Se o cartão foi lido a partir da digitação verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, a senha deve ser capturada e o authenticationMethod assume valor 2. Caso contrário, assume valor 1;<br><br>- Se o cartão foi lido a partir da trilha verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, deve ser verificado o bit 2 do mesmo campo. Se este estiver com valor 1 deve ser capturada a senha. Se estiver com valor 0 a captura da senha vai depender do último dígito do service code;<br><br>- Se o cartão foi lido através do chip EMV, o authenticationMethod será preenchido com base no retorno da função PP_GoOnChip(). No resultado PP_GoOnChip(), onde se o campo da posição 003 do retorno da PP_GoOnChip() estiver com valor 1 indica que o pin foi validado off-line, o authenticationMethod será 3. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valor 0, o authenticationMethod será 1. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valores 0 e 1 respectivamente, o authenticationMethod será 2.<br><br>1 - Sem senha = “NoPassword”;<br><br>2 - Senha online = “Online Authentication”;<br><br>3 - Senha off-line = “Offline Authentication”.|
|`CreditCard.TruncateCardNumberWhenPrinting`|Booleano|---|---|Indica se o número do cartão será truncado no momento da impressão do comprovante. A solução de captura deve tomar essa decisão com base no confParamOp03 presente nas tabelas BIN TABLE, PARAMETER TABLE e ISSUER TABLE.|
|`CreditCard.SaveCard`|Booleano|---|---|Identifica se vai salvar/tokenizar o cartão.|
|`CreditCard.IsFallback`|Booleano|---|---|Identifica se é uma transação de fallback.|
|`PinPadInformation.TerminalId`|String|---|Sim|Número Lógico definido no Concentrador Cielo.|
|`PinPadInformation.SerialNumber`|String|---|Sim|Número de Série do Equipamento.|
|`PinPadInformation.PhysicalCharacteristics`|String|---|Sim|Enum: WithoutPinPad PinPadWithoutChipReader PinPadWithChipReaderWithoutSamModule<br><br> PinPadWithChipReaderWithSamModule NotCertifiedPinPad PinPadWithChipReaderWithoutSamAndContactless<br><br>PinPadWithChipReaderWithSamModuleAndContactless<br><br>SemPIN-pad = WithoutPinPad;PIN-pad sem leitor de Chip = PinpadWithoutChipReader;<br><br>PIN-pad com leitor de Chip sem módulo SAM = PinPadWithChipReaderWithoutSamModule;<br><br>PIN-pad com leitor de Chip com módulo SAM = PinPadWithChipReaderWithSamModule;<br><br>PIN-pad não homologado = NotCertifiedPinPad;<br><br>PIN-pad com leitor de Chip sem SAM e Cartão Sem Contato = PinpadWithChipReaderWithoutSamAndContactless;<br><br>PIN-pad com leitor de Chip com SAM e Cartão Sem Contato = PinpadWithChipReaderWithSamAndContactless.<br><br>Obs. Caso a aplicação não consiga informar os dados acima, deve obter tais informações através do retorno da função PP_GetInfo() da BC.|
|`PinPadInformation.ReturnDataInfo`|String|---|Sim|Retorno da função PP_GetInfo() da biblioteca compartilhada|

#### Resposta

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`MerchantOrderId`|String|---|---|Número do documento gerado automaticamente pelo terminal e incrementado de 1 a cada transação realizada no terminal. Aceita apenas valores numéricos de 1 a 15 dígitos.|
|`Customer.Name`|String|---|---|---|
|`Customer.Identity`|---|---|---|---|
|`Customer.IdentityType`|---|---|---|---|
|`Customer.Email`|---|---|---|---|
|`Customer.Birthday`|---|---|---|---|
|`Address.Street`|String|---|---|---|
|`Address.Number`|String|---|---|---|
|`Address.Complement`|String|---|---|---|
|`Address.ZipCode`|String|---|---|---|
|`Address.City`|String|---|---|---|
|`Address.State`|String|---|---|---|
|`Address.Country`|String|---|---|---|
|`DeliveryAddress.Street`|String|---|---|---|
|`DeliveryAddress.Number`|String|---|---|---|
|`DeliveryAddress.Complement`|String|---|---|---|
|`DeliveryAddress.ZipCode`|String|---|---|---|
|`DeliveryAddress.City`|String|---|---|---|
|`DeliveryAddress.State`|String|---|---|---|
|`DeliveryAddress.Country`|String|---|---|---|
|`Payment.Installments`|Integer|---|---|Default: 1 / Quantidade de Parcelas: Varia de 2 a 99 para transação de financiamento. Deve ser verificado os atributos maxOfPayments1, maxOfPayments2, maxOfPayments3 e minValOfPayments da tabela productTable.|
|`Payment.Interest`|String|---|---|Default: ByMerchant<br><br>Enum: ByMerchant ByIssuer<br><br>Tipo de Parcelamento:<br><br>- Se o bit 6 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 6 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento sem juros pode ser efetuado.<br><br>- Se o bit 7 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 7 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento com juros pode ser efetuado. Sem juros = “ByMerchant”; Com juros = “ByIssuer”.|
|`Payment.Capture`|Booleano|---|---|Default: false / Booleano que identifica que a autorização deve ser com captura automática. A autorização sem captura automática é conhecida também como pré-autorização.|
|`CreditCard.ExpirationDate`|String|MM/yyyy|Sim|Data de validade do cartão.Dado obtido através do comando PP_GetCard na BC no momento da captura da transação.|
|`CreditCard.BrandId`|Integer|---|Sim|Identificação da bandeira obtida através do campo BrandId da PRODUCT TABLE.|
|`CreditCard.IssuerId`|Integer|---|Sim|Código do emissor obtido através do campo IssuerId da BIN TABLE.|
|`CreditCard.TruncateCardNumberWhenPrinting`|Booleano|---|---|Indica se o número do cartão será truncado no momento da impressão do comprovante. A solução de captura deve tomar essa decisão com base no confParamOp03 presente nas tabelas BIN TABLE, PARAMETER TABLE e ISSUER TABLE.|
|`CreditCard.InputMode`|String|---|Sim|Enum: Typed MagStripe Emv<br><br>Identificação do modo de captura do cartão na transação. Essa informação deve ser obtida através do retorno da função PP_GetCard da BC.<br><br>“00” – Magnético<br><br>“01” - Moedeiro VISA Cash sobre TIBC v1<br><br>“02” - Moedeiro VISA Cash sobre TIBC v3<br><br>“03” – EMV com contato<br><br>“04” - Easy-Entry sobre TIBC v1<br><br>“05” - Chip sem contato simulando tarja<br><br>|“06” - EMV sem contato.
|`CreditCard.AuthenticationMethod`|String|---|Sim|Enum: `NoPassword` `OnlineAuthentication` `OfflineAuthentication` <br><br>Método de autenticação <br><br>- Se o cartão foi lido a partir da digitação verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, a senha deve ser capturada e o authenticationMethod assume valor 2. Caso contrário, assume valor 1; <br><br>- Se o cartão foi lido a partir da trilha verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, deve ser verificado o bit 2 do mesmo campo. Se este estiver com valor 1 deve ser capturada a senha. Se estiver com valor 0 a captura da senha vai depender do último dígito do service code; <br><br>- Se o cartão foi lido através do chip EMV, o authenticationMethod será preenchido com base no retorno da função PP_GoOnChip(). No resultado PP_GoOnChip(), onde se o campo da posição 003 do retorno da PP_GoOnChip() estiver com valor 1 indica que o pin foi validado off-line, o authenticationMethod será 3. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valor 0, o authenticationMethod será 1. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valores 0 e 1 respectivamente, o authenticationMethod será 2. <br><br>1 - Sem senha = “NoPassword”; <br><br>2 - Senha online = “Online Authentication”; <br><br>3 - Senha off-line = “Offline Authentication”.|
|`CreditCard.EmvData`|String|---|---|Dados da transação EMV<br><br>Obtidos através do comando PP_GoOnChip na BC|
|`PinBlock.EncryptedPinBlock`|---|---|---|---|
|`PinBlock.EncryptionType`|String|---|---|---|
|`PinBlock.KsnIdentification`|String|---|---|---|
|`CreditCard.PanSequenceNumber`|Number|---|---|Número sequencial do cartão, utilizado para identificar a conta corrente do cartão adicional. Mandatório para transações com cartões Chip EMV e que possuam PAN Sequence Number (Tag 5F34).|
|`CreditCard.SaveCard`|---|---|---|---|
|`CreditCard.IsFallback`|---|---|---|---|
|`Payment.PaymentDateTime`|String|date-time|Sim|Data e Hora da captura da transação|
|`Payment.ServiceTaxAmount`|---|---|---|---|
|`Payment.SoftDescriptor`|String|13|---|Identificação do estabelecimento (nome reduzido) a ser impresso e identificado na fatura.|
|`Payment.ProductId`|Integer|---|Sim|Código do produto identificado através do bin do cartão.|
|`PinPadInformation.TerminalId`|String|---|Sim|Número Lógico definido no Concentrador Cielo.|
|`PinPadInformation.SerialNumber`|String|---|Sim|Número de Série do Equipamento.|
|`PinPadInformation.PhysicalCharacteristics`|String|-|Sim|Enum: `WithoutPinPad` `PinPadWithoutChipReader` `PinPadWithChipReaderWithoutSamModule` `PinPadWithChipReaderWithSamModule` `NotCertifiedPinPad` `PinPadWithChipReaderWithoutSamAndContactless` `PinPadWithChipReaderWithSamModuleAndContactless` <br><br> Sem PIN-pad = `WithoutPinPad`; <br><br> PIN-pad sem leitor de Chip = `PinpadWithoutChipReader`; <br><br>PIN-pad com leitor de Chip sem módulo SAM = `PinPadWithChipReaderWithoutSamModule`; <br><br> PIN-pad com leitor de Chip com módulo SAM = `PinPadWithChipReaderWithSamModule`; <br><br> PIN-pad não homologado = `NotCertifiedPinPad`; <br><br> PIN-pad com leitor de Chip sem SAM e Cartão Sem Contato = `PinpadWithChipReaderWithoutSamAndContactless`; <br><br> PIN-pad com leitor de Chip com SAM e Cartão Sem Contato = `PinpadWithChipReaderWithSamAndContactless`. <br><br><br> Obs. Caso a aplicação não consiga informar os dados acima, deve obter tais informações através do retorno da função PP_GetInfo() da BC.|
|`PinPadInformation.ReturnDataInfo`|String|---|Sim|Retorno da função PP_GetInfo() da biblioteca compartilhada|
|`Payment.Amount|Integer`|(int64)|---|Sim|Valor da transação (1079 = R$10,79)|
|`Payment.ReceivedDate`|---|---|---|---|
|`Payment.CapturedAmount`|---|---|---|---|
|`Payment.Provider`|String|-|---|---|
|`Payment.ConfirmationStatus`|-|---|---|---|
|`Payment.InitializationVersion`|-|---|---|---|
|`Payment.EmvResponseData`|---|---|---|---|
|`Payment.Status`|---|---|---|---|
|`Payment.IsSplitted`|Booleano|---|---|---|---|
|`Payment.ReturnCode`|---|---|---|---|
|`Payment.ReturnMessage`|String|---|---|---|
|`Payment.PaymentId`|---|---|---|---|
|`Payment.Type`|String|---|Sim|Value: PhysicalCreditCard / Tipo da Transação|
|`Payment.Currency`|String|---|---|Default: “BRL” / Value: “BRL” / Moeda (Preencher com “BRL”)|
|`Payment.Country`|String|---|---|Default: “BRA” / Value: “BRA” / País (Preencher com “BRA”)|
|`Receipt.MerchantName`|---|---|---|---|
|`Receipt.MerchantAddress`|---|---|---|---|
|`Receipt.MerchantCity`|---|---|---|---|
|`Receipt.MerchantState`|---|---|---|---|
|`Receipt.MerchantCode`|---|---|---|---|
|`Receipt.Terminal`|---|---|---|---|
|`Receipt.Nsu`|---|---|---|---|
|`Receipt.Date`|---|---|---|---|
|`Receipt.Hour`|---|---|---|---|
|`Receipt.IssuerName`|---|---|---|---|
|`Receipt.CardNumber`|---|---|---|---|
|`Receipt.TransactionType`|---|---|---|---|
|`Receipt.AuthorizationCode`|---|---|---|---|
|`Receipt.TransactionMode`|---|---|---|---|
|`Receipt.InputMethod`|---|---|---|---|
|`Receipt.Value`|---|---|---|---|
|`Receipt.SoftDescriptor`|---|---|---|---|
|`RecurrentPayment.RecurrentPaymentId`|---|---|---|---|
|`RecurrentPayment.ReasonCode`|---|---|---|---|
|`RecurrentPayment.ReasonMessage`|---|---|---|---|
|`RecurrentPayment.NextRecurrency`|---|---|---|---|
|`RecurrentPayment.EndDate`|---|---|---|---|
|`RecurrentPayment.Interval`|---|---|---|---|
|`SplitPayments.SubordinateMerchantId`|---|---|---|---|
|`SplitPayments.Amount`|---|---|---|---|
|`SplitPayments.Fares.Mdr`|---|---|---|---|
|`SplitPayments.Fares.Fee`|---|---|---|---|
|`SplitErrors.Code`|---|---|---|---|
|`SplitErrors.Message`|---|---|---|---|

### Crédito por tarja e senha

#### Requisição

<aside class="request"><span class="method post">POST</span> <span class="endpoint">/1/physicalSales/</span></aside>

```json
{
  "MerchantOrderId": "201904150002",
  "Payment": {
    "Type": "PhysicalCreditCard",
    "SoftDescriptor": "Description",
    "PaymentDateTime": "2019-04-15T12:00:00Z",
    "Amount": 15798,
    "Installments": 1,
    "Capture": true,
    "Interest": "ByMerchant",
    "ProductId": 1,
    "CreditCard": {
      "ExpirationDate": "12/2020",
      "SecurityCodeStatus": "Nonexistent",
      "BrandId": 1,
      "IssuerId": 2,
      "InputMode": "MagStripe",
      "AuthenticationMethod": "OnlinePassword",
      "TrackOneData": "A1234567890123456^FULANO OLIVEIRA SA ^12345678901234567890123",
      "TrackTwoData": "0123456789012345=012345678901234",
      "PinBlock": {
        "EncryptedPinBlock": "2280F6BDFD0C038D",
        "EncryptionType": "Dukpt3Des",
        "KsnIdentification": "1231vg31fv231313123"
      },
      "PanSequenceNumber": 123,
      "SaveCard": false,
      "IsFallback": false
    },
    "PinPadInformation": {
      "TerminalId": "10000001",
      "SerialNumber": "ABC123",
      "PhysicalCharacteristics": "PinPadWithChipReaderWithSamModuleAndContactless",
      "ReturnDataInfo": "00"
    }
  }
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`MerchantOrderId`|String|---|---| Número do documento gerado automaticamente pelo terminal e incrementado de 1 a cada transação realizada no terminal. Aceita apenas valores numéricos de 1 a 15 dígitos.|
|`Payment.Type`|String|---|Sim|Value: PhysicalCreditCard / Tipo da Transação|
|`Payment.SoftDescriptor`|String|13|---|Identificação do estabelecimento (nome reduzido) a ser impresso e identificado na fatura.|
|`Payment.PaymentDateTime`|String|date-time|Sim|Data e Hora da captura da transação|
|`Payment.Amount`|Integer(int64)|---|Sim|Valor da transação (1079 = R$10,79)|
|`Payment.Capture`|Booleano|---|---|Default: false / Booleano que identifica que a autorização deve ser com captura automática. A autorização sem captura automática é conhecida também como pré-autorização.|
|`Payment.Installments`|Integer|---|---|Default: 1 / Quantidade de Parcelas: Varia de 2 a 99 para transação de financiamento. Deve ser verificado os atributos maxOfPayments1, maxOfPayments2, maxOfPayments3 e minValOfPayments da tabela productTable.|
|`Payment.Interest`|String|---|---|Default: `ByMerchant` <br><br> Enum: `ByMerchant` `ByIssuer` <br><br> Tipo de Parcelamento: <br><br> - Se o bit 6 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 6 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento sem juros pode ser efetuado. <br><br> - Se o bit 7 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 7 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento com juros pode ser efetuado. Sem juros = “ByMerchant”; Com juros = “ByIssuer”.|
|`Payment.ProductId`|Integer|---|Sim|Código do produto identificado através do bin do cartão.|
|`CreditCard.ExpirationDate`|String|MM/yyyy|Sim|Data de validade do cartão. <br><br>Dado obtido através do comando PP_GetCard na BC no momento da captura da transação.|
|`CreditCard.SecurityCodeStatus`|String|---|---|Enum: `Collected` `Unreadable` `Nonexistent` <br><br> Status da coleta de código de segurança (CVV)|
|`CreditCard.BrandId`|Integer|---|Sim|Identificação da bandeira obtida através do campo BrandId da PRODUCT TABLE.|
|`CreditCard.IssuerId`|Integer|---|Sim|Código do emissor obtido através do campo IssuerId da BIN TABLE.|
|`CreditCard.InputMode`|String|---|Sim|Enum: `Typed` `MagStripe` `Emv` <br><br>Identificação do modo de captura do cartão na transação. Essa informação deve ser obtida através do retorno da função PP_GetCard da BC. <br><br>"00" – Magnético <br><br>"01" - Moedeiro VISA Cash sobre TIBC v1 <br><br>"02" - Moedeiro VISA Cash sobre TIBC v3 <br><br>"03" – EMV com contato <br><br>"04" - Easy-Entry sobre TIBC v1 <br><br>"05" - Chip sem contato simulando tarja <br><br> "06" - EMV sem contato.|
|`CreditCard.AuthenticationMethod`|String|---|Sim|Enum: `NoPassword` `OnlineAuthentication` `OfflineAuthentication` <br><br>Método de autenticação <br><br>- Se o cartão foi lido a partir da digitação verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, a senha deve ser capturada e o authenticationMethod assume valor 2. Caso contrário, assume valor 1; <br><br>- Se o cartão foi lido a partir da trilha verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, deve ser verificado o bit 2 do mesmo campo. Se este estiver com valor 1 deve ser capturada a senha. Se estiver com valor 0 a captura da senha vai depender do último dígito do service code; <br><br>- Se o cartão foi lido através do chip EMV, o authenticationMethod será preenchido com base no retorno da função PP_GoOnChip(). No resultado PP_GoOnChip(), onde se o campo da posição 003 do retorno da PP_GoOnChip() estiver com valor 1 indica que o pin foi validado off-line, o authenticationMethod será 3. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valor 0, o authenticationMethod será 1. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valores 0 e 1 respectivamente, o authenticationMethod será 2. <br><br>1 - Sem senha = “NoPassword”; <br><br>2 - Senha online = “Online Authentication”; <br><br>3 - Senha off-line = “Offline Authentication”.|
|`CreditCard.TrackOneData`|String|---|---|Dados da trilha 1 <br><br>Obtidos através do comando PP_GetCard na BC no momento da captura da transação|
|`CreditCard.TrackTwoData`|String|---|---|Dados da trilha 2 <br><br>Obtidos através do comando PP_GetCard na BC no momento da captura da transação|
|`PinBlock.EncryptedPinBlock`|String|---|Sim|PINBlock Criptografado<br><br>- Para transações EMV, esse campo é obtido através do retorno da função PP_GoOnChip(), mais especificamente das posições 007 até a posição 022;<br><br>- Para transações digitadas e com tarja magnética, verificar as posições 001 até 016 do retorno da função PP_GetPin().|
|`PinBlock.EncryptionType`|String|---|Sim|Tipo de Criptografia<br><br>Enum:<br><br>"DukptDes"<br><br>"Dukpt3Des"<br><br>"MasterKey"|
|`PinBlock.KsnIdentification`|String|---|Sim|Identificação do KSN<br><br>- Para transações EMV esse campo é obtido através do retorno da função PP_GoOnChip() nas posições 023 até 042;<br><br>- Para transações digitadas e com tarja magnética, verificar as posições 017 até 036 do retorno da função PP_GetPin().|
|`CreditCard.PanSequenceNumber`|Number|---|---|Número sequencial do cartão, utilizado para identificar a conta corrente do cartão adicional. Mandatório para transações com cartões Chip EMV e que possuam PAN Sequence Number (Tag 5F34).|
|`CreditCard.SaveCard`|Booleano|---|---|Identifica se vai salvar/tokenizar o cartão.|
|`CreditCard.IsFallback`|Booleano|---|---|Identifica se é uma transação de fallback.|
|`PinPadInformation.TerminalId`|String|---|Sim|Número Lógico definido no Concentrador Cielo.|
|`PinPadInformation.SerialNumber`|String|---|Sim|Número de Série do Equipamento.|
|`PinPadInformation.PhysicalCharacteristics`|String|---|Sim|Enum: `WithoutPinPad` `PinPadWithoutChipReader` `PinPadWithChipReaderWithoutSamModule` `PinPadWithChipReaderWithSamModule` `NotCertifiedPinPad` `PinPadWithChipReaderWithoutSamAndContactless` `PinPadWithChipReaderWithSamModuleAndContactless` <br><br> Sem PIN-pad = `WithoutPinPad`; <br><br> PIN-pad sem leitor de Chip = `PinpadWithoutChipReader`; <br><br>PIN-pad com leitor de Chip sem módulo SAM = `PinPadWithChipReaderWithoutSamModule`; <br><br> PIN-pad com leitor de Chip com módulo SAM = `PinPadWithChipReaderWithSamModule`; <br><br> PIN-pad não homologado = `NotCertifiedPinPad`; <br><br> PIN-pad com leitor de Chip sem SAM e Cartão Sem Contato = `PinpadWithChipReaderWithoutSamAndContactless`; <br><br> PIN-pad com leitor de Chip com SAM e Cartão Sem Contato = `PinpadWithChipReaderWithSamAndContactless`. <br><br><br> Obs. Caso a aplicação não consiga informar os dados acima, deve obter tais informações através do retorno da função PP_GetInfo() da BC.|
|`PinPadInformation.ReturnDataInfo`|String|---|Sim|Retorno da função PP_GetInfo() da biblioteca compartilhada|

#### Resposta

```json
{
  "MerchantOrderId": "20180204",
  "Customer": {
    "Name": "Comprador crédito completo",
    "Identity": "11225468954",
    "IdentityType": "CPF",
    "Email": "compradorteste@teste.com",
    "Birthday": "1991-01-02",
    "Address": {
      "Street": "Rua Teste",
      "Number": "123",
      "Complement": "AP 123",
      "ZipCode": "12345987",
      "City": "São Paulo",
      "State": "SP",
      "Country": "BRA"
    },
    "DeliveryAddress": {
      "Street": "Rua Teste",
      "Number": "123",
      "Complement": "AP 123",
      "ZipCode": "12345987",
      "City": "São Paulo",
      "State": "SP",
      "Country": "BRA"
    }
  },
  "Payment": {
    "Installments": 1,
    "Interest": "ByMerchant",
    "Capture": true,
    "CreditCard": {
      "ExpirationDate": "12/2020",
      "BrandId": 1,
      "IssuerId": 2,
      "TruncateCardNumberWhenPrinting": true,
      "InputMode": "Emv",
      "AuthenticationMethod": "OnlineAuthentication",
      "EmvData": "112233445566778899011AABBC012D3456789E0123FF45678AB901234C5D112233445566778800",
      "PinBlock": {
        "EncryptedPinBlock": "2280F6BDFD0C038D",
        "EncryptionType": "Dukpt3Des",
        "KsnIdentification": "1231vg31fv231313123"
      },
      "PanSequenceNumber": 123,
      "SaveCard": false,
      "IsFallback": false
    },
    "PaymentDateTime": "2019-04-15T12:00:00Z",
    "ServiceTaxAmount": 0,
    "SoftDescriptor": "Description",
    "ProductId": 1,
    "PinPadInformation": {
      "TerminalId": "10000001",
      "SerialNumber": "ABC123",
      "PhysicalCharacteristics": "PinPadWithChipReaderWithSamModule",
      "ReturnDataInfo": "00"
    },
    "Amount": 15798,
    "ReceivedDate": "2019-04-15T12:00:00Z",
    "CapturedAmount": 15798,
    "Provider": "Cielo",
    "ConfirmationStatus": 0,
    "InitializationVersion": 1558708320029,
    "EmvResponseData": "123456789ABCD1345DEA",
    "Status": 2,
    "IsSplitted": false,
    "ReturnCode": 0,
    "ReturnMessage": "Successful",
    "PaymentId": "f15889ea-5719-4e1a-a2da-f4e50d5bd702",
    "Type": "PhysicalDebitCard",
    "Currency": "BRL",
    "Country": "BRA",
    "Links": [
      {
        "Method": "GET",
        "Rel": "self",
        "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702"
      },
      {
        "Method": "DELETE",
        "Rel": "self",
        "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702"
      },
      {
        "Method": "PUT",
        "Rel": "self",
        "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702/confirmation"
      }
    ],
    "PrintMessage": [
      {
        "Position": "Top",
        "Message": "Transação autorizada"
      },
      {
        "Position": "Middle",
        "Message": "Informação adicional"
      },
      {
        "Position": "Bottom",
        "Message": "Obrigado e volte sempre!"
      }
    ],
    "ReceiptInformation": [
      {
        "Field": "MERCHANT_NAME",
        "Label": "NOME DO ESTABELECIMENTO",
        "Content": "Estabelecimento"
      },
      {
        "Field": "MERCHANT_ADDRESS",
        "Label": "ENDEREÇO DO ESTABELECIMENTO",
        "Content": "Rua Sem Saida, 0"
      },
      {
        "Field": "MERCHANT_CITY",
        "Label": "CIDADE DO ESTABELECIMENTO",
        "Content": "Cidade"
      },
      {
        "Field": "MERCHANT_STATE",
        "Label": "ESTADO DO ESTABELECIMENTO",
        "Content": "WA"
      },
      {
        "Field": "MERCHANT_CODE",
        "Label": "COD.ESTAB.",
        "Content": 1234567890123456
      },
      {
        "Field": "TERMINAL",
        "Label": "POS",
        "Content": 12345678
      },
      {
        "Field": "NSU",
        "Label": "DOC",
        "Content": 123456
      },
      {
        "Field": "DATE",
        "Label": "DATA",
        "Content": "01/01/20"
      },
      {
        "Field": "HOUR",
        "Label": "HORA",
        "Content": "01:01"
      },
      {
        "Field": "ISSUER_NAME",
        "Label": "EMISSOR",
        "Content": "NOME DO EMISSOR"
      },
      {
        "Field": "CARD_NUMBER",
        "Label": "CARTÃO",
        "Content": 5432123454321234
      },
      {
        "Field": "TRANSACTION_TYPE",
        "Label": "TIPO DE TRANSAÇÃO",
        "Content": "VENDA A CREDITO"
      },
      {
        "Field": "AUTHORIZATION_CODE",
        "Label": "AUTORIZAÇÃO",
        "Content": 123456
      },
      {
        "Field": "TRANSACTION_MODE",
        "Label": "MODO DA TRANSAÇÃO",
        "Content": "ONL"
      },
      {
        "Field": "INPUT_METHOD",
        "Label": "MODO DE ENTRADA",
        "Content": "X"
      },
      {
        "Field": "VALUE",
        "Label": "VALOR",
        "Content": "1,23"
      },
      {
        "Field": "SOFT_DESCRIPTOR",
        "Label": "SOFT DESCRIPTOR",
        "Content": "Simulado"
      }
    ],
    "Receipt": {
      "MerchantName": "Estabelecimento",
      "MerchantAddress": "Rua Sem Saida, 0",
      "MerchantCity": "Cidade",
      "MerchantState": "WA",
      "MerchantCode": 1234567890123456,
      "Terminal": 12345678,
      "Nsu": 123456,
      "Date": "01/01/20",
      "Hour": "01:01",
      "IssuerName": "NOME DO EMISSOR",
      "CardNumber": 5432123454321234,
      "TransactionType": "VENDA A CREDITO",
      "AuthorizationCode": 123456,
      "TransactionMode": "ONL",
      "InputMethod": "X",
      "Value": "1,23",
      "SoftDescriptor": "Simulado"
    },
    "RecurrentPayment": {
      "RecurrentPaymentId": "a6b719fa-a8df-ab11-4e1a-f4e50d5bd702",
      "ReasonCode": 0,
      "ReasonMessage": "Successful",
      "NextRecurrency": "2019-12-01",
      "EndDate": "2019-12-01",
      "Interval": 6
    },
    "SplitPayments": [
      {
        "SubordinateMerchantId": "491daf20-35f2-4379-874c-e7552ae8dc10",
        "Amount": 100,
        "Fares": {
          "Mdr": 5,
          "Fee": 0
        }
      },
      {
        "SubordinateMerchantId": "7e2846be-4e80-4f86-8ca9-eb35db6aea00",
        "Amount": 80,
        "Fares": {
          "Mdr": 3,
          "Fee": 1
        }
      }
    ],
    "SplitErrors": [
      {
        "Code": 326,
        "Message": "SubordinatePayment amount must be greater than zero"
      }
    ]
  }
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`MerchantOrderId`|String|---|---| Número do documento gerado automaticamente pelo terminal e incrementado de 1 a cada transação realizada no terminal. Aceita apenas valores numéricos de 1 a 15 dígitos.|
|`Customer.Name`|String|---|---|---|
|`Customer.Identity`|---|---|---|---|
|`Customer.IdentityType`|---|---|---|---|
|`Customer.Email`|---|---|---|---|
|`Customer.Birthday`|---|---|---|---|
|`Address.Street`|String|---|---|---|
|`Address.Number`|String|---|---|---|
|`Address.Complement`|String|---|---||---|
|`Address.ZipCode`|String|---|---|---|
|`Address.City`|String|---|---|---|
|`Address.State`|String|---|---|---|
|`Address.Country`|String|---|---|---|
|`DeliveryAddress.Street`|String|---|---|---|
|`DeliveryAddress.Number`|String|---|---|---|
|`DeliveryAddress.Complement`|String|---|---||---|
|`DeliveryAddress.ZipCode`|String|---|---|---|
|`DeliveryAddress.City`|String|---|---|---|
|`DeliveryAddress.State`|String|---|---|---|
|`DeliveryAddress.Country`|String|---|---|---|
|`Payment.Installments`|Integer|---|---|Default: 1 / Quantidade de Parcelas: Varia de 2 a 99 para transação de financiamento. Deve ser verificado os atributos maxOfPayments1, maxOfPayments2, maxOfPayments3 e minValOfPayments da tabela productTable.|
|`Payment.Interest`|String|---|---|Default: `ByMerchant` <br><br> Enum: `ByMerchant` `ByIssuer` <br><br> Tipo de Parcelamento: <br><br> - Se o bit 6 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 6 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento sem juros pode ser efetuado. <br><br> - Se o bit 7 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 7 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento com juros pode ser efetuado. Sem juros = “ByMerchant”; Com juros = “ByIssuer”.|
|`Payment.Capture`|Booleano|---|---|Default: false / Booleano que identifica que a autorização deve ser com captura automática. A autorização sem captura automática é conhecida também como pré-autorização.|
|`CreditCard.ExpirationDate`|String|MM/yyyy|Sim|Data de validade do cartão. <br><br>Dado obtido através do comando PP_GetCard na BC no momento da captura da transação.|
|`CreditCard.BrandId`|Integer|---|Sim|Identificação da bandeira obtida através do campo BrandId da PRODUCT TABLE.|
|`CreditCard.IssuerId`|Integer|---|Sim|Código do emissor obtido através do campo IssuerId da BIN TABLE.|
|`CreditCard.TruncateCardNumberWhenPrinting`|Booleano|---|---|Indica se o número do cartão será truncado no momento da impressão do comprovante. A solução de captura deve tomar essa decisão com base no `confParamOp03` presente nas tabelas BIN TABLE, PARAMETER TABLE e ISSUER TABLE.|
|`CreditCard.InputMode`|String|---|Sim|Enum: `Typed` `MagStripe` `Emv` <br><br>Identificação do modo de captura do cartão na transação. Essa informação deve ser obtida através do retorno da função PP_GetCard da BC. <br><br>"00" – Magnético <br><br>"01" - Moedeiro VISA Cash sobre TIBC v1 <br><br>"02" - Moedeiro VISA Cash sobre TIBC v3 <br><br>"03" – EMV com contato <br><br>"04" - Easy-Entry sobre TIBC v1 <br><br>"05" - Chip sem contato simulando tarja <br><br> "06" - EMV sem contato.|
|`CreditCard.AuthenticationMethod`|String|---|Sim|Enum: `NoPassword` `OnlineAuthentication` `OfflineAuthentication` <br><br>Método de autenticação <br><br>- Se o cartão foi lido a partir da digitação verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, a senha deve ser capturada e o authenticationMethod assume valor 2. Caso contrário, assume valor 1; <br><br>- Se o cartão foi lido a partir da trilha verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, deve ser verificado o bit 2 do mesmo campo. Se este estiver com valor 1 deve ser capturada a senha. Se estiver com valor 0 a captura da senha vai depender do último dígito do service code; <br><br>- Se o cartão foi lido através do chip EMV, o authenticationMethod será preenchido com base no retorno da função PP_GoOnChip(). No resultado PP_GoOnChip(), onde se o campo da posição 003 do retorno da PP_GoOnChip() estiver com valor 1 indica que o pin foi validado off-line, o authenticationMethod será 3. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valor 0, o authenticationMethod será 1. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valores 0 e 1 respectivamente, o authenticationMethod será 2. <br><br>1 - Sem senha = “NoPassword”; <br><br>2 - Senha online = “Online Authentication”; <br><br>3 - Senha off-line = “Offline Authentication”.|
|`CreditCard.EmvData`|String|---|---|Dados da transação EMV <br><br>Obtidos através do comando PP_GoOnChip na BC|
|`PinBlock.EncryptedPinBlock`|String|---|Sim|PINBlock Criptografado<br><br>- Para transações EMV, esse campo é obtido através do retorno da função PP_GoOnChip(), mais especificamente das posições 007 até a posição 022;<br><br>- Para transações digitadas e com tarja magnética, verificar as posições 001 até 016 do retorno da função PP_GetPin().|
|`PinBlock.EncryptionType`|String|---|Sim|Tipo de Criptografia<br><br>Enum:<br><br>"DukptDes"<br><br>"Dukpt3Des"<br><br>"MasterKey"|
|`PinBlock.KsnIdentification`|String|---|Sim|Identificação do KSN<br><br>- Para transações EMV esse campo é obtido através do retorno da função PP_GoOnChip() nas posições 023 até 042;<br><br>- Para transações digitadas e com tarja magnética, verificar as posições 017 até 036 do retorno da função PP_GetPin().|
|`CreditCard.PanSequenceNumber`|Number|---|---|Número sequencial do cartão, utilizado para identificar a conta corrente do cartão adicional. Mandatório para transações com cartões Chip EMV e que possuam PAN Sequence Number (Tag 5F34).|
|`CreditCard.SaveCard`|Booleano|---|---|Identifica se vai salvar/tokenizar o cartão.|
|`CreditCard.IsFallback`|Booleano|---|---|Identifica se é uma transação de fallback.|
|`Payment.PaymentDateTime`|String|date-time|Sim|Data e Hora da captura da transação|
|`Payment.ServiceTaxAmount`|---|---|---|---|
|`Payment.SoftDescriptor`|String|13|---|Identificação do estabelecimento (nome reduzido) a ser impresso e identificado na fatura.|
|`Payment.ProductId`|Integer|---|Sim|Código do produto identificado através do bin do cartão.|
|`PinPadInformation.TerminalId`|String|---|Sim|Número Lógico definido no Concentrador Cielo.|
|`PinPadInformation.SerialNumber`|String|---|Sim|Número de Série do Equipamento.|
|`PinPadInformation.PhysicalCharacteristics`|String|---|Sim|Enum: `WithoutPinPad` `PinPadWithoutChipReader` `PinPadWithChipReaderWithoutSamModule` `PinPadWithChipReaderWithSamModule` `NotCertifiedPinPad` `PinPadWithChipReaderWithoutSamAndContactless` `PinPadWithChipReaderWithSamModuleAndContactless` <br><br> Sem PIN-pad = `WithoutPinPad`; <br><br> PIN-pad sem leitor de Chip = `PinpadWithoutChipReader`; <br><br>PIN-pad com leitor de Chip sem módulo SAM = `PinPadWithChipReaderWithoutSamModule`; <br><br> PIN-pad com leitor de Chip com módulo SAM = `PinPadWithChipReaderWithSamModule`; <br><br> PIN-pad não homologado = `NotCertifiedPinPad`; <br><br> PIN-pad com leitor de Chip sem SAM e Cartão Sem Contato = `PinpadWithChipReaderWithoutSamAndContactless`; <br><br> PIN-pad com leitor de Chip com SAM e Cartão Sem Contato = `PinpadWithChipReaderWithSamAndContactless`. <br><br><br> Obs. Caso a aplicação não consiga informar os dados acima, deve obter tais informações através do retorno da função PP_GetInfo() da BC.|
|`PinPadInformation.ReturnDataInfo`|String|---|Sim|Retorno da função PP_GetInfo() da biblioteca compartilhada|
|`Payment.Amount`|Integer(int64)|---|Sim|Valor da transação (1079 = R$10,79)|
|`Payment.ReceivedDate`|---|---|---|---|
|`Payment.CapturedAmount`|---|---|---|---|
|`Payment.Provider`|String|---|---|---|
|`Payment.ConfirmationStatus`|---|---|---|---|
|`Payment.InitializationVersion`|---|---|---|---|
|`Payment.EmvResponseData`|---|---|---|---|
|`Payment.Status`|---|---|---|---|
|`Payment.IsSplitted`|Booleano|---|---|---|
|`Payment.ReturnCode`|---|---|---|---|
|`Payment.ReturnMessage`|String|---|---|---|
|`Payment.PaymentId`|---|---|---|---|
|`Payment.Type`|String|---|Sim|Value: `PhysicalCreditCard` / Tipo da Transação|
|`Payment.Currency`|String|---|---|Default: "BRL" / Value: "BRL" / Moeda (Preencher com “BRL”)|
|`Payment.Country`|String|---|---|Default: "BRA" / Value: "BRA" / País (Preencher com “BRA”)|
|`Receipt.MerchantName`|---|---|---|---|
|`Receipt.MerchantAddress`|---|---|---|---|
|`Receipt.MerchantCity`|---|---|---|---|
|`Receipt.MerchantState`|---|---|---|---|
|`Receipt.MerchantCode`|---|---|---|---|
|`Receipt.Terminal`|---|---|---|---|
|`Receipt.Nsu`|---|---|---|---|
|`Receipt.Date`|---|---|---|---|
|`Receipt.Hour`|---|---|---|---|
|`Receipt.IssuerName`|---|---|---|---|
|`Receipt.CardNumber`|---|---|---|---|
|`Receipt.TransactionType`|---|---|---|---|
|`Receipt.AuthorizationCode`|---|---|---|---|
|`Receipt.TransactionMode`|---|---|---|---|
|`Receipt.InputMethod`|---|---|---|---|
|`Receipt.Value`|---|---|---|---|
|`Receipt.SoftDescriptor`|---|---|---|---|
|`RecurrentPayment.RecurrentPaymentId`|---|---|---|---|
|`RecurrentPayment.ReasonCode`|---|---|---|---|
|`RecurrentPayment.ReasonMessage`|---|---|---|---|
|`RecurrentPayment.NextRecurrency`|---|---|---|---|
|`RecurrentPayment.EndDate`|---|---|---|---|
|`RecurrentPayment.Interval`|---|---|---|---|
|`SplitPayments.SubordinateMerchantId`|---|---|---|---|
|`SplitPayments.Amount`|---|---|---|---|
|`SplitPayments.Fares.Mdr`|---|---|---|---|
|`SplitPayments.Fares.Fee`|---|---|---|---|
|`SplitErrors.Code`|---|---|---|---|
|`SplitErrors.Message`|---|---|---|---|

### Débito por tarja e senha

#### Requisição

<aside class="request"><span class="method post">POST</span> <span class="endpoint">/1/physicalSales/</span></aside>

```json
{
  "MerchantOrderId": "201904150003",
  "Payment": {
    "Type": "PhysicalDebitCard",
    "SoftDescriptor": "Description",
    "PaymentDateTime": "2019-04-15T12:00:00Z",
    "Amount": 15798,
    "ProductId": 1,
    "DebitCard": {
      "ExpirationDate": "12/2020",
      "SecurityCodeStatus": "Nonexistent",
      "BrandId": 1,
      "IssuerId": 2,
      "InputMode": "MagStripe",
      "AuthenticationMethod": "OnlinePassword",
      "TrackOneData": "A1234567890123456^FULANO OLIVEIRA SA ^12345678901234567890123",
      "TrackTwoData": "0123456789012345=012345678901234",
      "PinBlock": {
        "EncryptedPinBlock": "2280F6BDFD0C038D",
        "EncryptionType": "Dukpt3Des",
        "KsnIdentification": "1231vg31fv231313123"
      },
      "PanSequenceNumber": 123,
      "SaveCard": false,
      "IsFallback": false
    },
    "PinPadInformation": {
      "TerminalId": "10000001",
      "SerialNumber": "ABC123",
      "PhysicalCharacteristics": "PinPadWithChipReaderWithSamModule",
      "ReturnDataInfo": "00"
    }
  }
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`MerchantOrderId`|String|---|---| Número do documento gerado automaticamente pelo terminal e incrementado de 1 a cada transação realizada no terminal. Aceita apenas valores numéricos de 1 a 15 dígitos.|
|`Payment.Type`|String|---|Sim|Value: `PhysicalCreditCard` / Tipo da Transação|
|`Payment.SoftDescriptor`|String|13|---|Identificação do estabelecimento (nome reduzido) a ser impresso e identificado na fatura.|
|`Payment.PaymentDateTime`|String|date-time|Sim|Data e Hora da captura da transação|
|`Payment.Amount`|Integer(int64)|---|Sim|Valor da transação (1079 = R$10,79)|
|`Payment.ProductId`|Integer|---|Sim|Código do produto identificado através do bin do cartão.|
|`DebitCard.ExpirationDate`|String|MM/yyyy|Sim|Data de validade do cartão. <br><br>Dado obtido através do comando PP_GetCard na BC no momento da captura da transação.|
|`DebitCard.SecurityCodeStatus`|String|---|---|Enum: `Collected` `Unreadable` `Nonexistent` <br><br> Status da coleta de código de segurança (CVV)|
|`DebitCard.BrandId`|Integer|---|Sim|Identificação da bandeira obtida através do campo BrandId da PRODUCT TABLE.|
|`DebitCard.IssuerId`|Integer|---|Sim|Código do emissor obtido através do campo IssuerId da BIN TABLE.|
|`DebitCard.InputMode`|String|---|Sim|Enum: `Typed` `MagStripe` `Emv` <br><br>Identificação do modo de captura do cartão na transação. Essa informação deve ser obtida através do retorno da função PP_GetCard da BC. <br><br>"00" – Magnético <br><br>"01" - Moedeiro VISA Cash sobre TIBC v1 <br><br>"02" - Moedeiro VISA Cash sobre TIBC v3 <br><br>"03" – EMV com contato <br><br>"04" - Easy-Entry sobre TIBC v1 <br><br>"05" - Chip sem contato simulando tarja <br><br> "06" - EMV sem contato.|
|`DebitCard.AuthenticationMethod`|String|---|Sim|Enum: `NoPassword` `OnlineAuthentication` `OfflineAuthentication` <br><br>Método de autenticação <br><br>- Se o cartão foi lido a partir da digitação verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, a senha deve ser capturada e o authenticationMethod assume valor 2. Caso contrário, assume valor 1; <br><br>- Se o cartão foi lido a partir da trilha verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, deve ser verificado o bit 2 do mesmo campo. Se este estiver com valor 1 deve ser capturada a senha. Se estiver com valor 0 a captura da senha vai depender do último dígito do service code; <br><br>- Se o cartão foi lido através do chip EMV, o authenticationMethod será preenchido com base no retorno da função PP_GoOnChip(). No resultado PP_GoOnChip(), onde se o campo da posição 003 do retorno da PP_GoOnChip() estiver com valor 1 indica que o pin foi validado off-line, o authenticationMethod será 3. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valor 0, o authenticationMethod será 1. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valores 0 e 1 respectivamente, o authenticationMethod será 2. <br><br>1 - Sem senha = “NoPassword”; <br><br>2 - Senha online = “Online Authentication”; <br><br>3 - Senha off-line = “Offline Authentication”.|
|`DebitCard.TrackOneData`|String|---|---|Dados da trilha 1 <br><br>Obtidos através do comando PP_GetCard na BC no momento da captura da transação|
|`DebitCard.TrackTwoData`|String|---|---|Dados da trilha 2 <br><br>Obtidos através do comando PP_GetCard na BC no momento da captura da transação|
|`PinBlock.EncryptedPinBlock`|String|---|Sim|PINBlock Criptografado<br><br>- Para transações EMV, esse campo é obtido através do retorno da função PP_GoOnChip(), mais especificamente das posições 007 até a posição 022;<br><br>- Para transações digitadas e com tarja magnética, verificar as posições 001 até 016 do retorno da função PP_GetPin().|
|`PinBlock.EncryptionType`|String|---|Sim|Tipo de Criptografia<br><br>Enum:<br><br>"DukptDes"<br><br>"Dukpt3Des"<br><br>"MasterKey"|
|`PinBlock.KsnIdentification`|String|---|Sim|Identificação do KSN<br><br>- Para transações EMV esse campo é obtido através do retorno da função PP_GoOnChip() nas posições 023 até 042;<br><br>- Para transações digitadas e com tarja magnética, verificar as posições 017 até 036 do retorno da função PP_GetPin().|
|`DebitCard.PanSequenceNumber`|Number|---|---|Número sequencial do cartão, utilizado para identificar a conta corrente do cartão adicional. Mandatório para transações com cartões Chip EMV e que possuam PAN Sequence Number (Tag 5F34).|
|`DebitCard.SaveCard`|Booleano|---|---|Identifica se vai salvar/tokenizar o cartão.|
|`DebitCard.IsFallback`|Booleano|---|---|Identifica se é uma transação de fallback.|
|`PinPadInformation.TerminalId`|String|---|Sim|Número Lógico definido no Concentrador Cielo.|
|`PinPadInformation.SerialNumber`|String|---|Sim|Número de Série do Equipamento.|
|`PinPadInformation.PhysicalCharacteristics`|String|---|Sim|Enum: `WithoutPinPad` `PinPadWithoutChipReader` `PinPadWithChipReaderWithoutSamModule` `PinPadWithChipReaderWithSamModule` `NotCertifiedPinPad` `PinPadWithChipReaderWithoutSamAndContactless` `PinPadWithChipReaderWithSamModuleAndContactless` <br><br> Sem PIN-pad = `WithoutPinPad`; <br><br> PIN-pad sem leitor de Chip = `PinpadWithoutChipReader`; <br><br>PIN-pad com leitor de Chip sem módulo SAM = `PinPadWithChipReaderWithoutSamModule`; <br><br> PIN-pad com leitor de Chip com módulo SAM = `PinPadWithChipReaderWithSamModule`; <br><br> PIN-pad não homologado = `NotCertifiedPinPad`; <br><br> PIN-pad com leitor de Chip sem SAM e Cartão Sem Contato = `PinpadWithChipReaderWithoutSamAndContactless`; <br><br> PIN-pad com leitor de Chip com SAM e Cartão Sem Contato = `PinpadWithChipReaderWithSamAndContactless`. <br><br><br> Obs. Caso a aplicação não consiga informar os dados acima, deve obter tais informações através do retorno da função PP_GetInfo() da BC.|
|`PinPadInformation.ReturnDataInfo`|String|---|Sim|Retorno da função PP_GetInfo() da biblioteca compartilhada|

#### Resposta

```json
{
  "MerchantOrderId": "20180204",
  "Customer": {
    "Name": "Comprador crédito completo",
    "Identity": "11225468954",
    "IdentityType": "CPF",
    "Email": "compradorteste@teste.com",
    "Birthday": "1991-01-02",
    "Address": {
      "Street": "Rua Teste",
      "Number": "123",
      "Complement": "AP 123",
      "ZipCode": "12345987",
      "City": "São Paulo",
      "State": "SP",
      "Country": "BRA"
    },
    "DeliveryAddress": {
      "Street": "Rua Teste",
      "Number": "123",
      "Complement": "AP 123",
      "ZipCode": "12345987",
      "City": "São Paulo",
      "State": "SP",
      "Country": "BRA"
    }
  },
  "Payment": {
    "Installments": 1,
    "Interest": "ByMerchant",
    "Capture": true,
    "CreditCard": {
      "ExpirationDate": "12/2020",
      "BrandId": 1,
      "IssuerId": 2,
      "TruncateCardNumberWhenPrinting": true,
      "InputMode": "Emv",
      "AuthenticationMethod": "OnlineAuthentication",
      "EmvData": "112233445566778899011AABBC012D3456789E0123FF45678AB901234C5D112233445566778800",
      "PinBlock": {
        "EncryptedPinBlock": "2280F6BDFD0C038D",
        "EncryptionType": "Dukpt3Des",
        "KsnIdentification": "1231vg31fv231313123"
      },
      "PanSequenceNumber": 123,
      "SaveCard": false,
      "IsFallback": false
    },
    "PaymentDateTime": "2019-04-15T12:00:00Z",
    "ServiceTaxAmount": 0,
    "SoftDescriptor": "Description",
    "ProductId": 1,
    "PinPadInformation": {
      "TerminalId": "10000001",
      "SerialNumber": "ABC123",
      "PhysicalCharacteristics": "PinPadWithChipReaderWithSamModule",
      "ReturnDataInfo": "00"
    },
    "Amount": 15798,
    "ReceivedDate": "2019-04-15T12:00:00Z",
    "CapturedAmount": 15798,
    "Provider": "Cielo",
    "ConfirmationStatus": 0,
    "InitializationVersion": 1558708320029,
    "EmvResponseData": "123456789ABCD1345DEA",
    "Status": 2,
    "IsSplitted": false,
    "ReturnCode": 0,
    "ReturnMessage": "Successful",
    "PaymentId": "f15889ea-5719-4e1a-a2da-f4e50d5bd702",
    "Type": "PhysicalDebitCard",
    "Currency": "BRL",
    "Country": "BRA",
    "Links": [
      {
        "Method": "GET",
        "Rel": "self",
        "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702"
      },
      {
        "Method": "DELETE",
        "Rel": "self",
        "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702"
      },
      {
        "Method": "PUT",
        "Rel": "self",
        "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702/confirmation"
      }
    ],
    "PrintMessage": [
      {
        "Position": "Top",
        "Message": "Transação autorizada"
      },
      {
        "Position": "Middle",
        "Message": "Informação adicional"
      },
      {
        "Position": "Bottom",
        "Message": "Obrigado e volte sempre!"
      }
    ],
    "ReceiptInformation": [
      {
        "Field": "MERCHANT_NAME",
        "Label": "NOME DO ESTABELECIMENTO",
        "Content": "Estabelecimento"
      },
      {
        "Field": "MERCHANT_ADDRESS",
        "Label": "ENDEREÇO DO ESTABELECIMENTO",
        "Content": "Rua Sem Saida, 0"
      },
      {
        "Field": "MERCHANT_CITY",
        "Label": "CIDADE DO ESTABELECIMENTO",
        "Content": "Cidade"
      },
      {
        "Field": "MERCHANT_STATE",
        "Label": "ESTADO DO ESTABELECIMENTO",
        "Content": "WA"
      },
      {
        "Field": "MERCHANT_CODE",
        "Label": "COD.ESTAB.",
        "Content": 1234567890123456
      },
      {
        "Field": "TERMINAL",
        "Label": "POS",
        "Content": 12345678
      },
      {
        "Field": "NSU",
        "Label": "DOC",
        "Content": 123456
      },
      {
        "Field": "DATE",
        "Label": "DATA",
        "Content": "01/01/20"
      },
      {
        "Field": "HOUR",
        "Label": "HORA",
        "Content": "01:01"
      },
      {
        "Field": "ISSUER_NAME",
        "Label": "EMISSOR",
        "Content": "NOME DO EMISSOR"
      },
      {
        "Field": "CARD_NUMBER",
        "Label": "CARTÃO",
        "Content": 5432123454321234
      },
      {
        "Field": "TRANSACTION_TYPE",
        "Label": "TIPO DE TRANSAÇÃO",
        "Content": "VENDA A CREDITO"
      },
      {
        "Field": "AUTHORIZATION_CODE",
        "Label": "AUTORIZAÇÃO",
        "Content": 123456
      },
      {
        "Field": "TRANSACTION_MODE",
        "Label": "MODO DA TRANSAÇÃO",
        "Content": "ONL"
      },
      {
        "Field": "INPUT_METHOD",
        "Label": "MODO DE ENTRADA",
        "Content": "X"
      },
      {
        "Field": "VALUE",
        "Label": "VALOR",
        "Content": "1,23"
      },
      {
        "Field": "SOFT_DESCRIPTOR",
        "Label": "SOFT DESCRIPTOR",
        "Content": "Simulado"
      }
    ],
    "Receipt": {
        "MerchantName": "Estabelecimento",
        "MerchantAddress": "Rua Sem Saida, 0",
        "MerchantCity": "Cidade",
        "MerchantState": "WA",
        "MerchantCode": 1234567890123456,
        "Terminal": 12345678,
        "Nsu": 123456,
        "Date": "01/01/20",
        "Hour": "01:01",
        "IssuerName": "NOME DO EMISSOR",
        "CardNumber": 5432123454321234,
        "TransactionType": "VENDA A CREDITO",
        "AuthorizationCode": 123456,
        "TransactionMode": "ONL",
        "InputMethod": "X",
        "Value": "1,23",
        "SoftDescriptor": "Simulado"
      },
        "RecurrentPayment": {
        "RecurrentPaymentId": "a6b719fa-a8df-ab11-4e1a-f4e50d5bd702",
        "ReasonCode": 0,
        "ReasonMessage": "Successful",
        "NextRecurrency": "2019-12-01",
        "EndDate": "2019-12-01",
        "Interval": 6
      },
        "SplitPayments": [
        {
           "SubordinateMerchantId": "491daf20-35f2-4379-874c-e7552ae8dc10",
           "Amount": 100,
           "Fares": {
              "Mdr": 5,
              "Fee": 0
           }
        },
        {
           "SubordinateMerchantId": "7e2846be-4e80-4f86-8ca9-eb35db6aea00",
           "Amount": 80,
           "Fares": {
           "Mdr": 3,
           "Fee": 1
        }
        }
      ],
        "SplitErrors": [
      {
        "Code": 326,
        "Message": "SubordinatePayment amount must be greater than zero"
      }
    ]
  }
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`MerchantOrderId`|String|---|---| Número do documento gerado automaticamente pelo terminal e incrementado de 1 a cada transação realizada no terminal. Aceita apenas valores numéricos de 1 a 15 dígitos.|
|`Customer.Name`|String|---|---|---|
|`Customer.Identity`|---|---|---|---|
|`Customer.IdentityType`|---|---|---|---|
|`Customer.Email`|---|---|---|---|
|`Customer.Birthday`|---|---|---|---|
|`Address.Street`|String|---|---|---|
|`Address.Number`|String|---|---|---|
|`Address.Complement`|String|---|---|---|
|`Address.ZipCode`|String|---|---|---|
|`Address.City`|String|---|---|---|
|`Address.State`|String|---|---|---|
|`Address.Country`|String|---|---|---|
|`DeliveryAddress.Street`|String|---|---|---|
|`DeliveryAddress.Number`|String|---|---|---|
|`DeliveryAddress.Complement`|String|---|---|---|
|`DeliveryAddress.ZipCode`|String|---|---|---|
|`DeliveryAddress.City`|String|---|---|---|
|`DeliveryAddress.State`|String|---|---|---|
|`DeliveryAddress.Country`|String|---|---|---|
|`Payment.Installments`|Integer|---|---|Default: 1 / Quantidade de Parcelas: Varia de 2 a 99 para transação de financiamento. Deve ser verificado os atributos maxOfPayments1, maxOfPayments2, maxOfPayments3 e minValOfPayments da tabela productTable.|
|`Payment.Interest`|String|---|---|Default: `ByMerchant` <br><br> Enum: `ByMerchant` `ByIssuer` <br><br> Tipo de Parcelamento: <br><br> - Se o bit 6 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 6 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento sem juros pode ser efetuado. <br><br> - Se o bit 7 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 7 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento com juros pode ser efetuado. Sem juros = “ByMerchant”; Com juros = “ByIssuer”.|
|`Payment.Capture`|Booleano|---|---|Default: false / Booleano que identifica que a autorização deve ser com captura automática. A autorização sem captura automática é conhecida também como pré-autorização.|
|`DebitCard.ExpirationDate`|String|MM/yyyy|Sim|Data de validade do cartão. <br><br>Dado obtido através do comando PP_GetCard na BC no momento da captura da transação.|
|`DebitCard.BrandId`|Integer|---|Sim|Identificação da bandeira obtida através do campo BrandId da PRODUCT TABLE.|
|`DebitCard.IssuerId`|Integer|---|Sim|Código do emissor obtido através do campo IssuerId da BIN TABLE.|
|`DebitCard.TruncateCardNumberWhenPrinting`|Booleano|---|---|Indica se o número do cartão será truncado no momento da impressão do comprovante. A solução de captura deve tomar essa decisão com base no `confParamOp03` presente nas tabelas BIN TABLE, PARAMETER TABLE e ISSUER TABLE.|
|`DebitCard.InputMode`|String|---|Sim|Enum: `Typed` `MagStripe` `Emv` <br><br>Identificação do modo de captura do cartão na transação. Essa informação deve ser obtida através do retorno da função PP_GetCard da BC. <br><br>"00" – Magnético <br><br>"01" - Moedeiro VISA Cash sobre TIBC v1 <br><br>"02" - Moedeiro VISA Cash sobre TIBC v3 <br><br>"03" – EMV com contato <br><br>"04" - Easy-Entry sobre TIBC v1 <br><br>"05" - Chip sem contato simulando tarja <br><br> "06" - EMV sem contato.|
|`DebitCard.AuthenticationMethod`|String|---|Sim|Enum: `NoPassword` `OnlineAuthentication` `OfflineAuthentication` <br><br>Método de autenticação <br><br>- Se o cartão foi lido a partir da digitação verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, a senha deve ser capturada e o authenticationMethod assume valor 2. Caso contrário, assume valor 1; <br><br>- Se o cartão foi lido a partir da trilha verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, deve ser verificado o bit 2 do mesmo campo. Se este estiver com valor 1 deve ser capturada a senha. Se estiver com valor 0 a captura da senha vai depender do último dígito do service code; <br><br>- Se o cartão foi lido através do chip EMV, o authenticationMethod será preenchido com base no retorno da função PP_GoOnChip(). No resultado PP_GoOnChip(), onde se o campo da posição 003 do retorno da PP_GoOnChip() estiver com valor 1 indica que o pin foi validado off-line, o authenticationMethod será 3. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valor 0, o authenticationMethod será 1. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valores 0 e 1 respectivamente, o authenticationMethod será 2. <br><br>1 - Sem senha = “NoPassword”; <br><br>2 - Senha online = “Online Authentication”; <br><br>3 - Senha off-line = “Offline Authentication”.|
|`DebitCard.EmvData`|String|---|---|Dados da transação EMV <br><br>Obtidos através do comando PP_GoOnChip na BC|
|`PinBlock.EncryptedPinBlock`|String|---|Sim|PINBlock Criptografado<br><br>- Para transações EMV, esse campo é obtido através do retorno da função PP_GoOnChip(), mais especificamente das posições 007 até a posição 022;<br><br>- Para transações digitadas e com tarja magnética, verificar as posições 001 até 016 do retorno da função PP_GetPin().|
|`PinBlock.EncryptionType`|String|---|Sim|Tipo de Criptografia<br><br>Enum:<br><br>"DukptDes"<br><br>"Dukpt3Des"<br><br>"MasterKey"|
|`PinBlock.KsnIdentification`|String|---|Sim|Identificação do KSN<br><br>- Para transações EMV esse campo é obtido através do retorno da função PP_GoOnChip() nas posições 023 até 042;<br><br>- Para transações digitadas e com tarja magnética, verificar as posições 017 até 036 do retorno da função PP_GetPin().|
|`DebitCard.PanSequenceNumber`|Number|---|---|Número sequencial do cartão, utilizado para identificar a conta corrente do cartão adicional. Mandatório para transações com cartões Chip EMV e que possuam PAN Sequence Number (Tag 5F34).|
|`DebitCard.SaveCard`|Booleano|---|---|Identifica se vai salvar/tokenizar o cartão.|
|`DebitCard.IsFallback`|Booleano|---|---|Identifica se é uma transação de fallback.|
|`Payment.PaymentDateTime`|String|date-time|Sim|Data e Hora da captura da transação|
|`Payment.ServiceTaxAmount`|---|---|---|---|
|`Payment.SoftDescriptor`|String|13|---|Identificação do estabelecimento (nome reduzido) a ser impresso e identificado na fatura.|
|`Payment.ProductId`|Integer|---|Sim|Código do produto identificado através do bin do cartão.|
|`PinPadInformation.TerminalId`|String|---|Sim|Número Lógico definido no Concentrador Cielo.|
|`PinPadInformation.SerialNumber`|String|---|Sim|Número de Série do Equipamento.|
|`PinPadInformation.PhysicalCharacteristics`|String|---|Sim|Enum: `WithoutPinPad` `PinPadWithoutChipReader` `PinPadWithChipReaderWithoutSamModule` `PinPadWithChipReaderWithSamModule` `NotCertifiedPinPad` `PinPadWithChipReaderWithoutSamAndContactless` `PinPadWithChipReaderWithSamModuleAndContactless` <br><br> Sem PIN-pad = `WithoutPinPad`; <br><br> PIN-pad sem leitor de Chip = `PinpadWithoutChipReader`; <br><br>PIN-pad com leitor de Chip sem módulo SAM = `PinPadWithChipReaderWithoutSamModule`; <br><br> PIN-pad com leitor de Chip com módulo SAM = `PinPadWithChipReaderWithSamModule`; <br><br> PIN-pad não homologado = `NotCertifiedPinPad`; <br><br> PIN-pad com leitor de Chip sem SAM e Cartão Sem Contato = `PinpadWithChipReaderWithoutSamAndContactless`; <br><br> PIN-pad com leitor de Chip com SAM e Cartão Sem Contato = `PinpadWithChipReaderWithSamAndContactless`. <br><br><br> Obs. Caso a aplicação não consiga informar os dados acima, deve obter tais informações através do retorno da função PP_GetInfo() da BC.|
|`PinPadInformation.ReturnDataInfo`|String|---|Sim|Retorno da função PP_GetInfo() da biblioteca compartilhada|
|`Payment.Amount`|Integer(int64)|---|Sim|Valor da transação (1079 = R$10,79)|
|`Payment.ReceivedDate`|---|---|---|---|
|`Payment.CapturedAmount`|---|---|---|---|
|`Payment.Provider`|String|---|---|---|
|`Payment.ConfirmationStatus`|---|---|---|---|
|`Payment.InitializationVersion`|---|---|---|---|
|`Payment.EmvResponseData`|---|---|---|---|
|`Payment.Status`|---|---|---|---|
|`Payment.IsSplitted`|Booleano|---|---|---|
|`Payment.ReturnCode`|---|---|---|---|
|`Payment.ReturnMessage`|String|---|---|---|
|`Payment.PaymentId`|---|---|---|---|
|`Payment.Type`|String|---|Sim|Value: `PhysicalCreditCard` / Tipo da Transação|
|`Payment.Currency`|String|---|---|Default: "BRL" / Value: "BRL" / Moeda (Preencher com “BRL”)|
|`Payment.Country`|String|---|---|Default: "BRA" / Value: "BRA" / País (Preencher com “BRA”)|
|`Receipt.MerchantName`|---|---|---|---|
|`Receipt.MerchantAddress`|---|---|---|---|
|`Receipt.MerchantCity`|---|---|---|---|
|`Receipt.MerchantState`|---|---|---|---|
|`Receipt.MerchantCode`|---|---|---|---|
|`Receipt.Terminal`|---|---|---|---|
|`Receipt.Nsu`|---|---|---|---|
|`Receipt.Date`|---|---|---|---|
|`Receipt.Hour`|---|---|---|---|
|`Receipt.IssuerName`|---|---|---|---|
|`Receipt.CardNumber`|---|---|---|---|
|`Receipt.TransactionType`|---|---|---|---|
|`Receipt.AuthorizationCode`|---|---|---|---|
|`Receipt.TransactionMode`|---|---|---|---|
|`Receipt.InputMethod`|---|---|---|---|
|`Receipt.Value`|---|---|---|---|
|`Receipt.SoftDescriptor`|---|---|---|---|
|`RecurrentPayment.RecurrentPaymentId`|---|---|---|---|
|`RecurrentPayment.ReasonCode`|---|---|---|---|
|`RecurrentPayment.ReasonMessage`|---|---|---|---|
|`RecurrentPayment.NextRecurrency`|---|---|---|---|
|`RecurrentPayment.EndDate`|---|---|---|---|
|`RecurrentPayment.Interval`|---|---|---|---|
|`SplitPayments.SubordinateMerchantId`|---|---|---|---|
|`SplitPayments.Amount`|---|---|---|---|
|`SplitPayments.Fares.Mdr`|---|---|---|---|
|`SplitPayments.Fares.Fee`|---|---|---|---|
|`SplitErrors.Code`|---|---|---|---|
|`SplitErrors.Message`|---|---|---|---|

### Débito por tarja com cartão criptografado

#### Requisição

<aside class="request"><span class="method post">POST</span> <span class="endpoint">/1/physicalSales/</span></aside>

```json
{
  "MerchantOrderId": "1596226820548",
  "Payment": {
    "Type": "PhysicalDebitCard",
    "SoftDescriptor": "Description",
    "PaymentDateTime": "2019-04-15T12:00:00Z",
    "Amount": 15798,
    "ProductId": 1,
    "DebitCard": {
      "ExpirationDate": "12/2020",
      "SecurityCodeStatus": "Nonexistent",
      "BrandId": 1,
      "IssuerId": 2,
      "InputMode": "MagStripe",
      "AuthenticationMethod": "OnlineAuthentication",
      "TrackOneData": "encryptedA1234567890123456^FULANO OLIVEIRA SA ^12345678901234567890123encrypted",
      "TrackTwoData": "encrypted0123456789012345=012345678901234encrypted",
      "EncryptedCardData": {
          "EncryptionType": "DUKPT3DES",
          "TrackOneDataKSN": "KSNforTrackOneData",
          "TrackTwoDataKSN": "KSNforTrackTwoData"
      },
      "PinBlock": {
        "EncryptedPinBlock": "2280F6BDFD0C038D",
        "EncryptionType": "Dukpt3Des",
        "KsnIdentification": "1231vg31fv231313123"
      },
      "PanSequenceNumber": 123,
      "SaveCard": false,
      "IsFallback": false
    },
    "PinPadInformation": {
      "TerminalId": "10000001",
      "SerialNumber": "ABC123",
      "PhysicalCharacteristics": "PinPadWithChipReaderWithSamModule",
      "ReturnDataInfo": "00"
    }
  }
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`MerchantOrderId`|String|---|---|Número do documento gerado automaticamente pelo terminal e incrementado de 1 a cada transação realizada no terminal. Aceita apenas valores numéricos de 1 a 15 dígitos.|
|`Payment.Type`|String|---|Sim|Value: PhysicalCreditCard / Tipo da Transação|
|`Payment.SoftDescriptor`|String|13|---|Identificação do estabelecimento (nome reduzido) a ser impresso e identificado na fatura.|
|`Payment.PaymentDateTime`|String|date-time|Sim|Data e Hora da captura da transação|
|`Payment.Amount`|Integer(int64)|---|Sim|Valor da transação (1079 = R$10,79)|
|`Payment.ProductId`|Integer|---|Sim|Código do produto identificado através do bin do cartão.|
|`DebitCard.ExpirationDate`|String|MM/yyyy|Sim|Data de validade do cartão.<br><br>Dado obtido através do comando PP_GetCard na BC no momento da captura da transação.|
|`DebitCard.SecurityCodeStatus`|String|---|---|Enum: Collected Unreadable Nonexistent<br><br>Status da coleta de código de segurança (CVV)|
|`DebitCard.BrandId`|Integer|---Sim|Identificação da bandeira obtida através do campo BrandId da PRODUCT TABLE.|
|`DebitCard.IssuerId`|Integer|---|Sim|Código do emissor obtido através do campo IssuerId da BIN TABLE.|
|`CreditCard.InputMode`|String|---|Sim|Enum: Typed MagStripe Emv<br><br>Identificação do modo de captura do cartão na transação. Essa informação deve ser obtida através do retorno da função<br><br> PP_GetCard da BC.<br><br>“00” – Magnético<br><br>“01” - Moedeiro VISA Cash sobre TIBC v1<br><br>“02” - Moedeiro VISA Cash sobre TIBC v3<br><br>“03” – EMV com contato<br><br>“04” - Easy-Entry sobre TIBC v1<br><br>“05” - Chip sem contato simulando tarja<br><br>“06” - EMV sem contato.<br><br>|
|`DebitCard.AuthenticationMethod`|String|---|Sim|Enum: `NoPassword` `OnlineAuthentication` `OfflineAuthentication` <br><br>Método de autenticação <br><br>- Se o cartão foi lido a partir da digitação verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, a senha deve ser capturada e o authenticationMethod assume valor 2. Caso contrário, assume valor 1; <br><br>- Se o cartão foi lido a partir da trilha verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, deve ser verificado o bit 2 do mesmo campo. Se este estiver com valor 1 deve ser capturada a senha. Se estiver com valor 0 a captura da senha vai depender do último dígito do service code; <br><br>- Se o cartão foi lido através do chip EMV, o authenticationMethod será preenchido com base no retorno da função PP_GoOnChip(). No resultado PP_GoOnChip(), onde se o campo da posição 003 do retorno da PP_GoOnChip() estiver com valor 1 indica que o pin foi validado off-line, o authenticationMethod será 3. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valor 0, o authenticationMethod será 1. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valores 0 e 1 respectivamente, o authenticationMethod será 2. <br><br>1 - Sem senha = “NoPassword”; <br><br>2 - Senha online = “Online Authentication”; <br><br>3 - Senha off-line = “Offline Authentication”.|
|`DebitCard.TrackOneData`|String|---|---|Dados da trilha 1 criptografados<br><br>Obtidos através do comando PP_GetCard na BC no momento da captura da transação|
|`DebitCard.TrackTwoData`|String|---|---|Dados da trilha 2 criptografados<br><br>Obtidos através do comando PP_GetCard na BC no momento da captura da transação|
|`CreditCard.EncryptedCardData.EncryptionType`|String|---|Sim|Tipo de encriptação utilizada<br><br>Enum:<br><br>"DukptDes" = 1,<br><br>"MasterKey" = 2 <br<br>"Dukpt3Des" = 3,<br><br>"Dukpt3DesCBC" = 4|
|`DebitCard.EncryptedCardData.TrackOneDataKSN`|String|---|---|Identificador KSN da criptografia da trilha 1 do cartão|
|`DebitCard.EncryptedCardData.TrackTwoDataKSN`|String|---|---|Identificador KSN da criptografia da trilha 2 do cartão|
|`DebitCard.EncryptedCardData.IsDataInTLVFormat`|Booleano|---|Não Identifica se os dados criptografados estão no formato TLV (tag / length / value).|
|`DebitCard.EncryptedCardData.InitializationVector`|String|---|Sim|Vetor de inicialização da encryptação|
|`PinBlock.EncryptedPinBlock`|String|---|Sim|PINBlock Criptografado<br><br>- Para transações EMV, esse campo é obtido através do retorno da função PP_GoOnChip(), mais especificamente das posições 007 até a posição 022;<br><br>- Para transações digitadas e com tarja magnética, verificar as posições 001 até 016 do retorno da função PP_GetPin().|
|`PinBlock.EncryptionType`|String|---|Sim|Tipo de Criptografia<br><br>Enum:<br><br>"DukptDes"<br><br>"Dukpt3Des"<br><br>"MasterKey"|
|`PinBlock.KsnIdentification`|String|---|Sim|Identificação do KSN<br><br>- Para transações EMV esse campo é obtido através do retorno da função PP_GoOnChip() nas posições 023 até 042;<br><br>- Para transações digitadas e com tarja magnética, verificar as posições 017 até 036 do retorno da função PP_GetPin().|
|`DebitCard.PanSequenceNumber`|Number|---|---|Número sequencial do cartão, utilizado para identificar a conta corrente do cartão adicional. Mandatório para transações com cartões Chip EMV e que possuam PAN Sequence Number (Tag 5F34).|
|`DebitCard.SaveCard`|Booleano|---|---|Identifica se vai salvar/tokenizar o cartão|
|`DebitCard.IsFallback`|Booleano|---|---|Identifica se é uma transação de fallback.|
|`PinPadInformation.TerminalId`|String|---|Sim|Número Lógico definido no Concentrador Cielo.|
|`PinPadInformation.SerialNumber`|String|---|Sim|Número de Série do Equipamento.|
|`PinPadInformation.PhysicalCharacteristics`|String|---|Sim|Enum: WithoutPinPad PinPadWithoutChipReader PinPadWithChipReaderWithoutSamModule<br><br> PinPadWithChipReaderWithSamModule NotCertifiedPinPad PinPadWithChipReaderWithoutSamAndContactless<br><br>PinPadWithChipReaderWithSamModuleAndContactless<br><br>SemPIN-pad = WithoutPinPad;PIN-pad sem leitor de Chip = PinpadWithoutChipReader;<br><br>PIN-pad com leitor de Chip sem módulo SAM = PinPadWithChipReaderWithoutSamModule;<br><br>PIN-pad com leitor de Chip com módulo SAM = PinPadWithChipReaderWithSamModule;<br><br>PIN-pad não homologado = NotCertifiedPinPad;<br><br>PIN-pad com leitor de Chip sem SAM e Cartão Sem Contato = PinpadWithChipReaderWithoutSamAndContactless;<br><br>PIN-pad com leitor de Chip com SAM e Cartão Sem Contato = PinpadWithChipReaderWithSamAndContactless.<br><br>Obs. Caso a aplicação não consiga informar os dados acima, deve obter tais informações através do retorno da função PP_GetInfo() da BC.|
|`PinPadInformation.ReturnDataInfo`|String|---|Sim|Retorno da função PP_GetInfo() da biblioteca compartilhada|

#### Resposta

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`MerchantOrderId`|String|---|---| Número do documento gerado automaticamente pelo terminal e incrementado de 1 a cada transação realizada no terminal. Aceita apenas valores numéricos de 1 a 15 dígitos.|
|`Customer.Name`|String|---|---|---|
|`Customer.Identity`|---|---|---|---|
|`Customer.IdentityType`|---|---|---|---|
|`Customer.Email`|---|---|---|---|
|`Customer.Birthday`|---|---|---|---|
|`Address.Street`|String|---|---|---|
|`Address.Number`|String|---|---|---|
|`Address.Complement`|String|---|---|---|
|`Address.ZipCode`|String|---|---|---|
|`Address.City`|String|---|---|---|
|`Address.State`|String|---|---|---|
|`Address.Country`|String|---|---|---|
|`DeliveryAddress.Street`|String|---|---|---|
|`DeliveryAddress.Number`|String|---|---|---|
|`DeliveryAddress.Complement`|String|---|---|---|
|`DeliveryAddress.ZipCode`|String|---|---|---|
|`DeliveryAddress.City`|String|---|---|---|
|`DeliveryAddress.State`|String|---|---|---|
|`DeliveryAddress.Country`|String|---|---|---|
|`Payment.Installments`|Integer|---|---|Default: 1 / Quantidade de Parcelas: Varia de 2 a 99 para transação de financiamento. Deve ser verificado os atributos maxOfPayments1, maxOfPayments2, maxOfPayments3 e minValOfPayments da tabela productTable.|
|`Payment.Interest`|String|---|---|Default: `ByMerchant` <br><br> Enum: `ByMerchant` `ByIssuer` <br><br> Tipo de Parcelamento: <br><br> - Se o bit 6 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 6 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento sem juros pode ser efetuado. <br><br> - Se o bit 7 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 7 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento com juros pode ser efetuado. Sem juros = “ByMerchant”; Com juros = “ByIssuer”.|
|`Payment.Capture`|Booleano|---|---|Default: false / Booleano que identifica que a autorização deve ser com captura automática. A autorização sem captura automática é conhecida também como pré-autorização.|
|`DebitCard.ExpirationDate`|String|MM/yyyy|Sim|Data de validade do cartão. <br><br>Dado obtido através do comando PP_GetCard na BC no momento da captura da transação.|
|`DebitCard.BrandId`|Integer|---|Sim|Identificação da bandeira obtida através do campo BrandId da PRODUCT TABLE.|
|`DebitCard.IssuerId`|Integer|---|Sim|Código do emissor obtido através do campo IssuerId da BIN TABLE.|
|`DebitCard.TruncateCardNumberWhenPrinting`|Booleano|---|---|Indica se o número do cartão será truncado no momento da impressão do comprovante. A solução de captura deve tomar essa decisão com base no `confParamOp03` presente nas tabelas BIN TABLE, PARAMETER TABLE e ISSUER TABLE.|
|`CreditCard.InputMode`|String|---|Sim|Enum: Typed MagStripe Emv<br><br>Identificação do modo de captura do cartão na transação. Essa informação deve ser obtida através do retorno da função PP_GetCard da BC.<br><br>“00” – Magnético<br><br>“01” - Moedeiro VISA Cash sobre TIBC v1<br><br>“02” - Moedeiro VISA Cash sobre TIBC v3<br><br>“03” – EMV com contato<br><br>“04” - Easy-Entry sobre TIBC v1<br><br>“05” - Chip sem contato simulando tarja<br><br>|“06” - EMV sem contato.|
|`CreditCard.AuthenticationMethod`|String|---|Sim|Enum: `NoPassword` `OnlineAuthentication` `OfflineAuthentication` <br><br>Método de autenticação <br><br>- Se o cartão foi lido a partir da digitação verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, a senha deve ser capturada e o authenticationMethod assume valor 2. Caso contrário, assume valor 1; <br><br>- Se o cartão foi lido a partir da trilha verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, deve ser verificado o bit 2 do mesmo campo. Se este estiver com valor 1 deve ser capturada a senha. Se estiver com valor 0 a captura da senha vai depender do último dígito do service code; <br><br>- Se o cartão foi lido através do chip EMV, o authenticationMethod será preenchido com base no retorno da função PP_GoOnChip(). No resultado PP_GoOnChip(), onde se o campo da posição 003 do retorno da PP_GoOnChip() estiver com valor 1 indica que o pin foi validado off-line, o authenticationMethod será 3. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valor 0, o authenticationMethod será 1. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valores 0 e 1 respectivamente, o authenticationMethod será 2. <br><br>1 - Sem senha = “NoPassword”; <br><br>2 - Senha online = “Online Authentication”; <br><br>3 - Senha off-line = “Offline Authentication”.|
|`CreditCard.EmvData`|String|---|---|Dados da transação EMV <br><br>Obtidos através do comando PP_GoOnChip na BC.|
|`PinBlock.EncryptedPinBlock`|---|---|---|---|
|`PinBlock.EncryptionType`|String|---|---|---|
|`PinBlock.KsnIdentification`|String|---|---|---|
|`CreditCard.PanSequenceNumber`|Number|---|---|Número sequencial do cartão, utilizado para identificar a conta corrente do cartão adicional. Mandatório para transações com cartões Chip EMV e que possuam PAN Sequence Number (Tag 5F34).|
|`CreditCard.SaveCard`|---|---|---|---|
|`CreditCard.IsFallback`|---|---|---|---|
|`Payment.PaymentDateTime`|String|date-time|Sim|Data e Hora da captura da transação|
|`Payment.ServiceTaxAmount`|---|---|---|---|
|`Payment.SoftDescriptor`|String|13|---|Identificação do estabelecimento (nome reduzido) a ser impresso e identificado na fatura.|
|`Payment.ProductId|Integer`|---|Sim|Código do produto identificado através do bin do cartão.|
|`PinPadInformation.TerminalId`|String|---|Sim|Número Lógico definido no Concentrador Cielo.|
|`PinPadInformation.PhysicalCharacteristics`|String|-|Sim|Enum: `WithoutPinPad` `PinPadWithoutChipReader` `PinPadWithChipReaderWithoutSamModule` `PinPadWithChipReaderWithSamModule` `NotCertifiedPinPad` `PinPadWithChipReaderWithoutSamAndContactless` `PinPadWithChipReaderWithSamModuleAndContactless` <br><br> Sem PIN-pad = `WithoutPinPad`; <br><br> PIN-pad sem leitor de Chip = `PinpadWithoutChipReader`; <br><br>PIN-pad com leitor de Chip sem módulo SAM = `PinPadWithChipReaderWithoutSamModule`; <br><br> PIN-pad com leitor de Chip com módulo SAM = `PinPadWithChipReaderWithSamModule`; <br><br> PIN-pad não homologado = `NotCertifiedPinPad`; <br><br> PIN-pad com leitor de Chip sem SAM e Cartão Sem Contato = `PinpadWithChipReaderWithoutSamAndContactless`; <br><br> PIN-pad com leitor de Chip com SAM e Cartão Sem Contato = `PinpadWithChipReaderWithSamAndContactless`. <br><br><br> Obs. Caso a aplicação não consiga informar os dados acima, deve obter tais informações através do retorno da função PP_GetInfo() da BC.|
|`PinPadInformation.SerialNumber`|String|---|Sim|Número de Série do Equipamento.|
|`PinPadInformation.ReturnDataInfo`|String|---|Sim|Retorno da função PP_GetInfo() da biblioteca compartilhada|
|`Payment.Amount|Integer`|(int64)|---|Sim|Valor da transação (1079 = R$10,79)|
|`Payment.ReceivedDate`|---|---|---|---|
|`Payment.CapturedAmount`|---|---|---|---|
|`Payment.Provider`|String|---|---|---|
|`Payment.ConfirmationStatus`|---|---|---|---|
|`Payment.InitializationVersion`|---|---|---|---|
|`Payment.EmvResponseData`|---|---|---|---|
|`Payment.Status`|---|---|---|---|
|`Payment.IsSplitted`|Booleano|---|---|---|
|`Payment.ReturnCode`|---|---|---|---|
|`Payment.ReturnMessage`|String|---|---|---|
|`Payment.PaymentId`|---|---|---|---|
|`Payment.Type`|String|---|Sim|Value: PhysicalCreditCard / Tipo da Transação|
|`Payment.Currency`|String|---|---|Default: “BRL” / Value: “BRL” / Moeda (Preencher com “BRL”)|
|`Payment.Country`|String|---|---|Default: “BRA” / Value: “BRA” / País (Preencher com “BRA”)|
|`Receipt.MerchantName`|---|---|---|---|
|`Receipt.MerchantAddress`|---|---|---|---|
|`Receipt.MerchantCity`|---|---|---|---|
|`Receipt.MerchantState`|---|---|---|---|
|`Receipt.MerchantCode`|---|---|---|---|
|`Receipt.Terminal`|---|---|---|---|
|`Receipt.Nsu`|---|---|---|---|
|`Receipt.Date`|---|---|---|---|
|`Receipt.Hour`|---|---|---|---|
|`Receipt.IssuerName`|---|---|---|---|
|`Receipt.CardNumber`|---|---|---|---|
|`Receipt.TransactionType`|---|---|---|---|
|`Receipt.AuthorizationCode`|---|---|---|---|
|`Receipt.TransactionMode`|---|---|---|---|
|`Receipt.InputMethod`|---|---|---|---|
|`Receipt.Value`|---|---|---|---|
|`Receipt.SoftDescriptor`|---|---|---|---|
|`RecurrentPayment.RecurrentPaymentId`|---|---|---|---|
|`RecurrentPayment.ReasonCode`|---|---|---|---|
|`RecurrentPayment.ReasonMessage`|---|---|---|---|
|`RecurrentPayment.NextRecurrency`|---|---|---|---|
|`RecurrentPayment.EndDate`|---|---|---|---|
|`RecurrentPayment.Interval`|---|---|---|---|
|`SplitPayments.SubordinateMerchantId`|---|---|---|---|
|`SplitPayments.Amount`|---|---|---|---|
|`SplitPayments.Fares.Mdr`|---|---|---|---|
|`SplitPayments.Fares.Fee`|---|---|---|---|
|`SplitErrors.Code`|---|---|---|---|
|`SplitErrors.Message`|---|---|---|---|

### Crédito por chip com senha online

#### Requisição

<aside class="request"><span class="method post">POST</span> <span class="endpoint">/1/physicalSales/</span></aside>

```json
{
  "MerchantOrderId": "201904150004",
  "Payment": {
    "Type": "PhysicalCreditCard",
    "SoftDescriptor": "Description",
    "PaymentDateTime": "2019-04-15T12:00:00Z",
    "Amount": 15798,
    "Installments": 1,
    "Interest": "ByMerchant",
    "Capture": true,
    "ProductId": 1,
    "CreditCard": {
      "ExpirationDate": "12/2020",
      "BrandId": 1,
      "IssuerId": 2,
      "InputMode": "Emv",
      "AuthenticationMethod": "OnlinePassword",
      "EmvData": "112233445566778899011AABBC012D3456789E0123FF45678AB901234C5D112233445566778800",
      "PinBlock": {
        "EncryptedPinBlock": "2280F6BDFD0C038D",
        "EncryptionType": "Dukpt3Des",
        "KsnIdentification": "1231vg31fv231313123"
      },
      "PanSequenceNumber": 123,
      "SaveCard": false,
      "IsFallback": false
    },
    "PinPadInformation": {
      "TerminalId": "10000001",
      "SerialNumber": "ABC123",
      "PhysicalCharacteristics": "PinPadWithChipReaderWithSamModule",
      "ReturnDataInfo": "00"
    }
  }
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`MerchantOrderId`|String|---|---| Número do documento gerado automaticamente pelo terminal e incrementado de 1 a cada transação realizada no terminal. Aceita apenas valores numéricos de 1 a 15 dígitos.|
|`Payment.Type`|String|---|Sim|Value: `PhysicalCreditCard` / Tipo da Transação|
|`Payment.SoftDescriptor`|String|13|---|Identificação do estabelecimento (nome reduzido) a ser impresso e identificado na fatura.|
|`Payment.PaymentDateTime`|String|date-time|Sim|Data e Hora da captura da transação|
|`Payment.Amount`|Integer(int64)|---|Sim|Valor da transação (1079 = R$10,79)|
|`Payment.Installments`|Integer|---|---|Default: 1 / Quantidade de Parcelas: Varia de 2 a 99 para transação de financiamento. Deve ser verificado os atributos maxOfPayments1, maxOfPayments2, maxOfPayments3 e minValOfPayments da tabela productTable.|
|`Payment.Interest`|String|---|---|Default: `ByMerchant` <br><br> Enum: `ByMerchant` `ByIssuer` <br><br> Tipo de Parcelamento: <br><br> - Se o bit 6 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 6 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento sem juros pode ser efetuado. <br><br> - Se o bit 7 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 7 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento com juros pode ser efetuado. Sem juros = “ByMerchant”; Com juros = “ByIssuer”.|
|`Payment.Capture`|Booleano|---|---|Default: false / Booleano que identifica que a autorização deve ser com captura automática. A autorização sem captura automática é conhecida também como pré-autorização.|
|`Payment.ProductId`|Integer|---|Sim|Código do produto identificado através do bin do cartão.|
|`CreditCard.ExpirationDate`|String|MM/yyyy|Sim|Data de validade do cartão. <br><br>Dado obtido através do comando PP_GetCard na BC no momento da captura da transação.|
|`CreditCard.BrandId`|Integer|---|Sim|Identificação da bandeira obtida através do campo BrandId da PRODUCT TABLE.|
|`CreditCard.IssuerId`|Integer|---|Sim|Código do emissor obtido através do campo IssuerId da BIN TABLE.|
|`CreditCard.InputMode`|String|---|Sim|Enum: `Typed` `MagStripe` `Emv` <br><br>Identificação do modo de captura do cartão na transação. Essa informação deve ser obtida através do retorno da função PP_GetCard da BC. <br><br>"00" – Magnético <br><br>"01" - Moedeiro VISA Cash sobre TIBC v1 <br><br>"02" - Moedeiro VISA Cash sobre TIBC v3 <br><br>"03" – EMV com contato <br><br>"04" - Easy-Entry sobre TIBC v1 <br><br>"05" - Chip sem contato simulando tarja <br><br> "06" - EMV sem contato.|
|`CreditCard.AuthenticationMethod`|String|---|Sim|Enum: `NoPassword` `OnlineAuthentication` `OfflineAuthentication` <br><br>Método de autenticação <br><br>- Se o cartão foi lido a partir da digitação verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, a senha deve ser capturada e o authenticationMethod assume valor 2. Caso contrário, assume valor 1; <br><br>- Se o cartão foi lido a partir da trilha verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, deve ser verificado o bit 2 do mesmo campo. Se este estiver com valor 1 deve ser capturada a senha. Se estiver com valor 0 a captura da senha vai depender do último dígito do service code; <br><br>- Se o cartão foi lido através do chip EMV, o authenticationMethod será preenchido com base no retorno da função PP_GoOnChip(). No resultado PP_GoOnChip(), onde se o campo da posição 003 do retorno da PP_GoOnChip() estiver com valor 1 indica que o pin foi validado off-line, o authenticationMethod será 3. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valor 0, o authenticationMethod será 1. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valores 0 e 1 respectivamente, o authenticationMethod será 2. <br><br>1 - Sem senha = “NoPassword”; <br><br>2 - Senha online = “Online Authentication”; <br><br>3 - Senha off-line = “Offline Authentication”.|
|`CreditCard.EmvData`|String|---|---|Dados da transação EMV <br><br>Obtidos através do comando PP_GoOnChip na BC|
|`PinBlock.EncryptedPinBlock`|String|---|Sim|PINBlock Criptografado<br><br>- Para transações EMV, esse campo é obtido através do retorno da função PP_GoOnChip(), mais especificamente das posições 007 até a posição 022;<br><br>- Para transações digitadas e com tarja magnética, verificar as posições 001 até 016 do retorno da função PP_GetPin().|
|`PinBlock.EncryptionType`|String|---|Sim|Tipo de Criptografia<br><br>Enum:<br><br>"DukptDes"<br><br>"Dukpt3Des"<br><br>"MasterKey"|
|`PinBlock.KsnIdentification`|String|---|Sim|Identificação do KSN<br><br>- Para transações EMV esse campo é obtido através do retorno da função PP_GoOnChip() nas posições 023 até 042;<br><br>- Para transações digitadas e com tarja magnética, verificar as posições 017 até 036 do retorno da função PP_GetPin().|
|`CreditCard.PanSequenceNumber`|Number|---|---|Número sequencial do cartão, utilizado para identificar a conta corrente do cartão adicional. Mandatório para transações com cartões Chip EMV e que possuam PAN Sequence Number (Tag 5F34).|
|`CreditCard.SaveCard`|Booleano|---|---|Identifica se vai salvar/tokenizar o cartão.|
|`CreditCard.IsFallback`|Booleano|---|---|Identifica se é uma transação de fallback.|
|`PinPadInformation.TerminalId`|String|---|Sim|Número Lógico definido no Concentrador Cielo.|
|`PinPadInformation.SerialNumber`|String|---|Sim|Número de Série do Equipamento.|
|`PinPadInformation.PhysicalCharacteristics`|String|---|Sim|Enum: `WithoutPinPad` `PinPadWithoutChipReader` `PinPadWithChipReaderWithoutSamModule` `PinPadWithChipReaderWithSamModule` `NotCertifiedPinPad` `PinPadWithChipReaderWithoutSamAndContactless` `PinPadWithChipReaderWithSamModuleAndContactless` <br><br> Sem PIN-pad = `WithoutPinPad`; <br><br> PIN-pad sem leitor de Chip = `PinpadWithoutChipReader`; <br><br>PIN-pad com leitor de Chip sem módulo SAM = `PinPadWithChipReaderWithoutSamModule`; <br><br> PIN-pad com leitor de Chip com módulo SAM = `PinPadWithChipReaderWithSamModule`; <br><br> PIN-pad não homologado = `NotCertifiedPinPad`; <br><br> PIN-pad com leitor de Chip sem SAM e Cartão Sem Contato = `PinpadWithChipReaderWithoutSamAndContactless`; <br><br> PIN-pad com leitor de Chip com SAM e Cartão Sem Contato = `PinpadWithChipReaderWithSamAndContactless`. <br><br><br> Obs. Caso a aplicação não consiga informar os dados acima, deve obter tais informações através do retorno da função PP_GetInfo() da BC.|
|`PinPadInformation.ReturnDataInfo`|String|---|Sim|Retorno da função PP_GetInfo() da biblioteca compartilhada|

#### Resposta

```json
{
  "MerchantOrderId": "20180204",
  "Customer": {
    "Name": "Comprador crédito completo",
    "Identity": "11225468954",
    "IdentityType": "CPF",
    "Email": "compradorteste@teste.com",
    "Birthday": "1991-01-02",
    "Address": {
      "Street": "Rua Teste",
      "Number": "123",
      "Complement": "AP 123",
      "ZipCode": "12345987",
      "City": "São Paulo",
      "State": "SP",
      "Country": "BRA"
    },
    "DeliveryAddress": {
      "Street": "Rua Teste",
      "Number": "123",
      "Complement": "AP 123",
      "ZipCode": "12345987",
      "City": "São Paulo",
      "State": "SP",
      "Country": "BRA"
    }
  },
  "Payment": {
    "Installments": 1,
    "Interest": "ByMerchant",
    "Capture": true,
    "CreditCard": {
      "ExpirationDate": "12/2020",
      "BrandId": 1,
      "IssuerId": 2,
      "TruncateCardNumberWhenPrinting": true,
      "InputMode": "Emv",
      "AuthenticationMethod": "OnlineAuthentication",
      "EmvData": "112233445566778899011AABBC012D3456789E0123FF45678AB901234C5D112233445566778800",
      "PinBlock": {
        "EncryptedPinBlock": "2280F6BDFD0C038D",
        "EncryptionType": "Dukpt3Des",
        "KsnIdentification": "1231vg31fv231313123"
      },
      "PanSequenceNumber": 123,
      "SaveCard": false,
      "IsFallback": false
    },
    "PaymentDateTime": "2019-04-15T12:00:00Z",
    "ServiceTaxAmount": 0,
    "SoftDescriptor": "Description",
    "ProductId": 1,
    "PinPadInformation": {
      "TerminalId": "10000001",
      "SerialNumber": "ABC123",
      "PhysicalCharacteristics": "PinPadWithChipReaderWithSamModule",
      "ReturnDataInfo": "00"
    },
    "Amount": 15798,
    "ReceivedDate": "2019-04-15T12:00:00Z",
    "CapturedAmount": 15798,
    "Provider": "Cielo",
    "ConfirmationStatus": 0,
    "InitializationVersion": 1558708320029,
    "EmvResponseData": "123456789ABCD1345DEA",
    "Status": 2,
    "IsSplitted": false,
    "ReturnCode": 0,
    "ReturnMessage": "Successful",
    "PaymentId": "f15889ea-5719-4e1a-a2da-f4e50d5bd702",
    "Type": "PhysicalDebitCard",
    "Currency": "BRL",
    "Country": "BRA",
    "Links": [
      {
        "Method": "GET",
        "Rel": "self",
        "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702"
      },
      {
        "Method": "DELETE",
        "Rel": "self",
        "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702"
      },
      {
        "Method": "PUT",
        "Rel": "self",
        "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702/confirmation"
      }
    ],
    "PrintMessage": [
      {
        "Position": "Top",
        "Message": "Transação autorizada"
      },
      {
        "Position": "Middle",
        "Message": "Informação adicional"
      },
      {
        "Position": "Bottom",
        "Message": "Obrigado e volte sempre!"
      }
    ],
    "ReceiptInformation": [
      {
        "Field": "MERCHANT_NAME",
        "Label": "NOME DO ESTABELECIMENTO",
        "Content": "Estabelecimento"
      },
      {
        "Field": "MERCHANT_ADDRESS",
        "Label": "ENDEREÇO DO ESTABELECIMENTO",
        "Content": "Rua Sem Saida, 0"
      },
      {
        "Field": "MERCHANT_CITY",
        "Label": "CIDADE DO ESTABELECIMENTO",
        "Content": "Cidade"
      },
      {
        "Field": "MERCHANT_STATE",
        "Label": "ESTADO DO ESTABELECIMENTO",
        "Content": "WA"
      },
      {
        "Field": "MERCHANT_CODE",
        "Label": "COD.ESTAB.",
        "Content": 1234567890123456
      },
      {
        "Field": "TERMINAL",
        "Label": "POS",
        "Content": 12345678
      },
      {
        "Field": "NSU",
        "Label": "DOC",
        "Content": 123456
      },
      {
        "Field": "DATE",
        "Label": "DATA",
        "Content": "01/01/20"
      },
      {
        "Field": "HOUR",
        "Label": "HORA",
        "Content": "01:01"
      },
      {
        "Field": "ISSUER_NAME",
        "Label": "EMISSOR",
        "Content": "NOME DO EMISSOR"
      },
      {
        "Field": "CARD_NUMBER",
        "Label": "CARTÃO",
        "Content": 5432123454321234
      },
      {
        "Field": "TRANSACTION_TYPE",
        "Label": "TIPO DE TRANSAÇÃO",
        "Content": "VENDA A CREDITO"
      },
      {
        "Field": "AUTHORIZATION_CODE",
        "Label": "AUTORIZAÇÃO",
        "Content": 123456
      },
      {
        "Field": "TRANSACTION_MODE",
        "Label": "MODO DA TRANSAÇÃO",
        "Content": "ONL"
      },
      {
        "Field": "INPUT_METHOD",
        "Label": "MODO DE ENTRADA",
        "Content": "X"
      },
      {
        "Field": "VALUE",
        "Label": "VALOR",
        "Content": "1,23"
      },
      {
        "Field": "SOFT_DESCRIPTOR",
        "Label": "SOFT DESCRIPTOR",
        "Content": "Simulado"
      }
    ],
    "Receipt": {
      "MerchantName": "Estabelecimento",
      "MerchantAddress": "Rua Sem Saida, 0",
      "MerchantCity": "Cidade",
      "MerchantState": "WA",
      "MerchantCode": 1234567890123456,
      "Terminal": 12345678,
      "Nsu": 123456,
      "Date": "01/01/20",
      "Hour": "01:01",
      "IssuerName": "NOME DO EMISSOR",
      "CardNumber": 5432123454321234,
      "TransactionType": "VENDA A CREDITO",
      "AuthorizationCode": 123456,
      "TransactionMode": "ONL",
      "InputMethod": "X",
      "Value": "1,23",
      "SoftDescriptor": "Simulado"
    },
    "RecurrentPayment": {
      "RecurrentPaymentId": "a6b719fa-a8df-ab11-4e1a-f4e50d5bd702",
      "ReasonCode": 0,
      "ReasonMessage": "Successful",
      "NextRecurrency": "2019-12-01",
      "EndDate": "2019-12-01",
      "Interval": 6
    },
    "SplitPayments": [
      {
        "SubordinateMerchantId": "491daf20-35f2-4379-874c-e7552ae8dc10",
        "Amount": 100,
        "Fares": {
          "Mdr": 5,
          "Fee": 0
        }
      },
      {
        "SubordinateMerchantId": "7e2846be-4e80-4f86-8ca9-eb35db6aea00",
        "Amount": 80,
        "Fares": {
          "Mdr": 3,
          "Fee": 1
        }
      }
    ],
    "SplitErrors": [
      {
        "Code": 326,
        "Message": "SubordinatePayment amount must be greater than zero"
      }
    ]
  }
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`MerchantOrderId`|String|---|---| Número do documento gerado automaticamente pelo terminal e incrementado de 1 a cada transação realizada no terminal. Aceita apenas valores numéricos de 1 a 15 dígitos.|
|`Customer.Name`|String|---|---|---|
|`Customer.Identity`|---|---|---|---|
|`Customer.IdentityType`|---|---|---|---|
|`Customer.Email`|---|---|---|---|
|`Customer.Birthday`|---|---|---|---|
|`Address.Street`|String|---|---|---|
|`Address.Number`|String|---|---|---|
|`Address.Complement`|String|---|---|---|
|`Address.ZipCode`|String|---|---|---|
|`Address.City`|String|---|---|---|
|`Address.State`|String|---|---|---|
|`Address.Country`|String|---|---|---|
|`DeliveryAddress.Street`|String|---|---|---|
|`DeliveryAddress.Number`|String|---|---|---|
|`DeliveryAddress.Complement`|String|---|---|---|
|`DeliveryAddress.ZipCode`|String|---|---|---|
|`DeliveryAddress.City`|String|---|---|---|
|`DeliveryAddress.State`|String|---|---|---|
|`DeliveryAddress.Country`|String|---|---|---|
|`Payment.Installments`|Integer|---|---|Default: 1 / Quantidade de Parcelas: Varia de 2 a 99 para transação de financiamento. Deve ser verificado os atributos maxOfPayments1, maxOfPayments2, maxOfPayments3 e minValOfPayments da tabela productTable.|
|`Payment.Interest`|String|---|---|Default: `ByMerchant` <br><br> Enum: `ByMerchant` `ByIssuer` <br><br> Tipo de Parcelamento: <br><br> - Se o bit 6 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 6 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento sem juros pode ser efetuado. <br><br> - Se o bit 7 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 7 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento com juros pode ser efetuado. Sem juros = “ByMerchant”; Com juros = “ByIssuer”.|
|`Payment.Capture`|Booleano|---|---|Default: false / Booleano que identifica que a autorização deve ser com captura automática. A autorização sem captura automática é conhecida também como pré-autorização.|
|`CreditCard.ExpirationDate`|String|MM/yyyy|Sim|Data de validade do cartão. <br><br>Dado obtido através do comando PP_GetCard na BC no momento da captura da transação.|
|`CreditCard.BrandId`|Integer|---|Sim|Identificação da bandeira obtida através do campo BrandId da PRODUCT TABLE.|
|`CreditCard.IssuerId`|Integer|---|Sim|Código do emissor obtido através do campo IssuerId da BIN TABLE.|
|`CreditCard.TruncateCardNumberWhenPrinting`|Booleano|---|---|Indica se o número do cartão será truncado no momento da impressão do comprovante. A solução de captura deve tomar essa decisão com base no `confParamOp03` presente nas tabelas BIN TABLE, PARAMETER TABLE e ISSUER TABLE.|
|`CreditCard.InputMode`|String|---|Sim|Enum: `Typed` `MagStripe` `Emv` <br><br>Identificação do modo de captura do cartão na transação. Essa informação deve ser obtida através do retorno da função PP_GetCard da BC. <br><br>"00" – Magnético <br><br>"01" - Moedeiro VISA Cash sobre TIBC v1 <br><br>"02" - Moedeiro VISA Cash sobre TIBC v3 <br><br>"03" – EMV com contato <br><br>"04" - Easy-Entry sobre TIBC v1 <br><br>"05" - Chip sem contato simulando tarja <br><br> "06" - EMV sem contato.|
|`CreditCard.AuthenticationMethod`|String|---|Sim|Enum: `NoPassword` `OnlineAuthentication` `OfflineAuthentication` <br><br>Método de autenticação <br><br>- Se o cartão foi lido a partir da digitação verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, a senha deve ser capturada e o authenticationMethod assume valor 2. Caso contrário, assume valor 1; <br><br>- Se o cartão foi lido a partir da trilha verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, deve ser verificado o bit 2 do mesmo campo. Se este estiver com valor 1 deve ser capturada a senha. Se estiver com valor 0 a captura da senha vai depender do último dígito do service code; <br><br>- Se o cartão foi lido através do chip EMV, o authenticationMethod será preenchido com base no retorno da função PP_GoOnChip(). No resultado PP_GoOnChip(), onde se o campo da posição 003 do retorno da PP_GoOnChip() estiver com valor 1 indica que o pin foi validado off-line, o authenticationMethod será 3. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valor 0, o authenticationMethod será 1. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valores 0 e 1 respectivamente, o authenticationMethod será 2. <br><br>1 - Sem senha = “NoPassword”; <br><br>2 - Senha online = “Online Authentication”; <br><br>3 - Senha off-line = “Offline Authentication”.|
|`CreditCard.EmvData`|String|---|---|Dados da transação EMV <br><br>Obtidos através do comando PP_GoOnChip na BC|
|`PinBlock.EncryptedPinBlock`|String|---|Sim|PINBlock Criptografado<br><br>- Para transações EMV, esse campo é obtido através do retorno da função PP_GoOnChip(), mais especificamente das posições 007 até a posição 022;<br><br>- Para transações digitadas e com tarja magnética, verificar as posições 001 até 016 do retorno da função PP_GetPin().|
|`PinBlock.EncryptionType`|String|---|Sim|Tipo de Criptografia<br><br>Enum:<br><br>"DukptDes"<br><br>"Dukpt3Des"<br><br>"MasterKey"|
|`PinBlock.KsnIdentification`|String|---|Sim|Identificação do KSN<br><br>- Para transações EMV esse campo é obtido através do retorno da função PP_GoOnChip() nas posições 023 até 042;<br><br>- Para transações digitadas e com tarja magnética, verificar as posições 017 até 036 do retorno da função PP_GetPin().
|`CreditCard.PanSequenceNumber`|Number|---|---|Número sequencial do cartão, utilizado para identificar a conta corrente do cartão adicional. Mandatório para transações com cartões Chip EMV e que possuam PAN Sequence Number (Tag 5F34).|
|`CreditCard.SaveCard`|Booleano|---|---|Identifica se vai salvar/tokenizar o cartão.|
|`CreditCard.IsFallback`|Booleano|---|---|Identifica se é uma transação de fallback.|
|`Payment.PaymentDateTime`|String|date-time|Sim|Data e Hora da captura da transação|
|`Payment.ServiceTaxAmount`|---|---|---|---|
|`Payment.SoftDescriptor`|String|13|---|Identificação do estabelecimento (nome reduzido) a ser impresso e identificado na fatura.|
|`Payment.ProductId`|Integer|---|Sim|Código do produto identificado através do bin do cartão.|
|`PinPadInformation.TerminalId`|String|---|Sim|Número Lógico definido no Concentrador Cielo.|
|`PinPadInformation.SerialNumber`|String|---|Sim|Número de Série do Equipamento.|
|`PinPadInformation.PhysicalCharacteristics`|String|---|Sim|Enum: `WithoutPinPad` `PinPadWithoutChipReader` `PinPadWithChipReaderWithoutSamModule` `PinPadWithChipReaderWithSamModule` `NotCertifiedPinPad` `PinPadWithChipReaderWithoutSamAndContactless` `PinPadWithChipReaderWithSamModuleAndContactless` <br><br> Sem PIN-pad = `WithoutPinPad`; <br><br> PIN-pad sem leitor de Chip = `PinpadWithoutChipReader`; <br><br>PIN-pad com leitor de Chip sem módulo SAM = `PinPadWithChipReaderWithoutSamModule`; <br><br> PIN-pad com leitor de Chip com módulo SAM = `PinPadWithChipReaderWithSamModule`; <br><br> PIN-pad não homologado = `NotCertifiedPinPad`; <br><br> PIN-pad com leitor de Chip sem SAM e Cartão Sem Contato = `PinpadWithChipReaderWithoutSamAndContactless`; <br><br> PIN-pad com leitor de Chip com SAM e Cartão Sem Contato = `PinpadWithChipReaderWithSamAndContactless`. <br><br><br> Obs. Caso a aplicação não consiga informar os dados acima, deve obter tais informações através do retorno da função PP_GetInfo() da BC.|
|`PinPadInformation.ReturnDataInfo`|String|---|Sim|Retorno da função PP_GetInfo() da biblioteca compartilhada|
|`Payment.Amount`|Integer(int64)|---|Sim|Valor da transação (1079 = R$10,79)|
|`Payment.ReceivedDate`|---|---|---|---|
|`Payment.CapturedAmount`|---|---|---|---|
|`Payment.Provider`|String|---|---|---|
|`Payment.ConfirmationStatus`|---|---|---|---|
|`Payment.InitializationVersion`|---|---|---|---|
|`Payment.EmvResponseData`|---|---|---|---|
|`Payment.Status`|---|---|---|---|
|`Payment.IsSplitted`|Booleano|---|---|---|
|`Payment.ReturnCode`|---|---|---|---|
|`Payment.ReturnMessage`|String|---|---|---|
|`Payment.PaymentId`|---|---|---|---|
|`Payment.Type`|String|---|Sim|Value: `PhysicalCreditCard` / Tipo da Transação|
|`Payment.Currency`|String|---|---|Default: "BRL" / Value: "BRL" / Moeda (Preencher com “BRL”)|
|`Payment.Country`|String|---|---|Default: "BRA" / Value: "BRA" / País (Preencher com “BRA”)|
|`Receipt.MerchantName`|---|---|---|---|
|`Receipt.MerchantAddress`|---|---|---|---|
|`Receipt.MerchantCity`|---|---|---|---|
|`Receipt.MerchantState`|---|---|---|---|
|`Receipt.MerchantCode`|---|---|---|---|
|`Receipt.Terminal`|---|---|---|---|
|`Receipt.Nsu`|---|---|---|---|
|`Receipt.Date`|---|---|---|---|
|`Receipt.Hour`|---|---|---|---|
|`Receipt.IssuerName`|---|---|---|---|
|`Receipt.CardNumber`|---|---|---|---|
|`Receipt.TransactionType`|---|---|---|---|
|`Receipt.AuthorizationCode`|---|---|---|---|
|`Receipt.TransactionMode`|---|---|---|---|
|`Receipt.InputMethod`|---|---|---|---|
|`Receipt.Value`|---|---|---|---|
|`Receipt.SoftDescriptor`|---|---|---|---|
|`RecurrentPayment.RecurrentPaymentId`|---|---|---|---|
|`RecurrentPayment.ReasonCode`|---|---|---|---|
|`RecurrentPayment.ReasonMessage`|---|---|---|---|
|`RecurrentPayment.NextRecurrency`|---|---|---|---|
|`RecurrentPayment.EndDate`|---|---|---|---|
|`RecurrentPayment.Interval`|---|---|---|---|
|`SplitPayments.SubordinateMerchantId`|---|---|---|---|
|`SplitPayments.Amount`|---|---|---|---|
|`SplitPayments.Fares.Mdr`|---|---|---|---|
|`SplitPayments.Fares.Fee`|---|---|---|---|
|`SplitErrors.Code`|---|---|---|---|
|`SplitErrors.Message`|---|---|---|---|

### Crédito por chip com cartão criptografado

#### Requisição

```json
{
  "MerchantOrderId": "1596226820548",
  "Payment": {
    "Type": "PhysicalCreditCard",
    "SoftDescriptor": "Description",
    "PaymentDateTime": "2019-04-15T12:00:00Z",
    "Amount": 15798,
    "Installments": 1,
    "Interest": "ByMerchant",
    "Capture": true,
    "ProductId": 1,
    "CreditCard": {
      "ExpirationDate": "12/2020",
      "BrandId": 1,
      "IssuerId": 2,
      "InputMode": "Emv",
      "AuthenticationMethod": "OnlineAuthentication",
      "TrackTwoData": "0123456789012345=012345678901234",
      "EncryptedCardData": {
          "EncryptionType": "DUKPT3DES",
          "TrackTwoDataKSN": "KSNforTrackTwoData"
      },
      "EmvData": "112233445566778899011AABBC012D3456789E0123FF45678AB901234C5D112233445566778800",
      "PinBlock": {
        "EncryptedPinBlock": "2280F6BDFD0C038D",
        "EncryptionType": "Dukpt3Des",
        "KsnIdentification": "1231vg31fv231313123"
      },
      "PanSequenceNumber": 123,
      "SaveCard": false,
      "IsFallback": false
    },
    "PinPadInformation": {
      "TerminalId": "10000001",
      "SerialNumber": "ABC123",
      "PhysicalCharacteristics": "PinPadWithChipReaderWithSamModule",
      "ReturnDataInfo": "00"
    }
  }
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`MerchantOrderId`|String|---|---| Número do documento gerado automaticamente pelo terminal e incrementado de 1 a cada transação realizada no terminal. Aceita apenas valores numéricos de 1 a 15 dígitos.|
|`Payment.Type`|String|---|Sim|Value: `PhysicalCreditCard` / Tipo da Transação|
|`Payment.SoftDescriptor`|String|13|---|Identificação do estabelecimento (nome reduzido) a ser impresso e identificado na fatura.|
|`Payment.PaymentDateTime`|String|date-time|Sim|Data e Hora da captura da transação|
|`Payment.Amount`|Integer(int64)|---|Sim|Valor da transação (1079 = R$10,79)|
|`Payment.Installments`|Integer|---|---|Default: 1 / Quantidade de Parcelas: Varia de 2 a 99 para transação de financiamento. Deve ser verificado os atributos maxOfPayments1, maxOfPayments2, maxOfPayments3 e minValOfPayments da tabela productTable.|
|`Payment.Interest`|String|---|---|Default: `ByMerchant` <br><br> Enum: `ByMerchant` `ByIssuer` <br><br> Tipo de Parcelamento: <br><br> - Se o bit 6 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 6 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento sem juros pode ser efetuado. <br><br> - Se o bit 7 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 7 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento com juros pode ser efetuado. Sem juros = “ByMerchant”; Com juros = “ByIssuer”.|
|`Payment.Capture`|Booleano|---|---|Default: false / Booleano que identifica que a autorização deve ser com captura automática. A autorização sem captura automática é conhecida também como pré-autorização.|
|`Payment.ProductId`|Integer|---|Sim|Código do produto identificado através do bin do cartão.|
|`CreditCard.ExpirationDate`|String|MM/yyyy|Sim|Data de validade do cartão. <br><br>Dado obtido através do comando PP_GetCard na BC no momento da captura da transação.|
|`CreditCard.BrandId`|Integer|---|Sim|Identificação da bandeira obtida através do campo BrandId da PRODUCT TABLE.|
|`CreditCard.IssuerId`|Integer|---|Sim|Código do emissor obtido através do campo IssuerId da BIN TABLE.|
|`CreditCard.InputMode`|String|---|Sim|Enum: `Typed` `MagStripe` `Emv` <br><br>Identificação do modo de captura do cartão na transação. Essa informação deve ser obtida através do retorno da função PP_GetCard da BC. <br><br>"00" – Magnético <br><br>"01" - Moedeiro VISA Cash sobre TIBC v1 <br><br>"02" - Moedeiro VISA Cash sobre TIBC v3 <br><br>"03" – EMV com contato <br><br>"04" - Easy-Entry sobre TIBC v1 <br><br>"05" - Chip sem contato simulando tarja <br><br> "06" - EMV sem contato.|
|`CreditCard.AuthenticationMethod`|String|---|Sim|Enum: `NoPassword` `OnlineAuthentication` `OfflineAuthentication` <br><br>Método de autenticação <br><br>- Se o cartão foi lido a partir da digitação verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, a senha deve ser capturada e o authenticationMethod assume valor 2. Caso contrário, assume valor 1; <br><br>- Se o cartão foi lido a partir da trilha verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, deve ser verificado o bit 2 do mesmo campo. Se este estiver com valor 1 deve ser capturada a senha. Se estiver com valor 0 a captura da senha vai depender do último dígito do service code; <br><br>- Se o cartão foi lido através do chip EMV, o authenticationMethod será preenchido com base no retorno da função PP_GoOnChip(). No resultado PP_GoOnChip(), onde se o campo da posição 003 do retorno da PP_GoOnChip() estiver com valor 1 indica que o pin foi validado off-line, o authenticationMethod será 3. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valor 0, o authenticationMethod será 1. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valores 0 e 1 respectivamente, o authenticationMethod será 2. <br><br>1 - Sem senha = “NoPassword”; <br><br>2 - Senha online = “Online Authentication”; <br><br>3 - Senha off-line = “Offline Authentication”.|
|`CreditCard.EmvData`|String|---|---|Dados da transação EMV <br><br>Obtidos através do comando PP_GoOnChip na BC|
|`PinBlock.EncryptedPinBlock`|String|---|Sim|PINBlock Criptografado<br><br>- Para transações EMV, esse campo é obtido através do retorno da função PP_GoOnChip(), mais especificamente das posições 007 até a posição 022;<br><br>- Para transações digitadas e com tarja magnética, verificar as posições 001 até 016 do retorno da função PP_GetPin().|
|`PinBlock.EncryptionType`|String|---|Sim|Tipo de Criptografia<br><br>Enum:<br><br>"DukptDes"<br><br>"Dukpt3Des"<br><br>"MasterKey"|
|`PinBlock.KsnIdentification`|String|---|Sim|Identificação do KSN<br><br>- Para transações EMV esse campo é obtido através do retorno da função PP_GoOnChip() nas posições 023 até 042;<br><br>- Para transações digitadas e com tarja magnética, verificar as posições 017 até 036 do retorno da função PP_GetPin().|
|`CreditCard.PanSequenceNumber`|Number|---|---|Número sequencial do cartão, utilizado para identificar a conta corrente do cartão adicional. Mandatório para transações com cartões Chip EMV e que possuam PAN Sequence Number (Tag 5F34).|
|`CreditCard.SaveCard`|Booleano|---|---|Identifica se vai salvar/tokenizar o cartão.|
|`CreditCard.IsFallback`|Booleano|---|---|Identifica se é uma transação de fallback.|
|`PinPadInformation.TerminalId`|String|---|Sim|Número Lógico definido no Concentrador Cielo.|
|`PinPadInformation.SerialNumber`|String|---|Sim|Número de Série do Equipamento.|
|`PinPadInformation.PhysicalCharacteristics`|String|---|Sim|Enum: `WithoutPinPad` `PinPadWithoutChipReader` `PinPadWithChipReaderWithoutSamModule` `PinPadWithChipReaderWithSamModule` `NotCertifiedPinPad` `PinPadWithChipReaderWithoutSamAndContactless` `PinPadWithChipReaderWithSamModuleAndContactless` <br><br> Sem PIN-pad = `WithoutPinPad`; <br><br> PIN-pad sem leitor de Chip = `PinpadWithoutChipReader`; <br><br>PIN-pad com leitor de Chip sem módulo SAM = `PinPadWithChipReaderWithoutSamModule`; <br><br> PIN-pad com leitor de Chip com módulo SAM = `PinPadWithChipReaderWithSamModule`; <br><br> PIN-pad não homologado = `NotCertifiedPinPad`; <br><br> PIN-pad com leitor de Chip sem SAM e Cartão Sem Contato = `PinpadWithChipReaderWithoutSamAndContactless`; <br><br> PIN-pad com leitor de Chip com SAM e Cartão Sem Contato = `PinpadWithChipReaderWithSamAndContactless`. <br><br><br> Obs. Caso a aplicação não consiga informar os dados acima, deve obter tais informações através do retorno da função PP_GetInfo() da BC.|
|`PinPadInformation.ReturnDataInfo`|String|---|Sim|Retorno da função PP_GetInfo() da biblioteca compartilhada|

#### Resposta

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`MerchantOrderId`|String|---|---| Número do documento gerado automaticamente pelo terminal e incrementado de 1 a cada transação realizada no terminal. Aceita apenas valores numéricos de 1 a 15 dígitos.|
|`Customer.Name`|String|---|---|---|
|`Customer.Identity`|---|---|---|---|
|`Customer.IdentityType`|---|---|---|---|
|`Customer.Email`|---|---|---|---|
|`Customer.Birthday`|---|---|---|---|
|`Address.Street`|String|---|---|---|
|`Address.Number`|String|---|---|---|
|`Address.Complement`|String|---|---|---|
|`Address.ZipCode`|String|---|---|---|
|`Address.City`|String|---|---|---|
|`Address.State`|String|---|---|---|
|`Address.Country`|String|---|---|---|
|`DeliveryAddress.Street`|String|---|---|---|
|`DeliveryAddress.Number`|String|---|---|---|
|`DeliveryAddress.Complement`|String|---|---|---|
|`DeliveryAddress.ZipCode`|String|---|---|---|
|`DeliveryAddress.City`|String|---|---|---|
|`DeliveryAddress.State`|String|---|---|---|
|`DeliveryAddress.Country`|String|---|---|---|
|`Payment.Installments`|Integer|---|---|Default: 1 / Quantidade de Parcelas: Varia de 2 a 99 para transação de financiamento. Deve ser verificado os atributos maxOfPayments1, maxOfPayments2, maxOfPayments3 e minValOfPayments da tabela productTable.|
|`Payment.Interest`|String|---|---|Default: `ByMerchant` <br><br> Enum: `ByMerchant` `ByIssuer` <br><br> Tipo de Parcelamento: <br><br> - Se o bit 6 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 6 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento sem juros pode ser efetuado. <br><br> - Se o bit 7 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 7 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento com juros pode ser efetuado. Sem juros = “ByMerchant”; Com juros = “ByIssuer”.|
|`Payment.Capture`|Booleano|---|---|Default: false / Booleano que identifica que a autorização deve ser com captura automática. A autorização sem captura automática é conhecida também como pré-autorização.|
|`CreditCard.ExpirationDate`|String|MM/yyyy|Sim|Data de validade do cartão. <br><br>Dado obtido através do comando PP_GetCard na BC no momento da captura da transação.|
|`CreditCard.BrandId`|Integer|---|Sim|Identificação da bandeira obtida através do campo BrandId da PRODUCT TABLE.|
|`CreditCard.IssuerId`|Integer|---|Sim|Código do emissor obtido através do campo IssuerId da BIN TABLE.|
|`CreditCard.TruncateCardNumberWhenPrinting`|Booleano|---|---|Indica se o número do cartão será truncado no momento da impressão do comprovante. A solução de captura deve tomar essa decisão com base no `confParamOp03` presente nas tabelas BIN TABLE, PARAMETER TABLE e ISSUER TABLE.|
|`CreditCard.InputMode`|String|---|Sim|Enum: `Typed` `MagStripe` `Emv` <br><br>Identificação do modo de captura do cartão na transação. Essa informação deve ser obtida através do retorno da função PP_GetCard da BC. <br><br>"00" – Magnético <br><br>"01" - Moedeiro VISA Cash sobre TIBC v1 <br><br>"02" - Moedeiro VISA Cash sobre TIBC v3 <br><br>"03" – EMV com contato <br><br>"04" - Easy-Entry sobre TIBC v1 <br><br>"05" - Chip sem contato simulando tarja <br><br> "06" - EMV sem contato.|
|`CreditCard.AuthenticationMethod`|String|---|Sim|Enum: `NoPassword` `OnlineAuthentication` `OfflineAuthentication` <br><br>Método de autenticação <br><br>- Se o cartão foi lido a partir da digitação verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, a senha deve ser capturada e o authenticationMethod assume valor 2. Caso contrário, assume valor 1; <br><br>- Se o cartão foi lido a partir da trilha verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, deve ser verificado o bit 2 do mesmo campo. Se este estiver com valor 1 deve ser capturada a senha. Se estiver com valor 0 a captura da senha vai depender do último dígito do service code; <br><br>- Se o cartão foi lido através do chip EMV, o authenticationMethod será preenchido com base no retorno da função PP_GoOnChip(). No resultado PP_GoOnChip(), onde se o campo da posição 003 do retorno da PP_GoOnChip() estiver com valor 1 indica que o pin foi validado off-line, o authenticationMethod será 3. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valor 0, o authenticationMethod será 1. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valores 0 e 1 respectivamente, o authenticationMethod será 2. <br><br>1 - Sem senha = “NoPassword”; <br><br>2 - Senha online = “Online Authentication”; <br><br>3 - Senha off-line = “Offline Authentication”.|
|`CreditCard.EmvData`|String|---|---|Dados da transação EMV <br><br>Obtidos através do comando PP_GoOnChip na BC|
|`PinBlock.EncryptedPinBlock`|String|---|Sim|PINBlock Criptografado<br><br>- Para transações EMV, esse campo é obtido através do retorno da função PP_GoOnChip(), mais especificamente das posições 007 até a posição 022;<br><br>- Para transações digitadas e com tarja magnética, verificar as posições 001 até 016 do retorno da função PP_GetPin().|
|`PinBlock.EncryptionType`|String|---|Sim|Tipo de Criptografia<br><br>Enum:<br><br>"DukptDes"<br><br>"Dukpt3Des"<br><br>"MasterKey"|
|`PinBlock.KsnIdentification`|String|---|Sim|Identificação do KSN<br><br>- Para transações EMV esse campo é obtido através do retorno da função PP_GoOnChip() nas posições 023 até 042;<br><br>- Para transações digitadas e com tarja magnética, verificar as posições 017 até 036 do retorno da função PP_GetPin().
|`CreditCard.PanSequenceNumber`|Number|---|---|Número sequencial do cartão, utilizado para identificar a conta corrente do cartão adicional. Mandatório para transações com cartões Chip EMV e que possuam PAN Sequence Number (Tag 5F34).|
|`CreditCard.SaveCard`|Booleano|---|---|Identifica se vai salvar/tokenizar o cartão.|
|`CreditCard.IsFallback`|Booleano|---|---|Identifica se é uma transação de fallback.|
|`Payment.PaymentDateTime`|String|date-time|Sim|Data e Hora da captura da transação|
|`Payment.ServiceTaxAmount`|---|---|---|---|
|`Payment.SoftDescriptor`|String|13|---|Identificação do estabelecimento (nome reduzido) a ser impresso e identificado na fatura.|
|`Payment.ProductId`|Integer|---|Sim|Código do produto identificado através do bin do cartão.|
|`PinPadInformation.TerminalId`|String|---|Sim|Número Lógico definido no Concentrador Cielo.|
|`PinPadInformation.SerialNumber`|String|---|Sim|Número de Série do Equipamento.|
|`PinPadInformation.PhysicalCharacteristics`|String|---|Sim|Enum: `WithoutPinPad` `PinPadWithoutChipReader` `PinPadWithChipReaderWithoutSamModule` `PinPadWithChipReaderWithSamModule` `NotCertifiedPinPad` `PinPadWithChipReaderWithoutSamAndContactless` `PinPadWithChipReaderWithSamModuleAndContactless` <br><br> Sem PIN-pad = `WithoutPinPad`; <br><br> PIN-pad sem leitor de Chip = `PinpadWithoutChipReader`; <br><br>PIN-pad com leitor de Chip sem módulo SAM = `PinPadWithChipReaderWithoutSamModule`; <br><br> PIN-pad com leitor de Chip com módulo SAM = `PinPadWithChipReaderWithSamModule`; <br><br> PIN-pad não homologado = `NotCertifiedPinPad`; <br><br> PIN-pad com leitor de Chip sem SAM e Cartão Sem Contato = `PinpadWithChipReaderWithoutSamAndContactless`; <br><br> PIN-pad com leitor de Chip com SAM e Cartão Sem Contato = `PinpadWithChipReaderWithSamAndContactless`. <br><br><br> Obs. Caso a aplicação não consiga informar os dados acima, deve obter tais informações através do retorno da função PP_GetInfo() da BC.|
|`PinPadInformation.ReturnDataInfo`|String|---|Sim|Retorno da função PP_GetInfo() da biblioteca compartilhada|
|`Payment.Amount`|Integer(int64)|---|Sim|Valor da transação (1079 = R$10,79)|
|`Payment.ReceivedDate`|---|---|---|---|
|`Payment.CapturedAmount`|---|---|---|---|
|`Payment.Provider`|String|---|---|---|
|`Payment.ConfirmationStatus`|---|---|---|---|
|`Payment.InitializationVersion`|---|---|---|---|
|`Payment.EmvResponseData`|---|---|---|---|
|`Payment.Status`|---|---|---|---|
|`Payment.IsSplitted`|Booleano|---|---|---|
|`Payment.ReturnCode`|---|---|---|---|
|`Payment.ReturnMessage`|String|---|---|---|
|`Payment.PaymentId`|---|---|---|---|
|`Payment.Type`|String|---|Sim|Value: `PhysicalCreditCard` / Tipo da Transação|
|`Payment.Currency`|String|---|---|Default: "BRL" / Value: "BRL" / Moeda (Preencher com “BRL”)|
|`Payment.Country`|String|---|---|Default: "BRA" / Value: "BRA" / País (Preencher com “BRA”)|
|`Receipt.MerchantName`|---|---|---|---|
|`Receipt.MerchantAddress`|---|---|---|---|
|`Receipt.MerchantCity`|---|---|---|---|
|`Receipt.MerchantState`|---|---|---|---|
|`Receipt.MerchantCode`|---|---|---|---|
|`Receipt.Terminal`|---|---|---|---|
|`Receipt.Nsu`|---|---|---|---|
|`Receipt.Date`|---|---|---|---|
|`Receipt.Hour`|---|---|---|---|
|`Receipt.IssuerName`|---|---|---|---|
|`Receipt.CardNumber`|---|---|---|---|
|`Receipt.TransactionType`|---|---|---|---|
|`Receipt.AuthorizationCode`|---|---|---|---|
|`Receipt.TransactionMode`|---|---|---|---|
|`Receipt.InputMethod`|---|---|---|---|
|`Receipt.Value`|---|---|---|---|
|`Receipt.SoftDescriptor`|---|---|---|---|
|`RecurrentPayment.RecurrentPaymentId`|---|---|---|---|
|`RecurrentPayment.ReasonCode`|---|---|---|---|
|`RecurrentPayment.ReasonMessage`|---|---|---|---|
|`RecurrentPayment.NextRecurrency`|---|---|---|---|
|`RecurrentPayment.EndDate`|---|---|---|---|
|`RecurrentPayment.Interval`|---|---|---|---|
|`SplitPayments.SubordinateMerchantId`|---|---|---|---|
|`SplitPayments.Amount`|---|---|---|---|
|`SplitPayments.Fares.Mdr`|---|---|---|---|
|`SplitPayments.Fares.Fee`|---|---|---|---|
|`SplitErrors.Code`|---|---|---|---|
|`SplitErrors.Message`|---|---|---|---|

### Débito por chip e senha online

#### Requisição

<aside class="request"><span class="method post">POST</span> <span class="endpoint">/1/physicalSales/</span></aside>

```json
{
  "MerchantOrderId": "201904150005",
  "Payment": {
    "Type": "PhysicalDebitCard",
    "SoftDescriptor": "Description",
    "PaymentDateTime": "2019-04-15T12:00:00Z",
    "Amount": 15798,
    "ProductId": 1,
    "DebitCard": {
      "ExpirationDate": "12/2020",
      "BrandId": 1,
      "IssuerId": 2,
      "InputMode": "Emv",
      "AuthenticationMethod": "OnlinePassword",
      "EmvData": "112233445566778899011AABBC012D3456789E0123FF45678AB901234C5D112233445566778800",
      "PinBlock": {
        "EncryptedPinBlock": "2280F6BDFD0C038D",
        "EncryptionType": "Dukpt3Des",
        "KsnIdentification": "1231vg31fv231313123"
      },
      "PanSequenceNumber": 123,
      "SaveCard": false,
      "IsFallback": false
    },
    "PinPadInformation": {
      "TerminalId": "10000001",
      "SerialNumber": "ABC123",
      "PhysicalCharacteristics": "PinPadWithChipReaderWithSamModule",
      "ReturnDataInfo": "00"
    }
  }
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`MerchantOrderId`|String|---|---| Número do documento gerado automaticamente pelo terminal e incrementado de 1 a cada transação realizada no terminal. Aceita apenas valores numéricos de 1 a 15 dígitos.|
|`Payment.Type`|String|---|Sim|Value: `PhysicalCreditCard` / Tipo da Transação|
|`Payment.SoftDescriptor`|String|13|---|Identificação do estabelecimento (nome reduzido) a ser impresso e identificado na fatura.|
|`Payment.PaymentDateTime`|String|date-time|Sim|Data e Hora da captura da transação|
|`Payment.Amount`|Integer(int64)|---|Sim|Valor da transação (1079 = R$10,79)|
|`Payment.ProductId`|Integer|---|Sim|Código do produto identificado através do bin do cartão.|
|`DebitCard.ExpirationDate`|String|MM/yyyy|Sim|Data de validade do cartão. <br><br>Dado obtido através do comando PP_GetCard na BC no momento da captura da transação.|
|`DebitCard.BrandId`|Integer|---|Sim|Identificação da bandeira obtida através do campo BrandId da PRODUCT TABLE.|
|`DebitCard.IssuerId`|Integer|---|Sim|Código do emissor obtido através do campo IssuerId da BIN TABLE.|
|`DebitCard.InputMode`|String|---|Sim|Enum: `Typed` `MagStripe` `Emv` <br><br>Identificação do modo de captura do cartão na transação. Essa informação deve ser obtida através do retorno da função PP_GetCard da BC. <br><br>"00" – Magnético <br><br>"01" - Moedeiro VISA Cash sobre TIBC v1 <br><br>"02" - Moedeiro VISA Cash sobre TIBC v3 <br><br>"03" – EMV com contato <br><br>"04" - Easy-Entry sobre TIBC v1 <br><br>"05" - Chip sem contato simulando tarja <br><br> "06" - EMV sem contato.|
|`DebitCard.AuthenticationMethod`|String|---|Sim|Enum: `NoPassword` `OnlineAuthentication` `OfflineAuthentication` <br><br>Método de autenticação <br><br>- Se o cartão foi lido a partir da digitação verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, a senha deve ser capturada e o authenticationMethod assume valor 2. Caso contrário, assume valor 1; <br><br>- Se o cartão foi lido a partir da trilha verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, deve ser verificado o bit 2 do mesmo campo. Se este estiver com valor 1 deve ser capturada a senha. Se estiver com valor 0 a captura da senha vai depender do último dígito do service code; <br><br>- Se o cartão foi lido através do chip EMV, o authenticationMethod será preenchido com base no retorno da função PP_GoOnChip(). No resultado PP_GoOnChip(), onde se o campo da posição 003 do retorno da PP_GoOnChip() estiver com valor 1 indica que o pin foi validado off-line, o authenticationMethod será 3. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valor 0, o authenticationMethod será 1. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valores 0 e 1 respectivamente, o authenticationMethod será 2. <br><br>1 - Sem senha = “NoPassword”; <br><br>2 - Senha online = “Online Authentication”; <br><br>3 - Senha off-line = “Offline Authentication”.|
|`DebitCard.EmvData`|String|---|---|Dados da transação EMV <br><br>Obtidos através do comando PP_GoOnChip na BC|
|`PinBlock.EncryptedPinBlock`|String|---|Sim|PINBlock Criptografado<br><br>- Para transações EMV, esse campo é obtido através do retorno da função PP_GoOnChip(), mais especificamente das posições 007 até a posição 022;<br><br>- Para transações digitadas e com tarja magnética, verificar as posições 001 até 016 do retorno da função PP_GetPin().|
|`PinBlock.EncryptionType`|String|---|Sim|Tipo de Criptografia<br><br>Enum:<br><br>"DukptDes"<br><br>"Dukpt3Des"<br><br>"MasterKey"|
|`PinBlock.KsnIdentification`|String|---|Sim|Identificação do KSN<br><br>- Para transações EMV esse campo é obtido através do retorno da função PP_GoOnChip() nas posições 023 até 042;<br><br>- Para transações digitadas e com tarja magnética, verificar as posições 017 até 036 do retorno da função PP_GetPin().|
|`DebitCard.PanSequenceNumber`|Number|---|---|Número sequencial do cartão, utilizado para identificar a conta corrente do cartão adicional. Mandatório para transações com cartões Chip EMV e que possuam PAN Sequence Number (Tag 5F34).|
|`DebitrCard.SaveCard`|Booleano|---|---|Identifica se vai salvar/tokenizar o cartão.|
|`DebitCard.IsFallback`|Booleano|---|---|Identifica se é uma transação de fallback.|
|`PinPadInformation.TerminalId`|String|---|Sim|Número Lógico definido no Concentrador Cielo.|
|`PinPadInformation.SerialNumber`|String|---|Sim|Número de Série do Equipamento.|
|`PinPadInformation.PhysicalCharacteristics`|String|---|Sim|Enum: `WithoutPinPad` `PinPadWithoutChipReader` `PinPadWithChipReaderWithoutSamModule` `PinPadWithChipReaderWithSamModule` `NotCertifiedPinPad` `PinPadWithChipReaderWithoutSamAndContactless` `PinPadWithChipReaderWithSamModuleAndContactless` <br><br> Sem PIN-pad = `WithoutPinPad`; <br><br> PIN-pad sem leitor de Chip = `PinpadWithoutChipReader`; <br><br>PIN-pad com leitor de Chip sem módulo SAM = `PinPadWithChipReaderWithoutSamModule`; <br><br> PIN-pad com leitor de Chip com módulo SAM = `PinPadWithChipReaderWithSamModule`; <br><br> PIN-pad não homologado = `NotCertifiedPinPad`; <br><br> PIN-pad com leitor de Chip sem SAM e Cartão Sem Contato = `PinpadWithChipReaderWithoutSamAndContactless`; <br><br> PIN-pad com leitor de Chip com SAM e Cartão Sem Contato = `PinpadWithChipReaderWithSamAndContactless`. <br><br><br> Obs. Caso a aplicação não consiga informar os dados acima, deve obter tais informações através do retorno da função PP_GetInfo() da BC.|
|`PinPadInformation.ReturnDataInfo`|String|---|Sim|Retorno da função PP_GetInfo() da biblioteca compartilhada|

#### Resposta

```json
{
  "MerchantOrderId": "20180204",
  "Customer": {
    "Name": "Comprador crédito completo",
    "Identity": "11225468954",
    "IdentityType": "CPF",
    "Email": "compradorteste@teste.com",
    "Birthday": "1991-01-02",
    "Address": {
      "Street": "Rua Teste",
      "Number": "123",
      "Complement": "AP 123",
      "ZipCode": "12345987",
      "City": "São Paulo",
      "State": "SP",
      "Country": "BRA"
    },
    "DeliveryAddress": {
      "Street": "Rua Teste",
      "Number": "123",
      "Complement": "AP 123",
      "ZipCode": "12345987",
      "City": "São Paulo",
      "State": "SP",
      "Country": "BRA"
    }
  },
  "Payment": {
    "Installments": 1,
    "Interest": "ByMerchant",
    "Capture": true,
    "CreditCard": {
      "ExpirationDate": "12/2020",
      "BrandId": 1,
      "IssuerId": 2,
      "TruncateCardNumberWhenPrinting": true,
      "InputMode": "Emv",
      "AuthenticationMethod": "OnlineAuthentication",
      "EmvData": "112233445566778899011AABBC012D3456789E0123FF45678AB901234C5D112233445566778800",
      "PinBlock": {
        "EncryptedPinBlock": "2280F6BDFD0C038D",
        "EncryptionType": "Dukpt3Des",
        "KsnIdentification": "1231vg31fv231313123"
      },
      "PanSequenceNumber": 123,
      "SaveCard": false,
      "IsFallback": false
    },
    "PaymentDateTime": "2019-04-15T12:00:00Z",
    "ServiceTaxAmount": 0,
    "SoftDescriptor": "Description",
    "ProductId": 1,
    "PinPadInformation": {
      "TerminalId": "10000001",
      "SerialNumber": "ABC123",
      "PhysicalCharacteristics": "PinPadWithChipReaderWithSamModule",
      "ReturnDataInfo": "00"
    },
    "Amount": 15798,
    "ReceivedDate": "2019-04-15T12:00:00Z",
    "CapturedAmount": 15798,
    "Provider": "Cielo",
    "ConfirmationStatus": 0,
    "InitializationVersion": 1558708320029,
    "EmvResponseData": "123456789ABCD1345DEA",
    "Status": 2,
    "IsSplitted": false,
    "ReturnCode": 0,
    "ReturnMessage": "Successful",
    "PaymentId": "f15889ea-5719-4e1a-a2da-f4e50d5bd702",
    "Type": "PhysicalDebitCard",
    "Currency": "BRL",
    "Country": "BRA",
    "Links": [
      {
        "Method": "GET",
        "Rel": "self",
        "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702"
      },
      {
        "Method": "DELETE",
        "Rel": "self",
        "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702"
      },
      {
        "Method": "PUT",
        "Rel": "self",
        "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702/confirmation"
      }
    ],
    "PrintMessage": [
      {
        "Position": "Top",
        "Message": "Transação autorizada"
      },
      {
        "Position": "Middle",
        "Message": "Informação adicional"
      },
      {
        "Position": "Bottom",
        "Message": "Obrigado e volte sempre!"
      }
    ],
    "ReceiptInformation": [
      {
        "Field": "MERCHANT_NAME",
        "Label": "NOME DO ESTABELECIMENTO",
        "Content": "Estabelecimento"
      },
      {
        "Field": "MERCHANT_ADDRESS",
        "Label": "ENDEREÇO DO ESTABELECIMENTO",
        "Content": "Rua Sem Saida, 0"
      },
      {
        "Field": "MERCHANT_CITY",
        "Label": "CIDADE DO ESTABELECIMENTO",
        "Content": "Cidade"
      },
      {
        "Field": "MERCHANT_STATE",
        "Label": "ESTADO DO ESTABELECIMENTO",
        "Content": "WA"
      },
      {
        "Field": "MERCHANT_CODE",
        "Label": "COD.ESTAB.",
        "Content": 1234567890123456
      },
      {
        "Field": "TERMINAL",
        "Label": "POS",
        "Content": 12345678
      },
      {
        "Field": "NSU",
        "Label": "DOC",
        "Content": 123456
      },
      {
        "Field": "DATE",
        "Label": "DATA",
        "Content": "01/01/20"
      },
      {
        "Field": "HOUR",
        "Label": "HORA",
        "Content": "01:01"
      },
      {
        "Field": "ISSUER_NAME",
        "Label": "EMISSOR",
        "Content": "NOME DO EMISSOR"
      },
      {
        "Field": "CARD_NUMBER",
        "Label": "CARTÃO",
        "Content": 5432123454321234
      },
      {
        "Field": "TRANSACTION_TYPE",
        "Label": "TIPO DE TRANSAÇÃO",
        "Content": "VENDA A CREDITO"
      },
      {
        "Field": "AUTHORIZATION_CODE",
        "Label": "AUTORIZAÇÃO",
        "Content": 123456
      },
      {
        "Field": "TRANSACTION_MODE",
        "Label": "MODO DA TRANSAÇÃO",
        "Content": "ONL"
      },
      {
        "Field": "INPUT_METHOD",
        "Label": "MODO DE ENTRADA",
        "Content": "X"
      },
      {
        "Field": "VALUE",
        "Label": "VALOR",
        "Content": "1,23"
      },
      {
        "Field": "SOFT_DESCRIPTOR",
        "Label": "SOFT DESCRIPTOR",
        "Content": "Simulado"
      }
    ],
    "Receipt": {
      "MerchantName": "Estabelecimento",
      "MerchantAddress": "Rua Sem Saida, 0",
      "MerchantCity": "Cidade",
      "MerchantState": "WA",
      "MerchantCode": 1234567890123456,
      "Terminal": 12345678,
      "Nsu": 123456,
      "Date": "01/01/20",
      "Hour": "01:01",
      "IssuerName": "NOME DO EMISSOR",
      "CardNumber": 5432123454321234,
      "TransactionType": "VENDA A CREDITO",
      "AuthorizationCode": 123456,
      "TransactionMode": "ONL",
      "InputMethod": "X",
      "Value": "1,23",
      "SoftDescriptor": "Simulado"
    },
    "RecurrentPayment": {
      "RecurrentPaymentId": "a6b719fa-a8df-ab11-4e1a-f4e50d5bd702",
      "ReasonCode": 0,
      "ReasonMessage": "Successful",
      "NextRecurrency": "2019-12-01",
      "EndDate": "2019-12-01",
      "Interval": 6
    },
    "SplitPayments": [
      {
        "SubordinateMerchantId": "491daf20-35f2-4379-874c-e7552ae8dc10",
        "Amount": 100,
        "Fares": {
          "Mdr": 5,
          "Fee": 0
        }
      },
      {
        "SubordinateMerchantId": "7e2846be-4e80-4f86-8ca9-eb35db6aea00",
        "Amount": 80,
        "Fares": {
          "Mdr": 3,
          "Fee": 1
        }
      }
    ],
    "SplitErrors": [
      {
        "Code": 326,
        "Message": "SubordinatePayment amount must be greater than zero"
      }
    ]
  }
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`MerchantOrderId`|String|---|---| Número do documento gerado automaticamente pelo terminal e incrementado de 1 a cada transação realizada no terminal. Aceita apenas valores numéricos de 1 a 15 dígitos.|
|`Customer.Name`|String|---|---|---|
|`Customer.Identity`|---|---|---|---|
|`Customer.IdentityType`|---|---|---|---|
|`Customer.Email`|---|---|---|---|
|`Customer.Birthday`|---|---|---|---|
|`Address.Street`|String|---|---|---|
|`Address.Number`|String|---|---|---|
|`Address.Complement`|String|---|---|---|
|`Address.ZipCode`|String|---|---|---|
|`Address.City`|String|---|---|---|
|`Address.State`|String|---|---|---|
|`Address.Country`|String|---|---|---|
|`DeliveryAddress.Street`|String|---|---|---|
|`DeliveryAddress.Number`|String|---|---|---|
|`DeliveryAddress.Complement`|String|---|---||---|
|`DeliveryAddress.ZipCode`|String|---|---||---|
|`DeliveryAddress.City`|String|---|---|---|
|`DeliveryAddress.State`|String|---|---|---|
|`DeliveryAddress.Country`|String|---|---|---|
|`Payment.Installments`|Integer|---|---|Default: 1 / Quantidade de Parcelas: Varia de 2 a 99 para transação de financiamento. Deve ser verificado os atributos maxOfPayments1, maxOfPayments2, maxOfPayments3 e minValOfPayments da tabela productTable.|
|`Payment.Interest`|String|---|---|Default: `ByMerchant` <br><br> Enum: `ByMerchant` `ByIssuer` <br><br> Tipo de Parcelamento: <br><br> - Se o bit 6 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 6 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento sem juros pode ser efetuado. <br><br> - Se o bit 7 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 7 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento com juros pode ser efetuado. Sem juros = “ByMerchant”; Com juros = “ByIssuer”.|
|`Payment.Capture`|Booleano|---|---|Default: false / Booleano que identifica que a autorização deve ser com captura automática. A autorização sem captura automática é conhecida também como pré-autorização.|
|`CreditCard.ExpirationDate`|String|MM/yyyy|Sim|Data de validade do cartão. <br><br>Dado obtido através do comando PP_GetCard na BC no momento da captura da transação.|
|`CreditCard.BrandId`|Integer|---|Sim|Identificação da bandeira obtida através do campo BrandId da PRODUCT TABLE.|
|`CreditCard.IssuerId`|Integer|---|Sim|Código do emissor obtido através do campo IssuerId da BIN TABLE.|
|`CreditCard.TruncateCardNumberWhenPrinting`|Booleano|---|---|Indica se o número do cartão será truncado no momento da impressão do comprovante. A solução de captura deve tomar essa decisão com base no `confParamOp03` presente nas tabelas BIN TABLE, PARAMETER TABLE e ISSUER TABLE.|
|`CreditCard.InputMode`|String|---|Sim|Enum: `Typed` `MagStripe` `Emv` <br><br>Identificação do modo de captura do cartão na transação. Essa informação deve ser obtida através do retorno da função PP_GetCard da BC. <br><br>"00" – Magnético <br><br>"01" - Moedeiro VISA Cash sobre TIBC v1 <br><br>"02" - Moedeiro VISA Cash sobre TIBC v3 <br><br>"03" – EMV com contato <br><br>"04" - Easy-Entry sobre TIBC v1 <br><br>"05" - Chip sem contato simulando tarja <br><br> "06" - EMV sem contato.|
|`CreditCard.AuthenticationMethod`|String|---|Sim|Enum: `NoPassword` `OnlineAuthentication` `OfflineAuthentication` <br><br>Método de autenticação <br><br>- Se o cartão foi lido a partir da digitação verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, a senha deve ser capturada e o authenticationMethod assume valor 2. Caso contrário, assume valor 1; <br><br>- Se o cartão foi lido a partir da trilha verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, deve ser verificado o bit 2 do mesmo campo. Se este estiver com valor 1 deve ser capturada a senha. Se estiver com valor 0 a captura da senha vai depender do último dígito do service code; <br><br>- Se o cartão foi lido através do chip EMV, o authenticationMethod será preenchido com base no retorno da função PP_GoOnChip(). No resultado PP_GoOnChip(), onde se o campo da posição 003 do retorno da PP_GoOnChip() estiver com valor 1 indica que o pin foi validado off-line, o authenticationMethod será 3. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valor 0, o authenticationMethod será 1. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valores 0 e 1 respectivamente, o authenticationMethod será 2. <br><br>1 - Sem senha = “NoPassword”; <br><br>2 - Senha online = “Online Authentication”; <br><br>3 - Senha off-line = “Offline Authentication”.|
|`CreditCard.EmvData`|String|---|---|Dados da transação EMV <br><br>Obtidos através do comando PP_GoOnChip na BC|
|`PinBlock.EncryptedPinBlock`|String|---|Sim|PINBlock Criptografado<br><br>- Para transações EMV, esse campo é obtido através do retorno da função PP_GoOnChip(), mais especificamente das posições 007 até a posição 022;<br><br>- Para transações digitadas e com tarja magnética, verificar as posições 001 até 016 do retorno da função PP_GetPin().|
|`PinBlock.EncryptionType`|String|---|Sim|Tipo de Criptografia<br><br>Enum:<br><br>"DukptDes"<br><br>"Dukpt3Des"<br><br>"MasterKey"|
|`PinBlock.KsnIdentification`|String|---|Sim|Identificação do KSN<br><br>- Para transações EMV esse campo é obtido através do retorno da função PP_GoOnChip() nas posições 023 até 042;<br><br>- Para transações digitadas e com tarja magnética, verificar as posições 017 até 036 do retorno da função PP_GetPin().|
|`CreditCard.PanSequenceNumber`|Number|---|---|Número sequencial do cartão, utilizado para identificar a conta corrente do cartão adicional. Mandatório para transações com cartões Chip EMV e que possuam PAN Sequence Number (Tag 5F34).|
|`DebitCard.SaveCard`|Booleano|---|---|Identifica se vai salvar/tokenizar o cartão.|
|`DebitCard.IsFallback`|Booleano|---|---|Identifica se é uma transação de fallback.|
|`Payment.PaymentDateTime`|String|date-time|Sim|Data e Hora da captura da transação|
|`Payment.ServiceTaxAmount`|---|---|---|---|
|`Payment.SoftDescriptor`|String|13|---|Identificação do estabelecimento (nome reduzido) a ser impresso e identificado na fatura.|
|`Payment.ProductId`|Integer|---|Sim|Código do produto identificado através do bin do cartão.|
|`PinPadInformation.TerminalId`|String|---|Sim|Número Lógico definido no Concentrador Cielo.|
|`PinPadInformation.SerialNumber`|String|---|Sim|Número de Série do Equipamento.|
|`PinPadInformation.PhysicalCharacteristics`|String|---|Sim|Enum: `WithoutPinPad` `PinPadWithoutChipReader` `PinPadWithChipReaderWithoutSamModule` `PinPadWithChipReaderWithSamModule` `NotCertifiedPinPad` `PinPadWithChipReaderWithoutSamAndContactless` `PinPadWithChipReaderWithSamModuleAndContactless` <br><br> Sem PIN-pad = `WithoutPinPad`; <br><br> PIN-pad sem leitor de Chip = `PinpadWithoutChipReader`; <br><br>PIN-pad com leitor de Chip sem módulo SAM = `PinPadWithChipReaderWithoutSamModule`; <br><br> PIN-pad com leitor de Chip com módulo SAM = `PinPadWithChipReaderWithSamModule`; <br><br> PIN-pad não homologado = `NotCertifiedPinPad`; <br><br> PIN-pad com leitor de Chip sem SAM e Cartão Sem Contato = `PinpadWithChipReaderWithoutSamAndContactless`; <br><br> PIN-pad com leitor de Chip com SAM e Cartão Sem Contato = `PinpadWithChipReaderWithSamAndContactless`. <br><br><br> Obs. Caso a aplicação não consiga informar os dados acima, deve obter tais informações através do retorno da função PP_GetInfo() da BC.|
|`PinPadInformation.ReturnDataInfo`|String|---|Sim|Retorno da função PP_GetInfo() da biblioteca compartilhada|
|`Payment.Amount`|Integer(int64)|---|Sim|Valor da transação (1079 = R$10,79)|
|`Payment.ReceivedDate`|---|---|---|---|
|`Payment.CapturedAmount`|---|---|---|---|
|`Payment.Provider`|String|---|---|---|
|`Payment.ConfirmationStatus`|---|---|---|---|
|`Payment.InitializationVersion`|---|---|---|---|
|`Payment.EmvResponseData`|---|---|---|---|
|`Payment.Status`|---|---|---|---|
|`Payment.IsSplitted`|Booleano|---|---|---|
|`Payment.ReturnCode`|---|---|---|---|
|`Payment.ReturnMessage`|String|---|---|---|
|`Payment.PaymentId`|---|---|---|---|
|`Payment.Type`|String|---|Sim|Value: `PhysicalCreditCard` / Tipo da Transação|
|`Payment.Currency`|String|---|---|Default: "BRL" / Value: "BRL" / Moeda (Preencher com “BRL”)|
|`Payment.Country`|String|---|---|Default: "BRA" / Value: "BRA" / País (Preencher com “BRA”)|
|`Receipt.MerchantName`|---|---|---|---|
|`Receipt.MerchantAddress`|---|---|---|---|
|`Receipt.MerchantCity`|---|---|---|---|
|`Receipt.MerchantState`|---|---|---|---|
|`Receipt.MerchantCode`|---|---|---|---|
|`Receipt.Terminal`|---|---|---|---|
|`Receipt.Nsu`|---|---|---|---|
|`Receipt.Date`|---|---|---|---|
|`Receipt.Hour`|---|---|---|---|
|`Receipt.IssuerName`|---|---|---|---|
|`Receipt.CardNumber`|---|---|---|---|
|`Receipt.TransactionType`|---|---|---|---|
|`Receipt.AuthorizationCode`|---|---|---|---|
|`Receipt.TransactionMode`|---|---|---|---|
|`Receipt.InputMethod`|---|---|---|---|
|`Receipt.Value`|---|---|---|---|
|`Receipt.SoftDescriptor`|---|---|---|---|
|`RecurrentPayment.RecurrentPaymentId`|---|---|---|---|
|`RecurrentPayment.ReasonCode`|---|---|---|---|
|`RecurrentPayment.ReasonMessage`|---|---|---|---|
|`RecurrentPayment.NextRecurrency`|---|---|---|---|
|`RecurrentPayment.EndDate`|---|---|---|---|
|`RecurrentPayment.Interval`|---|---|---|---|
|`SplitPayments.SubordinateMerchantId`|---|---|---|---|
|`SplitPayments.Amount`|---|---|---|---|
|`SplitPayments.Fares.Mdr`|---|---|---|---|
|`SplitPayments.Fares.Fee`|---|---|---|---|
|`SplitErrors.Code`|---|---|---|---|
|`SplitErrors.Message`|---|---|---|---|

### Voucher por chip e senha online

#### Requisição

<aside class="request"><span class="method post">POST</span> <span class="endpoint">/1/physicalSales/</span></aside>

```json
{
  "MerchantOrderId": "201904150005",
  "Payment": {
    "Type": "PhysicalVoucherCard",
    "SoftDescriptor": "Description",
    "PaymentDateTime": "2019-04-15T12:00:00Z",
    "Amount": 15798,
    "ProductId": 1,
    "VoucherCard": {
      "ExpirationDate": "12/2020",
      "BrandId": 1,
      "IssuerId": 2,
      "InputMode": "Emv",
      "AuthenticationMethod": "OnlinePassword",
      "EmvData": "112233445566778899011AABBC012D3456789E0123FF45678AB901234C5D112233445566778800",
      "PinBlock": {
        "EncryptedPinBlock": "2280F6BDFD0C038D",
        "EncryptionType": "Dukpt3Des",
        "KsnIdentification": "1231vg31fv231313123"
      },
      "PanSequenceNumber": 123,
      "SaveCard": false,
      "IsFallback": false
    },
    "PinPadInformation": {
      "TerminalId": "10000001",
      "SerialNumber": "ABC123",
      "PhysicalCharacteristics": "PinPadWithChipReaderWithSamModule",
      "ReturnDataInfo": "00"
    }
  }
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`MerchantOrderId`|String|---|---| Número do documento gerado automaticamente pelo terminal e incrementado de 1 a cada transação realizada no terminal. Aceita apenas valores numéricos de 1 a 15 dígitos.|
|`Payment.Type`|String|---|Sim|Value: `PhysicalCreditCard` / Tipo da Transação|
|`Payment.SoftDescriptor`|String|13|---|Identificação do estabelecimento (nome reduzido) a ser impresso e identificado na fatura.|
|`Payment.PaymentDateTime`|String|date-time|Sim|Data e Hora da captura da transação|
|`Payment.Amount`|Integer(int64)|---|Sim|Valor da transação (1079 = R$10,79)|
|`Payment.ProductId`|Integer|---|Sim|Código do produto identificado através do bin do cartão.|
|`VoucherCard.ExpirationDate`|String|MM/yyyy|Sim|Data de validade do cartão. <br><br>Dado obtido através do comando PP_GetCard na BC no momento da captura da transação.|
|`VoucherCard.BrandId`|Integer|---|Sim|Identificação da bandeira obtida através do campo BrandId da PRODUCT TABLE.|
|`VoucherCard.IssuerId`|Integer|---|Sim|Código do emissor obtido através do campo IssuerId da BIN TABLE.|
|`VoucherCard.InputMode`|String|---|Sim|Enum: `Typed` `MagStripe` `Emv` <br><br>Identificação do modo de captura do cartão na transação. Essa informação deve ser obtida através do retorno da função PP_GetCard da BC. <br><br>"00" – Magnético <br><br>"01" - Moedeiro VISA Cash sobre TIBC v1 <br><br>"02" - Moedeiro VISA Cash sobre TIBC v3 <br><br>"03" – EMV com contato <br><br>"04" - Easy-Entry sobre TIBC v1 <br><br>"05" - Chip sem contato simulando tarja "06" - EMV sem contato.|
|`VoucherCard.AuthenticationMethod`|String|---|Sim|Enum: `NoPassword` `OnlineAuthentication` `OfflineAuthentication` <br><br>Método de autenticação <br><br>- Se o cartão foi lido a partir da digitação verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, a senha deve ser capturada e o authenticationMethod assume valor 2. Caso contrário, assume valor 1; <br><br>- Se o cartão foi lido a partir da trilha verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, deve ser verificado o bit 2 do mesmo campo. Se este estiver com valor 1 deve ser capturada a senha. Se estiver com valor 0 a captura da senha vai depender do último dígito do service code; <br><br>- Se o cartão foi lido através do chip EMV, o authenticationMethod será preenchido com base no retorno da função PP_GoOnChip(). No resultado PP_GoOnChip(), onde se o campo da posição 003 do retorno da PP_GoOnChip() estiver com valor 1 indica que o pin foi validado off-line, o authenticationMethod será 3. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valor 0, o authenticationMethod será 1. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valores 0 e 1 respectivamente, o authenticationMethod será 2. <br><br>1 - Sem senha = “NoPassword”; <br><br>2 - Senha online = “Online Authentication”; <br><br>3 - Senha off-line = “Offline Authentication”.|
|`VoucherCard.EmvData`|String|---|---|Dados da transação EMV <br><br>Obtidos através do comando PP_GoOnChip na BC|
|`PinBlock.EncryptedPinBlock`|String|---|Sim|PINBlock Criptografado<br><br>- Para transações EMV, esse campo é obtido através do retorno da função PP_GoOnChip(), mais especificamente das posições 007 até a posição 022;<br><br>- Para transações digitadas e com tarja magnética, verificar as posições 001 até 016 do retorno da função PP_GetPin().|
|`PinBlock.EncryptionType`|String|---|Sim|Tipo de Criptografia<br><br>Enum:<br><br>"DukptDes"<br><br>"Dukpt3Des"<br><br>"MasterKey"|
|`PinBlock.KsnIdentification`|String|---|Sim|Identificação do KSN<br><br>- Para transações EMV esse campo é obtido através do retorno da função PP_GoOnChip() nas posições 023 até 042;<br><br>- Para transações digitadas e com tarja magnética, verificar as posições 017 até 036 do retorno da função PP_GetPin().|
|`VoucherCard.PanSequenceNumber`|Number|---|---|Número sequencial do cartão, utilizado para identificar a conta corrente do cartão adicional. Mandatório para transações com cartões Chip EMV e que possuam PAN Sequence Number (Tag 5F34).|
|`VoucherCard.SaveCard`|Booleano|---|---|Identifica se vai salvar/tokenizar o cartão.|
|`VoucherCard.IsFallback`|Booleano|---|---|Identifica se é uma transação de fallback.|
|`PinPadInformation.TerminalId`|String|---|Sim|Número Lógico definido no Concentrador Cielo.|
|`PinPadInformation.SerialNumber`|String|---|Sim|Número de Série do Equipamento.|
|`PinPadInformation.PhysicalCharacteristics`|String|---|Sim|Enum: `WithoutPinPad` `PinPadWithoutChipReader` `PinPadWithChipReaderWithoutSamModule` `PinPadWithChipReaderWithSamModule` `NotCertifiedPinPad` `PinPadWithChipReaderWithoutSamAndContactless` `PinPadWithChipReaderWithSamModuleAndContactless` <br><br> Sem PIN-pad = `WithoutPinPad`; <br><br> PIN-pad sem leitor de Chip = `PinpadWithoutChipReader`; <br><br>PIN-pad com leitor de Chip sem módulo SAM = `PinPadWithChipReaderWithoutSamModule`; <br><br> PIN-pad com leitor de Chip com módulo SAM = `PinPadWithChipReaderWithSamModule`; <br><br> PIN-pad não homologado = `NotCertifiedPinPad`; <br><br> PIN-pad com leitor de Chip sem SAM e Cartão Sem Contato = `PinpadWithChipReaderWithoutSamAndContactless`; <br><br> PIN-pad com leitor de Chip com SAM e Cartão Sem Contato = `PinpadWithChipReaderWithSamAndContactless`. <br><br><br> Obs. Caso a aplicação não consiga informar os dados acima, deve obter tais informações através do retorno da função PP_GetInfo() da BC.|
|`PinPadInformation.ReturnDataInfo`|String|---|Sim|Retorno da função PP_GetInfo() da biblioteca compartilhada|

#### Resposta

```json
{
  "MerchantOrderId": "20180204",
  "Customer": {
    "Name": "Comprador crédito completo",
    "Identity": "11225468954",
    "IdentityType": "CPF",
    "Email": "compradorteste@teste.com",
    "Birthday": "1991-01-02",
    "Address": {
      "Street": "Rua Teste",
      "Number": "123",
      "Complement": "AP 123",
      "ZipCode": "12345987",
      "City": "São Paulo",
      "State": "SP",
      "Country": "BRA"
    },
    "DeliveryAddress": {
      "Street": "Rua Teste",
      "Number": "123",
      "Complement": "AP 123",
      "ZipCode": "12345987",
      "City": "São Paulo",
      "State": "SP",
      "Country": "BRA"
    }
  },
  "Payment": {
    "Installments": 1,
    "Interest": "ByMerchant",
    "Capture": true,
    "CreditCard": {
      "ExpirationDate": "12/2020",
      "BrandId": 1,
      "IssuerId": 2,
      "TruncateCardNumberWhenPrinting": true,
      "InputMode": "Emv",
      "AuthenticationMethod": "OnlineAuthentication",
      "EmvData": "112233445566778899011AABBC012D3456789E0123FF45678AB901234C5D112233445566778800",
      "PinBlock": {
        "EncryptedPinBlock": "2280F6BDFD0C038D",
        "EncryptionType": "Dukpt3Des",
        "KsnIdentification": "1231vg31fv231313123"
      },
      "PanSequenceNumber": 123,
      "SaveCard": false,
      "IsFallback": false
    },
    "PaymentDateTime": "2019-04-15T12:00:00Z",
    "ServiceTaxAmount": 0,
    "SoftDescriptor": "Description",
    "ProductId": 1,
    "PinPadInformation": {
      "TerminalId": "10000001",
      "SerialNumber": "ABC123",
      "PhysicalCharacteristics": "PinPadWithChipReaderWithSamModule",
      "ReturnDataInfo": "00"
    },
    "Amount": 15798,
    "ReceivedDate": "2019-04-15T12:00:00Z",
    "CapturedAmount": 15798,
    "Provider": "Cielo",
    "ConfirmationStatus": 0,
    "InitializationVersion": 1558708320029,
    "EmvResponseData": "123456789ABCD1345DEA",
    "Status": 2,
    "IsSplitted": false,
    "ReturnCode": 0,
    "ReturnMessage": "Successful",
    "PaymentId": "f15889ea-5719-4e1a-a2da-f4e50d5bd702",
    "Type": "PhysicalDebitCard",
    "Currency": "BRL",
    "Country": "BRA",
    "Links": [
      {
        "Method": "GET",
        "Rel": "self",
        "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702"
      },
      {
        "Method": "DELETE",
        "Rel": "self",
        "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702"
      },
      {
        "Method": "PUT",
        "Rel": "self",
        "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702/confirmation"
      }
    ],
    "PrintMessage": [
      {
        "Position": "Top",
        "Message": "Transação autorizada"
      },
      {
        "Position": "Middle",
        "Message": "Informação adicional"
      },
      {
        "Position": "Bottom",
        "Message": "Obrigado e volte sempre!"
      }
    ],
    "ReceiptInformation": [
      {
        "Field": "MERCHANT_NAME",
        "Label": "NOME DO ESTABELECIMENTO",
        "Content": "Estabelecimento"
      },
      {
        "Field": "MERCHANT_ADDRESS",
        "Label": "ENDEREÇO DO ESTABELECIMENTO",
        "Content": "Rua Sem Saida, 0"
      },
      {
        "Field": "MERCHANT_CITY",
        "Label": "CIDADE DO ESTABELECIMENTO",
        "Content": "Cidade"
      },
      {
        "Field": "MERCHANT_STATE",
        "Label": "ESTADO DO ESTABELECIMENTO",
        "Content": "WA"
      },
      {
        "Field": "MERCHANT_CODE",
        "Label": "COD.ESTAB.",
        "Content": 1234567890123456
      },
      {
        "Field": "TERMINAL",
        "Label": "POS",
        "Content": 12345678
      },
      {
        "Field": "NSU",
        "Label": "DOC",
        "Content": 123456
      },
      {
        "Field": "DATE",
        "Label": "DATA",
        "Content": "01/01/20"
      },
      {
        "Field": "HOUR",
        "Label": "HORA",
        "Content": "01:01"
      },
      {
        "Field": "ISSUER_NAME",
        "Label": "EMISSOR",
        "Content": "NOME DO EMISSOR"
      },
      {
        "Field": "CARD_NUMBER",
        "Label": "CARTÃO",
        "Content": 5432123454321234
      },
      {
        "Field": "TRANSACTION_TYPE",
        "Label": "TIPO DE TRANSAÇÃO",
        "Content": "VENDA A CREDITO"
      },
      {
        "Field": "AUTHORIZATION_CODE",
        "Label": "AUTORIZAÇÃO",
        "Content": 123456
      },
      {
        "Field": "TRANSACTION_MODE",
        "Label": "MODO DA TRANSAÇÃO",
        "Content": "ONL"
      },
      {
        "Field": "INPUT_METHOD",
        "Label": "MODO DE ENTRADA",
        "Content": "X"
      },
      {
        "Field": "VALUE",
        "Label": "VALOR",
        "Content": "1,23"
      },
      {
        "Field": "SOFT_DESCRIPTOR",
        "Label": "SOFT DESCRIPTOR",
        "Content": "Simulado"
      }
    ],
    "Receipt": {
      "MerchantName": "Estabelecimento",
      "MerchantAddress": "Rua Sem Saida, 0",
      "MerchantCity": "Cidade",
      "MerchantState": "WA",
      "MerchantCode": 1234567890123456,
      "Terminal": 12345678,
      "Nsu": 123456,
      "Date": "01/01/20",
      "Hour": "01:01",
      "IssuerName": "NOME DO EMISSOR",
      "CardNumber": 5432123454321234,
      "TransactionType": "VENDA A CREDITO",
      "AuthorizationCode": 123456,
      "TransactionMode": "ONL",
      "InputMethod": "X",
      "Value": "1,23",
      "SoftDescriptor": "Simulado"
    },
    "RecurrentPayment": {
      "RecurrentPaymentId": "a6b719fa-a8df-ab11-4e1a-f4e50d5bd702",
      "ReasonCode": 0,
      "ReasonMessage": "Successful",
      "NextRecurrency": "2019-12-01",
      "EndDate": "2019-12-01",
      "Interval": 6
    },
    "SplitPayments": [
      {
        "SubordinateMerchantId": "491daf20-35f2-4379-874c-e7552ae8dc10",
        "Amount": 100,
        "Fares": {
          "Mdr": 5,
          "Fee": 0
        }
      },
      {
        "SubordinateMerchantId": "7e2846be-4e80-4f86-8ca9-eb35db6aea00",
        "Amount": 80,
        "Fares": {
          "Mdr": 3,
          "Fee": 1
        }
      }
    ],
    "SplitErrors": [
      {
        "Code": 326,
        "Message": "SubordinatePayment amount must be greater than zero"
      }
    ]
  }
}
```

|Property|Type|Size|Required|Description|
|---|---|---|---|---|
|`MerchantOrderId`|String|---|---| Número do documento gerado automaticamente pelo terminal e incrementado de 1 a cada transação realizada no terminal. Aceita apenas valores numéricos de 1 a 15 dígitos.|
|`Customer.Name`|String|---|---|---|---|
|`Customer.Identity`|---|---|---|---|
|`Customer.IdentityType`|---|---|---|---|
|`Customer.Email`|---|---|---||---|
|`Customer.Birthday`|---|---|---|---|
|`Address.Street`|String|---|---|---|
|`Address.Number`|String|---|---|---|
|`Address.Complement`|String|---|---|---|
|`Address.ZipCode`|String|---|---|---|
|`Address.City`|String|---|---|---|
|`Address.State`|String|---|---|---|
|`Address.Country`|String|---|---|---|
|`DeliveryAddress.Street`|String|---|---|---|
|`DeliveryAddress.Number`|String|---|---|---|
|`DeliveryAddress.Complement`|String|---|---|---|
|`DeliveryAddress.ZipCode`|String|---|---|---|
|`DeliveryAddress.City`|String|---|---|---|
|`DeliveryAddress.State`|String|---|---|---|
|`DeliveryAddress.Country`|String|---|---|---|
|`Payment.Installments`|Integer|---|---|Default: 1 / Quantidade de Parcelas: Varia de 2 a 99 para transação de financiamento. Deve ser verificado os atributos maxOfPayments1, maxOfPayments2, maxOfPayments3 e minValOfPayments da tabela productTable.|
|`Payment.Interest`|String|---|---|Default: `ByMerchant` <br><br> Enum: `ByMerchant` `ByIssuer` <br><br> Tipo de Parcelamento: <br><br> - Se o bit 6 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 6 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento sem juros pode ser efetuado. <br><br> - Se o bit 7 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 7 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento com juros pode ser efetuado. Sem juros = “ByMerchant”; Com juros = “ByIssuer”.|
|`Payment.Capture`|Booleano|---|---|Default: false / Booleano que identifica que a autorização deve ser com captura automática. A autorização sem captura automática é conhecida também como pré-autorização.|
|`CreditCard.ExpirationDate`|String|MM/yyyy|Sim|Data de validade do cartão. <br><br>Dado obtido através do comando PP_GetCard na BC no momento da captura da transação.|
|`CreditCard.BrandId`|Integer|---|Sim|Identificação da bandeira obtida através do campo BrandId da PRODUCT TABLE.|
|`CreditCard.IssuerId`|Integer|---|Sim|Código do emissor obtido através do campo IssuerId da BIN TABLE.|
|`CreditCard.TruncateCardNumberWhenPrinting`|Booleano|---|---|Indica se o número do cartão será truncado no momento da impressão do comprovante. A solução de captura deve tomar essa decisão com base no `confParamOp03` presente nas tabelas BIN TABLE, PARAMETER TABLE e ISSUER TABLE.|
|`CreditCard.InputMode`|String|---|Sim|Enum: `Typed` `MagStripe` `Emv` <br><br>Identificação do modo de captura do cartão na transação. Essa informação deve ser obtida através do retorno da função PP_GetCard da BC. <br><br>"00" – Magnético <br><br>"01" - Moedeiro VISA Cash sobre TIBC v1 <br><br>"02" - Moedeiro VISA Cash sobre TIBC v3 <br><br>"03" – EMV com contato <br><br>"04" - Easy-Entry sobre TIBC v1 <br><br>"05" - Chip sem contato simulando tarja <br><br> "06" - EMV sem contato.|
|`CreditCard.AuthenticationMethod`|String|---|Sim|Enum: `NoPassword` `OnlineAuthentication` `OfflineAuthentication` <br><br>Método de autenticação <br><br>- Se o cartão foi lido a partir da digitação verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, a senha deve ser capturada e o authenticationMethod assume valor 2. Caso contrário, assume valor 1; <br><br>- Se o cartão foi lido a partir da trilha verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, deve ser verificado o bit 2 do mesmo campo. Se este estiver com valor 1 deve ser capturada a senha. Se estiver com valor 0 a captura da senha vai depender do último dígito do service code; <br><br>- Se o cartão foi lido através do chip EMV, o authenticationMethod será preenchido com base no retorno da função PP_GoOnChip(). No resultado PP_GoOnChip(), onde se o campo da posição 003 do retorno da PP_GoOnChip() estiver com valor 1 indica que o pin foi validado off-line, o authenticationMethod será 3. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valor 0, o authenticationMethod será 1. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valores 0 e 1 respectivamente, o authenticationMethod será 2. <br><br>1 - Sem senha = “NoPassword”; <br><br>2 - Senha online = “Online Authentication”; <br><br>3 - Senha off-line = “Offline Authentication”.|
|`CreditCard.EmvData`|String|---|---|Dados da transação EMV <br><br>Obtidos através do comando PP_GoOnChip na BC|
|`PinBlock.EncryptedPinBlock`|String|---|Sim|PINBlock Criptografado<br><br>- Para transações EMV, esse campo é obtido através do retorno da função PP_GoOnChip(), mais especificamente das posições 007 até a posição 022;<br><br>- Para transações digitadas e com tarja magnética, verificar as posições 001 até 016 do retorno da função PP_GetPin().|
|`PinBlock.EncryptionType`|String|---|Sim|Tipo de Criptografia<br><br>Enum:<br><br>"DukptDes"<br><br>"Dukpt3Des"<br><br>"MasterKey"|
|`PinBlock.KsnIdentification`|String|---|Sim|Identificação do KSN<br><br>- Para transações EMV esse campo é obtido através do retorno da função PP_GoOnChip() nas posições 023 até 042;<br><br>- Para transações digitadas e com tarja magnética, verificar as posições 017 até 036 do retorno da função PP_GetPin().|
|`CreditCard.PanSequenceNumber`|Number|---|---|Número sequencial do cartão, utilizado para identificar a conta corrente do cartão adicional. Mandatório para transações com cartões Chip EMV e que possuam PAN Sequence Number (Tag 5F34).|
|`VoucherCard.SaveCard`|Booleano|---|---|Identifica se vai salvar/tokenizar o cartão.|
|`VoucherCard.IsFallback`|Booleano|---|---|Identifica se é uma transação de fallback.|
|`Payment.PaymentDateTime`|String|date-time|Sim|Data e Hora da captura da transação|
|`Payment.ServiceTaxAmount`|---|---|---|---|
|`Payment.SoftDescriptor`|String|13|---|Identificação do estabelecimento (nome reduzido) a ser impresso e identificado na fatura.|
|`Payment.ProductId`|Integer|---|Sim|Código do produto identificado através do bin do cartão.|
|`PinPadInformation.TerminalId`|String|---|Sim|Número Lógico definido no Concentrador Cielo.|
|`PinPadInformation.SerialNumber`|String|---|Sim|Número de Série do Equipamento.|
|`PinPadInformation.PhysicalCharacteristics`|String|---|Sim|Enum: `WithoutPinPad` `PinPadWithoutChipReader` `PinPadWithChipReaderWithoutSamModule` `PinPadWithChipReaderWithSamModule` `NotCertifiedPinPad` `PinPadWithChipReaderWithoutSamAndContactless` `PinPadWithChipReaderWithSamModuleAndContactless` <br><br> Sem PIN-pad = `WithoutPinPad`; <br><br> PIN-pad sem leitor de Chip = `PinpadWithoutChipReader`; <br><br>PIN-pad com leitor de Chip sem módulo SAM = `PinPadWithChipReaderWithoutSamModule`; <br><br> PIN-pad com leitor de Chip com módulo SAM = `PinPadWithChipReaderWithSamModule`; <br><br> PIN-pad não homologado = `NotCertifiedPinPad`; <br><br> PIN-pad com leitor de Chip sem SAM e Cartão Sem Contato = `PinpadWithChipReaderWithoutSamAndContactless`; <br><br> PIN-pad com leitor de Chip com SAM e Cartão Sem Contato = `PinpadWithChipReaderWithSamAndContactless`. <br><br><br> Obs. Caso a aplicação não consiga informar os dados acima, deve obter tais informações através do retorno da função PP_GetInfo() da BC.|
|`PinPadInformation.ReturnDataInfo`|String|---|Sim|Retorno da função PP_GetInfo() da biblioteca compartilhada|
|`Payment.Amount`|Integer(int64)|---|Sim|Valor da transação (1079 = R$10,79)|
|`Payment.ReceivedDate`|---|---|---|---|
|`Payment.CapturedAmount`|---|---|---|---|
|`Payment.Provider`|String|---|---|---|
|`Payment.ConfirmationStatus`|---|---|---|---|
|`Payment.InitializationVersion`|---|---|---|---|
|`Payment.EmvResponseData`|---|---|---|---|
|`Payment.Status`|---|---|---|---|
|`Payment.IsSplitted`|Booleano|---|---|---|
|`Payment.ReturnCode`|---|---|---|---|
|`Payment.ReturnMessage`|String|---|---|---|
|`Payment.PaymentId`|---|---|---|---|
|`Payment.Type`|String|---|Sim|Value: `PhysicalCreditCard` / Tipo da Transação|
|`Payment.Currency`|String|---|---|Default: "BRL" / Value: "BRL" / Moeda (Preencher com “BRL”)|
|`Payment.Country`|String|---|---|Default: "BRA" / Value: "BRA" / País (Preencher com “BRA”)|
|`Receipt.MerchantName`|---|---|---|---|
|`Receipt.MerchantAddress`|---|---|---|---|
|`Receipt.MerchantCity`|---|---|---|---|
|`Receipt.MerchantState`|---|---|---|---|
|`Receipt.MerchantCode`|---|---|---|---|
|`Receipt.Terminal`|---|---|---|---|
|`Receipt.Nsu`|---|---|---|---|
|`Receipt.Date`|---|---|---|---|
|`Receipt.Hour`|---|---|---|---|
|`Receipt.IssuerName`|---|---|---|---|
|`Receipt.CardNumber`|---|---|---|---|
|`Receipt.TransactionType`|---|---|---|---|
|`Receipt.AuthorizationCode`|---|---|---|---|
|`Receipt.TransactionMode`|---|---|---|---|
|`Receipt.InputMethod`|---|---|---|---|
|`Receipt.Value`|---|---|---|---|
|`Receipt.SoftDescriptor`|---|---|---|---|
|`RecurrentPayment.RecurrentPaymentId`|---|---|---|---|
|`RecurrentPayment.ReasonCode`|---|---|---|---|
|`RecurrentPayment.ReasonMessage`|---|---|---|---|
|`RecurrentPayment.NextRecurrency`|---|---|---|---|
|`RecurrentPayment.EndDate`|---|---|---|---|
|`RecurrentPayment.Interval`|---|---|---|---|
|`SplitPayments.SubordinateMerchantId`|---|---|---|---|
|`SplitPayments.Amount`|---|---|---|---|
|`SplitPayments.Fares.Mdr`|---|---|---|---|
|`SplitPayments.Fares.Fee`|---|---|---|---|
|`SplitErrors.Code`|---|---|---|---|
|`SplitErrors.Message`|---|---|---|---|

### Crédito a vista com QRCode

#### Requisição

<aside class="request"><span class="method post">POST</span> <span class="endpoint">/1/physicalSales/</span></aside>

```json
{
   "MerchantOrderId": "1596226820548",
   "Payment": {
      "SubordinatedMerchantId": "{Auth_ClientId}",
      "Type": "PhysicalCreditCard",
      "SoftDescriptor": "Description",
      "PaymentDateTime": "2020-07-31T20:20:20.548Z",
      "Amount": 200,
      "Installments": 1,
      "Interest": "ByMerchant",
      "Capture": true,
      "ProductId": 1,
      "CreditCard": {
         "InputMode": "QRCode",
         "QRCodeData": "180313001B1D4349",
         "PinBlock": {
            "EncryptedPinBlock": "2280F6BDFD0C038D",
            "EncryptionType": "Dukpt3Des",
            "KsnIdentification": "fffff9999900522000d6"
         }
      },
      "PinPadInformation": {
         "TerminalId": "12345678",
         "SerialNumber": "6C651996",
         "PhysicalCharacteristics": "PinPadWithChipReaderWithSamModuleAndContactless",
         "ReturnDataInfo": "00"
      }
   }
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`MerchantOrderId`|String|---|---| Número do documento gerado automaticamente pelo terminal e incrementado de 1 a cada transação realizada no terminal. Aceita apenas valores numéricos de 1 a 15 dígitos.|
|`Payment.Type`|String|---|Sim|Value: `PhysicalCreditCard` / Tipo da Transação|
|`Payment.SoftDescriptor`|String|13|---|Identificação do estabelecimento (nome reduzido) a ser impresso e identificado na fatura.|
|`Payment.PaymentDateTime`|String|date-time|Sim|Data e Hora da captura da transação|
|`Payment.Amount`|Integer(int64)|---|Sim|Valor da transação (1079 = R$10,79)|
|`Payment.Capture`|Booleano|---|---|Default: false / Booleano que identifica que a autorização deve ser com captura automática. A autorização sem captura automática é conhecida também como pré-autorização.|
|`Payment.Installments`|Integer|---|---|Default: 1 / Quantidade de Parcelas: Varia de 2 a 99 para transação de financiamento. Deve ser verificado os atributos maxOfPayments1, maxOfPayments2, maxOfPayments3 e minValOfPayments da tabela productTable.|
|`Payment.Interest`|String|---|---|Default: `ByMerchant` <br><br> Enum: `ByMerchant` `ByIssuer` <br><br> Tipo de Parcelamento: <br><br> - Se o bit 6 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 6 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento sem juros pode ser efetuado. <br><br> - Se o bit 7 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 7 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento com juros pode ser efetuado. Sem juros = “ByMerchant”; Com juros = “ByIssuer”.|
|`Payment.ProductId`|Integer|---|Sim|Código do produto identificado através do bin do cartão.|
|`CreditCard.InputMode`|String|---|Sim|Enum: `Typed` `MagStripe` `Emv` <br><br>Identificação do modo de captura do cartão na transação. Essa informação deve ser obtida através do retorno da função PP_GetCard da BC. <br><br>"00" – Magnético <br><br>"01" - Moedeiro VISA Cash sobre TIBC v1 <br><br>"02" - Moedeiro VISA Cash sobre TIBC v3 <br><br>"03" – EMV com contato <br><br>"04" - Easy-Entry sobre TIBC v1 <br><br>"05" - Chip sem contato simulando tarja <br><br> "06" - EMV sem contato.|
|`CreditCard.QRCodeData`|String|---|Sim|QRCodeData|
|`CreditCard.PinBlock.EncryptedPinBlock`|String|---|Sim|PINBlock Criptografado<br><br>- Para transações EMV, esse campo é obtido através do retorno da função PP_GoOnChip(), mais especificamente das posições 007 até a posição 022;<br><br>- Para transações digitadas e com tarja magnética, verificar as posições 001 até 016 do retorno da função PP_GetPin().|
|`CreditCard.PinBlock.EncryptionType`|String|---|Sim|Tipo de Criptografia<br><br>Enum:<br><br>"DukptDes"<br><br>"Dukpt3Des"<br><br>"MasterKey"|
|`CreditCard.PinBlock.KsnIdentification`|String|---|Sim|Identificação do KSN<br><br>- Para transações EMV esse campo é obtido através do retorno da função PP_GoOnChip() nas posições 023 até 042;<br><br>- Para transações digitadas e com tarja magnética, verificar as posições 017 até 036 do retorno da função PP_GetPin().|
|`PinPadInformation.TerminalId`|String|---|Sim|Número Lógico definido no Concentrador Cielo.|
|`PinPadInformation.SerialNumber`|String|---|Sim|Número de Série do Equipamento.|
|`PinPadInformation.PhysicalCharacteristics`|String|---|Sim|Enum: `WithoutPinPad` `PinPadWithoutChipReader` `PinPadWithChipReaderWithoutSamModule` `PinPadWithChipReaderWithSamModule` `NotCertifiedPinPad` `PinPadWithChipReaderWithoutSamAndContactless` `PinPadWithChipReaderWithSamModuleAndContactless` <br><br> Sem PIN-pad = `WithoutPinPad`; <br><br> PIN-pad sem leitor de Chip = `PinpadWithoutChipReader`; <br><br>PIN-pad com leitor de Chip sem módulo SAM = `PinPadWithChipReaderWithoutSamModule`; <br><br> PIN-pad com leitor de Chip com módulo SAM = `PinPadWithChipReaderWithSamModule`; <br><br> PIN-pad não homologado = `NotCertifiedPinPad`; <br><br> PIN-pad com leitor de Chip sem SAM e Cartão Sem Contato = `PinpadWithChipReaderWithoutSamAndContactless`; <br><br> PIN-pad com leitor de Chip com SAM e Cartão Sem Contato = `PinpadWithChipReaderWithSamAndContactless`. <br><br><br> Obs. Caso a aplicação não consiga informar os dados acima, deve obter tais informações através do retorno da função PP_GetInfo() da BC.|
|`PinPadInformation.ReturnDataInfo`|String|---|Sim|Retorno da função PP_GetInfo() da biblioteca compartilhada|

#### Resposta

|Property|Type|Size|Required|Description|
|---|---|---|---|---|
|`MerchantOrderId`|String|---|---| Número do documento gerado automaticamente pelo terminal e incrementado de 1 a cada transação realizada no terminal. Aceita apenas valores numéricos de 1 a 15 dígitos.|
|`Customer.Name`|String|---|---|---|---|
|`Customer.Identity`|---|---|---|---|
|`Customer.IdentityType`|---|---|---|---|
|`Customer.Email`|---|---|---||---|
|`Customer.Birthday`|---|---|---|---|
|`Address.Street`|String|---|---|---|
|`Address.Number`|String|---|---|---|
|`Address.Complement`|String|---|---|---|
|`Address.ZipCode`|String|---|---|---|
|`Address.City`|String|---|---|---|
|`Address.State`|String|---|---|---|
|`Address.Country`|String|---|---|---|
|`DeliveryAddress.Street`|String|---|---|---|
|`DeliveryAddress.Number`|String|---|---|---|
|`DeliveryAddress.Complement`|String|---|---|---|
|`DeliveryAddress.ZipCode`|String|---|---|---|
|`DeliveryAddress.City`|String|---|---|---|
|`DeliveryAddress.State`|String|---|---|---|
|`DeliveryAddress.Country`|String|---|---|---|
|`Payment.Installments`|Integer|---|---|Default: 1 / Quantidade de Parcelas: Varia de 2 a 99 para transação de financiamento. Deve ser verificado os atributos maxOfPayments1, maxOfPayments2, maxOfPayments3 e minValOfPayments da tabela productTable.|
|`Payment.Interest`|String|---|---|Default: `ByMerchant` <br><br> Enum: `ByMerchant` `ByIssuer` <br><br> Tipo de Parcelamento: <br><br> - Se o bit 6 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 6 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento sem juros pode ser efetuado. <br><br> - Se o bit 7 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 7 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento com juros pode ser efetuado. Sem juros = “ByMerchant”; Com juros = “ByIssuer”.|
|`Payment.Capture`|Booleano|---|---|Default: false / Booleano que identifica que a autorização deve ser com captura automática. A autorização sem captura automática é conhecida também como pré-autorização.|
|`CreditCard.ExpirationDate`|String|MM/yyyy|Sim|Data de validade do cartão. <br><br>Dado obtido através do comando PP_GetCard na BC no momento da captura da transação.|
|`CreditCard.BrandId`|Integer|---|Sim|Identificação da bandeira obtida através do campo BrandId da PRODUCT TABLE.|
|`CreditCard.IssuerId`|Integer|---|Sim|Código do emissor obtido através do campo IssuerId da BIN TABLE.|
|`CreditCard.TruncateCardNumberWhenPrinting`|Booleano|---|---|Indica se o número do cartão será truncado no momento da impressão do comprovante. A solução de captura deve tomar essa decisão com base no `confParamOp03` presente nas tabelas BIN TABLE, PARAMETER TABLE e ISSUER TABLE.|
|`CreditCard.InputMode`|String|---|Sim|Enum: `Typed` `MagStripe` `Emv` <br><br>Identificação do modo de captura do cartão na transação. Essa informação deve ser obtida através do retorno da função PP_GetCard da BC. <br><br>"00" – Magnético <br><br>"01" - Moedeiro VISA Cash sobre TIBC v1 <br><br>"02" - Moedeiro VISA Cash sobre TIBC v3 <br><br>"03" – EMV com contato <br><br>"04" - Easy-Entry sobre TIBC v1 <br><br>"05" - Chip sem contato simulando tarja <br><br> "06" - EMV sem contato.|
|`CreditCard.AuthenticationMethod`|String|---|Sim|Enum: `NoPassword` `OnlineAuthentication` `OfflineAuthentication` <br><br>Método de autenticação <br><br>- Se o cartão foi lido a partir da digitação verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, a senha deve ser capturada e o authenticationMethod assume valor 2. Caso contrário, assume valor 1; <br><br>- Se o cartão foi lido a partir da trilha verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, deve ser verificado o bit 2 do mesmo campo. Se este estiver com valor 1 deve ser capturada a senha. Se estiver com valor 0 a captura da senha vai depender do último dígito do service code; <br><br>- Se o cartão foi lido através do chip EMV, o authenticationMethod será preenchido com base no retorno da função PP_GoOnChip(). No resultado PP_GoOnChip(), onde se o campo da posição 003 do retorno da PP_GoOnChip() estiver com valor 1 indica que o pin foi validado off-line, o authenticationMethod será 3. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valor 0, o authenticationMethod será 1. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valores 0 e 1 respectivamente, o authenticationMethod será 2. <br><br>1 - Sem senha = “NoPassword”; <br><br>2 - Senha online = “Online Authentication”; <br><br>3 - Senha off-line = “Offline Authentication”.|
|`CreditCard.EmvData`|String|---|---|Dados da transação EMV <br><br>Obtidos através do comando PP_GoOnChip na BC|
|`PinBlock.EncryptedPinBlock`|String|---|Sim|PINBlock Criptografado<br><br>- Para transações EMV, esse campo é obtido através do retorno da função PP_GoOnChip(), mais especificamente das posições 007 até a posição 022;<br><br>- Para transações digitadas e com tarja magnética, verificar as posições 001 até 016 do retorno da função PP_GetPin().|
|`PinBlock.EncryptionType`|String|---|Sim|Tipo de Criptografia<br><br>Enum:<br><br>"DukptDes"<br><br>"Dukpt3Des"<br><br>"MasterKey"|
|`PinBlock.KsnIdentification`|String|---|Sim|Identificação do KSN<br><br>- Para transações EMV esse campo é obtido através do retorno da função PP_GoOnChip() nas posições 023 até 042;<br><br>- Para transações digitadas e com tarja magnética, verificar as posições 017 até 036 do retorno da função PP_GetPin().|
|`CreditCard.PanSequenceNumber`|Number|---|---|Número sequencial do cartão, utilizado para identificar a conta corrente do cartão adicional. Mandatório para transações com cartões Chip EMV e que possuam PAN Sequence Number (Tag 5F34).|
|`VoucherCard.SaveCard`|Booleano|---|---|Identifica se vai salvar/tokenizar o cartão.|
|`VoucherCard.IsFallback`|Booleano|---|---|Identifica se é uma transação de fallback.|
|`Payment.PaymentDateTime`|String|date-time|Sim|Data e Hora da captura da transação|
|`Payment.ServiceTaxAmount`|---|---|---|---|
|`Payment.SoftDescriptor`|String|13|---|Identificação do estabelecimento (nome reduzido) a ser impresso e identificado na fatura.|
|`Payment.ProductId`|Integer|---|Sim|Código do produto identificado através do bin do cartão.|
|`PinPadInformation.TerminalId`|String|---|Sim|Número Lógico definido no Concentrador Cielo.|
|`PinPadInformation.SerialNumber`|String|---|Sim|Número de Série do Equipamento.|
|`PinPadInformation.PhysicalCharacteristics`|String|---|Sim|Enum: `WithoutPinPad` `PinPadWithoutChipReader` `PinPadWithChipReaderWithoutSamModule` `PinPadWithChipReaderWithSamModule` `NotCertifiedPinPad` `PinPadWithChipReaderWithoutSamAndContactless` `PinPadWithChipReaderWithSamModuleAndContactless` <br><br> Sem PIN-pad = `WithoutPinPad`; <br><br> PIN-pad sem leitor de Chip = `PinpadWithoutChipReader`; <br><br>PIN-pad com leitor de Chip sem módulo SAM = `PinPadWithChipReaderWithoutSamModule`; <br><br> PIN-pad com leitor de Chip com módulo SAM = `PinPadWithChipReaderWithSamModule`; <br><br> PIN-pad não homologado = `NotCertifiedPinPad`; <br><br> PIN-pad com leitor de Chip sem SAM e Cartão Sem Contato = `PinpadWithChipReaderWithoutSamAndContactless`; <br><br> PIN-pad com leitor de Chip com SAM e Cartão Sem Contato = `PinpadWithChipReaderWithSamAndContactless`. <br><br><br> Obs. Caso a aplicação não consiga informar os dados acima, deve obter tais informações através do retorno da função PP_GetInfo() da BC.|
|`PinPadInformation.ReturnDataInfo`|String|---|Sim|Retorno da função PP_GetInfo() da biblioteca compartilhada|
|`Payment.Amount`|Integer(int64)|---|Sim|Valor da transação (1079 = R$10,79)|
|`Payment.ReceivedDate`|---|---|---|---|
|`Payment.CapturedAmount`|---|---|---|---|
|`Payment.Provider`|String|---|---|---|
|`Payment.ConfirmationStatus`|---|---|---|---|
|`Payment.InitializationVersion`|---|---|---|---|
|`Payment.EmvResponseData`|---|---|---|---|
|`Payment.Status`|---|---|---|---|
|`Payment.IsSplitted`|Booleano|---|---|---|
|`Payment.ReturnCode`|---|---|---|---|
|`Payment.ReturnMessage`|String|---|---|---|
|`Payment.PaymentId`|---|---|---|---|
|`Payment.Type`|String|---|Sim|Value: `PhysicalCreditCard` / Tipo da Transação|
|`Payment.Currency`|String|---|---|Default: "BRL" / Value: "BRL" / Moeda (Preencher com “BRL”)|
|`Payment.Country`|String|---|---|Default: "BRA" / Value: "BRA" / País (Preencher com “BRA”)|
|`Receipt.MerchantName`|---|---|---|---|
|`Receipt.MerchantAddress`|---|---|---|---|
|`Receipt.MerchantCity`|---|---|---|---|
|`Receipt.MerchantState`|---|---|---|---|
|`Receipt.MerchantCode`|---|---|---|---|
|`Receipt.Terminal`|---|---|---|---|
|`Receipt.Nsu`|---|---|---|---|
|`Receipt.Date`|---|---|---|---|
|`Receipt.Hour`|---|---|---|---|
|`Receipt.IssuerName`|---|---|---|---|
|`Receipt.CardNumber`|---|---|---|---|
|`Receipt.TransactionType`|---|---|---|---|
|`Receipt.AuthorizationCode`|---|---|---|---|
|`Receipt.TransactionMode`|---|---|---|---|
|`Receipt.InputMethod`|---|---|---|---|
|`Receipt.Value`|---|---|---|---|
|`Receipt.SoftDescriptor`|---|---|---|---|
|`RecurrentPayment.RecurrentPaymentId`|---|---|---|---|
|`RecurrentPayment.ReasonCode`|---|---|---|---|
|`RecurrentPayment.ReasonMessage`|---|---|---|---|
|`RecurrentPayment.NextRecurrency`|---|---|---|---|
|`RecurrentPayment.EndDate`|---|---|---|---|
|`RecurrentPayment.Interval`|---|---|---|---|
|`SplitPayments.SubordinateMerchantId`|---|---|---|---|
|`SplitPayments.Amount`|---|---|---|---|
|`SplitPayments.Fares.Mdr`|---|---|---|---|
|`SplitPayments.Fares.Fee`|---|---|---|---|
|`SplitErrors.Code`|---|---|---|---|
|`SplitErrors.Message`|---|---|---|---|

### Crédito Offline

#### Requisição

<aside class="request"><span class="method post">POST</span> <span class="endpoint">/1/physicalSales/</span></aside>

```json
{
   "MerchantOrderId": "1596226820548",
   "Payment": {
      "Type": "PhysicalCreditCard",
      "SoftDescriptor": "Description",
      "PaymentDateTime": "2020-07-31T20:20:20.548Z",
      "Amount": 300,
      "Installments": 1,
      "Interest": "ByMerchant",
      "Capture": true,
      "ProductId": 1,
      "OfflinePaymentType": "OfflineWithoutAttemptOnline",
      "CreditCard": {
         "InputMode": "Emv",
         "BrandId": 1,
         "IssuerId": 401,
         "TruncateCardNumberWhenPrinting": true,
         "ExpirationDate": "12/2020",
         "PanSequenceNumber": 1,
         "EmvData": "112233445566778899011AABBC012D3456789E0123FF45678AB901234C5D112233445566778800",
         "TrackOneData": "A1234567890123456^FULANO OLIVEIRA SA ^12345678901234567890123",
         "TrackTwoData": "0123456789012345=012345678901234",
         "AuthenticationMethod": "OnlineAuthentication",
         "PinBlock": {
            "EncryptedPinBlock": "2280F6BDFD0C038D",
            "EncryptionType": "Dukpt3Des",
            "KsnIdentification": "fffff9999900522000d6"
         }
      },
      "PinPadInformation": {
         "TerminalId": "12345678",
         "SerialNumber": "6C651996",
         "PhysicalCharacteristics": "PinPadWithChipReaderWithSamModuleAndContactless",
         "ReturnDataInfo": "00"
      }
   }
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`MerchantOrderId`|String|---|---| Número do documento gerado automaticamente pelo terminal e incrementado de 1 a cada transação realizada no terminal. Aceita apenas valores numéricos de 1 a 15 dígitos.|
|`Payment.Type`|String|---|Sim|Value: `PhysicalCreditCard` / Tipo da Transação|
|`Payment.SoftDescriptor`|String|13|---|Identificação do estabelecimento (nome reduzido) a ser impresso e identificado na fatura.|
|`Payment.PaymentDateTime`|String|date-time|Sim|Data e Hora da captura da transação|
|`Payment.Amount`|Integer(int64)|---|Sim|Valor da transação (1079 = R$10,79)|
|`Payment.Installments`|Integer|---|---|Default: 1 / Quantidade de Parcelas: Varia de 2 a 99 para transação de financiamento. Deve ser verificado os atributos maxOfPayments1, maxOfPayments2, maxOfPayments3 e minValOfPayments da tabela productTable.|
|`Payment.Interest`|String|---|---|Default: `ByMerchant` <br><br> Enum: `ByMerchant` `ByIssuer` <br><br> Tipo de Parcelamento: <br><br> - Se o bit 6 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 6 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento sem juros pode ser efetuado. <br><br> - Se o bit 7 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 7 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento com juros pode ser efetuado. Sem juros = “ByMerchant”; Com juros = “ByIssuer”.|
|`Payment.Capture`|Booleano|---|---|Default: false / Booleano que identifica que a autorização deve ser com captura automática. A autorização sem captura automática é conhecida também como pré-autorização.|
|`Payment.ProductId`|Integer|---|Sim|Código do produto identificado através do bin do cartão.|
|`CreditCard.ExpirationDate`|String|MM/yyyy|Sim|Data de validade do cartão. <br><br>Dado obtido através do comando PP_GetCard na BC no momento da captura da transação.|
|`CreditCard.BrandId`|Integer|---|Sim|Identificação da bandeira obtida através do campo BrandId da PRODUCT TABLE.|
|`CreditCard.IssuerId`|Integer|---|Sim|Código do emissor obtido através do campo IssuerId da BIN TABLE.|
|`CreditCard.InputMode`|String|---|Sim|Enum: `Typed` `MagStripe` `Emv` <br><br>Identificação do modo de captura do cartão na transação. Essa informação deve ser obtida através do retorno da função PP_GetCard da BC. <br><br>"00" – Magnético <br><br>"01" - Moedeiro VISA Cash sobre TIBC v1 <br><br>"02" - Moedeiro VISA Cash sobre TIBC v3 <br><br>"03" – EMV com contato <br><br>"04" - Easy-Entry sobre TIBC v1 <br><br>"05" - Chip sem contato simulando tarja <br><br> "06" - EMV sem contato.|
|`CreditCard.AuthenticationMethod`|String|---|Sim|Enum: `NoPassword` `OnlineAuthentication` `OfflineAuthentication` <br><br>Método de autenticação <br><br>- Se o cartão foi lido a partir da digitação verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, a senha deve ser capturada e o authenticationMethod assume valor 2. Caso contrário, assume valor 1; <br><br>- Se o cartão foi lido a partir da trilha verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, deve ser verificado o bit 2 do mesmo campo. Se este estiver com valor 1 deve ser capturada a senha. Se estiver com valor 0 a captura da senha vai depender do último dígito do service code; <br><br>- Se o cartão foi lido através do chip EMV, o authenticationMethod será preenchido com base no retorno da função PP_GoOnChip(). No resultado PP_GoOnChip(), onde se o campo da posição 003 do retorno da PP_GoOnChip() estiver com valor 1 indica que o pin foi validado off-line, o authenticationMethod será 3. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valor 0, o authenticationMethod será 1. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valores 0 e 1 respectivamente, o authenticationMethod será 2. <br><br>1 - Sem senha = “NoPassword”; <br><br>2 - Senha online = “Online Authentication”; <br><br>3 - Senha off-line = “Offline Authentication”.|
|`CreditCard.EmvData`|String|---|---|Dados da transação EMV<br><br>Obtidos através do comando PP_GoOnChip na BC|
|`CreditCard.PanSequenceNumber`|Number|---|---|Número sequencial do cartão, utilizado para identificar a conta corrente do cartão adicional. Mandatório para transações com cartões Chip EMV e que possuam PAN Sequence Number (Tag 5F34).|
|`CreditCard.SaveCard`|Booleano|---|---|Identifica se vai salvar/tokenizar o cartão.|
|`CreditCard.IsFallback`|Booleano|---|---|Identifica se é uma transação de fallback.|
|`PinBlock.EncryptedPinBlock`|String|---|Sim|PINBlock Criptografado<br><br>- Para transações EMV, esse campo é obtido através do retorno da função PP_GoOnChip(), mais especificamente das posições 007 até a posição 022;<br><br>- Para transações digitadas e com tarja magnética, verificar as posições 001 até 016 do retorno da função PP_GetPin().|
|`PinBlock.EncryptionType`|String|---|Sim|Tipo de Criptografia<br><br>Enum:<br><br>"DukptDes"<br><br>"Dukpt3Des"<br><br>"MasterKey"|
|`PinBlock.KsnIdentification`|String|---|Sim|Identificação do KSN<br><br>- Para transações EMV esse campo é obtido através do retorno da função PP_GoOnChip() nas posições 023 até 042;<br><br>- Para transações digitadas e com tarja magnética, verificar as posições 017 até 036 do retorno da função PP_GetPin().|
|`PinPadInformation.TerminalId`|String|---|Sim|Número Lógico definido no Concentrador Cielo.|
|`PinPadInformation.SerialNumber`|String|---|Sim|Número de Série do Equipamento.|
|`PinPadInformation.PhysicalCharacteristics`|String|---|Sim|Enum: `WithoutPinPad` `PinPadWithoutChipReader` `PinPadWithChipReaderWithoutSamModule` `PinPadWithChipReaderWithSamModule` `NotCertifiedPinPad` `PinPadWithChipReaderWithoutSamAndContactless` `PinPadWithChipReaderWithSamModuleAndContactless` <br><br> Sem PIN-pad = `WithoutPinPad`; <br><br> PIN-pad sem leitor de Chip = `PinpadWithoutChipReader`; <br><br>PIN-pad com leitor de Chip sem módulo SAM = `PinPadWithChipReaderWithoutSamModule`; <br><br> PIN-pad com leitor de Chip com módulo SAM = `PinPadWithChipReaderWithSamModule`; <br><br> PIN-pad não homologado = `NotCertifiedPinPad`; <br><br> PIN-pad com leitor de Chip sem SAM e Cartão Sem Contato = `PinpadWithChipReaderWithoutSamAndContactless`; <br><br> PIN-pad com leitor de Chip com SAM e Cartão Sem Contato = `PinpadWithChipReaderWithSamAndContactless`. <br><br><br> Obs. Caso a aplicação não consiga informar os dados acima, deve obter tais informações através do retorno da função PP_GetInfo() da BC.|
|`PinPadInformation.ReturnDataInfo`|String|---|Sim|Retorno da função PP_GetInfo() da biblioteca compartilhada|

#### Resposta

```json
{
  "MerchantOrderId": "1593196632858",
  "Customer": {
    "Name": "[Guest]"
  },
  "Payment": {
    "Installments": 1,
    "Interest": "ByMerchant",
    "Capture": true,
    "CreditCard": {
      "ExpirationDate": "12/2021",
      "BrandId": 1,
      "IssuerId": 401,
      "TruncateCardNumberWhenPrinting": true,
      "PanSequenceNumber": 1,
      "InputMode": "Emv",
      "AuthenticationMethod": "OnlineAuthentication",
      "TrackOneData": "B3764 361234 56006^NOME NOME NOME NOME NOME N^0905060640431",
      "TrackTwoData": "1111222233334444=09050606404312376450",
      "EmvData": "",
      "IsFallback": false,
      "PinBlock": {
        "EncryptedPinBlock": "2280F6BDFD0C038D",
        "EncryptionType": "Dukpt3Des",
        "KsnIdentification": "fffff9999900522000d6"
      },
      "BrandInformation": {
        "Type": "aq?t?{zgk",
        "Name": "f?b}wbrxa}",
        "Description": "zmhm|mhpxh?s{mphntbv|~?wvr~tp"
      },
      "SaveCard": false
    },
    "Amount": 300,
    "ReceivedDate": "2020-06-26T18:37:12Z",
    "CapturedAmount": 300,
    "CapturedDate": "2020-06-26T18:37:13Z",
    "Provider": "Cielo",
    "Status": 2,
    "IsSplitted": false,
    "ReturnMessage": "APROVADA 075551",
    "ReturnCode": "000",
    "PaymentId": "ef7530c9-ace3-4143-a7d2-c80ff70c9e93",
    "Type": "PhysicalCreditCard",
    "Currency": "BRL",
    "Country": "BRA",
    "Links": [
      {
        "Method": "GET",
        "Rel": "self",
        "Href": "https://apiquerysandbox.cieloecommerce.cielo.com.br/1/physicalSales/ef7530c9-ace3-4143-a7d2-c80ff70c9e93"
      },
      {
        "Method": "PUT",
        "Rel": "confirm",
        "Href": "https://apisandbox.cieloecommerce.cielo.com.br/1/physicalSales/ef7530c9-ace3-4143-a7d2-c80ff70c9e93/confirmation"
      },
      {
        "Method": "DELETE",
        "Rel": "reverse",
        "Href": "https://apisandbox.cieloecommerce.cielo.com.br/1/physicalSales/ef7530c9-ace3-4143-a7d2-c80ff70c9e93"
      }
    ],
    "PaymentDateTime": "2020-06-26T18:37:12.858Z",
    "ServiceTaxAmount": 0,
    "SoftDescriptor": "Description",
    "ProductId": 1,
    "PinPadInformation": {
      "TerminalId": "42004558",
      "SerialNumber": "6C651996",
      "PhysicalCharacteristics": "PinPadWithChipReaderWithSamModuleAndContactless",
      "ReturnDataInfo": "00"
    },
    "PrintMessage": [
      {
        "Position": "Top",
        "Message": "yglxgftc{w"
      },
      {
        "Position": "Middle",
        "Message": "aji~?n?ckp"
      },
      {
        "Position": "Bottom",
        "Message": "uyai?|?{s?"
      }
    ],
    "ReceiptInformation": [
      {
        "Field": "MERCHANT_NAME",
        "Label": "NOME DO ESTABELECIMENTO",
        "Content": "ipwa~jpkp"
      },
      {
        "Field": "MERCHANT_CITY",
        "Label": "CIDADE DO ESTABELECIMENTO",
        "Content": "odw{piu~?x{"
      },
      {
        "Field": "INPUT_METHOD",
        "Label": "MODO DE ENTRADA",
        "Content": "oq?p{nket|"
      },
      {
        "Field": "TERMINAL",
        "Label": "POS",
        "Content": "56712930"
      },
      {
        "Field": "ISSUER_NAME",
        "Label": "EMISSOR",
        "Content": "j~qaw"
      },
      {
        "Field": "NSU",
        "Label": "DOC",
        "Content": "415858"
      },
      {
        "Field": "MERCHANT_CODE",
        "Label": "COD.ESTAB.",
        "Content": "13112148245096"
      },
      {
        "Field": "MERCHANT_ADDRESS",
        "Label": "ENDEREÇO DO ESTABELECIMENTO",
        "Content": "wcjh~qf?gzjjv?"
      },
      {
        "Field": "AUTHORIZATION_CODE",
        "Label": "AUTORIZAÇÃO",
        "Content": "41101"
      },
      {
        "Field": "CARD_HOLDER",
        "Label": "NOME DO CLIENTE",
        "Content": "obl}oh|q|b?xr??"
      },
      {
        "Field": "TRANSACTION_TYPE",
        "Label": "TIPO DE TRANSAÇÃO",
        "Content": "gtfph?bwh?"
      },
      {
        "Field": "MERCHANT_STATE",
        "Label": "ESTADO DO ESTABELECIMENTO",
        "Content": "rc"
      },
      {
        "Field": "DATE",
        "Label": "DATA",
        "Content": "6/26/2020"
      },
      {
        "Field": "HOUR",
        "Label": "HORA",
        "Content": "3:37 PM"
      },
      {
        "Field": "VALUE",
        "Label": "VALOR",
        "Content": "300"
      },
      {
        "Field": "TRANSACTION_MODE",
        "Label": "MODO DA TRANSAÇÃO",
        "Content": "dug"
      },
      {
        "Field": "CARD_NUMBER",
        "Label": "CARTÃO"
      }
    ],
    "Receipt": {
      "MerchantName": "ipwa~jpkpg~ee?h",
      "MerchantCity": "odw{piu~?x{wcd{xu?kr",
      "InputMethod": "oq?p{nket|skqp{v?dyfyvlmekabfr~",
      "Terminal": "56712930",
      "IssuerName": "j~qaw{ydyslbiv{e{a?caw??lgcqs?el",
      "Nsu": "415858",
      "MerchantCode": "13112148245096",
      "MerchantAddress": "wcjh~qf?gzjjv?sx{fr~a?|nld?{?i?zwjclx|pbq?n???z~o~xl??zso??~uwleqldoc|ubnp",
      "AuthorizationCode": "41101",
      "CardHolder": "obl}oh|q|b?xr?uoiuzjr",
      "TransactionType": "gtfph?bwh?hkd|k??ypva?}?vi?gq",
      "MerchantState": "rc",
      "Date": "6/26/2020",
      "Hour": "3:37 PM",
      "Value": "300",
      "TransactionMode": "dug",
      "CardNumber": null
    },
    "AuthorizationCode": "075551",
    "ProofOfSale": "441836",
    "InitializationVersion": 1593196200000,
    "ConfirmationStatus": 0,
    "EmvResponseData": "956648280",
    "SubordinatedMerchantId": "b99a463f-88db-442a-b5fa-982187b68f5c",
    "OfflinePaymentType": "OfflineWithoutAttemptOnline"
  }
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`MerchantOrderId`|String|---|---| Número do documento gerado automaticamente pelo terminal e incrementado de 1 a cada transação realizada no terminal. Aceita apenas valores numéricos de 1 a 15 dígitos.|
|`Customer.Name`|String|---|---|---|
|`Customer.Identity`|---|---|---|---|
|`Customer.IdentityType`|---|---|---|---|
|`Customer.Email`|---|---|---|---|
|`Customer.Birthday`|---|---|---|---|
|`Address.Street`|String|---|---|---|
|`Address.Number`|String|---|---|---|
|`Address.Complement`|String|---|---||---|
|`Address.ZipCode`|String|---|---|---|
|`Address.City`|String|---|---|---|
|`Address.State`|String|---|---|---|
|`Address.Country`|String|---|---|---|
|`DeliveryAddress.Street`|String|---|---|---|
|`DeliveryAddress.Number`|String|---|---|---|
|`DeliveryAddress.Complement`|String|---|---||---|
|`DeliveryAddress.ZipCode`|String|---|---|---|
|`DeliveryAddress.City`|String|---|---|---|
|`DeliveryAddress.State`|String|---|---|---|
|`DeliveryAddress.Country`|String|---|---|---|
|`Payment.Installments`|Integer|---|---|Default: 1 / Quantidade de Parcelas: Varia de 2 a 99 para transação de financiamento. Deve ser verificado os atributos maxOfPayments1, maxOfPayments2, maxOfPayments3 e minValOfPayments da tabela productTable.|
|`Payment.Interest`|String|---|---|Default: `ByMerchant` <br><br> Enum: `ByMerchant` `ByIssuer` <br><br> Tipo de Parcelamento: <br><br> - Se o bit 6 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 6 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento sem juros pode ser efetuado. <br><br> - Se o bit 7 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 7 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento com juros pode ser efetuado. Sem juros = “ByMerchant”; Com juros = “ByIssuer”.|
|`Payment.Capture`|Booleano|---|---|Default: false / Booleano que identifica que a autorização deve ser com captura automática. A autorização sem captura automática é conhecida também como pré-autorização.|
|`CreditCard.ExpirationDate`|String|MM/yyyy|Sim|Data de validade do cartão. <br><br>Dado obtido através do comando PP_GetCard na BC no momento da captura da transação.|
|`CreditCard.BrandId`|Integer|---|Sim|Identificação da bandeira obtida através do campo BrandId da PRODUCT TABLE.|
|`CreditCard.IssuerId`|Integer|---|Sim|Código do emissor obtido através do campo IssuerId da BIN TABLE.|
|`CreditCard.TruncateCardNumberWhenPrinting`|Booleano|---|---|Indica se o número do cartão será truncado no momento da impressão do comprovante. A solução de captura deve tomar essa decisão com base no `confParamOp03` presente nas tabelas BIN TABLE, PARAMETER TABLE e ISSUER TABLE.|
|`CreditCard.InputMode`|String|---|Sim|Enum: `Typed` `MagStripe` `Emv` <br><br>Identificação do modo de captura do cartão na transação. Essa informação deve ser obtida através do retorno da função PP_GetCard da BC. <br><br>"00" – Magnético <br><br>"01" - Moedeiro VISA Cash sobre TIBC v1 <br><br>"02" - Moedeiro VISA Cash sobre TIBC v3 <br><br>"03" – EMV com contato <br><br>"04" - Easy-Entry sobre TIBC v1 <br><br>"05" - Chip sem contato simulando tarja <br><br> "06" - EMV sem contato.|
|`CreditCard.AuthenticationMethod`|String|---|Sim|Enum: `NoPassword` `OnlineAuthentication` `OfflineAuthentication` <br><br>Método de autenticação <br><br>- Se o cartão foi lido a partir da digitação verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, a senha deve ser capturada e o authenticationMethod assume valor 2. Caso contrário, assume valor 1; <br><br>- Se o cartão foi lido a partir da trilha verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, deve ser verificado o bit 2 do mesmo campo. Se este estiver com valor 1 deve ser capturada a senha. Se estiver com valor 0 a captura da senha vai depender do último dígito do service code; <br><br>- Se o cartão foi lido através do chip EMV, o authenticationMethod será preenchido com base no retorno da função PP_GoOnChip(). No resultado PP_GoOnChip(), onde se o campo da posição 003 do retorno da PP_GoOnChip() estiver com valor 1 indica que o pin foi validado off-line, o authenticationMethod será 3. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valor 0, o authenticationMethod será 1. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valores 0 e 1 respectivamente, o authenticationMethod será 2. <br><br>1 - Sem senha = “NoPassword”; <br><br>2 - Senha online = “Online Authentication”; <br><br>3 - Senha off-line = “Offline Authentication”.|
|`CreditCard.EmvData`|String|---|---|Dados da transação EMV <br><br>Obtidos através do comando PP_GoOnChip na BC|
|`PinBlock.EncryptedPinBlock`|---|---|---|---|
|`PinBlock.EncryptionType`|String|---|---|---|
|`PinBlock.KsnIdentification`|String|---|---|---|
|`CreditCard.PanSequenceNumber`|Number|---|---|Número sequencial do cartão, utilizado para identificar a conta corrente do cartão adicional. Mandatório para transações com cartões Chip EMV e que possuam PAN Sequence Number (Tag 5F34).|
|`CreditCard.SaveCard`|---|---|---|---|
|`CreditCard.IsFallback`|---|---|---|---|
|`Payment.PaymentDateTime`|String|date-time|Sim|Data e Hora da captura da transação|
|`Payment.ServiceTaxAmount`|---|---|---|---|
|`Payment.SoftDescriptor`|String|13|---|Identificação do estabelecimento (nome reduzido) a ser impresso e identificado na fatura.|
|`Payment.ProductId`|Integer|---|Sim|Código do produto identificado através do bin do cartão.|
|`PinPadInformation.TerminalId`|String|---|Sim|Número Lógico definido no Concentrador Cielo.|
|`PinPadInformation.SerialNumber`|String|---|Sim|Número de Série do Equipamento.|
|`PinPadInformation.PhysicalCharacteristics`|String|---|Sim|Enum: `WithoutPinPad` `PinPadWithoutChipReader` `PinPadWithChipReaderWithoutSamModule` `PinPadWithChipReaderWithSamModule` `NotCertifiedPinPad` `PinPadWithChipReaderWithoutSamAndContactless` `PinPadWithChipReaderWithSamModuleAndContactless` <br><br> Sem PIN-pad = `WithoutPinPad`; <br><br> PIN-pad sem leitor de Chip = `PinpadWithoutChipReader`; <br><br>PIN-pad com leitor de Chip sem módulo SAM = `PinPadWithChipReaderWithoutSamModule`; <br><br> PIN-pad com leitor de Chip com módulo SAM = `PinPadWithChipReaderWithSamModule`; <br><br> PIN-pad não homologado = `NotCertifiedPinPad`; <br><br> PIN-pad com leitor de Chip sem SAM e Cartão Sem Contato = `PinpadWithChipReaderWithoutSamAndContactless`; <br><br> PIN-pad com leitor de Chip com SAM e Cartão Sem Contato = `PinpadWithChipReaderWithSamAndContactless`. <br><br><br> Obs. Caso a aplicação não consiga informar os dados acima, deve obter tais informações através do retorno da função PP_GetInfo() da BC.|
|`PinPadInformation.ReturnDataInfo`|String|---|Sim|Retorno da função PP_GetInfo() da biblioteca compartilhada|
|`Payment.Amount`|Integer(int64)|---|Sim|Valor da transação (1079 = R$10,79)|
|`Payment.ReceivedDate`|---|---|---|---|
|`Payment.CapturedAmount`|---|---|---|---|
|`Payment.Provider`|String|---|---|---|
|`Payment.ConfirmationStatus`|---|---|---|---|
|`Payment.InitializationVersion`|---|---|---|---|
|`Payment.EmvResponseData`|---|---|---|---|
|`Payment.Status`|---|---|---|---|
|`Payment.IsSplitted`|Booleano|---|---|---|
|`Payment.ReturnCode`|---|---|---|---|
|`Payment.ReturnMessage`|String|---|---|---|
|`Payment.PaymentId`|---|---|---|---|
|`Payment.Type`|String|---|Sim|Value: `PhysicalCreditCard` / Tipo da Transação|
|`Payment.Currency`|String|---|---|Default: "BRL" / Value: "BRL" / Moeda (Preencher com “BRL”)|
|`Payment.Country`|String|---|---|Default: "BRA" / Value: "BRA" / País (Preencher com “BRA”)|
|`Receipt.MerchantName`|---|---|---|---|
|`Receipt.MerchantAddress`|---|---|---|---|
|`Receipt.MerchantCity`|---|---|---|---|
|`Receipt.MerchantState`|---|---|---|---|
|`Receipt.MerchantCode`|---|---|---|---|
|`Receipt.Terminal`|---|---|---|---|
|`Receipt.Nsu`|---|---|---|---|
|`Receipt.Date`|---|---|---|---|
|`Receipt.Hour`|---|---|---|---|
|`Receipt.IssuerName`|---|---|---|---|
|`Receipt.CardNumber`|---|---|---|---|
|`Receipt.TransactionType`|---|---|---|---|
|`Receipt.AuthorizationCode`|---|---|---|---|
|`Receipt.TransactionMode`|---|---|---|---|
|`Receipt.InputMethod`|---|---|---|---|
|`Receipt.Value`|---|---|---|---|
|`Receipt.SoftDescriptor`|---|---|---|---|
|`RecurrentPayment.RecurrentPaymentId`|---|---|---|---|
|`RecurrentPayment.ReasonCode`|---|---|---|---|
|`RecurrentPayment.ReasonMessage`|---|---|---|---|
|`RecurrentPayment.NextRecurrency`|---|---|---|---|
|`RecurrentPayment.EndDate`|---|---|---|---|
|`RecurrentPayment.Interval`|---|---|---|---|
|`SplitPayments.SubordinateMerchantId`|---|---|---|---|
|`SplitPayments.Amount`|---|---|---|---|
|`SplitPayments.Fares.Mdr`|---|---|---|---|
|`SplitPayments.Fares.Fee`|---|---|---|---|
|`SplitErrors.Code`|---|---|---|---|
|`SplitErrors.Message`|---|---|---|---|

### MassTransit - MTT

#### Requisição

<aside class="request"><span class="method post">POST</span> <span class="endpoint">/1/physicalSales/</span></aside>

```json
{
   "MerchantOrderId": "1596226820548",
   "Payment": {
      "Type": "PhysicalCreditCard",
      "SoftDescriptor": "Description",
      "PaymentDateTime": "2020-07-31T20:20:20.548Z",
      "Amount": 100,
      "Installments": 1,
      "Interest": "ByMerchant",
      "Capture": true,
      "ProductId": 1,
      "CreditCard": {
         "InputMode": "ContactlessEmv",
         "BrandId": 1,
         "IssuerId": 401,
         "TruncateCardNumberWhenPrinting": true,
         "ExpirationDate": "12/2020",
         "PanSequenceNumber": 1,
         "EmvSequenceNumber": 1,
         "EmvData": "112233445566778899011AABBC012D3456789E0123FF45678AB901234C5D112233445566778800",
         "TrackOneData": "A1234567890123456^FULANO OLIVEIRA SA ^12345678901234567890123",
         "TrackTwoData": "0123456789012345=012345678901234",
         "AuthenticationMethod": "OnlineAuthentication",
         "PinBlock": {
            "EncryptedPinBlock": "2280F6BDFD0C038D",
            "EncryptionType": "Dukpt3Des",
            "KsnIdentification": "fffff9999900522000d6"
         }
      },
      "PinPadInformation": {
         "TerminalId": "12345678",
         "SerialNumber": "6C651996",
         "PhysicalCharacteristics": "PinPadWithChipReaderWithSamModuleAndContactless",
         "ReturnDataInfo": "00"
      },
      "MassTransit": {
         "IsDebtRecovery": false,
         "FirstTravelDate": "2018-01-01 04:12"
      }
   }
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`MerchantOrderId`|String|---|---| Número do documento gerado automaticamente pelo terminal e incrementado de 1 a cada transação realizada no terminal. Aceita apenas valores numéricos de 1 a 15 dígitos.|
|`Payment.Type`|String|---|Sim|Value: `PhysicalCreditCard` / Tipo da Transação|
|`Payment.SoftDescriptor`|String|13|---|Identificação do estabelecimento (nome reduzido) a ser impresso e identificado na fatura.|
|`Payment.PaymentDateTime`|String|date-time|Sim|Data e Hora da captura da transação|
|`Payment.Amount`|Integer(int64)|---|Sim|Valor da transação (1079 = R$10,79)|
|`Payment.Installments`|Integer|---|---|Default: 1 / Quantidade de Parcelas: Varia de 2 a 99 para transação de financiamento. Deve ser verificado os atributos maxOfPayments1, maxOfPayments2, maxOfPayments3 e minValOfPayments da tabela productTable.|
|`Payment.Interest`|String|---|---|Default: `ByMerchant` <br><br> Enum: `ByMerchant` `ByIssuer` <br><br> Tipo de Parcelamento: <br><br> - Se o bit 6 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 6 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento sem juros pode ser efetuado. <br><br> - Se o bit 7 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 7 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento com juros pode ser efetuado. Sem juros = “ByMerchant”; Com juros = “ByIssuer”.|
|`Payment.Capture`|Booleano|---|---|Default: false / Booleano que identifica que a autorização deve ser com captura automática. A autorização sem captura automática é conhecida também como pré-autorização.|
|`Payment.ProductId`|Integer|---|Sim|Código do produto identificado através do bin do cartão.|
|`CreditCard.ExpirationDate`|String|MM/yyyy|Sim|Data de validade do cartão. <br><br>Dado obtido através do comando PP_GetCard na BC no momento da captura da transação.|
|`CreditCard.BrandId`|Integer|---|Sim|Identificação da bandeira obtida através do campo BrandId da PRODUCT TABLE.|
|`CreditCard.IssuerId`|Integer|---|Sim|Código do emissor obtido através do campo IssuerId da BIN TABLE.|
|`CreditCard.InputMode`|String|---|Sim|Enum: `Typed` `MagStripe` `Emv` <br><br>Identificação do modo de captura do cartão na transação. Essa informação deve ser obtida através do retorno da função PP_GetCard da BC. <br><br>"00" – Magnético <br><br>"01" - Moedeiro VISA Cash sobre TIBC v1 <br><br>"02" - Moedeiro VISA Cash sobre TIBC v3 <br><br>"03" – EMV com contato <br><br>"04" - Easy-Entry sobre TIBC v1 <br><br>"05" - Chip sem contato simulando tarja <br><br> "06" - EMV sem contato.|
|`CreditCard.AuthenticationMethod`|String|---|Sim|Enum: `NoPassword` `OnlineAuthentication` `OfflineAuthentication` <br><br>Método de autenticação <br><br>- Se o cartão foi lido a partir da digitação verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, a senha deve ser capturada e o authenticationMethod assume valor 2. Caso contrário, assume valor 1; <br><br>- Se o cartão foi lido a partir da trilha verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, deve ser verificado o bit 2 do mesmo campo. Se este estiver com valor 1 deve ser capturada a senha. Se estiver com valor 0 a captura da senha vai depender do último dígito do service code; <br><br>- Se o cartão foi lido através do chip EMV, o authenticationMethod será preenchido com base no retorno da função PP_GoOnChip(). No resultado PP_GoOnChip(), onde se o campo da posição 003 do retorno da PP_GoOnChip() estiver com valor 1 indica que o pin foi validado off-line, o authenticationMethod será 3. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valor 0, o authenticationMethod será 1. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valores 0 e 1 respectivamente, o authenticationMethod será 2. <br><br>1 - Sem senha = “NoPassword”; <br><br>2 - Senha online = “Online Authentication”; <br><br>3 - Senha off-line = “Offline Authentication”.|
|`CreditCard.EmvData`|String|---|---|Dados da transação EMV<br><br>Obtidos através do comando PP_GoOnChip na BC|
|`PinBlock.EncryptedPinBlock`|String|---|Sim|PINBlock Criptografado<br><br>- Para transações EMV, esse campo é obtido através do retorno da função PP_GoOnChip(), mais especificamente das posições 007 até a posição 022;<br><br>- Para transações digitadas e com tarja magnética, verificar as posições 001 até 016 do retorno da função PP_GetPin().|
|`PinBlock.EncryptionType`|String|---|Sim|Tipo de Criptografia<br><br>Enum:<br><br>"DukptDes"<br><br>"Dukpt3Des"<br><br>"MasterKey"|
|`PinBlock.KsnIdentification`|String|---|Sim|Identificação do KSN<br><br>- Para transações EMV esse campo é obtido através do retorno da função PP_GoOnChip() nas posições 023 até 042;<br><br>- Para transações digitadas e com tarja magnética, verificar as posições 017 até 036 do retorno da função PP_GetPin().|
|`CreditCard.PanSequenceNumber`|Number|---|---|Número sequencial do cartão, utilizado para identificar a conta corrente do cartão adicional. Mandatório para transações com cartões Chip EMV e que possuam PAN Sequence Number (Tag 5F34).|
|`CreditCard.SaveCard`|Booleano|---|---|Identifica se vai salvar/tokenizar o cartão.|
|`CreditCard.IsFallback`|Booleano|---|---|Identifica se é uma transação de fallback.|
|`PinPadInformation.TerminalId`|String|---|Sim|Número Lógico definido no Concentrador Cielo.|
|`PinPadInformation.SerialNumber`|String|---|Sim|Número de Série do Equipamento.|
|`PinPadInformation.PhysicalCharacteristics`|String|---|Sim|Enum: `WithoutPinPad` `PinPadWithoutChipReader` `PinPadWithChipReaderWithoutSamModule` `PinPadWithChipReaderWithSamModule` `NotCertifiedPinPad` `PinPadWithChipReaderWithoutSamAndContactless` `PinPadWithChipReaderWithSamModuleAndContactless` <br><br> Sem PIN-pad = `WithoutPinPad`; <br><br> PIN-pad sem leitor de Chip = `PinpadWithoutChipReader`; <br><br>PIN-pad com leitor de Chip sem módulo SAM = `PinPadWithChipReaderWithoutSamModule`; <br><br> PIN-pad com leitor de Chip com módulo SAM = `PinPadWithChipReaderWithSamModule`; <br><br> PIN-pad não homologado = `NotCertifiedPinPad`; <br><br> PIN-pad com leitor de Chip sem SAM e Cartão Sem Contato = `PinpadWithChipReaderWithoutSamAndContactless`; <br><br> PIN-pad com leitor de Chip com SAM e Cartão Sem Contato = `PinpadWithChipReaderWithSamAndContactless`. <br><br><br> Obs. Caso a aplicação não consiga informar os dados acima, deve obter tais informações através do retorno da função PP_GetInfo() da BC.|
|`PinPadInformation.ReturnDataInfo`|String|---|Sim|Retorno da função PP_GetInfo() da biblioteca compartilhada|
|`MassTransit.IsDebtRecovery`|Booleano|—|Sim|Identifica de a operação é de recuperação de pagamento.|
|`MassTransit.FirstTravelDate`|String|date-time|Sim|Data da primeira viagem|

#### Resposta

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`MerchantOrderId`|String|---|---| Número do documento gerado automaticamente pelo terminal e incrementado de 1 a cada transação realizada no terminal. Aceita apenas valores numéricos de 1 a 15 dígitos.|
|`Customer.Name`|String|---|---|---|
|`Customer.Identity`|---|---|---|---|
|`Customer.IdentityType`|---|---|---|---|
|`Customer.Email`|---|---|---|---|
|`Customer.Birthday`|---|---|---|---|
|`Address.Street`|String|---|---|---|
|`Address.Number`|String|---|---|---|
|`Address.Complement`|String|---|---||---|
|`Address.ZipCode`|String|---|---|---|
|`Address.City`|String|---|---|---|
|`Address.State`|String|---|---|---|
|`Address.Country`|String|---|---|---|
|`DeliveryAddress.Street`|String|---|---|---|
|`DeliveryAddress.Number`|String|---|---|---|
|`DeliveryAddress.Complement`|String|---|---||---|
|`DeliveryAddress.ZipCode`|String|---|---|---|
|`DeliveryAddress.City`|String|---|---|---|
|`DeliveryAddress.State`|String|---|---|---|
|`DeliveryAddress.Country`|String|---|---|---|
|`Payment.Installments`|Integer|---|---|Default: 1 / Quantidade de Parcelas: Varia de 2 a 99 para transação de financiamento. Deve ser verificado os atributos maxOfPayments1, maxOfPayments2, maxOfPayments3 e minValOfPayments da tabela productTable.|
|`Payment.Interest`|String|---|---|Default: `ByMerchant` <br><br> Enum: `ByMerchant` `ByIssuer` <br><br> Tipo de Parcelamento: <br><br> - Se o bit 6 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 6 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento sem juros pode ser efetuado. <br><br> - Se o bit 7 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 7 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento com juros pode ser efetuado. Sem juros = “ByMerchant”; Com juros = “ByIssuer”.|
|`Payment.Capture`|Booleano|---|---|Default: false / Booleano que identifica que a autorização deve ser com captura automática. A autorização sem captura automática é conhecida também como pré-autorização.|
|`CreditCard.ExpirationDate`|String|MM/yyyy|Sim|Data de validade do cartão. <br><br>Dado obtido através do comando PP_GetCard na BC no momento da captura da transação.|
|`CreditCard.BrandId`|Integer|---|Sim|Identificação da bandeira obtida através do campo BrandId da PRODUCT TABLE.|
|`CreditCard.IssuerId`|Integer|---|Sim|Código do emissor obtido através do campo IssuerId da BIN TABLE.|
|`CreditCard.TruncateCardNumberWhenPrinting`|Booleano|---|---|Indica se o número do cartão será truncado no momento da impressão do comprovante. A solução de captura deve tomar essa decisão com base no `confParamOp03` presente nas tabelas BIN TABLE, PARAMETER TABLE e ISSUER TABLE.|
|`CreditCard.InputMode`|String|---|Sim|Enum: `Typed` `MagStripe` `Emv` <br><br>Identificação do modo de captura do cartão na transação. Essa informação deve ser obtida através do retorno da função PP_GetCard da BC. <br><br>"00" – Magnético <br><br>"01" - Moedeiro VISA Cash sobre TIBC v1 <br><br>"02" - Moedeiro VISA Cash sobre TIBC v3 <br><br>"03" – EMV com contato <br><br>"04" - Easy-Entry sobre TIBC v1 <br><br>"05" - Chip sem contato simulando tarja <br><br> "06" - EMV sem contato.|
|`CreditCard.AuthenticationMethod`|String|---|Sim|Enum: `NoPassword` `OnlineAuthentication` `OfflineAuthentication` <br><br>Método de autenticação <br><br>- Se o cartão foi lido a partir da digitação verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, a senha deve ser capturada e o authenticationMethod assume valor 2. Caso contrário, assume valor 1; <br><br>- Se o cartão foi lido a partir da trilha verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, deve ser verificado o bit 2 do mesmo campo. Se este estiver com valor 1 deve ser capturada a senha. Se estiver com valor 0 a captura da senha vai depender do último dígito do service code; <br><br>- Se o cartão foi lido através do chip EMV, o authenticationMethod será preenchido com base no retorno da função PP_GoOnChip(). No resultado PP_GoOnChip(), onde se o campo da posição 003 do retorno da PP_GoOnChip() estiver com valor 1 indica que o pin foi validado off-line, o authenticationMethod será 3. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valor 0, o authenticationMethod será 1. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valores 0 e 1 respectivamente, o authenticationMethod será 2. <br><br>1 - Sem senha = “NoPassword”; <br><br>2 - Senha online = “Online Authentication”; <br><br>3 - Senha off-line = “Offline Authentication”.|
|`CreditCard.EmvData`|String|---|---|Dados da transação EMV <br><br>Obtidos através do comando PP_GoOnChip na BC|
|`PinBlock.EncryptedPinBlock`|---|---|---|---|
|`PinBlock.EncryptionType`|String|---|---|---|
|`PinBlock.KsnIdentification`|String|---|---|---|
|`CreditCard.PanSequenceNumber`|Number|---|---|Número sequencial do cartão, utilizado para identificar a conta corrente do cartão adicional. Mandatório para transações com cartões Chip EMV e que possuam PAN Sequence Number (Tag 5F34).|
|`CreditCard.SaveCard`|---|---|---|---|
|`CreditCard.IsFallback`|---|---|---|---|
|`Payment.PaymentDateTime`|String|date-time|Sim|Data e Hora da captura da transação|
|`Payment.ServiceTaxAmount`|---|---|---|---|
|`Payment.SoftDescriptor`|String|13|---|Identificação do estabelecimento (nome reduzido) a ser impresso e identificado na fatura.|
|`Payment.ProductId`|Integer|---|Sim|Código do produto identificado através do bin do cartão.|
|`PinPadInformation.TerminalId`|String|---|Sim|Número Lógico definido no Concentrador Cielo.|
|`PinPadInformation.SerialNumber`|String|---|Sim|Número de Série do Equipamento.|
|`PinPadInformation.PhysicalCharacteristics`|String|---|Sim|Enum: `WithoutPinPad` `PinPadWithoutChipReader` `PinPadWithChipReaderWithoutSamModule` `PinPadWithChipReaderWithSamModule` `NotCertifiedPinPad` `PinPadWithChipReaderWithoutSamAndContactless` `PinPadWithChipReaderWithSamModuleAndContactless` <br><br> Sem PIN-pad = `WithoutPinPad`; <br><br> PIN-pad sem leitor de Chip = `PinpadWithoutChipReader`; <br><br>PIN-pad com leitor de Chip sem módulo SAM = `PinPadWithChipReaderWithoutSamModule`; <br><br> PIN-pad com leitor de Chip com módulo SAM = `PinPadWithChipReaderWithSamModule`; <br><br> PIN-pad não homologado = `NotCertifiedPinPad`; <br><br> PIN-pad com leitor de Chip sem SAM e Cartão Sem Contato = `PinpadWithChipReaderWithoutSamAndContactless`; <br><br> PIN-pad com leitor de Chip com SAM e Cartão Sem Contato = `PinpadWithChipReaderWithSamAndContactless`. <br><br><br> Obs. Caso a aplicação não consiga informar os dados acima, deve obter tais informações através do retorno da função PP_GetInfo() da BC.|
|`PinPadInformation.ReturnDataInfo`|String|---|Sim|Retorno da função PP_GetInfo() da biblioteca compartilhada|
|`Payment.Amount`|Integer(int64)|---|Sim|Valor da transação (1079 = R$10,79)|
|`Payment.ReceivedDate`|---|---|---|---|
|`Payment.CapturedAmount`|---|---|---|---|
|`Payment.Provider`|String|---|---|---|
|`Payment.ConfirmationStatus`|---|---|---|---|
|`Payment.InitializationVersion`|---|---|---|---|
|`Payment.EmvResponseData`|---|---|---|---|
|`Payment.Status`|---|---|---|---|
|`Payment.IsSplitted`|Booleano|---|---|---|
|`Payment.ReturnCode`|---|---|---|---|
|`Payment.ReturnMessage`|String|---|---|---|
|`Payment.PaymentId`|---|---|---|---|
|`Payment.Type`|String|---|Sim|Value: `PhysicalCreditCard` / Tipo da Transação|
|`Payment.Currency`|String|---|---|Default: "BRL" / Value: "BRL" / Moeda (Preencher com “BRL”)|
|`Payment.Country`|String|---|---|Default: "BRA" / Value: "BRA" / País (Preencher com “BRA”)|
|`Receipt.MerchantName`|---|---|---|---|
|`Receipt.MerchantAddress`|---|---|---|---|
|`Receipt.MerchantCity`|---|---|---|---|
|`Receipt.MerchantState`|---|---|---|---|
|`Receipt.MerchantCode`|---|---|---|---|
|`Receipt.Terminal`|---|---|---|---|
|`Receipt.Nsu`|---|---|---|---|
|`Receipt.Date`|---|---|---|---|
|`Receipt.Hour`|---|---|---|---|
|`Receipt.IssuerName`|---|---|---|---|
|`Receipt.CardNumber`|---|---|---|---|
|`Receipt.TransactionType`|---|---|---|---|
|`Receipt.AuthorizationCode`|---|---|---|---|
|`Receipt.TransactionMode`|---|---|---|---|
|`Receipt.InputMethod`|---|---|---|---|
|`Receipt.Value`|---|---|---|---|
|`Receipt.SoftDescriptor`|---|---|---|---|
|`RecurrentPayment.RecurrentPaymentId`|---|---|---|---|
|`RecurrentPayment.ReasonCode`|---|---|---|---|
|`RecurrentPayment.ReasonMessage`|---|---|---|---|
|`RecurrentPayment.NextRecurrency`|---|---|---|---|
|`RecurrentPayment.EndDate`|---|---|---|---|
|`RecurrentPayment.Interval`|---|---|---|---|
|`SplitPayments.SubordinateMerchantId`|---|---|---|---|
|`SplitPayments.Amount`|---|---|---|---|
|`SplitPayments.Fares.Mdr`|---|---|---|---|
|`SplitPayments.Fares.Fee`|---|---|---|---|
|`SplitErrors.Code`|---|---|---|---|
|`SplitErrors.Message`|---|---|---|---|

### MassTransit - KFT

#### Requisição

<aside class="request"><span class="method post">POST</span> <span class="endpoint">/1/physicalSales/</span></aside>

```json
{
   "MerchantOrderId": "1596226820548",
   "Payment": {
      "Type": "PhysicalCreditCard",
      "SoftDescriptor": "Description",
      "PaymentDateTime": "2020-07-31T20:20:20.548Z",
      "Amount": 300,
      "Installments": 1,
      "Interest": "ByMerchant",
      "Capture": true,
      "ProductId": 1,
      "CreditCard": {
         "InputMode": "ContactlessEmv",
         "BrandId": 1,
         "IssuerId": 401,
         "TruncateCardNumberWhenPrinting": true,
         "ExpirationDate": "12/2020",
         "EmvSequenceNumber": 1,
         "EmvData": "112233445566778899011AABBC012D3456789E0123FF45678AB901234C5D112233445566778800",
         "TrackOneData": "A1234567890123456^FULANO OLIVEIRA SA ^12345678901234567890123",
         "TrackTwoData": "0123456789012345=012345678901234",
         "AuthenticationMethod": "OnlineAuthentication",
         "PinBlock": {
            "EncryptedPinBlock": "2280F6BDFD0C038D",
            "EncryptionType": "Dukpt3Des",
            "KsnIdentification": "fffff9999900522000d6"
         }
      },
      "PinPadInformation": {
         "TerminalId": "12345678",
         "SerialNumber": "6C651996",
         "PhysicalCharacteristics": "PinPadWithChipReaderWithSamModuleAndContactless",
         "ReturnDataInfo": "00"
      },
      "MassTransit": {
         "IsDebtRecovery": false,
         "IsKnownValue": true,
         "FirstTravelDate": "2018-01-01 04:12"
      }
   }
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`MerchantOrderId`|String|---|---| Número do documento gerado automaticamente pelo terminal e incrementado de 1 a cada transação realizada no terminal. Aceita apenas valores numéricos de 1 a 15 dígitos.|
|`Payment.Type`|String|---|Sim|Value: `PhysicalCreditCard` / Tipo da Transação|
|`Payment.SoftDescriptor`|String|13|---|Identificação do estabelecimento (nome reduzido) a ser impresso e identificado na fatura.|
|`Payment.PaymentDateTime`|String|date-time|Sim|Data e Hora da captura da transação|
|`Payment.Amount`|Integer(int64)|---|Sim|Valor da transação (1079 = R$10,79)|
|`Payment.Installments`|Integer|---|---|Default: 1 / Quantidade de Parcelas: Varia de 2 a 99 para transação de financiamento. Deve ser verificado os atributos maxOfPayments1, maxOfPayments2, maxOfPayments3 e minValOfPayments da tabela productTable.|
|`Payment.Interest`|String|---|---|Default: `ByMerchant` <br><br> Enum: `ByMerchant` `ByIssuer` <br><br> Tipo de Parcelamento: <br><br> - Se o bit 6 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 6 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento sem juros pode ser efetuado. <br><br> - Se o bit 7 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 7 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento com juros pode ser efetuado. Sem juros = “ByMerchant”; Com juros = “ByIssuer”.|
|`Payment.Capture`|Booleano|---|---|Default: false / Booleano que identifica que a autorização deve ser com captura automática. A autorização sem captura automática é conhecida também como pré-autorização.|
|`Payment.ProductId`|Integer|---|Sim|Código do produto identificado através do bin do cartão.|
|`CreditCard.ExpirationDate`|String|MM/yyyy|Sim|Data de validade do cartão. <br><br>Dado obtido através do comando PP_GetCard na BC no momento da captura da transação.|
|`CreditCard.BrandId`|Integer|---|Sim|Identificação da bandeira obtida através do campo BrandId da PRODUCT TABLE.|
|`CreditCard.IssuerId`|Integer|---|Sim|Código do emissor obtido através do campo IssuerId da BIN TABLE.|
|`CreditCard.InputMode`|String|---|Sim|Enum: `Typed` `MagStripe` `Emv` <br><br>Identificação do modo de captura do cartão na transação. Essa informação deve ser obtida através do retorno da função PP_GetCard da BC. <br><br>"00" – Magnético <br><br>"01" - Moedeiro VISA Cash sobre TIBC v1 <br><br>"02" - Moedeiro VISA Cash sobre TIBC v3 <br><br>"03" – EMV com contato <br><br>"04" - Easy-Entry sobre TIBC v1 <br><br>"05" - Chip sem contato simulando tarja <br><br> "06" - EMV sem contato.|
|`CreditCard.AuthenticationMethod`|String|---|Sim|Enum: `NoPassword` `OnlineAuthentication` `OfflineAuthentication` <br><br>Método de autenticação <br><br>- Se o cartão foi lido a partir da digitação verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, a senha deve ser capturada e o authenticationMethod assume valor 2. Caso contrário, assume valor 1; <br><br>- Se o cartão foi lido a partir da trilha verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, deve ser verificado o bit 2 do mesmo campo. Se este estiver com valor 1 deve ser capturada a senha. Se estiver com valor 0 a captura da senha vai depender do último dígito do service code; <br><br>- Se o cartão foi lido através do chip EMV, o authenticationMethod será preenchido com base no retorno da função PP_GoOnChip(). No resultado PP_GoOnChip(), onde se o campo da posição 003 do retorno da PP_GoOnChip() estiver com valor 1 indica que o pin foi validado off-line, o authenticationMethod será 3. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valor 0, o authenticationMethod será 1. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valores 0 e 1 respectivamente, o authenticationMethod será 2. <br><br>1 - Sem senha = “NoPassword”; <br><br>2 - Senha online = “Online Authentication”; <br><br>3 - Senha off-line = “Offline Authentication”.|
|`CreditCard.EmvData`|String|---|---|Dados da transação EMV<br><br>Obtidos através do comando PP_GoOnChip na BC|
|`PinBlock.EncryptedPinBlock`|String|---|Sim|PINBlock Criptografado<br><br>- Para transações EMV, esse campo é obtido através do retorno da função PP_GoOnChip(), mais especificamente das posições 007 até a posição 022;<br><br>- Para transações digitadas e com tarja magnética, verificar as posições 001 até 016 do retorno da função PP_GetPin().|
|`PinBlock.EncryptionType`|String|---|Sim|Tipo de Criptografia<br><br>Enum:<br><br>"DukptDes"<br><br>"Dukpt3Des"<br><br>"MasterKey"|
|`PinBlock.KsnIdentification`|String|---|Sim|Identificação do KSN<br><br>- Para transações EMV esse campo é obtido através do retorno da função PP_GoOnChip() nas posições 023 até 042;<br><br>- Para transações digitadas e com tarja magnética, verificar as posições 017 até 036 do retorno da função PP_GetPin().|
|`CreditCard.PanSequenceNumber`|Number|---|---|Número sequencial do cartão, utilizado para identificar a conta corrente do cartão adicional. Mandatório para transações com cartões Chip EMV e que possuam PAN Sequence Number (Tag 5F34).|
|`CreditCard.SaveCard`|Booleano|---|---|Identifica se vai salvar/tokenizar o cartão.|
|`CreditCard.IsFallback`|Booleano|---|---|Identifica se é uma transação de fallback.|
|`PinPadInformation.TerminalId`|String|---|Sim|Número Lógico definido no Concentrador Cielo.|
|`PinPadInformation.SerialNumber`|String|---|Sim|Número de Série do Equipamento.|
|`PinPadInformation.PhysicalCharacteristics`|String|---|Sim|Enum: `WithoutPinPad` `PinPadWithoutChipReader` `PinPadWithChipReaderWithoutSamModule` `PinPadWithChipReaderWithSamModule` `NotCertifiedPinPad` `PinPadWithChipReaderWithoutSamAndContactless` `PinPadWithChipReaderWithSamModuleAndContactless` <br><br> Sem PIN-pad = `WithoutPinPad`; <br><br> PIN-pad sem leitor de Chip = `PinpadWithoutChipReader`; <br><br>PIN-pad com leitor de Chip sem módulo SAM = `PinPadWithChipReaderWithoutSamModule`; <br><br> PIN-pad com leitor de Chip com módulo SAM = `PinPadWithChipReaderWithSamModule`; <br><br> PIN-pad não homologado = `NotCertifiedPinPad`; <br><br> PIN-pad com leitor de Chip sem SAM e Cartão Sem Contato = `PinpadWithChipReaderWithoutSamAndContactless`; <br><br> PIN-pad com leitor de Chip com SAM e Cartão Sem Contato = `PinpadWithChipReaderWithSamAndContactless`. <br><br><br> Obs. Caso a aplicação não consiga informar os dados acima, deve obter tais informações através do retorno da função PP_GetInfo() da BC.|
|`PinPadInformation.ReturnDataInfo`|String|---|Sim|Retorno da função PP_GetInfo() da biblioteca compartilhada|
|`MassTransit.IsDebtRecovery`|Booleano|—|Sim|Identifica de a operação é de recuperação de pagamento.|
|`MassTransit.FirstTravelDate`|String|date-time|Sim|Data da primeira viagem|
|`MassTransit.IsKnownValue`|Booleano|-|Sim|Indica que a operação é de valor conhecido|

#### Resposta

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`MerchantOrderId`|String|---|---| Número do documento gerado automaticamente pelo terminal e incrementado de 1 a cada transação realizada no terminal. Aceita apenas valores numéricos de 1 a 15 dígitos.|
|`Customer.Name`|String|---|---|---|
|`Customer.Identity`|---|---|---|---|
|`Customer.IdentityType`|---|---|---|---|
|`Customer.Email`|---|---|---|---|
|`Customer.Birthday`|---|---|---|---|
|`Address.Street`|String|---|---|---|
|`Address.Number`|String|---|---|---|
|`Address.Complement`|String|---|---||---|
|`Address.ZipCode`|String|---|---|---|
|`Address.City`|String|---|---|---|
|`Address.State`|String|---|---|---|
|`Address.Country`|String|---|---|---|
|`DeliveryAddress.Street`|String|---|---|---|
|`DeliveryAddress.Number`|String|---|---|---|
|`DeliveryAddress.Complement`|String|---|---||---|
|`DeliveryAddress.ZipCode`|String|---|---|---|
|`DeliveryAddress.City`|String|---|---|---|
|`DeliveryAddress.State`|String|---|---|---|
|`DeliveryAddress.Country`|String|---|---|---|
|`Payment.Installments`|Integer|---|---|Default: 1 / Quantidade de Parcelas: Varia de 2 a 99 para transação de financiamento. Deve ser verificado os atributos maxOfPayments1, maxOfPayments2, maxOfPayments3 e minValOfPayments da tabela productTable.|
|`Payment.Interest`|String|---|---|Default: `ByMerchant` <br><br> Enum: `ByMerchant` `ByIssuer` <br><br> Tipo de Parcelamento: <br><br> - Se o bit 6 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 6 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento sem juros pode ser efetuado. <br><br> - Se o bit 7 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 7 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento com juros pode ser efetuado. Sem juros = “ByMerchant”; Com juros = “ByIssuer”.|
|`Payment.Capture`|Booleano|---|---|Default: false / Booleano que identifica que a autorização deve ser com captura automática. A autorização sem captura automática é conhecida também como pré-autorização.|
|`CreditCard.ExpirationDate`|String|MM/yyyy|Sim|Data de validade do cartão. <br><br>Dado obtido através do comando PP_GetCard na BC no momento da captura da transação.|
|`CreditCard.BrandId`|Integer|---|Sim|Identificação da bandeira obtida através do campo BrandId da PRODUCT TABLE.|
|`CreditCard.IssuerId`|Integer|---|Sim|Código do emissor obtido através do campo IssuerId da BIN TABLE.|
|`CreditCard.TruncateCardNumberWhenPrinting`|Booleano|---|---|Indica se o número do cartão será truncado no momento da impressão do comprovante. A solução de captura deve tomar essa decisão com base no `confParamOp03` presente nas tabelas BIN TABLE, PARAMETER TABLE e ISSUER TABLE.|
|`CreditCard.InputMode`|String|---|Sim|Enum: `Typed` `MagStripe` `Emv` <br><br>Identificação do modo de captura do cartão na transação. Essa informação deve ser obtida através do retorno da função PP_GetCard da BC. <br><br>"00" – Magnético <br><br>"01" - Moedeiro VISA Cash sobre TIBC v1 <br><br>"02" - Moedeiro VISA Cash sobre TIBC v3 <br><br>"03" – EMV com contato <br><br>"04" - Easy-Entry sobre TIBC v1 <br><br>"05" - Chip sem contato simulando tarja <br><br> "06" - EMV sem contato.|
|`CreditCard.AuthenticationMethod`|String|---|Sim|Enum: `NoPassword` `OnlineAuthentication` `OfflineAuthentication` <br><br>Método de autenticação <br><br>- Se o cartão foi lido a partir da digitação verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, a senha deve ser capturada e o authenticationMethod assume valor 2. Caso contrário, assume valor 1; <br><br>- Se o cartão foi lido a partir da trilha verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, deve ser verificado o bit 2 do mesmo campo. Se este estiver com valor 1 deve ser capturada a senha. Se estiver com valor 0 a captura da senha vai depender do último dígito do service code; <br><br>- Se o cartão foi lido através do chip EMV, o authenticationMethod será preenchido com base no retorno da função PP_GoOnChip(). No resultado PP_GoOnChip(), onde se o campo da posição 003 do retorno da PP_GoOnChip() estiver com valor 1 indica que o pin foi validado off-line, o authenticationMethod será 3. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valor 0, o authenticationMethod será 1. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valores 0 e 1 respectivamente, o authenticationMethod será 2. <br><br>1 - Sem senha = “NoPassword”; <br><br>2 - Senha online = “Online Authentication”; <br><br>3 - Senha off-line = “Offline Authentication”.|
|`CreditCard.EmvData`|String|---|---|Dados da transação EMV <br><br>Obtidos através do comando PP_GoOnChip na BC|
|`PinBlock.EncryptedPinBlock`|---|---|---|---|
|`PinBlock.EncryptionType`|String|---|---|---|
|`PinBlock.KsnIdentification`|String|---|---|---|
|`CreditCard.PanSequenceNumber`|Number|---|---|Número sequencial do cartão, utilizado para identificar a conta corrente do cartão adicional. Mandatório para transações com cartões Chip EMV e que possuam PAN Sequence Number (Tag 5F34).|
|`CreditCard.SaveCard`|---|---|---|---|
|`CreditCard.IsFallback`|---|---|---|---|
|`Payment.PaymentDateTime`|String|date-time|Sim|Data e Hora da captura da transação|
|`Payment.ServiceTaxAmount`|---|---|---|---|
|`Payment.SoftDescriptor`|String|13|---|Identificação do estabelecimento (nome reduzido) a ser impresso e identificado na fatura.|
|`Payment.ProductId`|Integer|---|Sim|Código do produto identificado através do bin do cartão.|
|`PinPadInformation.TerminalId`|String|---|Sim|Número Lógico definido no Concentrador Cielo.|
|`PinPadInformation.SerialNumber`|String|---|Sim|Número de Série do Equipamento.|
|`PinPadInformation.PhysicalCharacteristics`|String|---|Sim|Enum: `WithoutPinPad` `PinPadWithoutChipReader` `PinPadWithChipReaderWithoutSamModule` `PinPadWithChipReaderWithSamModule` `NotCertifiedPinPad` `PinPadWithChipReaderWithoutSamAndContactless` `PinPadWithChipReaderWithSamModuleAndContactless` <br><br> Sem PIN-pad = `WithoutPinPad`; <br><br> PIN-pad sem leitor de Chip = `PinpadWithoutChipReader`; <br><br>PIN-pad com leitor de Chip sem módulo SAM = `PinPadWithChipReaderWithoutSamModule`; <br><br> PIN-pad com leitor de Chip com módulo SAM = `PinPadWithChipReaderWithSamModule`; <br><br> PIN-pad não homologado = `NotCertifiedPinPad`; <br><br> PIN-pad com leitor de Chip sem SAM e Cartão Sem Contato = `PinpadWithChipReaderWithoutSamAndContactless`; <br><br> PIN-pad com leitor de Chip com SAM e Cartão Sem Contato = `PinpadWithChipReaderWithSamAndContactless`. <br><br><br> Obs. Caso a aplicação não consiga informar os dados acima, deve obter tais informações através do retorno da função PP_GetInfo() da BC.|
|`PinPadInformation.ReturnDataInfo`|String|---|Sim|Retorno da função PP_GetInfo() da biblioteca compartilhada|
|`Payment.Amount`|Integer(int64)|---|Sim|Valor da transação (1079 = R$10,79)|
|`Payment.ReceivedDate`|---|---|---|---|
|`Payment.CapturedAmount`|---|---|---|---|
|`Payment.Provider`|String|---|---|---|
|`Payment.ConfirmationStatus`|---|---|---|---|
|`Payment.InitializationVersion`|---|---|---|---|
|`Payment.EmvResponseData`|---|---|---|---|
|`Payment.Status`|---|---|---|---|
|`Payment.IsSplitted`|Booleano|---|---|---|
|`Payment.ReturnCode`|---|---|---|---|
|`Payment.ReturnMessage`|String|---|---|---|
|`Payment.PaymentId`|---|---|---|---|
|`Payment.Type`|String|---|Sim|Value: `PhysicalCreditCard` / Tipo da Transação|
|`Payment.Currency`|String|---|---|Default: "BRL" / Value: "BRL" / Moeda (Preencher com “BRL”)|
|`Payment.Country`|String|---|---|Default: "BRA" / Value: "BRA" / País (Preencher com “BRA”)|
|`Receipt.MerchantName`|---|---|---|---|
|`Receipt.MerchantAddress`|---|---|---|---|
|`Receipt.MerchantCity`|---|---|---|---|
|`Receipt.MerchantState`|---|---|---|---|
|`Receipt.MerchantCode`|---|---|---|---|
|`Receipt.Terminal`|---|---|---|---|
|`Receipt.Nsu`|---|---|---|---|
|`Receipt.Date`|---|---|---|---|
|`Receipt.Hour`|---|---|---|---|
|`Receipt.IssuerName`|---|---|---|---|
|`Receipt.CardNumber`|---|---|---|---|
|`Receipt.TransactionType`|---|---|---|---|
|`Receipt.AuthorizationCode`|---|---|---|---|
|`Receipt.TransactionMode`|---|---|---|---|
|`Receipt.InputMethod`|---|---|---|---|
|`Receipt.Value`|---|---|---|---|
|`Receipt.SoftDescriptor`|---|---|---|---|
|`RecurrentPayment.RecurrentPaymentId`|---|---|---|---|
|`RecurrentPayment.ReasonCode`|---|---|---|---|
|`RecurrentPayment.ReasonMessage`|---|---|---|---|
|`RecurrentPayment.NextRecurrency`|---|---|---|---|
|`RecurrentPayment.EndDate`|---|---|---|---|
|`RecurrentPayment.Interval`|---|---|---|---|
|`SplitPayments.SubordinateMerchantId`|---|---|---|---|
|`SplitPayments.Amount`|---|---|---|---|
|`SplitPayments.Fares.Mdr`|---|---|---|---|
|`SplitPayments.Fares.Fee`|---|---|---|---|
|`SplitErrors.Code`|---|---|---|---|
|`SplitErrors.Message`|---|---|---|---|

### MassTransit - AVR

#### Requisição

<aside class="request"><span class="method post">POST</span> <span class="endpoint">/1/physicalSales/</span></aside>

```json
{
   "MerchantOrderId": "1596226820548",
   "Payment": {
      "Type": "PhysicalCreditCard",
      "SoftDescriptor": "Description",
      "PaymentDateTime": "2020-07-31T20:20:20.548Z",
      "Amount": 0,
      "Installments": 1,
      "Interest": "ByMerchant",
      "Capture": true,
      "ProductId": 1,
      "CreditCard": {
         "InputMode": "ContactlessEmv",
         "BrandId": 1,
         "IssuerId": 401,
         "TruncateCardNumberWhenPrinting": true,
         "ExpirationDate": "12/2020",
         "PanSequenceNumber": 1,
         "EmvSequenceNumber": 1,
         "EmvData": "112233445566778899011AABBC012D3456789E0123FF45678AB901234C5D112233445566778800",
         "TrackOneData": "A1234567890123456^FULANO OLIVEIRA SA ^12345678901234567890123",
         "TrackTwoData": "0123456789012345=012345678901234",
         "AuthenticationMethod": "OnlineAuthentication",
         "PinBlock": {
            "EncryptedPinBlock": "2280F6BDFD0C038D",
            "EncryptionType": "Dukpt3Des",
            "KsnIdentification": "fffff9999900522000d6"
         }
      },
      "PinPadInformation": {
         "TerminalId": "12345678",
         "SerialNumber": "6C651996",
         "PhysicalCharacteristics": "PinPadWithChipReaderWithSamModuleAndContactless",
         "ReturnDataInfo": "00"
      },
      "MassTransit": {
         "IsDebtRecovery": false,
         "FirstTravelDate": "2018-01-01 04:12"
      }
   }
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`MerchantOrderId`|String|---|---| Número do documento gerado automaticamente pelo terminal e incrementado de 1 a cada transação realizada no terminal. Aceita apenas valores numéricos de 1 a 15 dígitos.|
|`Payment.Type`|String|---|Sim|Value: `PhysicalCreditCard` / Tipo da Transação|
|`Payment.SoftDescriptor`|String|13|---|Identificação do estabelecimento (nome reduzido) a ser impresso e identificado na fatura.|
|`Payment.PaymentDateTime`|String|date-time|Sim|Data e Hora da captura da transação|
|`Payment.Amount`|Integer(int64)|---|Sim|Valor da transação (1079 = R$10,79)|
|`Payment.Installments`|Integer|---|---|Default: 1 / Quantidade de Parcelas: Varia de 2 a 99 para transação de financiamento. Deve ser verificado os atributos maxOfPayments1, maxOfPayments2, maxOfPayments3 e minValOfPayments da tabela productTable.|
|`Payment.Interest`|String|---|---|Default: `ByMerchant` <br><br> Enum: `ByMerchant` `ByIssuer` <br><br> Tipo de Parcelamento: <br><br> - Se o bit 6 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 6 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento sem juros pode ser efetuado. <br><br> - Se o bit 7 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 7 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento com juros pode ser efetuado. Sem juros = “ByMerchant”; Com juros = “ByIssuer”.|
|`Payment.Capture`|Booleano|---|---|Default: false / Booleano que identifica que a autorização deve ser com captura automática. A autorização sem captura automática é conhecida também como pré-autorização.|
|`Payment.ProductId`|Integer|---|Sim|Código do produto identificado através do bin do cartão.|
|`CreditCard.ExpirationDate`|String|MM/yyyy|Sim|Data de validade do cartão. <br><br>Dado obtido através do comando PP_GetCard na BC no momento da captura da transação.|
|`CreditCard.BrandId`|Integer|---|Sim|Identificação da bandeira obtida através do campo BrandId da PRODUCT TABLE.|
|`CreditCard.IssuerId`|Integer|---|Sim|Código do emissor obtido através do campo IssuerId da BIN TABLE.|
|`CreditCard.InputMode`|String|---|Sim|Enum: `Typed` `MagStripe` `Emv` <br><br>Identificação do modo de captura do cartão na transação. Essa informação deve ser obtida através do retorno da função PP_GetCard da BC. <br><br>"00" – Magnético <br><br>"01" - Moedeiro VISA Cash sobre TIBC v1 <br><br>"02" - Moedeiro VISA Cash sobre TIBC v3 <br><br>"03" – EMV com contato <br><br>"04" - Easy-Entry sobre TIBC v1 <br><br>"05" - Chip sem contato simulando tarja <br><br> "06" - EMV sem contato.|
|`CreditCard.AuthenticationMethod`|String|---|Sim|Enum: `NoPassword` `OnlineAuthentication` `OfflineAuthentication` <br><br>Método de autenticação <br><br>- Se o cartão foi lido a partir da digitação verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, a senha deve ser capturada e o authenticationMethod assume valor 2. Caso contrário, assume valor 1; <br><br>- Se o cartão foi lido a partir da trilha verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, deve ser verificado o bit 2 do mesmo campo. Se este estiver com valor 1 deve ser capturada a senha. Se estiver com valor 0 a captura da senha vai depender do último dígito do service code; <br><br>- Se o cartão foi lido através do chip EMV, o authenticationMethod será preenchido com base no retorno da função PP_GoOnChip(). No resultado PP_GoOnChip(), onde se o campo da posição 003 do retorno da PP_GoOnChip() estiver com valor 1 indica que o pin foi validado off-line, o authenticationMethod será 3. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valor 0, o authenticationMethod será 1. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valores 0 e 1 respectivamente, o authenticationMethod será 2. <br><br>1 - Sem senha = “NoPassword”; <br><br>2 - Senha online = “Online Authentication”; <br><br>3 - Senha off-line = “Offline Authentication”.|
|`CreditCard.EmvData`|String|---|---|Dados da transação EMV<br><br>Obtidos através do comando PP_GoOnChip na BC|
|`PinBlock.EncryptedPinBlock`|String|---|Sim|PINBlock Criptografado<br><br>- Para transações EMV, esse campo é obtido através do retorno da função PP_GoOnChip(), mais especificamente das posições 007 até a posição 022;<br><br>- Para transações digitadas e com tarja magnética, verificar as posições 001 até 016 do retorno da função PP_GetPin().|
|`PinBlock.EncryptionType`|String|---|Sim|Tipo de Criptografia<br><br>Enum:<br><br>"DukptDes"<br><br>"Dukpt3Des"<br><br>"MasterKey"|
|`PinBlock.KsnIdentification`|String|---|Sim|Identificação do KSN<br><br>- Para transações EMV esse campo é obtido através do retorno da função PP_GoOnChip() nas posições 023 até 042;<br><br>- Para transações digitadas e com tarja magnética, verificar as posições 017 até 036 do retorno da função PP_GetPin().|
|`CreditCard.PanSequenceNumber`|Number|---|---|Número sequencial do cartão, utilizado para identificar a conta corrente do cartão adicional. Mandatório para transações com cartões Chip EMV e que possuam PAN Sequence Number (Tag 5F34).|
|`CreditCard.SaveCard`|Booleano|---|---|Identifica se vai salvar/tokenizar o cartão.|
|`CreditCard.IsFallback`|Booleano|---|---|Identifica se é uma transação de fallback.|
|`PinPadInformation.TerminalId`|String|---|Sim|Número Lógico definido no Concentrador Cielo.|
|`PinPadInformation.SerialNumber`|String|---|Sim|Número de Série do Equipamento.|
|`PinPadInformation.PhysicalCharacteristics`|String|---|Sim|Enum: `WithoutPinPad` `PinPadWithoutChipReader` `PinPadWithChipReaderWithoutSamModule` `PinPadWithChipReaderWithSamModule` `NotCertifiedPinPad` `PinPadWithChipReaderWithoutSamAndContactless` `PinPadWithChipReaderWithSamModuleAndContactless` <br><br> Sem PIN-pad = `WithoutPinPad`; <br><br> PIN-pad sem leitor de Chip = `PinpadWithoutChipReader`; <br><br>PIN-pad com leitor de Chip sem módulo SAM = `PinPadWithChipReaderWithoutSamModule`; <br><br> PIN-pad com leitor de Chip com módulo SAM = `PinPadWithChipReaderWithSamModule`; <br><br> PIN-pad não homologado = `NotCertifiedPinPad`; <br><br> PIN-pad com leitor de Chip sem SAM e Cartão Sem Contato = `PinpadWithChipReaderWithoutSamAndContactless`; <br><br> PIN-pad com leitor de Chip com SAM e Cartão Sem Contato = `PinpadWithChipReaderWithSamAndContactless`. <br><br><br> Obs. Caso a aplicação não consiga informar os dados acima, deve obter tais informações através do retorno da função PP_GetInfo() da BC.|
|`PinPadInformation.ReturnDataInfo`|String|---|Sim|Retorno da função PP_GetInfo() da biblioteca compartilhada|
|`MassTransit.IsDebtRecovery`|Booleano|—|Sim|Identifica de a operação é de recuperação de pagamento.|
|`MassTransit.FirstTravelDate`|String|date-time|Sim|Data da primeira viagem|

#### Resposta

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`MerchantOrderId`|String|---|---| Número do documento gerado automaticamente pelo terminal e incrementado de 1 a cada transação realizada no terminal. Aceita apenas valores numéricos de 1 a 15 dígitos.|
|`Customer.Name`|String|---|---|---|
|`Customer.Identity`|---|---|---|---|
|`Customer.IdentityType`|---|---|---|---|
|`Customer.Email`|---|---|---|---|
|`Customer.Birthday`|---|---|---|---|
|`Address.Street`|String|---|---|---|
|`Address.Number`|String|---|---|---|
|`Address.Complement`|String|---|---||---|
|`Address.ZipCode`|String|---|---|---|
|`Address.City`|String|---|---|---|
|`Address.State`|String|---|---|---|
|`Address.Country`|String|---|---|---|
|`DeliveryAddress.Street`|String|---|---|---|
|`DeliveryAddress.Number`|String|---|---|---|
|`DeliveryAddress.Complement`|String|---|---||---|
|`DeliveryAddress.ZipCode`|String|---|---|---|
|`DeliveryAddress.City`|String|---|---|---|
|`DeliveryAddress.State`|String|---|---|---|
|`DeliveryAddress.Country`|String|---|---|---|
|`Payment.Installments`|Integer|---|---|Default: 1 / Quantidade de Parcelas: Varia de 2 a 99 para transação de financiamento. Deve ser verificado os atributos maxOfPayments1, maxOfPayments2, maxOfPayments3 e minValOfPayments da tabela productTable.|
|`Payment.Interest`|String|---|---|Default: `ByMerchant` <br><br> Enum: `ByMerchant` `ByIssuer` <br><br> Tipo de Parcelamento: <br><br> - Se o bit 6 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 6 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento sem juros pode ser efetuado. <br><br> - Se o bit 7 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 7 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento com juros pode ser efetuado. Sem juros = “ByMerchant”; Com juros = “ByIssuer”.|
|`Payment.Capture`|Booleano|---|---|Default: false / Booleano que identifica que a autorização deve ser com captura automática. A autorização sem captura automática é conhecida também como pré-autorização.|
|`CreditCard.ExpirationDate`|String|MM/yyyy|Sim|Data de validade do cartão. <br><br>Dado obtido através do comando PP_GetCard na BC no momento da captura da transação.|
|`CreditCard.BrandId`|Integer|---|Sim|Identificação da bandeira obtida através do campo BrandId da PRODUCT TABLE.|
|`CreditCard.IssuerId`|Integer|---|Sim|Código do emissor obtido através do campo IssuerId da BIN TABLE.|
|`CreditCard.TruncateCardNumberWhenPrinting`|Booleano|---|---|Indica se o número do cartão será truncado no momento da impressão do comprovante. A solução de captura deve tomar essa decisão com base no `confParamOp03` presente nas tabelas BIN TABLE, PARAMETER TABLE e ISSUER TABLE.|
|`CreditCard.InputMode`|String|---|Sim|Enum: `Typed` `MagStripe` `Emv` <br><br>Identificação do modo de captura do cartão na transação. Essa informação deve ser obtida através do retorno da função PP_GetCard da BC. <br><br>"00" – Magnético <br><br>"01" - Moedeiro VISA Cash sobre TIBC v1 <br><br>"02" - Moedeiro VISA Cash sobre TIBC v3 <br><br>"03" – EMV com contato <br><br>"04" - Easy-Entry sobre TIBC v1 <br><br>"05" - Chip sem contato simulando tarja <br><br> "06" - EMV sem contato.|
|`CreditCard.AuthenticationMethod`|String|---|Sim|Enum: `NoPassword` `OnlineAuthentication` `OfflineAuthentication` <br><br>Método de autenticação <br><br>- Se o cartão foi lido a partir da digitação verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, a senha deve ser capturada e o authenticationMethod assume valor 2. Caso contrário, assume valor 1; <br><br>- Se o cartão foi lido a partir da trilha verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, deve ser verificado o bit 2 do mesmo campo. Se este estiver com valor 1 deve ser capturada a senha. Se estiver com valor 0 a captura da senha vai depender do último dígito do service code; <br><br>- Se o cartão foi lido através do chip EMV, o authenticationMethod será preenchido com base no retorno da função PP_GoOnChip(). No resultado PP_GoOnChip(), onde se o campo da posição 003 do retorno da PP_GoOnChip() estiver com valor 1 indica que o pin foi validado off-line, o authenticationMethod será 3. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valor 0, o authenticationMethod será 1. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valores 0 e 1 respectivamente, o authenticationMethod será 2. <br><br>1 - Sem senha = “NoPassword”; <br><br>2 - Senha online = “Online Authentication”; <br><br>3 - Senha off-line = “Offline Authentication”.|
|`CreditCard.EmvData`|String|---|---|Dados da transação EMV <br><br>Obtidos através do comando PP_GoOnChip na BC|
|`PinBlock.EncryptedPinBlock`|---|---|---|---|
|`PinBlock.EncryptionType`|String|---|---|---|
|`PinBlock.KsnIdentification`|String|---|---|---|
|`CreditCard.PanSequenceNumber`|Number|---|---|Número sequencial do cartão, utilizado para identificar a conta corrente do cartão adicional. Mandatório para transações com cartões Chip EMV e que possuam PAN Sequence Number (Tag 5F34).|
|`CreditCard.SaveCard`|---|---|---|---|
|`CreditCard.IsFallback`|---|---|---|---|
|`Payment.PaymentDateTime`|String|date-time|Sim|Data e Hora da captura da transação|
|`Payment.ServiceTaxAmount`|---|---|---|---|
|`Payment.SoftDescriptor`|String|13|---|Identificação do estabelecimento (nome reduzido) a ser impresso e identificado na fatura.|
|`Payment.ProductId`|Integer|---|Sim|Código do produto identificado através do bin do cartão.|
|`PinPadInformation.TerminalId`|String|---|Sim|Número Lógico definido no Concentrador Cielo.|
|`PinPadInformation.SerialNumber`|String|---|Sim|Número de Série do Equipamento.|
|`PinPadInformation.PhysicalCharacteristics`|String|---|Sim|Enum: `WithoutPinPad` `PinPadWithoutChipReader` `PinPadWithChipReaderWithoutSamModule` `PinPadWithChipReaderWithSamModule` `NotCertifiedPinPad` `PinPadWithChipReaderWithoutSamAndContactless` `PinPadWithChipReaderWithSamModuleAndContactless` <br><br> Sem PIN-pad = `WithoutPinPad`; <br><br> PIN-pad sem leitor de Chip = `PinpadWithoutChipReader`; <br><br>PIN-pad com leitor de Chip sem módulo SAM = `PinPadWithChipReaderWithoutSamModule`; <br><br> PIN-pad com leitor de Chip com módulo SAM = `PinPadWithChipReaderWithSamModule`; <br><br> PIN-pad não homologado = `NotCertifiedPinPad`; <br><br> PIN-pad com leitor de Chip sem SAM e Cartão Sem Contato = `PinpadWithChipReaderWithoutSamAndContactless`; <br><br> PIN-pad com leitor de Chip com SAM e Cartão Sem Contato = `PinpadWithChipReaderWithSamAndContactless`. <br><br><br> Obs. Caso a aplicação não consiga informar os dados acima, deve obter tais informações através do retorno da função PP_GetInfo() da BC.|
|`PinPadInformation.ReturnDataInfo`|String|---|Sim|Retorno da função PP_GetInfo() da biblioteca compartilhada|
|`Payment.Amount`|Integer(int64)|---|Sim|Valor da transação (1079 = R$10,79)|
|`Payment.ReceivedDate`|---|---|---|---|
|`Payment.CapturedAmount`|---|---|---|---|
|`Payment.Provider`|String|---|---|---|
|`Payment.ConfirmationStatus`|---|---|---|---|
|`Payment.InitializationVersion`|---|---|---|---|
|`Payment.EmvResponseData`|---|---|---|---|
|`Payment.Status`|---|---|---|---|
|`Payment.IsSplitted`|Booleano|---|---|---|
|`Payment.ReturnCode`|---|---|---|---|
|`Payment.ReturnMessage`|String|---|---|---|
|`Payment.PaymentId`|---|---|---|---|
|`Payment.Type`|String|---|Sim|Value: `PhysicalCreditCard` / Tipo da Transação|
|`Payment.Currency`|String|---|---|Default: "BRL" / Value: "BRL" / Moeda (Preencher com “BRL”)|
|`Payment.Country`|String|---|---|Default: "BRA" / Value: "BRA" / País (Preencher com “BRA”)|
|`Receipt.MerchantName`|---|---|---|---|
|`Receipt.MerchantAddress`|---|---|---|---|
|`Receipt.MerchantCity`|---|---|---|---|
|`Receipt.MerchantState`|---|---|---|---|
|`Receipt.MerchantCode`|---|---|---|---|
|`Receipt.Terminal`|---|---|---|---|
|`Receipt.Nsu`|---|---|---|---|
|`Receipt.Date`|---|---|---|---|
|`Receipt.Hour`|---|---|---|---|
|`Receipt.IssuerName`|---|---|---|---|
|`Receipt.CardNumber`|---|---|---|---|
|`Receipt.TransactionType`|---|---|---|---|
|`Receipt.AuthorizationCode`|---|---|---|---|
|`Receipt.TransactionMode`|---|---|---|---|
|`Receipt.InputMethod`|---|---|---|---|
|`Receipt.Value`|---|---|---|---|
|`Receipt.SoftDescriptor`|---|---|---|---|
|`RecurrentPayment.RecurrentPaymentId`|---|---|---|---|
|`RecurrentPayment.ReasonCode`|---|---|---|---|
|`RecurrentPayment.ReasonMessage`|---|---|---|---|
|`RecurrentPayment.NextRecurrency`|---|---|---|---|
|`RecurrentPayment.EndDate`|---|---|---|---|
|`RecurrentPayment.Interval`|---|---|---|---|
|`SplitPayments.SubordinateMerchantId`|---|---|---|---|
|`SplitPayments.Amount`|---|---|---|---|
|`SplitPayments.Fares.Mdr`|---|---|---|---|
|`SplitPayments.Fares.Fee`|---|---|---|---|
|`SplitErrors.Code`|---|---|---|---|
|`SplitErrors.Message`|---|---|---|---|

### Debt Recovery - MTT/KFT

#### Requisição

<aside class="request"><span class="method post">POST</span> <span class="endpoint">/1/physicalSales/</span></aside>

```json
{
   "MerchantOrderId": "1596226820548",
   "Payment": {
      "SubordinatedMerchantId": "{Auth_ClientId}",
      "Type": "PhysicalCreditCard",
      "SoftDescriptor": "Transação API",
      "PaymentDateTime": "2020-07-31T20:20:20.548Z",
      "Amount": 100,
      "Installments": 1,
      "Interest": "ByMerchant",
      "Capture": true,
      "ProductId": 1,
      "CreditCard": {
         "CardToken": "{{CardToken}}",
         "ExpirationDate": "12/2020",
         "SecurityCodeStatus": "Collected",
         "SecurityCode": "{{SecurityCode}}",
         "BrandId": 1,
         "IssuerId": 401,
         "InputMode": "Typed",
         "AuthenticationMethod": "NoPassword",
         "TruncateCardNumberWhenPrinting": true,
         "SaveCard": false
      },
      "PinPadInformation": {
         "PhysicalCharacteristics": "PinPadWithChipReaderWithoutSamAndContactless",
         "ReturnDataInfo": "00",
         "SerialNumber": "0820471929",
         "TerminalId": "12345678"
      },
      "MassTransit": {
         "IsDebtRecovery": true,
         "ISKnownValue": false,
         "FirstTravelDate": "2018-01-01 04:12",
         "PaymentId": "53f4b75e-b20e-46e8-bf2e-07db03c17aef"
      }
   }
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`MerchantOrderId`|String|---|---| Número do documento gerado automaticamente pelo terminal e incrementado de 1 a cada transação realizada no terminal. Aceita apenas valores numéricos de 1 a 15 dígitos.|
|`Payment.Type`|String|---|Sim|Value: `PhysicalCreditCard` / Tipo da Transação|
|`Payment.SoftDescriptor`|String|13|---|Identificação do estabelecimento (nome reduzido) a ser impresso e identificado na fatura.|
|`Payment.PaymentDateTime`|String|date-time|Sim|Data e Hora da captura da transação|
|`Payment.Amount`|Integer(int64)|---|Sim|Valor da transação (1079 = R$10,79)|
|`Payment.Installments`|Integer|---|---|Default: 1 / Quantidade de Parcelas: Varia de 2 a 99 para transação de financiamento. Deve ser verificado os atributos maxOfPayments1, maxOfPayments2, maxOfPayments3 e minValOfPayments da tabela productTable.|
|`Payment.Interest`|String|---|---|Default: `ByMerchant` <br><br> Enum: `ByMerchant` `ByIssuer` <br><br> Tipo de Parcelamento: <br><br> - Se o bit 6 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 6 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento sem juros pode ser efetuado. <br><br> - Se o bit 7 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 7 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento com juros pode ser efetuado. Sem juros = “ByMerchant”; Com juros = “ByIssuer”.|
|`Payment.Capture`|Booleano|---|---|Default: false / Booleano que identifica que a autorização deve ser com captura automática. A autorização sem captura automática é conhecida também como pré-autorização.|
|`Payment.ProductId`|Integer|---|Sim|Código do produto identificado através do bin do cartão.|
|`CreditCard.ExpirationDate`|String|MM/yyyy|Sim|Data de validade do cartão. <br><br>Dado obtido através do comando PP_GetCard na BC no momento da captura da transação.|
|`CreditCard.BrandId`|Integer|---|Sim|Identificação da bandeira obtida através do campo BrandId da PRODUCT TABLE.|
|`CreditCard.IssuerId`|Integer|---|Sim|Código do emissor obtido através do campo IssuerId da BIN TABLE.|
|`CreditCard.InputMode`|String|---|Sim|Enum: `Typed` `MagStripe` `Emv` <br><br>Identificação do modo de captura do cartão na transação. Essa informação deve ser obtida através do retorno da função PP_GetCard da BC. <br><br>"00" – Magnético <br><br>"01" - Moedeiro VISA Cash sobre TIBC v1 <br><br>"02" - Moedeiro VISA Cash sobre TIBC v3 <br><br>"03" – EMV com contato <br><br>"04" - Easy-Entry sobre TIBC v1 <br><br>"05" - Chip sem contato simulando tarja <br><br> "06" - EMV sem contato.|
|`CreditCard.AuthenticationMethod`|String|---|Sim|Enum: `NoPassword` `OnlineAuthentication` `OfflineAuthentication` <br><br>Método de autenticação <br><br>- Se o cartão foi lido a partir da digitação verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, a senha deve ser capturada e o authenticationMethod assume valor 2. Caso contrário, assume valor 1; <br><br>- Se o cartão foi lido a partir da trilha verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, deve ser verificado o bit 2 do mesmo campo. Se este estiver com valor 1 deve ser capturada a senha. Se estiver com valor 0 a captura da senha vai depender do último dígito do service code; <br><br>- Se o cartão foi lido através do chip EMV, o authenticationMethod será preenchido com base no retorno da função PP_GoOnChip(). No resultado PP_GoOnChip(), onde se o campo da posição 003 do retorno da PP_GoOnChip() estiver com valor 1 indica que o pin foi validado off-line, o authenticationMethod será 3. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valor 0, o authenticationMethod será 1. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valores 0 e 1 respectivamente, o authenticationMethod será 2. <br><br>1 - Sem senha = “NoPassword”; <br><br>2 - Senha online = “Online Authentication”; <br><br>3 - Senha off-line = “Offline Authentication”.|
|`CreditCard.EmvData`|String|---|---|Dados da transação EMV<br><br>Obtidos através do comando PP_GoOnChip na BC|
|`PinBlock.EncryptedPinBlock`|String|---|Sim|PINBlock Criptografado<br><br>- Para transações EMV, esse campo é obtido através do retorno da função PP_GoOnChip(), mais especificamente das posições 007 até a posição 022;<br><br>- Para transações digitadas e com tarja magnética, verificar as posições 001 até 016 do retorno da função PP_GetPin().|
|`PinBlock.EncryptionType`|String|---|Sim|Tipo de Criptografia<br><br>Enum:<br><br>"DukptDes"<br><br>"Dukpt3Des"<br><br>"MasterKey"|
|`PinBlock.KsnIdentification`|String|---|Sim|Identificação do KSN<br><br>- Para transações EMV esse campo é obtido através do retorno da função PP_GoOnChip() nas posições 023 até 042;<br><br>- Para transações digitadas e com tarja magnética, verificar as posições 017 até 036 do retorno da função PP_GetPin().|
|`CreditCard.PanSequenceNumber`|Number|---|---|Número sequencial do cartão, utilizado para identificar a conta corrente do cartão adicional. Mandatório para transações com cartões Chip EMV e que possuam PAN Sequence Number (Tag 5F34).|
|`CreditCard.SaveCard`|Booleano|---|---|Identifica se vai salvar/tokenizar o cartão.|
|`CreditCard.IsFallback`|Booleano|---|---|Identifica se é uma transação de fallback.|
|`PinPadInformation.TerminalId`|String|---|Sim|Número Lógico definido no Concentrador Cielo.|
|`PinPadInformation.SerialNumber`|String|---|Sim|Número de Série do Equipamento.|
|`PinPadInformation.PhysicalCharacteristics`|String|---|Sim|Enum: `WithoutPinPad` `PinPadWithoutChipReader` `PinPadWithChipReaderWithoutSamModule` `PinPadWithChipReaderWithSamModule` `NotCertifiedPinPad` `PinPadWithChipReaderWithoutSamAndContactless` `PinPadWithChipReaderWithSamModuleAndContactless` <br><br> Sem PIN-pad = `WithoutPinPad`; <br><br> PIN-pad sem leitor de Chip = `PinpadWithoutChipReader`; <br><br>PIN-pad com leitor de Chip sem módulo SAM = `PinPadWithChipReaderWithoutSamModule`; <br><br> PIN-pad com leitor de Chip com módulo SAM = `PinPadWithChipReaderWithSamModule`; <br><br> PIN-pad não homologado = `NotCertifiedPinPad`; <br><br> PIN-pad com leitor de Chip sem SAM e Cartão Sem Contato = `PinpadWithChipReaderWithoutSamAndContactless`; <br><br> PIN-pad com leitor de Chip com SAM e Cartão Sem Contato = `PinpadWithChipReaderWithSamAndContactless`. <br><br><br> Obs. Caso a aplicação não consiga informar os dados acima, deve obter tais informações através do retorno da função PP_GetInfo() da BC.|
|`PinPadInformation.ReturnDataInfo`|String|---|Sim|Retorno da função PP_GetInfo() da biblioteca compartilhada|
|`MassTransit.IsDebtRecovery`|Booleano|—|Sim|Identifica de a operação é de recuperação de pagamento.|
|`MassTransit.FirstTravelDate`|String|date-time|Sim|Data da primeira viagem|
|`MassTransit.IsKnownValue`|Booleano|-|Sim|Indica que a operação é de valor conhecido|
|`MassTransit.PaymentId`|String|-|Sim|Identificador do pagamento a ser recuperado|

#### Resposta

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`MerchantOrderId`|String|---|---| Número do documento gerado automaticamente pelo terminal e incrementado de 1 a cada transação realizada no terminal. Aceita apenas valores numéricos de 1 a 15 dígitos.|
|`Customer.Name`|String|---|---|---|
|`Customer.Identity`|---|---|---|---|
|`Customer.IdentityType`|---|---|---|---|
|`Customer.Email`|---|---|---|---|
|`Customer.Birthday`|---|---|---|---|
|`Address.Street`|String|---|---|---|
|`Address.Number`|String|---|---|---|
|`Address.Complement`|String|---|---||---|
|`Address.ZipCode`|String|---|---|---|
|`Address.City`|String|---|---|---|
|`Address.State`|String|---|---|---|
|`Address.Country`|String|---|---|---|
|`DeliveryAddress.Street`|String|---|---|---|
|`DeliveryAddress.Number`|String|---|---|---|
|`DeliveryAddress.Complement`|String|---|---||---|
|`DeliveryAddress.ZipCode`|String|---|---|---|
|`DeliveryAddress.City`|String|---|---|---|
|`DeliveryAddress.State`|String|---|---|---|
|`DeliveryAddress.Country`|String|---|---|---|
|`Payment.Installments`|Integer|---|---|Default: 1 / Quantidade de Parcelas: Varia de 2 a 99 para transação de financiamento. Deve ser verificado os atributos maxOfPayments1, maxOfPayments2, maxOfPayments3 e minValOfPayments da tabela productTable.|
|`Payment.Interest`|String|---|---|Default: `ByMerchant` <br><br> Enum: `ByMerchant` `ByIssuer` <br><br> Tipo de Parcelamento: <br><br> - Se o bit 6 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 6 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento sem juros pode ser efetuado. <br><br> - Se o bit 7 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 7 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento com juros pode ser efetuado. Sem juros = “ByMerchant”; Com juros = “ByIssuer”.|
|`Payment.Capture`|Booleano|---|---|Default: false / Booleano que identifica que a autorização deve ser com captura automática. A autorização sem captura automática é conhecida também como pré-autorização.|
|`CreditCard.ExpirationDate`|String|MM/yyyy|Sim|Data de validade do cartão. <br><br>Dado obtido através do comando PP_GetCard na BC no momento da captura da transação.|
|`CreditCard.BrandId`|Integer|---|Sim|Identificação da bandeira obtida através do campo BrandId da PRODUCT TABLE.|
|`CreditCard.IssuerId`|Integer|---|Sim|Código do emissor obtido através do campo IssuerId da BIN TABLE.|
|`CreditCard.TruncateCardNumberWhenPrinting`|Booleano|---|---|Indica se o número do cartão será truncado no momento da impressão do comprovante. A solução de captura deve tomar essa decisão com base no `confParamOp03` presente nas tabelas BIN TABLE, PARAMETER TABLE e ISSUER TABLE.|
|`CreditCard.InputMode`|String|---|Sim|Enum: `Typed` `MagStripe` `Emv` <br><br>Identificação do modo de captura do cartão na transação. Essa informação deve ser obtida através do retorno da função PP_GetCard da BC. <br><br>"00" – Magnético <br><br>"01" - Moedeiro VISA Cash sobre TIBC v1 <br><br>"02" - Moedeiro VISA Cash sobre TIBC v3 <br><br>"03" – EMV com contato <br><br>"04" - Easy-Entry sobre TIBC v1 <br><br>"05" - Chip sem contato simulando tarja <br><br> "06" - EMV sem contato.|
|`CreditCard.AuthenticationMethod`|String|---|Sim|Enum: `NoPassword` `OnlineAuthentication` `OfflineAuthentication` <br><br>Método de autenticação <br><br>- Se o cartão foi lido a partir da digitação verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, a senha deve ser capturada e o authenticationMethod assume valor 2. Caso contrário, assume valor 1; <br><br>- Se o cartão foi lido a partir da trilha verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, deve ser verificado o bit 2 do mesmo campo. Se este estiver com valor 1 deve ser capturada a senha. Se estiver com valor 0 a captura da senha vai depender do último dígito do service code; <br><br>- Se o cartão foi lido através do chip EMV, o authenticationMethod será preenchido com base no retorno da função PP_GoOnChip(). No resultado PP_GoOnChip(), onde se o campo da posição 003 do retorno da PP_GoOnChip() estiver com valor 1 indica que o pin foi validado off-line, o authenticationMethod será 3. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valor 0, o authenticationMethod será 1. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valores 0 e 1 respectivamente, o authenticationMethod será 2. <br><br>1 - Sem senha = “NoPassword”; <br><br>2 - Senha online = “Online Authentication”; <br><br>3 - Senha off-line = “Offline Authentication”.|
|`CreditCard.EmvData`|String|---|---|Dados da transação EMV <br><br>Obtidos através do comando PP_GoOnChip na BC|
|`PinBlock.EncryptedPinBlock`|---|---|---|---|
|`PinBlock.EncryptionType`|String|---|---|---|
|`PinBlock.KsnIdentification`|String|---|---|---|
|`CreditCard.PanSequenceNumber`|Number|---|---|Número sequencial do cartão, utilizado para identificar a conta corrente do cartão adicional. Mandatório para transações com cartões Chip EMV e que possuam PAN Sequence Number (Tag 5F34).|
|`CreditCard.SaveCard`|---|---|---|---|
|`CreditCard.IsFallback`|---|---|---|---|
|`Payment.PaymentDateTime`|String|date-time|Sim|Data e Hora da captura da transação|
|`Payment.ServiceTaxAmount`|---|---|---|---|
|`Payment.SoftDescriptor`|String|13|---|Identificação do estabelecimento (nome reduzido) a ser impresso e identificado na fatura.|
|`Payment.ProductId`|Integer|---|Sim|Código do produto identificado através do bin do cartão.|
|`PinPadInformation.TerminalId`|String|---|Sim|Número Lógico definido no Concentrador Cielo.|
|`PinPadInformation.SerialNumber`|String|---|Sim|Número de Série do Equipamento.|
|`PinPadInformation.PhysicalCharacteristics`|String|---|Sim|Enum: `WithoutPinPad` `PinPadWithoutChipReader` `PinPadWithChipReaderWithoutSamModule` `PinPadWithChipReaderWithSamModule` `NotCertifiedPinPad` `PinPadWithChipReaderWithoutSamAndContactless` `PinPadWithChipReaderWithSamModuleAndContactless` <br><br> Sem PIN-pad = `WithoutPinPad`; <br><br> PIN-pad sem leitor de Chip = `PinpadWithoutChipReader`; <br><br>PIN-pad com leitor de Chip sem módulo SAM = `PinPadWithChipReaderWithoutSamModule`; <br><br> PIN-pad com leitor de Chip com módulo SAM = `PinPadWithChipReaderWithSamModule`; <br><br> PIN-pad não homologado = `NotCertifiedPinPad`; <br><br> PIN-pad com leitor de Chip sem SAM e Cartão Sem Contato = `PinpadWithChipReaderWithoutSamAndContactless`; <br><br> PIN-pad com leitor de Chip com SAM e Cartão Sem Contato = `PinpadWithChipReaderWithSamAndContactless`. <br><br><br> Obs. Caso a aplicação não consiga informar os dados acima, deve obter tais informações através do retorno da função PP_GetInfo() da BC.|
|`PinPadInformation.ReturnDataInfo`|String|---|Sim|Retorno da função PP_GetInfo() da biblioteca compartilhada|
|`Payment.Amount`|Integer(int64)|---|Sim|Valor da transação (1079 = R$10,79)|
|`Payment.ReceivedDate`|---|---|---|---|
|`Payment.CapturedAmount`|---|---|---|---|
|`Payment.Provider`|String|---|---|---|
|`Payment.ConfirmationStatus`|---|---|---|---|
|`Payment.InitializationVersion`|---|---|---|---|
|`Payment.EmvResponseData`|---|---|---|---|
|`Payment.Status`|---|---|---|---|
|`Payment.IsSplitted`|Booleano|---|---|---|
|`Payment.ReturnCode`|---|---|---|---|
|`Payment.ReturnMessage`|String|---|---|---|
|`Payment.PaymentId`|---|---|---|---|
|`Payment.Type`|String|---|Sim|Value: `PhysicalCreditCard` / Tipo da Transação|
|`Payment.Currency`|String|---|---|Default: "BRL" / Value: "BRL" / Moeda (Preencher com “BRL”)|
|`Payment.Country`|String|---|---|Default: "BRA" / Value: "BRA" / País (Preencher com “BRA”)|
|`Receipt.MerchantName`|---|---|---|---|
|`Receipt.MerchantAddress`|---|---|---|---|
|`Receipt.MerchantCity`|---|---|---|---|
|`Receipt.MerchantState`|---|---|---|---|
|`Receipt.MerchantCode`|---|---|---|---|
|`Receipt.Terminal`|---|---|---|---|
|`Receipt.Nsu`|---|---|---|---|
|`Receipt.Date`|---|---|---|---|
|`Receipt.Hour`|---|---|---|---|
|`Receipt.IssuerName`|---|---|---|---|
|`Receipt.CardNumber`|---|---|---|---|
|`Receipt.TransactionType`|---|---|---|---|
|`Receipt.AuthorizationCode`|---|---|---|---|
|`Receipt.TransactionMode`|---|---|---|---|
|`Receipt.InputMethod`|---|---|---|---|
|`Receipt.Value`|---|---|---|---|
|`Receipt.SoftDescriptor`|---|---|---|---|
|`RecurrentPayment.RecurrentPaymentId`|---|---|---|---|
|`RecurrentPayment.ReasonCode`|---|---|---|---|
|`RecurrentPayment.ReasonMessage`|---|---|---|---|
|`RecurrentPayment.NextRecurrency`|---|---|---|---|
|`RecurrentPayment.EndDate`|---|---|---|---|
|`RecurrentPayment.Interval`|---|---|---|---|
|`SplitPayments.SubordinateMerchantId`|---|---|---|---|
|`SplitPayments.Amount`|---|---|---|---|
|`SplitPayments.Fares.Mdr`|---|---|---|---|
|`SplitPayments.Fares.Fee`|---|---|---|---|
|`SplitErrors.Code`|---|---|---|---|
|`SplitErrors.Message`|---|---|---|---|

### Debt Recovery EMV - MTT/KFT

#### Requisição

<aside class="request"><span class="method post">POST</span> <span class="endpoint">/1/physicalSales/</span></aside>

```json
{
   "MerchantOrderId": "1596226820548",
   "Payment": {
      "Type": "PhysicalCreditCard",
      "SoftDescriptor": "Description",
      "PaymentDateTime": "2020-07-31T20:20:20.548Z",
      "Amount": 300,
      "Installments": 1,
      "Interest": "ByMerchant",
      "Capture": true,
      "ProductId": 1,
      "CreditCard": {
         "InputMode": "ContactlessEmv",
         "BrandId": 1,
         "IssuerId": 401,
         "TruncateCardNumberWhenPrinting": true,
         "ExpirationDate": "12/2020",
         "EmvSequenceNumber": 1,
         "EmvData": "112233445566778899011AABBC012D3456789E0123FF45678AB901234C5D112233445566778800",
         "TrackOneData": "A1234567890123456^FULANO OLIVEIRA SA ^12345678901234567890123",
         "TrackTwoData": "0123456789012345=012345678901234",
         "AuthenticationMethod": "OnlineAuthentication",
         "PinBlock": {
            "EncryptedPinBlock": "2280F6BDFD0C038D",
            "EncryptionType": "Dukpt3Des",
            "KsnIdentification": "fffff9999900522000d6"
         }
      },
      "PinPadInformation": {
         "TerminalId": "12345678",
         "SerialNumber": "6C651996",
         "PhysicalCharacteristics": "PinPadWithChipReaderWithSamModuleAndContactless",
         "ReturnDataInfo": "00"
      },
      "MassTransit": {
         "IsDebtRecovery": true,
         "IsKnownValue": true,
         "FirstTravelDate": "2018-01-01 04:12"
      }
   }
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`MerchantOrderId`|String|---|---| Número do documento gerado automaticamente pelo terminal e incrementado de 1 a cada transação realizada no terminal. Aceita apenas valores numéricos de 1 a 15 dígitos.|
|`Payment.Type`|String|---|Sim|Value: `PhysicalCreditCard` / Tipo da Transação|
|`Payment.SoftDescriptor`|String|13|---|Identificação do estabelecimento (nome reduzido) a ser impresso e identificado na fatura.|
|`Payment.PaymentDateTime`|String|date-time|Sim|Data e Hora da captura da transação|
|`Payment.Amount`|Integer(int64)|---|Sim|Valor da transação (1079 = R$10,79)|
|`Payment.Installments`|Integer|---|---|Default: 1 / Quantidade de Parcelas: Varia de 2 a 99 para transação de financiamento. Deve ser verificado os atributos maxOfPayments1, maxOfPayments2, maxOfPayments3 e minValOfPayments da tabela productTable.|
|`Payment.Interest`|String|---|---|Default: `ByMerchant` <br><br> Enum: `ByMerchant` `ByIssuer` <br><br> Tipo de Parcelamento: <br><br> - Se o bit 6 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 6 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento sem juros pode ser efetuado. <br><br> - Se o bit 7 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 7 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento com juros pode ser efetuado. Sem juros = “ByMerchant”; Com juros = “ByIssuer”.|
|`Payment.Capture`|Booleano|---|---|Default: false / Booleano que identifica que a autorização deve ser com captura automática. A autorização sem captura automática é conhecida também como pré-autorização.|
|`Payment.ProductId`|Integer|---|Sim|Código do produto identificado através do bin do cartão.|
|`CreditCard.ExpirationDate`|String|MM/yyyy|Sim|Data de validade do cartão. <br><br>Dado obtido através do comando PP_GetCard na BC no momento da captura da transação.|
|`CreditCard.BrandId`|Integer|---|Sim|Identificação da bandeira obtida através do campo BrandId da PRODUCT TABLE.|
|`CreditCard.IssuerId`|Integer|---|Sim|Código do emissor obtido através do campo IssuerId da BIN TABLE.|
|`CreditCard.InputMode`|String|---|Sim|Enum: `Typed` `MagStripe` `Emv` <br><br>Identificação do modo de captura do cartão na transação. Essa informação deve ser obtida através do retorno da função PP_GetCard da BC. <br><br>"00" – Magnético <br><br>"01" - Moedeiro VISA Cash sobre TIBC v1 <br><br>"02" - Moedeiro VISA Cash sobre TIBC v3 <br><br>"03" – EMV com contato <br><br>"04" - Easy-Entry sobre TIBC v1 <br><br>"05" - Chip sem contato simulando tarja <br><br> "06" - EMV sem contato.|
|`CreditCard.AuthenticationMethod`|String|---|Sim|Enum: `NoPassword` `OnlineAuthentication` `OfflineAuthentication` <br><br>Método de autenticação <br><br>- Se o cartão foi lido a partir da digitação verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, a senha deve ser capturada e o authenticationMethod assume valor 2. Caso contrário, assume valor 1; <br><br>- Se o cartão foi lido a partir da trilha verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, deve ser verificado o bit 2 do mesmo campo. Se este estiver com valor 1 deve ser capturada a senha. Se estiver com valor 0 a captura da senha vai depender do último dígito do service code; <br><br>- Se o cartão foi lido através do chip EMV, o authenticationMethod será preenchido com base no retorno da função PP_GoOnChip(). No resultado PP_GoOnChip(), onde se o campo da posição 003 do retorno da PP_GoOnChip() estiver com valor 1 indica que o pin foi validado off-line, o authenticationMethod será 3. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valor 0, o authenticationMethod será 1. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valores 0 e 1 respectivamente, o authenticationMethod será 2. <br><br>1 - Sem senha = “NoPassword”; <br><br>2 - Senha online = “Online Authentication”; <br><br>3 - Senha off-line = “Offline Authentication”.|
|`CreditCard.EmvData`|String|---|---|Dados da transação EMV<br><br>Obtidos através do comando PP_GoOnChip na BC|
|`PinBlock.EncryptedPinBlock`|String|---|Sim|PINBlock Criptografado<br><br>- Para transações EMV, esse campo é obtido através do retorno da função PP_GoOnChip(), mais especificamente das posições 007 até a posição 022;<br><br>- Para transações digitadas e com tarja magnética, verificar as posições 001 até 016 do retorno da função PP_GetPin().|
|`PinBlock.EncryptionType`|String|---|Sim|Tipo de Criptografia<br><br>Enum:<br><br>"DukptDes"<br><br>"Dukpt3Des"<br><br>"MasterKey"|
|`PinBlock.KsnIdentification`|String|---|Sim|Identificação do KSN<br><br>- Para transações EMV esse campo é obtido através do retorno da função PP_GoOnChip() nas posições 023 até 042;<br><br>- Para transações digitadas e com tarja magnética, verificar as posições 017 até 036 do retorno da função PP_GetPin().|
|`CreditCard.PanSequenceNumber`|Number|---|---|Número sequencial do cartão, utilizado para identificar a conta corrente do cartão adicional. Mandatório para transações com cartões Chip EMV e que possuam PAN Sequence Number (Tag 5F34).|
|`CreditCard.SaveCard`|Booleano|---|---|Identifica se vai salvar/tokenizar o cartão.|
|`CreditCard.IsFallback`|Booleano|---|---|Identifica se é uma transação de fallback.|
|`PinPadInformation.TerminalId`|String|---|Sim|Número Lógico definido no Concentrador Cielo.|
|`PinPadInformation.SerialNumber`|String|---|Sim|Número de Série do Equipamento.|
|`PinPadInformation.PhysicalCharacteristics`|String|---|Sim|Enum: `WithoutPinPad` `PinPadWithoutChipReader` `PinPadWithChipReaderWithoutSamModule` `PinPadWithChipReaderWithSamModule` `NotCertifiedPinPad` `PinPadWithChipReaderWithoutSamAndContactless` `PinPadWithChipReaderWithSamModuleAndContactless` <br><br> Sem PIN-pad = `WithoutPinPad`; <br><br> PIN-pad sem leitor de Chip = `PinpadWithoutChipReader`; <br><br>PIN-pad com leitor de Chip sem módulo SAM = `PinPadWithChipReaderWithoutSamModule`; <br><br> PIN-pad com leitor de Chip com módulo SAM = `PinPadWithChipReaderWithSamModule`; <br><br> PIN-pad não homologado = `NotCertifiedPinPad`; <br><br> PIN-pad com leitor de Chip sem SAM e Cartão Sem Contato = `PinpadWithChipReaderWithoutSamAndContactless`; <br><br> PIN-pad com leitor de Chip com SAM e Cartão Sem Contato = `PinpadWithChipReaderWithSamAndContactless`. <br><br><br> Obs. Caso a aplicação não consiga informar os dados acima, deve obter tais informações através do retorno da função PP_GetInfo() da BC.|
|`PinPadInformation.ReturnDataInfo`|String|---|Sim|Retorno da função PP_GetInfo() da biblioteca compartilhada|
|`MassTransit.FirstTravelDate`|String|date-time|Sim|Data da primeira viagem|
|`MassTransit.IsKnownValue`|Booleano|-|Sim|Indica que a operação é de valor conhecido|
|`MassTransit.PaymentId`|String|-|Sim|Identificador do pagamento a ser recuperado|
|`MassTransit.IsDebtRecovery`|Booleano|—|Sim|Identifica de a operação é de recuperação de pagamento.|

#### Resposta

```json
{
   "MerchantOrderId": "1593196287377",
   "Customer": {
      "Name": "[Guest]"
   },
   "Payment": {
      "Installments": 1,
      "Interest": "ByMerchant",
      "Capture": true,
      "CreditCard": {
         "ExpirationDate": "12/2021",
         "BrandId": 1,
         "IssuerId": 401,
         "TruncateCardNumberWhenPrinting": true,
         "InputMode": "ContactlessEmv",
         "AuthenticationMethod": "OnlineAuthentication",
         "TrackOneData": "B3764 361234 56006^NOME NOME NOME NOME NOME N^0905060640431",
         "TrackTwoData": "1111222233334444=09050606404312376450",
         "EmvData": "",
         "IsFallback": false,
         "PinBlock": {
            "EncryptedPinBlock": "2280F6BDFD0C038D",
            "EncryptionType": "Dukpt3Des",
            "KsnIdentification": "fffff9999900522000d6"
         },
         "BrandInformation": {
            "Type": "c?s?jue?qz",
            "Name": "h?mbs{zxcc",
            "Description": "aleegncacoiqxl?qyc"
         },
         "SaveCard": false
      },
      "Amount": 300,
      "ReceivedDate": "2020-06-26T18:31:27Z",
      "CapturedAmount": 300,
      "CapturedDate": "2020-06-26T18:31:27Z",
      "Provider": "Cielo",
      "Status": 2,
      "IsSplitted": false,
      "ReturnMessage": "APROVADA 382242",
      "ReturnCode": "000",
      "PaymentId": "6a9e8136-3d63-4f60-8a23-394782c485e3",
      "Type": "PhysicalCreditCard",
      "Currency": "BRL",
      "Country": "BRA",
      "Links": [
         {
            "Method": "GET",
            "Rel": "self",
            "Href": "https://apiquerysandbox.cieloecommerce.cielo.com.br/1/physicalSales/6a9e8136-3d63-4f60-8a23-394782c485e3"
         },
         {
            "Method": "PUT",
            "Rel": "confirm",
            "Href": "https://apisandbox.cieloecommerce.cielo.com.br/1/physicalSales/6a9e8136-3d63-4f60-8a23-394782c485e3/confirmation"
         },
         {
            "Method": "DELETE",
            "Rel": "reverse",
            "Href": "https://apisandbox.cieloecommerce.cielo.com.br/1/physicalSales/6a9e8136-3d63-4f60-8a23-394782c485e3"
         }
      ],
      "PaymentDateTime": "2020-06-26T18:31:27.377Z",
      "ServiceTaxAmount": 0,
      "SoftDescriptor": "Description",
      "ProductId": 1,
      "PinPadInformation": {
         "TerminalId": "42004558",
         "SerialNumber": "6C651996",
         "PhysicalCharacteristics": "PinPadWithChipReaderWithSamModuleAndContactless",
         "ReturnDataInfo": "00"
      },
      "PrintMessage": [
         {
            "Position": "Top",
            "Message": "f??nzfbgks"
         },
         {
            "Position": "Middle",
            "Message": "s?kn?fcemf"
         },
         {
            "Position": "Bottom",
            "Message": "}zk}zet|?l"
         }
      ],
      "ReceiptInformation": [
         {
            "Field": "MERCHANT_NAME",
            "Label": "NOME DO ESTABELECIMENTO",
            "Content": "gtevvj~a?otlubavsxm"
         },
         {
            "Field": "MERCHANT_CITY",
            "Label": "CIDADE DO ESTABELECIMENTO",
            "Content": "??dors|yk?elu"
         },
         {
            "Field": "INPUT_METHOD",
            "Label": "MODO DE ENTRADA",
            "Content": "?uhtrrrc~pe?rudlt?l}c?|w?phds|v"
         },
         {
            "Field": "TERMINAL",
            "Label": "POS",
            "Content": "54206393"
         },
         {
            "Field": "ISSUER_NAME",
            "Label": "EMISSOR",
            "Content": "qpqm?pf{kpefzfu{"
         },
         {
            "Field": "NSU",
            "Label": "DOC",
            "Content": "729860"
         },
         {
            "Field": "MERCHANT_CODE",
            "Label": "COD.ESTAB.",
            "Content": "03388928143200"
         },
         {
            "Field": "MERCHANT_ADDRESS",
            "Label": "ENDEREÇO DO ESTABELECIMENTO",
            "Content": "hqqdy|fxb?ltdakwm{c?vjj??w?nxwo?nltm"
         },
         {
            "Field": "AUTHORIZATION_CODE",
            "Label": "AUTORIZAÇÃO",
            "Content": "76715"
         },
         {
            "Field": "CARD_HOLDER",
            "Label": "NOME DO CLIENTE",
            "Content": "~zm?wfk|ecbn?~cdb{xyvba?g?mankfa?"
         },
         {
            "Field": "TRANSACTION_TYPE",
            "Label": "TIPO DE TRANSAÇÃO",
            "Content": "?kx|uh?p??cohiakmvgcm??cmucjl"
         },
         {
            "Field": "MERCHANT_STATE",
            "Label": "ESTADO DO ESTABELECIMENTO",
            "Content": "xo"
         },
         {
            "Field": "DATE",
            "Label": "DATA",
            "Content": "6/26/2020"
         },
         {
            "Field": "HOUR",
            "Label": "HORA",
            "Content": "3:31 PM"
         },
         {
            "Field": "VALUE",
            "Label": "VALOR",
            "Content": "300"
         },
         {
            "Field": "TRANSACTION_MODE",
            "Label": "MODO DA TRANSAÇÃO",
            "Content": "prt"
         },
         {
            "Field": "CARD_NUMBER",
            "Label": "CARTÃO"
         }
      ],
      "Receipt": {
         "MerchantName": "gtevvj~a?otlubavsxmhxmfetnpcct?k?m",
         "MerchantCity": "??dors|yk?eluwv{?h",
         "InputMethod": "?uhtrrrc~pe?rudlt?l}c?|w?phds|vyv{nj?wrpe",
         "Terminal": "54206393",
         "IssuerName": "qpqm?pf{kpefzfu??u~ftpssck|o??a|??b",
         "Nsu": "729860",
         "MerchantCode": "03388928143200",
         "MerchantAddress": "hqqdy|fxb?ltdakwm{c?vjj??w?nxwo?nltm|?g?",
         "AuthorizationCode": "76715",
         "CardHolder": "~zm?wfk|ecbn?~cdb{xyvba?g?mankfa??{|~?",
         "TransactionType": "?kx|uh?p??cohiakmvgcm??cmucjlcg{xmyrwvjyecefi?",
         "MerchantState": "xo",
         "Date": "6/26/2020",
         "Hour": "3:31 PM",
         "Value": "300",
         "TransactionMode": "prt",
         "CardNumber": null
      },
      "AuthorizationCode": "382242",
      "ProofOfSale": "876502",
      "InitializationVersion": 1593196200000,
      "ConfirmationStatus": 0,
      "EmvResponseData": "290293329",
      "SubordinatedMerchantId": "b99a463f-88db-442a-b5fa-982187b68f5c",
      "MassTransit": {
         "IsDebtRecovery": true,
         "IsKnownValue": true,
         "FirstTravelDate": "2018-01-01T04:12:00"
      },
      "OfflinePaymentType": "Online"
   }
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`MerchantOrderId`|String|---|---| Número do documento gerado automaticamente pelo terminal e incrementado de 1 a cada transação realizada no terminal. Aceita apenas valores numéricos de 1 a 15 dígitos.|
|`Customer.Name`|String|---|---|---|
|`Customer.Identity`|---|---|---|---|
|`Customer.IdentityType`|---|---|---|---|
|`Customer.Email`|---|---|---|---|
|`Customer.Birthday`|---|---|---|---|
|`Address.Street`|String|---|---|---|
|`Address.Number`|String|---|---|---|
|`Address.Complement`|String|---|---||---|
|`Address.ZipCode`|String|---|---|---|
|`Address.City`|String|---|---|---|
|`Address.State`|String|---|---|---|
|`Address.Country`|String|---|---|---|
|`DeliveryAddress.Street`|String|---|---|---|
|`DeliveryAddress.Number`|String|---|---|---|
|`DeliveryAddress.Complement`|String|---|---||---|
|`DeliveryAddress.ZipCode`|String|---|---|---|
|`DeliveryAddress.City`|String|---|---|---|
|`DeliveryAddress.State`|String|---|---|---|
|`DeliveryAddress.Country`|String|---|---|---|
|`Payment.Installments`|Integer|---|---|Default: 1 / Quantidade de Parcelas: Varia de 2 a 99 para transação de financiamento. Deve ser verificado os atributos maxOfPayments1, maxOfPayments2, maxOfPayments3 e minValOfPayments da tabela productTable.|
|`Payment.Interest`|String|---|---|Default: `ByMerchant` <br><br> Enum: `ByMerchant` `ByIssuer` <br><br> Tipo de Parcelamento: <br><br> - Se o bit 6 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 6 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento sem juros pode ser efetuado. <br><br> - Se o bit 7 do atributo confParamOp05, presente nas tabelas issuerTable e binTable e bit 7 do atributo confParamOp03 da tabela productTable estiverem todos habilitados indica que o tipo de parcelamento com juros pode ser efetuado. Sem juros = “ByMerchant”; Com juros = “ByIssuer”.|
|`Payment.Capture`|Booleano|---|---|Default: false / Booleano que identifica que a autorização deve ser com captura automática. A autorização sem captura automática é conhecida também como pré-autorização.|
|`CreditCard.ExpirationDate`|String|MM/yyyy|Sim|Data de validade do cartão. <br><br>Dado obtido através do comando PP_GetCard na BC no momento da captura da transação.|
|`CreditCard.BrandId`|Integer|---|Sim|Identificação da bandeira obtida através do campo BrandId da PRODUCT TABLE.|
|`CreditCard.IssuerId`|Integer|---|Sim|Código do emissor obtido através do campo IssuerId da BIN TABLE.|
|`CreditCard.TruncateCardNumberWhenPrinting`|Booleano|---|---|Indica se o número do cartão será truncado no momento da impressão do comprovante. A solução de captura deve tomar essa decisão com base no `confParamOp03` presente nas tabelas BIN TABLE, PARAMETER TABLE e ISSUER TABLE.|
|`CreditCard.InputMode`|String|---|Sim|Enum: `Typed` `MagStripe` `Emv` <br><br>Identificação do modo de captura do cartão na transação. Essa informação deve ser obtida através do retorno da função PP_GetCard da BC. <br><br>"00" – Magnético <br><br>"01" - Moedeiro VISA Cash sobre TIBC v1 <br><br>"02" - Moedeiro VISA Cash sobre TIBC v3 <br><br>"03" – EMV com contato <br><br>"04" - Easy-Entry sobre TIBC v1 <br><br>"05" - Chip sem contato simulando tarja <br><br> "06" - EMV sem contato.|
|`CreditCard.AuthenticationMethod`|String|---|Sim|Enum: `NoPassword` `OnlineAuthentication` `OfflineAuthentication` <br><br>Método de autenticação <br><br>- Se o cartão foi lido a partir da digitação verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, a senha deve ser capturada e o authenticationMethod assume valor 2. Caso contrário, assume valor 1; <br><br>- Se o cartão foi lido a partir da trilha verificar o bit 3 do atributo confParamOp04 das tabelas binTable, parameterTable e issuerTable. Se todos estiverem habilitados, deve ser verificado o bit 2 do mesmo campo. Se este estiver com valor 1 deve ser capturada a senha. Se estiver com valor 0 a captura da senha vai depender do último dígito do service code; <br><br>- Se o cartão foi lido através do chip EMV, o authenticationMethod será preenchido com base no retorno da função PP_GoOnChip(). No resultado PP_GoOnChip(), onde se o campo da posição 003 do retorno da PP_GoOnChip() estiver com valor 1 indica que o pin foi validado off-line, o authenticationMethod será 3. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valor 0, o authenticationMethod será 1. Se o campo da posição 003 e o campo da posição 006 do retorno da PP_GoOnChip() estiverem com valores 0 e 1 respectivamente, o authenticationMethod será 2. <br><br>1 - Sem senha = “NoPassword”; <br><br>2 - Senha online = “Online Authentication”; <br><br>3 - Senha off-line = “Offline Authentication”.|
|`CreditCard.EmvData`|String|---|---|Dados da transação EMV <br><br>Obtidos através do comando PP_GoOnChip na BC|
|`PinBlock.EncryptedPinBlock`|---|---|---|---|
|`PinBlock.EncryptionType`|String|---|---|---|
|`PinBlock.KsnIdentification`|String|---|---|---|
|`CreditCard.PanSequenceNumber`|Number|---|---|Número sequencial do cartão, utilizado para identificar a conta corrente do cartão adicional. Mandatório para transações com cartões Chip EMV e que possuam PAN Sequence Number (Tag 5F34).|
|`CreditCard.SaveCard`|---|---|---|---|
|`CreditCard.IsFallback`|---|---|---|---|
|`Payment.PaymentDateTime`|String|date-time|Sim|Data e Hora da captura da transação|
|`Payment.ServiceTaxAmount`|---|---|---|---|
|`Payment.SoftDescriptor`|String|13|---|Identificação do estabelecimento (nome reduzido) a ser impresso e identificado na fatura.|
|`Payment.ProductId`|Integer|---|Sim|Código do produto identificado através do bin do cartão.|
|`PinPadInformation.TerminalId`|String|---|Sim|Número Lógico definido no Concentrador Cielo.|
|`PinPadInformation.SerialNumber`|String|---|Sim|Número de Série do Equipamento.|
|`PinPadInformation.PhysicalCharacteristics`|String|---|Sim|Enum: `WithoutPinPad` `PinPadWithoutChipReader` `PinPadWithChipReaderWithoutSamModule` `PinPadWithChipReaderWithSamModule` `NotCertifiedPinPad` `PinPadWithChipReaderWithoutSamAndContactless` `PinPadWithChipReaderWithSamModuleAndContactless` <br><br> Sem PIN-pad = `WithoutPinPad`; <br><br> PIN-pad sem leitor de Chip = `PinpadWithoutChipReader`; <br><br>PIN-pad com leitor de Chip sem módulo SAM = `PinPadWithChipReaderWithoutSamModule`; <br><br> PIN-pad com leitor de Chip com módulo SAM = `PinPadWithChipReaderWithSamModule`; <br><br> PIN-pad não homologado = `NotCertifiedPinPad`; <br><br> PIN-pad com leitor de Chip sem SAM e Cartão Sem Contato = `PinpadWithChipReaderWithoutSamAndContactless`; <br><br> PIN-pad com leitor de Chip com SAM e Cartão Sem Contato = `PinpadWithChipReaderWithSamAndContactless`. <br><br><br> Obs. Caso a aplicação não consiga informar os dados acima, deve obter tais informações através do retorno da função PP_GetInfo() da BC.|
|`PinPadInformation.ReturnDataInfo`|String|---|Sim|Retorno da função PP_GetInfo() da biblioteca compartilhada|
|`Payment.Amount`|Integer(int64)|---|Sim|Valor da transação (1079 = R$10,79)|
|`Payment.ReceivedDate`|---|---|---|---|
|`Payment.CapturedAmount`|---|---|---|---|
|`Payment.Provider`|String|---|---|---|
|`Payment.ConfirmationStatus`|---|---|---|---|
|`Payment.InitializationVersion`|---|---|---|---|
|`Payment.EmvResponseData`|---|---|---|---|
|`Payment.Status`|---|---|---|---|
|`Payment.IsSplitted`|Booleano|---|---|---|
|`Payment.ReturnCode`|---|---|---|---|
|`Payment.ReturnMessage`|String|---|---|---|
|`Payment.PaymentId`|---|---|---|---|
|`Payment.Type`|String|---|Sim|Value: `PhysicalCreditCard` / Tipo da Transação|
|`Payment.Currency`|String|---|---|Default: "BRL" / Value: "BRL" / Moeda (Preencher com “BRL”)|
|`Payment.Country`|String|---|---|Default: "BRA" / Value: "BRA" / País (Preencher com “BRA”)|
|`Receipt.MerchantName`|---|---|---|---|
|`Receipt.MerchantAddress`|---|---|---|---|
|`Receipt.MerchantCity`|---|---|---|---|
|`Receipt.MerchantState`|---|---|---|---|
|`Receipt.MerchantCode`|---|---|---|---|
|`Receipt.Terminal`|---|---|---|---|
|`Receipt.Nsu`|---|---|---|---|
|`Receipt.Date`|---|---|---|---|
|`Receipt.Hour`|---|---|---|---|
|`Receipt.IssuerName`|---|---|---|---|
|`Receipt.CardNumber`|---|---|---|---|
|`Receipt.TransactionType`|---|---|---|---|
|`Receipt.AuthorizationCode`|---|---|---|---|
|`Receipt.TransactionMode`|---|---|---|---|
|`Receipt.InputMethod`|---|---|---|---|
|`Receipt.Value`|---|---|---|---|
|`Receipt.SoftDescriptor`|---|---|---|---|
|`RecurrentPayment.RecurrentPaymentId`|---|---|---|---|
|`RecurrentPayment.ReasonCode`|---|---|---|---|
|`RecurrentPayment.ReasonMessage`|---|---|---|---|
|`RecurrentPayment.NextRecurrency`|---|---|---|---|
|`RecurrentPayment.EndDate`|---|---|---|---|
|`RecurrentPayment.Interval`|---|---|---|---|
|`SplitPayments.SubordinateMerchantId`|---|---|---|---|
|`SplitPayments.Amount`|---|---|---|---|
|`SplitPayments.Fares.Mdr`|---|---|---|---|
|`SplitPayments.Fares.Fee`|---|---|---|---|
|`SplitErrors.Code`|---|---|---|---|
|`SplitErrors.Message`|---|---|---|---|

## Desfazimento

Em casos de timeout ou retorno com erro que não deixe claro a negativa da transação deve-se solicitar o desfazimento do pagamento. Outras situações como por exemplo a não impressão do comprovante no POS também podem gerar a necessidade desta operação.

|**URL**|https://apisandbox.cieloecommerce.cielo.com.br/1/physicalSales|
|**Scope:**|PhysicalCieloTransactional|

**Simular respostas**:

Para simular alguma resposta especifica utilize o campo Amount, onde de acordo com o valor dos centavos informado nesse campo é possivel receber uma resposta conforme descrito na tabela abaixo:

|Amount (valor dos centados)|Retorno simulado do Desfazimento|Exemplo de valor simulado|
|---|---|---|
|20|Aprovado|5020 = R$50,20|
|21|Negado|20021 = R$200,21|
|22|Timeout|3522 = R$35,22|
|29|Erro|1029 = R$10,29|

### Desfazimento por PaymentId

Quando o pagamento retorna ele devolve o PaymentId em seu corpo e este é o melhor identificador do pagamento e pode ser utilizado para solicitar seu desfazimento.

#### Cartao digitado.

O pagamento retornou com sucesso e pode ser desfeito.

Deve-se solicitar o desfazimento através do PaymentId recebido no retorno do pagamento. 

##### Requisição

<aside class="request"><span class="method put">PUT</span> <span class="endpoint">/1/physicalSales/{PaymentId}/undo</span></aside>

**Path Parameters:**

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`PaymentId`|String uuid|---|Sim|Código do Pagamento|

##### Resposta

```json
{
  "ConfirmationStatus": 2,
  "Status": 2,
  "ReturnCode": 0,
  "Links": [
    {
      "Method": "GET",
      "Rel": "self",
      "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702"
    }
  ]
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`ConfirmationStatus`|Integer int16|---|---|Status da confirmação. <br><br>0 = Pendente <br><br>1 = Confirmado <br><br>2 = Desfeito|
|`Status`|Integer int16|---|---|Status da transação <br><br>0 = Não Finalizado <br><br>1 = Autorizado <br><br>2 = Pago <br><br>3 = Negado <br><br>10 = Cancelado<br><br>13 = Abortado|
|`ReasonCode`|Integer(int16)|---|---|Código de referência para análises.|
|`ReasonMessage`|String|---|---|Mensagem explicativa para análise.|
|`Links.Method`|String|---|---|Enum: "POST", "GET", "PUT".<br><br>Método HTTP a ser utilizado na operação.|
|`Links.Rel`|String|---|---|Enum: "self", "cancel", "confirm".<br><br>Referência da operação.|
|`Links.Href`|String|---|---|Endereço de URL de chamada da API|

#### Cartao com chip

O pagamento retornou com sucesso e pode ser desfeito.

Deve-se solicitar o desfazimento através do PaymentId recebido no retorno do pagamento. 

##### Requisição

<aside class="request"><span class="method put">PUT</span> <span class="endpoint">/1/physicalSales/{PaymentId}/undo</span></aside>

**Path Parameters:**

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`PaymentId`|String uuid|---|Sim|Código do Pagamento|

```json
{
  "EmvData": "112233445566778899011AABBC012D3456789E0123FF45678AB901234C5D112233445566778800",
  "IssuerScriptsResults": "0000"
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`EmvData`|String|---|---|Dados da transação EMV <br><br>Dados obtidos através do comando PP_GoOnChip na BC|
|`IssuerScriptsResults`|String|---|---|Resultado dos scripts EMV do emissor|

##### Resposta

```json
{
  "ConfirmationStatus": 2,
  "Status": 2,
  "ReturnCode": 0,
  "Links": [
    {
      "Method": "GET",
      "Rel": "self",
      "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702"
    }
  ]
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`ConfirmationStatus`|Integer int16|---|---|Status da confirmação. <br><br>0 = Pendente <br><br>1 = Confirmado <br><br>2 = Desfeito|
|`Status`|Integer int16|---|---|Status da transação <br><br>0 = Não Finalizado <br><br>1 = Autorizado <br><br>2 = Pago <br><br>3 = Negado <br><br>10 = Cancelado<br <br>13 = Abortado|
|`ReasonCode`|Integer(int16)|---|---|Código de referência para análises.|
|`ReasonMessage`|String|---|---|Mensagem explicativa para análise.|
|`Links.Method`|String|---|---|Enum: "POST", "GET", "PUT".<br><br>Método HTTP a ser utilizado na operação.|
|`Links.Rel`|String|---|---|Enum: "self", "cancel", "confirm".<br><br>Referência da operação.|
|`Links.Href`|String|---|---|Endereço de URL de chamada da API|

### Desfazimento por OrderId

#### Cartão digitado

Quando o pagamento não retornar, o mesmo deve ser desfeito.

Para solicitar o desfazimento é necessário informar o MerchantOrderId enviado no pagamento.

##### Requisição

<aside class="request"><span class="method put">PUT</span> <span class="endpoint">/1/physicalSales/orderId/{MerchantOrderId}/undo</span></aside>

**Path Parameters:**

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`MerchantOrderId`|String|—|Sim|Número do documento gerado automáticamente pelo terminal e incrementado de 1 acada transação realizada no terminal. Aceita apenas valores numéricos de 1 a 15 dígitos|

##### Resposta

```json
{
  "ConfirmationStatus": 2,
  "Status": 2,
  "ReturnCode": 0,
  "ReturnMessage": "Success",
  "Links": [
    {
      "Method": "GET",
      "Rel": "self",
      "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702"
    }
  ]
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`ConfirmationStatus`|Integer int16|---|---|Status da confirmação. <br><br>0 = Pendente <br><br>1 = Confirmado <br><br>2 = Desfeito|
|`Status`|Integer int16|---|---|Status da transação <br><br>0 = Não Finalizado <br><br>1 = Autorizado <br><br>2 = Pago <br><br>3 = Negado <br><br>10 = Cancelado<br><br>13 = Abortado|
|`ReasonCode`|Integer(int16)|---|---|Código de referência para análises.|
|`ReasonMessage`|String|---|---|Mensagem explicativa para análise.|
|`Links.Method`|String|---|---|Enum: "POST", "GET", "PUT".<br><br>Método HTTP a ser utilizado na operação.|
|`Links.Rel`|String|---|---|Enum: "self", "cancel", "confirm".<br><br>Referência da operação.|
|`Links.Href`|String|---|---|Endereço de URL de chamada da API|

#### Cartão com chip

Quando o pagamento não retornar, o mesmo deve ser desfeito.

Para solicitar o desfazimento é necessário informar o MerchantOrderId enviado no pagamento.

##### Requisição

<aside class="request"><span class="method put">PUT</span> <span class="endpoint">/1/physicalSales/orderId/{MerchantOrderId}/undo</span></aside>

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`MerchantOrderId`|String|—|Sim|Número do documento gerado automáticamente pelo terminal e incrementado de 1 acada transação realizada no terminal. Aceita apenas valores numéricos de 1 a 15 dígitos.|

```json
{
  "EmvData": "112233445566778899011AABBC012D3456789E0123FF45678AB901234C5D112233445566778800",
  "IssuerScriptsResults": "0000"
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`EmvData`|String|---|---|Dados da transação EMV <br><br>Dados obtidos através do comando PP_GoOnChip na BC|
|`IssuerScriptsResults`|String|---|---|Resultado dos scripts EMV do emissor|

##### Resposta

```json
{
  "ConfirmationStatus": 2,
  "Status": 2,
  "ReturnCode": 0,
  "ReturnMessage": "Success",
  "Links": [
    {
      "Method": "GET",
      "Rel": "self",
      "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702"
    }
  ]
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`ConfirmationStatus`|Integer int16|---|---|Status da confirmação. <br><br>0 = Pendente <br><br>1 = Confirmado <br><br>2 = Desfeito|
|`Status`|Integer int16|---|---|Status da transação <br><br>0 = Não Finalizado <br><br>1 = Autorizado <br><br>2 = Pago <br><br>3 = Negado <br><br>10 = Cancelado<br><br>13 = Abortado|
|`ReasonCode`|Integer(int16)|---|---|Código de referência para análises.|
|`ReasonMessage`|String|---|---|Mensagem explicativa para análise.|
|`Links.Method`|String|---|---|Enum: "POST", "GET", "PUT".<br><br>Método HTTP a ser utilizado na operação.|
|`Links.Rel`|String|---|---|Enum: "self", "cancel", "confirm".<br><br>Referência da operação.|
|`Links.Href`|String|---|---|Endereço de URL de chamada da API|

## Confirmação

Quando o pagamento retornar sucesso e pode ser confirmado.

Esta operação requer o PaymentId recebido no retorno do pagamento, além dos dados EmvData se o pagamento foi realizado atráves de Chip.

A confirmação somente é necessária para pagamentos feitos através do POS.

| SandBox                                             | Produção                                      |
|:---------------------------------------------------:|:---------------------------------------------:|
| https://apisandbox.cieloecommerce.cielo.com.br      | https://api.cieloecommerce.cielo.com.br/      |

**Simular Respostas:**

Para simular alguma resposta especifica utilize o campo Amount, onde de acordo com o valor dos centavos informado nesse campo é possivel receber uma resposta conforme descrito na tabela abaixo:

|Amount(valor dos centados)|Retorno simulado da Confirmação|Exemplo de valor simulado|
|---|---|---|
|20|Aprovado|5020 = R$50,20|
|21|Negado|20021 = R$200,21|
|22|Timeout|3522 = R$35,22|
|29|Erro|1029 = R$10,29|

Também é possivel simular respostas de uma confirmação de autorização negada. Para deixar claro esse fluxo segue os passos abaixo:
1 - Cliente faz uma requisição de autorização e recebe a resposta de autorização negada;
2 - Cliente faz uma requisição de confirmação para essa autorização negada e recebe a resposta simulada.

Para simular respostas de uma confirmação de autorização negada, segue a tabela abaixo:

|Amount(valor dos centados)|Retorno simulado da Confirmação|Exemplo de valor simulado|
|---|---|---|
|30|Aprovado|5030 = R$50,30|
|31|Negado|20031 = R$200,31|
|32|Timeout|3532 = R$35,32|
|39|Erro|1039 = R$10,39|

### Cartao Digitado

#### Requisição

<aside class="request"><span class="method put">PUT</span> <span class="endpoint">/1/physicalSales/{PaymentId}/confirmation</span></aside>

**Path Parameters:**

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`PaymentId`|String uuid|---|Sim|Código do Pagamento|

```json
null
```

#### Resposta

```json
{
  "ConfirmationStatus": 1,
  "Status": 2,
  "ReturnCode": 0,
  "ReturnMessage": "Successful",
  "Links": [
    {
      "Method": "GET",
      "Rel": "self",
      "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702"
    },
    {
      "Method": "POST",
      "Rel": "void",
      "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702/voids"
    }
  ]
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`ConfirmationStatus`|Integer int16|---|---|Status da confirmação. <br><br>0 = Pendente <br><br>1 = Confirmado <br><br>2 = Desfeito|
|`Status`|Integer int16|---|---|Status da transação <br><br>0 = Não Finalizado <br><br>1 = Autorizado <br><br>2 = Pago <br><br>3 = Negado <br><br>10 = Cancelado <br><br>13 = Abortado|
|`ReturnCode`|String|---|---|Código de erro/resposta da transação da Adquirência.|
|`ReturnMessage`|String|---|---|Mensagem de erro/resposta da transação da Adquirência|
|`Links.Method`|String|---|---|Enum: "POST", "GET", "PUT".<br><br>Método HTTP a ser utilizado na operação.|
|`Links.Rel`|String|---|---|Enum: "self", "cancel", "confirm".<br><br>Referência da operação.|
|`Links.Href`|String|---|---|Endereço de URL de chamada da API|

### Cartao com chip

#### Requisição

<aside class="request"><span class="method put">PUT</span> <span class="endpoint">/1/physicalSales/{PaymentId}/confirmation</span></aside>

**Path Parameters:**

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`PaymentId`|String uuid|---|Sim|Código do Pagamento|

```json
{
  "EmvData": "112233445566778899011AABBC012D3456789E0123FF45678AB901234C5D112233445566778800",
  "IssuerScriptResults": "0000"
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`EmvData`|String|---|---|Dados da transação EMV <br><br>Dados obtidos através do comando PP_GoOnChip na BC|
|`IssuerScriptResults`|String|---|---|Resultado dos scripts EMV do emissor|

#### Resposta

```json
{
  "ConfirmationStatus": 1,
  "Status": 2,
  "ReturnCode": 0,
  "ReturnMessage": "Successful",
  "Links": [
    {
      "Method": "GET",
      "Rel": "self",
      "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702"
    },
    {
      "Method": "POST",
      "Rel": "void",
      "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702/voids"
    }
  ]
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`ConfirmationStatus`|Integer int16|---|---|Status da confirmação. <br><br>0 = Pendente <br><br>1 = Confirmado <br><br>2 = Desfeito|
|`Status`|Integer int16|---|---|Status da transação <br><br>0 = Não Finalizado <br><br>1 = Autorizado <br><br>2 = Pago <br><br>3 = Negado <br><br>10 = Cancelado <br><br>13 = Abortado|
|`ReturnCode`|String|---|---|Código de erro/resposta da transação da Adquirência.|
|`ReturnMessage`|String|---|---|Mensagem de erro/resposta da transação da Adquirência|
|`Links.Method`|String|---|---|Enum: "POST", "GET", "PUT".<br><br>Método HTTP a ser utilizado na operação.|
|`Links.Rel`|String|---|---|Enum: "self", "cancel", "confirm".<br><br>Referência da operação.|
|`Links.Href`|String|---|---|Endereço de URL de chamada da API|

## Consultas

### Consulta de Pagamento

### Requisição

<aside class="request"><span class="method get">GET</span> <span class="endpoint">/1/physicalSales/{PaymentId}</span></aside>

**Path Parameters:**

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`PaymentId`|String|uuid|—|Sim|Código do Pagamento|

### Resposta

```json
{
  "MerchantOrderId": "20180204",
  "Customer": {
    "Name": "[Guest]"
  },
  "Payment": {
    "Installments": 1,
    "Interest": "ByMerchant",
    "CreditCard": {
      "ExpirationDate": "12/2020",
      "BrandId": 1,
      "IssuerId": 2,
      "TruncateCardNumberWhenPrinting": true,
      "InputMode": "Emv",
      "AuthenticationMethod": "OnlineAuthentication",
      "EmvData": "112233445566778899011AABBC012D3456789E0123FF45678AB901234C5D112233445566778800",
      "PinBlock": {
        "EncryptedPinBlock": "2280F6BDFD0C038D",
        "EncryptionType": "Dukpt3Des",
        "KsnIdentification": "1231vg31fv231313123"
      }
    },
    "PaymentDateTime": "2019-04-15T12:00:00Z",
    "ServiceTaxAmount": 0,
    "SoftDescriptor": "Description",
    "ProductId": 1,
    "PinPadInformation": {
      "TerminalId": "10000001",
      "SerialNumber": "ABC123",
      "PhysicalCharacteristics": "PinPadWithChipReaderWithSamModule",
      "ReturnDataInfo": "00"
    },
    "Amount": 15798,
    "ReceivedDate": "2019-04-15T12:00:00Z",
    "CapturedAmount": 15798,
    "Provider": "Cielo",
    "ConfirmationStatus": 0,
    "InitializationVersion": 1558708320029,
    "EmvResponseData": "123456789ABCD1345DEA",
    "Status": 2,
    "IsSplitted": false,
    "ReturnCode": 0,
    "ReturnMessage": "Successful",
    "PaymentId": "f15889ea-5719-4e1a-a2da-f4e50d5bd702",
    "Type": "PhysicalDebitCard",
    "Currency": "BRL",
    "Country": "BRA",
    "Links": [
      {
        "Method": "GET",
        "Rel": "self",
        "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702"
      },
      {
        "Method": "DELETE",
        "Rel": "self",
        "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702"
      },
      {
        "Method": "PUT",
        "Rel": "self",
        "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702/confirmation"
      }
    ],
    "PrintMessage": [
      {
        "Position": "Top",
        "Message": "Transação autorizada"
      },
      {
        "Position": "Bottom",
        "Message": "Obrigado e volte sempre!"
      }
    ],
    "ReceiptInformation": [
      {
        "Field": "MERCHANT_NAME",
        "Label": "NOME DO ESTABELECIMENTO",
        "Content": "Estabelecimento"
      },
      {
        "Field": "MERCHANT_ADDRESS",
        "Label": "ENDEREÇO DO ESTABELECIMENTO",
        "Content": "Rua Sem Saida, 0"
      },
      {
        "Field": "MERCHANT_CITY",
        "Label": "CIDADE DO ESTABELECIMENTO",
        "Content": "Cidade"
      },
      {
        "Field": "MERCHANT_STATE",
        "Label": "ESTADO DO ESTABELECIMENTO",
        "Content": "WA"
      },
      {
        "Field": "MERCHANT_CODE",
        "Label": "COD.ESTAB.",
        "Content": 1234567890123456
      },
      {
        "Field": "TERMINAL",
        "Label": "POS",
        "Content": 12345678
      },
      {
        "Field": "NSU",
        "Label": "DOC",
        "Content": 123456
      },
      {
        "Field": "DATE",
        "Label": "DATA",
        "Content": "01/01/20"
      },
      {
        "Field": "HOUR",
        "Label": "HORA",
        "Content": "01:01"
      },
      {
        "Field": "ISSUER_NAME",
        "Label": "EMISSOR",
        "Content": "NOME DO EMISSOR"
      },
      {
        "Field": "CARD_NUMBER",
        "Label": "CARTÃO",
        "Content": 5432123454321234
      },
      {
        "Field": "TRANSACTION_TYPE",
        "Label": "TIPO DE TRANSAÇÃO",
        "Content": "VENDA A CREDITO"
      },
      {
        "Field": "AUTHORIZATION_CODE",
        "Label": "AUTORIZAÇÃO",
        "Content": 123456
      },
      {
        "Field": "TRANSACTION_MODE",
        "Label": "MODO DA TRANSAÇÃO",
        "Content": "ONL"
      },
      {
        "Field": "INPUT_METHOD",
        "Label": "MODO DE ENTRADA",
        "Content": "X"
      },
      {
        "Field": "VALUE",
        "Label": "VALOR",
        "Content": "1,23"
      },
      {
        "Field": "SOFT_DESCRIPTOR",
        "Label": "SOFT DESCRIPTOR",
        "Content": "Simulado"
      }
    ],
    "Receipt": {
      "MerchantName": "Estabelecimento",
      "MerchantAddress": "Rua Sem Saida, 0",
      "MerchantCity": "Cidade",
      "MerchantState": "WA",
      "MerchantCode": 1234567890123456,
      "Terminal": 12345678,
      "Nsu": 123456,
      "Date": "01/01/20",
      "Hour": "01:01",
      "IssuerName": "NOME DO EMISSOR",
      "CardNumber": 5432123454321234,
      "TransactionType": "VENDA A CREDITO",
      "AuthorizationCode": 123456,
      "TransactionMode": "ONL",
      "InputMethod": "X",
      "Value": "1,23",
      "SoftDescriptor": "Simulado"
    }
  }
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`MerchantOrderId`|---|---|---|---|
|`Customer.Name`|---|---|---|---|
|`Payment.Installments`|---|---|---|---|
|`Payment.Interest`|---|---|---|---|
|`CreditCard.ExpirationDate`|---|---|---|---|
|`CreditCard.BrandId`|---|---|---|---|
|`CreditCard.IssuerId`|---|---|---|---|
|`CreditCard.TruncateCardNumberWhenPrinting`|---|---|---|---|
|`CreditCard.InputMode`|---|---|---|---|
|`CreditCard.AuthenticationMethod`|---|---|---|---|
|`CreditCard.EmvData`|---|---|---|---|
|`PinBlock.EncryptedPinBlock`|---|---|---|---|
|`PinBlock.EncryptionType`|---|---|---|---|
|`PinBlock.KsnIdentification`|---|---|---|---|
|`Payment.PaymentDateTime`|---|---|---|---|
|`Payment.ServiceTaxAmount`|---|---|---|---|
|`Payment.SoftDescriptor`|---|---|---|---|
|`Payment.ProductId`|---|---|---|---|
|`PinPadInformation.TerminalId`|---|---|---|---|
|`PinPadInformation.SerialNumber`|---|---|---|---|
|`PinPadInformation.PhysicalCharacteristics`|---|---|---|---|
|`PinPadInformation.ReturnDataInfo`|---|---|---|---|
|`Payment.Amount`|---|---|---|---|
|`Payment.ReceivedDate`|---|---|---|---|
|`Payment.CapturedAmount`|---|---|---|---|
|`Payment.Provider`|---|---|---|---|
|`Payment.ConfirmationStatus`|---|---|---|---|
|`Payment.InitializationVersion`|---|---|---|---|
|`Payment.EmvResponseData`|---|---|---|---|
|`Payment.Status`|---|---|---|---|
|`Payment.IsSplitted`|---|---|---|---|
|`Payment.ReturnCode`|---|---|---|---|
|`Payment.ReturnMessage`|---|---|---|---|
|`Payment.PaymentId`|---|---|---|---|
|`Payment.Type`|---|---|---|---|
|`Payment.Currency`|---|---|---|---|
|`Payment.Country`|---|---|---|---|
|`Receipt.MerchantName`|---|---|---|---|
|`Receipt.MerchantAddress`|---|---|---|---|
|`Receipt.MerchantCity`|---|---|---|---|
|`Receipt.MerchantState`|---|---|---|---|
|`Receipt.MerchantCode`|---|---|---|---|
|`Receipt.Terminal`|---|---|---|---|
|`Receipt.Nsu`|---|---|---|---|
|`Receipt.Date`|---|---|---|---|
|`Receipt.Hour`|---|---|---|---|
|`Receipt.IssuerName`|---|---|---|---|
|`Receipt.CardNumber`|---|---|---|---|
|`Receipt.TransactionType`|---|---|---|---|
|`Receipt.AuthorizationCode`|---|---|---|---|
|`Receipt.TransactionMode`|---|---|---|---|
|`Receipt.InputMethod`|---|---|---|---|
|`Receipt.Value`|---|---|---|---|
|`Receipt.SoftDescriptor`|---|---|---|---|

### Cancelamento

Consulta um cancelamento

#### Requisição

<aside class="request"><span class="method get">GET</span> <span class="endpoint">/1/physicalSales/{PaymentId}/voids/{VoidId}</span></aside>

#### Resposta

```json
{
  "VoidId": "a4bc7892-b455-4cd1-b902-c791802cd72b",
  "CancellationStatus": 1,
  "Status": 10
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`VoidId`|---|---|---|---|
|`CancellationStatus`|---|---|---|---|
|`Status`|---|---|---|---|

## Fluxo de pagamento (Biblioteca Compartilhada)

**Exemplo fluxo (Biblioteca Compartilhada):**

|ID|Descrição do Fluxo|
|---|---|
| 1 | Inserção do valor da transação (campo `Amount` do request da transação) |
| 2 | Recuperar Data/Hora da transação (campo `PaymentDateTime` do request da transação) |
| 3 | Seleção do tipo de pagamento (débito, crédito, voucher...) (campo `Type` do request da transação) |
| 4 | Chamada do PP_StartGetCard passando os valores: |
| 4.1 | Identificador da rede adquirente (Cielo `03`) |
| 4.2 | Tipo de aplicação (relacionado ao item 3) |
| 4.3 | Valor inicial da transação (item 1) |
| 4.4 | Data da transação (item 2) |
| 4.5 | Hora da transação (item 2) |
| 5 | Caso tenha sido utilizado um cartão com chip, recuperar o aid através da tag 4F no retorno da PP_getCard. |
| 6 | Seleção de produtos (campo "ProductId" do request da transação): 

**Transações com chip:**

|ID|Descrição do Fluxo|
|---|---|
|1| Realizar a busca na tabela "Emv" pelo AID do cartão (campo `Aid`) e selecionar os produtos associados através do campo `ProductIds` |
|2| Nos produtos associados, recuperar aqueles que possuem o mesmo `ProductType` (tabela `Products`) que iniciado na transação (DÉBITO, CRÉDITO..) e o mesmo fluxo do host (campo `HostFlow`) que os definido pela Cielo. |

**Transações com tarja/digitada:**

|ID|Descrição do Fluxo|
|---|---|
| 1 | Ao recuperar o pan do cartão, buscar na tabela `Bins` um que o bin esteja entre os valores `InitialBin` e `FinalBin` (considerar sempre a faixa de Bins mais específica) e recuperar o produto associado no campo `ProductId`; 
| 2 | Recuperar os produtos que tem o mesmo `ProductType` (tabela `Products`) que iniciado na transação (DÉBITO, CRÉDITO...) e o mesmo fluxo do host (campo `HostFlow`) que os definido pela Cielo.

**Simular respostas:**

|Amount (valor dos centavos)|Retorno simulado do Pagamento|Exemplo de valor simulado|
|---|---|---|
|10|Aprovado|5010 = R$50,10|
|11|Negado|20011 = R$200,11|
|12|Timeout|3512 = R$35,12|
|19|Erro|1019 = R$10,19|

# Baixa de parâmetros

Essa operação é necessária para que o parceiro de negócio / Subadquirente receba todas as tabelas de parâmetros necessários para que a solução de captura possa efetuar as transações via chamada de API. Essa informação será recebida através de API e deverá ser instalada na BC.

| SandBox                                             | Produção                                      |
|:---------------------------------------------------:|:---------------------------------------------:|
| https://parametersdownloadsandbox.cieloecommerce.cielo.com.br/api/v0.1      | https://parametersdownload.cieloecommerce.cielo.com.br/api/v0.1      |

## Baixa parâmetros de inicialização

Solicita as tabelas e parametros para operação do terminal

### Requisição

<aside class="request"><span class="method get">GET</span> <span class="endpoint">/initialization/{SubordinatedMerchantId}/{TerminalId}</span></aside>

### Resposta

```json
{
  "MerchantId": "string",
  "TerminalId": "string",
  "Acquirer": {
    "EnableContaclessCardReader": true,
    "LockAppFunctionsExceptInitialization": true,
    "HasChipReader": true,
    "HasMagneticTrackReader": true,
    "HasKeyboard": true
  },
  "Merchant": {
    "MerchantId": "string",
    "NetworkName": "string",
    "MerchantName": "string",
    "MerchantAddress": "string",
    "NationalId": "string"
  },
  "Bins": [
    {
      "InitialBin": "string",
      "FinalBin": "string",
      "ProductId": 0,
      "Type": 0,
      "AllowFallbackWhenChipReadingFails": true,
      "AllowChargingMoedeiroFromCash": true,
      "AllowPurchaseWithCompreESaque": true,
      "AllowOfflineFunctionExceptForEMVCard": true,
      "AllowTypingCardNumber": true,
      "MaskCardNumberUsingLast4Digits": true,
      "MaskCardNumberUsingFirst6AndLast4Digits": true,
      "AllowPrintCardHolderBalance": true,
      "AllowDisplayCardHolderBalance": true,
      "AllowPrintingPartialCardNumberInReceipt": true,
      "RestrictSaleWithDuplicateValueWhenPostdated": true,
      "RestrictSaleWithDuplicateValue": true,
      "RequiresPassword": true,
      "InterpretsLastDigitOfSecurityCode": true,
      "RequiresPasswordExceptForEMVCard": true,
      "EnableAdditionalSecurityCodeOptions_Unreadable_NoCode": true,
      "RequiresSecurityCodeWhenMagneticTrackIsRead": true,
      "RequiresSecurityCodeWhenCardNumberIsTyped": true,
      "RequiresTypingLast4Digits": true,
      "AllowCaptureOfFirstInstallmentValue": true,
      "AllowCaptureOfDownpaymentValue": true,
      "AllowGuaranteeHandling": true,
      "AllowPostdatingTheFirstInstallmentForSaleAndCDCQuery": true,
      "AllowPostdating": true,
      "AllowCDCSale": true,
      "AllowFinancingByStore": true,
      "AllowFinancingByCreditCardCompany": true,
      "ValidateCardTrack1": true,
      "DoNotValidateCardModule10": true,
      "CheckExpiryDateWhenCardNumberIsTyped": true,
      "CheckExpiryDateWhenMagneticTrackIsRead": true,
      "IssuerId": 0
    }
  ],
  "Products": [
    {
      "ProductId": 0,
      "ProductName": "string",
      "ProductType": 0,
      "BrandId": "string",
      "AllowTransactionWithContactlessCard": true,
      "IsFinancialProduct": true,
      "AllowOfflineAuthorizationForEMVCard": true,
      "AllowReprintReceipt": true,
      "AllowPrintReceipt": true,
      "AllowOfflineAuthorizationForContactlessCard": true,
      "AllowCancel": true,
      "AllowUndo": true,
      "AllowCaptureOfFirstInstallmentValue": true,
      "AllowCaptureOfDownpaymentValue": true,
      "AllowGuaranteeHandling": true,
      "AllowPostdatingTheFirstInstallmentForSaleAndCDCQuery": true,
      "AllowPostdating": true,
      "AllowCDCSale": true,
      "AllowFinancingByStore": true,
      "AllowFinancingByCreditCardCompany": true,
      "MaximumNumberOfInstallmentsWhenFinancingByCreditCardCompany": 0,
      "MaximumNumberOfInstallmentsWhenFinancingByStore": 0,
      "MaximumNumberOfinstallmentsForSaleAndCDCQuery": 0,
      "MinimumNumberOfInstallmentsWhenFinancingByStore": 0,
      "PostdatedSaleGuaranteeType": "DoNotAllowSalesWithGuarantee",
      "PostdatedDayCountLimit": 0,
      "FirstInstallmentDayCountLimit": 0
    }
  ],
  "Emv": [
    {
      "Aid": "string",
      "TagsFirst": "string",
      "TagsSecond": "string",
      "IdxRecord": 0,
      "Type": 0,
      "RCodeFirst": "string",
      "RCodeSecond": "string",
      "InvalidateFunctionIfCardIsOnBlacklist": true,
      "RequireBINToBeInCardRangeTable": true,
      "StoreTransactionsRejectedByTerminalAppAndSendToHost": true,
      "AllowOnlineAuthorizationTransactionRequest": true,
      "AllowExtendedCardHolderName": true,
      "NatEmvConctactRiskFloorLimit": 0,
      "NatEmvConctactRiskMinValue": 0,
      "NatEmvConctactRiskMinPercent": 0,
      "NatEmvConctactRiskMaxPercent": 0,
      "IntEmvConctactRiskFloorLimit": 0,
      "IntEmvConctactRiskMinValue": 0,
      "IntEmvConctactRiskMinPercent": 0,
      "IntEmvConctactRiskMaxPercent": 0,
      "ProductIds": [
        0
      ]
    }
  ],
  "Parameters": [
    {
      "Currency": "string",
      "AllowFallbackWhenChipReadingFails": true,
      "AllowChargingMoedeiroFromCash": true,
      "AllowPurchaseWithCompreESaque": true,
      "AllowOfflineFunctionExceptForEMVCard": true,
      "AllowTypingCardNumber": true,
      "MaskCardNumberUsingLast4Digits": true,
      "MaskCardNumberUsingFirst6AndLast4Digits": true,
      "AllowPrintCardHolderBalance": true,
      "AllowDisplayCardHolderBalance": true,
      "AllowPrintingPartialCardNumberInReceipt": true,
      "RestrictSaleWithDuplicateValueWhenPostdated": true,
      "RestrictSaleWithDuplicateValue": true,
      "RequiresPassword": true,
      "InterpretsLastDigitOfSecurityCode": true,
      "RequiresPasswordExceptForEMVCard": true,
      "EnableAdditionalSecurityCodeOptions_Unreadable_NoCode": true,
      "RequiresSecurityCodeWhenMagneticTrackIsRead": true,
      "RequiresSecurityCodeWhenCardNumberIsTyped": true,
      "RequiresTypingLast4Digits": true,
      "CapturesServiceFee": true,
      "AllowCancellationWithValueGreaterThanTheValueOfTheSale": true,
      "CaptureBoardingFee": true
    }
  ],
  "Issuers": [
    {
      "IssuerId": 0,
      "IssuerName": "string",
      "AllowFallbackWhenChipReadingFails": true,
      "AllowChargingMoedeiroFromCash": true,
      "AllowPurchaseWithCompreESaque": true,
      "AllowOfflineFunctionExceptForEMVCard": true,
      "AllowTypingCardNumber": true,
      "MaskCardNumberUsingLast4Digits": true,
      "MaskCardNumberUsingFirst6AndLast4Digits": true,
      "AllowPrintCardHolderBalance": true,
      "AllowDisplayCardHolderBalance": true,
      "Option03BiAllowPrintingPartialCardNumberInReceipt07": true,
      "RestrictSaleWithDuplicateValueWhenPostdated": true,
      "RestrictSaleWithDuplicateValue": true,
      "RequiresPassword": true,
      "InterpretsLastDigitOfSecurityCode": true,
      "RequiresPasswordExceptForEMVCard": true,
      "EnableAdditionalSecurityCodeOptions_Unreadable_NoCode": true,
      "RequiresSecurityCodeWhenMagneticTrackIsRead": true,
      "RequiresSecurityCodeWhenCardNumberIsTyped": true,
      "RequiresTypingLast4Digits": true,
      "AllowCaptureOfFirstInstallmentValue": true,
      "AllowCaptureOfDownpaymentValue": true,
      "AllowGuaranteeHandling": true,
      "AllowPostdatingTheFirstInstallmentForSaleAndCDCQuery": true,
      "AllowPostdating": true,
      "AllowCDCSale": true,
      "AllowFinancingByStore": true,
      "AllowFinancingByCreditCardCompany": true,
      "RequiresChipReader": true,
      "RequiresPinpad": true,
      "LimitDayforReversal": 0,
      "LimitValueforReversal": "string",
      "LimitPercentforReversal": 0,
      "IssuerNameForDisplay": "string",
      "IssuerNameForPrint": "string"
    }
  ],
  "AidParameters": "string",
  "PublicKeys": "string",
  "InitializationVersion": 1558708320029
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`MerchantId`|String|---|---|Identificador da loja|
|`TerminalId`|String|---|---|Identificador do terminal|
|`Acquirer.EnableContaclessCardReader`|Booleano|---|---|Habilita Leitora Cartão Sem Contato|
|`Acquirer.LockAppFunctionsExceptInitialization`|Booleano|---|---|Bloquear as funções do aplicativo, com exceção da Inicialização|
|`Acquirer.HasChipReader`|Booleano|---|---|Indica que tem leitora de Chip-Card|
|`Acquirer.HasMagneticTrackReader`|Booleano|---|---|Indica que tem leitor da trilha magnética|
|`Acquirer.HasKeyboard`|Booleano|---|---|Indica que tem teclado para digitação|
|`Merchant.MerchantId`|String|---|---|Código do Lojista na PayStore, definido no momento da criação do lojista.|
|`Merchant.NetworkName`|String|---|---|Nome da rede da sub-adquirente cadastrado pelo Gestor da PayStore.|
|`Merchant.MerchantName`|String|---|---|Nome fantasia do lojista, definido no momento da criação do mesmo no portal da PayStore.|
|`Merchant.MerchantAddress`|String|---|---|Endereço do lojista obtido a partir da digitação do CEP momento da criação do mesmo no portal da PayStore.|
|`Merchant.NationalId`|String|---|---|CPF ou CNPJ, definido no momento da criação do Lojista no portal da PayStore.|
|`Bins.InitialBin`|String|---|---|Início do range de BIN’s.|
|`Bins.FinalBin`|String|---|---|Final do range de BIN’s.|
|`Bins.ProductId`|Integer int32|---|---|Chave estrangeira de “PRODUCT TABLE”.|
|`Bins.Type`|Integer int32|---|---|Admite os seguintes valores <br><br>0 - ESPECÍFICO <br><br>1 – GENERICO.|
|`Bins.AllowFallbackWhenChipReadingFails`|Booleano|---|---|Permite fallback se houver erro na leitura do Chip.|
|`Bins.AllowChargingMoedeiroFromCash`|Booleano|---|---|Permite carga de moedeiro a partir de dinheiro em espécie.|
|`Bins.AllowPurchaseWithCompreESaque`|Booleano|---|---|Permite venda com Compre & Saque.|
|`Bins.AllowOfflineFunctionExceptForEMVCard`|Booleano|---|---|Permite função offline, exceto cartão EMV.|
|`Bins.AllowTypingCardNumber`|Booleano|---|---|Permite digitação do número do cartão.|
|`Bins.MaskCardNumberUsingLast4Digits`|Booleano|---|---|Imprimir apenas os 4 últimos dígitos do cartão.|
|`Bins.MaskCardNumberUsingFirst6AndLast4Digits`|Booleano|---|---|Imprimir os 6 primeiros e os 4 últimos dígitos do cartão.|
|`Bins.AllowPrintCardHolderBalance`|Booleano|---|---|Permite imprimir o saldo do portador.|
|`Bins.AllowDisplayCardHolderBalance`|Booleano|---|---|Permite exibir no display o saldo do portador.|
|`Bins.AllowPrintingPartialCardNumberInReceipt`|Booleano|---|---|Permite impressão parcial do número do cartão no comprovante das transações.|
|`Bins.RestrictSaleWithDuplicateValueWhenPostdated`|Booleano|---|---|Impede venda com valor duplicado para prédatamento.|
|`Bins.RestrictSaleWithDuplicateValueWhenPostdated`|Booleano|---|---|Impede venda com valor duplicado.|
|`Bins.RequiresPassword`|Booleano|---|---|Solicita senha.|
|`Bins.InterpretsLastDigitOfSecurityCode`|Booleano|---|---|Interpreta último dígito do Código de Serviço.|
|`Bins.RequiresPasswordExceptForEMVCard`|Booleano|---|---|Solicita senha, exceto cartão EMV.|
|`Bins.EnableAdditionalSecurityCodeOptions_Unreadable_NoCode`|Booleano|---|---|Habilita opções “Ilegível” e “Não Possui” para Código de Segurança.|
|`Bins.RequiresSecurityCodeWhenMagneticTrackIsRead`|Booleano|---|---|Solicita Código de Segurança na leitura de trilha.|
|`Bins.RequiresSecurityCodeWhenCardNumberIsTyped`|Booleano|---|---|Solicita Código de Segurança para cartão digitado.|
|`Bins.RequiresTypingLast4Digits`|Booleano|---|---|Solicita digitação dos últimos 4 dígitos.|
|`Bins.AllowCaptureOfFirstInstallmentValue`|Booleano|---|---|Permite captura do valor da primeira parcela.|
|`Bins.AllowCaptureOfDownpaymentValue`|Booleano|---|---|Permite captura do valor de entrada.|
|`Bins.AllowGuaranteeHandling`|Booleano|---|---|Permite tratamento de garantia.|
|`Bins.AllowPostdatingTheFirstInstallmentForSaleAndCDCQuery`|Booleano|---|---|Permite pré-datar a primeira parcela para venda e consulta CDC.|
|`Bins.AllowPostdating`|Booleano|---|---|Permite pré-datamento.|
|`Bins.AllowCDCSale`|Booleano|---|---|Permite venda CDC.|
|`Bins.AllowFinancingByStore`|Booleano|---|---|Permite financiamento pela Loja.|
|`Bins.AllowFinancingByCreditCardCompany`|Booleano|---|---|Permite financiamento pela Administradora.|
|`Bins.ValidateCardTrack1`|Booleano|---|---|Verifica Trilha 1 do cartão.|
|`Bins.DoNotValidateCardModule10`|Booleano|---|---|Não validar o Módulo 10 do cartão.|
|`Bins.CheckExpiryDateWhenCardNumberIsTyped`|Booleano|---|---|Verifica data de validade do cartão digitado.|
|`Bins.CheckExpiryDateWhenMagneticTrackIsRead`|Booleano|---|---|Verifica data de validade da trilha.|
|`Bins.IssuerId`|Integer int32|---|---|Chave estrangeira de “ISSUER TABLE”.|
|`Products.ProductId`|Integer int32|---|---|Identificador do produto.|
|`Products.ProductName`|String|---|---|Nome do produto.|
|`Products.ProductType`|Integer int32|---|---|Admite os seguintes valores <br><br>0 - CREDITO <br><br>1 – DEBITO|
|`Products.BrandId`|String|---|---|Identificador da bandeira do cartão.|
|`Products.AllowTransactionWithContactlessCard`|Booleano|---|---|Permite Transação com Cartão Sem Contato.|
|`Products.IsFinancialProduct`|Booleano|---|---|Produto Financeiro.|
|`Products.AllowOfflineAuthorizationForEMVCard`|Booleano|---|---|Permite Autorização Offline EMV.|
|`Products.AllowReprintReceipt`|Booleano|---|---|Permite Reimpressão do comprovante.|
|`Products.AllowPrintReceipt`|Booleano|---|---|Permite Impressão do comprovante.|
|`Products.AllowOfflineAuthorizationForContactlessCard`|Booleano|---|---|Permite Autorização Offline para Cartão Sem Contato.|
|`Products.AllowCancel`|Booleano|---|---|Permite Cancelamento.|
|`Products.AllowUndo`|Booleano|---|---|Permite Desfazimento.|
|`Products.AllowCaptureOfFirstInstallmentValue`|Booleano|---|---|Permite captura do valor da primeira parcela.|
|`Products.AllowCaptureOfDownpaymentValue`|Booleano|---|---|Permite captura do valor de entrada.|
|`Products.AllowGuaranteeHandling`|Booleano|---|---|Permite tratamento de garantia.|
|`Products.AllowPostdatingTheFirstInstallmentForSaleAndCDCQuery`|Booleano|---|---|Permite pré-datar a primeira parcela para venda e consulta CDC.|
|`Products.AllowPostdating`|Booleano|---|---|Permite pré-datamento.|
|`Products.AllowCDCSale`|Booleano|---|---|Permite venda CDC.|
|`Products.AllowFinancingByStore`|Booleano|---|---|Permite financiamento pela Loja.|
|`Products.AllowFinancingByCreditCardCompany`|Booleano|---|---|Permite financiamento pela Administradora.|
|`Products.MaximumNumberOfInstallmentsWhenFinancingByCreditCardCompany`|Integer int32|---|---|Número máximo de parcelas para financiamento ADM.|
|`Products.MaximumNumberOfInstallmentsWhenFinancingByStore`|Integer int32|---|---|Número máximo de parcelas para financiamento Loja.|
|`Products.MaximumNumberOfinstallmentsForSaleAndCDCQuery`|Integer int32|---|---|Número máximo de parcelas para venda e consulta CDC.|
|`Products.MinimumNumberOfInstallmentsWhenFinancingByStore`|Integer int64|---|---|Valor mínimo de parcelas para financiamento Loja.|
|`Products.PostdatedSaleGuaranteeType`|String|---|---|Tipo de Garantia para o Pré-datado. <br><br>Admite os seguintes valores <br><br>00 – Não permite tratamento de Garantia (Venda Garantida); <br><br>05 – Permite transações Pré-datadas Garantidas; <br><br>07 – Permite transações Pré-datadas Garantidas e Sem Garantia.|
|`Products.PostdatedDayCountLimit`|Integer int32|---|---|Limite máximo em dias para pré-datar a partir da data atual. <br><br>00 – Não aceita. <br><br>XX - Pré-datado.|
|`Products.FirstInstallmentDayCountLimit`|Integer int32|---|---|Limite da data de primeira parcela. <br><br>00 – Não aceita. <br><br>XX - Limite de dias.|
|`Emv.Aid`|String|---|---|Identificador da aplicação EMV.|
|`Emv.TagsFirst`|String|---|---|Conjunto de tags obrigatórias enviadas o 1º Generate AC.|
|`Emv.TagsSecond`|String|---|---|Conjunto de tags obrigatórias enviadas o 2º Generate AC.|
|`Emv.IdxRecord`|Integer int32|---|---|---|
|`Emv.Type`|Integer int32|---|---|Admite os seguintes valores <br><br>0 - CREDITO <br><br>1 – DEBITO|
|`Emv.RCodeFirst`|String|---|---|---|
|`Emv.RCodeSecond`|String|---|---|---|
|`Emv.InvalidateFunctionIfCardIsOnBlacklist`|Booleano|---|---|Invalida a função se o cartão consta na Lista Negra.|
|`Emv.RequireBINToBeInCardRangeTable`|Booleano|---|---|Obriga que o BIN esteja na tabela de Range de Cartões (tipo 2B).|
|`Emv.StoreTransactionsRejectedByTerminalAppAndSendToHost`|Booleano|---|---|Armazena e envia para o Host as transações rejeitadas pelo aplicativo do terminal.|
|`Emv.AllowOnlineAuthorizationTransactionRequest`|Booleano|---|---|---|
|`Emv.AllowExtendedCardHolderName`|Booleano|---|---|---|
|`Emv.NatEmvConctactRiskFloorLimit`|Integer int32|---|---|Valor máximo de verificação para autorização offline das transações. As transações realizadas com a leitura do Chip EMV e com valor acima do “Floor limit”, deverão ser autorizadas no modo online.|
|`Emv.NatEmvConctactRiskMinValue`|Integer int32|---|---|Valor mínimo para o cálculo de seleção aleatória para autorização offline. Conforme processo definido na Especificação EMV.|
|`Emv.NatEmvConctactRiskMinPercent`|Integer int32|---|---|Porcentagem mínima para seleção aleatória. Utilizar apenas o conteúdo do último byte.|
|`Emv.NatEmvConctactRiskMaxPercent`|Integer int32|---|---|Porcentagem máxima para seleção aleatória. Utilizar apenas o conteúdo do último byte.|
|`Emv.IntEmvConctactRiskFloorLimit`|Integer int32|---|---|Valor máximo de verificação para autorização offline das transações. As transações realizadas com a leitura do Chip EMV e com valor acima do “Floor limit”, deverão ser autorizadas no modo online.|
|`Emv.IntEmvConctactRiskMinValue`|Integer int32|---|---|Valor mínimo para o cálculo de seleção aleatória para autorização offline. Conforme processo definido na Especificação EMV.|
|`Emv.IntEmvConctactRiskMinPercent`|Integer int32|---|---|Porcentagem mínima para seleção aleatória. Utilizar apenas o conteúdo do último byte.|
|`Emv.IntEmvConctactRiskMaxPercent`|Integer int32|---|---|Porcentagem máxima para seleção aleatória. Utilizar apenas o conteúdo do último byte.|
|`Emv.ProductIds`|Array of integers int32|---|---|Produtos habilitados para esta aplicação EMV.|
|`Parameters.Currency`|String|---|---|---|
|`Parameters.AllowFallbackWhenChipReadingFails`|Booleano|---|---|Permite fallback se houver erro na leitura do Chip.|
|`Parameters.AllowChargingMoedeiroFromCash`|Booleano|---|---|Permite carga de moedeiro a partir de dinheiro em espécie.|
|`Parameters.AllowPurchaseWithCompreESaque`|Booleano|---|---|Permite venda com Compre & Saque.|
|`Parameters.AllowOfflineFunctionExceptForEMVCard`|Booleano|---|---|Permite função offline, exceto cartão EMV.|
|`Parameters.AllowTypingCardNumber`|Booleano|---|---|Permite entrada manual do número do cartão.|
|`Parameters.MaskCardNumberUsingLast4Digits`|Booleano|---|---|Imprimir apenas os 4 últimos dígitos do cartão.|
|`Parameters.MaskCardNumberUsingFirst6AndLast4Digits`|Booleano|---|---|Imprimir os 6 primeiros e os 4 últimos dígitos do cartão.|
|`Parameters.AllowPrintCardHolderBalance`|Booleano|---|---|Permite imprimir o saldo do portador.|
|`Parameters.AllowDisplayCardHolderBalance`|Booleano|---|---|Permite exibir no display o saldo do portador.|
|`Parameters.AllowPrintingPartialCardNumberInReceipt`|Booleano|---|---|Permite impressão parcial do número do cartão no comprovante das transações.|
|`Parameters.RestrictSaleWithDuplicateValueWhenPostdated`|Booleano|---|---|Impede venda com valor duplicado para pré-datamento.|
|`Parameters.RestrictSaleWithDuplicateValue`|Booleano|---|---|Impede venda com valor duplicado.|
|`Parameters.RequiresPassword`|Booleano|---|---|Solicita senha.|
|`Parameters.InterpretsLastDigitOfSecurityCode`|Booleano|---|---|Interpreta último dígito do Código de Serviço.|
|`Parameters.RequiresPasswordExceptForEMVCard`|Booleano|---|---|Solicita senha, exceto cartão EMV.|
|`Parameters.EnableAdditionalSecurityCodeOptions_Unreadable_NoCode`|Booleano|---|---|Habilita opções “Ilegível” e “Não Possui” para Código de Segurança.|
|`Parameters.RequiresSecurityCodeWhenMagneticTrackIsRead`|Booleano|---|---|Solicita Código de Segurança na leitura de trilha.|
|`Parameters.RequiresSecurityCodeWhenCardNumberIsTyped`|Booleano|---|---|Solicita Código de Segurança para cartão digitado.|
|`Parameters.RequiresTypingLast4Digits`|Booleano|---|---|Solicita digitação dos últimos 4 dígitos.|
|`Parameters.CapturesServiceFee`|Booleano|---|---|Captura Taxa de Serviço.|
|`Parameters.AllowCancellationWithValueGreaterThanTheValueOfTheSale`|Booleano|---|---|Permite valor do Cancelamento maior que o valor da venda original.|
|`Parameters.CaptureBoardingFee`|Booleano|---|---|Captura Taxa de Embarque.|
|`Issuers.IssuerId`|integer int32|---|---|Identificador do emissor.|
|`Issuers.IssuerName`|String|---|---|Nome do emissor.|
|`Issuers.AllowFallbackWhenChipReadingFails`|Booleano|---|---|Permite fallback se houver erro na leitura do Chip.|
|`Issuers.AllowChargingMoedeiroFromCash`|Booleano|---|---|Permite fallback se houver erro na leitura do Chip.|
|`Issuers.AllowPurchaseWithCompreESaque`|Booleano|---|---|Permite venda com Compre & Saque.|
|`Issuers.AllowOfflineFunctionExceptForEMVCard`|Booleano|---|---|Permite função offline, exceto cartão EMV.|
|`Issuers.AllowTypingCardNumber`|Booleano|---|---|Permite digitação do número do cartão.|
|`Issuers.MaskCardNumberUsingLast4Digits`|Booleano|---|---|Imprimir apenas os 4 últimos dígitos do cartão.|
|`Issuers.MaskCardNumberUsingFirst6AndLast4Digits`|Booleano|---|---|Imprimir os 6 primeiros e os 4 últimos dígitos do cartão.|
|`Issuers.AllowPrintCardHolderBalance`|Booleano|---|---|Permite imprimir o saldo do portador.|
|`Issuers.AllowDisplayCardHolderBalance`|Booleano|---|---|Permite exibir no display o saldo do portador.|
|`Issuers.Option03BiAllowPrintingPartialCardNumberInReceipt07`|Booleano|---|---|Permite impressão parcial do número do cartão no comprovante das transações.|
|`Issuers.RestrictSaleWithDuplicateValueWhenPostdated`|Booleano|---|---|Impede venda com valor duplicado para pré-datamento.|
|`Issuers.RestrictSaleWithDuplicateValue`|Booleano|---|---|Impede venda com valor duplicado.|
|`Issuers.RequiresPassword`|Booleano|---|---|Solicita senha.|
|`Issuers.InterpretsLastDigitOfSecurityCode`|Booleano|---|---|Interpreta último dígito do Código de Serviço.|
|`Issuers.RequiresPasswordExceptForEMVCard`|Booleano|---|---|Solicita senha, exceto cartão EMV.|
|`Issuers.EnableAdditionalSecurityCodeOptions_Unreadable_NoCode`|Booleano|---|---|Habilita opções “Ilegível” e “Não Possui” para Código de Segurança.|
|`Issuers.RequiresSecurityCodeWhenMagneticTrackIsRead`|Booleano|---|---|Solicita Código de Segurança na leitura de trilha.|
|`Issuers.RequiresSecurityCodeWhenCardNumberIsTyped`|Booleano|---|---|Solicita Código de Segurança para cartão digitado.|
|`Issuers.RequiresTypingLast4Digits`|Booleano|---|---|Solicita digitação dos últimos 4 dígitos.|
|`Issuers.AllowCaptureOfFirstInstallmentValue`|Booleano|---|---|Permite captura do valor da primeira parcela.|
|`Issuers.AllowCaptureOfDownpaymentValue`|Booleano|---|---|Permite captura do valor de entrada.|
|`Issuers.AllowGuaranteeHandling`|Booleano|---|---|Permite tratamento de garantia.|
|`Issuers.AllowPostdatingTheFirstInstallmentForSaleAndCDCQuery`|Booleano|---|---|Permite pré-datar a primeira parcela para venda e consulta CDC.|
|`Issuers.AllowPostdating`|Booleano|---|---|Permite pré-datamento.|
|`Issuers.AllowCDCSale`|Booleano|---|---|Permite venda CDC.|
|`Issuers.AllowFinancingByStore`|Booleano|---|---|Permite financiamento pela Loja.|
|`Issuers.AllowFinancingByCreditCardCompany`|Booleano|---|---|Permite financiamento pela Administradora.|
|`Issuers.RequiresChipReader`|Booleano|---|---|Exige a existência de Leitor de Chip.|
|`Issuers.RequiresPinpad`|Booleano|---|---|Exige a existência de PIN-pad.|
|`Issuers.LimitDayforReversal`|integer int32|---|---|Data limite em dias para permitir Cancelamento.|
|`Issuers.LimitValueforReversal`|String|---|---|Valor máximo para Cancelamento.|
|`Issuers.LimitPercentforReversal`|integer int64|---|---|Percentual máximo para Cancelamento.|
|`Issuers.IssuerNameForDisplay`|String|---|---|Nome do Issuer para o Display.|
|`Issuers.IssuerNameForPrint`|String|---|---|Nome do Issuer para Impressão.|
|`AidParameters`|---|---|---|---|
|`PublicKeys`|---|---|---|---|
|`InitializationVersion`|---|---|---|---|

## Baixa parâmetros de inicialização (loja padrão)

Solicita as tabelas e parametros para operação do terminal. Como não foi informado o `SubordinatedMerchantId`, será assumida a loja principal do facilitador, isto é, a loja que tem o ID igual ao `ClientId` usado para a autenticação. Esta loja é criada automaticamente durante o processo de cadastro do facilitador executado pela Cielo.

### Requisição

<aside class="request"><span class="method get">GET</span> <span class="endpoint">/initialization/{TerminalId}</span></aside>

### Resposta

```json
{
  "MerchantId": "string",
  "TerminalId": "string",
  "Acquirer": {
    "EnableContaclessCardReader": true,
    "LockAppFunctionsExceptInitialization": true,
    "HasChipReader": true,
    "HasMagneticTrackReader": true,
    "HasKeyboard": true
  },
  "Merchant": {
    "MerchantId": "string",
    "NetworkName": "string",
    "MerchantName": "string",
    "MerchantAddress": "string",
    "NationalId": "string"
  },
  "Bins": [
    {
      "InitialBin": "string",
      "FinalBin": "string",
      "ProductId": 0,
      "Type": 0,
      "AllowFallbackWhenChipReadingFails": true,
      "AllowChargingMoedeiroFromCash": true,
      "AllowPurchaseWithCompreESaque": true,
      "AllowOfflineFunctionExceptForEMVCard": true,
      "AllowTypingCardNumber": true,
      "MaskCardNumberUsingLast4Digits": true,
      "MaskCardNumberUsingFirst6AndLast4Digits": true,
      "AllowPrintCardHolderBalance": true,
      "AllowDisplayCardHolderBalance": true,
      "AllowPrintingPartialCardNumberInReceipt": true,
      "RestrictSaleWithDuplicateValueWhenPostdated": true,
      "RestrictSaleWithDuplicateValue": true,
      "RequiresPassword": true,
      "InterpretsLastDigitOfSecurityCode": true,
      "RequiresPasswordExceptForEMVCard": true,
      "EnableAdditionalSecurityCodeOptions_Unreadable_NoCode": true,
      "RequiresSecurityCodeWhenMagneticTrackIsRead": true,
      "RequiresSecurityCodeWhenCardNumberIsTyped": true,
      "RequiresTypingLast4Digits": true,
      "AllowCaptureOfFirstInstallmentValue": true,
      "AllowCaptureOfDownpaymentValue": true,
      "AllowGuaranteeHandling": true,
      "AllowPostdatingTheFirstInstallmentForSaleAndCDCQuery": true,
      "AllowPostdating": true,
      "AllowCDCSale": true,
      "AllowFinancingByStore": true,
      "AllowFinancingByCreditCardCompany": true,
      "ValidateCardTrack1": true,
      "DoNotValidateCardModule10": true,
      "CheckExpiryDateWhenCardNumberIsTyped": true,
      "CheckExpiryDateWhenMagneticTrackIsRead": true,
      "IssuerId": 0
    }
  ],
  "Products": [
    {
      "ProductId": 0,
      "ProductName": "string",
      "ProductType": 0,
      "BrandId": "string",
      "AllowTransactionWithContactlessCard": true,
      "IsFinancialProduct": true,
      "AllowOfflineAuthorizationForEMVCard": true,
      "AllowReprintReceipt": true,
      "AllowPrintReceipt": true,
      "AllowOfflineAuthorizationForContactlessCard": true,
      "AllowCancel": true,
      "AllowUndo": true,
      "AllowCaptureOfFirstInstallmentValue": true,
      "AllowCaptureOfDownpaymentValue": true,
      "AllowGuaranteeHandling": true,
      "AllowPostdatingTheFirstInstallmentForSaleAndCDCQuery": true,
      "AllowPostdating": true,
      "AllowCDCSale": true,
      "AllowFinancingByStore": true,
      "AllowFinancingByCreditCardCompany": true,
      "MaximumNumberOfInstallmentsWhenFinancingByCreditCardCompany": 0,
      "MaximumNumberOfInstallmentsWhenFinancingByStore": 0,
      "MaximumNumberOfinstallmentsForSaleAndCDCQuery": 0,
      "MinimumNumberOfInstallmentsWhenFinancingByStore": 0,
      "PostdatedSaleGuaranteeType": "DoNotAllowSalesWithGuarantee",
      "PostdatedDayCountLimit": 0,
      "FirstInstallmentDayCountLimit": 0
    }
  ],
  "Emv": [
    {
      "Aid": "string",
      "TagsFirst": "string",
      "TagsSecond": "string",
      "IdxRecord": 0,
      "Type": 0,
      "RCodeFirst": "string",
      "RCodeSecond": "string",
      "InvalidateFunctionIfCardIsOnBlacklist": true,
      "RequireBINToBeInCardRangeTable": true,
      "StoreTransactionsRejectedByTerminalAppAndSendToHost": true,
      "AllowOnlineAuthorizationTransactionRequest": true,
      "AllowExtendedCardHolderName": true,
      "NatEmvConctactRiskFloorLimit": 0,
      "NatEmvConctactRiskMinValue": 0,
      "NatEmvConctactRiskMinPercent": 0,
      "NatEmvConctactRiskMaxPercent": 0,
      "IntEmvConctactRiskFloorLimit": 0,
      "IntEmvConctactRiskMinValue": 0,
      "IntEmvConctactRiskMinPercent": 0,
      "IntEmvConctactRiskMaxPercent": 0,
      "ProductIds": [
        0
      ]
    }
  ],
  "Parameters": [
    {
      "Currency": "string",
      "AllowFallbackWhenChipReadingFails": true,
      "AllowChargingMoedeiroFromCash": true,
      "AllowPurchaseWithCompreESaque": true,
      "AllowOfflineFunctionExceptForEMVCard": true,
      "AllowTypingCardNumber": true,
      "MaskCardNumberUsingLast4Digits": true,
      "MaskCardNumberUsingFirst6AndLast4Digits": true,
      "AllowPrintCardHolderBalance": true,
      "AllowDisplayCardHolderBalance": true,
      "AllowPrintingPartialCardNumberInReceipt": true,
      "RestrictSaleWithDuplicateValueWhenPostdated": true,
      "RestrictSaleWithDuplicateValue": true,
      "RequiresPassword": true,
      "InterpretsLastDigitOfSecurityCode": true,
      "RequiresPasswordExceptForEMVCard": true,
      "EnableAdditionalSecurityCodeOptions_Unreadable_NoCode": true,
      "RequiresSecurityCodeWhenMagneticTrackIsRead": true,
      "RequiresSecurityCodeWhenCardNumberIsTyped": true,
      "RequiresTypingLast4Digits": true,
      "CapturesServiceFee": true,
      "AllowCancellationWithValueGreaterThanTheValueOfTheSale": true,
      "CaptureBoardingFee": true
    }
  ],
  "Issuers": [
    {
      "IssuerId": 0,
      "IssuerName": "string",
      "AllowFallbackWhenChipReadingFails": true,
      "AllowChargingMoedeiroFromCash": true,
      "AllowPurchaseWithCompreESaque": true,
      "AllowOfflineFunctionExceptForEMVCard": true,
      "AllowTypingCardNumber": true,
      "MaskCardNumberUsingLast4Digits": true,
      "MaskCardNumberUsingFirst6AndLast4Digits": true,
      "AllowPrintCardHolderBalance": true,
      "AllowDisplayCardHolderBalance": true,
      "Option03BiAllowPrintingPartialCardNumberInReceipt07": true,
      "RestrictSaleWithDuplicateValueWhenPostdated": true,
      "RestrictSaleWithDuplicateValue": true,
      "RequiresPassword": true,
      "InterpretsLastDigitOfSecurityCode": true,
      "RequiresPasswordExceptForEMVCard": true,
      "EnableAdditionalSecurityCodeOptions_Unreadable_NoCode": true,
      "RequiresSecurityCodeWhenMagneticTrackIsRead": true,
      "RequiresSecurityCodeWhenCardNumberIsTyped": true,
      "RequiresTypingLast4Digits": true,
      "AllowCaptureOfFirstInstallmentValue": true,
      "AllowCaptureOfDownpaymentValue": true,
      "AllowGuaranteeHandling": true,
      "AllowPostdatingTheFirstInstallmentForSaleAndCDCQuery": true,
      "AllowPostdating": true,
      "AllowCDCSale": true,
      "AllowFinancingByStore": true,
      "AllowFinancingByCreditCardCompany": true,
      "RequiresChipReader": true,
      "RequiresPinpad": true,
      "LimitDayforReversal": 0,
      "LimitValueforReversal": "string",
      "LimitPercentforReversal": 0,
      "IssuerNameForDisplay": "string",
      "IssuerNameForPrint": "string"
    }
  ],
  "AidParameters": "string",
  "PublicKeys": "string",
  "InitializationVersion": 1558708320029
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`MerchantId`|String|---|---|Identificador da loja|
|`TerminalId`|String|---|---|Identificador do terminal|
|`Acquirer.EnableContaclessCardReader`|Booleano|---|---|Habilita Leitora Cartão Sem Contato|
|`Acquirer.LockAppFunctionsExceptInitialization`|Booleano|---|---|Bloquear as funções do aplicativo, com exceção da Inicialização|
|`Acquirer.HasChipReader`|Booleano|---|---|Indica que tem leitora de Chip-Card|
|`Acquirer.HasMagneticTrackReader`|Booleano|---|---|Indica que tem leitor da trilha magnética|
|`Acquirer.HasKeyboard`|Booleano|---|---|Indica que tem teclado para digitação|
|`Merchant.MerchantId`|String|---|---|Código do Lojista na PayStore, definido no momento da criação do lojista.|
|`Merchant.NetworkName`|String|---|---|Nome da rede da sub-adquirente cadastrado pelo Gestor da PayStore.|
|`Merchant.MerchantName`|String|---|---|Nome fantasia do lojista, definido no momento da criação do mesmo no portal da PayStore.|
|`Merchant.MerchantAddress`|String|---|---|Endereço do lojista obtido a partir da digitação do CEP momento da criação do mesmo no portal da PayStore.|
|`Merchant.NationalId`|String|---|---|CPF ou CNPJ, definido no momento da criação do Lojista no portal da PayStore.|
|`Bins.InitialBin`|String|---|---|Início do range de BIN’s.|
|`Bins.FinalBin`|String|---|---|Final do range de BIN’s.|
|`Bins.ProductId`|Integer int32|---|---|Chave estrangeira de “PRODUCT TABLE”.|
|`Bins.Type`|Integer int32|---|---|Admite os seguintes valores <br><br>0 - ESPECÍFICO <br><br>1 – GENERICO.|
|`Bins.AllowFallbackWhenChipReadingFails`|Booleano|---|---|Permite fallback se houver erro na leitura do Chip.|
|`Bins.AllowChargingMoedeiroFromCash`|Booleano|---|---|Permite carga de moedeiro a partir de dinheiro em espécie.|
|`Bins.AllowPurchaseWithCompreESaque`|Booleano|---|---|Permite venda com Compre & Saque.|
|`Bins.AllowOfflineFunctionExceptForEMVCard`|Booleano|---|---|Permite função offline, exceto cartão EMV.|
|`Bins.AllowTypingCardNumber`|Booleano|---|---|Permite digitação do número do cartão.|
|`Bins.MaskCardNumberUsingLast4Digits`|Booleano|---|---|Imprimir apenas os 4 últimos dígitos do cartão.|
|`Bins.MaskCardNumberUsingFirst6AndLast4Digits`|Booleano|---|---|Imprimir os 6 primeiros e os 4 últimos dígitos do cartão.|
|`Bins.AllowPrintCardHolderBalance`|Booleano|---|---|Permite imprimir o saldo do portador.|
|`Bins.AllowDisplayCardHolderBalance`|Booleano|---|---|Permite exibir no display o saldo do portador.|
|`Bins.AllowPrintingPartialCardNumberInReceipt`|Booleano|---|---|Permite impressão parcial do número do cartão no comprovante das transações.|
|`Bins.RestrictSaleWithDuplicateValueWhenPostdated`|Booleano|---|---|Impede venda com valor duplicado para prédatamento.|
|`Bins.RestrictSaleWithDuplicateValueWhenPostdated`|Booleano|---|---|Impede venda com valor duplicado.|
|`Bins.RequiresPassword`|Booleano|---|---|Solicita senha.|
|`Bins.InterpretsLastDigitOfSecurityCode`|Booleano|---|---|Interpreta último dígito do Código de Serviço.|
|`Bins.RequiresPasswordExceptForEMVCard`|Booleano|---|---|Solicita senha, exceto cartão EMV.|
|`Bins.EnableAdditionalSecurityCodeOptions_Unreadable_NoCode`|Booleano|---|---|Habilita opções “Ilegível” e “Não Possui” para Código de Segurança.|
|`Bins.RequiresSecurityCodeWhenMagneticTrackIsRead`|Booleano|---|---|Solicita Código de Segurança na leitura de trilha.|
|`Bins.RequiresSecurityCodeWhenCardNumberIsTyped`|Booleano|---|---|Solicita Código de Segurança para cartão digitado.|
|`Bins.RequiresTypingLast4Digits`|Booleano|---|---|Solicita digitação dos últimos 4 dígitos.|
|`Bins.AllowCaptureOfFirstInstallmentValue`|Booleano|---|---|Permite captura do valor da primeira parcela.|
|`Bins.AllowCaptureOfDownpaymentValue`|Booleano|---|---|Permite captura do valor de entrada.|
|`Bins.AllowGuaranteeHandling`|Booleano|---|---|Permite tratamento de garantia.|
|`Bins.AllowPostdatingTheFirstInstallmentForSaleAndCDCQuery`|Booleano|---|---|Permite pré-datar a primeira parcela para venda e consulta CDC.|
|`Bins.AllowPostdating`|Booleano|---|---|Permite pré-datamento.|
|`Bins.AllowCDCSale`|Booleano|---|---|Permite venda CDC.|
|`Bins.AllowFinancingByStore`|Booleano|---|---|Permite financiamento pela Loja.|
|`Bins.AllowFinancingByCreditCardCompany`|Booleano|---|---|Permite financiamento pela Administradora.|
|`Bins.ValidateCardTrack1`|Booleano|---|---|Verifica Trilha 1 do cartão.|
|`Bins.DoNotValidateCardModule10`|Booleano|---|---|Não validar o Módulo 10 do cartão.|
|`Bins.CheckExpiryDateWhenCardNumberIsTyped`|Booleano|---|---|Verifica data de validade do cartão digitado.|
|`Bins.CheckExpiryDateWhenMagneticTrackIsRead`|Booleano|---|---|Verifica data de validade da trilha.|
|`Bins.IssuerId`|Integer int32|---|---|Chave estrangeira de “ISSUER TABLE”.|
|`Products.ProductId`|Integer int32|---|---|Identificador do produto.|
|`Products.ProductName`|String|---|---|Nome do produto.|
|`Products.ProductType`|Integer int32|---|---|Admite os seguintes valores <br><br>0 - CREDITO <br><br>1 – DEBITO|
|`Products.BrandId`|String|---|---|Identificador da bandeira do cartão.|
|`Products.AllowTransactionWithContactlessCard`|Booleano|---|---|Permite Transação com Cartão Sem Contato.|
|`Products.IsFinancialProduct`|Booleano|---|---|Produto Financeiro.|
|`Products.AllowOfflineAuthorizationForEMVCard`|Booleano|---|---|Permite Autorização Offline EMV.|
|`Products.AllowReprintReceipt`|Booleano|---|---|Permite Reimpressão do comprovante.|
|`Products.AllowPrintReceipt`|Booleano|---|---|Permite Impressão do comprovante.|
|`Products.AllowOfflineAuthorizationForContactlessCard`|Booleano|---|---|Permite Autorização Offline para Cartão Sem Contato.|
|`Products.AllowCancel`|Booleano|---|---|Permite Cancelamento.|
|`Products.AllowUndo`|Booleano|---|---|Permite Desfazimento.|
|`Products.AllowCaptureOfFirstInstallmentValue`|Booleano|---|---|Permite captura do valor da primeira parcela.|
|`Products.AllowCaptureOfDownpaymentValue`|Booleano|---|---|Permite captura do valor de entrada.|
|`Products.AllowGuaranteeHandling`|Booleano|---|---|Permite tratamento de garantia.|
|`Products.AllowPostdatingTheFirstInstallmentForSaleAndCDCQuery`|Booleano|---|---|Permite pré-datar a primeira parcela para venda e consulta CDC.|
|`Products.AllowPostdating`|Booleano|---|---|Permite pré-datamento.|
|`Products.AllowCDCSale`|Booleano|---|---|Permite venda CDC.|
|`Products.AllowFinancingByStore`|Booleano|---|---|Permite financiamento pela Loja.|
|`Products.AllowFinancingByCreditCardCompany`|Booleano|---|---|Permite financiamento pela Administradora.|
|`Products.MaximumNumberOfInstallmentsWhenFinancingByCreditCardCompany`|Integer int32|---|---|Número máximo de parcelas para financiamento ADM.|
|`Products.MaximumNumberOfInstallmentsWhenFinancingByStore`|Integer int32|---|---|Número máximo de parcelas para financiamento Loja.|
|`Products.MaximumNumberOfinstallmentsForSaleAndCDCQuery`|Integer int32|---|---|Número máximo de parcelas para venda e consulta CDC.|
|`Products.MinimumNumberOfInstallmentsWhenFinancingByStore`|Integer int64|---|---|Valor mínimo de parcelas para financiamento Loja.|
|`Products.PostdatedSaleGuaranteeType`|String|---|---|Tipo de Garantia para o Pré-datado. <br><br>Admite os seguintes valores <br><br>00 – Não permite tratamento de Garantia (Venda Garantida); <br><br>05 – Permite transações Pré-datadas Garantidas; <br><br>07 – Permite transações Pré-datadas Garantidas e Sem Garantia.|
|`Products.PostdatedDayCountLimit`|Integer int32|---|---|Limite máximo em dias para pré-datar a partir da data atual. <br><br>00 – Não aceita. <br><br>XX - Pré-datado.|
|`Products.FirstInstallmentDayCountLimit`|Integer int32|---|---|Limite da data de primeira parcela. <br><br>00 – Não aceita. <br><br>XX - Limite de dias.|
|`Emv.Aid`|String|---|---|Identificador da aplicação EMV.|
|`Emv.TagsFirst`|String|---|---|Conjunto de tags obrigatórias enviadas o 1º Generate AC.|
|`Emv.TagsSecond`|String|---|---|Conjunto de tags obrigatórias enviadas o 2º Generate AC.|
|`Emv.IdxRecord`|Integer int32|---|---|---|
|`Emv.Type`|Integer int32|---|---|Admite os seguintes valores <br><br>0 - CREDITO <br><br>1 – DEBITO|
|`Emv.RCodeFirst`|String|---|---|---|
|`Emv.RCodeSecond`|String|---|---|---|
|`Emv.InvalidateFunctionIfCardIsOnBlacklist`|Booleano|---|---|Invalida a função se o cartão consta na Lista Negra.|
|`Emv.RequireBINToBeInCardRangeTable`|Booleano|---|---|Obriga que o BIN esteja na tabela de Range de Cartões (tipo 2B).|
|`Emv.StoreTransactionsRejectedByTerminalAppAndSendToHost`|Booleano|---|---|Armazena e envia para o Host as transações rejeitadas pelo aplicativo do terminal.|
|`Emv.AllowOnlineAuthorizationTransactionRequest`|Booleano|---|---|---|
|`Emv.AllowExtendedCardHolderName`|Booleano|---|---|---|
|`Emv.NatEmvConctactRiskFloorLimit`|Integer int32|---|---|Valor máximo de verificação para autorização offline das transações. As transações realizadas com a leitura do Chip EMV e com valor acima do “Floor limit”, deverão ser autorizadas no modo online.|
|`Emv.NatEmvConctactRiskMinValue`|Integer int32|---|---|Valor mínimo para o cálculo de seleção aleatória para autorização offline. Conforme processo definido na Especificação EMV.|
|`Emv.NatEmvConctactRiskMinPercent`|Integer int32|---|---|Porcentagem mínima para seleção aleatória. Utilizar apenas o conteúdo do último byte.|
|`Emv.NatEmvConctactRiskMaxPercent`|Integer int32|---|---|Porcentagem máxima para seleção aleatória. Utilizar apenas o conteúdo do último byte.|
|`Emv.IntEmvConctactRiskFloorLimit`|Integer int32|---|---|Valor máximo de verificação para autorização offline das transações. As transações realizadas com a leitura do Chip EMV e com valor acima do “Floor limit”, deverão ser autorizadas no modo online.|
|`Emv.IntEmvConctactRiskMinValue`|Integer int32|---|---|Valor mínimo para o cálculo de seleção aleatória para autorização offline. Conforme processo definido na Especificação EMV.|
|`Emv.IntEmvConctactRiskMinPercent`|Integer int32|---|---|Porcentagem mínima para seleção aleatória. Utilizar apenas o conteúdo do último byte.|
|`Emv.IntEmvConctactRiskMaxPercent`|Integer int32|---|---|Porcentagem máxima para seleção aleatória. Utilizar apenas o conteúdo do último byte.|
|`Emv.ProductIds`|Array of integers int32|---|---|Produtos habilitados para esta aplicação EMV.|
|`Parameters.Currency`|String|---|---|---|
|`Parameters.AllowFallbackWhenChipReadingFails`|Booleano|---|---|Permite fallback se houver erro na leitura do Chip.|
|`Parameters.AllowChargingMoedeiroFromCash`|Booleano|---|---|Permite carga de moedeiro a partir de dinheiro em espécie.|
|`Parameters.AllowPurchaseWithCompreESaque`|Booleano|---|---|Permite venda com Compre & Saque.|
|`Parameters.AllowOfflineFunctionExceptForEMVCard`|Booleano|---|---|Permite função offline, exceto cartão EMV.|
|`Parameters.AllowTypingCardNumber`|Booleano|---|---|Permite entrada manual do número do cartão.|
|`Parameters.MaskCardNumberUsingLast4Digits`|Booleano|---|---|Imprimir apenas os 4 últimos dígitos do cartão.|
|`Parameters.MaskCardNumberUsingFirst6AndLast4Digits`|Booleano|---|---|Imprimir os 6 primeiros e os 4 últimos dígitos do cartão.|
|`Parameters.AllowPrintCardHolderBalance`|Booleano|---|---|Permite imprimir o saldo do portador.|
|`Parameters.AllowDisplayCardHolderBalance`|Booleano|---|---|Permite exibir no display o saldo do portador.|
|`Parameters.AllowPrintingPartialCardNumberInReceipt`|Booleano|---|---|Permite impressão parcial do número do cartão no comprovante das transações.|
|`Parameters.RestrictSaleWithDuplicateValueWhenPostdated`|Booleano|---|---|Impede venda com valor duplicado para pré-datamento.|
|`Parameters.RestrictSaleWithDuplicateValue`|Booleano|---|---|Impede venda com valor duplicado.|
|`Parameters.RequiresPassword`|Booleano|---|---|Solicita senha.|
|`Parameters.InterpretsLastDigitOfSecurityCode`|Booleano|---|---|Interpreta último dígito do Código de Serviço.|
|`Parameters.RequiresPasswordExceptForEMVCard`|Booleano|---|---|Solicita senha, exceto cartão EMV.|
|`Parameters.EnableAdditionalSecurityCodeOptions_Unreadable_NoCode`|Booleano|---|---|Habilita opções “Ilegível” e “Não Possui” para Código de Segurança.|
|`Parameters.RequiresSecurityCodeWhenMagneticTrackIsRead`|Booleano|---|---|Solicita Código de Segurança na leitura de trilha.|
|`Parameters.RequiresSecurityCodeWhenCardNumberIsTyped`|Booleano|---|---|Solicita Código de Segurança para cartão digitado.|
|`Parameters.RequiresTypingLast4Digits`|Booleano|---|---|Solicita digitação dos últimos 4 dígitos.|
|`Parameters.CapturesServiceFee`|Booleano|---|---|Captura Taxa de Serviço.|
|`Parameters.AllowCancellationWithValueGreaterThanTheValueOfTheSale`|Booleano|---|---|Permite valor do Cancelamento maior que o valor da venda original.|
|`Parameters.CaptureBoardingFee`|Booleano|---|---|Captura Taxa de Embarque.|
|`Issuers.IssuerId`|integer int32|---|---|Identificador do emissor.|
|`Issuers.IssuerName`|String|---|---|Nome do emissor.|
|`Issuers.AllowFallbackWhenChipReadingFails`|Booleano|---|---|Permite fallback se houver erro na leitura do Chip.|
|`Issuers.AllowChargingMoedeiroFromCash`|Booleano|---|---|Permite fallback se houver erro na leitura do Chip.|
|`Issuers.AllowPurchaseWithCompreESaque`|Booleano|---|---|Permite venda com Compre & Saque.|
|`Issuers.AllowOfflineFunctionExceptForEMVCard`|Booleano|---|---|Permite função offline, exceto cartão EMV.|
|`Issuers.AllowTypingCardNumber`|Booleano|---|---|Permite digitação do número do cartão.|
|`Issuers.MaskCardNumberUsingLast4Digits`|Booleano|---|---|Imprimir apenas os 4 últimos dígitos do cartão.|
|`Issuers.MaskCardNumberUsingFirst6AndLast4Digits`|Booleano|---|---|Imprimir os 6 primeiros e os 4 últimos dígitos do cartão.|
|`Issuers.AllowPrintCardHolderBalance`|Booleano|---|---|Permite imprimir o saldo do portador.|
|`Issuers.AllowDisplayCardHolderBalance`|Booleano|---|---|Permite exibir no display o saldo do portador.|
|`Issuers.Option03BiAllowPrintingPartialCardNumberInReceipt07`|Booleano|---|---|Permite impressão parcial do número do cartão no comprovante das transações.|
|`Issuers.RestrictSaleWithDuplicateValueWhenPostdated`|Booleano|---|---|Impede venda com valor duplicado para pré-datamento.|
|`Issuers.RestrictSaleWithDuplicateValue`|Booleano|---|---|Impede venda com valor duplicado.|
|`Issuers.RequiresPassword`|Booleano|---|---|Solicita senha.|
|`Issuers.InterpretsLastDigitOfSecurityCode`|Booleano|---|---|Interpreta último dígito do Código de Serviço.|
|`Issuers.RequiresPasswordExceptForEMVCard`|Booleano|---|---|Solicita senha, exceto cartão EMV.|
|`Issuers.EnableAdditionalSecurityCodeOptions_Unreadable_NoCode`|Booleano|---|---|Habilita opções “Ilegível” e “Não Possui” para Código de Segurança.|
|`Issuers.RequiresSecurityCodeWhenMagneticTrackIsRead`|Booleano|---|---|Solicita Código de Segurança na leitura de trilha.|
|`Issuers.RequiresSecurityCodeWhenCardNumberIsTyped`|Booleano|---|---|Solicita Código de Segurança para cartão digitado.|
|`Issuers.RequiresTypingLast4Digits`|Booleano|---|---|Solicita digitação dos últimos 4 dígitos.|
|`Issuers.AllowCaptureOfFirstInstallmentValue`|Booleano|---|---|Permite captura do valor da primeira parcela.|
|`Issuers.AllowCaptureOfDownpaymentValue`|Booleano|---|---|Permite captura do valor de entrada.|
|`Issuers.AllowGuaranteeHandling`|Booleano|---|---|Permite tratamento de garantia.|
|`Issuers.AllowPostdatingTheFirstInstallmentForSaleAndCDCQuery`|Booleano|---|---|Permite pré-datar a primeira parcela para venda e consulta CDC.|
|`Issuers.AllowPostdating`|Booleano|---|---|Permite pré-datamento.|
|`Issuers.AllowCDCSale`|Booleano|---|---|Permite venda CDC.|
|`Issuers.AllowFinancingByStore`|Booleano|---|---|Permite financiamento pela Loja.|
|`Issuers.AllowFinancingByCreditCardCompany`|Booleano|---|---|Permite financiamento pela Administradora.|
|`Issuers.RequiresChipReader`|Booleano|---|---|Exige a existência de Leitor de Chip.|
|`Issuers.RequiresPinpad`|Booleano|---|---|Exige a existência de PIN-pad.|
|`Issuers.LimitDayforReversal`|integer int32|---|---|Data limite em dias para permitir Cancelamento.|
|`Issuers.LimitValueforReversal`|String|---|---|Valor máximo para Cancelamento.|
|`Issuers.LimitPercentforReversal`|integer int64|---|---|Percentual máximo para Cancelamento.|
|`Issuers.IssuerNameForDisplay`|String|---|---|Nome do Issuer para o Display.|
|`Issuers.IssuerNameForPrint`|String|---|---|Nome do Issuer para Impressão.|
|`AidParameters`|---|---|---|---|
|`PublicKeys`|---|---|---|---|
|`InitializationVersion`|---|---|---|---|

# Cancelamento

| SandBox                                             | Produção                                      |
|:---------------------------------------------------:|:---------------------------------------------:|
| https://apisandbox.cieloecommerce.cielo.com.br      | https://api.cieloecommerce.cielo.com.br/      |

**Simular respostas:**

Para simular alguma resposta especifica utilize o campo Amount, onde de acordo com o valor dos centavos informado nesse campo é possivel receber uma resposta conforme descrito na tabela abaixo:

|Amount (valor dos centados)|Retorno simulado do Cancelamento|Exemplo de valor simulado|
|40|Aprovado|5040 = R$50,40|
|41|Negado|20041 = R$200,41|
|42|Timeout|3542 = R$35,42|
|49|Erro|1049 = R$10,49|

## Cancelamento de Pagamento

### Cartao Digitado

#### Requisição

<aside class="request"><span class="method post">POST</span> <span class="endpoint">/1/physicalSales/{PaymentId}/voids/</span></aside>

**Path Parameters:**

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`PaymentId`|String uuid|---|Sim|Código do Pagamento|

```json
{
  "MerchantVoidId": 2019042204,
  "MerchantVoidDate": "2019-04-15T12:00:00Z",
  "Card": {
    "InputMode": "Typed",
    "CardNumber": 1234567812345678
  }
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`MerchantVoidId`|String|---|Sim|Número do documento gerado automáticamente pelo terminal e incrementado de 1 acada transação realizada no terminal|
|`MerchantVoidDate`|String|---|Sim|Data do cancelamento.|
|`Card.InputMode`|---|---|---|Enum: "Typed", "MagStripe", "Emv", "ContactlessMagStripe", "ContactlessEmv"|
|`Card.CardNumber`|String|---|---|Número do cartão <br><br>Requerido quando a transação for digitada.|
|`Card.ExpirationDate`|String|---|Sim|Data de expiração do cartão.|

#### Resposta

```json
{
  "VoidId": "f15889ea-5719-4e1a-a2da-f4e50d5bd702",
  "InitializationVersion": 1558708320029,
  "PrintMessage": [
    {
      "Position": "Top",
      "Message": "Transação autorizada"
    },
    {
      "Position": "Middle",
      "Message": "Informação adicional"
    },
    {
      "Position": "Bottom",
      "Message": "Obrigado e volte sempre!"
    }
  ],
  "Receipt": {
    "MerchantName": "Estabelecimento",
    "MerchantAddress": "Rua Sem Saida, 0",
    "MerchantCity": "Cidade",
    "MerchantState": "WA",
    "MerchantCode": 549798965249,
    "Terminal": 12345678,
    "Nsu": 123456,
    "Date": "01/01/20",
    "Hour": 720,
    "IssuerName": "VISA  PROD-I",
    "CardNumber": 1111222233334444,
    "TransactionType": "CANCELAMENTO DE TRANSACAO",
    "AuthorizationCode": 123456,
    "TransactionMode": "ONL",
    "InputMethod": "C",
    "CancelValue": "3,00",
    "OriginalTransactonData": "DADOS DO PAGAMENTO ORIGNAL",
    "OriginalTransactonType": "VENDA A CREDITO",
    "OriginalTransactonNsu": 5349,
    "OriginalTransactonAuthCode": 543210,
    "OriginalTransactionDate": "01/01/20",
    "OriginalTransactionHour": 720,
    "OrignalTransactionValue": "3,00",
    "OrignalTransactionCardHolder": "NOME NOME NOME NOME NOME N",
    "OriginalTransactionMode": "ONL",
    "OriginalInputMethod": "C"
  },  
  "Status": 10,
  "ReturnCode": 0,
  "ReturnMessage": "Success",
  "Links": [
    {
      "Method": "GET",
      "Rel": "self",
      "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702"
    },
    {
      "Method": "POST",
      "Rel": "void",
      "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702/voids"
    },
    {
      "Method": "DELETE",
      "Rel": "reverse",
      "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702/voids/e5c889ea-5719-4e1a-a2da-f4f50d5bd7ca"
    },
    {
      "Method": "PUT",
      "Rel": "confirm",
      "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702/voids/e5c889ea-5719-4e1a-a2da-f4f50d5bd7ca/confirmation"
    }
  ]
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`VoidId`|String - uuid|---|---|Identificador do cancelamento|
|`InitializationVersion`|Integer int16|---|---|Número de versão dos parametros baixados na inicialização do equipamento.|
|`PrintMessage.Position`|String|---|---|Default: "Top"<br><br>Enum: "Top", "Middle", "Bottom"<br><br>Posição da mensagem no comprovante:<br><br>Top - Início do comprovante, antes do código do estabelecimento<br><br>Middle - Meio do comprovante, após a descrição dos valores<br><br>Bottom - Final do comprovante|
|`PrintMessage.Message`|String|---|---|Indica a mensagem que deve ser impressa no comprovante de acordo com a posição indicada no campo Position|
|`Status`|Integerint16|---|---|Status da transação.<br><br>0 = Não Finalizado<br><br>1 = Autorizado<br><br>2 = Pago<br><br>3 = Negado<br><br>10 = Cancelado<br><br>13 = Abortado|
|`CancellationStatus`|Integer int16|---|---|Status do cancelamento.<br><br>0 = Não Finalizado<br><br>1 = Autorizado<br><br>2 = Negado<br><br>3 = Confirmado<br><br>4 = Desfeito|
|`ReasonCode`|Integer int16|---|---|Código de referência para análises.|
|`ReasonMessage`|String|---|---|Mensagem explicativa para análise.|
|`ReturnCode`|String|---|---|Código de erro/resposta da transação da Adquirência.|
|`ReturnMessage`|String|---|---|Mensagem de erro/resposta da transação da Adquirência.|
|`Receipt.MerchantName`|---|---|---|---|
|`Receipt.MerchantAddress`|---|---|---|---|
|`Receipt.MerchantCity`|---|---|---|---|
|`Receipt.MerchantState`|---|---|---|---|
|`Receipt.MerchantCode`|---|---|---|---|
|`Receipt.Terminal`|---|---|---|---|
|`Receipt.Nsu`|---|---|---|---|
|`Receipt.Date`|---|---|---|---|
|`Receipt.Hour`|---|---|---|---|
|`Receipt.IssuerName`|---|---|---|---|
|`Receipt.CardNumber`|---|---|---|---|
|`Receipt.TransactionType`|---|---|---|---|
|`Receipt.AuthorizationCode`|---|---|---|---|
|`Receipt.TransactionMode`|---|---|---|---|
|`Receipt.InputMethod`|---|---|---|---|
|`Receipt.CancelValue`|---|---|---|---|
|`Receipt.OriginalTransactonData`|---|---|---|---|
|`Receipt.OriginalTransactonType`|---|---|---|---|
|`Receipt.OriginalTransactonNsu`|---|---|---|---|
|`Receipt.OriginalTransactonAuthCode`|---|---|---|---|
|`Receipt.OriginalTransactionDate`|---|---|---|---|
|`Receipt.OriginalTransactionHour`|---|---|---|---|
|`Receipt.OrignalTransactionValue`|---|---|---|---|
|`Receipt.OrignalTransactionCardHolder`|---|---|---|---|
|`Receipt.OriginalTransactionMode`|---|---|---|---|
|`Receipt.OriginalInputMethod`|---|---|---|---|
|`Links.Method`|String|---|---|Enum: "POST", "GET", "PUT".<br><br>Método HTTP a ser utilizado na operação.|
|`Links.Rel`|String|---|---|Enum: "self", "cancel", "confirm".<br><br>Referência da operação.|
|`Links.Href`|String|---|---|Endereço de URL de chamada da API|

### Cartão digitado com cartão criptografado

#### Requisição

<aside class="request"><span class="method post">POST</span> <span class="endpoint">/1/physicalSales/{PaymentId}/voids/</span></aside>

**Path Parameters:**

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`PaymentId`|String uuid|---|Sim|Código do Pagamento|

```json
{
  "MerchantVoidId": 2019042204,
  "MerchantVoidDate": "2019-04-15T12:00:00Z",
  "Card": {
    "InputMode": "Typed",
    "CardNumber": 1234567812345678,
    "ExpirationDate": "12/2020"
  }
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`MerchantVoidId`|String|---|Sim|Número do documento gerado automáticamente pelo terminal e incrementado de 1 acada transação realizada no terminal|
|`MerchantVoidDate`|String|---|Sim|Data do cancelamento.|
|`Card.InputMode`|String|---|Sim|Enum: "Typed", "MagStripe", "Emv", "ContactlessMagStripe", "ContactlessEmv"|
|`Card.CardNumber`|String|---|Sim|Número do cartão<br><br>Requerido quando a transação for digitada.|
|`Card.EncryptedCardData.EncryptionType`|String|---|Sim|Tipo de encriptação utilizada<br><br>Enum:<br><br>"DukptDes" = 1,<br><br>"MasterKey" = 2,<br><br>"Dukpt3Des" = 3|
|`Card.EncryptedCardData.CardNumberKSN`|String|---|---|Identificador KSN da criptografia do número do cartão
|`Card.EncryptedCardData.IsDataInTLVFormat`|Bool|---|Não|Identifica se os dados criptografados estão no formato TLV (tag / length / value).|
|`Card.EncryptedCardData.InitializationVector`|String|---|Sim|Vetor de inicialização da encryptação|
|`Card.ExpirationDate|String`|---|Sim|Data de expiração do cartão.|

#### Resposta

```json
{
  "VoidId": "f15889ea-5719-4e1a-a2da-f4e50d5bd702",
  "InitializationVersion": 1558708320029,
  "PrintMessage": [
    {
      "Position": "Top",
      "Message": "Transação autorizada"
    },
    {
      "Position": "Middle",
      "Message": "Informação adicional"
    },
    {
      "Position": "Bottom",
      "Message": "Obrigado e volte sempre!"
    }
  ],
  "Receipt": {
    "MerchantName": "Estabelecimento",
    "MerchantAddress": "Rua Sem Saida, 0",
    "MerchantCity": "Cidade",
    "MerchantState": "WA",
    "MerchantCode": 549798965249,
    "Terminal": 12345678,
    "Nsu": 123456,
    "Date": "01/01/20",
    "Hour": 720,
    "IssuerName": "VISA  PROD-I",
    "CardNumber": 1111222233334444,
    "TransactionType": "CANCELAMENTO DE TRANSACAO",
    "AuthorizationCode": 123456,
    "TransactionMode": "ONL",
    "InputMethod": "C",
    "CancelValue": "3,00",
    "OriginalTransactonData": "DADOS DO PAGAMENTO ORIGNAL",
    "OriginalTransactonType": "VENDA A CREDITO",
    "OriginalTransactonNsu": 5349,
    "OriginalTransactonAuthCode": 543210,
    "OriginalTransactionDate": "01/01/20",
    "OriginalTransactionHour": 720,
    "OrignalTransactionValue": "3,00",
    "OrignalTransactionCardHolder": "NOME NOME NOME NOME NOME N",
    "OriginalTransactionMode": "ONL",
    "OriginalInputMethod": "C"
  },
  "Status": 10,
  "ReturnCode": 0,
  "ReturnMessage": "Success",
  "Links": [
    {
      "Method": "GET",
      "Rel": "self",
      "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702"
    },
    {
      "Method": "POST",
      "Rel": "void",
      "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702/voids"
    },
    {
      "Method": "DELETE",
      "Rel": "reverse",
      "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702/voids/e5c889ea-5719-4e1a-a2da-f4f50d5bd7ca"
    },
    {
      "Method": "PUT",
      "Rel": "confirm",
      "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702/voids/e5c889ea-5719-4e1a-a2da-f4f50d5bd7ca/confirmation"
    }
  ]
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`VoidId`|String - uuid|---|---|Identificador do cancelamento|
|`InitializationVersion`|Integer int16|---|---|Número de versão dos parametros baixados na inicialização do equipamento.|
|`PrintMessage.Position`|String|---|---|Default: "Top"<br><br>Enum: "Top", "Middle", "Bottom"<br><br>Posição da mensagem no comprovante:<br><br>Top - Início do comprovante, antes do código do estabelecimento<br><br>Middle - Meio do comprovante, após a descrição dos valores<br><br>Bottom - Final do comprovante|
|`PrintMessage.Message`|String|---|---|Indica a mensagem que deve ser impressa no comprovante de acordo com a posição indicada no campo Position|
|`Status`|Integerint16|---|---|Status da transação.<br><br>0 = Não Finalizado<br><br>1 = Autorizado<br><br>2 = Pago<br><br>3 = Negado<br><br>10 = Cancelado<br><br>13 = Abortado|
|`CancellationStatus`|Integer int16|---|---|Status do cancelamento.<br><br>0 = Não Finalizado<br><br>1 = Autorizado<br><br>2 = Negado<br><br>3 = Confirmado<br><br>4 = Desfeito|
|`ReasonCode`|Integer int16|---|---|Código de referência para análises.|
|`ReasonMessage`|String|---|---|Mensagem explicativa para análise.|
|`ReturnCode`|String|---|---|Código de erro/resposta da transação da Adquirência.|
|`ReturnMessage`|String|---|---|Mensagem de erro/resposta da transação da Adquirência.|
|`Receipt.MerchantName`|---|---|---|---|
|`Receipt.MerchantAddress`|---|---|---|---|
|`Receipt.MerchantCity`|---|---|---|---|
|`Receipt.MerchantState`|---|---|---|---|
|`Receipt.MerchantCode`|---|---|---|---|
|`Receipt.Terminal`|---|---|---|---|
|`Receipt.Nsu`|---|---|---|---|
|`Receipt.Date`|---|---|---|---|
|`Receipt.Hour`|---|---|---|---|
|`Receipt.IssuerName`|---|---|---|---|
|`Receipt.CardNumber`|---|---|---|---|
|`Receipt.TransactionType`|---|---|---|---|
|`Receipt.AuthorizationCode`|---|---|---|---|
|`Receipt.TransactionMode`|---|---|---|---|
|`Receipt.InputMethod`|---|---|---|---|
|`Receipt.CancelValue`|---|---|---|---|
|`Receipt.OriginalTransactonData`|---|---|---|---|
|`Receipt.OriginalTransactonType`|---|---|---|---|
|`Receipt.OriginalTransactonNsu`|---|---|---|---|
|`Receipt.OriginalTransactonAuthCode`|---|---|---|---|
|`Receipt.OriginalTransactionDate`|---|---|---|---|
|`Receipt.OriginalTransactionHour`|---|---|---|---|
|`Receipt.OrignalTransactionValue`|---|---|---|---|
|`Receipt.OrignalTransactionCardHolder`|---|---|---|---|
|`Receipt.OriginalTransactionMode`|---|---|---|---|
|`Receipt.OriginalInputMethod`|---|---|---|---|
|`Links.Method`|String|---|---|Enum: "POST", "GET", "PUT".<br><br>Método HTTP a ser utilizado na operação.|
|`Links.Rel`|String|---|---|Enum: "self", "cancel", "confirm".<br><br>Referência da operação.|
|`Links.Href`|String|---|---|Endereço de URL de chamada da API|

### Cartão por tarja

#### Requisição

<aside class="request"><span class="method post">POST</span> <span class="endpoint">/1/physicalSales/{PaymentId}/voids/</span></aside>

**Path Parameters:**

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`PaymentId`|String uuid|---|Sim|Código do Pagamento|

```json
{
   "MerchantVoidId": 2019042204,
   "MerchantVoidDate": "2019-04-15T12:00:00Z",
   "Card": {
      "InputMode": "MagStripe",
      "TrackOneData": "A1234567890123456^FULANO OLIVEIRA SA ^12345678901234567890123",
      "TrackTwoData": "0123456789012345=012345678901234"
   }
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`MerchantVoidId`|String|---|Sim|Número do documento gerado automáticamente pelo terminal e incrementado de 1 acada transação realizada no terminal|
|`MerchantVoidDate`|String|---|Sim|Data do cancelamento.|
|`Card.InputMode`|String|---|Sim|Enum: "Typed", "MagStripe", "Emv", "ContactlessMagStripe", "ContactlessEmv"|
|`Card.TrackOneData`|String|---|Sim|Dados da trilha 1<br><br>Dado obtido através do comando PP_GetCard na BC no momento da captura da transação|
|`Card.TrackTwoData`|String|---|Sim|Dados da trilha 2<br><br>Dado obtido através do comando PP_GetCard na BC no momento da captura da transação|
|`Card.AuthenticationMethod`|String|---|Não|Enum: "NoPassword", "OnlineAuthentication", "OfflineAuthentication"<br><br>Método de autenticação<br><br>1 - Sem senha “NoPassword” <br><br>2 - Senha online = “Online Authentication”;<br><br>3 - Senha off-line = “Offline Authentication”.|

#### Resposta

```json
{
  "VoidId": "f15889ea-5719-4e1a-a2da-f4e50d5bd702",
  "InitializationVersion": 1558708320029,
  "PrintMessage": [
    {
      "Position": "Top",
      "Message": "Transação autorizada"
    },
    {
      "Position": "Middle",
      "Message": "Informação adicional"
    },
    {
      "Position": "Bottom",
      "Message": "Obrigado e volte sempre!"
    }
  ],
  "Receipt": {
    "MerchantName": "Estabelecimento",
    "MerchantAddress": "Rua Sem Saida, 0",
    "MerchantCity": "Cidade",
    "MerchantState": "WA",
    "MerchantCode": 549798965249,
    "Terminal": 12345678,
    "Nsu": 123456,
    "Date": "01/01/20",
    "Hour": 720,
    "IssuerName": "VISA  PROD-I",
    "CardNumber": 1111222233334444,
    "TransactionType": "CANCELAMENTO DE TRANSACAO",
    "AuthorizationCode": 123456,
    "TransactionMode": "ONL",
    "InputMethod": "C",
    "CancelValue": "3,00",
    "OriginalTransactonData": "DADOS DO PAGAMENTO ORIGNAL",
    "OriginalTransactonType": "VENDA A CREDITO",
    "OriginalTransactonNsu": 5349,
    "OriginalTransactonAuthCode": 543210,
    "OriginalTransactionDate": "01/01/20",
    "OriginalTransactionHour": 720,
    "OrignalTransactionValue": "3,00",
    "OrignalTransactionCardHolder": "NOME NOME NOME NOME NOME N",
    "OriginalTransactionMode": "ONL",
    "OriginalInputMethod": "C"
  },
  "Status": 10,
  "ReturnCode": 0,
  "ReturnMessage": "Success",
  "Links": [
    {
      "Method": "GET",
      "Rel": "self",
      "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702"
    },
    {
      "Method": "POST",
      "Rel": "void",
      "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702/voids"
    },
    {
      "Method": "DELETE",
      "Rel": "reverse",
      "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702/voids/e5c889ea-5719-4e1a-a2da-f4f50d5bd7ca"
    },
    {
      "Method": "PUT",
      "Rel": "confirm",
      "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702/voids/e5c889ea-5719-4e1a-a2da-f4f50d5bd7ca/confirmation"
    }
  ]
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`VoidId`|String - uuid|---|---|Identificador do cancelamento|
|`InitializationVersion`|Integer int16|---|---|Número de versão dos parametros baixados na inicialização do equipamento.|
|`PrintMessage.Position`|String|---|---|Default: "Top"<br><br>Enum: "Top", "Middle", "Bottom"<br><br>Posição da mensagem no comprovante:<br><br>Top - Início do comprovante, antes do código do estabelecimento<br><br>Middle - Meio do comprovante, após a descrição dos valores<br><br>Bottom - Final do comprovante|
|`PrintMessage.Message`|String|---|---|Indica a mensagem que deve ser impressa no comprovante de acordo com a posição indicada no campo Position|
|`Status`|Integerint16|---|---|Status da transação.<br><br>0 = Não Finalizado<br><br>1 = Autorizado<br><br>2 = Pago<br><br>3 = Negado<br><br>10 = Cancelado<br><br>13 = Abortado|
|`CancellationStatus`|Integer int16|---|---|Status do cancelamento.<br><br>0 = Não Finalizado<br><br>1 = Autorizado<br><br>2 = Negado<br><br>3 = Confirmado<br><br>4 = Desfeito|
|`ReasonCode`|Integer int16|---|---|Código de referência para análises.|
|`ReasonMessage`|String|---|---|Mensagem explicativa para análise.|
|`ReturnCode`|String|---|---|Código de erro/resposta da transação da Adquirência.|
|`ReturnMessage`|String|---|---|Mensagem de erro/resposta da transação da Adquirência.|
|`Receipt.MerchantName`|---|---|---|---|
|`Receipt.MerchantAddress`|---|---|---|---|
|`Receipt.MerchantCity`|---|---|---|---|
|`Receipt.MerchantState`|---|---|---|---|
|`Receipt.MerchantCode`|---|---|---|---|
|`Receipt.Terminal`|---|---|---|---|
|`Receipt.Nsu`|---|---|---|---|
|`Receipt.Date`|---|---|---|---|
|`Receipt.Hour`|---|---|---|---|
|`Receipt.IssuerName`|---|---|---|---|
|`Receipt.CardNumber`|---|---|---|---|
|`Receipt.TransactionType`|---|---|---|---|
|`Receipt.AuthorizationCode`|---|---|---|---|
|`Receipt.TransactionMode`|---|---|---|---|
|`Receipt.InputMethod`|---|---|---|---|
|`Receipt.CancelValue`|---|---|---|---|
|`Receipt.OriginalTransactonData`|---|---|---|---|
|`Receipt.OriginalTransactonType`|---|---|---|---|
|`Receipt.OriginalTransactonNsu`|---|---|---|---|
|`Receipt.OriginalTransactonAuthCode`|---|---|---|---|
|`Receipt.OriginalTransactionDate`|---|---|---|---|
|`Receipt.OriginalTransactionHour`|---|---|---|---|
|`Receipt.OrignalTransactionValue`|---|---|---|---|
|`Receipt.OrignalTransactionCardHolder`|---|---|---|---|
|`Receipt.OriginalTransactionMode`|---|---|---|---|
|`Receipt.OriginalInputMethod`|---|---|---|---|
|`Links.Method`|String|---|---|Enum: "POST", "GET", "PUT".<br><br>Método HTTP a ser utilizado na operação.|
|`Links.Rel`|String|---|---|Enum: "self", "cancel", "confirm".<br><br>Referência da operação.|
|`Links.Href`|String|---|---|Endereço de URL de chamada da API|

### Cartão por tarja com cartão criptografado

#### Requisição

<aside class="request"><span class="method post">POST</span> <span class="endpoint">/1/physicalSales/{PaymentId}/voids/</span></aside>

**Path Parameters:**

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`PaymentId`|String uuid|---|Sim|Código do Pagamento|

```json
{
   "MerchantVoidId": 2019042204,
   "MerchantVoidDate": "2019-04-15T12:00:00Z",
   "Card": {
      "InputMode": "MagStripe",
      "TrackOneData": "A1234567890123456^FULANO OLIVEIRA SA ^12345678901234567890123",
      "TrackTwoData": "0123456789012345=012345678901234",
      "EncryptedCardData": {
         "EncryptionType": "DUKPT3DES",
         "TrackOneDataKSN": "KSNforTrackOneData",
         "TrackTwoDataKSN": "KSNforTrackTwoData"
      }
   }
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`MerchantVoidId`|String|---|Sim|Número do documento gerado automáticamente pelo terminal e incrementado de 1 acada transação realizada no terminal|
|`MerchantVoidDate`|String|---|Sim|Data do cancelamento.|
|`Card.InputMode`|String|---|Sim|Enum: "Typed", "MagStripe", "Emv", "ContactlessMagStripe", "ContactlessEmv"|
|`Card.TrackOneData`|String|---|---|Dados da trilha 1 <br><br>Obtidos através do comando PP_GetCard na BC no momento da captura da transação|
|`Card.TrackTwoData`|String|---|---|Dados da trilha 2 <br><br>Obtidos através do comando PP_GetCard na BC no momento da captura da transação|
|`Card.EncryptedCardData.EncryptionType`|String|---|Sim|Tipo de encriptação utilizada<br><br>Enum:<br><br>"DukptDes" = 1,<br><br>"MasterKey" = 2 <br<br>"Dukpt3Des" = 3,<br><br>"Dukpt3DesCBC" = 4|
|`Card.EncryptedCardData.TrackOneDataKSN`|String|---|---|Identificador KSN da criptografia da trilha 1 do cartão|
|`Card.EncryptedCardData.TrackTwoDataKSN`|String|---|---|Identificador KSN da criptografia da trilha 2 do cartão|
|`Card.EncryptedCardData.IsDataInTLVFormat`|Booleano|---|Não Identifica se os dados criptografados estão no formato TLV (tag / length / value).|
|`Card.EncryptedCardData.InitializationVector`|String|---|Sim|Vetor de inicialização da encryptação|
|`Card.AuthenticationMethod`|String|---|Sim|Enum: "NoPassword", , "OnlineAuthentication", "OfflineAuthentication"<br><br>Método de autenticação<br><br>1 - Sem senha = “NoPassword”;<br><br>2 - Senha online = “Online Authentication”;<br><br>3 - Senha off-line = “Offline Authentication”.|

#### Resposta

```json
{
  "VoidId": "f15889ea-5719-4e1a-a2da-f4e50d5bd702",
  "InitializationVersion": 1558708320029,
  "PrintMessage": [
    {
      "Position": "Top",
      "Message": "Transação autorizada"
    },
    {
      "Position": "Middle",
      "Message": "Informação adicional"
    },
    {
      "Position": "Bottom",
      "Message": "Obrigado e volte sempre!"
    }
  ],
  "Receipt": {
    "MerchantName": "Estabelecimento",
    "MerchantAddress": "Rua Sem Saida, 0",
    "MerchantCity": "Cidade",
    "MerchantState": "WA",
    "MerchantCode": 549798965249,
    "Terminal": 12345678,
    "Nsu": 123456,
    "Date": "01/01/20",
    "Hour": 720,
    "IssuerName": "VISA  PROD-I",
    "CardNumber": 1111222233334444,
    "TransactionType": "CANCELAMENTO DE TRANSACAO",
    "AuthorizationCode": 123456,
    "TransactionMode": "ONL",
    "InputMethod": "C",
    "CancelValue": "3,00",
    "OriginalTransactonData": "DADOS DO PAGAMENTO ORIGNAL",
    "OriginalTransactonType": "VENDA A CREDITO",
    "OriginalTransactonNsu": 5349,
    "OriginalTransactonAuthCode": 543210,
    "OriginalTransactionDate": "01/01/20",
    "OriginalTransactionHour": 720,
    "OrignalTransactionValue": "3,00",
    "OrignalTransactionCardHolder": "NOME NOME NOME NOME NOME N",
    "OriginalTransactionMode": "ONL",
    "OriginalInputMethod": "C"
  },
  "Status": 10,
  "ReturnCode": 0,
  "ReturnMessage": "Success",
  "Links": [
    {
      "Method": "GET",
      "Rel": "self",
      "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702"
    },
    {
      "Method": "POST",
      "Rel": "void",
      "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702/voids"
    },
    {
      "Method": "DELETE",
      "Rel": "reverse",
      "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702/voids/e5c889ea-5719-4e1a-a2da-f4f50d5bd7ca"
    },
    {
      "Method": "PUT",
      "Rel": "confirm",
      "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702/voids/e5c889ea-5719-4e1a-a2da-f4f50d5bd7ca/confirmation"
    }
  ]
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`VoidId`|String - uuid|---|---|Identificador do cancelamento|
|`InitializationVersion`|Integer int16|---|---|Número de versão dos parametros baixados na inicialização do equipamento.|
|`PrintMessage.Position`|String|---|---|Default: "Top"<br><br>Enum: "Top", "Middle", "Bottom"<br><br>Posição da mensagem no comprovante:<br><br>Top - Início do comprovante, antes do código do estabelecimento<br><br>Middle - Meio do comprovante, após a descrição dos valores<br><br>Bottom - Final do comprovante|
|`PrintMessage.Message`|String|---|---|Indica a mensagem que deve ser impressa no comprovante de acordo com a posição indicada no campo Position|
|`Status`|Integerint16|---|---|Status da transação.<br><br>0 = Não Finalizado<br><br>1 = Autorizado<br><br>2 = Pago<br><br>3 = Negado<br><br>10 = Cancelado<br><br>13 = Abortado|
|`CancellationStatus`|Integer int16|---|---|Status do cancelamento.<br><br>0 = Não Finalizado<br><br>1 = Autorizado<br><br>2 = Negado<br><br>3 = Confirmado<br><br>4 = Desfeito|
|`ReasonCode`|Integer int16|---|---|Código de referência para análises.|
|`ReasonMessage`|String|---|---|Mensagem explicativa para análise.|
|`ReturnCode`|String|---|---|Código de erro/resposta da transação da Adquirência.|
|`ReturnMessage`|String|---|---|Mensagem de erro/resposta da transação da Adquirência.|
|`Receipt.MerchantName`|---|---|---|---|
|`Receipt.MerchantAddress`|---|---|---|---|
|`Receipt.MerchantCity`|---|---|---|---|
|`Receipt.MerchantState`|---|---|---|---|
|`Receipt.MerchantCode`|---|---|---|---|
|`Receipt.Terminal`|---|---|---|---|
|`Receipt.Nsu`|---|---|---|---|
|`Receipt.Date`|---|---|---|---|
|`Receipt.Hour`|---|---|---|---|
|`Receipt.IssuerName`|---|---|---|---|
|`Receipt.CardNumber`|---|---|---|---|
|`Receipt.TransactionType`|---|---|---|---|
|`Receipt.AuthorizationCode`|---|---|---|---|
|`Receipt.TransactionMode`|---|---|---|---|
|`Receipt.InputMethod`|---|---|---|---|
|`Receipt.CancelValue`|---|---|---|---|
|`Receipt.OriginalTransactonData`|---|---|---|---|
|`Receipt.OriginalTransactonType`|---|---|---|---|
|`Receipt.OriginalTransactonNsu`|---|---|---|---|
|`Receipt.OriginalTransactonAuthCode`|---|---|---|---|
|`Receipt.OriginalTransactionDate`|---|---|---|---|
|`Receipt.OriginalTransactionHour`|---|---|---|---|
|`Receipt.OrignalTransactionValue`|---|---|---|---|
|`Receipt.OrignalTransactionCardHolder`|---|---|---|---|
|`Receipt.OriginalTransactionMode`|---|---|---|---|
|`Receipt.OriginalInputMethod`|---|---|---|---|
|`Links.Method`|String|---|---|Enum: "POST", "GET", "PUT".<br><br>Método HTTP a ser utilizado na operação.|
|`Links.Rel`|String|---|---|Enum: "self", "cancel", "confirm".<br><br>Referência da operação.|
|`Links.Href`|String|---|---|Endereço de URL de chamada da API|

### Cartão por chip

#### Requisição

<aside class="request"><span class="method post">POST</span> <span class="endpoint">/1/physicalSales/{PaymentId}/voids/</span></aside>

**Path Parameters:**

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`PaymentId`|String uuid|---|Sim|Código do Pagamento|

```json
{
  "MerchantVoidId": 2019042204,
  "MerchantVoidDate": "2019-04-15T12:00:00Z",
  "Card": {
    "InputMode": "MagStripe",
    "EmvData": "112233445566778899011AABBC012D3456789E0123FF45678AB901234C5D112233445566778800",
    "TrackOneData": "A1234567890123456^FULANO OLIVEIRA SA ^12345678901234567890123",
    "TrackTwoData": "0123456789012345=012345678901234"
  }
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`MerchantVoidId`|String|---|Sim|Número do documento gerado automáticamente pelo terminal e incrementado de 1 acada transação realizada no terminal|
|`MerchantVoidDate`|String|---|Sim|Data do cancelamento.|
|`Card.InputMode`|String|---|Sim|Enum: "Typed", "MagStripe", "Emv", "ContactlessMagStripe", "ContactlessEmv"|
|`Card.TrackOneData`|String|---|---|Dados da trilha 1 <br><br>Obtidos através do comando PP_GetCard na BC no momento da captura da transação|
|`Card.TrackTwoData`|String|---|---|Dados da trilha 2 <br><br>Obtidos através do comando PP_GetCard na BC no momento da captura da transação|
|`Card.EmvData`|String|---|---|Dados de cancelamento EMV|
|`Card.AuthenticationMethod`|String|---|Sim|Enum: "NoPassword", , "OnlineAuthentication", "OfflineAuthentication"<br><br>Método de autenticação<br><br>1 - Sem senha = “NoPassword”;<br><br>2 - Senha online = “Online Authentication”;<br><br>3 - Senha off-line = “Offline Authentication”.|
|`PinBlock.EncryptedPinBlock`|String|---|Sim|PINBlock Criptografado<br><br>- Para transações EMV, esse campo é obtido através do retorno da função PP_GoOnChip(), mais especificamente das posições 007 até a posição 022;<br><br>- Para transações digitadas e com tarja magnética, verificar as posições 001 até 016 do retorno da função PP_GetPin().|
|`Card.EncryptedCardData.EncryptionType`|String|---|Sim|Tipo de encriptação utilizada<br><br>Enum:<br><br>"DukptDes" = 1,<br><br>"MasterKey" = 2 <br<br>"Dukpt3Des" = 3,<br><br>"Dukpt3DesCBC" = 4|
|`PinBlock.EmvData`|String|—|Sim|Identificação do KSN<br><br>- Para transações EMV esse campo é obtido através do retorno da função PP_GoOnChip() nas posições 023 até 042;<br><br>- Para transações digitadas e com tarja magnética, verificar as posições 017 até 036 do retorno da função PP_GetPin().|

#### Resposta

```json
{
  "VoidId": "f15889ea-5719-4e1a-a2da-f4e50d5bd702",
  "InitializationVersion": 1558708320029,
  "PrintMessage": [
    {
      "Position": "Top",
      "Message": "Transação autorizada"
    },
    {
      "Position": "Middle",
      "Message": "Informação adicional"
    },
    {
      "Position": "Bottom",
      "Message": "Obrigado e volte sempre!"
    }
  ],
  "Receipt": {
    "MerchantName": "Estabelecimento",
    "MerchantAddress": "Rua Sem Saida, 0",
    "MerchantCity": "Cidade",
    "MerchantState": "WA",
    "MerchantCode": 549798965249,
    "Terminal": 12345678,
    "Nsu": 123456,
    "Date": "01/01/20",
    "Hour": 720,
    "IssuerName": "VISA  PROD-I",
    "CardNumber": 1111222233334444,
    "TransactionType": "CANCELAMENTO DE TRANSACAO",
    "AuthorizationCode": 123456,
    "TransactionMode": "ONL",
    "InputMethod": "C",
    "CancelValue": "3,00",
    "OriginalTransactonData": "DADOS DO PAGAMENTO ORIGNAL",
    "OriginalTransactonType": "VENDA A CREDITO",
    "OriginalTransactonNsu": 5349,
    "OriginalTransactonAuthCode": 543210,
    "OriginalTransactionDate": "01/01/20",
    "OriginalTransactionHour": 720,
    "OrignalTransactionValue": "3,00",
    "OrignalTransactionCardHolder": "NOME NOME NOME NOME NOME N",
    "OriginalTransactionMode": "ONL",
    "OriginalInputMethod": "C"
  },
  "Status": 10,
  "ReturnCode": 0,
  "ReturnMessage": "Success",
  "Links": [
    {
      "Method": "GET",
      "Rel": "self",
      "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702"
    },
    {
      "Method": "POST",
      "Rel": "void",
      "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702/voids"
    },
    {
      "Method": "DELETE",
      "Rel": "reverse",
      "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702/voids/e5c889ea-5719-4e1a-a2da-f4f50d5bd7ca"
    },
    {
      "Method": "PUT",
      "Rel": "confirm",
      "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702/voids/e5c889ea-5719-4e1a-a2da-f4f50d5bd7ca/confirmation"
    }
  ]
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`VoidId`|String - uuid|---|---|Identificador do cancelamento|
|`InitializationVersion`|Integer int16|---|---|Número de versão dos parametros baixados na inicialização do equipamento.|
|`PrintMessage.Position`|String|---|---|Default: "Top"<br><br>Enum: "Top", "Middle", "Bottom"<br><br>Posição da mensagem no comprovante:<br><br>Top - Início do comprovante, antes do código do estabelecimento<br><br>Middle - Meio do comprovante, após a descrição dos valores<br><br>Bottom - Final do comprovante|
|`PrintMessage.Message`|String|---|---|Indica a mensagem que deve ser impressa no comprovante de acordo com a posição indicada no campo Position|
|`Status`|Integerint16|---|---|Status da transação.<br><br>0 = Não Finalizado<br><br>1 = Autorizado<br><br>2 = Pago<br><br>3 = Negado<br><br>10 = Cancelado<br><br>13 = Abortado|
|`CancellationStatus`|Integer int16|---|---|Status do cancelamento.<br><br>0 = Não Finalizado<br><br>1 = Autorizado<br><br>2 = Negado<br><br>3 = Confirmado<br><br>4 = Desfeito|
|`ReasonCode`|Integer int16|---|---|Código de referência para análises.|
|`ReasonMessage`|String|---|---|Mensagem explicativa para análise.|
|`ReturnCode`|String|---|---|Código de erro/resposta da transação da Adquirência.|
|`ReturnMessage`|String|---|---|Mensagem de erro/resposta da transação da Adquirência.|
|`Receipt.MerchantName`|---|---|---|---|
|`Receipt.MerchantAddress`|---|---|---|---|
|`Receipt.MerchantCity`|---|---|---|---|
|`Receipt.MerchantState`|---|---|---|---|
|`Receipt.MerchantCode`|---|---|---|---|
|`Receipt.Terminal`|---|---|---|---|
|`Receipt.Nsu`|---|---|---|---|
|`Receipt.Date`|---|---|---|---|
|`Receipt.Hour`|---|---|---|---|
|`Receipt.IssuerName`|---|---|---|---|
|`Receipt.CardNumber`|---|---|---|---|
|`Receipt.TransactionType`|---|---|---|---|
|`Receipt.AuthorizationCode`|---|---|---|---|
|`Receipt.TransactionMode`|---|---|---|---|
|`Receipt.InputMethod`|---|---|---|---|
|`Receipt.CancelValue`|---|---|---|---|
|`Receipt.OriginalTransactonData`|---|---|---|---|
|`Receipt.OriginalTransactonType`|---|---|---|---|
|`Receipt.OriginalTransactonNsu`|---|---|---|---|
|`Receipt.OriginalTransactonAuthCode`|---|---|---|---|
|`Receipt.OriginalTransactionDate`|---|---|---|---|
|`Receipt.OriginalTransactionHour`|---|---|---|---|
|`Receipt.OrignalTransactionValue`|---|---|---|---|
|`Receipt.OrignalTransactionCardHolder`|---|---|---|---|
|`Receipt.OriginalTransactionMode`|---|---|---|---|
|`Receipt.OriginalInputMethod`|---|---|---|---|
|`Links.Method`|String|---|---|Enum: "POST", "GET", "PUT".<br><br>Método HTTP a ser utilizado na operação.|
|`Links.Rel`|String|---|---|Enum: "self", "cancel", "confirm".<br><br>Referência da operação.|
|`Links.Href`|String|---|---|Endereço de URL de chamada da API|

### Cartão por chip com cartão criptografado

#### Requisição

<aside class="request"><span class="method post">POST</span> <span class="endpoint">/1/physicalSales/{PaymentId}/voids/</span></aside>

**Path Parameters:**

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`PaymentId`|String uuid|---|Sim|Código do Pagamento|

```json
{
  "MerchantVoidId": 2019042204,
  "MerchantVoidDate": "2019-04-15T12:00:00Z",
  "Card": {
    "InputMode": "MagStripe",
    "EmvData": "112233445566778899011AABBC012D3456789E0123FF45678AB901234C5D112233445566778800",
    "TrackOneData": "A1234567890123456^FULANO OLIVEIRA SA ^12345678901234567890123",
    "TrackTwoData": "0123456789012345=012345678901234"
  }
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`MerchantVoidId`|String|---|Sim|Número do documento gerado automáticamente pelo terminal e incrementado de 1 acada transação realizada no terminal|
|`MerchantVoidDate`|String|---|Sim|Data do cancelamento.|
|`Card.InputMode`|String|---|Sim|Enum: "Typed", "MagStripe", "Emv", "ContactlessMagStripe", "ContactlessEmv"|
|`Card.EmvData`|String|---|---|Dados de cancelamento EMV|
|`Card.TrackTwoData`|String|---|---|Dados da trilha 2 <br><br>Obtidos através do comando PP_GetCard na BC no momento da captura da transação|
|`Card.EncryptedCardData.EncryptionType`|String|---|Sim|Tipo de encriptação utilizada<br><br>Enum:<br><br>"DukptDes" = 1,<br><br>"MasterKey" = 2 <br<br>"Dukpt3Des" = 3 <br<br>"Dukpt3DesCBC" = 4|
|`Card.EncryptedCardData.TrackTwoDataKSN`|String|---|---|Identificador KSN da criptografia da trilha 2 do cartão|
|`Card.EncryptedCardData.IsDataInTLVFormat`|Bool|---|Não|Identifica se os dados criptografados estão no formato TLV (tag / length / value).|
|`Card.EncryptedCardData.InitializationVector`|String|---|Sim|Vetor de inicialização da encryptação|
|`Card.AuthenticationMethod`|String|---|Sim|Enum: "NoPassword", "OnlineAuthentication", "OfflineAuthentication"<br><br>Método de autenticação<br><br>1 - Sem senha = “NoPassword”;<br><br>2 - Senha online = “Online Authentication”;<br><br>3 - Senha off-line = “Offline Authentication”.|
|`PinBlock.EncryptedPinBlock`|String|---|Sim|PINBlock Criptografado<br><br>- Para transações EMV, esse campo é obtido através do retorno da função PP_GoOnChip(), mais especificamente das posições 007 até a posição 022;<br><br>- Para transações digitadas e com tarja magnética, verificar as posições 001 até 016 do retorno da função PP_GetPin().|
|`PinBlock.EncryptionType`|String|---|Sim|Tipo de Criptografia<br><br>Enum:<br><br>"DukptDes"<br><br>"Dukpt3Des"<br><br>"MasterKey"|
|`PinBlock.EmvData`|String|---|Sim|Identificação do KSN<br><br>- Para transações EMV esse campo é obtido através do retorno da função PP_GoOnChip() nas posições 023 até 042;<br><br>-Para transações digitadas e com tarja magnética, verificar as posições 017 até 036 do retorno da função PP_GetPin().|

#### Resposta

```json
{
  "VoidId": "f15889ea-5719-4e1a-a2da-f4e50d5bd702",
  "InitializationVersion": 1558708320029,
  "PrintMessage": [
    {
      "Position": "Top",
      "Message": "Transação autorizada"
    },
    {
      "Position": "Middle",
      "Message": "Informação adicional"
    },
    {
      "Position": "Bottom",
      "Message": "Obrigado e volte sempre!"
    }
  ],
  "Receipt": {
    "MerchantName": "Estabelecimento",
    "MerchantAddress": "Rua Sem Saida, 0",
    "MerchantCity": "Cidade",
    "MerchantState": "WA",
    "MerchantCode": 549798965249,
    "Terminal": 12345678,
    "Nsu": 123456,
    "Date": "01/01/20",
    "Hour": 720,
    "IssuerName": "VISA  PROD-I",
    "CardNumber": 1111222233334444,
    "TransactionType": "CANCELAMENTO DE TRANSACAO",
    "AuthorizationCode": 123456,
    "TransactionMode": "ONL",
    "InputMethod": "C",
    "CancelValue": "3,00",
    "OriginalTransactonData": "DADOS DO PAGAMENTO ORIGNAL",
    "OriginalTransactonType": "VENDA A CREDITO",
    "OriginalTransactonNsu": 5349,
    "OriginalTransactonAuthCode": 543210,
    "OriginalTransactionDate": "01/01/20",
    "OriginalTransactionHour": 720,
    "OrignalTransactionValue": "3,00",
    "OrignalTransactionCardHolder": "NOME NOME NOME NOME NOME N",
    "OriginalTransactionMode": "ONL",
    "OriginalInputMethod": "C"
  },
  "Status": 10,
  "ReturnCode": 0,
  "ReturnMessage": "Success",
  "Links": [
    {
      "Method": "GET",
      "Rel": "self",
      "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702"
    },
    {
      "Method": "POST",
      "Rel": "void",
      "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702/voids"
    },
    {
      "Method": "DELETE",
      "Rel": "reverse",
      "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702/voids/e5c889ea-5719-4e1a-a2da-f4f50d5bd7ca"
    },
    {
      "Method": "PUT",
      "Rel": "confirm",
      "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/f15889ea-5719-4e1a-a2da-f4e50d5bd702/voids/e5c889ea-5719-4e1a-a2da-f4f50d5bd7ca/confirmation"
    }
  ]
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`VoidId`|String - uuid|---|---|Identificador do cancelamento|
|`InitializationVersion`|Integer int16|---|---|Número de versão dos parametros baixados na inicialização do equipamento.|
|`PrintMessage.Position`|String|---|---|Default: "Top"<br><br>Enum: "Top", "Middle", "Bottom"<br><br>Posição da mensagem no comprovante:<br><br>Top - Início do comprovante, antes do código do estabelecimento<br><br>Middle - Meio do comprovante, após a descrição dos valores<br><br>Bottom - Final do comprovante|
|`PrintMessage.Message`|String|---|---|Indica a mensagem que deve ser impressa no comprovante de acordo com a posição indicada no campo Position|
|`Status`|Integerint16|---|---|Status da transação.<br><br>0 = Não Finalizado<br><br>1 = Autorizado<br><br>2 = Pago<br><br>3 = Negado<br><br>10 = Cancelado<br><br>13 = Abortado|
|`CancellationStatus`|Integer int16|---|---|Status do cancelamento.<br><br>0 = Não Finalizado<br><br>1 = Autorizado<br><br>2 = Negado<br><br>3 = Confirmado<br><br>4 = Desfeito|
|`ReasonCode`|Integer int16|---|---|Código de referência para análises.|
|`ReasonMessage`|String|---|---|Mensagem explicativa para análise.|
|`ReturnCode`|String|---|---|Código de erro/resposta da transação da Adquirência.|
|`ReturnMessage`|String|---|---|Mensagem de erro/resposta da transação da Adquirência.|
|`Receipt.MerchantName`|---|---|---|---|
|`Receipt.MerchantAddress`|---|---|---|---|
|`Receipt.MerchantCity`|---|---|---|---|
|`Receipt.MerchantState`|---|---|---|---|
|`Receipt.MerchantCode`|---|---|---|---|
|`Receipt.Terminal`|---|---|---|---|
|`Receipt.Nsu`|---|---|---|---|
|`Receipt.Date`|---|---|---|---|
|`Receipt.Hour`|---|---|---|---|
|`Receipt.IssuerName`|---|---|---|---|
|`Receipt.CardNumber`|---|---|---|---|
|`Receipt.TransactionType`|---|---|---|---|
|`Receipt.AuthorizationCode`|---|---|---|---|
|`Receipt.TransactionMode`|---|---|---|---|
|`Receipt.InputMethod`|---|---|---|---|
|`Receipt.CancelValue`|---|---|---|---|
|`Receipt.OriginalTransactonData`|---|---|---|---|
|`Receipt.OriginalTransactonType`|---|---|---|---|
|`Receipt.OriginalTransactonNsu`|---|---|---|---|
|`Receipt.OriginalTransactonAuthCode`|---|---|---|---|
|`Receipt.OriginalTransactionDate`|---|---|---|---|
|`Receipt.OriginalTransactionHour`|---|---|---|---|
|`Receipt.OrignalTransactionValue`|---|---|---|---|
|`Receipt.OrignalTransactionCardHolder`|---|---|---|---|
|`Receipt.OriginalTransactionMode`|---|---|---|---|
|`Receipt.OriginalInputMethod`|---|---|---|---|
|`Links.Method`|String|---|---|Enum: "POST", "GET", "PUT".<br><br>Método HTTP a ser utilizado na operação.|
|`Links.Rel`|String|---|---|Enum: "self", "cancel", "confirm".<br><br>Referência da operação.|
|`Links.Href`|String|---|---|Endereço de URL de chamada da API|

## Desfazimento de cancelamento

| SandBox                                             | Produção                                      |
|:---------------------------------------------------:|:---------------------------------------------:|
| https://apisandbox.cieloecommerce.cielo.com.br      | https://api.cieloecommerce.cielo.com.br/      |

**Simular respostas:**

Para simular alguma resposta especifica utilize o campo Amount, onde de acordo com o valor dos centavos informado nesse campo é possivel receber uma resposta conforme descrito na tabela abaixo:

|Amount (valor dos centados)|Retorno simulado do Cancelamento|Exemplo de valor simulado|
|50|Aprovado|5050 = R$50,40|
|51|Negado|20051 = R$200,41|
|52|Timeout|3552 = R$35,42|
|59|Erro|1059 = R$10,49|

### Desfaz por VoidId

#### Requisição

<aside class="request"><span class="method delete">DELETE</span> <span class="endpoint">/1/physicalSales/{PaymentId}/voids/{VoidId}</span></aside>

**Path Parameters:**

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`PaymentId`|String guid|36|Sim|Código do Pagamento|
|`MerchantVoidId`|String|---|Sim|Número do documento gerado automáticamente pelo terminal e incrementado de 1 a cada transação realizada no terminal|

#### Resposta

```json
{
  "CancellationStatus": 4,
  "Status": 2,
  "ReturnCode": 0,
  "ReturnMessage": "Success",
  "Links": [
    {
      "Method": "GET",
      "Rel": "self",
      "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/fffef2e6-15ef-4493-869f-62ea285fbfde"
    },
    {
      "Method": "POST",
      "Rel": "void",
      "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/fffef2e6-15ef-4493-869f-62ea285fbfde/voids"
    }
  ]
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`CancellationStatus`|Integer|2|Sim|Status do cancelamento. <br><br>0 = Não Finalizado <br><br>1 = Autorizado <br><br>2 = Negado <br><br>3 = Confirmado <br><br>4 = Desfeito|
|`ConfirmationStatuss`|Integer|2|Sim|Status do confirmação. <br><br>0 = Pendente <br><br>1 = Confirmado <br><br>2 = Desfeito|
|`Status`|Integer|2|Sim|Status da transação <br><br>0 = Não Finalizado <br><br>1 = Autorizado <br><br>2 = Pago <br><br>3 = Negado <br><br>10 = Cancelado <br><br>13 = Abortado|
|`ReturnCode`|String|3|Sim|Código de erro/resposta da transação da Adquirência.|
|`ReturnMessage`|String|---|Sim|Mensagem de erro/resposta da transação da Adquirência.|

### Desfaz por MerchantVoidId

#### Requisição

<aside class="request"><span class="method delete">DELETE</span> <span class="endpoint">/1/physicalSales/{PaymentId}/voids/merchantVoidId/{MerchantVoidId}</span></aside>

**Path Parameters:**

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`PaymentId`|String guid|36|Sim|Código do Pagamento|
|`MerchantVoidId`|String|---|Sim|Número do documento gerado automáticamente pelo terminal e incrementado de 1 a cada transação realizada no terminal|

#### Resposta

```json
{
  "CancellationStatus": 4,
  "Status": 2,
  "ReturnCode": 0,
  "ReturnMessage": "Success",
  "Links": [
    {
      "Method": "GET",
      "Rel": "self",
      "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/fffef2e6-15ef-4493-869f-62ea285fbfde"
    },
    {
      "Method": "POST",
      "Rel": "void",
      "Href": "https://api.cieloecommerce.cielo.com.br/1/physicalSales/fffef2e6-15ef-4493-869f-62ea285fbfde/voids"
    }
  ]
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`CancellationStatus`|Integer int16|---|---|Status do cancelamento. <br><br>0 = Não Finalizado <br><br>1 = Autorizado <br><br>2 = Negado <br><br>3 = Confirmado <br><br>4 = Desfeito|
|`ConfirmationStatuss`|Integer|2|Sim|Status do confirmação. <br><br>0 = Pendente <br><br>1 = Confirmado <br><br>2 = Desfeito|
|`Status`|Integer int16|---|---|Status da transação <br><br>0 = Não Finalizado <br><br>1 = Autorizado <br><br>2 = Pago <br><br>3 = Negado <br><br>10 = Cancelado <br><br>13 = Abortado|
|`ReturnCode`|String|---|---|Código de erro/resposta da transação da Adquirência.|
|`ReturnMessage`|---|---|---|Mensagem de erro/resposta da transação da Adquirência.|

# Lojas

Essa operação permite o cadastro de lojas e terminais , viabilizando modelos de negócios onde o facilitador necessite segmentar sua operação.

## Merchant

### POST Merchant - Requisição

Cria um novo merchant.

<aside class="request"><span class="method post">POST</span> <span class="endpoint">/merchants</span></aside>

```json
{
  "Address": {
    "ZipCode": "string",
    "Street": "string",
    "Number": "string",
    "Complement": "string"
  },
  "TradeName": "string",
  "CompanyName": "string",
  "Email": "string",
  "PhoneNumber": "string",
  "Mcc": 0,
  "DocumentNumber": "string",
  "DocumentType": "Cpf",
  "SubordinatedMerchantId": "string",
  "Owner": {
    "Name": "string",
    "Email": "string",
    "PhoneNumber": "string",
    "MessengerPhone": "string",
    "Gender": "Other",
    "DocumentNumber": "string"
  }
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`Address.ZipCode`|String|---|---|---|
|`Address.Street`|String|---|---|---|
|`Address.Number`|String|---|---|---|
|`Address.Number`|String|---|---|---|
|`TradeName`|String|---|---|---|
|`CompanyName`|String|---|---|---|
|`Email`|String|---|---|---|
|`PhoneNumber`|String|---|---|---|
|`Mcc`|String|Integer|---|---|
|`DocumentNumber`|String|---|---|---|
|`DocumentType`|String|---|---|Enum: `Cpf` `Cnpj`|
|`SubordinatedMerchantId`|String|---|---|O ID que a loja subordinada deve assumir.|
|`Owner.Name`|String|---|---|---|
|`Owner.Email`|String|---|---|---|
|`Owner.PhoneNumber`|String|---|---|---|
|`Owner.MessengerPhone`|String|---|---|---|
|`Owner.Gender`|String|---|---|Enum: `Other` `Male` `Female`|

### GET Merchant - Resposta

Encontra uma loja subordinada pelo seu ID.

<aside class="request"><span class="method get">GET</span> <span class="endpoint">/merchants/{subordinatedMerchantId}</span></aside>

```json
{
  "Merchant": {
    "SubordinatedMerchantId": "string",
    "Owner": {
      "Name": "string",
      "Email": "string",
      "PhoneNumber": "string",
      "MessengerPhone": "string",
      "Gender": "Other",
      "DocumentNumber": "string"
    }
  }
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`SubordinatedMerchantId`|---|---|---|---|
|`Owner.Name`|String|---|---|---|
|`Owner.Email`|String|---|---|---|
|`Owner.PhoneNumber`|String|---|---|---|
|`Owner.MessengerPhone`|String|---|---|---|
|`Owner.Gender`|String|---|---|Enum: `Other` `Male` `Female`|
|`Owner.DocumentNumber`|---|---|---|

### PUT Merchant - Requisição

Salva alterações em na loja subordinado com o ID especificado.

<aside class="request"><span class="method put">PUT</span> <span class="endpoint">/merchants/{subordinatedMerchantId}</span></aside>

```json
{
  "Address": {
    "ZipCode": "string",
    "Street": "string",
    "Number": "string",
    "Complement": "string"
  },
  "TradeName": "string",
  "CompanyName": "string",
  "Email": "string",
  "PhoneNumber": "string",
  "Mcc": 0,
  "DocumentNumber": "string",
  "DocumentType": "Cpf"
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`Address.ZipCode`|String|---|---|---|
|`Address.Street`|String|---|---|---|
|`Address.Number`|String|---|---|---|
|`Address.Number`|String|---|---|---|
|`TradeName`|String|---|---|---|
|`CompanyName`|String|---|---|---|
|`Email`|String|String|---|---|
|`PhoneNumber`|String|---|---|---|
|`Mcc`|String|Integer|---|---|
|`DocumentNumber`|String|---|---|---|
|`DocumentType`|String|---|---|Enum: `Cpf` `Cnpj`|

## Terminal

### Cria um novo terminal

#### Requisição

<aside class="request"><span class="method post">POST</span> <span class="endpoint">/terminals</span></aside>

```json
{
  "CommunicationType": "string",
  "EquipmentModel": 0,
  "EquipmentSerialNumber": "string",
  "TerminalId": "string",
  "SubordinatedMerchantId": "string"
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`CommunicationType`|String|---|---|---|
|`EquipmentModel`|Integer|---|---|---|
|`EquipmentSerialNumber`|String|---|---|---|
|`TerminalId`|String|---|---|---|
|`SubordinatedMerchantId`|String <uuid>|---|---|ID da loja subordinada vinculada ao terminal|

#### Resposta

```json
{
  "Terminal": {
    "CommunicationType": "string",
    "EquipmentModel": 0,
    "EquipmentSerialNumber": "string",
    "TerminalId": "string",
    "SubordinatedMerchantId": "string"
  }
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`TerminalBaseModel.CommunicationType`|String|---|---|---|
|`TerminalBaseModel.EquipmentModel`|Integer|---|---|---|
|`TerminalBaseModel.EquipmentSerialNumber`|String|---|---|---|
|`TerminalBaseModel.TerminalId`|String|---|---|---|
|`Terminal.SubordinatedMerchantId`|String uuid|---|---|---|

### Desabilita um terminal removendo o vínculo com um equipamento físico.

#### Requisição

<aside class="request"><span class="method post">POST</span> <span class="endpoint">/terminals/disable</span></aside>

```json
{
  "SubordinatedMerchantId": "string",
  "TerminalId": "string"
}
```

|Propriedade|Tipo|Tamanho|Obrigatório|Descrição|
|---|---|---|---|---|
|`SubordinatedMerchantId`|String <uuid>|---|---|ID da loja subordinada vinculada ao terminal|
|`TerminalId`|String|---|---|---|