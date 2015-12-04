# Integração Rede – Autorização com captura posterior

Como mostramos no primeiro artigo da série Integração Rede, todo o processo de integração passa, necessariamente, pela autorização; ainda, que a autorização é o processo de verificação, junto ao banco emissor do cartão, se o cliente tem limite disponível para aquela transação. Neste artigo trataremos da autorização com captura posterior, ou seja, o vendedor fará a verificação de limite e reserva de fundos com o banco emissor e fará a captura em um segundo momento, dentro do prazo de validade de 30 dias da autorização, quando for mais apropriado.

Esse tipo de autorização e captura pode acontecer em diversos cenários, por exemplo, de um hotel ou de uma companhia aérea que faz a autorização no momento da reserva do cliente, mas só faz a captura no momento do check-in, ou, ainda, quando a empresa possui um serviço de anti-fraude. Nesse tutorial mostraremos o fluxo de integração de um hotel, onde a autorização será feita no momento da reserva e a captura no momento do check-in.

## Fluxo de integração

<p align="center">
    <img src="https://gist.githubusercontent.com/marcoescudeiro/00fb40991ad9035ad6c5/raw/650dd6927db6a3fae8a3415c3540858e31475215/auth.png" />
</p>

### Autorização

Após a reserva, o cliente procederá para a tela de pagamento, onde informará os dados do seu cartão. Com os dados do cartão do cliente, a loja enviará uma mensagem de autorização para a Rede, informando que o valor não deverá ser capturado, que por sua vez verificará junto ao banco emissor se o cliente possui limite disponível para aquela transação e retornará o status da autorização.

Ao realizar a autorização com sucesso, o valor da transação fica reservado no saldo do portador mas ainda não será efetivamente debitada até que seja capturada. Caso a transação tenha sido autorizada, a Rede retornará o código de autorização e o identificador da transação; do contrário, o motivo da não autorização será retornado.

1. Cliente faz a reserva de um quarto;
2. Cliente finaliza reserva;
3. Cliente informa os dados do cartão;
4. Hotel envia mensagem de autorização para a Rede;
5. Rede verifica junto ao banco emissor se há limite disponível para aquela transação;
6. Rede retorna código da autorização para o hotel;
7. Hotel avisa ao cliente sobre o status da transação e, se autorizado, efetua a reserva do quarto.

### Captura

Assim que o cliente for fazer o check-in, o hotel deverá enviar uma mensagem de captura para a Rede, informando o código de autorização recebido no passo anterior. A autorização será válida por 30 dias, ou até que seja capturada.

1. Cliente faz check-in no hotel dentro do prazo de validade da autorização;
2. Hotel envia a mensagem de captura para a Rede com o TID, quantidade de parcelas e o valor autorizado para ser capturado;
3. Rede retorna o status da mensagem de captura para o hotel.

## Mensagens de autorização e de captura

### Autorização

A mensagem de autorização o seguinte padrão:

```php
<?php
//...
$data = array('credit_card' => '5448280000000007',
              'exp_month' => '01',
              'exp_year' => 2016,
              'cvv' => "132",
              'amount' => '1050',
              'capture' => false, //define que a captura será feita depois
              'installments' => 1, //número de parcelas
              'reference' => uniqid() //identificador único do pedido
);
```

Os campos `credit_card`, `exp_month`, `exp_year` e `cvv` identificam o número do cartão, mês e ano de expiração e o código verificador do cartão. O campo `amount` é o valor da transação em centavos, `installments` especifica o número de parcelas ou 1 para transações à vista. O campo `reference` é o identificador único do pedido na plataforma de vendas da loja; esse campo é **obrigatório** e deve ser enviado para a Rede sempre que a loja enviar uma nova mensagem de autorização. Por se tratar de captura posterior, o parâmetro `capture` **deve, necessariamente**, ser enviado como `false`.

### Captura

A mensagem de captura segue o seguinte padrão:

```php
<?php
//...
$data = array('tid' => '123456',
              'installments' => 1,
			  'amount' => '1050'
);
```

O campo `tid` deve conter exatamente o mesmo valor retornado pela rede na mensagem de autorização. O campo `amount` especifica o valor que será capturado; o valor enviado nesse campo deverá ser exatamente o mesmo que foi enviado na fase da autorização. O campo `installments` especifica o total de parcelas em que o valor capturado será dividido.

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

## Utilizando o SDK para fazer uma autorização

O primeiro passo é criar uma instância de `Acquirer`, que é onde informaremos nossa chave de filiação, senha e o ambiente onde trabalharemos – produção ou homologação:

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

Uma vez criada a instância de Acquirer, definimos qual o tipo de transação iremos trabalhar. Isso é feito através do método `Acquirer::fetch`, passando `TransactionType::CREDIT` como parâmetro:

```php
$transaction = $acquirer->fetch(TransactionType::CREDIT);
```

Agora basta definir os parâmetros da integração, informando os dados do cartão, valor da transação e número do pedido, informando que **não faremos a captura de forma automática** definindo o parâmetro `capture` como `false`:

```php
$data = array('credit_card' => '5448280000000007',
              'exp_month' => '01',
              'exp_year' => 2016,
              'cvv' => "132",
              'amount' => '1050',
              'capture' => false, //não queremos captura automática
              'installments' => 1,
              'reference' => uniqid());
```

Com os dados da transação definidos, solicitamos a autorização utilizando o método `authorize`:

```php
$response = $transaction->authorize($data);
```

Ao responder a mensagem de autorização, em caso de sucesso, a Rede retornará o parâmetro `tid`, que é o identificador único da transação. Seja um hotel, seja uma companhia aérea ou uma loja que possui um serviço anti-fraude, esse identificador deverá ser armazenado num banco de dados pois será, através do `tid`, que conseguiremos fazer a captura posterior.

No momento em que for apropriado, respeitando o prazo limite de 30 dias de validade da autorização, o hotel enviará a mensagem de captura utilizando esse identificador. Por se tratar de um segundo momento, novamente precisaremos criar uma instância de `Acquirer` como descrito anteriormente e definiremos que estamos trabalhando com uma transação de crédito, utilizando ométodo `Acquirer::fetch` e passando `TransactionType::CREDIT` como parâmetro:

```php
<?php
require 'vendor/autoload.php';

use ERede\Acquiring\Acquirer;
use ERede\Acquiring\TransactionType;

$filiation = 'xxxxxx'; //Chave de afiliação junto a Rede
$password = 'yyyy'; //Senha do vendedor
$environment = 'homolog'; //tipo de ambiente

$acquirer = new Acquirer($filiation, $password, $environment);
$transaction = $acquirer->fetch(TransactionType::CREDIT);
```

Com o SDK preparado para essa etapa da integração, definimos os parâmetros da captura, informando o valor que foi autorizado para captura, número de parcelas e o identificador único da transação - `tid`:

```php
$data = array('tid' => '123456',
              'installments' => 1,
			  'amount' => '1050'
);
```

Pronto, tudo o que precisamos agora é enviar a mensagem de captura para a Rede:

```php
$response = $transaction->capture($data);
```

## Conclusão

Como pudemos ver, a integração com a Rede utilizando o SDK PHP é bastente simples. Como o SDK abstrai toda a complexidade da integração, troca de mensagens, formatação, etc, conseguimos fazer uma integração em apenas alguns instantes, informando os valores solicitados, tipo de autorização, captura automática e posterior, etc.
