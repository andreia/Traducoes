Crie o seu próprio framework... utilizando os Componentes do Symfony2 (parte 4)
===============================================================================

Antes de iniciarmos o tópico de hoje, vamos refatorar um pouco o nosso framework atual, 
para tornar os templates ainda mais legíveis::

    <?php

    // example.com/web/front.php

    require_once __DIR__.'/../src/autoload.php';

    use Symfony\Component\HttpFoundation\Request;
    use Symfony\Component\HttpFoundation\Response;

    $request = Request::createFromGlobals();

    $map = array(
        '/hello' => 'hello',
        '/bye'   => 'bye',
    );

    $path = $request->getPathInfo();
    if (isset($map[$path])) {
        ob_start();
        extract($request->query->all(), EXTR_SKIP);
        include sprintf(__DIR__.'/../src/pages/%s.php', $map[$path]);
        $response = new Response(ob_get_clean());
    } else {
        $response = new Response('Not Found', 404);
    }

    $response->send();

Como extraímos agora os parâmetros da query do pedido, simplifique o template
``hello.php`` como segue::

    <!-- example.com/src/pages/hello.php -->

    Hello <?php echo htmlspecialchars($name, ENT_QUOTES, 'UTF-8') ?>

Agora, estamos em boa forma para adicionarmos novos recursos.

Um aspecto muito importante de qualquer site é o formato das suas URLs. Graças ao
mapa de URL, temos dissociadas as URLs do código que gera a
resposta associada, mas ainda não é suficientemente flexível. Por exemplo, poderíamos
desejar suporte à caminhos dinâmicos para permitir a incorporação de dados diretamente dentro da URL
ao invés de depender de uma query string::

    # Antes
    /hello?name=Fabien

    # Depois
    /hello/Fabien

Para suportar este recurso, vamos usar o componente Routing do Symfony2.
Como sempre, adicione-o ao ``composer.json`` e execute o comando
``php composer.phar update`` para instalá-lo:

.. code-block:: json

    {
        "require": {
            "symfony/class-loader": "2.1.*",
            "symfony/http-foundation": "2.1.*",
            "symfony/routing": "2.1.*"
        }
    }

A partir de agora, vamos usar o autoloader Composer gerado em vez de
nosso próprio ``autoload.php``. Remova o arquivo ``autoload.php`` e substitua sua
referência em ``front.php``::

    <?php

    // example.com/web/front.php

    require_once __DIR__.'/../vendor/.composer/autoload.php';

    // ...

Em vez de um array para o mapa de URL, o componente Routing baseia-se em uma
instância ``RouteCollection``::

    use Symfony\Component\Routing\RouteCollection;

    $routes = new RouteCollection();

Vamos adicionar uma rota que descreve a URL ``/hello/SOMETHING`` e adiciona uma outra
para o simples ``/bye``::

    use Symfony\Component\Routing\Route;

    $routes->add('hello', new Route('/hello/{name}', array('name' => 'World')));
    $routes->add('bye', new Route('/bye'));

Cada entrada na coleção é definida por um nome (``hello``) e uma instância
``Route``, que é definida por um padrão de rota (``/hello/{name}``) e um array
de valores padrão para os atributos de rota (``array('name' => 'World')``).

.. note::

    Leia a `documentação`_ oficial do componente Routing para saber mais sobre as 
    suas muitas funcionalidades como: a geração de URL, os requisitos de atributo, 
    aplicação de métodos HTTP, carregadores para arquivos YAML ou XML, dumpers para 
    o PHP ou regras de rewrite do Apache para um melhor desempenho e muito mais.

Com base nas informações armazenadas na instância ``RouteCollection``, uma
instância ``UrlMatcher`` pode buscar os caminhos URL correspondentes::

    use Symfony\Component\Routing\RequestContext;
    use Symfony\Component\Routing\Matcher\UrlMatcher;

    $context = new RequestContext();
    $context->fromRequest($request);
    $matcher = new UrlMatcher($routes, $context);

    $attributes = $matcher->match($request->getPathInfo());

O método ``match()`` utiliza o caminho do pedido e retorna um array de atributos
(note que a rota correspondente é automaticamente armazenada sob um atributo 
``_route`` especial)::

    print_r($matcher->match('/bye'));
    array (
      '_route' => 'bye',
    );

    print_r($matcher->match('/hello/Fabien'));
    array (
      'name' => 'Fabien',
      '_route' => 'hello',
    );

    print_r($matcher->match('/hello'));
    array (
      'name' => 'World',
      '_route' => 'hello',
    );

.. note::

    Mesmo se não for estritamente necessário o contexto do pedido em nossos exemplos, ele é
    usado em aplicações do mundo real para impor requisitos de método e muito mais.

O ``URL matcher`` gera uma exceção quando nenhuma das rotas corresponder::

    $matcher->match('/not-found');

    // throws a Symfony\Component\Routing\Exception\ResourceNotFoundException

Com estas informações em mente, vamos escrever a nova versão do nosso framework:

  .. code-block:: php

    <?php

    // example.com/web/front.php

    require_once __DIR__.'/../vendor/.composer/autoload.php';

    use Symfony\Component\HttpFoundation\Request;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\Routing;

    $request = Request::createFromGlobals();
    $routes = include __DIR__.'/../src/app.php';

    $context = new Routing\RequestContext();
    $context->fromRequest($request);
    $matcher = new Routing\Matcher\UrlMatcher($routes, $context);

    try {
        extract($matcher->match($request->getPathInfo()), EXTR_SKIP);
        ob_start();
        include sprintf(__DIR__.'/../src/pages/%s.php', $_route);

        $response = new Response(ob_get_clean());
    } catch (Routing\Exception\ResourceNotFoundException $e) {
        $response = new Response('Not Found', 404);
    } catch (Exception $e) {
        $response = new Response('An error occurred', 500);
    }

    $response->send();

Existem algumas coisas novas no código:

* Os nomes das Rotas são usados ​​para os nomes dos templates;

* Erros ``500`` são gerenciados corretamente agora;

* Os atributos do Pedido são extraídos para manter os nossos templates simples::

      <!-- example.com/src/pages/hello.php -->

      Hello <?php echo htmlspecialchars($name, ENT_QUOTES, 'UTF-8') ?>

* A configuração das Rotas foi movida para o seu próprio arquivo:

  .. code-block:: php

      <?php

      // example.com/src/app.php

      use Symfony\Component\Routing;

      $routes = new Routing\RouteCollection();
      $routes->add('hello', new Routing\Route('/hello/{name}', array('name' => 'World')));
      $routes->add('bye', new Routing\Route('/bye'));

      return $routes;

  Agora temos uma separação clara entre a configuração (tudo
  o que for referente a nossa aplicação em ``app.php``) e o framework (o código genérico
  que alimenta a nossa aplicação em ``front.php``).

Com menos de 30 linhas de código, temos um framework novo, mais poderoso e
flexível do que o anterior. Divirta-se!

O uso do componente Routing tem uma outra grande vantagem adicional: a capacidade de
gerenciar URLs com base nas definições da rota. Ao utilizar ``URL matching`` e ``URL generation``
em seu código, quando for alterar os padrões de URL, não deverá ter nenhum outro
impacto. Quer saber como usar o gerador? Insanamente fácil::

    use Symfony\Component\Routing;

    $generator = new Routing\Generator\UrlGenerator($routes, $context);

    echo $generator->generate('hello', array('name' => 'Fabien'));
    // outputs /hello/Fabien

O código deve ser auto-explicativo, e, graças ao contexto, você pode até mesmo
gerar URLs absolutas::

    echo $generator->generate('hello', array('name' => 'Fabien'), true);
    // outputs something like http://example.com/somewhere/hello/Fabien

.. tip::

    Preocupado com o desempenho? Com base nas definições da sua rota, crie uma
    classe ``URL matcher`` altamente otimizada que possa substituir o ``UrlMatcher``
    padrão::

        $dumper = new Routing\Matcher\Dumper\PhpMatcherDumper($routes);

        echo $dumper->dump();

    Quer ainda mais desempenho? Faça ``dump`` de suas rotas como um conjunto de regras de rewrite do 
    Apache::

        $dumper = new Routing\Matcher\Dumper\ApacheMatcherDumper($routes);

        echo $dumper->dump();

.. _`documentação`: http://symfony.com/doc/current/components/routing.html