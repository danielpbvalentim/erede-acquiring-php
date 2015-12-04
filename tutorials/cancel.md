# Integração Rede – Cancelamento de transações

Nos últimos artigos da série Integração Rede, mostramos os processos de integração de autorização com captura automática, captura manual e consulta de transações. Nesse artigo falaremos sobre uma outra operação que faz parte do dia a dia da loja: o cancelamento de transações.

Você certamente já deve ter ouvido falar - e provavelmente ficou em dúvidas - sobre quando utilizar o cancelamento e quando utilizar o estorno. Esses dois termos causam tanta confusão, de fato, que a Rede decidiu simplificar, tratando tudo como cancelamento de transações.

## Regras para o cancelamento de transações

As regras para o cancelamento são as seguintes:

1. O cancelamento da autorização pode ser solicitada durante todo o período de validade da autorização, desde que ela não tenha sido capturada; ou seja, durante o período em que a transação estiver pendente de captura.
3. O cancelamento de autorização com captura automática só pode ser solicitada no mesmo dia em que a transação de captura foi realizada, isto é, até as 23:59h do horário oficial de Brasília.
2. O cancelamento de uma captura só pode ser feito no mesmo dia em que a transação de captura foi realizada, isto é, até as 23:59h do horário oficial de Brasília.
4. O cancelamento de uma captura após as 23:59 do dia da captura só pode ser feita através do Cancelamento de Vendas, dentro Portal da Rede, na aba Serviços > Cancelamento de Vendas, via Central de atendimento ou com o arquivo EDI.

## Mensagem de cancelamento

Compreendidas as regras para o cancelamento de transações, a mensagem de cancelamento segue o seguinte padrão:

```php
$data = array(
    'tid' => '123456'
);
```

 O campo tid é o mesmo identificador da transação retornado durante a autorização e que, como foi dito nos artigos anteriores, deve ser armazenado pela plataforma da loja. Ao receber a requisição de cancelamento, a Rede retornará um código com o retorno da requisição, uma mensagem descritiva e o mesmo `tid` enviado.
 
## Instalando o SDK via Composer

Se você já tem arquivo composer.json, então basta adicionar o fragmento abaixo e, então, fazer um `composer install`.

```json
{
  "require": {
    "rede/acquiring": "1.0.2"
  }
}
```

Se você ainda não tem um arquivo `composer.json`, então você pode simplesmente fazer um `composer require "rede/acquiring"`.

## Utilizando o SDK para fazer o cancelamento de uma transação

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

Agora basta definir os parâmetros da integração, informando o campo `tid`, que é o identificador da transação retornado pela Rede durante o processo de autorização, da transação que será cancelada:

```php
$data = array(
    'tid' => '123456'
);
```

Com o identificador da transação definido, basta executar o método `cancel` do SDK:

```php
$response = $transaction->cancel($data);
```

## Conclusão

Como pudemos ver durante toda essa série, a integração com a Rede utilizando o SDK PHP é bastente simples. Como o SDK abstrai toda a complexidade da integração, troca de mensagens, formatação, etc, conseguimos fazer uma integração em apenas alguns instantes, informando os valores solicitados, tipo de autorização, captura automática e posterior, consulta de transações através do campo `reference` ou do identificador da transação `tid` e, por fim, o cancelamento de transações.
