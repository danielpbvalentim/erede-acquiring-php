# Integração Rede – Consultando transações

Nos dois artigos anteriores da série Integração Rede, mostramos os processos de integração com captura automática e com captura manual. Esses artigos destacaram a importância do envio do campo `reference`, que é o identificador único na plataforma de vendas da loja. Neste artigo abordaremos a pesquisa de transações que retornará, entre vários outros campos, o status da transação.

Esse tipo de operação de consulta é de extrema importância, seja para verificar o status da autorização, seja para a criação/funcionamento de um backoffice. É uma das funcionalidades mais simples de se integrar, mas é também um dos recursos mais importantes no dia a dia da operação.

## Mensagem de consulta

A mensagem de consulta segue o seguinte padrão:

```php
$data = array(
    'tid' => '1234',
    'reference' => '123456'
);
```

O campo `tid` é o mesmo identificador da transação retornado durante a autorização. O campo `reference` é o identificador único do pedido na plataforma da loja. Para fazer uma consulta, um campo ou o outro deve ser informado; não há a necessidade de informar os dois ao mesmo tempo, mas é obrigatório que pelo menos um deles seja enviado para a operação de consulta.

Ao receber a requisição de consulta, a Rede retornará uma série de informações sobre a transação, caso a mesma tenha sido localizada. A tabela abaixo lista todos os campos retornados pela Rede:

|Campo|Tamanho|Tipo|Descrição|
|-----|-------|----|---------|
|`reference`|16|Alfaumérico|Número do pedido gerado pelo estabelecimento.|
|`tid`|20|Numérico|Número identificador único da transação original.|
|`authorization_number`|6|Numérico|Número de Autorização retornado pelo Emissor do cartão na transação de original.|
|`nsu`|12|Numérico|Número sequencial retornado pela Rede (NSU).|
|`date`|8|Numérico|Data em que foi realizada a transação que está sendo consultada. Será informada no formato AAAAMMDD.|
|`time`|8|Alfanumérico|Hora em que foi realizada a transação que está sendo consultada. Será informada no formato HH:MM:SS|
|`transaction`|2|Numérico|Tipo de Transação. Para os valores válidos vide tabela ‘Tipo de transação’ no inicio do documento no item Transações de Crédito.|
|`installments`|2|Numérico|Número de parcelas. Deve ser enviado somente para os tipos de transação 06 (Parcelado com juros), 08 (Parcelado sem juros) e 40 (Transação IATA Parcelada sem juros – apenas para cias aéreas).|
|`amount`|10|Numérico|Valor total da compra, sem separador de milhar e com até duas casas decimais separados por ponto (Ex: 34.60).|
|`departure_tax`|10|Numérico|Taxa de embarque. Obrigatório para os tipos de transação 39 (IATA à vista) e 40 (IATA parcelado sem juros).|
|`currency`|4|Alfabético|Moeda em que a transação original foi realizada. Retornará sempre “Real”.|
|`return_code`|2|Numérico|Código de retorno da transação. Para os códigos possíveis vide tabela do item 15.|
|`message`|100|Alfanumérico|Mensagem de retorno da transação. Para as possíveis mensagens, vide tabelas dos itens 14 e 15.|
|`status`|1|Numérico|Status na data e hora da requisição da query da transação que está sendo consultada. As possibilidades de retorno são: 1 – Aprovada; 2 – Não aprovada; 3 – Estornada; 4 – Pendente|
|`authorization_date`|8|Numérico|Data em que a pré-autorização foi confirmada/capturada. Será informada no formato AAAAMMDD.|
|`authorization_expiration_date`|8|Numérico|Data em que a pré-autorização irá expirar caso não seja confirmada/capturada. Será informada no formato AAAAMMDD.|
|`cancel_date`|8|Numérico|Data de cancelamento da transação. Será informada no formato AAAAMMDD.|
|`card_holder_name`|30|Alfanumérico|Nome do Portador.|
|`soft_descriptor`|13|Alfanumérico|Mensagem que será exibida ao lado do nome do estabelecimento na fatura do portador. Para utilizar esse recurso é necessário habilitá-lo através do portal https://www.userede.com.br > e-Rede > Configuracoes > Nome na fatura.|
|`credit_card_number`|16|Numérico|Número do cartão mascarado da seguinte maneira: 123456XXXXXX1234|

## Instalando o SDK via Composer

Se você já tem arquivo composer.json, então basta adicionar o fragmento abaixo e, então, fazer um `composer install`.

```json
{
  "require": {
    "rede/acquiring": "1.0.1"
  }
}
```

Se você ainda não tem um arquivo `composer.json`, então você pode simplesmente fazer um `composer require "rede/acquiring"`.

## Utilizando o SDK para fazer uma consulta de transação

O primeiro passo é criar uma instância de `Acquirer`, que é onde informaremos nossa chave de filiação, senha e o ambiente onde trabalharemos – produção ou homologação.

```php
<?php
require 'vendor/autoload.php';

use ERede\Acquiring\Acquirer;
use ERede\Acquiring\TransactionType;

$filiation = 'xxxxxx'; //Chave de afiliação junto a Rede
$password = 'yyyy'; //Senha do vendedor
$environment = 'homolog'; //tipo de ambiente

$acquirer = new Acquirer($filiation, $password, $environment);
```

Uma vez criada a instância de Acquirer, definimos qual o tipo de transação iremos trabalhar. Isso é feito através do método Acquirer::fetch, passando TransactionType::CREDIT como parâmetro:

```php
$transaction = $acquirer->fetch(TransactionType::CREDIT);
```

Agora basta definir os parâmetros da integração, informando o campo `reference`, que é o identificador único do pedido que se deseja pesquisar, ou o campo `tid`, que é o identificador da transação retornado pela Rede durante o processo de autorização:

```php
$data = array('reference' => '123456');
```

Com o critério da pesquisa definido, basta executar o método `find` do SDK:

```php
$response = $transaction->find($data);
```

## Conclusão

Como pudemos ver, a integração com a Rede utilizando o SDK PHP é bastente simples. Como o SDK abstrai toda a complexidade da integração, troca de mensagens, formatação, etc, conseguimos fazer uma integração em apenas alguns instantes, informando os valores solicitados, tipo de autorização, captura automática e posterior, consulta de transações através do campo `reference` ou do identificador da transação `tid`, etc.