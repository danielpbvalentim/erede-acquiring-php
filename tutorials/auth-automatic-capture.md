# Integração Rede – Autorização com Captura automática

O processo de integração com a Rede passa, necessariamente, pela autorização e, eventualmente, pela captura. Autorização é o processo de verificação, junto ao banco emissor do cartão, se o cliente tem limite disponível para aquela transação. A captura é o processo de debitar o valor autorizado da transação e pode ocorrer a qualquer momento dentro do prazo de validade de 30 dias da autorização.

Muitas vezes é altamente conveniente para a loja que a captura seja feita automaticamente, assim que a autorização é feita com sucesso. Isso pode ocorrer, por exemplo, em operações como lanchonetes, pizzarias e/ou em quaisquer situações onde a captura do valor autorizado deva ocorrer tão logo a transação tenha sido autorizada. Nesse tutorial vamos fazer a integração com a Rede utilizando o SDK PHP para fazer uma autorização com captura automática.

## Fluxo de integração

Após a escolha e adição de seus produtos ao carrinho, o cliente procederá para a tela de checkout, onde informará os dados de seu cartão. Com os dados do cartão do cliente, a loja enviará uma mensagem de autorização com captura automática para a Rede, que por sua vez verificará junto ao banco emissor se o cliente possui limite disponível para aquela transação e retornará o status da autorização.

Caso a transação tenha sido autorizada, a Rede retornará o código de autorização e o identificador da transação; do contrário, o motivo da não autorização será retornado.

<p align="center">
    <img src="https://gist.githubusercontent.com/marcoescudeiro/6c9113c59df71bb19b61/raw/11685c0bd12931354e4471a6c5f796d218879dc7/fluxo-rede.png" />
</p>

1. Cliente adiciona um ou mais produtos ao carrinho.
2. Cliente finaliza pedido e procede para a página de checkout.
3. Cliente informa os dados de seu cartão.
4. Loja envia a mensagem de autorização com captura automática para a Rede.
5. Rede verifica junto ao banco emissor se há limite disponível para aquela transação.
6. Rede retorna o status da autorização para a loja.
7. Loja avisa ao cliente sobre o status da transação e, se autorizado, libera o pedido do cliente.

## Mensagem de autorização com captura automática

A mensagem de autorização com captura automática segue o seguinte padrão:

```php
$data = array('credit_card' => '5448280000000007',
              'exp_month' => '01',
              'exp_year' => 2016,
              'cvv' => "132",
              'amount' => '1050',
              'capture' => true, //define a autorização com captura automática
              'installments' => 4, //número de parcelas
              'reference' => uniqid() //identificador único do pedido
);
```
Os campos `credit_card`, `exp_month`, `exp_year` e `cvv` identificam o número do cartão, mês e ano de expiração e o código verificador do cartão. O campo `amount` é o valor da transação em **centavos**, `installments` especifica o número de parcelas ou 1 para transações à vista. O campo `reference` é o **identificador único** do pedido na plataforma de vendas da loja; esse campo é **obrigatório** e deve ser enviado para a Rede sempre que a loja enviar uma nova mensagem de autorização.

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

## Utilizando o SDK para fazer uma autorização e captura automática

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

Agora basta definir os parâmetros da integração, informando os dados do cartão, valor da transação, número do pedido e informando que queremos a captura automática:

```php
$data = array('credit_card' => '5448280000000007',
              'exp_month' => '01',
              'exp_year' => 2016,
              'cvv' => "132",
              'amount' => '1050',
              'capture' => true,
              'installments' => 4,
              'reference' => uniqid());
```

Com os dados da transação definidos, solicitamos a autorização utilizando o método `authorize`:

```php
$response = $transaction->authorize($data);
```

## Caso de uso – Pizzaria

Compreendido o funcionamento do SDK, vamos utilizá-lo para implementar a página de pagamento de uma pizzaria, onde após o cliente escolher sua pizza, informa os dados do cartão de crédito e fazemos a autorização com captura automática.

Para essa implementação utilizaremos componente Javascript para deixar a tela de inserção dos dados do cartão mais amigável. Você pode instalá-lo via `npm` ou fazer um clone do repositório `jessepollak/card` no github.

Com a biblioteca no projeto, vamos criar a página de checkout:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Exemplo integração Rede</title>
    <meta name="viewport" content="initial-scale=1">
    <link rel="stylesheet" href="css/sample.css">
</head>
<body>
    <div class="demo-container">
        <div class="cart-area">
            <table class="table-cart">
                <thead>
                    <tr>
                        <th>Produto</th>
                        <th>Quantidade</th>
                        <th>Total</th>
                    </tr>
                </thead>
                <tbody>
                    <tr>
                        <td>Pizza</td>
                        <td>1</td>
                        <td>R$ 60,00</td>
                    </tr>

                    <tr>
                        <td>Refrigerante</td>
                        <td>1</td>
                        <td>R$ 4,00</td>
                    </tr>

                    <tr>
                        <td>Taxa de entrega</td>
                        <td>1</td>
                        <td>R$ 4,00</td>
                    </tr>
                </tbody>
                <tfoot>
                    <tr>
                        <td></td>
                        <td></td>
                       <td class="total">R$ 68,00</td>
                    </tr>
                </tfoot>
            </table>
        </div>
        <div class="card-area">
            <div class="card-wrapper"></div>

            <div class="form-container active">
                <form action="auth.php" method="post">
                    <input placeholder="Número do cartão" type="text" name="number">
                    <input placeholder="Nome completo" type="text" name="name">
                    <input placeholder="MM/YY" type="text" name="expiry">
                    <input placeholder="CVC" type="text" name="cvc">
                    <input type="submit" value="Pagar">
                </form>
            </div>
        </div>
    </div>

    <script src="js/card.js"></script>
    <script>
        new Card({
            form: document.querySelector('form'),
            container: '.card-wrapper',
            placeholders: {
                number: '•••• •••• •••• ••••',
                name: 'Nome do portador',
                expiry: 'MM/YYYY',
                cvc: '•••'
            }
       });
    </script>
</body>
</html>
```

Assim que o cliente informar os dados do cartão na página de checkout, faremos a submissão do formulário para o arquivo que fará a integração para a autorização com captura. O arquivo `auth.php` utiliza os dados do cartão para popular os campos do SDK como descrito anteriormente:

```php
<?php
require '../vendor/autoload.php';

use ERede\Acquiring\Acquirer;
use ERede\Acquiring\TransactionType;
use ERede\Acquiring\TransactionStatus;

$filiation = 'xxxxxx';
$password = 'yyyy';
$environment = 'homolog';

$acquirer = new Acquirer($filiation, $password, $environment);

# just choose that is a credit transaction
$transaction = $acquirer->fetch(TransactionType::CREDIT);

$expiry = explode(' / ', $_POST['expiry']);

# setup the data to send to the service
$data = array('credit_card' => preg_replace('/[^\d]/', '', $_POST['number']),
              'exp_month' => $expiry[0],
              'exp_year' => $expiry[1],
              'cvv' => $_POST['cvc'],
              'amount' => '6800',
              //ativa a captura automática
              'capture' => true,
              'installments' => 1,
              'reference' => uniqid() //identificador único do pedido
);

# do the transaction
$response = $transaction->authorize($data);

if ($response->status == TransactionStatus::SUCCESS) {
    require 'success.php';
} else {
    require 'error.php';
}
```

Em caso de sucesso, a página success.php é carregada, mostrando que tudo ocorreu bem. Em caso de falha, a página error.php é carregada, mostrando para o cliente o motivo do erro.

## Conclusão

Como pudemos ver, a integração com a Rede utilizando o SDK PHP é bastente simples. Como o SDK abstrai toda a complexidade da integração, troca de mensagens, formatação, etc, conseguimos fazer uma integração em apenas alguns instantes, informando os valores solicitados e, no caso da autorização com captura automática, apenas informando capture=true antes de enviar a requisição de autorização.
