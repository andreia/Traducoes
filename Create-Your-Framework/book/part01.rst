Crie o seu próprio framework... utilizando os Componentes do Symfony2 (parte 1)
===============================================================================

.. note::
    Esta é uma tradução do artigo do Fabien Potencier, publicado `em seu blog`_.

O Symfony2 é um conjunto reutilizável de componentes PHP coesivos, independentes e desacoplados 
que solucionam problemas comuns do desenvolvimento web.

Em vez de usar esses componentes de baixo nível, você pode utilizar o framework web full-stack Symfony2, 
pronto para uso, que utiliza estes componentes como base ... ou
você pode criar o seu próprio framework. Esta série é sobre este último assunto.

.. note::

    Se você quer apenas usar o framework full-stack Symfony2, é melhor, no lugar, 
    ler a sua `documentação`_ oficial.

Por que você deseja criar o seu próprio framework?
---------------------------------------------------

Em primeiro lugar, porque você gostaria de criar o seu próprio framework? Se você
olhar em volta, todos vão dizer que é algo ruim reinventar a
roda, e, que é melhor você escolher um framework existente e esquecer completamente 
de criar o seu próprio. Na maioria das vezes, eles estão certos, mas eu posso pensar
em algumas boas razões para você começar a criar o seu próprio framework:

* Para saber mais sobre a arquitetura de baixo nível dos frameworks web modernos
  em geral e sobre o funcionamento interno do framework full-stack Symfony2, em particular;

* Para criar um framework adequado às suas necessidades muito específicas (não se esqueça
  de, primeiro, certificar-se que as suas necessidades são realmente muito específicas);

* Para experimentar a criação de um framework para se divertir (em uma abordagem "aprender e 
  descartar");

* Para refatorar uma aplicação antiga/existente que precisa de uma boa dose das
  melhores práticas de desenvolvimento web recentes;

* Para provar ao mundo que você realmente pode criar um framework próprio (...
  mas com pouco esforço).

Irei guiá-lo lentamente através da criação de um framework web, um passo de cada
vez. Em cada etapa, você terá um framework totalmente funcional que, poderá já utilizá-lo,
ou, então, usar como ponto de partida para o seu próprio. Começaremos com frameworks simples
e, com o tempo, mais recursos serão adicionados. Eventualmente, você terá um
framework web full-stack com recursos completos.

E, claro, cada passo será a ocasião para aprender mais sobre alguns dos
Componentes do Symfony2.

.. tip::

    Se você não tem tempo para ler toda a série, ou se você deseja obter
    um início rápido, você também pode dar uma olhada no micro-framework `Silex`_
    que é baseado nos Componentes do Symfony2. O código é bastante enxuto e ele aproveita
    muitos aspectos dos Componentes do Symfony2.

Muitos frameworks web modernos chamam a si mesmos de frameworks MVC. Nós não vamos falar sobre
MVC aqui, pois os Componentes do Symfony2 são capazes de criar qualquer tipo de framework,
não apenas os que seguem a arquitetura MVC. Enfim, se você olhar através da semântica MVC, 
esta série é sobre como criar a parte do Controlador de um framework. 
Para o Modelo e a Visão, realmente vai depender da sua preferência 
pessoal e eu vou deixar você utilizar qualquer uma das bibliotecas existentes de terceiros (Doctrine,
Propel, ou apenas o PDO para o modelo; e PHP ou Twig para a Visão).

Ao criar um framework, seguir o padrão MVC não é a forma correta.
O objetivo principal deve ser o "Separation of Concerns", eu realmente acho que este
é o padrão de design que você deve realmente se preocupar. Os princípios 
fundamentais dos Componentes do Symfony2 estão centrados na especificação 
HTTP. Como tal, os frameworks que nós vamos criar devem ser
mais precisamente rotulados como frameworks HTTP ou frameworks Request/Response.

Antes de Começarmos
-------------------

Ler sobre como criar um framework não é o suficiente. Você terá que acompanhar
juntamente, digitando todos os exemplos que iremos trabalhar. Para isso, você precisa de uma
versão recente do PHP (5.3.8 ou posterior é o suficiente), um servidor web (como o 
Apache ou Nginx), um bom conhecimento de PHP e uma compreensão de programação 
Orientada à Objeto.

Pronto para ir? Vamos começar!

Bootstrapping
-------------

Antes de sequer pensarmos em criar o nosso primeiro framework, precisamos conversar
sobre algumas convenções: onde vamos guardar o nosso código, como nós iremos nomear as nossas
classes, como vamos referenciar dependências externas, etc.

Para armazenar o nosso framework, crie um diretório em algum lugar na sua máquina:

.. code-block:: sh

    $ mkdir framework
    $ cd framework

Padrão de Codificação
~~~~~~~~~~~~~~~~~~~~~

Antes que alguém comece uma *flame war* sobre padrões de codificação e porque determinado padrão  
está sendo utilizado aqui, vamos todos admitir que isso não importa muito, desde que
você seja consistente. Para este livro, vamos usar o `Padrão de Codificação do 
Symfony2`_.

Instalação de Componentes
~~~~~~~~~~~~~~~~~~~~~~~~~

Para instalar os Componentes do Symfony2 que precisamos para o nosso framework, vamos
usar o `Composer`_, um gerenciador de dependências de projeto para PHP. Primeiro, liste as suas
dependências em um arquivo ``composer.json``:

.. code-block:: json

    # framework/composer.json
    {
        "require": {
            "symfony/class-loader": "2.1.*"
        }
    }

Aqui, nós dizemos ao Composer que o nosso projeto depende do componente ClassLoader do 
Symfony2, versão 2.1.0 ou posterior. Para realmente instalar as dependências 
do projeto, baixe o binário do composer e execute-o:

.. code-block:: sh

    $ wget http://getcomposer.org/composer.phar
    $ # or
    $ curl -O http://getcomposer.org/composer.phar

    $ php composer.phar install

Depois de executar o comando ``install``, você verá um novo diretório
``vendor/`` que deve conter o código do ClassLoader do Symfony2.

.. note::

    Mesmo nós recomendando fortemente que você utilize o Composer, você também pode baixar
    os arquivos dos componentes diretamente ou usar o Git submodules. Isso depende 
    somente de você.

Convenções de Nomenclatura e Autoloading
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Nós vamos agora fazer o `autoload`_ de todas as nossas classes. Sem o autoloading, você precisa
especificar o arquivo onde uma classe é definida antes de poder usá-la. Mas,
com algumas convenções, podemos simplesmente deixar o PHP fazer este trabalho duro para nós.

O Symfony2 segue o padrão PHP de-facto, `PSR-0`_, para os nomes de classe e
o autoloading. O Componente ClassLoader do Symfony2 fornece um autoloader que
implementa o padrão PSR-0 e, na maioria das vezes, o ClassLoader do Symfony2
é tudo o que você precisa para fazer o autoload de todas as classes de seu projeto.

Crie um autoloader vazio em um novo arquivo ``autoload.php``:

.. code-block:: php

    <?php

    // framework/autoload.php

    require_once __DIR__.'/vendor/symfony/class-loader/Symfony/Component/ClassLoader/UniversalClassLoader.php';

    use Symfony\Component\ClassLoader\UniversalClassLoader;

    $loader = new UniversalClassLoader();
    $loader->register();

Agora você pode executar ``autoload.php`` na linha de comando, ele não deve fazer nada e
não deve exibir nenhum erro:

.. code-block:: sh

    $ php autoload.php

.. tip::

    O site do Symfony contém mais informações sobre o componente
    `ClassLoader`_.

.. note::

    O Composer cria automaticamente um autoloader para todas as suas dependências
    instaladas; em vez de usar o componente ClassLoader, você também pode
    apenas utilizar o require ``vendor/.composer/autoload.php``.

Nosso Projeto
-------------

Em vez de criar o nosso framework a partir do zero, vamos escrever a mesma
"aplicação" repetidamente, adicionando uma abstração no momento. Vamos 
iniciar com a aplicação web mais simples que podemos pensar em PHP::

    <?php

    $input = $_GET['name'];

    printf('Hello %s', $input);

Isso é tudo para a primeira parte desta série. No próximo artigo, vamos introduzir o
Componente HttpFoundation e ver o que ele nos fornece.

.. _`documentação`:             http://symfony.com/doc
.. _`Silex`:                     http://silex.sensiolabs.org/
.. _`autoload`:                  http://fr.php.net/autoload
.. _`Composer`:                  http://packagist.org/about-composer
.. _`PSR-0`:                     https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-0.md
.. _`Padrão de Codificação do Symfony2`: http://symfony.com/doc/current/contributing/code/standards.html
.. _`ClassLoader`:               http://symfony.com/doc/current/components/class_loader.html
.. _`em seu blog`:               http://fabien.potencier.org/article/50/create-your-own-framework-on-top-of-the-symfony2-components-part-1
