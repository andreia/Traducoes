O Componente HttpFoundation
===========================

Antes de mergulhar na refatoração do código, primeiro, eu quero voltar um passo e 
verificar porque você gostaria de usar um framework em vez de manter as suas
aplicações no bom e velho PHP. Por que usar um framework é realmente uma boa
idéia, até mesmo para o mais simples trecho de código e por que criar o seu framework 
utilizando os componentes do Symfony2 é melhor do que criar um framework do
zero.

.. note::

    Eu não vou falar sobre os benefícios óbvios e tradicionais de usar um
    framework ao trabalhar em grandes aplicações com mais do que alguns
    desenvolvedores; a Internet já possui bons e abundantes recursos sobre este
    tópico.

Mesmo sendo bastante simples a "aplicação" que nós escrevemos ontem, ela possui
alguns problemas::

    <?php

    // framework/index.php

    $input = $_GET['name'];

    printf('Hello %s', $input);

Primeiro, se o parâmetro de consulta ``name`` não for fornecido na query string da URL,
você receberá um aviso PHP, por isso, vamos corrigí-lo::

    <?php

    // framework/index.php

    $input = isset($_GET['name']) ? $_GET['name'] : 'World';

    printf('Hello %s', $input);

Então, esta *aplicação não é segura*. Dá pra acreditar? Mesmo este simples
trecho de código PHP é vulnerável a uma das questões de segurança mais difundidas 
na Internet, o XSS (Cross-Site Scripting). Aqui está uma versão mais segura::

    <?php

    $input = isset($_GET['name']) ? $_GET['name'] : 'World';

    header('Content-Type: text/html; charset=utf-8');

    printf('Hello %s', htmlspecialchars($input, ENT_QUOTES, 'UTF-8'));

.. note::

    Como você deve ter notado, proteger o seu código com o ``htmlspecialchars`` é algo
    tedioso e propenso à erros. Essa é uma das razões pelas quais usar uma template
    engine como o `Twig`_, onde o auto-escaping é ativado por padrão, pode ser uma
    boa idéia (e escapar explicitamente é também menos penoso com o uso de um
    filtro simples ``e``).

Como você pode verificar por si mesmo, o código simples que tínhamos escrito anteriormente não é 
mais simples, se quisermos evitar os avisos (warnings/notices) do PHP e tornar o código
mais seguro.

Além da segurança, este código não é facilmente testável. Mesmo se não houver
muito para testar, parece-me que, escrever testes de unidade para o mais simples possível
trecho de código PHP não é algo natural e parece desagradável. Aqui está uma tentativa de 
teste unitário no PHPUnit para o código acima::

    <?php

    // framework/test.php

    class IndexTest extends \PHPUnit_Framework_TestCase
    {
        public function testHello()
        {
            $_GET['name'] = 'Fabien';

            ob_start();
            include 'index.php';
            $content = ob_get_clean();

            $this->assertEquals('Hello Fabien', $content);
        }
    }

.. note::

    Se a nossa aplicação for apenas um pouco maior, poderíamos 
    encontrar ainda mais problemas. Se você está curioso sobre eles, leia o capítulo `Symfony2 
    versus o PHP puro`_ na documentação do Symfony2.

Neste ponto, se você ainda não está convencido de que a segurança e os testes são de fato
duas boas razões para parar de escrever código na forma antiga e adotar um framework
no lugar (independentemente do que signifique adotar um framework neste contexto), você pode parar
de ler esta série agora e voltar para qualquer código que você estava trabalhando
antes.

.. note::

    É claro, usar um framework deve dar-lhe mais do que apenas segurança e
    testabilidade, mas a coisa mais importante a ter em mente é que o
    framework que você escolher deve permitir que você escreva código melhor e mais rápido.

Iniciando OOP com o Componente HttpFoundation
---------------------------------------------

Escrever código web significa interagir com o HTTP. Então, os princípios fundamentais
de nosso framework devem ser centrados em torno da `especificação
HTTP`_.

A especificação HTTP descreve como um cliente (um navegador, por exemplo)
interage com um servidor (a nossa aplicação através de um servidor web). O diálogo entre
o cliente e o servidor é especificado por *mensagens* bem definidas, pedidos e
e respostas: *o cliente envia um pedido para o servidor e com base neste
pedido, o servidor retorna uma resposta*.

No PHP, o pedido é representado por variáveis ​​globais (``$_GET``, ``$_POST``,
``$_FILE``, ``$_COOKIE``, ``$_SESSION``...) E a resposta é gerada por
funções (``echo``, ``header``, ``setcookie``, ...).

O primeiro passo para um código melhor é, provavelmente, usar uma abordagem Orientada à 
Objeto, que é o objetivo principal do componente HttpFoundation do Symfony2:
substituindo as variáveis ​​globais e funções padrão do PHP por uma camada Orientada à 
Objeto.

Para usar este componente, abra o arquivo ``composer.json`` e, adicione-o como uma
dependência para o projeto:

.. code-block:: json

    {
        "require": {
            "symfony/class-loader": "2.1.*",
            "symfony/http-foundation": "2.1.*"
        }
    }

Em seguida, execute o comando ``update`` do composer:

.. code-block:: sh

    $ php composer.phar update

Finalmente, na parte inferior do arquivo ``autoload.php``, adicione o código necessário para
fazer o autoload do componente::

    <?php

    // framework/autoload.php

    $loader->registerNamespace('Symfony\\Component\\HttpFoundation', __DIR__.'/vendor/symfony/http-foundation');

Agora, vamos reescrever a nossa aplicação usando as classes ``Request`` e 
``Response``::

    <php

    // framework/index.php

    require_once __DIR__.'/autoload.php';

    use Symfony\Component\HttpFoundation\Request;
    use Symfony\Component\HttpFoundation\Response;

    $request = Request::createFromGlobals();

    $input = $request->get('name', 'World');

    $response = new Response(sprintf('Hello %s', htmlspecialchars($input, ENT_QUOTES, 'UTF-8')));

    $response->send();

O método``createFromGlobals()`` cria um objeto ``Request`` baseado nas
variáveis ​​globais atuais do PHP.

O método ``send()`` envia o objeto ``Response`` de volta para o cliente (que
primeiro exibe os cabeçalhos HTTP seguidos do conteúdo).

.. tip::

    Antes de chamar ``send()``, nós devemos acrescentar uma chamada para o
    método ``prepare()`` (``$response->prepare($request);``) para garantir que
    a nossa Resposta é compatível com a especificação HTTP. Por exemplo, se
    chamarmos a página com o método ``HEAD``, ele teria removido
    o conteúdo da Resposta.

A principal diferença do código anterior é que você tem controle total das
mensagens HTTP. Você pode criar qualquer pedido que desejar e você está 
encarregado de enviar a resposta, sempre que você considerar oportuno.

.. note::

    Não vamos definir explicitamente o cabeçalho ``Content-Type`` na reescrita
    do código pois o charset padrão do objeto ``Response`` é ``UTF-8``.

Com a classe ``Request``, você tem todas as informações do pedido ao
seu alcance, graças a uma API simples e atraente::

    <?php

    // A URI que está sendo solicitada (ex.: /about) menos quaisquer parâmetros de consulta (query)
    $request->getPathInfo();

    // recuperar as variáveis ​​GET e POST, respectivamente
    $request->query->get('foo');
    $request->request->get('bar', 'default value if bar does not exist');

    // recuperar as variáveis ​​SERVER
    $request->server->get('HTTP_HOST');

    // recuperar uma instância de UploadedFile identificado por foo
    $request->files->get('foo');

    // recuperar um valor de COOKIE
    $request->cookies->get('PHPSESSID');

    // recuperar um cabeçalho de pedido HTTP, com chaves normalizadas e minúsculas
    $request->headers->get('host');
    $request->headers->get('content_type');

    $request->getMethod();    // GET, POST, PUT, DELETE, HEAD
    $request->getLanguages(); // um array dos idiomas que o cliente aceita

Você também pode simular um pedido::

    $request = Request::create('/index.php?name=Fabien');

Com a classe ``Response``, você pode facilmente ajustar a resposta::

    <?php

    $response = new Response();

    $response->setContent('Hello world!');
    $response->setStatusCode(200);
    $response->headers->set('Content-Type', 'text/html');

    // configure os cabeçalhos HTTP de cache
    $response->setMaxAge(10);

.. tip::

    Para debugar uma Resposta, altere o seu tipo para string, ela irá retornar a representação HTTP
    da resposta (cabeçalhos e conteúdo).

Por último, mas não menos importante, essas classes, como qualquer outra classe no código do 
Symfony, foram `auditadas`_ para verificação de problemas de segurança por uma empresa independente. E
sendo um projeto *Open-Source* também significa que muitos desenvolvedores, ao redor do
mundo, leram o código e já corrigiram potenciais problemas de segurança.
Quando foi a última vez que você encomendou uma auditoria de segurança profissional para o  
seu framework caseiro?

Mesmo algo tão simples, como obter o endereço IP do cliente, pode ser inseguro::

    <?php

    if ($myIp == $_SERVER['REMOTE_ADDR']) {
        // o cliente é conhecido, então, concede-se mais algum privilégio
    }

Ele funciona perfeitamente bem até você adicionar um proxy reverso na frente dos
servidores de produção; neste ponto, você terá que alterar o seu código para fazer
ele funcionar tanto na máquina de desenvolvimento (onde você não tem um proxy) quanto
nos seus servidores::

    <?php

    if ($myIp == $_SERVER['HTTP_X_FORWARDED_FOR'] || $myIp == $_SERVER['REMOTE_ADDR']) {
        // o cliente é conhecido, então, concede-se mais algum privilégio
    }

O método ``Request::getClientIp()`` lhe fornece o funcionamento 
correto desde o primeiro dia (e teria coberto também o caso onde você tem
proxies encadeados)::

    <?php

    $request = Request::createFromGlobals();

    if ($myIp == $request->getClientIp()) {
        // o cliente é conhecido, então, concede-se mais algum privilégio
    }

E há um benefício adicional: é *seguro* por padrão. O que quero dizer com 
seguro? Não se pode confiar no valor de ``$_SERVER['HTTP_X_FORWARDED_FOR']``, uma vez que,
ele pode ser manipulado pelo usuário final, quando não há proxy. Então, se você estiver
usando este código em produção, sem um proxy, torna-se bem fácil 
abusar do seu sistema. Isso não é o caso do método ``getClientIp()`` pois
você deve confiar explicitamente neste cabeçalho chamando ``trustProxyData()``::

    <?php

    Request::trustProxyData();

    if ($myIp == $request->getClientIp(true)) {
        // o cliente é conhecido, então, concede-se mais algum privilégio
    }

Assim, o método ``getClientIp()`` funciona com segurança em todas as circunstâncias. Você pode
usá-lo em todos os seus projetos, seja qual for a sua configuração, ele irá funcionar 
corretamente e com segurança. Esse é um dos objetivos da utilização de um framework. Se você fosse
escrever um framework a partir do zero, você teria que pensar em todos estes
casos por si mesmo. Por que, então, não usar uma tecnologia que já funciona?

.. note::

    Se você quiser saber mais sobre o componente HttpFoundation, você pode 
    dar uma olhada na `API`_ ou ler a `documentação`_ dedicada no site do 
    Symfony.

Acredite ou não, mas temos o nosso primeiro framework. Você pode parar agora se quiser.
Apenas usando o componente HttpFoundation do Symfony2 já é possível escrever
código melhor e mais testável. Ele também permite que você escreva código mais rápido pois, muitos
dos problemas do dia-a-dia, já foram resolvidos para você.

Por uma questão de fato, projetos como o Drupal adotaram (para a próxima
versão 8) o componente HttpFoundation, se funciona para eles, provavelmente 
funcionará para você também. Não reinvente a roda.

Eu quase me esqueci de falar sobre um benefício adicional: o uso componente HttpFoundation
é o início de uma melhor interoperabilidade entre todos os frameworks e
aplicações que o utilizam (atualmente: `Symfony2`_, `Drupal 8`_, `phpBB 4`_,
`Silex`_, `Midgard CMS`_, `Zikula`_ ...).

.. _`Twig`:                     http://twig.sensiolabs.com/
.. _`Symfony2 versus o PHP puro`: http://symfony.com/doc/current/book/from_flat_php_to_symfony2.html
.. _`especificação HTTP`:       http://tools.ietf.org/wg/httpbis/
.. _`API`:                      http://api.symfony.com/2.0/Symfony/Component/HttpFoundation.html
.. _`documentação`:             http://symfony.com/doc/current/components/http_foundation.html
.. _`auditadas`:                http://symfony.com/blog/symfony2-security-audit
.. _`Symfony2`:                 http://symfony.com/
.. _`Drupal 8`:                 http://drupal.org/
.. _`phpBB 4`:                  http://www.phpbb.com/
.. _`Silex`:                    http://silex.sensiolabs.org/
.. _`Midgard CMS`:              http://www.midgard-project.org/
.. _`Zikula`:                   http://zikula.org/
