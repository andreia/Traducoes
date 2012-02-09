O Componente EventDispatcher
============================

Ainda falta, em nosso framework, uma funcionalidade que é importante em qualquer bom framework:
*extensibilidade*. Sendo extensível significa que, o desenvolvedor poderá
facilmente acessar o ciclo de vida do framework para modificar a forma como o pedido é
manipulado.

Que tipo de *hooks* estamos falando? Autenticação ou armazenamento em cache, por exemplo. 
Para serem flexíveis, os *hooks* devem ser plug-and-play, os que você "registra"
para uma aplicação são diferentes do próximo, dependendo da sua necessidade
específica. Muitos softwares têm um conceito semelhante, como o Drupal ou Wordpress. Em 
algumas linguagens, existe um padrão como o `WSGI`_ no Python ou o `Rack`_ no Ruby.

Por não existir um padrão para PHP, vamos usar um *design pattern* bem conhecido,
o *Observer*, para permitir que qualquer tipo de comportamento seja anexado ao nosso
framework; o componente *EventDispatcher* do Symfony2 implementa uma versão leve
deste padrão:

.. code-block:: javascript

    {
        "require": {
            "symfony/class-loader": "2.1.*",
            "symfony/http-foundation": "2.1.*",
            "symfony/routing": "2.1.*",
            "symfony/http-kernel": "2.1.*",
            "symfony/event-dispatcher": "2.1.*"
        },
        "autoload": {
            "psr-0": { "Simplex": "src/", "Calendar": "src/" }
        }
    }

Como isso funciona? O *dispatcher*, o objeto central do sistema *dispatcher* de eventos, 
notifica os *listeners* (ou ouvintes) de um *evento* enviado à ele. Dito de outra forma:
o seu código envia um evento para o *dispatcher*, o *dispatcher* notifica todos os
*listeners* registrados para o evento, e cada *listener* faz o que desejar
com o evento.

Como exemplo, vamos criar um *listener* que adiciona o código do Google Analytics de forma 
transparente à todas as respostas.

Para fazer ele funcionar, o framework deve enviar um evento pouco antes de retornar
a instância de Resposta::

    <?php

    // example.com/src/Simplex/Framework.php

    namespace Simplex;

    use Symfony\Component\HttpFoundation\Request;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\Routing\Matcher\UrlMatcherInterface;
    use Symfony\Component\Routing\Exception\ResourceNotFoundException;
    use Symfony\Component\HttpKernel\Controller\ControllerResolverInterface;
    use Symfony\Component\EventDispatcher\EventDispatcher;

    class Framework
    {
        protected $matcher;
        protected $resolver;
        protected $dispatcher;

        public function __construct(EventDispatcher $dispatcher, UrlMatcherInterface $matcher, ControllerResolverInterface $resolver)
        {
            $this->matcher = $matcher;
            $this->resolver = $resolver;
            $this->dispatcher = $dispatcher;
        }

        public function handle(Request $request)
        {
            $this->matcher->getContext()->fromRequest($request);

            try {
                $request->attributes->add($this->matcher->match($request->getPathInfo()));

                $controller = $this->resolver->getController($request);
                $arguments = $this->resolver->getArguments($request, $controller);

                $response = call_user_func_array($controller, $arguments);
            } catch (ResourceNotFoundException $e) {
                $response = new Response('Not Found', 404);
            } catch (\Exception $e) {
                $response = new Response('An error occurred', 500);
            }

            // dispatch a response event
            $this->dispatcher->dispatch('response', new ResponseEvent($response, $request));

            return $response;
        }
    }

Cada vez que o framework lidar com um Pedido, um evento ``ResponseEvent``
será agora enviado::

    <?php

    // example.com/src/Simplex/ResponseEvent.php

    namespace Simplex;

    use Symfony\Component\HttpFoundation\Request;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\EventDispatcher\Event;

    class ResponseEvent extends Event
    {
        private $request;
        private $response;

        public function __construct(Response $response, Request $request)
        {
            $this->response = $response;
            $this->request = $request;
        }

        public function getResponse()
        {
            return $this->response;
        }

        public function getRequest()
        {
            return $this->request;
        }
    }

O último passo é a criação do *dispatcher* no ``front controller`` e
registrar um *listener* para o evento ``response``::

    <?php

    // example.com/web/front.php

    require_once __DIR__.'/../vendor/.composer/autoload.php';

    // ...

    use Symfony\Component\EventDispatcher\EventDispatcher;

    $dispatcher = new EventDispatcher();
    $dispatcher->addListener('response', function (Simplex\ResponseEvent $event) {
        $response = $event->getResponse();

        if ($response->isRedirection()
            || ($response->headers->has('Content-Type') && false === strpos($response->headers->get('Content-Type'), 'html'))
            || 'html' !== $event->getRequest()->getRequestFormat()
        ) {
            return;
        }

        $response->setContent($response->getContent().'GA CODE');
    });

    $framework = new Simplex\Framework($dispatcher, $matcher, $resolver);
    $response = $framework->handle($request);

    $response->send();

.. note::

    O *listener* é apenas uma prova de conceito e, você deve adicionar o código do Google
    Analytics antes da tag body.

Como você pode ver, o ``addListener()`` associa um callback PHP válido à um evento
nomeado (``response``); o nome do evento deve ser o mesmo utilizado no
chamada ``dispatch()``.

No *listener*, vamos adicionar o código do Google Analytics apenas se a resposta não for
um redirecionamento, se o formato solicitado é HTML e se o ``content type`` da resposta
é HTML (estas condições demonstram a facilidade de manipular os
dados do Pedido e da Resposta no seu código).

Até aqui tudo bem, mas, vamos adicionar outro *listener* no mesmo evento. Vamos dizer que
eu quero definir o ``Content-Length`` da Resposta, caso ele ainda não estiver 
definido::

    $dispatcher->addListener('response', function (Simplex\ResponseEvent $event) {
        $response = $event->getResponse();
        $headers = $response->headers;

        if (!$headers->has('Content-Length') && !$headers->has('Transfer-Encoding')) {
            $headers->set('Content-Length', strlen($response->getContent()));
        }
    });

Dependendo se você adicionou este pedaço de código antes do registro do *listener*
ou depois dele, você vai ter o valor errado ou correto para o cabeçalho ``Content-Length``. 
Às vezes, a ordem dos *listeners* importa, mas, por padrão, todos os *listeners* são 
registrados com a mesma prioridade, ``0``. Para dizer ao *dispatcher* para executar um 
*listener* antes, altere a prioridade para um número positivo; números negativos podem ser 
utilizados para os *listeners* de baixa prioridade.
Aqui, queremos que o *listener* ``Content-Length`` seja executado por último, então, altere
a prioridade para ``-255``::

    $dispatcher->addListener('response', function (Simplex\ResponseEvent $event) {
        $response = $event->getResponse();
        $headers = $response->headers;

        if (!$headers->has('Content-Length') && !$headers->has('Transfer-Encoding')) {
            $headers->set('Content-Length', strlen($response->getContent()));
        }
    }, -255);

.. tip::

    Ao criar o seu framework, tenha em mente as prioridades (reserve alguns números
    para os *listeners* internos, por exemplo) e documente-os totalmente.

Vamos refatorar o código um pouco movendo o *listener* Google para sua própria classe::

    <?php

    // example.com/src/Simplex/GoogleListener.php

    namespace Simplex;

    class GoogleListener
    {
        public function onResponse(ResponseEvent $event)
        {
            $response = $event->getResponse();

            if ($response->isRedirection()
                || ($response->headers->has('Content-Type') && false === strpos($response->headers->get('Content-Type'), 'html'))
                || 'html' !== $event->getRequest()->getRequestFormat()
            ) {
                return;
            }

            $response->setContent($response->getContent().'GA CODE');
        }
    }

E faça o mesmo com o outro *listener*::

    <?php

    // example.com/src/Simplex/ContentLengthListener.php

    namespace Simplex;

    class ContentLengthListener
    {
        public function onResponse(ResponseEvent $event)
        {
            $response = $event->getResponse();
            $headers = $response->headers;

            if (!$headers->has('Content-Length') && !$headers->has('Transfer-Encoding')) {
                $headers->set('Content-Length', strlen($response->getContent()));
            }
        }
    }

Nosso ``front controller`` deve ter agora a seguinte aparência:

    $dispatcher = new EventDispatcher();
    $dispatcher->addListener('response', array(new Simplex\ContentLengthListener(), 'onResponse'), -255);
    $dispatcher->addListener('response', array(new Simplex\GoogleListener(), 'onResponse'));

Mesmo que o código esteja agora bem envolto em classes, ainda há uma 
questão: o conhecimento das prioridades é "hardcoded" no ``front controller``,
em vez de estar nos *listeners*. Para cada aplicação, você deverá
lembrar-se de definir as prioridades apropriadas. Além disso, os nomes dos métodos 
*listener* também são expostos aqui, o que significa que ao fazer a refatoração de 
nossos ouvintes precisaremos alterar todas as aplicações que dependem desses *listeners*. 
É claro, há uma solução: utilize *subscribers* em vez de *listeners*::

    $dispatcher = new EventDispatcher();
    $dispatcher->addSubscriber(new Simplex\ContentLengthListener());
    $dispatcher->addSubscriber(new Simplex\GoogleListener());

Um *subscriber* conhece todos os eventos em que está interessado e passa estas
informações ao *dispatcher* através do método ``getSubscribedEvents()``. Dê uma 
olhada na nova versão do ``GoogleListener``::

    <?php

    // example.com/src/Simplex/GoogleListener.php

    namespace Simplex;

    use Symfony\Component\EventDispatcher\EventSubscriberInterface;

    class GoogleListener implements EventSubscriberInterface
    {
        // ...

        public static function getSubscribedEvents()
        {
            return array('response' => 'onResponse');
        }
    }

E aqui está a nova versão do ``ContentLengthListener``::

    <?php

    // example.com/src/Simplex/ContentLengthListener.php

    namespace Simplex;

    use Symfony\Component\EventDispatcher\EventSubscriberInterface;

    class ContentLengthListener implements EventSubscriberInterface
    {
        // ...

        public static function getSubscribedEvents()
        {
            return array('response' => array('onResponse', -255));
        }
    }

.. tip::

    Um único *subscriber* pode hospedar muitos *listeners* que você deseja em tantos
    eventos quando necessário.

Para tornar o seu framework verdadeiramente flexível, não hesite em adicionar mais eventos; e
para torná-lo mais impressionante, com tudo pronto que você precisa, adicione mais ouvintes. 
Mais uma vez, esta série não é sobre a criação de um framework genérico, mas sim um que é adaptado 
às suas necessidades. Pare quando achar melhor, e continue a evoluir o código a partir daí.

.. _`WSGI`: http://www.python.org/dev/peps/pep-0333/#middleware-components-that-play-both-sides
.. _`Rack`: http://rack.rubyforge.org/
